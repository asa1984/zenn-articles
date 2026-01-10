---
title: "　§2. 組み込み関数"
---

## builtins

Nix言語ではグローバルな名前空間に`builtins`という、組み込み関数をまとめたAttrSetが定義されています。基本的に`builtins.<関数名>`という形で呼び出しますが、いくつかの組み込み関数は`builtins`と同様にグローバルな名前空間でも有効化されています。

本書で登場するNix式では、`import`/`throw`/`derivation`関数以外は明示的に`builtins.<関数名>`という形で記述します。

## 重要な組み込み関数

### import

```:importの型
import :: Path -> <式>
```

Nix言語でファイル分割を行うときに使用します。`import`関数はPathを受け取り、Pathが指すNixファイルのNix式を返します。

試しに`add.nix`を`main.nix`からインポートしてみます。

```:ディレクトリ構成
./
├── add.nix
└── main.nix
```

```nix :add.nix
{ a, b }: a + b
```

```nix :main.nix
let
  add = import ./add.nix;
in
add {
  a = 1;
  b = 2;
}

# 省略して以下の書き方をすることが多い
# import ./add.nix { a = 1; b = 2; }
```

```bash :ファイル分割されたNix式の評価
$ nix eval --file ./main.nix
3
```

ファイルインポートが特別な構文ではなく、組み込み関数として用意されているのが面白いですね（後のセクションで述べますが内部的には特別な関数です）。

### throw

```:throwの型
throw :: String -> 虚無
```

`throw`は例外を投げます。引数の文字列はエラーメッセージとして表示されます。

Nix言語にはtry-catchのようなエラーハンドリングの機能がなく、例外の発生は即ちNix言語の評価プロセス全体の終了を意味します。

### derivation

ビルドに関わる最重要関数です。[_1.4. Nix言語とderivation_](ch01-04-derivation)で詳しく解説します。

## よく使う組み込み関数

### readFile

```:readFileの型
readFile :: Path -> String
```

`readFile`はファイルの内容を文字列として読み取ります。

```bash :hello.txtを読み取る
$ echo "Hello, world!" > hello.txt

$ nix repl
nix-repl> builtins.readFile ./hello.txt
"Hello, world!"
```

### toString

```:readFileの型
toString :: 任意の型 -> string
```

`toString`は任意の値を文字列に変換する関数です。

#### プリミティブ型に適用すると…

```bash
# Number: 見た目通り
nix-repl> builtins.toString 1234
"1234"

# String: なにも変わらない
nix-repl> builtins.toString "Hello, world!"
"Hello, world!"

# Path: 絶対パスの文字列に変換
nix-repl> builtins.toString ./path/to/file
"/absolute/path/to/file"

# Boolean: trueなら"1"、falseなら空文字列
nix-repl> builtins.toString true
"1"

nil-repl> builtins.toString false
""

# Null: 空文字列
nix-repl> builtins.toString null
""
```

#### Listに適用すると…

```bash
# 文字列要素をスペース区切りで連結
nix-repl> builtins.toString ["Hello," "world!"]
"Hello, world!"

# 各要素にtoStringを適用して連結
nix-repl> builtins.toString [1 2 3]
"1 2 3"

# 異なる型の要素でも同じくtoStringを適用して連結
nix-repl> toString [1 true "String" null]
"1 1 String "

# 空リストは空文字列
nix-repl> builtins.toString []
""
```

#### AttrSetに適用すると…

基本的に`toString`をAttrSetに適用するとエラーになりますが、`__toString`または`outPath`というattributeを持っていた場合のみ例外的に文字列に変換できます。

`__toString`は関数でなければならず、引数にAttrSet自身を取ります。`toString`適用時、`toString`はこのAttrSet自体を`__toString`の引数に渡し、その返り値にまた`toString`を適用した結果を返します。

```nix
let
  attr = {
    someAttr = 1234;

    # toString適用時、selfにはattr自体が渡される
    __toString = self: someAttr;
  };
}
in
builtins.toString attr

# 評価結果: "1234"
```

`outPath`はPath型でなければならず、`toString`を適用するとそのPathを文字列に変換します。

```nix
builtins.toString {
  outPath = /path/to/something;
}

# 評価結果: "/path/to/something"
```

なぜ、このような挙動になっているのかは[_1.4. Nix言語とderivation_](ch01-04-derivation)で詳しく解説します。

### パース・シリアライズ系

いくつかのデータ形式に対応したパース・シリアライズ関数が組み込まれています。

- fromJSON
- fromTOML
- toJSON
- toXML

```:型
# パース
fromJSON :: String -> AttrSet
fromTOML :: String -> AttrSet

# シリアライズ
toJSON :: AttrSet -> string
toXML :: AttrSet -> string
```

## 非純粋な関数

builtinsのいくつかの関数は純粋ではありません。これらの関数は副作用を持ち参照透過でないため、Nixの再現性を損ってしまいます。そのため、次の章で解説するNix言語のプロジェクト管理機能Flakesは非純粋な関数の使用を制限しており、明示的に`--impure`オプションを付けて評価しないとエラーになります。REPLやFlakesで管理されていないNix式ではそのまま使用できます。

:::details 「非純粋」の定義
一部の読者は「前述の`readFile`なども非純粋なのでは？」と疑問に思ったかもしれません。一般的な純粋関数型言語ではあらゆるI/Oが副作用として明示的に扱われることを考えれば当然の疑問です。

