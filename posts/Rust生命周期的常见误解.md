# Rust生命周期常见误区

_5月19日, 2020 · 阅读大概需要34分钟 · #rust · #生命周期_

**目录**
- [Intro](#Intro)
- [The Misconceptions](#the-misconceptions)
    - [1) `T` only contains owned types](#1-t-only-contains-owned-types)
    - [2) if `T: 'static` then `T` must be valid for the entire program](#2-if-t-static-then-t-must-be-valid-for-the-entire-program)
    - [3) `&'a T` and `T: 'a` are the same thing](#3-a-t-and-t-a-are-the-same-thing)
    - [4) my code isn't generic and doesn't have lifetimes](#4-my-code-isnt-generic-and-doesnt-have-lifetimes)
    - [5) if it compiles then my lifetime annotations are correct](#5-if-it-compiles-then-my-lifetime-annotations-are-correct)
    - [6) boxed trait objects don't have lifetimes](#6-boxed-trait-objects-dont-have-lifetimes)
    - [7) compiler error messages will tell me how to fix my program](#7-compiler-error-messages-will-tell-me-how-to-fix-my-program)
    - [8) lifetimes can grow and shrink at run-time](#8-lifetimes-can-grow-and-shrink-at-run-time)
    - [9) downgrading mut refs to shared refs is safe](#9-downgrading-mut-refs-to-shared-refs-is-safe)
    - [10) closures follow the same lifetime elision rules as functions](#10-closures-follow-the-same-lifetime-elision-rules-as-functions)
- [Conclusion](#conclusion)
- [Discuss](#discuss)
- [Follow](#follow)



## 介绍

我曾经有过的所有这些对生命周期的误解，现在有很多初学者也深陷于此。
我用到的术语可能不是标准的，所以列了一个表格来解释它们的用意。


| 短语 | 意为 |
|-|-|
| `T` | 1) 包含了所有可能类型的集合 _或_<br>2) 这个集合中的类型 |
| 所有权类型 | 不含引用的类型, 例如 `i32`, `String`, `Vec`, 等 |
| 1) 借用类型 _或_<br>2) 引用类型 | 不考虑可变性的引用类型, 例如 `&i32`, `&mut i32`, 等 |
| 1) 可变引用 _或_<br>2) 独占引用 | 独占的可变引用, 即 `&mut T` |
| 1) 不可变引用 _or_<br>2) 共享引用 | 共享的不可变引用, 即 `&T` |



## 误解列表

简而言之：变量的生命周期指的是这个变量所指的数据 可以被编译器静态验证的 在当前内存地址有效期的长度。
我现在会用大约TODO字来详细地解释一下那些容易误解的地方。

### 1) `T` 只包含所有权类型

这个误解比起说生命周期，它和泛型更相关，但在Rust中泛型和生命周期是紧密联系在一起的，不可只谈其一。

当我刚开始学习Rust的时候，我理解`i32`，`&i32`，和`&mut i32`是不同的类型，也明白泛型变量`T`代表着所有可能类型的集合。
但尽管这二者分开都懂，当它们结合在一起的时候我却陷入困惑。在我这个Rust初学者的眼中，泛型是这样的运作的：

| | | | |
|-|-|-|-|
| **类型变量** | `T` | `&T` | `&mut T` |
| **例子** | `i32` | `&i32` | `&mut i32` |

`T` 包含一切所有权类型； `&T` 包含一切不可变借用类型； `&mut T` 包含一切可变借用类型。
`T`， `&T`， 和 `&mut T` 是不相交的有限集。 简洁明了，符合直觉，但却完全错误。
这才是泛型真正的运作方式：

| | | | |
|-|-|-|-|
| **类型变量** | `T` | `&T` | `&mut T` |
| **例子** | `i32`, `&i32`, `&mut i32`, `&&i32`, `&mut &mut i32`, ... | `&i32`, `&&i32`, `&&mut i32`, ... | `&mut i32`, `&mut &mut i32`, `&mut &i32`, ... |

`T`, `&T`, 和 `&mut T` 都是无限集, 因为你可以无限借用一个类型。
`T` 是 `&T` 和 `&mut T`的超集. `&T` 和 `&mut T` 是不相交的集合。
让我们用几个例子来检验一下这些概念:

