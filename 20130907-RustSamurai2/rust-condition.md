% Rustにおける<br>エラー処理方法
% 三浦 雅弘
% Rust Samurai vol.2<br>2013-09-07
<style type="text/css">
// body { background-image: -webkit-linear-gradient(black,#202440,#505060); color:#f0f0f0; }
// body { background-color: #202440; color: #f0f0f0; }
// code { color: #ffff80; }
// code.url { color: #ffff80; }
.red { color: #ff0000; }
// .section titleslide slide level1 { color: #00ff00; }
.section.titleslide.slide.level1 {
    color: #000000; text-align: center;
	margin-top: 220px;
}
</style>
<meta name="copyright" content="| Rust Samurai vol.2 | 2013-09-07" />

## 自己紹介

三浦 雅弘  
gplus.to/Dubhead  
Twitter @Dubhead

## エラー処理って?

他言語では:

- 特殊な値を返す
    - 例: 何かの長さを返す関数が、エラーがあると-1を返す
- 例外
    - try, throw/raise, catch/except, finally
- オプション型
    - 例: ScalaのOption[A] 型、Some(a) または None
	- C++14 で採用予定

## 本日の発表内容

Rust公式サイトにある次の文書を概説する

Rust Condition and Error-handling Tutorial  
<http://static.rust-lang.org/doc/tutorial-conditions.html>

<br>

- Option型
- Result型
- Failure
- Condition

# 例題

## 例題 (1)

ファイルを読み込んで整形出力する

~~~~
$ cat numbers.txt
1 2
34 56
789 123
45 67
~~~~

~~~~
$ ./example numbers.txt
0001, 0002
0034, 0056
0789, 0123
0045, 0067
~~~~

## 例題 (2)

~~~~
extern mod extra;
use extra::fileinput::FileInput;
use std::int;

fn main() {
    let pairs = read_int_pairs();
    for &(a,b) in pairs.iter() {
        println(fmt!("%4.4d, %4.4d", a, b));
    }
}
~~~~

## 例題 (3)

~~~~
fn read_int_pairs() -> ~[(int,int)] {

    let mut pairs = ~[];

    let fi = FileInput::from_args();
    while ! fi.eof() {

        // 1. 1行読み込み
        let line = fi.read_line();

        // 2. 行をフィールド (word) へ分割
        let fields = line.word_iter().to_owned_vec();
~~~~

## 例題 (4)

~~~~
        // 3. フィールドのベクターにパターンマッチをかける
        match fields {

            // 4. この行にフィールドが2つあるなら…
            [a, b] => {

                // 5. その2つを整数としてパースしてみる
                match (int::from_str(a), int::from_str(b)) {

                    // 6. 両方ともパースできたら、プッシュする
                    (Some(a), Some(b)) => pairs.push((a,b)),

                    // 7. 整数でないフィールドは無視
                    _ => ()
                }
            }

            // 8. フィールド2つでない行は無視
            _ => ()
        }
    }

    pairs
}
~~~~

# Option型

## Option型

src/libstd/option.rs より抜粋:

~~~~
pub enum Option<T> {
    None,
    Some(T),
}
~~~~

つまり、以下のどちらかの値を取る enum である:

- None &mdash; 値がない
- Some(T) &mdash; 型Tの値がある

「型Tの値」はパターンマッチなどで取得する

例題では int::from_str が使われている  
これは Option&lt;int&gt; を返す  
つまり、文字列から整数へ変換できれば Some(int) を、できなければ None を返す

## Option型の利点

- APIがシンプル
- 効率が非常によい
- 呼び出し側が、エラーを自分で処理するかどうか決められる
- 呼び出し側に、エラーの可能性を認識させられる

## Option型の欠点

- 冗長 &mdash; パターンマッチや Option::unwrap が常に必要
    - unwrapすると値を取り出せるが、値がないとFailure (後述)
- 常に unwrap するという安直な対処で済ませがち
- 様々なエラーが None 1つにまとめられる
    - 呼び出し側でエラーごとに処理を切り替えたりできない
    - 呼び出し側で正確なエラーメッセージさえ出せない

# Result型

## Result型

<span class="red">std::result::Result&lt;T,E&gt;</span>

Option型と同様の enum で、Ok(T) または Err(E)

もし int::from_str を改良してエラー内容が分かるようにするなら:

~~~~
enum IntParseErr {
     EmptyInput,
     Overflow,
     BadChar(char)
}

fn int::from_str(&str) -> Result<int,IntParseErr> {
  // ...
}
~~~~

## Result型の利点と欠点

- エラー内容が、より詳細に伝わる
    - 呼び出し側でエラー処理や報告に使える情報が増える
- Option型の冗長さという問題はそのまま
    - 呼び出し側が自分でエラー処理したくない場合は、
      自分もResult型を返す必要がある

# Failure

## Failure (1)

タスクがfailする原因:

- 間接的な非同期イヴェントによる場合 (killシグナルなど)
- assert! に失敗した場合
- fail! マクロを呼んだ場合

<br>

タスクがfailすると:

- 通常の実行を中止
- 制御スタックの巻き戻しを行う
    - その間、ヒープに取得したリソースを解放し、デストラクタを実行する

## Failure (2)

- 他言語の例外機構に似ている
- タスクは必ず終了し、catch できない
- ただし子タスクの Failure を親タスクでトラップできる

<br>

Option 型の unwrap メソッドでの Failure 使用例:

~~~~
pub fn unwrap(self) -> T {
    match self {
      Some(x) => return x,
      None => fail!("option::unwrap `None`")
    }
}
~~~~

## Failure (3)

利点

- シンプルかつ簡潔
    - エラーが起きたら実行継続できない場合に向いている
- 子タスクのあらゆるエラーを親タスクでまとめてトラップできる
    - 可用性の高い大規模プログラムにはよくある造り

<br>

欠点

- タスクごと落ちるしかない

## Failure (4)

例題プログラムに不正な入力を食わせてみると

~~~~
$ cat bad.txt
1 2
34 56
ostrich
789 123
45 67
~~~~

不正な行を単に無視する

~~~~
$ ./example bad.txt
0001, 0002
0034, 0056
0789, 0123
0045, 0067
~~~~

## Failure (5)

例題プログラムを Failure を使って書き変えてみる

~~~~
fn main() {
    let result = do task::try {
        let pairs = read_int_pairs();
        // 〜〜〜略〜〜〜
    };
    if result.is_err() {
        println("parsing failed");
    }
}
~~~~

## Failure (6)

~~~~
fn read_int_pairs() -> ~[(int,int)] {
        // 〜〜〜略〜〜〜
        match fields {
            [a, b] => pairs.push((int::from_str(a).unwrap(),
                                  int::from_str(b).unwrap())),

            // 不正な入力行では明示的にfailする。
            _ => fail!()
        }
        // 〜〜〜略〜〜〜
}
~~~~

## Failure (7)

~~~~
$ ./example bad.txt
rust: task failed at 'explicit failure', ./example.rs:44
parsing failed
~~~~

<br>

read_int_pairs で起きたfailureをmainでトラップして、
実行を正常に継続することも可能

# Condition

## Condition (1)

Failureほど大雑把でなく、  
Option型やResult型ほど冗長でもない。  
その中間くらいの使い勝手を狙ったもの

特徴

- エラーの起きる場所とトラップする場所とが分かれている
- conditionのトラップに成功した場合は、
    <span class="red">エラーが起きた場所から</span>実行再開する
- トラップされなかった場合は Failure となる

## Condition (2): 宣言方法

Condition は condition! マクロを使って宣言する

Condition には、それぞれ名前と入出力の型がある

例題において、1行が2フィールドでなかった場合の例:  

~~~~
condition! {
    pub malformed_line : ~str -> (int,int);
}
~~~~

## Condition (3): 発生させる方法

condition! マクロを使うと、その名前でモジュールが作成される

このモジュールには cond というstaticな値がある

Conditionを発生させるには、cond の raise メソッドを使う

Conditionをトラップするには、cond の trap メソッドを使う

## Condition (4): 発生させる例

~~~~
fn read_int_pairs() -> ~[(int,int)] {
        // 〜〜〜略〜〜〜
        match fields {
            [a, b] => pairs.push((int::from_str(a).unwrap(),
                                  int::from_str(b).unwrap())),

            // 不正な入力行では、conditionハンドラを呼び出し、
            // それが返してきたものをとにかく pairs へ突っ込む
            _ => pairs.push(malformed_line::cond.raise(line.clone()))
        }
        // 〜〜〜略〜〜〜
}
~~~~

## Condition (5)

Conditionが発生すると、raise メソッドは  
そのタスクで最も内側にあるハンドラを探し、  
これを呼び出して入力値を渡す。

ハンドラが見つからない場合は、そのタスクがFailureとなる。

Failure となる例:

~~~~
$ ./example bad.txt
rust: task failed at 'Unhandled condition:
malformed_line: ~"ostrich"', .../libstd/condition.rs:43
~~~~

<!-- 「rust:」以降は実際は1行 -->

## Condition (6): トラップ方法

Conditionをトラップし、不正入力行を (-1, -1) で置き換える例

~~~~
fn main() {
    // conditionをトラップする:
    do malformed_line::cond.trap(|_| (-1,-1)).inside {

        // 保護されたロジック
        let pairs = read_int_pairs();
        // 〜〜〜略〜〜〜
    }
}
~~~~

## Condition (7)

~~~~
$ ./example bad.txt
0001, 0002
0034, 0056
-0001, -0001
0789, 0123
0045, 0067
~~~~

トラップの処理を入れた以外は元のままだが、  
不正入力行はハンドラが返す値で置き換えて、  
それ以降も処理を継続する。

# (時間の都合で) Condition まとめ

## Condition まとめ

- read_int_pairs のシグニチャは変えないまま、エラー処理の方針を Condition 発生側とトラップ側とで色々変えられる
- 発生箇所とトラップ箇所の間に関数呼び出しが何段か挟まる場合でも、エラー内容を上層へ伝える処理を毎回書かなくてよい
    - つまりエラー発生箇所と処理箇所とを分離できる
- トラップをネストすることにより、複数の Condition に対処できる

# これらのテクニックの使い分け

## これらのテクニックの使い分け

- <span class="red">Option か Result を返すべき場合</span>: エラーが頻繁に起きると予想され、
  直接の呼び出し元がすぐエラー処理する場合
    - これらの型は呼び出し元が処理せざるを得ない
	- 高性能 (enumのタグ用に1ワード増えるだけ)
	- エラーが1種類だけなら Option、
	  それ以外は enum を定義して Result&lt;T,FooErr&gt;
- <span class="red">Conditionを返すべき場合</span>: エラーは起きた場所で対処可能だが、
  どの対処方法がよいかは呼び出し側で決めたい場合
- <span class="red">Failureを返すべき場合</span>: エラーが起きた場所では対処できず、
  プログラム実行 (の大きな1かたまり) をばっさり止めるしか手がない場合

## これらのテクニックの使い分け (2)

(注)  

- ハンドルされない Condition は Failure になる
- このため、呼び出し側が「エラーを無視して実行継続したい」可能性があるなら、
  fail!() を直接使う代わりに Condition を使ってもほぼ問題ない

# 質疑応答

## ご清聴ありがとうございました

# 〜時間があったら使うシート〜

## 今日<span class="red">しない</span>話

リソース管理の話

- ファイルを必ずクローズしたい
- データベースを必ずコミットまたはロールバックしたい

<br>
他言語では

- デストラクタの活用
- 例外処理の finally 節
- Pythonのwith文
- Dのscope文、Goのdefer文

## Condition を改良する

さっきの例では不正入力行を (-1, -1) で置き換えたが、  
方針が変わり、無視することになったとする

＼ソフトウェア開発ではよくあること／

この場合、Condition の出力型を  
(int, int) から Option&lt;(int, int)&gt; に変更してみる

## Condition を改良する (2)

~~~~
// conditionのシグニチャを変え、Optionを返すよう変更
condition! {
    pub malformed_line : ~str -> Option<(int,int)>;
}

fn main() {
    // conditionをトラップし、Noneを返すよう変更
    do malformed_line::cond.trap(|_| None).inside {
        // 保護されたロジック
        let pairs = read_int_pairs();
        // 〜〜〜略〜〜〜
    }
}
~~~~

## Condition を改良する (3)

~~~~
fn read_int_pairs() -> ~[(int,int)] {
        // 〜〜〜略〜〜〜
        match fields {
            [a, b] => 〜〜〜略〜〜〜

            // 不正入力行ではconditionハンドラを呼び出す。
            // ハンドラが None を返してきたら行を無視し、
            // Some(pair) を返してきたら値をpushする。
            _ => {
                match malformed_line::cond.raise(line.clone()) {
                    Some(pair) => pairs.push(pair),
                    None => ()
                }
            }
        }
        // 〜〜〜略〜〜〜
}
~~~~

## Condition を改良する (4)

~~~~
$ ./example bad.txt
0001, 0002
0034, 0056
0789, 0123
0045, 0067
~~~~

プログラムはあまり変わってない  
(特に、read_int_pairs のシグニチャは元のまま)

不正入力行を無視するか、あるいは代わりの値を入れるかは、  
トラップする側で決められるようになった

## Condition を更に改良する

さっきの例では不正入力行を無視したが、  
方針が変わり、前の行の内容を使うことになったとする

＼ソフトウェア開発ではよくあること／

enum MalformedLineFix という型を導入する

## Condition を更に改良する (2)

~~~~
// conditionを扱う方針をトラップ側からエラー発生箇所側へ
// 伝えるため、新たな enum を導入する。
pub enum MalformedLineFix {
     UsePair(int,int),
     IgnoreLine,
     UsePreviousLine
}

// conditionのシグニチャを変更し、上記の enum を返すようにする。
// 注: conditionはモジュールなので super:: が必要。
condition! {
    pub malformed_line : ~str -> super::MalformedLineFix;
}
~~~~

## Condition を更に改良する (3)

~~~~
fn main() {
    // condition をトラップし、 UsePreviousLine を返す。
    do malformed_line::cond.trap(|_| UsePreviousLine).inside {

        // 保護されたロジック
        let pairs = read_int_pairs();
        // 〜〜〜略〜〜〜
    }
}
~~~~

## Condition を更に改良する (4)

~~~~
fn read_int_pairs() -> ~[(int,int)] {
        // 〜〜〜略〜〜〜
        match fields {
            [a, b] => 〜〜〜略〜〜〜

            // 不正入力行では condition ハンドラを呼び、
            // 返ってきた enum の値に合わせて対処する
            _ => {
                match malformed_line::cond.raise(line.clone()) {
                    UsePair(a,b) => pairs.push((a,b)),
                    IgnoreLine => (),
                    UsePreviousLine => {
                        let prev = pairs[pairs.len() - 1];
                        pairs.push(prev)
                    }
                }
            }
            // 〜〜〜略〜〜〜
}
~~~~

## Condition を更に改良する (5)

トラップ側で UsePreviousLine を使っているので、結果はこうなる:

~~~~
$ ./example bad.txt
0001, 0002
0034, 0056
0034, 0056
0789, 0123
0045, 0067
~~~~

## Condition を更に改良する (6)

この時点で例題プログラムは様々なエラー処理の仕組みを備えたが、  
read_int_pairs のエラーを処理する気のない者にとっては  
無視してよいものばかり

→ condition とは、そのための仕組みである

- 途中のcallerは、エラーを上層へ伝える処理を毎回書かなくてよい
- エラーを扱う方針が変わるたびにシグニチャを変えなくてよい

## 複数のconditionと途中のcaller

~~~~
$ cat bad.txt
1 2
34 56
7 marmot
789 123
45 67
~~~~

~~~~
$ ./example bad.txt
task <unnamed> failed at 'called `Option::unwrap()`
on a `None` value', .../libstd/option.rs:314
~~~~

「フィールドの数は合ってるけど整数じゃない」場合に対処してみる

## 複数のconditionと途中のcaller (2)

~~~~
pub enum MalformedLineFix {
     UsePair(int,int),
     IgnoreLine,
     UsePreviousLine
}

condition! {
    pub malformed_line : ~str -> ::MalformedLineFix;
}

// 2つ目のconditionを導入する。
condition! {
    pub malformed_int : ~str -> int;
}
~~~~

## 複数のconditionと途中のcaller (3)

~~~~
fn main() {
    // malformed_int のconditionをトラップし、-1を返す。
    do malformed_int::cond.trap(|_| -1).inside {

        // malformed_line のconditionをトラップし、UsePreviousLine を返す。
        do malformed_line::cond.trap(|_| UsePreviousLine).inside {

            // 保護されたロジック
            let pairs = read_int_pairs();
            // 〜〜〜略〜〜〜
        }
    }
}
~~~~

## 複数のconditionと途中のcaller (4)

~~~~
// 整数をパースする。パースが失敗したらconditionハンドラを
// 呼び出し、それが返してきたものを返す。
fn parse_int(x: &str) -> int {
    match int::from_str(x) {
        Some(v) => v,
        None => malformed_int::cond.raise(x.to_owned())
    }
}
~~~~

## 複数のconditionと途中のcaller (5)

~~~~
fn read_int_pairs() -> ~[(int,int)] {
        // 〜〜〜略〜〜〜
        match fields {
            // 整数のパースを補助関数に任せる。
            // これはパースエラーが起きると malformed_int を呼び出す。
            [a, b] => pairs.push((parse_int(a), parse_int(b))),

            _ => {
                match malformed_line::cond.raise(line.clone()) {
                    // 〜〜〜略〜〜〜
                }
            }
        }
    // 〜〜〜略〜〜〜
}
~~~~

## 複数のconditionと途中のcaller (6)

- read_int_pairs のシグニチャは元のまま
- malformed_line の発生やトラップの仕組みは元のまま
- 「フィールドの数は合ってるが整数じゃない」場合に対処できるようになった

~~~~
$ ./example bad.txt
0001, 0002
0034, 0056
0007, -0001
0789, 0123
0045, 0067
~~~~

## 複数のconditionと途中のcaller (7)

最後の例題プログラムから分かること:

- トラップをネストすることにより、複数のconditionに対処できる
- トラップ側とraise側との間に関数呼び出しが何段か挟まっていてもよい
    - つまりエラー発生箇所と処理箇所を分離できている
- intライブラリの設計方針に、呼び出し側が影響されない
    - int::from_str は Option&lt;int&gt; を返す設計になっているが、
	  このプログラムでは None が返ってきたら condition になる

<!--
Local Variables:
mode: markdown
End:
-->
