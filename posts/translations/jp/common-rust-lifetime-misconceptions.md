# Rustのライフタイムについてのよくある誤解

2020年5月19日 · 読了時間 33分 · #rust · #lifetimes

## 目次

- [はじめに](#はじめに)
- [誤解](#誤解)
  - [1) `T` は所有型のみ取りうる](#1-Tは所有型のみ取りうる)
  - [2) `T: 'static`ならば`T`はプログラム全体で有効でなければならない](#2-if-t-static-then-t-must-be-valid-for-the-entire-program)
  - [3) `&'a T`と`T: 'a`は同じ](#3-a-t-and-t-a-are-the-same-thing)
  - [4) 自分のコードはジェネリックではなくライフタイムを持たない](#4-my-code-isnt-generic-and-doesnt-have-lifetimes)
  - [5) コンパイルされたならライフタイムの記述は正しい](#5-if-it-compiles-then-my-lifetime-annotations-are-correct)
  - [6) Boxトレイトオブジェクトはライフタイムを持たない](#6-boxed-trait-objects-dont-have-lifetimes)
  - [7) コンパイラのエラーメッセージはプログラムの直し方を教えてくれる](#7-compiler-error-messages-will-tell-me-how-to-fix-my-program)
  - [8) ライフタイムは実行時に伸び縮みする](#8-lifetimes-can-grow-and-shrink-at-run-time)
  - [9) 可変参照から共有参照へ降格することは安全](#9-downgrading-mut-refs-to-shared-refs-is-safe)
  - [10) クロージャは関数と同じライフタイム省略ルールに従う](#10-closures-follow-the-same-lifetime-elision-rules-as-functions)
  - [11) `'static`な参照は`'a`な参照になることを常に強制される](#11-static-refs-can-always-be-coerced-into-a-refs)
- [まとめ](#まとめ)
- [ディスカッション](#ディスカッション)
- [お知らせ](#お知らせ)
- [参考](#参考)

## はじめに

私は何らかのタイミングでこのような誤解をしたことがあり、今日多くの初心者がこうした誤解に苦戦しています。私が使っている用語の中には普通ではないものもあるかもしれませんので、ここでは私が使っている略語とその意味を表にしてみました。

| 用語 | 意味 |
|-|-|
| `T` | 1) 成りうる型を要素として持つ集合 _または_ <br> 2) その集合の要素となる型 |
| 所有型 | 何らかの参照ではない型 <br>例) `i32`, `String`, `Vec`, など |
| 1) 借用型 _または_<br>2) 参照型 | 可変性を問わない何らかの参照型 <br>例) `&i32`, `&mut i32`, など |
| 1) 可変参照 _または_<br>2) 排他的参照 | 排他的で可変性のある参照、つまり `&mut T` |
| 1) 不可変参照 _または_<br>2) 共有参照 | 不可変で共有された参照、つまり `&T` |

## 誤解

結局のところ、変数のライフタイムとはその変数が指すデータが現在のメモリアドレスで有効であることをコンパイラが静的に検証することができる期間です。以降では、どこで困惑してしまうのかについて詳しく述べていこうと思います。

### 1) `T` は所有型のみ取りうる

この誤解はライフタイムに関するというよりジェネリクスについての話ですが、ジェネリクスとライフタイムはRustでは密接に絡み合ったもので、なので一方について言及せず他方について言及することは不可能なのです。

私がRustを学び始めたとき、`i32`と`&i32`、`&mut i32`は異なる型だと知りました。また、あるジェネリックな型変数`T`は可能性のあるあらゆる型を内包した集合を表現していると知りました。しかし、これらを別々に理解しているとはいえ、これらを一緒に理解するのは不可能でした。Rust初心者だった私の頭の中ではジェネリクスはこのようなものだと理解していました。

| | | | |
|-|-|-|-|
| **型変数** | `T` | `&T` | `&mut T` |
| **例** | `i32` | `&i32` | `&mut i32` |

`T`はあらゆる所有型を持つ。`&T`はあらゆる不可変な借用型を持つ。`&mut T`はあらゆる可変な借用型を持つ。`T`と`&T`、`&mut T`は非連続な有限集合なのです。素晴らしい、シンプルで綺麗、簡単で直感的、そして完全に間違っていた。Rustではジェネリクスが実際にこのように動いているのだ;

| | | | |
|-|-|-|-|
| **型変数** | `T` | `&T` | `&mut T` |
| **例** | `i32`, `&i32`, `&mut i32`, `&&i32`, `&mut &mut i32`, ... | `&i32`, `&&i32`, `&&mut i32`, ... | `&mut i32`, `&mut &mut i32`, `&mut &i32`, ... |

`T`と`&T`、`&mut T`は全て無限集合で、それゆえに無限に型を借用できるのです。`T`は`&T`と`&mut T`の上位集合で、`&T`と`&mut T`は部分集合なのです。こうした概念を検証するためにいくつか例を見てみましょう。

```rust
trait Trait {}

impl<T> Trait for T {}

impl<T> Trait for &T {} // コンパイルエラー

impl<T> Trait for &mut T {} // コンパイルエラー
```

上記のプログラムは期待した通りにはコンパイルされません。

```rust
error[E0119]: conflicting implementations of trait `Trait` for type `&_`:
 --> src/lib.rs:5:1
  |
3 | impl<T> Trait for T {}
  | ------------------- first implementation here
4 |
5 | impl<T> Trait for &T {}
  | ^^^^^^^^^^^^^^^^^^^^ conflicting implementation for `&_`

error[E0119]: conflicting implementations of trait `Trait` for type `&mut _`:
 --> src/lib.rs:7:1
  |
3 | impl<T> Trait for T {}
  | ------------------- first implementation here
...
7 | impl<T> Trait for &mut T {}
  | ^^^^^^^^^^^^^^^^^^^^^^^^ conflicting implementation for `&mut _`
```

コンパイラは`&T`や`&mut T`に対しての`Trait`を実装させません。これはすでに`&T`や`&mut T`などを含んだ`T`への`Trait`の実装と衝突が起こるかもしれないからです。以下のプログラムは期待した通りにコンパイルされます。これは`&T`と`&mut T`は互いに独立だからです。