```rust
trait Trait {}

impl<T> Trait for T {}

impl<T> Trait for &T {} // 编译错误

impl<T> Trait for &mut T {} // 编译错误
```

上面的代码并不能如愿编译:

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

编译器不允许我们为`&T`和`&mut T`实现`Trait`，因为这样会与为`T`实现的`Trait`冲突，
`T`本身已经包含了所有`T`和`&mut T`。下面的代码能够如愿编译，因为`&T`和`&mut T`是不相交的：

```rust
trait Trait {}

impl<T> Trait for &T {} // 编译通过

impl<T> Trait for &mut T {} // 编译通过
```

**要点**
- `T` 是 `&T` 和 `&mut T`的超集
- `&T` 和 `&mut T` 是不相交的集合


### 2) 如果 `T: 'static` 那么 `T` 必须在整个程序运行中都是有效的

**误解推论**
- `T: 'static` 应该被看作 _"`T` 拥有 `'static` 生命周期"_
- `&'static T` 和 `T: 'static` 没有区别
- 如果 `T: 'static` 那么 `T` 必须为不可变的
- 如果 `T: 'static` 那么 `T` 只能在编译期创建

大部分Rust初学者是从类似下面这个代码示例中接触到 `'static` 生命周期的：

```rust
fn main() {
    let str_literal: &'static str = "str literal";
}
```

他们被告知 `"str literal"` 是硬编码在编译出来的二进制文件中的，
并会在运行时被加载到只读内存，所以必须是不可变的且在整个程序的运行中都是有效的，
这就是它成为 `'static` 的原因。
这些观念又进一步被用 `static` 关键字来定义静态变量的规则所加强。


```rust
static BYTES: [u8; 3] = [1, 2, 3];
static mut MUT_BYTES: [u8; 3] = [1, 2, 3];

fn main() {
   MUT_BYTES[0] = 99; // 编译错误，修改静态变量是unsafe的

    unsafe {
        MUT_BYTES[0] = 99;
        assert_eq!(99, MUT_BYTES[0]);
    }
}
```

认为静态变量
- 只可以在编译期创建
- 必须是不可变的，修改它们是unsafe的
- 在整个程序的运行过程中都是有效的

`'static` 生命周期大概是以静态变量的默认生命周期命名的，对吧？
那么有理由认为`'static`生命周期也应该遵守相同的规则，不是吗？

是的，但拥有`'static`生命周期的类型与`'static`约束的类型是不同的。
后者能在运行时动态分配，可以安全地、自由地修改，可以被drop，
还可以有任意长度的生命周期。

在这个点，很重要的是要区分 `&'static T` 和 `T: 'static`。

`&'static T`是对某个`T`的不可变引用，这个引用可以被无限期地持有直到程序结束。
这只可能发生在`T`本身不可变且不会在引用被创建后移动的情况下。
`T`并不需要在编译期就被创建，因为我们可以在运行时动态生成随机数据，
然后以内存泄漏为代价返回`'static`引用，例如：


```rust
use rand;

// 在运行时生成随机&'static str
fn rand_str_generator() -> &'static str {
    let rand_string = rand::random::<u64>().to_string();
    Box::leak(rand_string.into_boxed_str())
}
```

`T: 'static` 是指`T`可以被无限期安全地持有直到程序结束。
`T: 'static`包括所有`&'static T`，此外还包括所有的所有权类型，比如`String`, `Vec`等。
数据的所有者能够保证数据只要还被持有就不会失效，因此所有者可以无限期安全地持有该数据直到程序结束。
`T: 'static`应该被看作“`T`受`'static`生命周期约束”而非“`T`有着`'static`生命周期”。
这段代码能帮我们阐释这些概念：


```rust
use rand;

fn drop_static<T: 'static>(t: T) {
    std::mem::drop(t);
}

fn main() {
    let mut strings: Vec<String> = Vec::new();
    for _ in 0..10 {
        if rand::random() {
            // 所有字符串都是随机生成的
            // 并且是在运行时动态申请的
            let string = rand::random::<u64>().to_string();
            strings.push(string);
        }
    }

    // 这些字符串都是所有权类型，所以它们满足'static约束
    for mut string in strings {
        // 这些字符串都是可以修改的
        string.push_str("a mutation");
        // 这些字符串都是可以被drop的
        drop_static(string); // 编译通过
    }

    // 这些字符串都在程序结束之前失效
    println!("i am the end of the program");
}
```