Nix言語の純粋性は言語だけでなく、Nixストアやビルドシステムによって保証されています。ここは[_1.4. Nix言語とderivation_](ch01-04-derivation)に繋がる話なので詳しくは述べませんが、大域的には純粋に見えるようになっているのです。よって、本書での「非純粋」は「ビルドの再現性を損い得る」程度の意味で捉えた方がいいかもしれません。

筆者もこの表現をどうすべきか悩みましたが、最終的に公式リファレンスの「impure」という表現をそのまま使うことにしました。
:::

### currentSystem

```:currentSystemの型
currentSystem :: string
```

```bash :REPL
nix-repl> builtins.currentSystem
"x86_64-linux"
```

現在Nix式を評価しているシステムのアーキテクチャを文字列で返します。

### getEnv

```:getEnvの型
getEnv :: string -> string
```

```bash :REPL
nix-repl> builtins.getEnv "EDITOR"
"nvim"
```

`getEnv`は環境変数の値を取得します。環境変数が存在しない場合は空文字列を返します。Flakes管理下のNix式で`getEnv`を使い、`--impure`オプションを付けないで評価した場合は、常に空文字列を返します。

## Fetcher

Nix言語においてインターネットからリソースを取得する関数は**Fetcher**と呼ばれています。一見これも非純粋な関数に見えますが、Nixにはハッシュ関数を用いて冪等性を保ちながらインターネットアクセスを行う仕組みが用意されており、fetcherはそれを利用します。詳細は[_3.2. Fetcher_](ch03-02-fetchers)で解説します。

## その他の組み込み関数

文字列操作やリスト操作、型判定など、他にも多くの組み込み関数があります。以降、本書で新しい組み込み関数が登場したら都度解説します。

詳細は公式リファレンスを参照してください。

https://nix.dev/manual/nix/2.20/language/builtins

## 【余談】非純粋な関数で遊ぶ

`currentTime`という非純粋な組み込み関数があります。この関数は現在のUNIX時間を返します。

```:currentTimeの型
currentTime :: integer
```

```bash :REPL
nix-repl> builtins.currentTime
1722052322 # 日本時間で2024/07/27 12:52:02
```

もちろん、ビルドで現在時刻を取得するなど言語道断なので、こんな関数を使う機会は一切ありません。

……が、筆者がはりきって`currentTime`を利用した関数を実装してしまったのでよかったら見てやってください。

```:today関数
today :: Number -> { year, month, day }
```

:::details today.nix
うるう年を考慮してUNIX時間を年月日に変換する関数です。
Nix言語には剰余演算子がないので、`can_divide`という関数を自分で定義して割り切れるかどうかを判定しています。`builtins.elemAt`はListの指定したインデックスの要素を取得する関数です。

```nix :today.nix
# Number -> { year, month, day }
unix_time:
let
  base_year = 1970;
  seconds_in_day = 86400;
  total_days = unix_time / seconds_in_day;

  can_divide = dividend: divisor: (dividend / divisor) * divisor == dividend;

  is_leap_year =
    year:
    if can_divide year 400 then
      true
    else if can_divide year 100 then
      false
    else
      can_divide year 4;

  calc_year =
    { year, days }:
    let
      days_in_year = 365 + (if (is_leap_year year) then 1 else 0);
    in
    if days < days_in_year then
      { inherit year days; }
    else
      calc_year {
        year = year + 1;
        days = days - days_in_year;
      };

  calc_month =
    {
      year,
      month,
      days,
    }:
    let
      days_per_month = [
        31
        (if (is_leap_year year) then 29 else 28)
        31
        30
        31
        30
        31
        31
        30
        31
        30
        31
      ];
      days_in_month = builtins.elemAt days_per_month (month - 1);
    in
    if days < days_in_month then
      {
        inherit year month;
        days = days + 1;
      }
    else
      calc_month {
        inherit year;
        month = month + 1;
        days = days - days_in_month;
      };

  year_and_remaining_days = calc_year {
    year = base_year;
    days = total_days;
  };
  month_and_day = calc_month {
    year = year_and_remaining_days.year;
    month = 1;
    days = year_and_remaining_days.days;
  };
in
{
  year = year_and_remaining_days.year;
  month = month_and_day.month;
  day = month_and_day.days;
}
```

:::

`today`に9時間分の補正をかけたUNIX時間を渡すと、現在の年月日が返ってきます。

```bash :REPL
nix-repl> today = import ./today.nix

# 日本時間に直すために9時間分の補正をかける
nix-repl> jst_bias = 9 * 60 * 60

# 2024-08-16に評価した
nix-repl> today (builtins.currentTime + jst_bias)
{ day = 16; month = 8; year = 2024; }
```

汎用的な言語なら必須の関数ですが、Nix言語では無用の長物です。

このような実用性皆無のNix式の発展例として、Nix言語で無理矢理乱数生成を行うトンデモ関数を実装している方^[[figsoda](https://github.com/figsoda)氏: [fenix](https://github.com/nix-community/fenix)や[nurl](https://github.com/nix-community/nurl)といった非常に便利なNix関連のツールを作っている方です。]がいます。

https://github.com/figsoda/rand-nix

READMEにはこう記されています。

> _Impure, unreproducible, and indeterministic_
>
> 「_非純粋、再現不可能、不確定_」

**非純粋な関数、ダメ絶対^[あくまでNixのビルドの再現性を保証するためです。汎用的なプログラミング言語においてはその限りではありません。]^[次のセクションで解説するFlakesは非純粋な関数を機械的に制限します。]**