```rust
trait Trait {}

impl<T> Trait for &T {} // コンパイルされる

impl<T> Trait for &mut T {} // コンパイルされる
```

**キーポイント**

- `T`は`&T`と`&mut T`の上位集合
- `&T`と`&mut T`は互いに独立


### 2) `T: 'static`ならば`T`はプログラム全体で有効でなければならない

**誤解の流れ**

- `T: 'static`は _"`T`はライフタイム`'static`を持っている"_ と読まれるべき。
- `&'static T` と `T: 'static` は同じもの
- `T: 'static` ならば `T` は不可変でなければならない
- `T: 'static` ならば `T` はコンパイル時にのみ作られる

Rust初心者の多くは初めてライフタイム`'static`を学ぶ際、このような感じのコードの例から入ります。

```rust
fn main() {
    let str_literal: &'static str = "str literal";
}
```

`"str literal"`はコンパイルされたバイナリにハードコーディングされ、実行時には読み込み専用のメモリへ積まれ、そうしてプログラム全体で不可変で有効で、それが`'static`だと教わる。こうした概念は、キーワード`static`を用いて定義される`static`変数に関するルールによってさらに強まります。

```rust
static BYTES: [u8; 3] = [1, 2, 3];
static mut MUT_BYTES: [u8; 3] = [1, 2, 3];

fn main() {
   MUT_BYTES[0] = 99; // コンパイルエラー、static変数を変更するのはunsafe

    unsafe {
        MUT_BYTES[0] = 99;
        assert_eq!(99, MUT_BYTES[0]);
    }
}
```

`static`変数について

- コンパイル時にのみ作られる
- 不可変であるべきで、変更するのはunsafe
- プログラム全体で有効

ライフタイム`'static`はひょっとしたら`static`変数のデフォルトのライフタイムにちなんで名付けられたのではないでしょうか？なのでライフタイム`'static`は全く同じルールに従う必要があるのではないでしょうか？

ええ、しかしライフタイム`'static`のある型はライフタイム`'static`で _境界付けられた_ 型とは異なるのです。後者は実行時に動的に確保され、安全に、かつ自由に変化させられ、ドロップでき、意図した通りの期間で有効なのです。

ここで`&'static T`と`T: 'static`を区別しておくことは大切です。

`&'static T`はプログラムの最後までを含んだ限りなく長い間安全に保持される`T`への不可変な参照なのです。これは`T`自身が不可変で _参照が作られた後に_ moveしない場合にのみ可能です。`T`はコンパイル時に作られる必要はないのです。メモリがリークするコストの元で、実行時にランダムに動的に確保されたデータを作り、それへの`'static`な参照を返すことが可能です。例えば

```rust
use rand;

// 実行時にランダムな'staticなstrへの参照を作成
fn rand_str_generator() -> &'static str {
    let rand_string = rand::random::<u64>().to_string();
    Box::leak(rand_string.into_boxed_str())
}
```

`T: 'static`はプログラム終了までを含んだ限りなく長い間で安全に保持される`T`なのです。`T: 'static`は`&'static T`を含みますが、同時に`String`や`Vec`などの所有型も含むのです。データの所有者は、データを保持している限りデータが無効になることがないことが保証されているので、プログラム終了までを含めて無期限にデータを保持することができ、安全にデータを保持することができます。`T: 'static`は _"`T`はライフタイム`'static`によって境界付けられている"_ と読むべきなのです。以下にこうした概念を説明するためのプログラムを示します。

```rust
use rand;

fn drop_static<T: 'static>(t: T) {
    std::mem::drop(t);
}

fn main() {
    let mut strings: Vec<String> = Vec::new();
    for _ in 0..10 {
        if rand::random() {
            // 文字列はランダムに生成され実行時に動的に確保される
            let string = rand::random::<u64>().to_string();
            strings.push(string);
        }
    }

    // 文字列は所有型なので'staticで境界付けられる
    for mut string in strings {
        // 文字列は可変
        string.push_str("a mutation");
        // 文字列はドロップ可能
        drop_static(string); // コンパイル可能
    }

    // 文字列はプログラム終了前に無効になっている
    println!("i am the end of the program");
}
```

**キーポイント**

- `T: 'static`は _"`T`はライフタイム`'static`で境界づけられている"_ と読むべき
- `T: 'static`ならば`T`はライフタイム`'static` _または_ 所有型を持った借用型でありうる。
- `T: 'static`は所有型を含み、これの意味するところとして`T`は
  - 実行時に動的に確保される
  - プログラム全体で有効である必要はない
  - 安全かつ自由に変更可能
  - 実行時に動的にドロップされる
  - 異なる期間のライフタイムを持つ

### 3) `&'a T`と`T: 'a`は同じ

この誤解は一つ前のものの一般化されたものです。

`&'a T`は`T: 'a`を要求し、そして示唆しています。これは`T`自身が`'a`に対して有効でない場合ライフタイム`'a`の`T`への参照は有効となり得ないからです。例えば`Ref`は`'a`に対してのみ有効である場合それへの`'static`な参照を作ることができないので、Rustコンパイラは`&'static Ref<'a, T>`型を許さないでしょう。

`T: 'a`は`&'a T`を含みますがその逆は成り立ちません。

```rust
// 'aによって境界づけられた参照型のみ取る
fn t_ref<'a, T: 'a>(t: &'a T) {}

// 'aによって境界づけられた任意の型を取る
fn t_bound<'a, T: 'a>(t: T) {}

// 参照を含んだ所有型
struct Ref<'a, T: 'a>(&'a T);

fn main() {
    let string = String::from("string");

    t_bound(&string); // コンパイル可能
    t_bound(Ref(&string)); // コンパイル可能
    t_bound(&Ref(&string)); // コンパイル可能

    t_ref(&string); // コンパイル可能
    t_ref(Ref(&string)); // コンパイルエラー, expected ref, found struct
    t_ref(&Ref(&string)); // コンパイル可能

    // 文字列の変数は'aで境界づけられた'staticで境界づけられている
    t_bound(string); // コンパイル可能
}
```

**キーポイント**