**要点**
- `T: 'static` 应该被看作 _“`T`受`'static`生命周期约束”_
- 如果 `T: 'static` 那么`T`可以是有着`'static`生命周期的借用类型
- 由于 `T: 'static` 包括了所有权类型，这意味着`T`
    - 可以在运行时动态分配
    - 不一定要在整个程序的运行过程中都有效
    - 可以被安全地、自由地修改
    - 可以在运行时被动态drop掉
    - 可以有不同长度的生命周期


### 3) `&'a T` 和 `T: 'a` 是相同的

这个误解是上一个的泛化版本。

`&'a T` 不光要求，同时也隐含着 `T: 'a`， 因为如果`T`本身都不能在`'a`内有效，
那对`T`的有`'a`生命周期的引用也不可能是有效的。
例如，Rust编译器从来不会允许创建`&'static Ref<'a, T>`这个类型，因为如果`Ref`只在`'a`内有效，我们不可能弄出一个对它的`'static`的引用。

`T: 'a`包括了所有`&'a T`，但反过来不对。

```rust
// 只接受以'a约束的引用类型
fn t_ref<'a, T: 'a>(t: &'a T) {}

// 接受所有以'a约束的类型
fn t_bound<'a, T: 'a>(t: T) {}

// 包含引用的所有权类型
struct Ref<'a, T: 'a>(&'a T);

fn main() {
    let string = String::from("string");

    t_bound(&string); // 编译通过
    t_bound(Ref(&string)); // 编译通过
    t_bound(&Ref(&string)); // 编译通过

    t_ref(&string); // 编译通过
    t_ref(Ref(&string)); // 编译错误, 期待接收一个引用，但收到一个结构体
    t_ref(&Ref(&string)); // 编译通过

    // string变量是以'static约束的，也满足'a约束
    t_bound(string); // 编译通过
}
```

**要点**
- `T: 'a` 比起 `&'a T`更泛化也更灵活
- `T: 'a` 接受所有权类型、包含引用的所有权类型以及引用
- `&'a T` 只接受引用
- 如果 `T: 'static` 那么 `T: 'a`, 因为对于所有`'a`都有`'static` >= `'a`



### 4) my code isn't generic and doesn't have lifetimes

**Misconception Corollaries**
- it's possible to avoid using generics and lifetimes

This comforting misconception is kept alive thanks to Rust's lifetime elision rules, which allow you to omit lifetime annotations in functions because the Rust borrow checker will infer them following these rules:
- every input ref to a function gets a distinct lifetime
- if there's exactly one input lifetime it gets applied to all output refs
- if there's multiple input lifetimes but one of them is `&self` or `&mut self` then the lifetime of `self` is applied to all output refs
- otherwise output lifetimes have to be made explicit

That's a lot to take in so lets look at some examples:

```rust
// elided
fn print(s: &str);

// expanded
fn print<'a>(s: &'a str);

// elided
fn trim(s: &str) -> &str;

// expanded
fn trim<'a>(s: &'a str) -> &'a str;

// illegal, can't determine output lifetime, no inputs
fn get_str() -> &str;

// explicit options include
fn get_str<'a>() -> &'a str; // generic version
fn get_str() -> &'static str; // 'static version

// illegal, can't determine output lifetime, multiple inputs
fn overlap(s: &str, t: &str) -> &str;

// explicit (but still partially elided) options include
fn overlap<'a>(s: &'a str, t: &str) -> &'a str; // output can't outlive s
fn overlap<'a>(s: &str, t: &'a str) -> &'a str; // output can't outlive t
fn overlap<'a>(s: &'a str, t: &'a str) -> &'a str; // output can't outlive s & t
fn overlap(s: &str, t: &str) -> &'static str; // output can outlive s & t
fn overlap<'a>(s: &str, t: &str) -> &'a str; // no relationship between input & output lifetimes

// expanded
fn overlap<'a, 'b>(s: &'a str, t: &'b str) -> &'a str;
fn overlap<'a, 'b>(s: &'a str, t: &'b str) -> &'b str;
fn overlap<'a>(s: &'a str, t: &'a str) -> &'a str;
fn overlap<'a, 'b>(s: &'a str, t: &'b str) -> &'static str;
fn overlap<'a, 'b, 'c>(s: &'a str, t: &'b str) -> &'c str;

// elided
fn compare(&self, s: &str) -> &str;

// expanded
fn compare<'a, 'b>(&'a self, &'b str) -> &'a str;
```

