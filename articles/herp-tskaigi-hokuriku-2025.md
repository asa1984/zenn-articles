---
title: "TSKaigi Hokuriku 2025 現地参加レポート"
emoji: "✍️"
type: "idea"
topics: ["イベントレポート", "TypeScript", "TSKaigi"]
published: true
publication_name: "herp_inc"
---

こんにちは、[HERP](https://herp.co.jp/) で TypeScript をシバき回して生計を立てている asa1984 です。

先日、言わずと知れた TypeScript カンファレンス TSKaigi の地方版: [TSKaigi Hokuriku 2025](https://hokuriku.tskaigi.org/) にスポンサーとして参加してきました。

![青い太枠の中に白い背景の長方形型のカード。左上にTSKaigi Hokuriku 2025 というロゴ、中央に HERP というロゴ、中央下側に1行目は株式会社HERP、2行目には Silver という文字、右下には卵形のキャラクターが3つ配置されている。](/images/herp-tskaigi-hokuriku-2025/sponsor-herp.jpeg)

かなり良いカンファレンスだったので、面白かったセッションを紹介しながらその様子をレポートしたいと思います。

## セッション紹介

独断と偏見で現地で観てきたセッションをいくつかピックアップして紹介します。

### TypeScript 6.0で非推奨化されるオプションたち

https://speakerdeck.com/uhyo/typescript-6-dot-0defei-tui-jiang-hua-sareruopusiyontati

[uhyo](https://zenn.dev/uhyo) さんによる基調講演です。
TypeScript では 7.0 (tsgo) のリリースに向けて、5.0 から古いオプションの非推奨化運動が進んでおり、遂に 6.0 で削除されるオプションについての話が講演の内容です。

そもそも TypeScript はオプションが多く、ブラウザと Node.js の溝や TypeScript 黎明期の独自仕様など、カオスな歴史的背景とその後方互換性の担保が複雑に絡み合っていますが、それらを今後の TypeScript の方向性も踏まえつつ整理して解説してくれる大変ありがたいセッションでした。

私は ESM/CommonJS 周りに散々苦しめられてきた人間なのですが、6.0 ではモジュールに関するオプションがいくつか廃止されており、その中に ESM/CommonJS ですらない聞いたこともないモジュールシステム（`amd`, `umd`, `systemjs`）があり、まだ自分が知らないカオスの歴史があったのかとかなりウケてしまいました。

一方で明るい未来の話として ESM を推進する流れが TypeScript にも来ているようです。`esModuleInterop` オプションは廃止・デフォルト有効化され、別のセッション（『`tsc --init` の設計思想の変化とその背景を追う - “教育的”アプローチから実用性重視への転換』^[こちらも面白いセッションでした。]）の内容ですが、`tsc --init` で生成される `tsconfig.json` のデフォルト設定も ESM を想定したものに変わるようです^[こちらの変更は TS 5.9 から適用される。]。
個人的に [Move on to ESM-only](https://antfu.me/posts/move-on-to-esm-only) を推しているので、この流れは嬉しい限りです。

### フロントエンドアーキテクチャの設計方法論 Feature-Sliced Designの紹介

https://speakerdeck.com/motikoma/tskaigi-hokuriku-2025-hurontoentoakitekutiyanoshe-ji-fang-fa-lun-feature-sliced-designnoshao-jie

ASCEND さんのスポンサー LT で、フロントエンドのアーキテクチャを **Feature-Sliced Design**（巷で "features ディレクトリ" と呼ばれがちなドメインごとに凝集性を高めるアレ）に移行したという内容です。

最近、弊社は Cycle.js という UI フレームワークの React 移行を進めていますが、まさしく同様の課題（Container コンポーネントの肥大化）を感じており、LT を聞いて Cycle.js の React 移行は実質 Feature-Sliced Design への移行ともいえるな〜と思いました。というのも Cycle.js はその設計思想上、状態管理やロジックを Container コンポーネントに相当する、プログラムのエントリーポイントに近い部分へ集約することを強制します。その点、React は分散統治的な設計を志向している^[これは Meta がコンポーネント設計においてコロケーションを重視している（GraphQL のコロケーション、RSC の fetch 重複排除）ことから読み取れる。]ので、Cycle.js に比べると自然と Feature-Sliced Design に近づきます。

私が今開発に携わっているプロダクトは React かつ Feature-Sliced Design になっているので、参考になると同時に首がもげるぐらい共感しました。みんなも使おう Feature-Sliced Design.

### 「TSのAPI型安全」の対価は誰が払う？ 不公平なスキーマ駆動に終止符を打つハイブリッド戦略

https://speakerdeck.com/hal_spidernight/tsnoapixing-an-quan-nodui-jia-hashui-kafu-u-bu-gong-ping-nasukimaqu-dong-nizhong-zhi-fu-woda-tuhaihuritutozhan-lue

スキーマ駆動開発について、それを適用すべき開発フェーズなのか？保守コストはどうなるのか？開発プロセスはどうする？といった視点から切り込んだセッションです。

自分自身、技術好きな者として、技術選定・戦略についての発信を見るとつい手法ばかりに目がいきがちですが、実際の仕事に落とし込む場合はその手法をとるに至ったコンテキストを理解する方が大切になってきます。その点、このセッションでは、開発のステークホルダー、目的と必要な合意、フェーズごとに変化するトレードオフといった観点からスキーマ駆動開発を整理しており、実践的で参考になる内容でした。

懇親会にて登壇者の方とお話したのですが、スライド中に登場する開発コストのグラフは実際にストーリーポイント^[アジャイル開発でユーザーストーリーなどのタスクの相対的な規模を見積もるための単位。]を元に算出しているらしいです（すごい）。

### Building AI Agents with TypeScript

https://speakerdeck.com/izumin5210/building-ai-agents-with-typescript

[Vercel AI SDK](https://ai-sdk.dev/) を用いてどのように TypeScript で AI エージェントを構築していくかを紹介するセッションです。

AI SDK は弊社でも利用していますが^[[HERP AIリクルーター](https://lp.herp.cloud/ai-recruiter)で活躍中。]、スライドで紹介されている通りとても便利なライブラリで、簡単なチャットツール程度であればすぐに構築できます。

一方で、多段ステップで実行されるワークフローや agentic な振る舞いを実装し、それを実際に本番環境で提供するには、そもそも生成に時間がかかり不安定な LLM と、LLM が呼び出す外部 API などを組み合わせてもタスクを最後まで実行しきれるような実行基盤が必要という話がなされていました。これは本当にその通りで弊社プロダクトの [HERP AIリクルーター](https://lp.herp.cloud/ai-recruiter) でも同様の課題に直面しました。

このセッションでは durable (途中でクラッシュしてもリトライして耐える) で resumable (任意の時間一時停止してその後再開できる) な実行基盤として [vercel/workflow](https://vercel.com/docs/workflow) が紹介されています。

vercel/workflow は Vercel お馴染みの `use` ディレクティブを用いることで、本来ジョブキューなどをよしなに扱いながら実装するような耐障害性のあるワークフローを TypeScript の関数として記述できます。仕組み的には Next.js の `"use client"` や `"use server"` と同じで、バンドラーが注釈付けられた関数を特別扱いしてリトライや一時停止が可能なコードに変換し、それをタスクを永続化できる実行基盤^[vercel/workflow ではこの基盤のことを `world` と呼んでいる。ネーミングが不遜！]上で動かすイメージです。
まだベータ版ですが、面白い思想のライブラリなので、今後の行方をウォッチしていきたいと思います。

ちなみに弊社では AI ワークフローを動かす durable な実行基盤として [BullMQ](https://bullmq.io/) というライブラリを使っています。

## 懇親会

地方開催の良いところは、普段の生活圏では中々接点を持てない方々と話せるところと、ご飯が美味しいところです^[激アツ。]。金沢工業大学の学生の方や北陸・関西で活躍されている方々とお話できて楽しかったです。技術イベントの多くが東京で開催される中、こういう地方開催のイベントは貴重なので、今後も地方版 TSKaigi が続いていくといいな〜と思いました。

また、これは懇親会ではないですが、大きなカンファレンスではイベント前日・翌日に会場近くでサイドイベントが開催されていることがあります。今回、Findy さん主催の Drinkup に参加してきました。おすすめです。

https://x.com/pika_findy/status/1992201633959391369

## ガチその他

せっかく金沢に来たんだから観光しようということで21世紀美術館に行ってきました。この時やっていた展覧会がよかったのでおすすめです（2026-03-15 までやってるらしい）。

https://www.kanazawa21.jp/data_list.php?g=65&d=1828

金沢市街を TSKaigi Hokuriku のノベルティであるトートバッグを片手にぶらついてみました。内装や建築が凝っているところが多くて街並みが高級〜という気持ちになりました。東京で荒んだ心が浄化されるようです。

![コンクリート製のモニュメント。背景にはビルと青空。](/images/herp-tskaigi-hokuriku-2025/yuyo.jpg)
_金沢市制百周年記念事業モニュメント。共産主義国家にありそう。_

## 最後に

総じて非常に良いカンファレンスでした。皆さんにも是非参加してほしい。

ちなみに TSKaigi 2026 の開催も決まったらしいです（ヤッタ〜！）

![「TSKaigi 2026 開催決定！！2026/5/22-23（2日間） ベルサール羽田空港」と記載されたスライド。](/images/herp-tskaigi-hokuriku-2025/tskaigi-hokuriku-2026.jpg)

---

**宣伝コーナー**

最近、HERP は社内の標準技術スタックを TypeScript 中心に据える[意思決定](https://taketo957.hatenablog.com/entry/2025/09/02/125043)をしました。例えば、私も開発に携わっている [HERP AIリクルーター](https://lp.herp.cloud/ai-recruiter) は、Bun + React + Hono RPC + Prisma というモダン Full-Stack TypeScript スタックで開発されています。
これから TypeScript やっていき企業として活動していくためにも、こうしたカンファレンスには積極的に参加してキャッチアップしつつ、コミュニティに貢献できたらイイナーと思っております。

ひいては TypeScript やっていき人材や、自分も会社のお金で TSKaigi に参加したいぜ！という人を絶賛募集中です。ぜひ！！！（[インターン](https://herp.careers/v1/herpinc/-Z20OVrOz6m8) もあるよ）

https://herp.careers/v1/herpinc/9tYFbxcUPSEP?utm_source=zenn&utm_medium=article&utm_content=6xr7ti42z4

https://herp.careers/v1/herpinc/G5kBv3sKYPye?utm_source=zenn&utm_medium=article&utm_content=6xr7ti42z4