- `T: 'a`は`&'a T`よりも一般的で柔軟
- `T: 'a`は所有型と参照を含んだ所有型、参照を受け付ける
- `&'a T`は参照のみ受け付ける
- 任意の`'a`に対して`'static` >= `'a`なので`T: 'static`ならば`T: 'a`

### 4) 自分のコードはジェネリックではなくライフタイムを持たない

**誤解の流れ**

- ジェネリクスとライフタイムを使わないことは可能

この心地よい誤解はRustのライフタイムの省略ルールのおかげで起きていて、Rustの借用チェックが以下の規則を推論しているため関数でライフタイムの記述を省くことができるのです。
- 関数への全ての引数の参照は別々のライフタイムを得る
- 引数のライフタイムが正確に1つであれば、全ての返り値の参照に適用される
- 複数の引数のライフタイムがあるがそれらのうち1つが`&self`か`&mut self`である場合は`self`のライフタイムは全ての返り値の参照に適用される
- それ以外の場合は返り値のライフタイムは明示的でなければならない

たくさん出てきたのでいくつか例を見てみましょう。

```rust
// 省略された記法
fn print(s: &str);

// 元々の記法
fn print<'a>(s: &'a str);

// 省略された記法
fn trim(s: &str) -> &str;

// 元々の記法
fn trim<'a>(s: &'a str) -> &'a str;

// エラー、引数がないため返り値のライフタイムが決定できない
fn get_str() -> &str;

// 明示的な記法
fn get_str<'a>() -> &'a str; // ジェネリックなバージョン
fn get_str() -> &'static str; // 'staticなバージョン

// エラー、複数の引数があるので返り値のライフタイムが決定できない
fn overlap(s: &str, t: &str) -> &str;

// 元々の記法(部分的には省略記法)
fn overlap<'a>(s: &'a str, t: &str) -> &'a str; // 返り値はsより長持ちしない
fn overlap<'a>(s: &str, t: &'a str) -> &'a str; // 返り値はtより長持ちしない
fn overlap<'a>(s: &'a str, t: &'a str) -> &'a str; // 返り値はsとtより長持ちしない
fn overlap(s: &str, t: &str) -> &'static str; // // 返り値はsとtより長持ちしない
fn overlap<'a>(s: &str, t: &str) -> &'a str; // 引数と返り値のライフタイムとは関係ない

// 元々の記法
fn overlap<'a, 'b>(s: &'a str, t: &'b str) -> &'a str;
fn overlap<'a, 'b>(s: &'a str, t: &'b str) -> &'b str;
fn overlap<'a>(s: &'a str, t: &'a str) -> &'a str;
fn overlap<'a, 'b>(s: &'a str, t: &'b str) -> &'static str;
fn overlap<'a, 'b, 'c>(s: &'a str, t: &'b str) -> &'c str;

// 省略された記法
fn compare(&self, s: &str) -> &str;

// 元々の記法
fn compare<'a, 'b>(&'a self, &'b str) -> &'a str;
```

もし以下のものを書いたことがあるなら、ジェネリックなライフタイムの省略がなされています。

- 構造体
- 参照をとる関数
- 参照を返す関数
- ジェネリックな関数
- トレイトオブジェクト (これについては後ほど)
- クロージャ (これについては後ほど)

**キーポイント**

- Rustのコードのほとんどはジェネリックなコードで、どこにでも省略されたライフタイムが存在する

### 5) コンパイルされたならライフタイムの記述は正しい

**誤解の流れ**

- Rustの関数へのライフタイムの省略ルールは常に正しい
- Rustの借用チェックは技術的に、そして _意味的に_ 常に正しい
- Rustはプログラマー以上にプログラムの意味について把握している

Rustのプログラムは技術的にコンパイル可能でも意味論的には間違っていることはありえます。この例を見てみましょう。

```rust
struct ByteIter<'a> {
    remainder: &'a [u8]
}

impl<'a> ByteIter<'a> {
    fn next(&mut self) -> Option<&u8> {
        if self.remainder.is_empty() {
            None
        } else {
            let byte = &self.remainder[0];
            self.remainder = &self.remainder[1..];
            Some(byte)
        }
    }
}

fn main() {
    let mut bytes = ByteIter { remainder: b"1" };
    assert_eq!(Some(&b'1'), bytes.next());
    assert_eq!(None, bytes.next());
}
```

`ByteIter`はバイトのスライスを繰り返すイテレータです。ここでは簡潔さのために`Iterator`トレイトの実装を省いています。これはちゃんと動くようですが、一度に何バイトかチェックしたいときはどうなるでしょう？

```rust
fn main() {
    let mut bytes = ByteIter { remainder: b"1123" };
    let byte_1 = bytes.next();
    let byte_2 = bytes.next();
    if byte_1 == byte_2 {
        // 何かしらの処理
    }
}
```

おやおや、コンパイルエラーです。

```rust
error[E0499]: cannot borrow `bytes` as mutable more than once at a time
  --> src/main.rs:20:18
   |
19 |     let byte_1 = bytes.next();
   |                  ----- first mutable borrow occurs here
20 |     let byte_2 = bytes.next();
   |                  ^^^^^ second mutable borrow occurs here
21 |     if byte_1 == byte_2 {
   |        ------ first borrow later used here
```

それぞれのバイトをコピーすることは可能っぽいです。バイト列の処理をするときはコピーは大丈夫ですが、`ByteIter`を`&'a [T]`を繰り返すことができるジェネリックなスライスのイテレータにしたとき、それを将来的にはコピーやクローンをするのが高コストだったり不可能な型と一緒に使いたくなるでしょう。ええ、これについてできることは無さそうです。コードはコンパイルされライフタイムの記述は正しいでしょう。ね？

いいえ、実は現在のライフタイムの記述はバグの原因なのです。そのバグとなりがちなライフタイムの記述は省略されるため、特に見分けがつきにくいのです。それでは省略されたライフタイムを広げてみて問題の原因を明確にしてみましょう。

```rust
struct ByteIter<'a> {
    remainder: &'a [u8]
}

impl<'a> ByteIter<'a> {
    fn next<'b>(&'b mut self) -> Option<&'b u8> {
        if self.remainder.is_empty() {
            None
        } else {
            let byte = &self.remainder[0];
            self.remainder = &self.remainder[1..];
            Some(byte)
        }
    }
}
```