If you've ever written
- a struct method
- a function which takes references
- a function which returns references
- a generic function
- a trait object (more on this later)
- a closure (more on this later)

then your code has generic elided lifetime annotations all over it.

**Key Takeaways**
- almost all Rust code is generic code and there's elided lifetime annotations everywhere



### 5) if it compiles then my lifetime annotations are correct

**Misconception Corollaries**
- Rust's lifetime elision rules for functions are always right
- Rust's borrow checker is always right, technically _and semantically_
- Rust knows more about the semantics of my program than I do

It's possible for a Rust program to be technically compilable but still semantically wrong. Take this for example:

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

`ByteIter` is an iterator that iterates over a slice of bytes. We're skipping the `Iterator` trait implementation for conciseness. It seems to work fine, but what if we want to check a couple bytes at a time?

```rust
fn main() {
    let mut bytes = ByteIter { remainder: b"1123" };
    let byte_1 = bytes.next();
    let byte_2 = bytes.next();
    if byte_1 == byte_2 {
        // do something
    }
}
```

Uh oh! Compile error:

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

I guess we can copy each byte. Copying is okay when we're working with bytes but if we turned `ByteIter` into a generic slice iterator that can iterate over any `&'a [T]` then we might want to use it in the future with types that may be very expensive or impossible to copy / clone. Oh well, I guess there's nothing we can do about that, the code compiles so the lifetime annotations must be right, right?

Nope, the current lifetime annotations are actually the source of the bug! It's particularly hard to spot because the buggy lifetime annotations are elided. Lets expand the elided lifetimes to get a clearer look at the problem:

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

That didn't help at all. I'm still confused. Here's a hot tip that only Rust pros know: give your lifetime annotations descriptive names. Lets try again:

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

Each returned byte is annotated with `'mut_self` but the bytes are clearly coming from `'remainder`! Lets fix it.

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
    std::mem::drop(bytes); // we can even drop the iterator now!
    if byte_1 == byte_2 { // compiles
        // do something
    }
}
```

Now that we look back on the previous version of our program it was obviously wrong, so why did Rust compile it? The answer is simple: it was memory safe.

The Rust borrow checker only cares about the lifetime annotations in a program to the extent it can use them to statically verify the memory safety of the program. Rust will happily compile programs even if the lifetime annotations have semantic errors, and the consequence of this is that the program becomes unnecessarily restrictive.

Here's a quick example that's the opposite of the previous example: Rust's lifetime elision rules happen to be semantically correct in this instance but we unintentionally write a very restrictive method with our own unnecessary explicit lifetime annotations.

```rust
#[derive(Debug)]
struct NumRef<'a>(&'a i32);

impl<'a> NumRef<'a> {
    // my struct is generic over 'a so that means I need to annotate
    // my self parameters with 'a too, right? (answer: no, not right)
    fn some_method(&'a mut self) {}
}

fn main() {
    let mut num_ref = NumRef(&5);
    num_ref.some_method(); // mutably borrows num_ref for the rest of its lifetime
    num_ref.some_method(); // compile error
    println!("{:?}", num_ref); // also compile error
}
```

If we have some struct generic over `'a` we almost never want to write a method with a `&'a mut self` receiver. What we're communicating to Rust is "this method will mutably borrow the struct for the entirety of the struct's lifetime". In practice this means Rust's borrow checker will only allow at most one call to `some_method` before the struct becomes permanently mutably borrowed and thus unusable. The use-cases for this are extremely rare but the code above is very easy for confused beginners to write and it compiles. The fix is to not add unnecessary explicit lifetime annotations and let Rust's lifetime elision rules handle it:

