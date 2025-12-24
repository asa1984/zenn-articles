---
title: "Nix言語"
---

Nixのコンセプトの一つ「宣言的」を実現する要素がNixのDSL^[ドメイン固有言語。特定の問題に特化したプログラミング言語（SQL: データベース問い合わせのためのDSL）]である**Nix言語**です。

Nix言語でパッケージを定義しstore derivationを生成すれば、あとはNixが自動的に依存関係を解決し、ビルドを実行してくれます。宣言的ビルドにおいて、必要なパッケージがインストールされているか、ビルド手順が正しいか、などの心配は必要ありません。

ビルドシステムの強い制約やDerivationのような独自概念があるにも関わらず、Nixが既存のソフトウェアのビルドをラップできる理由はNix言語の柔軟性にあります。パッケージのビルドをプログラムとして記述できるため、拡張性や再利用性が高いのです。

本章ではその役割と特徴について解説します。

## Nix言語の役割

前章でstore derivationはNix言語から生成されると述べましたが、Nix言語のコンパイルターゲットがstore derivationというわけではありません。DerivationはNix言語におけるデータの1つに過ぎず、Nix言語自体の用途はもう少し汎用的です。具体的には以下のような用途があります。

1. パッケージ定義
2. 開発環境の定義
3. OS環境の設定（NixOS）
4. その他（サードパーティ）

注意しなければならないのが、Nix言語自体は何もしないということです。Nix式が返すのはただのデータであり、実際に何かを行うのはNix式が返した値を利用するプログラムです。例えば、`nix build`コマンドはNix言語で定義したパッケージビルドしますが、`nix develop`はNix言語で定義した開発環境を起動します。

## Instantiation^[[11. Glossary - Nix Reference Manual](https://nixos.org/manual/nix/stable/glossary.html?highlight=instant#gloss-instantiate)]

Nix式からstore derivationを生成することを**Instantiation**（インスタンス化）と言います。Nix式をinstantiateするとき、NixはNix式がDerivationを返すことを期待します。DerivationはNix言語のビルトイン関数`derivation`を使って生成します。

### Nix言語によるビルドの流れ

1. Nix式を評価
2. Nix式が返すDerivationからstore derivationを生成（Instantiation）
3. store derivationからストアオブジェクトをビルド（Realisation）

## Nix言語の特徴

- 純粋関数型言語
- 遅延評価
- ドメイン特化の機能
- 動的型付け

Nix言語は純粋関数型言語ですが、モナドといった難しい概念は存在しないので安心してください。DSLなのでNixストアとDerivationに関連したビルトイン関数が用意されています。

## データ型

- プリミティブ型
  - 文字列
  - 数値
  - **パス**
  - 論理値
  - Null
- リスト
- **Attribute Set**

### プリミティブ

#### 文字列

```nix
"string"
```

`''`で複数行の文字列を記述できます。

```nix
''
string
string
string
''
```

`${}`で変数の埋め込みができます。

```nix
# x = 1;
"x is ${x}" # -> x is 1
```

#### 数値

整数

```nix
123
```

小数

```nix
3.14
```

#### パス

Nix言語はファイルパスを文字列ではなく1つのデータ型としてサポートしています。

`./`から始まるパスは、このパスが記述されたNix言語ファイルからの相対パスです。

```
./example.txt
```

UNIXのファイルパスと同じように扱えます。

```
../example.txt
```

#### 論理値

```nix
true
```

```nix
false
```

#### Null

```
null
```

### リスト

リストは要素をスペースで区切ります。

```
[ "foo" "bar" "baz" ]
```

```
[ 1 2 3]
```

### Attribute Set

**Attribute Set**（略称: **AttrSet**）は、直訳すると「属性の集合」を意味し、名前と値のペア（attribute）の集合です。他のプログラミング言語における構造体やオブジェクト型に近いデータ型です。

名前と値がイコールで結ばれ、各attributeはセミコロンで閉じる必要があります。

```nix
{
  x = 1;
  y = 2;
}
```

recursiveを意味する`rec`キーワードを付けるとAttrSet内の値を参照できます。

```nix
rec {
  x = 1;
  y = x;
}
```

`.`でフィールドにアクセスできます。

```nix
# a = { x = 1; y = 2; };
a.x # -> 1
```

## 関数

Nix言語では1ファイルが1つの関数になっている必要があります。

引数と返り値をコロンで区切ります。

```nix
引数: 返り値
```

```nix
# number = 1だった場合、返り値は2
number: number + 1
```

関数を評価するには関数名にスペースを空けて引数を指定すればよいです。

```nix
# add = number: number + 1
add 1 # -> 2
```

### 分割代入