全く役に立ちそうにないです。私はまだ混乱してます。これはRustのプロだけが知っているコツなのですが、ライフタイムの記述を説明的な名前にしてみましょう。それではもう一度見てみます。

```rust
struct ByteIter<'remainder> {
    remainder: &'remainder [u8]
}

impl<'remainder> ByteIter<'remainder> {
    fn next<'mut_self>(&'mut_self mut self) -> Option<&'mut_self u8> {
        if self.remainder.is_empty() {
            None
        } else {
            let byte = &self.remainder[0];
            self.remainder = &self.remainder[1..];
            Some(byte)
        }
    }
}
```

それぞれ返り値のバイトは`'mut_self`と注釈づけられていますが、バイトは明らかに`'remainder`から来ています！それでは直してみましょう。

```rust
struct ByteIter<'remainder> {
    remainder: &'remainder [u8]
}

impl<'remainder> ByteIter<'remainder> {
    fn next(&mut self) -> Option<&'remainder u8> {
        if self.remainder.is_empty() {
            None
        } else {
            let byte = &self.remainder[0];
            self.remainder = &self.remainder[1..];
            Some(byte)
        }
    }
}

fn main() {
    let mut bytes = ByteIter { remainder: b"1123" };
    let byte_1 = bytes.next();
    let byte_2 = bytes.next();
    std::mem::drop(bytes); // イテレータをドロップすることができます！
    if byte_1 == byte_2 { // コンパイル可能
        // 何らかの処理
    }
}
```

今こうして前のプログラムを見返してみると、明らかに間違っていましたね。どうしてRustはコンパイルできたんでしょう？その答えはシンプルで、メモリ安全だったからです。

Rustの借用チェッカはメモリ安全かどうかを静的に検証できる範囲でのみプログラム内のライフタイム注釈を見ています。Rustは幸いなことに、ライフタイム注釈が意味的に間違っていてもコンパイル可能ですが、その結果プログラムは不必要に制限されるのです。

ここで先ほどの例とは反対の簡単な例を見てみましょう。Rustのライフタイム省略ルールはこの場では意味的に正しくても、意図せず不要な明示的ライフタイム注釈を使って制限的なメソッドを書いているのです。

```rust
#[derive(Debug)]
struct NumRef<'a>(&'a i32);

impl<'a> NumRef<'a> {
    // この構造体は'aにおいてジェネリックなので注釈をつける必要がある
    // 'aがついたselfパラメータもそうでしょう？ (答え: 間違い)
    fn some_method(&'a mut self) {}
}

fn main() {
    let mut num_ref = NumRef(&5);
    num_ref.some_method(); // mutably borrows num_ref for the rest of its lifetime
    num_ref.some_method(); // コンパイルエラー
    println!("{:?}", num_ref); // 同様にコンパイルエラー
}
```

`'a`でジェネリックな構造体があったとき`&'a mut self`を受け取るようなメソッドを書きたいことはほぼないでしょう。Rustに伝えていることは _"このメソッドは構造体のライフタイムの中で構造体を可変に借用する"_ ということなのです。実際、これは構造体が永久に可変借用されて利用不可になる前に、Rustの借用チェッカは最大でも1回しか`some_method`の呼び出しを許可しないのです。このユースケースは究極的にはほとんど無いのですが、上記のコードは混乱していた初心者が書いてコンパイルするにはとても簡単です。修正点としては、不必要に明示的なライフタイム注釈をつけることではなくRustのライフタイム省略ルールに任せるのです。

```rust
#[derive(Debug)]
struct NumRef<'a>(&'a i32);

impl<'a> NumRef<'a> {
    // mut selfには'aをつけない
    fn some_method(&mut self) {}

    // 上記の糖衣を外した版
    fn some_method_desugared<'b>(&'b mut self){}
}

fn main() {
    let mut num_ref = NumRef(&5);
    num_ref.some_method();
    num_ref.some_method(); // コンパイル可能
    println!("{:?}", num_ref); // コンパイル可能
}
```

**キーポイント**

- Rustの関数のライフタイム省略ルールはあらゆる状況で常に正しいわけではない
- Rustはプログラムの意味についてはプログラマ以上に知っているわけではない
- ライフタイム注釈には説明的な名前をつけよう
- どこに、そしてなぜ明示的なライフタイム注釈をつけるかについて気にかけよう

### 6) Boxトレイトオブジェクトはライフタイムを持たない

先ほどRustの _関数の_ ライフタイムの省略ルールについて見ました。Rustは同様にトレイトオブジェクトについてもライフタイムの省略ルールがあり、それは

- トレイトオブジェクトがジェネリックな型への型引数として使われた場合、そのライフタイム境界は保持している型から推論される
  - 保持しているのが一意であればそれが使われる
  - 保持しているのが複数あれば明示的な境界がなければならない
- もし上記が適用されない場合
  - トレイトが一つのライフタイム境界で定義されているならばそれが使われる
  - ライフタイム境界に`'static`が使われている場合、`'static`となる
  - 何もライフタイム境界が無い場合、式から推論され、式の外では`'static`となる

これらは非常に複雑に思えますが、 _トレイトオブジェクトのライフタイム境界は文脈から推論される_ とシンプルにまとめられます。いくつか手軽な例を見ればライフタイム境界の推論が極めて直感的で形式的なルールを覚える必要がないとわかるでしょう。

```rust
use std::cell::Ref;

trait Trait {}

// 省略された記法
type T1 = Box<dyn Trait>;
// 元々の記法、Box<T> は T については何もライフタイム境界を持っていないので 'static と推論される
type T2 = Box<dyn Trait + 'static>;

// 省略された記法
impl dyn Trait {}
// 元々の記法
impl dyn Trait + 'static {}

// 省略された記法
type T3<'a> = &'a dyn Trait;
// 元々の記法、&'a T は T: 'a を要求するので 'a と推論される
type T4<'a> = &'a (dyn Trait + 'a);

// 省略された記法
type T5<'a> = Ref<'a, dyn Trait>;
// 元々の記法、Ref<'a, T> は T: 'a を要求するので 'a と推論される
type T6<'a> = Ref<'a, dyn Trait + 'a>;

trait GenericTrait<'a>: 'a {}

// 省略された記法
type T7<'a> = Box<dyn GenericTrait<'a>>;
// 元々の記法
type T8<'a> = Box<dyn GenericTrait<'a> + 'a>;

// 省略された記法
impl<'a> dyn GenericTrait<'a> {}
// 元々の記法
impl<'a> dyn GenericTrait<'a> + 'a {}
```