```rust
#[derive(Debug)]
struct NumRef<'a>(&'a i32);

impl<'a> NumRef<'a> {
    // no more 'a on mut self
    fn some_method(&mut self) {}

    // above line desugars to
    fn some_method_desugared<'b>(&'b mut self){}
}

fn main() {
    let mut num_ref = NumRef(&5);
    num_ref.some_method();
    num_ref.some_method(); // compiles
    println!("{:?}", num_ref); // compiles
}
```

**Key Takeaways**
- Rust's lifetime elision rules for functions are not always right for every situation
- Rust does not know more about the semantics of your program than you do
- give your lifetime annotations descriptive names
- try to be mindful of where you place explicit lifetime annotations and why



### 6) boxed trait objects don't have lifetimes

Earlier we discussed Rust's lifetime elision rules _for functions_. Rust also has lifetime elision rules for trait objects, which are:
- if a trait object is used as a type argument to a generic type then its life bound is inferred from the containing type
    - if there's a unique bound from the containing then that's used
    - if there's more than one bound from the containing type then an explicit bound must be specified
- if the above doesn't apply then
    - if the trait is defined with a single lifetime bound then that bound is used
    - if `'static` is used for any lifetime bound then `'static` is used
    - if the trait has no lifetime bounds then its lifetime is inferred in expressions and is `'static` outside of expressions

All of that sounds super complicated but can be simply summarized as _"a trait object's lifetime bound is inferred from context."_ After looking at a handful of examples we'll see the lifetime bound inferences are pretty intuitive so we don't have to memorize the formal rules:

```rust
use std::cell::Ref;

trait Trait {}

// elided
type T1 = Box<dyn Trait>;
// expanded, Box<T> has no lifetime bound on T, so inferred as 'static
type T2 = Box<dyn Trait + 'static>;

// elided
impl dyn Trait {}
// expanded
impl dyn Trait + 'static {}

// elided
type T3<'a> = &'a dyn Trait;
// expanded, &'a T requires T: 'a, so inferred as 'a
type T4<'a> = &'a (dyn Trait + 'a);

// elided
type T5<'a> = Ref<'a, dyn Trait>;
// expanded, Ref<'a, T> requires T: 'a, so inferred as 'a
type T6<'a> = Ref<'a, dyn Trait + 'a>;

trait GenericTrait<'a>: 'a {}

// elided
type T7<'a> = Box<dyn GenericTrait<'a>>;
// expanded
type T8<'a> = Box<dyn GenericTrait<'a> + 'a>;

// elided
impl<'a> dyn GenericTrait<'a> {}
// expanded
impl<'a> dyn GenericTrait<'a> + 'a {}
```

Concrete types which implement traits can have references and thus they also have lifetime bounds, and so their corresponding trait objects have lifetime bounds. Also you can implement traits directly for references which obviously have lifetime bounds:

```rust
trait Trait {}

struct Struct {}
struct Ref<'a, T>(&'a T);

impl Trait for Struct {}
impl Trait for &Struct {} // impl Trait directly on a ref type
impl<'a, T> Trait for Ref<'a, T> {} // impl Trait on a type containing refs
```

Anyway, this is worth going over because it often confuses beginners when they refactor a function from using trait objects to generics or vice versa. Take this program for example:

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

It throws these compile errors:

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

Okay great, the compiler tells us how to fix the issue so lets fix the issue.

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

It compiles now but these two functions look awkward next to each other, why does the second function require a `'static` bound on `T` where the first function doesn't? That's a trick question. Using the lifetime elision rules Rust automatically infers a `'static` bound in the first function so both actually have `'static` bounds. This is what the Rust compiler sees:

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

**Key Takeaways**
- all trait objects have some inferred default lifetime bounds



### 7) compiler error messages will tell me how to fix my program

**Misconception Corollaries**
- Rust's lifetime elision rules for trait objects are always right
- Rust knows more about the semantics of my program than I do

This misconception is the previous 2 misconceptions combined into one example:

```rust
use std::fmt::Display;

fn box_displayable<T: Display>(t: T) -> Box<dyn Display> {
    Box::new(t)
}
```

Throws this error:

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

