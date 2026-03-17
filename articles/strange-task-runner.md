---
title: "Vite+ の異常なタスクランナー: vite-task は如何にしてキャッシュの手動依存管理をなくしたか"
emoji: "🧪"
type: "tech"
topics: ["linux", "rust", "vite", "viteplus"]
published: true
publication_name: "herp_inc"
---

原題『_Vite+の異常なタスクランナー または vite-task は如何にしてキャッシュの手動依存管理を止めてファイルアクセスを自動捕捉するようになったか_』^[『博士の異常な愛情』のパロディタイトルをつけようとしたら Zenn の記事タイトルの文字数制限に引っかかった。悔しい。]

---

先日、ついに Vite+ の alpha 版がリリースされました。oxc エコシステムの全面バックアップを受けた高速なネイティブ実装ツールチェーンが揃い踏みしており、開発コミュニティも沸き立っています。
私も仕事で扱っているプロジェクトを手元で Vite+ に置き換えてみたのですが、 **_vp (ヴイピー) 鬼はえええ！このまま遅いやつら全部 Vite+ に置き換えていこうぜ！_** という気持ちになりました。

そんな Vite+ ですが、Rolldown や oxlint, oxfmt に紛れて、かなり面白いことをしているツールが入っています。**vite-task** です。

## vite-task: キャッシュの手動依存管理からの解放