```nix
# args = { x = 1; y = 2; }だった場合、返り値は{ x = 2; y = 1; }
args: {
  x = args.y;
  y = args.x;
}
```

引数がAttrSetだった場合は分割代入ができます。

```nix
# args = { x = 1; y = 2; }だった場合、返り値は{ x = 2; y = 1; }
{ x, y }: {
  x = y;
  y = x;
}
```

引数に与えられたAttrSetの内、一部のattributeだけを使いたい時は、`...`キーワードで無視できます。

```nix
# 引数が{ x = 1; y = 2; }だった場合、返り値は{ y = 1; }
{ x, ... }: {
  y = x;
}
```

### let-in構文

let-in構文を使うことで関数内で変数を宣言できます。関数型言語ではよくある構文です。

```nix
# 引数が{ x = 1; y = 2; }だった場合、返り値は{ a = 2; b = 3; }
{ x, y }: let
  add = number: number + 1;
  a = add x;
  b = add y;
in {
  a = a;
  b = b;
}
```

### 定数

引数がない場合は定数ファイルになります。

```nix
# 常に123を返す
123
```

```nix
# 常に{ x = 1; y = 2; }を返す
{
  x = 1;
  y = 2;
}
```

## ビルトイン関数

ビルトイン関数は大量にあるのでいくつか抜粋して紹介します。全てのビルトイン関数の説明は以下のドキュメントに記載されています。

https://nixos.org/manual/nix/stable/language/builtins#builtins-fetchurl

### readFile

`readFile`はファイルを文字列として読み込む関数です。引数にはパス型を指定します。

```nix
readFile <パス>
```

一方、`writeFile`のような関数は存在しません。Nix言語のビルトイン関数は基本的に読み取り専用であり、書き込み系の関数は存在しません^[正確には`derivation`関数がNixストアへの書き込みを発生させる。]。

### fetchurl

`fetchurl`は最も原始的なFetcherです。

```nix
fetchurl {
  url = "<URL>";
  sha256 = "<ダウンロード予定のコンテンツのSHA256ハッシュ>";
}
```

### fromJSON/fromTOML/toJSON/toXML

よく使うデータフォーマットについて、文字列からAttribute SetまたはAttribute Setから文字列への変換を行う関数が用意されています。

## import関数

`import`関数は外部のNixファイルの関数を呼び出すビルトイン関数です。

以下のようなファイル構成があったとします。

```
./
├─main.nix
└─sub/
  ├─default.nix
  └─imported.nix
```

`imported.nix`が以下のような関数だったとして、`main.nix`から`imported.nix`を呼び出すことを考えます。

```nix :imported.nix
{ x, y }: {
  a = x;
  b = y;
}
```

`import`の引数にパスを与えてインポートします。

```nix :main.nix
let
  f = import ./sub/imported.nix;
in
f { x = 1; y = 2; } # -> { a = 1; b = 2; }
```

以下は上のコードと等価です。

```nix :main.nix
import ./sub/imported.nix { x = 1; y = 2; }; # -> { a = 1; b = 2; }
```

### default.nix

`default.nix`という特別なファイルがあります。

`imported.nix`を`default.nix`としてコピーします。

```
./
├─main.nix
└─sub/
  ├─default.nix
  └─imported.nix
```

`import`でディレクトリのパスを指定すると、`import`はそのディレクトリに含まれる`default.nix`をインポートします。

```nix :main.nix
import ./sub { x = 1; y = 2; }; # -> { a = 1; b = 2; }
```

ちょうどNode.jsの`index.js`やRustの`mod.rs`と同じ使い方ができます。

## derivation関数

`derivation`関数はNix言語の最も重要なビルトイン関数です。この関数はAttribute Setを返し、副作用としてstore derivationを生成します。副作用を持つためNix言語の純粋性が損なわれるように思えますが、NixストアとNix言語によって内部動作が巧妙に隠蔽されているため、見かけ上は純粋に保たれています。

```nix
derivation {
  name = "<パッケージ名>";
  system = "<システムアーキテクチャ>";
  builder = "<ビルドを実行する実行可能ファイルのパス>";
}
```

`derivation`はプリミティブな関数なので、実際にパッケージをビルドする時は次章で解説するNixpkgsから提供されているライブラリを利用します。

## Nix言語とNixストアの相互作用

実は、`import`や`readFile`などの外部のファイルを読み取る関数はDerivationと密接に関係しています。これらの関数はパスで指定したファイルを直接読み込んでいるわけではなく、ストアパスを読み込んでいます。どういうことかというと対象のファイルを内部でストアオブジェクトとしてNixストアにrealiseさせているのです。

これ以上は込み入った話になるので、詳しく知りたい方は以下のドキュメントを参照してください。

https://nixos.org/manual/nix/stable/language/import-from-derivation