トレイトを実装した具体的な型は参照を持ち、またライフタイム境界を持っていて、対応するトレイトオブジェクトはライフタイム境界を持ちます。同様に明らかにライフタイム境界を持つ参照に対してトレイトを直接実装することもできます。

```rust
trait Trait {}

struct Struct {}
struct Ref<'a, T>(&'a T);

impl Trait for Struct {}
impl Trait for &Struct {} // 参照型への直接的なimpl Trait
impl<'a, T> Trait for Ref<'a, T> {} // 参照を保持した型へのimpl Trait
```

何にせよ、初心者が関数をトレイトオブジェクトを使ったものからジェネリクスを使ったものへリファクタリングする際、またはその逆を行うとしばしば混乱するため、この点については説明する価値があります。例として以下のプログラムを見てみましょう。

```rust
use std::fmt::Display;

fn dynamic_thread_print(t: Box<dyn Display + Send>) {
    std::thread::spawn(move || {
        println!("{}", t);
    }).join();
}

fn static_thread_print<T: Display + Send>(t: T) {
    std::thread::spawn(move || {
        println!("{}", t);
    }).join();
}
```

これはコンパイルエラーを吐きます。

```rust
error[E0310]: the parameter type `T` may not live long enough
  --> src/lib.rs:10:5
   |
9  | fn static_thread_print<T: Display + Send>(t: T) {
   |                        -- help: consider adding an explicit lifetime bound...: `T: 'static +`