Okay, let's fix it how the compiler is telling us to fix it, nevermind the fact that it's automatically inferring a `'static` lifetime bound for our boxed trait object without telling us and its recommended fix is based on that unstated fact:

```rust
use std::fmt::Display;

fn box_displayable<T: Display + 'static>(t: T) -> Box<dyn Display> {
    Box::new(t)
}
```

So the program compiles now... but is this what we actually want? Probably, but maybe not. The compiler didn't mention any other fixes but this would have also been appropriate:

```rust
use std::fmt::Display;

fn box_displayable<'a, T: Display + 'a>(t: T) -> Box<dyn Display + 'a> {
    Box::new(t)
}
```

This function accepts all the same arguments as the previous version plus a lot more! Does that make it better? Not necessarily, it depends on the requirements and constraints of our program. This example is a bit abstract so lets take a look at a simpler and more obvious case:

```rust
fn return_first(a: &str, b: &str) -> &str {
    a
}
```

Throws:

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

The error message recommends annotating both inputs and the output with the same lifetime. If we did this our program would compile but this function would overly-constrain the return type. What we actually want is this:

```rust
fn return_first<'a>(a: &'a str, b: &str) -> &'a str {
    a
}
```

**Key Takeaways**
- Rust's lifetime elision rules for trait objects are not always right for every situation
- Rust does not know more about the semantics of your program than you do
- Rust compiler error messages suggest fixes which will make your program compile which is not that same as fixes which will make you program compile _and_ best suit the requirements of your program



### 8) lifetimes can grow and shrink at run-time

**Misconception Corollaries**
- container types can swap references at run-time to change their lifetime
- Rust borrow checker does advanced control flow analysis

This does not compile:

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
        // "switch" to short lifetime
        has.lifetime = &short;
        assert_eq!(has.lifetime, "short");

        // "switch back" to long lifetime (but not really)
        has.lifetime = &long;
        assert_eq!(has.lifetime, "long");
        // `short` dropped here
    }

    // compile error, `short` still "borrowed" after drop
    assert_eq!(has.lifetime, "long");
}
```

It throws:

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

This also does not compile, throws the exact same error as above:

```rust
struct Has<'lifetime> {
    lifetime: &'lifetime str,
}

fn main() {
    let long = String::from("long");
    let mut has = Has { lifetime: &long };
    assert_eq!(has.lifetime, "long");

    // this block will never run
    if false {
        let short = String::from("short");
        // "switch" to short lifetime
        has.lifetime = &short;
        assert_eq!(has.lifetime, "short");

        // "switch back" to long lifetime (but not really)
        has.lifetime = &long;
        assert_eq!(has.lifetime, "long");
        // `short` dropped here
    }

    // still a compile error, `short` still "borrowed" after drop
    assert_eq!(has.lifetime, "long");
}
```

Lifetimes have to be statically verified at compile-time and the Rust borrow checker only does very basic control flow analysis, so it assumes every block in an `if-else` statement and every match arm in a `match` statement can be taken and then chooses the shortest possible lifetime for the variable. Once a variable is bounded by a lifetime it is bounded by that lifetime _forever_. The lifetime of a variable can only shrink, and all the shrinkage is determined at compile-time.

**Key Takeaways**
- lifetimes are statically verified at compile-time
- lifetimes cannot grow or shrink or change in any way at run-time
- Rust borrow checker will always choose the shortest possible lifetime for a variable assuming all code paths can be taken



### 9) downgrading mut refs to shared refs is safe

**Misconception Corollaries**
- re-borrowing a reference ends its lifetime and starts a new one

You can pass a mut ref to a function expecting a shared ref because Rust will implicitly re-borrow the mut ref as immutable:

```rust
fn takes_shared_ref(n: &i32) {}

fn main() {
    let mut a = 10;
    takes_shared_ref(&mut a); // compiles
    takes_shared_ref(&*(&mut a)); // above line desugared
}
```

Intuitively this makes sense, since there's no harm in re-borrowing a mut ref as immutable, right? Surprisingly no, as the program below does not compile:

```rust
fn main() {
    let mut a = 10;
    let b: &i32 = &*(&mut a); // re-borrowed as immutable
    let c: &i32 = &a;
    dbg!(b, c); // compile error
}
```

Throws this error:

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

A mutable borrow does occur, but it's immediately and unconditionally re-borrowed as immutable and then dropped. Why is Rust treating the immutable re-borrow as if it still has the mut ref's exclusive lifetime? While there's no issue in the particular example above, allowing the ability to downgrade mut refs to shared refs does indeed introduce potential memory safety issues:

```rust
use std::sync::Mutex;