[vite-task](https://github.com/voidzero-dev/vite-task) は、[Turborepo](https://turborepo.dev/) や [Wireit](https://github.com/google/wireit) のような monorepo 向けタスクランナーの仲間です。

https://github.com/voidzero-dev/vite-task

vite-task 最大の特徴は、既存タスクランナーと異なり、キャッシュのキーとなるファイルを手動で管理する必要がないことです。

どういうことかというと、大規模プロジェクトでの使用を前提として設計されている Turborepo のようなタスクランナーは、どれも効率的にプロジェクトを静的検査・ビルドするためにキャッシュ機構を搭載しています。monorepo 内の複数のパッケージのうち、変更がないものについては静的検査・ビルドをスキップし、キャッシュに保存されている前回の実行結果を再利用することで、時間を節約するわけです。

ただし、この種のキャッシュ機構には、タスクの実行結果の再現性を開発者が正しく担保してやらないといけないという運用上の難点があります。例えば、Turborepo や Wireit の場合、そのタスクの実行結果を左右するファイル（ソースコード、静的アセット、その他）を設定ファイルに記述する必要があります^[一応、何から何まで全て開発者が明示的に指定しないといけないわけではなく、ある程度暗黙的にキャッシュの依存となるファイルを捕捉してくれます。]。これは不足すると誤ったキャッシュを使用することになり、逆に広く指定しすぎるとキャッシュヒット率が低下します。

vite-task は、「vite-task から起動したプロセスが読み取ったファイルを自動で捕捉し、キャッシュのキーとして利用する」という方法でこの問題を解決します。

## fspy

vite-task は、Vite+ ツールチェーンの例に漏れず Rust 実装です。プロセスがアクセスしたファイルをトレースする仕組みは [fspy](https://github.com/voidzero-dev/vite-task/tree/main/crates/fspy) という crate に切り出されています。fspy の動作検証用 CLI があるので、vite-task をクローンして実行してみましょう。

:::message

話を簡単にするため、以降は前置きのない限り Linux を前提として話を進めます。コマンド例については、以下の Dockerfile を用いて、aarch64-darwin の Docker 上で実行しています。

:::details Dockerfile

```Dockerfile
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
  curl \
  git \
  build-essential \
  pkg-config \
  libssl-dev \
  ca-certificates \
  musl-tools \
  && rm -rf /var/lib/apt/lists/*

# Install rustup (nightly version is picked up from rust-toolchain.toml in the repo)
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain none
ENV PATH="/root/.cargo/bin:$PATH"

WORKDIR /workspace

RUN git clone https://github.com/voidzero-dev/vite-task .

# Install the toolchain specified in rust-toolchain.toml
RUN rustup show active-toolchain || rustup toolchain install

# Add musl target matching the host arch (required by fspy's artifact dependency on fspy_test_bin)
RUN case "$(uname -m)" in \
      aarch64) rustup target add aarch64-unknown-linux-musl ;; \
      x86_64)  rustup target add x86_64-unknown-linux-musl ;; \
    esac

# .cargo/config.toml uses zigcc wrappers intended for non-Linux hosts.
# On native Linux, musl-gcc can link musl targets directly, so replace them.
RUN printf '#!/bin/sh\nexec musl-gcc -fno-sanitize=all "$@"\n' \
      > .cargo/zigcc-aarch64-unknown-linux-musl \
    && printf '#!/bin/sh\nexec musl-gcc -fno-sanitize=all "$@"\n' \
      > .cargo/zigcc-x86_64-unknown-linux-musl \
    && chmod +x .cargo/zigcc-aarch64-unknown-linux-musl .cargo/zigcc-x86_64-unknown-linux-musl

# Pre-fetch dependencies to make subsequent builds faster
RUN cargo fetch

CMD ["bash"]
```

:::

手始めに `cat` をトレースしてみます。

```bash
# `cat ./package.json` を fspy でトレースする
cargo run -p fspy --example cli -- -o - cat ./package.json
```

上側は通常の `cat` の出力結果で、下側が fspy が捕捉したファイルの一覧です。`package.json` の読み取りが捕捉されています。

```:出力結果
# cat の出力結果は省略
"""/workspace/./package.json""",AccessMode(READ)
"""/usr/bin/cat""",AccessMode(READ)
"""/usr/bin/cat""",AccessMode(READ)
```

トレース対象から更に子プロセスを起動するとどうなるのか見てみます。

```bash
# `bash -c 'cat ./package.json'` を fspy でトレースする
cargo run -p fspy --example cli -- -o - bash -c 'cat ./package.json'
```

`"""/workspace/./package.json""",AccessMode(READ)` と、こちらも `package.json` の読み取りを捕捉できていることが分かります。

```:出力結果
# cat の出力結果は省略
"""/workspace""",AccessMode(READ)
"""/workspace/.""",AccessMode(READ)
"""/workspace""",AccessMode(READ)
"""/workspace/.""",AccessMode(READ)
"""/root/.cargo/bin/cat""",AccessMode(READ)
"""/usr/local/sbin/cat""",AccessMode(READ)
"""/usr/local/bin/cat""",AccessMode(READ)
"""/usr/sbin/cat""",AccessMode(READ)
"""/usr/bin/cat""",AccessMode(READ)
"""/usr/bin/cat""",AccessMode(READ)
"""/usr/bin/cat""",AccessMode(READ)
"""/usr/bin/cat""",AccessMode(READ)
"""/usr/bin/cat""",AccessMode(READ)
"""/usr/bin/cat""",AccessMode(READ)
"""/usr/bin/cat""",AccessMode(READ)
"""/workspace/./package.json""",AccessMode(READ)
"""/usr/bin/bash""",AccessMode(READ)
"""/usr/bin/bash""",AccessMode(READ)
```

## fspy の仕組みをどう実現するか

プロセスを監視し、ファイルアクセスに関するシステムコールを捕捉できればよさそうです。

### ptrace(2)

https://man7.org/linux/man-pages/man2/ptrace.2.html

ptrace は、あるプロセス（"tracer"）がトレース対象のプロセス（"tracee"）を監視し、システムコール呼び出しなどを追跡するために使われるシステムコールです。
システムコールの勉強をすると、`strace` コマンドとして実行することになるアレです。

最も素朴な方法だと思いますが、システムコールのたびに

1. tracee プロセスを停止
2. tracer プロセスにコンテキストスイッチ
3. 処理完了後、tracee プロセス再開

というステップを踏むため、致命的なオーバーヘッドが生じてしまいます。使うと遅くなるタスクランナーなんてキャッシュ以前の問題なので、今回のケースでは適さなそうです。

### seccomp(2)

https://man7.org/linux/man-pages/man2/seccomp.2.html

ptrace が全てのシステムコールを監視するせいでゲロ遅になってしまうのであれば、興味のあるシステムコールだけ監視すればいいじゃない、ということで使えるのが seccomp システムコールです。

seccomp (Secure Computing Mode) は、Linux カーネルのセキュリティ機能の1つで、あるプロセスからのシステムコールを制限することができます。サンドボックス化の仕組みとして、コンテナランタイムなどに用いられています。

seccomp には `SECCOMP_RET_USER_NOTIF` というモードがあります。このモードでは、指定したシステムコールが発行されると対象プロセスを一時停止し、事前に指定しておいたユーザー空間の監視用プロセスに通知、そこでシステムコールを許可するか否か処理することができます。つまり、これを使えば、特定のシステムコール専用の ptrace のようなことができるわけです。ptrace と同様の構造のオーバーヘッドはあるものの、一部のシステムコールについてのみそれが発生するので一定許容できるでしょう。

fspy は、後述する本命の方法が使えない場合、補助的に seccomp によるシステムコールの捕捉を行います。seccomp とのバインディングは `fspy_seccomp_unotify` という crate に切り出されています。

https://github.com/voidzero-dev/vite-task/tree/main/crates/fspy_seccomp_unotify

#### 余談: eBPF

seccomp のフィルターは Berkeley Packet Filter (BPF) として記述されます。BPF を機能拡張し、さらに柔軟なプログラムを書けるようにしたのが eBPF (extended Berkeley Packet Filter) です。

素人考えながら eBPF を使えばカーネル空間に閉じて色々できるのでコンテキストスイッチを減らせるのでは？と思いましたが、eBPF の実行には特権が必要なので今回のケースでは不適格でした（あたりまえ体操）。また、eBPF が基本的に read-only なのに対し、後述の手法で `execve` をインターセプトし、子プロセスへ `LD_PRELOAD` 環境変数を注入する必要があるため、もし eBPF を採用していたとしてもハイブリッドな構成が必要なようです。

## vite-task の解法: libc をフックする

前述の手法は「システムコールを直接インターセプトする」というアイディアでした。しかし、それだとどうしてもオーバーヘッドが生じてしまいます。

そこで vite-task が取ったのが「libc の関数呼び出しをインターセプトする」という手法です。
世の大半のソフトウェアは libc を介してシステムコールを発行しています。なので、ユーザー空間内でそこをインターセプトすることで、seccomp 方式のようなオーバーヘッドなしにファイルアクセスのトレースを実現するという戦略です。

### LD_PRELOAD による共有ライブラリのロード

`LD_PRELOAD` は、Linux の動的リンカ用の環境変数で、プロセス起動時にこの環境変数に指定した共有ライブラリを他のライブラリより先にロードさせます。
vite-task には `fspy_preload_unix` という Linux macOS 向けの共有ライブラリを実装した crate があり、ここではトレース対象となる libc の関数と同名の関数が定義されています。リンカは先にロードされたライブラリを優先するので、これで libc の呼び出しを横取りできます。

https://github.com/voidzero-dev/vite-task/blob/main/crates/fspy_preload_unix

例として、`open()` 関数を見てみます。`open()` は、[open(2)](https://man7.org/linux/man-pages/man2/open.2.html) システムコールを呼び出してファイルを開く（ファイルディスクリプタを取得する）関数です。

https://github.com/voidzero-dev/vite-task/blob/63f07330e8037b0504f830367eadf80a05026877/crates/fspy_preload_unix/src/interceptions/open.rs#L28-L41

最終的にはオリジナルの libc の関数（`open::original()`）を呼び出すようになっています。この共有ライブラリはあくまで透過的にトレースするのが目的です。

この方式では、fspy はあるタスクのプロセスがどのパスにどのモード（`READ`, `WRITE`, `READ_DIR`）でアクセスしたかのログ（[定義](https://github.com/voidzero-dev/vite-task/blob/63f07330e8037b0504f830367eadf80a05026877/crates/fspy_shared/src/ipc/mod.rs#L37-L42)）をインプロセスで共有メモリに書き込み、タスク完了後にそれを一括で読み取るという、ランタイムの協調を必要としない仕組みになっているので、seccomp 方式のようにプロセスはブロックされず、コンテキストスイッチが発生しません。

### 子プロセスのトレース

タスクランナーが直接実行したプロセスについては問題なさそうですが、そのプロセスからさらに子プロセスが起動された場合はどうなるでしょうか？

fspy は libc の exec 系関数もインターセプトします。例えば、`execve()` 関数（[execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) システムコールに対応）は、`handle_exec()` を呼び出すようになっています。

https://github.com/voidzero-dev/vite-task/blob/63f07330e8037b0504f830367eadf80a05026877/crates/fspy_preload_unix/src/interceptions/spawn/exec/mod.rs#L56-L67

これを辿っていくと、タスクランナーの一番最初にプロセスを起動する箇所と同様に、`LD_PRELOAD` の設定が行われています。よって、fspy はプロセスツリー全体を再帰的にトレースすることができます。

### 静的リンクの場合

当然ながら、この手法は libc を動的リンクしない静的リンクのバイナリ（Go 製バイナリなど）では使えません。libc が動的リンクされていないバイナリが execve された場合は、seccomp をフォールバックとして使用します。

https://github.com/voidzero-dev/vite-task/blob/63f07330e8037b0504f830367eadf80a05026877/crates/fspy_shared_unix/src/spawn/linux/mod.rs#L35-L46

## まとめ

高レイヤーから徐々に低レイヤーへ興味・関心が移っていくというのはソフトウェアエンジニアあるあるですが、最近は JS エコシステムにネイティブ実装のツールチェーンが増えてきたので、Web フロントエンドから直接そういった領域に飛ぶ機会も増えてきました。面白いですね。

特に vite-task は、JS 関連のツールチェーンでよく見聞きするパーサー・コンパイラー関連の話とは異なり、より OS と近接した技術領域の話がメインだったので、テンションが上がりました（~~でもこれ Web と関係なくね？~~）。

なお、筆者は特段 OS や低レイヤーの知識があるわけではなく、教科書レベルの OS 知識を元手に調べ上げた感じの記事になっているので、誤りなどあればご指摘願います。

vite-task のこの機構は結構すごい仕組みで動いているので、実際に使うには怖い部分もありますが、キャッシュの依存関係の手動管理から解放されるのであれば全然 welcome と個人的には思っているので、今後の動向や実運用の知見含めてウォッチしていきたいと思っています（終）

---

徹夜明け朝5時くらいに「vite-task おもろすぎ！」と衝動的に記事を書き始めてしまいましたが、弊社がフルフレックスだったので助かりました。弊社は衝動的技術記事書き人を積極的に募集しております。

https://herp.careers/v1/herpinc/9tYFbxcUPSEP

https://herp.careers/v1/herpinc/-Z20OVrOz6m8

あと私が勝手に Vite+ に置き換えて遊んでいたプロダクトはこちらです。

https://lp.herp.cloud/ai-recruiter