10 |     std::thread::spawn(move || {
   |     ^^^^^^^^^^^^^^^^^^
   |
note: ...so that the type `[closure@src/lib.rs:10:24: 12:6 t:T]` will meet its required lifetime bounds
  --> src/lib.rs:10:5
   |
10 |     std::thread::spawn(move || {
   |     ^^^^^^^^^^^^^^^^^^
```

よし、素晴らしい。コンパイラはどのように修正すれば良いかを教えてくれているので早速問題になっている箇所を直しましょう。

```rust
use std::fmt::Display;

fn dynamic_thread_print(t: Box<dyn Display + Send>) {
    std::thread::spawn(move || {
        println!("{}", t);
    }).join();
}

fn static_thread_print<T: Display + Send + 'static>(t: T) {
    std::thread::spawn(move || {
        println!("{}", t);
    }).join();
}
```

今はこれでコンパイルされますが、この2つの関数は両者を比べるとおかしく見えます。どうして2番目の関数は`T`に対して1番目の関数が要求していない`'static`を要求しているのでしょうか？トリッキーな質問です。ライフタイムの省略ルールを適用することでRustは自動で1番目の関数では`'static`境界を推論しており、実際は両方とも`'static`境界を持っているのです。これがRustコンパイラが見ているものです。

```rust
use std::fmt::Display;

fn dynamic_thread_print(t: Box<dyn Display + Send + 'static>) {
    std::thread::spawn(move || {
        println!("{}", t);
    }).join();
}

fn static_thread_print<T: Display + Send + 'static>(t: T) {
    std::thread::spawn(move || {
        println!("{}", t);
    }).join();
}
```

**キーポイント**

- 全てのトレイトオブジェクトは推論されたデフォルトのライフタイム境界を持っている

### 7) コンパイラのエラーメッセージはプログラムの直し方を教えてくれる

**誤解の流れ**

- Rustのトレイトオブジェクトへのライフタイム省略ルールは常に正しい
- Rustはプログラムの意味についてプログラマよりも知っている

この誤解は前の2つのものが合わさったもので、例として

```rust
use std::fmt::Display;

fn box_displayable<T: Display>(t: T) -> Box<dyn Display> {
    Box::new(t)
}
```

これはエラーを吐きます。

```rust
error[E0310]: the parameter type `T` may not live long enough
 --> src/lib.rs:4:5
  |
3 | fn box_displayable<T: Display>(t: T) -> Box<dyn Display> {
  |                    -- help: consider adding an explicit lifetime bound...: `T: 'static +`
4 |     Box::new(t)
  |     ^^^^^^^^^^^
  |
note: ...so that the type `T` will meet its required lifetime bounds
 --> src/lib.rs:4:5
  |
4 |     Box::new(t)
  |     ^^^^^^^^^^^
```

よし、Boxトレイトオブジェクトにはライフタイム`'static`境界が自動で推論されて、推奨されている修正はその不文律に基づいていることを心に留めつつ早速コンパイラが教えてくれている通りに直してみよう。

```rust
use std::fmt::Display;

fn box_displayable<T: Display + 'static>(t: T) -> Box<dyn Display> {
    Box::new(t)
}
```

プログラムはコンパイルされます... しかしこれは本当にやりたかったことなんでしょうか？ひょっとしたら、いやもしかしたらそうではないのかもだけど。コンパイラは他の修正については言及してませんが、こちらもまた良さそうです。

```rust
use std::fmt::Display;

fn box_displayable<'a, T: Display + 'a>(t: T) -> Box<dyn Display + 'a> {
    Box::new(t)
}
```

この関数は前のものと全く同じ引数を受け付けるのに加え、それ以上です！改善したのでしょうか？いえ、必ずしもそうとは限らず、プログラムの要求と制約次第です。この例は少々抽象的なので、もっとシンプルで明らかなケースを見てみましょう。

```rust
fn return_first(a: &str, b: &str) -> &str {
    a
}
```

これは以下のようになります。

```rust
error[E0106]: missing lifetime specifier
 --> src/lib.rs:1:38
  |
1 | fn return_first(a: &str, b: &str) -> &str {
  |                    ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `a` or `b`
help: consider introducing a named lifetime parameter
  |
1 | fn return_first<'a>(a: &'a str, b: &'a str) -> &'a str {
  |                ^^^^    ^^^^^^^     ^^^^^^^     ^^^
```

エラーメッセージは引数と返り値の両方に同じライフタイムをつけるように推奨しています。もしこうすると、プログラムはコンパイルは通りますが、この関数の返り値の型は制約が強くなりすぎます。実際にやりたいのはこういうことでしょう。

```rust
fn return_first<'a>(a: &'a str, b: &str) -> &'a str {
    a
}
```

**キーポイント**

- Rustのトレイトオブジェクトへのライフタイム省略ルールはどんな状況でも正しいというわけではない
- Rustはプログラムの意味についてプログラマよりも知っているわけではない
- Rustのコンパイラが吐くエラーメッセージでおすすめされる修正はコンパイルが通るようにするが、コンパイルが通り、 _かつ_ プログラムへの要求をちゃんと満たしたものするわけではない

### 8) ライフタイムは実行時に伸び縮みする

**誤解の流れ**

- 保有者の型は実行時に参照をスワップしてライフタイムを変える
- Rustの借用チェッカは高度な制御フロー解析を行う

これはコンパイルされません。

```rust
struct Has<'lifetime> {
    lifetime: &'lifetime str,
}

fn main() {
    let long = String::from("long");
    let mut has = Has { lifetime: &long };
    assert_eq!(has.lifetime, "long");

    {
        let short = String::from("short");
        // 短いライフタイムへ"切り替える"
        has.lifetime = &short;
        assert_eq!(has.lifetime, "short");

        // 長いライフタイムへ"切り戻す" (しかし実際はそうではない)
        has.lifetime = &long;
        assert_eq!(has.lifetime, "long");
        // 短い方はここでドロップされる
    }

    // コンパイルエラー、短い方はドロップの後もまだ"借用されている"
    assert_eq!(has.lifetime, "long");
}
```

これは以下のようになります。

```rust
error[E0597]: `short` does not live long enough
  --> src/main.rs:11:24
   |
11 |         has.lifetime = &short;
   |                        ^^^^^^ borrowed value does not live long enough
...
15 |     }
   |     - `short` dropped here while still borrowed
16 |     assert_eq!(has.lifetime, "long");
   |     --------------------------------- borrow later used here
```

これも同様にコンパイルが通らず、上記と全く同様のエラーとなります。

```rust
struct Has<'lifetime> {
    lifetime: &'lifetime str,
}

fn main() {
    let long = String::from("long");
    let mut has = Has { lifetime: &long };
    assert_eq!(has.lifetime, "long");

    // このブロックは実行されない
    if false {
        let short = String::from("short");
        // 短いライフタイムへ"切り替える"
        has.lifetime = &short;
        assert_eq!(has.lifetime, "short");

        // 長いライフタイムへ"切り戻す" (しかし実際はそうではない)
        has.lifetime = &long;
        assert_eq!(has.lifetime, "long");
        // 短い方はここでドロップされる
    }

    // コンパイルエラー、短い方はドロップの後もまだ"借用されている"
    assert_eq!(has.lifetime, "long");
}
```

ライフタイムはコンパイル時に静的に検証される必要があり、Rustの借用チェッカはとても基礎的な制御フロー解析のみを行います。そのため`if-else`節のブロックや`match`節では変数にとって最短で可能性のあるライフタイムが取られます。変数は一度ライフタイムに境界づけられると _永遠に_ そのライフタイムに境界づけられます。変数のライフタイムは縮小はしますが、その縮小はコンパイル時に決定されるのです。

**キーポイント**

- ライフタイムはコンパイル時に静的に検証される
- ライフタイムは実行時には決して伸び縮みしない
- Rustの借用チェッカは、あらゆるコードパスは通りうるものという前提で常に変数の最短のライフタイムをとる

### 9) 可変参照から共有参照へ降格することは安全

**誤解の流れ**

- 参照を再借用するとそのライフタイムは終わって新しいライフタイムが始まる

Rustは暗黙的に可変借用を不可変なものとして再借用するため共有借用を期待して関数に可変借用を渡すことができます。

```rust
fn takes_shared_ref(n: &i32) {}

fn main() {
    let mut a = 10;
    takes_shared_ref(&mut a); // コンパイル可能
    takes_shared_ref(&*(&mut a)); // 上記の糖衣を外したもの
}
```

直感的には筋が通っていて、可変借用を不可変なものとして再借用するのは問題がないからです、ね？驚いたことにこれは違っていて、以下のプログラムはコンパイルが通りません。

```rust
fn main() {
    let mut a = 10;
    let b: &i32 = &*(&mut a); // 不可変として再借用
    let c: &i32 = &a;
    dbg!(b, c); // コンパイルエラー
}
```

以下のようなエラーを吐きます。

```rust
error[E0502]: cannot borrow `a` as immutable because it is also borrowed as mutable
 --> src/main.rs:4:19
  |
3 |     let b: &i32 = &*(&mut a);
  |                     -------- mutable borrow occurs here
4 |     let c: &i32 = &a;
  |                   ^^ immutable borrow occurs here
5 |     dbg!(b, c);
  |          - mutable borrow later used here
```

可変借用は起きていますが、すぐに不可変として再借用され、そしてドロップされます。どうしてRustは不可変な再借用をまだ可変参照の排他的なライフタイムを持っているかのように扱うのでしょう？特に上記の例は何も問題がないのですが、実際には可変参照を共有参照へ降格することを可能とすることでメモリ安全の問題が起きているのです。

```rust
use std::sync::Mutex;

struct Struct {
    mutex: Mutex<String>
}

impl Struct {
    // 可変参照を共有参照へ降格
    fn get_string(&mut self) -> &str {
        self.mutex.get_mut().unwrap()
    }
    fn mutate_string(&self) {
        // 可変参照を共有参照へ降格を可能にしてしまうと
        // 以下の行ではget_stringメソッドから得られる共有参照を
        // 無効にしてしまうのです
        *self.mutex.lock().unwrap() = "surprise!".to_owned();
    }
}

fn main() {
    let mut s = Struct {
        mutex: Mutex::new("string".to_owned())
    };
    let str_ref = s.get_string(); // 共有参照へ降格された可変参照
    s.mutate_string(); // str_refは無効化され、ダングリングポインタになっているg!(str_ref); // 予想通りコンパイルエラー
}
```

ここでのポイントは、可変参照を共有参照として再借用したとき、覚悟がないと共有参照は使えません。共有参照自体がドロップされていても再借用の期間だけ共有参照のライフタイムを延長するのです。再借用された共有参照を使用するのはとても難しく、これは不可変でありつつも他のどの共有参照とも重複してはいけないからなのです。再借用された共有参照は可変参照の短所と共有参照の短所を全て持っており、どちらの長所も持っていません。可変参照を共有参照として再借用することはRustのアンチパターンであるべきだと私は考えています。以下のようなコードを見たとき簡単にわかるよう、このアンチパターンを意識しておくことは重要です。

```rust
// 可変なTを共有なTへ降格
fn some_function<T>(some_arg: &mut T) -> &T;

struct Struct;

impl Struct {
    // 可変なselfを共有なselfへ降格
    fn some_method(&mut self) -> &self;

    // 可変なselfを共有なTへ降格
    fn other_method(&mut self) -> &T;
}
```

関数やメソッドのシグネチャで再借用を回避してもRustは自動的に暗黙の再借用が行われるため、気づかないうちにこの問題にぶつかってしまうことがあります。

```rust
use std::collections::HashMap;

type PlayerID = i32;

#[derive(Debug, Default)]
struct Player {
    score: i32,
}

fn start_game(player_a: PlayerID, player_b: PlayerID, server: &mut HashMap<PlayerID, Player>) {
    // サーバーからプレーヤーをもらい、まだ存在しなければ作成と挿入を行う
    let player_a: &Player = server.entry(player_a).or_default();
    let player_b: &Player = server.entry(player_b).or_default();

    // プレーヤーで何かしらの処理
    dbg!(player_a, player_b); // コンパイルエラー
}
```

上記の例はコンパイルが通りません。明示的な型注釈をつけているので`or_default()`は暗黙的に`&Player`として再借用した`&mut Player`を返します。やりたいことを実現するには以下のようにする必要があります。

```rust
use std::collections::HashMap;

type PlayerID = i32;

#[derive(Debug, Default)]
struct Player {
    score: i32,
}

fn start_game(player_a: PlayerID, player_b: PlayerID, server: &mut HashMap<PlayerID, Player>) {
    // 返ってきた可変なPlayerをドロップ、これは同時に扱うことができないため
    server.entry(player_a).or_default();
    server.entry(player_b).or_default();

    // 再びプレーヤーを取り寄せ、暗黙の再借用をすることなくここで不可変に取得
    let player_a = server.get(&player_a);
    let player_b = server.get(&player_b);

    // プレーヤーで何かしらの処理
    dbg!(player_a, player_b); // compiles
}
```

どこか変な感じがしますが、メモリ安全のためには必要な儀式なのです。

**キーポイント**

- 可変参照を共有参照として再借用しようとしよう、さもなくば嫌なことになります
- 可変参照を再借用しても、例えその参照がドロップされていたとしてもそのライフタイムは終わらない

### 10) クロージャは関数と同じライフタイム省略ルールに従う

これは誤解というよりRustの嬉しいところです。

クロージャは関数であるにも関わらず、関数と同じライフタイム省略ルールに従わないのです。

```rust
fn function(x: &i32) -> &i32 {
    x
}

fn main() {
    let closure = |x: &i32| x;
}
```

これは以下の通りとなります。

```rust
error: lifetime may not live long enough
 --> src/main.rs:6:29
  |
6 |     let closure = |x: &i32| x;
  |                       -   - ^ returning this value requires that `'1` must outlive `'2`
  |                       |   |
  |                       |   return type of closure is &'2 i32
  |                       let's call the lifetime of this reference `'1`
```

糖衣を外すと以下のようになります。

```rust
// 引数のライフタイムが返り値に適用される
fn function<'a>(x: &'a i32) -> &'a i32 {
    x
}

fn main() {
    // 引数と返り値は別々のライフタイムを持つ
    let closure = for<'a, 'b> |x: &'a i32| -> &'b i32 { x };
    // メモ: この行は有効な構文ではないが説明のためにこうしている
}
```

この不一致には何の理由もありません。クロージャは最初は関数とは異なる型推論のセマンティクスで実装されていましたが、ここでクロージャと関数を統一してしまうのは破壊的な変更になってしまうのでずっと行き詰まっているのです。そのため、どうしたらクロージャの型を明示的に注釈できるでしょう？選択肢としては以下の通りです。

```rust
fn main() {
    // トレイトオブジェクトへキャスト、そうするとunsizedになり、あらら、コンパイルエラーだ
    let identity: dyn Fn(&i32) -> &i32 = |x: &i32| x;

    // ワークアラウンドとしてヒープに確保、しかし変な感じだ
    let identity: Box<dyn Fn(&i32) -> &i32> = Box::new(|x: &i32| x);

    // 確保をスキップしてただ静的な参照を作る
    let identity: &dyn Fn(&i32) -> &i32 = &|x: &i32| x;

    // 上のものの構文糖衣を外したもの :)
    let identity: &'static (dyn for<'a> Fn(&'a i32) -> &'a i32 + 'static) = &|x: &i32| -> &i32 { x };

    // これが理想的なのだけれど無効な構文
    let identity: impl Fn(&i32) -> &i32 = |x: &i32| x;

    // これもまた良いのだけれど、これも無効な構文
    let identity = for<'a> |x: &'a i32| -> &'a i32 { x };

    // "impl trait"は関数の返り値の位置で動くため
    fn return_identity() -> impl Fn(&i32) -> &i32 {
        |x| x
    }
    let identity = return_identity();

    // 前のものの更にジェネリックなもの
    fn annotate<T, F>(f: F) -> F where F: Fn(&T) -> &T {
        f
    }
    let identity = annotate(|x: &i32| x);
}
```

上記の例から既に気付いてると思いますが、クロージャの型がトレイト境界として使われたとき普通の関数のライフタイム省略ルールに従うのです。

実際はここから得られる学びや発見は何もなくて、ただそうなのです。

**キーポイント**

- どの言語にもあららってなる部分があるんです 🤷


### 11) `'static`な参照は`'a`な参照になることを常に強制される

私は先ほどこんなコード例を出しました。

```rust
fn get_str<'a>() -> &'a str; // ジェネリックなもの
fn get_str() -> &'static str; // 'static のもの
```

何人かの読者からこの2つに実用的な違いはあるのかという質問がありました。最初はわからなかったのですが、少し調べたところ残念なことにその答えはYesで、これら2つの関数には実用的な違いがあります。

普通、値を扱うとき`'a`の参照の代わりに`'static`の参照が使えます。これはRustは自動的に`'static`な参照を`'a`の参照へと自動で強制するからです。直感的にはこれは筋が通っていて、短いライフタイムが必要なところで長いライフタイムを持つ参照を用いても何のメモリ安全の問題にならないからです。以下のプログラムは期待通りコンパイルされます。

```rust
use rand;

fn generic_str_fn<'a>() -> &'a str {
    "str"
}

fn static_str_fn() -> &'static str {
    "str"
}

fn a_or_b<T>(a: T, b: T) -> T {
    if rand::random() {
        a
    } else {
        b
    }
}

fn main() {
    let some_string = "string".to_owned();
    let some_str = &some_string[..];
    let str_ref = a_or_b(some_str, generic_str_fn()); // compiles
    let str_ref = a_or_b(some_str, static_str_fn()); // compiles
}
```

しかしこの強制は参照が関数の型シグネチャの一部のときは起きず、そのため以下のものはコンパイルされません。

```rust
use rand;

fn generic_str_fn<'a>() -> &'a str {
    "str"
}

fn static_str_fn() -> &'static str {
    "str"
}