struct Struct {
    mutex: Mutex<String>
}

impl Struct {
    // downgrades mut self to shared str
    fn get_string(&mut self) -> &str {
        self.mutex.get_mut().unwrap()
    }
    fn mutate_string(&self) {
        // if Rust allowed downgrading mut refs to shared refs
        // then the following line would invalidate any shared
        // refs returned from the get_string method
        *self.mutex.lock().unwrap() = "surprise!".to_owned();
    }
}

fn main() {
    let mut s = Struct {
        mutex: Mutex::new("string".to_owned())
    };
    let str_ref = s.get_string(); // mut ref downgraded to shared ref
    s.mutate_string(); // str_ref invalidated, now a dangling pointer
    dbg!(str_ref); // compile error as expected
}
```

The point here is that when you re-borrow a mut ref as a shared ref you don't get that shared ref without a big gotcha: it extends the mut ref's lifetime for the duration of the re-borrow even if the mut ref itself is dropped. Using the re-borrowed shared ref is very difficult because it's immutable but it can't overlap with any other shared refs. The re-borrowed shared ref has all the cons of a mut ref and all the cons of a shared ref and has the pros of neither. I believe re-borrowing a mut ref as a shared ref should be considered a Rust anti-pattern. Being aware of this anti-pattern is important so that you can easily spot it when you see code like this:

```rust
// downgrades mut T to shared T
fn some_function<T>(some_arg: &mut T) -> &T;

struct Struct;

impl Struct {
    // downgrades mut self to shared self
    fn some_method(&mut self) -> &self;

    // downgrades mut self to shared T
    fn other_method(&mut self) -> &T;
}
```

Even if you avoid re-borrows in function and method signatures Rust still does automatic implicit re-borrows so it's easy to bump into this problem without realizing it like so:

```rust
use std::collections::HashMap;

type PlayerID = i32;

#[derive(Debug, Default)]
struct Player {
    score: i32,
}

fn start_game(player_a: PlayerID, player_b: PlayerID, server: &mut HashMap<PlayerID, Player>) {
    // get players from server or create & insert new players if they don't yet exist
    let player_a: &Player = server.entry(player_a).or_default();
    let player_b: &Player = server.entry(player_b).or_default();

    // do something with players
    dbg!(player_a, player_b); // compile error
}
```

The above fails to compile. `or_default()` returns a `&mut Player` which we're implicitly re-borrowing as `&Player` because of our explicit type annotations. To do what we want we have to:

```rust
use std::collections::HashMap;

type PlayerID = i32;

#[derive(Debug, Default)]
struct Player {
    score: i32,
}

fn start_game(player_a: PlayerID, player_b: PlayerID, server: &mut HashMap<PlayerID, Player>) {
    // drop the returned mut Player refs since we can't use them together anyway
    server.entry(player_a).or_default();
    server.entry(player_b).or_default();

    // fetch the players again, getting them immutably this time, without any implicit re-borrows
    let player_a = server.get(&player_a);
    let player_b = server.get(&player_b);

    // do something with players
    dbg!(player_a, player_b); // compiles
}
```

Kinda awkward and clunky but this is the sacrifice we make at the Altar of Memory Safety.

**Key Takeaways**
- try not to re-borrow mut refs as shared refs, or you're gonna have a bad time
- re-borrowing a mut ref doesn't end its lifetime, even if the ref is dropped



### 10) closures follow the same lifetime elision rules as functions

This is more of a Rust Gotcha than a misconception.

Closures, despite being functions, do not follow the same lifetime elision rules as functions.

```rust
fn function(x: &i32) -> &i32 {
    x
}

fn main() {
    let closure = |x: &i32| x;
}
```

Throws:

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

After desugaring we get:

```rust
// input lifetime gets applied to output
fn function<'a>(x: &'a i32) -> &'a i32 {
    x
}

fn main() {
    // input and output each get their own distinct lifetimes
    let closure = for<'a, 'b> |x: &'a i32| -> &'b i32 { x };
    // note: the above line is not valid syntax, but we need it for illustrative purposes
}
```

There's no good reason for this discrepancy. Closures were first implemented with different type inference semantics than functions and now we're stuck with it forever because to unify them at this point would be a breaking change. So how can we explicitly annotate a closure's type? Our options include:

```rust
fn main() {
    // cast to trait object, becomes unsized, oops, compile error
    let identity: dyn Fn(&i32) -> &i32 = |x: &i32| x;

    // can allocate it on the heap as a workaround but feels clunky
    let identity: Box<dyn Fn(&i32) -> &i32> = Box::new(|x: &i32| x);

    // can skip the allocation and just create a static reference
    let identity: &dyn Fn(&i32) -> &i32 = &|x: &i32| x;

    // previous line desugared :)
    let identity: &'static (dyn for<'a> Fn(&'a i32) -> &'a i32 + 'static) = &|x: &i32| -> &i32 { x };

    // this would be ideal but it's invalid syntax
    let identity: impl Fn(&i32) -> &i32 = |x: &i32| x;

    // this would also be nice but it's also invalid syntax
    let identity = for<'a> |x: &'a i32| -> &'a i32 { x };

    // since "impl trait" works in the function return position
    fn return_identity() -> impl Fn(&i32) -> &i32 {
        |x| x
    }
    let identity = return_identity();

    // more generic version of the previous solution
    fn annotate<T, F>(f: F) -> F where F: Fn(&T) -> &T {
        f
    }
    let identity = annotate(|x: &i32| x);
}
```

As I'm sure you've already noticed from the examples above, when closure types are used as trait bounds they do follow the usual function lifetime elision rules.

There's no real lesson or insight to be had here, it just is what it is.

**Key Takeaways**
- every language has gotchas 🤷



## Conclusion

- `T` is a superset of both `&T` and `&mut T`
- `&T` and `&mut T` are disjoint sets
- `T: 'static` should be read as _"`T` is bounded by a `'static` lifetime"_
- if `T: 'static` then `T` can be a borrowed type with a `'static` lifetime _or_ an owned type
- since `T: 'static` includes owned types that means `T`
    - can be dynamically allocated at run-time
    - does not have to be valid for the entire program
    - can be safely and freely mutated
    - can be dynamically dropped at run-time
    - can have lifetimes of different durations
- `T: 'a` is more general and more flexible than `&'a T`
- `T: 'a` accepts owned types, owned types which contain references, and references
- `&'a T` only accepts references
- if `T: 'static` then `T: 'a` since `'static` >= `'a` for all `'a`
- almost all Rust code is generic code and there's elided lifetime annotations everywhere
- Rust's lifetime elision rules are not always right for every situation
- Rust does not know more about the semantics of your program than you do
- give your lifetime annotations descriptive names
- try to be mindful of where you place explicit lifetime annotations and why
- all trait objects have some inferred default lifetime bounds
- Rust compiler error messages suggest fixes which will make your program compile which is not that same as fixes which will make you program compile _and_ best suit the requirements of your program
- lifetimes are statically verified at compile-time
- lifetimes cannot grow or shrink or change in any way at run-time
- Rust borrow checker will always choose the shortest possible lifetime for a variable assuming all code paths can be taken
- try not to re-borrow mut refs as shared refs, or you're gonna have a bad time
- re-borrowing a mut ref doesn't end its lifetime, even if the ref is dropped
- every language has gotchas 🤷



## Discuss

Discuss this article on
- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/gmrcrq/common_rust_lifetime_misconceptions/)
- [official Rust users forum](https://users.rust-lang.org/t/blog-post-common-rust-lifetime-misconceptions/42950)
- [Twitter](https://twitter.com/pretzelhammer/status/1263505856903163910)
- [rust subreddit](https://www.reddit.com/r/rust/comments/golrsx/common_rust_lifetime_misconceptions/)
- [Hackernews](https://news.ycombinator.com/item?id=23279731)



## Follow

[Follow pretzelhammer on Twitter](https://twitter.com/pretzelhammer) to get notified of future blog posts!