fn a_or_b_fn<T, F>(a: T, b_fn: F) -> T
    where F: Fn() -> T
{
    if rand::random() {
        a
    } else {
        b_fn()
    }
}

fn main() {
    let some_string = "string".to_owned();
    let some_str = &some_string[..];
    let str_ref = a_or_b_fn(some_str, generic_str_fn); // compiles
    let str_ref = a_or_b_fn(some_str, static_str_fn); // compile error
}
```

これは以下のようなエラーになります。

```rust
error[E0597]: `some_string` does not live long enough
  --> src/main.rs:23:21
   |
23 |     let some_str = &some_string[..];
   |                     ^^^^^^^^^^^ borrowed value does not live long enough
...
25 |     let str_ref = a_or_b_fn(some_str, static_str_fn);
   |                   ---------------------------------- argument requires that `some_string` is borrowed for `'static`
26 | }
   | - `some_string` dropped here while still borrowed
```

これがRustのあららってなってしまうとこなのかは議論を呼ぶところです。なぜならこれは`&'static str`を`&'a str`と強制する簡単な場合ではなく、`for<T> Fn() -> &'static T`を`for<'a, T> Fn() -> &'a T`に強制するものだからです。前者は値同士の強制であり、後者は型同士の強制なのです。

**キーポイント**

- `for<'a, T> fn() -> &'a T`のシグネチャのある関数は`for<T> fn() -> &'static T`のシグネチャのある関数よりも柔軟で色々なケースで動く

## まとめ

- `T`は`&T`と`&mut T`の上位集合
- `&T`と`&mut T`は互いに独立
- `T: 'static`は _"`T`はライフタイム`'static`で境界づけられている"_ と読むべき
- `T: 'static`ならば`T`はライフタイム`'static` _または_ 所有型を持った借用型でありうる。
- `T: 'static`は所有型を含み、これの意味するところとして`T`は
  - 実行時に動的に確保される
  - プログラム全体で有効である必要はない
  - 安全かつ自由に変更可能
  - 実行時に動的にドロップされる
  - 異なる期間のライフタイムを持つ
- `T: 'a`は`&'a T`よりも一般的で柔軟
- `T: 'a`は所有型と参照を含んだ所有型、参照を受け付ける
- `&'a T`は参照のみ受け付ける
- 任意の`'a`に対して`'static` >= `'a`なので`T: 'static`ならば`T: 'a`
- Rustのコードのほとんどはジェネリックなコードで、どこにでも省略されたライフタイムが存在する
- Rustの関数のライフタイム省略ルールはあらゆる状況で常に正しいわけではない
- Rustはプログラムの意味についてはプログラマ以上に知っているわけではない
- ライフタイム注釈には説明的な名前をつけよう
- どこに、そしてなぜ明示的なライフタイム注釈をつけるかについて気にかけよう
- 全てのトレイトオブジェクトは推論されたデフォルトのライフタイム境界を持っている
- Rustのトレイトオブジェクトへのライフタイム省略ルールはどんな状況でも正しいというわけではない
- Rustはプログラムの意味についてプログラマよりも知っているわけではない
- Rustのコンパイラが吐くエラーメッセージでおすすめされる修正はコンパイルが通るようにするが、コンパイルが通り、 _かつ_ プログラムへの要求をちゃんと満たしたものするわけではない
- 可変参照を共有参照として再借用しようとしよう、さもなくば嫌なことになります
- 可変参照を再借用しても、例えその参照がドロップされていたとしてもそのライフタイムは終わらない
- どの言語にもあららってなる部分があるんです 🤷
- `for<'a, T> fn() -> &'a T`のシグネチャのある関数は`for<T> fn() -> &'static T`のシグネチャのある関数よりも柔軟で色々なケースで動く



## ディスカッション

この記事については以下でディスカッションされています。

- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/gmrcrq/common_rust_lifetime_misconceptions/)
- [official Rust users forum](https://users.rust-lang.org/t/blog-post-common-rust-lifetime-misconceptions/42950)
- [Twitter](https://twitter.com/pretzelhammer/status/1263505856903163910)
- [rust subreddit](https://www.reddit.com/r/rust/comments/golrsx/common_rust_lifetime_misconceptions/)
- [Hackernews](https://news.ycombinator.com/item?id=23279731)

## お知らせ

次のブログが公開されたときのお知らせを受け取ってください。

- [pretzelhammerのTwitterをフォローする](https://twitter.com/pretzelhammer) or
- このリポジトリのリリースをチェックする (`Watch`のドロップダウンをクリックして`Releases only`を選択)

## 参考

- [Sizedness in Rust](./../../sizedness-in-rust.md)
- [Learning Rust in 2020](./../../learning-rust-in-2020.md)
