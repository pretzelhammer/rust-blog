# Rust 标准库特性指南

_2021年 3月 31日 · #rust · #traits_

**目录**

- [引入 Intro](#引入-intro)
- [特性的基础知识 Trait Basics](#特性的基础知识-trait-basics)
    - [特性的记号 Trait Items](#特性的记号-trait-items)
        - [Self](#self)
        - [函数 Functions](#函数-functions)
        - [方法 Methods](#方法-methods)
        - [关联类型 Associated Types](#关联类型-associated-types)
        - [泛型参数 Generic Parameters](#泛型参数-generic-parameters)
        - [泛型类型与关联类型 Generic Types vs Associated Types](#泛型类型与关联类型-generic-types-vs-associated-types)
    - [作用域 Scope](#作用域-scope)
    - [衍生宏 Derive Macros](#衍生宏-derive-macros)
    - [默认实现 Default Impls](#默认实现-default-impls)
    - [一揽子泛型实现 Generic Blanket Impls](#一揽子泛型实现-generic-blanket-impls)
    - [子特性与超特性 Subtraits & Supertraits](#子特性与超特性-subtraits--supertraits)
    - [特性对象 Trait Objects](#特性对象-trait-objects)
    - [仅用于标记的特性 Marker Traits](#仅用于标记的特性-marker-traits)
    - [可自动实现的特性 Auto Traits](#可自动实现的特性-auto-traits)
    - [不安全的特性 Unsafe Traits](#不安全的特性-unsafe-traits)
- [可自动实现的特性 Auto Traits](#可自动实现的特性-auto-traits-1)
    - [Send & Sync](#send--sync)
    - [Sized](#sized)
- [常用特性 General Traits](#常用特性-general-traits)
    - [Default](#default)
    - [Clone](#clone)
    - [Copy](#copy)
    - [Any](#any)
- [文本格式化特性 Formatting Traits](#文本格式化特性-formatting-traits)
    - [Display & ToString](#display--tostring)
    - [Debug](#debug)
- [算符重载特性 Operator Traits](#算符重载特性-operator-traits)
    - [比较特性 Comparison Traits](#比较特性-comparison-traits)
        - [PartialEq & Eq](#partialeq--eq)
        - [Hash](#hash)
        - [PartialOrd & Ord](#partialord--ord)
    - [算术特性 Arithmetic Traits](#算术特性-arithmetic-traits)
        - [Add & AddAssign](#add--addassign)
    - [闭包特性 Closure Traits](#闭包特性-closure-traits)
        - [FnOnce, FnMut, & Fn](#fnonce-fnmut--fn)
    - [其它特性 Other Traits](#其它特性-other-traits)
        - [Deref & DerefMut](#deref--derefmut)
        - [Index & IndexMut](#index--indexmut)
        - [Drop](#drop)
- [转换特性 Conversion Traits](#转换特性-conversion-traits)
    - [From & Into](#from--into)
- [错误处理 Error Handling](#错误处理-error-handling)
    - [Error](#error)
- [转换特性深入 Conversion Traits Continued](#转换特性深入-conversion-traits-continued)
    - [TryFrom & TryInto](#tryfrom--tryinto)
    - [FromStr](#fromstr)
    - [AsRef & AsMut](#asref--asmut)
    - [Borrow & BorrowMut](#borrow--borrowmut)
    - [ToOwned](#toowned)
- [迭代特性 Iteration Traits](#迭代特性-iteration-traits)
    - [Iterator](#iterator)
    - [IntoIterator](#intoiterator)
    - [FromIterator](#fromiterator)
- [输入输出特性 I/O Traits](#输入输出特性-io-traits)
    - [Read & Write](#read--write)
- [结语 Conclusion](#结语-conclusion)
- [讨论 Discuss](#讨论-discuss)
- [更多资料 Further Reading](#更多资料-further-reading)
- [翻译 Translation](#翻译-translation)



## 引入 Intro

> Have you ever wondered what's the difference between:
> - `Deref<Target = T>`, `AsRef<T>`, and `Borrow<T>`?
> - `Clone`, `Copy`, and `ToOwned`?
> - `From<T>` and `Into<T>`?
> - `TryFrom<&str>` and `FromStr`?
> - `FnOnce`, `FnMut`, `Fn`, and `fn`?
>

你是否曾对以下特性的区别感到困惑：
- `Deref<Target = T>` ， `AsRef<T>` 和 `Borrow<T>`？
- `Clone` ， `Copy` 和 `ToOwned`？
- `From<T>` 和 `Into<T>`？
- `TryFrom<&str>` 和 `FromStr`？
- `FnOnce` ， `FnMut` ， `Fn` 和 `fn`？

> Or ever asked yourself the questions:
> - _"When do I use associated types vs generic types in my trait?"_
> - _"What are generic blanket impls?"_
> - _"How do subtraits and supertraits work?"_
> - _"Why does this trait not have any methods?"_
>

或者有这样的疑问：

- _“我应该在特性中使用关联类型还是泛型类型？”_
- _"什么是一揽子泛型实现？"_
- _"子特性与超特性是如何工作的？"_
- _"为什么某个特性没有实现任何方法？"_

> Well then this is the article for you! It answers all of the above questions and much much more. Together we'll do a quick flyby tour of all of the most popular and commonly used traits from the Rust standard library!
>

本文正是为你解答以上困惑而撰写！而且本文绝不仅仅只回答了以上问题。下面，我们将一起对 Rust 标准库中所有最流行、最常用的特性做一个走马观花般的概览！

> You can read this article in order section by section or jump around to whichever traits interest you the most because each trait section begins with a list of links to **Prerequisite** sections that you should read to have adequate context to understand the current section's explanations.
>

你可以按顺序阅读本文，也可以直接跳读至你最感兴趣的特性。每节都会提供**预备知识**列表，它会帮助你获得相应的背景知识，不必担心跳读带来的理解困难。



## 特性的基础知识 Trait Basics

> We'll cover just enough of the basics so that the rest of the article can be streamlined without having to repeat the same explanations of the same concepts over and over as they reappear in different traits.
>

本章覆盖了特性的基础知识，相应内容在以后的章节中不再赘述。



### 特性的记号 Trait Items

> Trait items are any items that are part of a trait declaration.

特性的记号指的是，在特性的声明中可使用的记号。



#### Self

> `Self` always refers to the implementing type.

`Self` 永远引用正被实现的类型。



```rust
trait Trait {
    // always returns i32
    // 总是返回 i32
    fn returns_num() -> i32;

    // returns implementing type
    // 总是返回正被实现的类型
    fn returns_self() -> Self;
}

struct SomeType;
struct OtherType;

impl Trait for SomeType {
    fn returns_num() -> i32 {
        5
    }

    // Self == SomeType
    fn returns_self() -> Self {
        SomeType
    }
}

impl Trait for OtherType {
    fn returns_num() -> i32 {
        6
    }

    // Self == OtherType
    fn returns_self() -> Self {
        OtherType
    }
}
```



#### 函数 Functions

> A trait function is any function whose first parameter does not use the `self` keyword.

特性的函数指的是，任何不以 `self` 关键字作为首参数的函数。

```rust
trait Default {
    // function
    // 函数
    fn default() -> Self;
}
```

> Trait functions can be called namespaced by the trait or implementing type:

特性的函数同时声明在特性本身以及具体实现类型的命名空间中。

```rust
fn main() {
    let zero: i32 = Default::default();
    let zero = i32::default();
}
```



#### 方法 Methods

> A trait method is any function whose first parameter uses the `self` keyword and is of type `Self`, `&Self`, `&mut Self`. The former types can also be wrapped with a `Box`, `Rc`, `Arc`, or `Pin`.

特性的方法指的是，任何以 `self` 关键字作为首参数的函数，其类型是 `Self` ， `&Self` 或 `&mut Self`。前者的类型也可以包裹在 `Box` ， `Rc` ， `Arc` 或 `Pin` 中。

```rust
trait Trait {
    // methods
    // 方法
    fn takes_self(self);
    fn takes_immut_self(&self);
    fn takes_mut_self(&mut self);

    // above methods desugared
    // 以上代码等价于
    fn takes_self(self: Self);
    fn takes_immut_self(self: &Self);
    fn takes_mut_self(self: &mut Self);
}

// example from standard library
// 来自于标准库的示例
trait ToString {
    fn to_string(&self) -> String;
}
```

> Methods can be called using the dot operator on the implementing type:

可以使用点算符在具体实现类型上调用方法：

```rust
fn main() {
    let five = 5.to_string();
}
```

> However, similarly to functions, they can also be called namespaced by the trait or implementing type:

并且，与函数相似地，方法也声明在特性本身以及具体实现类型的命名空间中。

```rust
fn main() {
    let five = ToString::to_string(&5);
    let five = i32::to_string(&5);
}
```



#### 关联类型 Associated Types

> A trait can have associated types. This is useful when we need to use some type other than `Self` within function signatures but would still like the type to be chosen by the implementer rather than being hardcoded in the trait declaration:

特性内部可以声明关联类型。当我们希望在特性函数的签名中使用某种 `Self` 以外的类型，又不希望硬编码这种类型，而是希望后来的实现该特性的程序员来选择该类型具体是什么的时候，关联类型会很有用。

```rust
trait Trait {
    type AssociatedType;
    fn func(arg: Self::AssociatedType);
}

struct SomeType;
struct OtherType;

// any type implementing Trait can
// choose the type of AssociatedType
// 我们可以在实现 Trait 特性的时候
// 再决定 AssociatedType 的具体类型
// 而不必是在声明 Trait 特性的时候

impl Trait for SomeType {
    type AssociatedType = i8; // chooses i8
    fn func(arg: Self::AssociatedType) {}
}

impl Trait for OtherType {
    type AssociatedType = u8; // chooses u8
    fn func(arg: Self::AssociatedType) {}
}

fn main() {
    SomeType::func(-1_i8); // can only call func with i8 on SomeType
    OtherType::func(1_u8); // can only call func with u8 on OtherType
                           // 同一特性实现在不同类型上时，可以具有不同的函数签名
}
```



#### 泛型参数 Generic Parameters

> _"Generic parameters"_ broadly refers to generic type parameters, generic lifetime parameters, and generic const parameters. Since all of those are a mouthful to say people commonly abbreviate them to _"generic types"_, _"lifetimes"_, and _"generic consts"_. Since generic consts are not used in any of the standard library traits we'll be covering they're outside the scope of this article.
>

“泛型参数” 是泛型类型参数、泛型寿命参数以及泛型常量参数的统称。由于这些术语过于佶屈聱牙，我们通常将他们缩略为“泛型类型”，“泛型寿命”和“泛型常量”。鉴于标准库中的特性无一采用泛型常量，本文也略过不讲。

> We can generalize a trait declaration using parameters:

我们可以使用以下参数来声明特性：

```rust
// trait declaration generalized with lifetime & type parameters
// 使用泛型寿命与泛型类型声明特性
trait Trait<'a, T> {
    // signature uses generic type
    // 在签名中使用泛型类型
    fn func1(arg: T);
    
    // signature uses lifetime
    // 在签名中使用泛型寿命
    fn func2(arg: &'a i32);
    
    // signature uses generic type & lifetime
    // 在签名中同时使用泛型类型与泛型寿命
    fn func3(arg: &'a T);
}

struct SomeType;

impl<'a> Trait<'a, i8> for SomeType {
    fn func1(arg: i8) {}
    fn func2(arg: &'a i32) {}
    fn func3(arg: &'a i8) {}
}

impl<'b> Trait<'b, u8> for SomeType {
    fn func1(arg: u8) {}
    fn func2(arg: &'b i32) {}
    fn func3(arg: &'b u8) {}
}
```

> It's possible to provide default values for generic types. The most commonly used default value is `Self` but any type works:

可以为泛型类型指定默认值，最常用的默认值是 `Self` ，此外任何其它类型都是可以的。

```rust
// make T = Self by default
// T 的默认值是 Self
trait Trait<T = Self> {
    fn func(t: T) {}
}

// any type can be used as the default
// 任何其它类型都可用作默认值
trait Trait2<T = i32> {
    fn func2(t: T) {}
}

struct SomeType;

// omitting the generic type will
// cause the impl to use the default
// value, which is Self here
// 省略泛型类型时， impl 块使用默认值，在这里是 Self
impl Trait for SomeType {
    fn func(t: SomeType) {}
}

// default value here is i32
// 这里的默认值是 i32
impl Trait2 for SomeType {
    fn func2(t: i32) {}
}

// the default is overridable as we'd expect
// 默认值可以被重写，正如我们希望的那样
impl Trait<String> for SomeType {
    fn func(t: String) {}
}

// overridable here too
// 这里也可以重写
impl Trait2<String> for SomeType {
    fn func2(t: String) {}
}
```

> Aside from parameterizing the trait it's also possible to parameterize individual functions and methods:

不仅可以为特性提供泛型，也可以独立地为函数或方法提供泛型。

```rust
trait Trait {
    fn func<'a, T>(t: &'a T);
}
```



#### 泛型类型与关联类型 Generic Types vs Associated Types

> Both generic types and associated types defer the decision to the implementer on which concrete types should be used in the trait's functions and methods, so this section seeks to explain when to use one over the other.

通过使用泛型类型与关联类型，我们都可以将具体类型的选择问题抛给后来实现该特性的程序员来决定，这一节将解释我们如何在相似的两者之间做出选择。

> The general rule-of-thumb is:
> - Use associated types when there should only be a single impl of the trait per type.
> - Use generic types when there can be many possible impls of the trait per type.
>

按照惯常的经验：
- 对于某一特性，每个类型仅应当有单一实现时，使用关联类型。
- 对于某一特性，每个类型可以有多个实现时，使用泛型类型。

> Let's say we want to define a trait called `Add` which allows us to add values together. Here's an initial design and impl that only uses associated types:

例如，我们声明一个 `Add` 特性，它允许将各值加总在一起。这是仅使用关联类型的初始设计：

```rust
trait Add {
    type Rhs;
    type Output;
    fn add(self, rhs: Self::Rhs) -> Self::Output;
}

struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Rhs = Point;
    type Output = Point;
    fn add(self, rhs: Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 1 };
    let p2 = Point { x: 2, y: 2 };
    let p3 = p1.add(p2);
    assert_eq!(p3.x, 3);
    assert_eq!(p3.y, 3);
}
```

> Let's say we wanted to add the ability to add `i32`s to `Point`s where the `i32` would be added to both the `x` and `y` members:

例如，我们希望程序允许将 i32 类型的值与 Point 类型的值相加，其规则是该 i32 类型的值分别加到成员 `x` 与成员 `y` 。

```rust
trait Add {
    type Rhs;
    type Output;
    fn add(self, rhs: Self::Rhs) -> Self::Output;
}

struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Rhs = Point;
    type Output = Point;
    fn add(self, rhs: Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

impl Add for Point { // ❌
    type Rhs = i32;
    type Output = Point;
    fn add(self, rhs: i32) -> Point {
        Point {
            x: self.x + rhs,
            y: self.y + rhs,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 1 };
    let p2 = Point { x: 2, y: 2 };
    let p3 = p1.add(p2);
    assert_eq!(p3.x, 3);
    assert_eq!(p3.y, 3);
    
    let p1 = Point { x: 1, y: 1 };
    let int2 = 2;
    let p3 = p1.add(int2); // ❌
    assert_eq!(p3.x, 3);
    assert_eq!(p3.y, 3);
}
```

> Throws:

编译出错：

```none
error[E0119]: conflicting implementations of trait `Add` for type `Point`:
  --> src/main.rs:23:1
   |
12 | impl Add for Point {
   | ------------------ first implementation here
...
23 | impl Add for Point {
   | ^^^^^^^^^^^^^^^^^^ conflicting implementation for `Point`
```

> Since the `Add` trait is not parameterized by any generic types we can only impl it once per type, which means we can only pick the types for both `Rhs` and `Output` once! To allow adding both `Points`s and `i32`s to `Point` we have to refactor `Rhs` from an associated type to a generic type, which would allow us to impl the trait multiple times for `Point` with different type arguments for `Rhs`:

由于 `Add` 特性未提供泛型类型，因而每个类型只能具有该特性的单一实现，这即是说一旦我们指定了 `Rhs` 和 `Output` 的类型后就不可再更改了！为了 Point 类型的值能同时接受 i32 类型和 Point 类型的值作为被加数，我们应当重构之以将 `Rhs` 从关联类型改为泛型类型，这将允许我们为 `Rhs` 指定不同的类型并为同一类型多次实现某一特性。

```rust
trait Add<Rhs> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}

struct Point {
    x: i32,
    y: i32,
}

impl Add<Point> for Point {
    type Output = Self;
    fn add(self, rhs: Point) -> Self::Output {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

impl Add<i32> for Point { // ✅
    type Output = Self;
    fn add(self, rhs: i32) -> Self::Output {
        Point {
            x: self.x + rhs,
            y: self.y + rhs,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 1 };
    let p2 = Point { x: 2, y: 2 };
    let p3 = p1.add(p2);
    assert_eq!(p3.x, 3);
    assert_eq!(p3.y, 3);
    
    let p1 = Point { x: 1, y: 1 };
    let int2 = 2;
    let p3 = p1.add(int2); // ✅
    assert_eq!(p3.x, 3);
    assert_eq!(p3.y, 3);
}
```

> Let's say we add a new type called `Line` which contains two `Point`s, and now there are contexts within our program where adding two `Point`s should produce a `Line` instead of a `Point`. This is not possible given the current design of the `Add` trait where `Output` is an associated type but we can satisfy these new requirements by refactoring `Output` from an associated type into a generic type:

例如，我们现在声明一个包含两个 `Point` 类型的新类型 `Line` ，要求当两个 `Point` 类型相加时返回 `Line` 而不是 `Point` 。在当前 `Add` 特性的设计中 `Output` 是关联类型，不能满足这一要求，重构之以将关联类型改为泛型类型：

```rust
trait Add<Rhs, Output> {
    fn add(self, rhs: Rhs) -> Output;
}

struct Point {
    x: i32,
    y: i32,
}

impl Add<Point, Point> for Point {
    fn add(self, rhs: Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

impl Add<i32, Point> for Point {
    fn add(self, rhs: i32) -> Point {
        Point {
            x: self.x + rhs,
            y: self.y + rhs,
        }
    }
}

struct Line {
    start: Point,
    end: Point,
}

impl Add<Point, Line> for Point { // ✅
    fn add(self, rhs: Point) -> Line {
        Line {
            start: self,
            end: rhs,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 1 };
    let p2 = Point { x: 2, y: 2 };
    let p3: Point = p1.add(p2);
    assert!(p3.x == 3 && p3.y == 3);

    let p1 = Point { x: 1, y: 1 };
    let int2 = 2;
    let p3 = p1.add(int2);
    assert!(p3.x == 3 && p3.y == 3);

    let p1 = Point { x: 1, y: 1 };
    let p2 = Point { x: 2, y: 2 };
    let l: Line = p1.add(p2); // ✅
    assert!(l.start.x == 1 && l.start.y == 1 && l.end.x == 2 && l.end.y == 2)
}
```

> So which `Add` trait above is the best? It really depends on the requirements of your program! They're all good in the right situations.

所以说，哪一种 `Add` 特性最好？答案是具体问题具体分析！不管白猫黑猫，会捉老鼠就是好猫。



### 作用域 Scope

> Trait items cannot be used unless the trait is in scope. Most Rustaceans learn this the hard way the first time they try to write a program that does anything with I/O because the `Read` and `Write` traits are not in the standard library prelude:

特性仅当被引入当前作用域时才可以使用。绝大多数的初学者要在编写 I/O 程序时经历一番痛苦挣扎后，才能领悟到这一点，原因是 `Read` 和 `Write` 两个特性并未包含在标准库的 prelude 模块中。

```rust
use std::fs::File;
use std::io;

fn main() -> Result<(), io::Error> {
    let mut file = File::open("Cargo.toml")?;
    let mut buffer = String::new();
    file.read_to_string(&mut buffer)?; // ❌ read_to_string not found in File
                                       // ❌ 当前文件中找不到 read_to_string
    Ok(())
}
```

> `read_to_string(buf: &mut String)` is declared by the `std::io::Read` trait and implemented by the `std::fs::File` struct but in order to call it `std::io::Read` must be in scope:

`read_to_string(buf: &mut String)` 声明于 `std::io::Read` 特性，并实现于 `std::fs::File` 类型，若要调用该函数还须得 `std::io::Read` 特性处于当前作用域中：

```rust
use std::fs::File;
use std::io;
use std::io::Read; // ✅

fn main() -> Result<(), io::Error> {
    let mut file = File::open("Cargo.toml")?;
    let mut buffer = String::new();
    file.read_to_string(&mut buffer)?; // ✅
    Ok(())
}
```

> The standard library prelude is a module in the standard library, i.e. `std::prelude::v1`, that gets auto imported at the top of every other module, i.e. `use std::prelude::v1::*`. Thus the following traits are always in scope and we never have to explicitly import them ourselves because they're part of the prelude:

诸如 `std::prelude::v1` ，prelude 是标准库的一类模块，其特点是该模块命名空间下的成员将被自动导入到任何其它模块的顶部，其作用等效于 `use std::prelude::v1::*` 。因此，以下 prelude 模块中的特性无需我们显式导入，它们永远存在于当前作用域：

- [AsMut](#asref--asmut)
- [AsRef](#asref--asmut)
- [Clone](#clone)
- [Copy](#copy)
- [Default](#default)
- [Drop](#drop)
- [Eq](#partialeq--eq)
- [Fn](#fnonce-fnmut--fn)
- [FnMut](#fnonce-fnmut--fn)
- [FnOnce](#fnonce-fnmut--fn)
- [From](#from--into)
- [Into](#from--into)
- [ToOwned](#toowned)
- [IntoIterator](#intoiterator)
- [Iterator](#iterator)
- [PartialEq](#partialeq--eq)
- [PartialOrd](#partialord--ord)
- [Send](#send--sync)
- [Sized](#sized)
- [Sync](#send--sync)
- [ToString](#display--tostring)
- [Ord](#partialord--ord)



### 衍生宏 Derive Macros

> The standard library exports a handful of derive macros which we can use to quickly and conveniently impl a trait on a type if all of its members also impl the trait. The derive macros are named after the traits they impl:

标准库导出了一系列实用的衍生宏，我们可以利用它们方便快捷地为特定类型实现某种特性，前提是该类型的成员亦实现了相应的特性。衍生宏与它们各自所实现的特性同名：

- [Clone](#clone)
- [Copy](#copy)
- [Debug](#debug)
- [Default](#default)
- [Eq](#partialeq--eq)
- [Hash](#hash)
- [Ord](#partialord--ord)
- [PartialEq](#partialeq--eq)
- [PartialOrd](#partialord--ord)

> Example usage:
>

用例：

```rust
// macro derives Copy & Clone impl for SomeType
// 利用宏的方式为特定类型衍生出 Copy 与 Clone 特性的具体实现
#[derive(Copy, Clone)]
struct SomeType;
```

> Note: derive macros are just procedural macros and can do anything, there's no hard rule that they must impl a trait or that they can only work if all the members of the type impl a trait, these are just the conventions followed by the derive macros in the standard library.
>

注意：衍生宏仅是一种机械的过程，宏展开之后发生的事情并无一定之规。并没有绝对的规定要求衍生宏展开之后必须要为类型实现某种特性，又或者它们必须要求该类型的所有成员都必须实现某种特性才能为当前类型实现该特性，这仅仅是在标准库衍生宏的编纂过程中逐渐约定俗成的规则。



### 默认实现 Default Impls

> Traits can provide default impls for their functions and methods.

特性可为函数与方法提供默认的实现。

```rust
trait Trait {
    fn method(&self) {
        println!("default impl");
    }
}

struct SomeType;
struct OtherType;

// use default impl for Trait::method
// 省略时使用默认实现
impl Trait for SomeType {}

impl Trait for OtherType {
    // use our own impl for Trait::method
    // 重写时覆盖默认实现
    fn method(&self) {
        println!("OtherType impl");
    }
}

fn main() {
    SomeType.method(); // prints "default impl"
    OtherType.method(); // prints "OtherType impl"
}
```

> This is especially handy if some of the trait methods can be implemented solely using other trait methods.

这对于实现特性中某些仅依赖于其它方法的方法来说极其方便。

```rust
trait Greet {
    fn greet(&self, name: &str) -> String;
    fn greet_loudly(&self, name: &str) -> String {
        self.greet(name) + "!"
    }
}

struct Hello;
struct Hola;

impl Greet for Hello {
    fn greet(&self, name: &str) -> String {
        format!("Hello {}", name)
    }
    // use default impl for greet_loudly
    // 省略时使用 greet_loudly 的默认实现
}

impl Greet for Hola {
    fn greet(&self, name: &str) -> String {
        format!("Hola {}", name)
    }
    // override default impl
    // 重写时覆盖 greet_loudly 的默认实现
    fn greet_loudly(&self, name: &str) -> String {
        let mut greeting = self.greet(name);
        greeting.insert_str(0, "¡");
        greeting + "!"
    }
}

fn main() {
    println!("{}", Hello.greet("John")); // prints "Hello John"
    println!("{}", Hello.greet_loudly("John")); // prints "Hello John!"
    println!("{}", Hola.greet("John")); // prints "Hola John"
    println!("{}", Hola.greet_loudly("John")); // prints "¡Hola John!"
}
```

> Many traits in the standard library provide default impls for many of their methods.

标准库中的许多特性都为它们的方法提供默认实现。



### 一揽子泛型实现 Generic Blanket Impls

> A generic blanket impl is an impl on a generic type instead of a concrete type. To explain why and how we'd use one let's start by writing an `is_even` method for number types:

一揽子泛型实现是对泛型类型的实现，与之对应的是对特定类型的实现。我们将以 is_even 方法为例说明如何对数字类型实现一揽子泛型实现。

```rust
trait Even {
    fn is_even(self) -> bool;
}

impl Even for i8 {
    fn is_even(self) -> bool {
        self % 2_i8 == 0_i8
    }
}

impl Even for u8 {
    fn is_even(self) -> bool {
        self % 2_u8 == 0_u8
    }
}

impl Even for i16 {
    fn is_even(self) -> bool {
        self % 2_i16 == 0_i16
    }
}

// etc

#[test] // ✅
fn test_is_even() {
    assert!(2_i8.is_even());
    assert!(4_u8.is_even());
    assert!(6_i16.is_even());
    // etc
}
```

> Obviously, this is very verbose. Also, all of our impls are almost identical. Furthermore, in the unlikely but still possible event that Rust decides to add more number types in the future we have to remember to come back to this code and update it with the new number types. We can solve all these problems using a generic blanket impl:

显而易见地，我们重复实现了近乎相同的逻辑，这非常的繁琐。进一步来讲，如果 Rust 在将来决定增加更多的数字类型（小概率事件并非绝不可能），那么我们将不得不重新回到这里对新增的数字类型编写代码。一揽子泛型实现恰可以解决这些问题：

```rust
use std::fmt::Debug;
use std::convert::TryInto;
use std::ops::Rem;

trait Even {
    fn is_even(self) -> bool;
}

// generic blanket impl
// 一揽子泛型实现
impl<T> Even for T
where
    T: Rem<Output = T> + PartialEq<T> + Sized,
    u8: TryInto<T>,
    <u8 as TryInto<T>>::Error: Debug,
{
    fn is_even(self) -> bool {
        // these unwraps will never panic
        // 以下 unwrap 永远不会 panic
        self % 2.try_into().unwrap() == 0.try_into().unwrap()
    }
}

#[test] // ✅
fn test_is_even() {
    assert!(2_i8.is_even());
    assert!(4_u8.is_even());
    assert!(6_i16.is_even());
    // etc
}
```

> Unlike default impls, which provide _an_ impl, generic blanket impls provide _the_ impl, so they are not overridable.

默认实现可以重写，而一揽子泛型实现不可重写。

```rust
use std::fmt::Debug;
use std::convert::TryInto;
use std::ops::Rem;

trait Even {
    fn is_even(self) -> bool;
}

impl<T> Even for T
where
    T: Rem<Output = T> + PartialEq<T> + Sized,
    u8: TryInto<T>,
    <u8 as TryInto<T>>::Error: Debug,
{
    fn is_even(self) -> bool {
        self % 2.try_into().unwrap() == 0.try_into().unwrap()
    }
}

impl Even for u8 { // ❌
    fn is_even(self) -> bool {
        self % 2_u8 == 0_u8
    }
}
```

> Throws:

编译出错：

```none
error[E0119]: conflicting implementations of trait `Even` for type `u8`:
  --> src/lib.rs:22:1
   |
10 | / impl<T> Even for T
11 | | where
12 | |     T: Rem<Output = T> + PartialEq<T> + Sized,
13 | |     u8: TryInto<T>,
...  |
19 | |     }
20 | | }
   | |_- first implementation here
21 | 
22 |   impl Even for u8 {
   |   ^^^^^^^^^^^^^^^^ conflicting implementation for `u8`
```

> These impls overlap, hence they conflict, hence Rust rejects the code to ensure trait coherence. Trait coherence is the property that there exists at most one impl of a trait for any given type. The rules Rust uses to enforce trait coherence, the implications of those rules, and workarounds for the implications are outside the scope of this article.

重叠的实现产生了冲突，于是 Rust 拒绝了该代码以确保特性一致性。特性一致性指的是，对任意给定类型，仅能对某一特性具有单一实现。Rust 强制实现特性一致性，而这一规则的潜在影响与变通方法超出了本文的讨论范围。



### 子特性与超特性 Subtraits & Supertraits

> The "sub" in "subtrait" refers to subset and the "super" in "supertrait" refers to superset. If we have this trait declaration:

子特性的“子”即为子集，超特性的“超”即为超集。若有下列特性声明：

```rust
trait Subtrait: Supertrait {}
```

> All of the types which impl `Subtrait` are a subset of all the types which impl `Supertrait`, or to put it in opposite but equivalent terms: all the types which impl `Supertrait` are a superset of all the types which impl `Subtrait`.

所有实现了子特性的类型都是实现了超特性的类型的子集，也可以说，所有实现了超特性的类型都是实现了子特性的类型的超集。

> Also, the above is just syntax sugar for:

以上代码等价于：

```rust
trait Subtrait where Self: Supertrait {}
```

> It's a subtle yet important distinction to understand that the bound is on `Self`, i.e. the type impling `Subtrait`, and not on `Subtrait` itself. The latter would not make any sense, since trait bounds can only be applied to concrete types which can impl traits. Traits cannot impl other traits:

这是一种易于忽略但又至关重要的区别 —— 约束是 `Self` 的约束，而不是 `Subtrait` 的约束。后者没有任何意义，因为特性约束只能应用于具体类型。不能用一种特性去实现其它特性：

```rust
trait Supertrait {
    fn method(&self) {
        println!("in supertrait");
    }
}

trait Subtrait: Supertrait {
    // this looks like it might impl or
    // override Supertrait::method but it
    // does not
    // 这可能会令你产生超特性的方法被覆盖的错觉（实际不会）
    fn method(&self) {
        println!("in subtrait")
    }
}


struct SomeType;

// adds Supertrait::method to SomeType
impl Supertrait for SomeType {}

// adds Subtrait::method to SomeType
impl Subtrait for SomeType {}

// both methods exist on SomeType simultaneously
// neither overriding or shadowing the other
// 两个同名方法同时存在于同一类型时，既不重写也不影射

fn main() {
    SomeType.method(); // ❌ ambiguous method call
                       // ❌ 不允许语义模糊的函数调用
    // must disambiguate using fully-qualified syntax
    // 必须使用完全限定的记号来明确你要使用的函数
    <SomeType as Supertrait>::method(&st); // ✅ prints "in supertrait"
    <SomeType as Subtrait>::method(&st); // ✅ prints "in subtrait"
}
```

> Furthermore, there are no rules for how a type must impl both a subtrait and a supertrait. It can use the methods from either in the impl of the other.

此外，对于特定类型如何同时实现子特性与超特性并没有规定。子、超特性之间的方法也可以相互调用。

```rust
trait Supertrait {
    fn super_method(&mut self);
}

trait Subtrait: Supertrait {
    fn sub_method(&mut self);
}

struct CallSuperFromSub;

impl Supertrait for CallSuperFromSub {
    fn super_method(&mut self) {
        println!("in super");
    }
}

impl Subtrait for CallSuperFromSub {
    fn sub_method(&mut self) {
        println!("in sub");
        self.super_method();
    }
}

struct CallSubFromSuper;

impl Supertrait for CallSubFromSuper {
    fn super_method(&mut self) {
        println!("in super");
        self.sub_method();
    }
}

impl Subtrait for CallSubFromSuper {
    fn sub_method(&mut self) {
        println!("in sub");
    }
}

struct CallEachOther(bool);

impl Supertrait for CallEachOther {
    fn super_method(&mut self) {
        println!("in super");
        if self.0 {
            self.0 = false;
            self.sub_method();
        }
    }
}

impl Subtrait for CallEachOther {
    fn sub_method(&mut self) {
        println!("in sub");
        if self.0 {
            self.0 = false;
            self.super_method();
        }
    }
}

fn main() {
    CallSuperFromSub.super_method(); // prints "in super"
    CallSuperFromSub.sub_method(); // prints "in sub", "in super"
    
    CallSubFromSuper.super_method(); // prints "in super", "in sub"
    CallSubFromSuper.sub_method(); // prints "in sub"
    
    CallEachOther(true).super_method(); // prints "in super", "in sub"
    CallEachOther(true).sub_method(); // prints "in sub", "in super"
}
```

> Hopefully the examples above show that the relationship between subtraits and supertraits can be complex. Before introducing a mental model that neatly encapsulates all of that complexity let's quickly review and establish the mental model we use for understanding trait bounds on generic types:

通过以上示例，希望读者能够领会到，子特性与超特性之间的关系并未被一刀切的限制住。接下来我们将学习一种将所有这些复杂性巧妙地封装在一起的心智模型，在这之前我们先来回顾一下我们用来理解泛型类型与特性约束的关系的心智模型。

```rust
fn function<T: Clone>(t: T) {
    // impl
}
```

> Without knowing anything about the impl of this function we could reasonably guess that `t.clone()` gets called at some point because when a generic type is bounded by a trait that strongly implies it has a dependency on the trait. The mental model for understanding the relationship between generic types and their trait bounds is a simple and intuitive one: generic types _depend on_ their trait bounds.

即便我们不知道这个函数的具体实现，我们仍旧可以有理有据地猜测 `t.clone()` 将在函数的某处被调用，因为当泛型类型被特性所约束的时候，会给人一种它依赖于该特性的强烈暗示。这就是一种理解泛型类型与特性约束的关系的心智模型，它简单且可凭直觉 —— 泛型类型依赖于它们的特性约束。

> Now let's look the trait declaration for `Copy`:

现在，让我们看看 `Copy` 特性的声明：

```rust
trait Copy: Clone {}
```

> The syntax above looks very similar to the syntax for applying a trait bound on a generic type and yet `Copy` doesn't depend on `Clone` at all. The mental model we developed earlier doesn't help us here. In my opinion, the most simple and elegant mental model for understanding the relationship between subtraits and supertraits is: subtraits _refine_ their supertraits.

以上的记号和之前我们为泛型添加特性约束的记号非常相似，但是 `Copy` 却完全不依赖 `Clone` 。早前建立的心智模型现在不适用了。在我看来，理解子特性与超特性的关系的最简单和最优雅的心智模型莫过于 —— 子特性 *改良* 了超特性。

> "Refinement" is intentionally kept somewhat vague because it can mean different things in different contexts:
> - a subtrait might make its supertrait's methods' impls more specialized, faster, use less memory, e.g. `Copy: Clone`
> - a subtrait might make additional guarantees about the supertrait's methods' impls, e.g. `Eq: PartialEq`, `Ord: PartialOrd`, `ExactSizeIterator: Iterator`
> - a subtrait might make the supertrait's methods more flexible or easier to call, e.g. `FnMut: FnOnce`, `Fn: FnMut`
> - a subtrait might extend a supertrait and add new methods, e.g. `DoubleEndedIterator: Iterator`, `ExactSizeIterator: Iterator`
>

“改良”一词故意地预留了一些模糊的空间，它的具体含义在不同的上下文中有所不同：

- 子特性可能比超特性的方法更加特异化、运行更快或使用更少内存等等，例如 `Copy: Clone` 
- 子特性可能比超特性的方法具有额外的功能，例如 `Eq: PartialEq` ， `Ord: PartialOrd` 和 `ExactSizeIterator: Iterator` 
- 子特性可能比超特性的方法更灵活和更易于调用，例如 `FnMut: FnOnce` 和 `Fn: FnMut`
- 子特性可能扩展了超特性并添加了新的方法，例如 `DoubleEndedIterator: Iterator` 和 `ExactSizeIterator: Iterator` 

### 特性对象 Trait Objects

> Generics give us compile-time polymorphism where trait objects give us run-time polymorphism. We can use trait objects to allow functions to dynamically return different types at run-time:

如果说泛型给了我们编译时的多态性，那么特性对象就给了我们运行时的多态性。通过特性对象，我们可以允许函数在运行时动态地返回不同的类型。

```rust
fn example(condition: bool, vec: Vec<i32>) -> Box<dyn Iterator<Item = i32>> {
    let iter = vec.into_iter();
    if condition {
        // Has type:
        // Box<Map<IntoIter<i32>, Fn(i32) -> i32>>
        // But is cast to:
        // Box<dyn Iterator<Item = i32>>
        Box::new(iter.map(|n| n * 2))
    } else {
        // Has type:
        // Box<Filter<IntoIter<i32>, Fn(&i32) -> bool>>
        // But is cast to:
        // Box<dyn Iterator<Item = i32>>
        Box::new(iter.filter(|&n| n >= 2))
    }
}
        // 以上代码中，两种不同的指针类型转换成相同的指针类型
```

> Trait objects also allow us to store heterogeneous types in collections:

特性对象也允许我们在集合中存储不同类型的值：

```rust
use std::f64::consts::PI;

struct Circle {
    radius: f64,
}

struct Square {
    side: f64
}

trait Shape {
    fn area(&self) -> f64;
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        PI * self.radius * self.radius
    }
}

impl Shape for Square {
    fn area(&self) -> f64 {
        self.side * self.side
    }
}

fn get_total_area(shapes: Vec<Box<dyn Shape>>) -> f64 {
    shapes.into_iter().map(|s| s.area()).sum()
}

fn example() {
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle { radius: 1.0 }), // Box<Circle> cast to Box<dyn Shape>
        Box::new(Square { side: 1.0 }), // Box<Square> cast to Box<dyn Shape>
    ];
    assert_eq!(PI + 1.0, get_total_area(shapes)); // ✅
}
```

> Trait objects are unsized so they must always be behind a pointer. We can tell the difference between a concrete type and a trait object at the type level based on the presence of the `dyn` keyword within the type:

特性对象的结构体大小是未知的，所以必须要通过指针来引用它们。具体类型与特性对象在字面上的区别在于，特性对象必须要用 `dyn` 关键字来修饰前缀，了解了这一点我们可以轻松辨别二者。

```rust
struct Struct;
trait Trait {}

// regular struct
// 这是一般的结构
&Struct
Box<Struct>
Rc<Struct>
Arc<Struct>

// trait objects
// 这是特性对象
&dyn Trait
Box<dyn Trait>
Rc<dyn Trait>
Arc<dyn Trait>
```

> Not all traits can be converted into trait objects. A trait is object-safe if it meets these requirements:
> - trait doesn't require `Self: Sized`
> - all of the trait's methods are object-safe
>

并非全部的特性都可以转换为特性对象，一个 “对象安全” 的特性必须满足：

- 该特性不要求 `Self: Sized`
- 该特性的所有方法都是 “对象安全” 的

> A trait method is object-safe if it meets these requirements:
>
> - method requires `Self: Sized` or
> - method only uses a `Self` type in receiver position
>

一个特性的方法若要是 “对象安全” 的，必须满足：

- 该方法要求 `Self: Sized` 
- 该方法仅在接收参数中使用 `Self` 类型

Understanding why the requirements are what they are is not relevant to the rest of this article, but if you're still curious it's covered in [Sizedness in Rust](../../sizedness-in-rust.md).

关于具有这些限制条件的原因超出了本文的讨论范围且与下文无关，如果你对此深感兴趣不妨阅读 [Sizedness in Rust](../../sizedness-in-rust.md) 以了解详情。



### 仅用于标记的特性 Marker Traits

> Marker traits are traits that have no trait items. Their job is to "mark" the implementing type as having some property which is otherwise not possible to represent using the type system.

仅用于标记的特性，即是某种声明体为空的特性。它们存在的意义在于 “标记” 所实现的类型，且该类型具有某种类型系统所无法表达的属性。

```rust
// Impling PartialEq for a type promises
// that equality for the type has these properties:
// - symmetry: a == b implies b == a, and
// - transitivity: a == b && b == c implies a == c
// But DOES NOT promise this property:
// - reflexivity: a == a
// 为特定类型实现 PartialEq 特性确保了该类型的相等算符具有以下性质：
// - 对称性： 若有 a == b ， 则必有 b == a
// - 传递性： 若有 a == b 和 b == c ， 则必有 a == c
// 但是不能确保具有以下性质：
// - 自反性： a == a
trait PartialEq {
    fn eq(&self, other: &Self) -> bool;
}

// Eq has no trait items! The eq method is already
// declared by PartialEq, but "impling" Eq
// for a type promises this additional equality property:
// - reflexivity: a == a
// Eq 特性的声明体是空的！ 而 eq 方法已经被 PartialEq 所声明，
// 但是对特定类型“实现” Eq 特性确保了额外的相等性质：
// - 自反性： a == a
trait Eq: PartialEq {}

// f64 impls PartialEq but not Eq because NaN != NaN
// i32 impls PartialEq & Eq because there's no NaNs :)
// f64 实现了 PartialEq 特性但是没有实现 Eq 特性，因为 NaN != NaN
// i32 同时实现了 PartialEq 特性与 Eq 特性，因为没有 NaN 来捣乱 :)
```



### 可自动实现的特性 Auto Traits

> Auto traits are traits that get automatically implemented for a type if all of its members also impl the trait. What "members" means depends on the type, for example: fields of a struct, variants of an enum, elements of an array, items of a tuple, and so on.

可自动实现的特性指的是，存在这样一种特性，若给定类型的成员都实现了该特性，那么该类型就隐式地自动实现该特性。这里所说的 “成员” 依据上下文而具有不同的含义，包括而又不限于结构体的字段、枚举的变量、数组的元素和元组的内容等等。

> All auto traits are marker traits but not all marker traits are auto traits. Auto traits must be marker traits so the compiler can provide an automatic default impl for them, which would not be possible if they had any trait items.

所有可自动实现的特性都是仅用于标记的特性，反之则不是。正是由于可自动实现的特性必须是仅用于标记的特性，所以编译器才能够自动为其提供一个默认实现，反之编译器就无能为力了。

> Examples of auto traits:

可自动实现的特性的示例：

```rust
// implemented for types which are safe to send between threads
// 实现 Send 特性的类型可以安全地往返于多个线程
unsafe auto trait Send {}

// implemented for types whose references are safe to send between threads
// 实现 Sync 特性的类型，其引用可以安全地往返于多个线程
unsafe auto trait Sync {}
```



### 不安全的特性 Unsafe Traits

> Traits can be marked unsafe to indicate that impling the trait might require unsafe code. Both `Send` and `Sync` are marked `unsafe` because if they aren't automatically implemented for a type that means it must contains some non-`Send` or non-`Sync` member and we have to take extra care as the implementers to make sure there are no data races if we want to manually mark the type as `Send` and `Sync`.

以 `unsafe` 修饰前缀的特性，意味着该特性的实现可能需要不安全的代码。`Send` 特性与 `Sync` 特性以 `unsafe` 修饰前缀意味着，如果特定类型没有自动实现该特性，那么说明该类型的成员并非都实现了该特性，这提示着我们手动实现该特性一定要谨慎小心，以确保没有发生数据竞争。

```rust
// SomeType is not Send or Sync
// SomeType 没有实现 Send 和 Sync
struct SomeType {
    not_send_or_sync: *const (),
}

// but if we're confident that our impl doesn't have any data
// races we can explicitly mark it as Send and Sync using unsafe
// 倘若我们得以社会主义伟大成就的庇佑自信地写出没有数据竞争的代码
// 可以使用 unsafe 来修饰前缀，以显式地实现 Send 特性与 Sync 特性
unsafe impl Send for SomeType {}
unsafe impl Sync for SomeType {}
```



## 可自动实现的特性 Auto Traits



### Send & Sync

预备知识
- [Marker Traits](#marker-traits)
- [Auto Traits](#auto-traits)
- [Unsafe Traits](#unsafe-traits)

```rust
unsafe auto trait Send {}
unsafe auto trait Sync {}
```

> If a type is `Send` that means it's safe to send between threads. If a type is `Sync` that means it's safe to share references of it between threads. In more precise terms some type `T` is `Sync` if and only if `&T` is `Send`.

实现 `Send` 特性的类型可以安全地往返于多线程。实现 `sync` 特性的类型，其引用可以安全地往返于多线程。用更加准确的术语来讲，当且仅当 `&T` 实现 `Send` 特性时，`T` 才能实现 `Sync` 特性。

> Almost all types are `Send` and `Sync`. The only notable `Send` exception is `Rc` and the only notable `Sync` exceptions are `Rc`, `Cell`, `RefCell`. If we need a `Send` version of `Rc` we can use `Arc`. If we need a `Sync` version of `Cell` or `RefCell` we can `Mutex` or `RwLock`. Although if we're using the `Mutex` or `RwLock` to just wrap a primitive type it's often better to use the atomic primitive types provided by the standard library such as `AtomicBool`, `AtomicI32`, `AtomicUsize`, and so on.

几乎所有类型都实现了 `Send` 特性和 `Sync` 特性。对于 `Send` 唯一需要注意的例外是 `Rc` ，对于 `Sync` 唯三需要注意的例外是 `Rc`，`Cell` 和 `RefCell` 。如果我们需要 `Send` 版的 `Rc` ，可以使用 `Arc` 。如果我们需要 `Sync` 版的 `Cell` 或 `RefCell` ，可以使用 `Mutex` 或 `RwLock` 。尽管我们可以使用 `Mutex` 或 `RwLock` 来包裹住原语类型，但通常使用标准库提供的原子原语类型会更好，诸如 `AtomicBool` ，`AtomicI32` 和 `AtomicUsize` 等等。

> That almost all types are `Sync` might be a surprise to some people, but yup, it's true even for types without any internal synchronization. This is possible thanks to Rust's strict borrowing rules.

多亏了 Rust 严格的借用规则，几乎所有的类型都是 `Sync` 的。这对于一些人来讲可能会很惊讶，但事实胜于雄辩，甚至对于那些没有内部同步机制的类型来说也是如此。

> We can pass many immutable references to the same data to many threads and we're guaranteed there are no data races because as long as any immutable references exist Rust statically guarantees the underlying data cannot be mutated:

对于同一数据，我们可以放心地将该数据的多个不可变引用传递给多个线程，因为只要当前存在一个该数据的不可变引用，那么 Rust 就会静态地确保该数据不会被改变：

```rust
use crossbeam::thread;

fn main() {
    let mut greeting = String::from("Hello");
    let greeting_ref = &greeting;
    
    thread::scope(|scoped_thread| {
        // spawn 3 threads
        // 产生三个线程
        for n in 1..=3 {
            // greeting_ref copied into every thread
            // greeting_ref 被拷贝到每个线程
            scoped_thread.spawn(move |_| {
                println!("{} {}", greeting_ref, n); // prints "Hello {n}"
            });
        }
        
        // line below could cause UB or data races but compiler rejects it
        // 下面这行代码可能导致数据竞争，于是编译器拒绝了它
        greeting += " world"; 
        // ❌ cannot mutate greeting while immutable refs exist
        // ❌ 当不可变引用存在时，不可以修改引用的数据
    });
    
    // can mutate greeting after every thread has joined
    // 当所有的线程结束之后，可以修改数据
    greeting += " world"; // ✅
    println!("{}", greeting); // prints "Hello world"
}
```

> Likewise we can pass a single mutable reference to some data to a single thread and we're guaranteed there will be no data races because Rust statically guarantees aliased mutable references cannot exist and the underlying data cannot be mutated through anything other than the single existing mutable reference:

同样地，我们可以将某个数据的单个可变引用传递给单个线程，在此过程中不必担心出现数据竞争，因为 Rust 静态地确保了不存在其它可变引用。以下数据即仅可通过已经存在的单个可变引用而改变：

```rust
use crossbeam::thread;

fn main() {
    let mut greeting = String::from("Hello");
    let greeting_ref = &mut greeting;
    
    thread::scope(|scoped_thread| {
        // greeting_ref moved into thread
        // greeting_ref 移动到当前线程
        scoped_thread.spawn(move |_| {
            *greeting_ref += " world";
            println!("{}", greeting_ref); // prints "Hello world"
        });
        
        // line below could cause UB or data races but compiler rejects it
        // 下面这行代码可能导致数据竞争，于是编译器拒绝了它
        greeting += "!!!"; 
        // ❌ cannot mutate greeting while mutable refs exist
        // ❌ 可变引用存在时不可改变数据
    });
    
    // can mutate greeting after the thread has joined
    // 当所有的线程结束之后，可以修改数据
    greeting += "!!!"; // ✅
    println!("{}", greeting); // prints "Hello world!!!"
}
```

> This is why most types are `Sync` without requiring any explicit synchronization. In the event we need to simultaneously mutate some data `T` across multiple threads the compiler won't let us until we wrap the data in a `Arc<Mutex<T>>` or `Arc<RwLock<T>>` so the compiler enforces that explicit synchronization is used when it's needed.
>

这就是为什么绝大多数的类型都是 Sync 的而不需要实现任何显式的同步机制。对于数据 T ，如果我们试图从多个线程同时修改的话，编译器会对我们作出警告，除非我们将数据包裹在 `Arc<Mutex<T>>` 或 `Arc<RwLock<T>>` 中。所以说，当我们真的需要显式的同步机制时，编译器会强制要求我们这样做的。



### Sized

预备知识
- [Marker Traits](#marker-traits)
- [Auto Traits](#auto-traits)

> If a type is `Sized` that means its size in bytes is known at compile-time and it's possible to put instances of the type on the stack.

如果一个类型实现了 `Sized` ，那么说明该类型具体大小的字节数在编译时可以确定，并且也就说明该类型的实例可以存放在栈上。

> Sizedness of types and its implications is a subtle yet huge topic that affects a lot of different aspects of the language. It's so important that I wrote an entire article on it called [Sizedness in Rust](../../sizedness-in-rust.md) which I highly recommend reading for anyone who would like to understand sizedness in-depth. I'll summarize a few key things which are relevant to this article.

类型的大小以及其所带来的潜在影响，是一个易于忽略但是又十分宏大的话题，它深刻地影响着本门语言的诸多方面。鉴于它的重要性，我已经写了一整篇文章（[Sizedness in Rust](../../sizedness-in-rust.md)）来具体阐述其内容，我高度推荐对于希望深入 sizedness 的人阅读此篇文章。下面是此篇文章的要点：

> 1. All generic types get an implicit `Sized` bound.

1. 所有的泛型类型都具有隐式的 `Sized` 约束。

```rust
fn func<T>(t: &T) {}

// example above desugared
// 以上代码等价于
fn func<T: Sized>(t: &T) {}
```

> 2. Since there's an implicit `Sized` bound on all generic types, if we want to opt-out of this implicit bound we need to use the special _"relaxed bound"_ syntax `?Sized` which currently only exists for the `Sized` trait:

2. 由于所有的泛型类型都具有隐式的 `Sized` 约束，如果我们希望摆脱这样的隐式约束，那么我们需要使用特殊的 *“宽松约束”* 记号 `?Sized` ，目前这样的记号仅适用于 `Sized` 特性：

```rust
// now T can be unsized
// 现在 T 的大小可以是未知的
fn func<T: ?Sized>(t: &T) {}
```

> 3. There's an implicit `?Sized` bound on all traits.

3. 所有的特性都具有隐式的 `?Sized` 约束。

```rust
trait Trait {}

// example above desugared
// 以上代码等价于
trait Trait: ?Sized {}
```

> This is so that trait objects can impl the trait. Again, all of the nitty gritty details are in [Sizedness in Rust](../../sizedness-in-rust.md).

这就是为什么特性对象可以实现具体特性。再次，向您推荐关于一切真相的[Sizedness in Rust](../../sizedness-in-rust.md)。



## 常用特性 General traits



### Default

预备知识
- [Self](#self)
- [Functions](#functions)
- [Derive Macros](#derive-macros)

```rust
trait Default {
    fn default() -> Self;
}
```

> It's possible to construct default values of `Default` types.

为特定类型实现 `Default` 特性时，即为该类型赋予了可选的默认值。

```rust
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

impl Default for Color {
    // default color is black
    // 默认颜色是黑色
    fn default() -> Self {
        Color {
            r: 0,
            g: 0,
            b: 0,
        }
    }
}
```

> This is useful for quick prototyping but also in any instance where we just need an instance of a type and aren't picky about what it is:

这不仅利于快速原型设计，另外，在有时我们仅仅只是需要该类型的一个值，却完全不在意该值是什么的时候，这也非常方便。

```rust
fn main() {
    // just give me some color!
    let color = Color::default();
}
```

> This is also an option we may want to explicitly expose to the users of our functions:

如此，我们可以明确地向该函数的用户传达出该函数某个参数的可选择性：

```rust
struct Canvas;
enum Shape {
    Circle,
    Rectangle,
}

impl Canvas {
    // let user optionally pass a color
    // 用户可选地传入一个 color
    fn paint(&mut self, shape: Shape, color: Option<Color>) {
        // if no color is passed use the default color
        // 若用户没有传入 color ，即使用默认的 color
        let color = color.unwrap_or_default();
        // etc
    }
}
```

> `Default` is also useful in generic contexts where we need to construct generic types:

在泛型编程的语境中，`Default` 特性也可显其威力。

```rust
fn guarantee_length<T: Default>(mut vec: Vec<T>, min_len: usize) -> Vec<T> {
    for _ in 0..min_len.saturating_sub(vec.len()) {
        vec.push(T::default());
    }
    vec
}
```

> Another way we can take advantage of `Default` types is for partial initialization of structs using Rust's struct update syntax. We may have a `new` constructor for `Color` which takes every member as an argument:

另外，我们在使用 update 记号构造结构体时也可享受到 `Default` 特性带来的便利。我们以 `Color` 结构的 `new` 构造器函数为例，它接受该结构的全部成员作为参数：

```rust
impl Color {
    fn new(r: u8, g: u8, b: u8) -> Self {
        Color {
            r,
            g,
            b,
        }
    }
}
```

> However we can also have convenience constructors that only accept a particular struct member each and fall back to the default values for the other struct members:

考虑以下更加便捷的构造器函数 —— 它仅接受该结构的部分成员作为参数，其它未指定的成员则回落到默认值：

```rust
impl Color {
    fn red(r: u8) -> Self {
        Color {
            r,
            ..Color::default()
        }
    }
    fn green(g: u8) -> Self {
        Color {
            g,
            ..Color::default()
        }
    }
    fn blue(b: u8) -> Self {
        Color {
            b,
            ..Color::default()
        }
    }
}
```

> There's also a `Default` derive macro for so we can write `Color` like this:

`Default` 特性也可以用衍生宏的方式来实现：

```rust
// default color is still black
// because u8::default() == 0
// 默认颜色仍旧是黑色
// 因为 u8::default() == 0
#[derive(Default)]
struct Color {
    r: u8,
    g: u8,
    b: u8
}
```



### Clone

预备知识
- [Self](#self)
- [Methods](#methods)
- [Default Impls](#default-impls)
- [Derive Macros](#derive-macros)

```rust
trait Clone {
    fn clone(&self) -> Self;

    // provided default impls
    // 提供默认实现
    fn clone_from(&mut self, source: &Self);
}
```

> We can convert immutable references of `Clone` types into owned values, i.e. `&T` -> `T`. `Clone` makes no promises about the efficiency of this conversion so it can be slow and expensive. To quickly impl `Clone` on a type we can use the derive macro:

对于实现了 `Clone` 特性的类型，我们可以将一个不可变的引用转换为自有的类型，比如 `&T` -> `T` 。`Clone` 特性对于这种转换的效率不做出保证，所以这样的转换速度可能很慢，代价可能很昂贵。

```rust
#[derive(Clone)]
struct SomeType {
    cloneable_member1: CloneableType1,
    cloneable_member2: CloneableType2,
    // etc
}

// macro generates impl below
// 宏展开后为
impl Clone for SomeType {
    fn clone(&self) -> Self {
        SomeType {
            cloneable_member1: self.cloneable_member1.clone(),
            cloneable_member2: self.cloneable_member2.clone(),
            // etc
        }
    }
}
```

> `Clone` can also be useful in constructing instances of a type within a generic context. Here's an example from the previous section except using `Clone` instead of `Default`:

`Clone` 特性也有利于在泛型编程的语境中构造类型。请看下例：

```rust
fn guarantee_length<T: Clone>(mut vec: Vec<T>, min_len: usize, fill_with: &T) -> Vec<T> {
    for _ in 0..min_len.saturating_sub(vec.len()) {
        vec.push(fill_with.clone());
    }
    vec
}
```

> People also commonly use cloning as an escape hatch to avoid dealing with the borrow checker. Managing structs with references can be challenging, but we can turn the references into owned values by cloning them.

克隆确是一个可以逃避借用检查器的好方法。倘若我们编写的代码无法通过借用检查，那么不妨通过克隆将这些引用转换为自有类型。

```rust
// oof, we gotta worry about lifetimes 😟
// 糟糕！我们真的有自信处理好 lifetime 吗？ 😟
struct SomeStruct<'a> {
    data: &'a Vec<u8>,
}

// now we're on easy street 😎
// 好耶！人生苦短，我用 Clone ! 😎
struct SomeStruct {
    data: Vec<u8>,
}
```

> If we're working on a program where performance is not the utmost concern then we don't need to sweat cloning data. Rust is a low-level language that exposes a lot of low-level details so it's easy to get caught up in premature optimizations instead of actually solving the problem at hand. For many programs the best order of priorities is usually to build for correctness first, elegance second, and performance third, and only focus on performance after the program has been profiled and the performance bottlenecks have been identified. This is good general advice to follow, and if it doesn't apply to your particular program then you would know.

如果性能因素微不足道，我们不必羞于使用克隆。Rust 是一门底层语言，人们可以自由地控制程序行为的方方面面，这就很容易令人陷入盲目追求优化的陷阱，而不是专注于着手解决问题。对此我给出的建议是：正确第一，优雅第二，性能第三。只有程序初具雏形后，性能瓶颈的问题才可能凸显，这时我们再解决性能问题也不迟。与其说这是一条编程建议，更不如说这是一条人生建议，万事万物大抵如此，如果你现在不信，总有一天你会的。



### Copy

预备知识
- [Marker Traits](#marker-traits)
- [Subtraits & Supertraits](#subtraits--supertraits)
- [Derive Macros](#derive-macros)

```rust
trait Copy: Clone {}
```

> We copy `Copy` types, e.g. `T` -> `T`. `Copy` promises the copy operation will be a simple bitwise copy so it will be very fast and efficient. We cannot impl `Copy` ourselves, only the compiler can provide an impl, but we can tell it to do so by using the `Copy` derive macro, together with the `Clone` derive macro since `Copy` is a subtrait of `Clone`:

对于实现了 `Copy` 特性的类型，我们可以拷贝它，即 `T` -> `T` 。`Copy` 特性确保了拷贝操作是按位的拷贝，所以它更快更高效。`Copy` 特性不可手动实现，必须由编译器提供其实现。注意：当使用衍生宏为类型实现 `Copy` 特性时，必须同时使用 `Clone` 衍生宏，因为 `Copy` 是 `Clone` 的子特性：

```rust
#[derive(Copy, Clone)]
struct SomeType;
```

> `Copy` refines `Clone`. A clone may be slow and expensive but a copy is guaranteed to be fast and cheap, so a copy is just a fast clone. If a type impls `Copy` that makes the `Clone` impl trivial:

`Copy` 改良了 `Clone` 。克隆操作可能速度缓慢且代价昂贵，但是拷贝操作一定是高效低耗的，可以说拷贝就是一种物美价廉的克隆。`Copy` 特性的实现会令 `Clone` 特性的实现变得微不足道：

```rust
// this is what the derive macro generates
// 衍生宏展开如下
impl<T: Copy> Clone for T {
    // the clone method becomes just a copy
    // 克隆实际上变成了一种拷贝
    fn clone(&self) -> Self {
        *self
    }
}
```

> Impling `Copy` for a type changes its behavior when it gets moved. By default all types have _move semantics_ but once a type impls `Copy` it gets _copy semantics_. To explain the difference between the two let's examine these simple scenarios:

实现了 `Copy` 特性的类型，其在移动时的行为会发生变化。默认情况下，所有的类型都具有 _移动语义_ ，但是一旦该类型实现了 `Copy` 特性，则会变为 _拷贝语义_。 请考虑下例中语义的不同：

```rust
// a "move", src: !Copy
// 移动语义，src 没有实现 Copy 特性
let dest = src; 

// a "copy", src: Copy
// 拷贝语义，src 实现 Copy 特性
let dest = src;
```
> In both cases, `dest = src` performs a simple bitwise copy of `src`'s contents and moves the result into `dest`, the only difference is that in the case of _"a move"_ the borrow checker invalidates the `src` variable and makes sure it's not used anywhere else later and in the case of _"a copy"_ `src` remains valid and usable.

事实上，这两种语义背后执行的操作是完全相同的，都是将 `src` 按位复制到 `dest` 。其不同在于，在移动语义下，借用检查器从此吊销了 `src` 的可用性，而在拷贝语义下，`src` 保持可用。

> In a nutshell: Copies _are_ moves. Moves _are_ copies. The only difference is how they're treated by the borrow checker.

言而总之，拷贝就是移动，移动就是拷贝。它们在底层毫无二致，仅仅是借用检查器对待它们的方式不同。

> For a more concrete example of a move, imagine `src` was a `Vec<i32>` and its contents looked something like this:

对于移动行为来讲更具体的例子 —— 你可以将 `src` 想象为一个 `Vec<i32>`，它的结构体大致如下：

```rust
{ data: *mut [i32], length: usize, capacity: usize }
```

> When we write `dest = src` we end up with:

执行 `desc = src` 的结果如下：

```rust
src = { data: *mut [i32], length: usize, capacity: usize }
dest = { data: *mut [i32], length: usize, capacity: usize }
```

> At this point both `src` and `dest` have aliased mutable references to the same data, which is a big no-no, so the borrow checker invalidates the `src` variable so it can't be used again without throwing a compile error.

此时 `src` 和 `dest` 就都是同一数据的可变引用了，这可就糟tm的大糕了，所以借用检查器就吊销了 `src` 的可用性，一旦再次使用 `src` 就会引发编译错误。

> For a more concrete example of a copy, imagine `src` was an `Option<i32>` and its contents looked something like this:

对于拷贝行为来讲更具体的例子 —— 你可以将 `src` 想象为一个 `Option<i32>` ，它的结构体大致如下：

```rust
{ is_valid: bool, data: i32 }
```

> Now when we write `dest = src` we end up with:

执行 `desc = src` 的结果如下：

```rust
src = { is_valid: bool, data: i32 }
dest = { is_valid: bool, data: i32 }
```

> These are both usable simultaneously! Hence `Option<i32>` is `Copy`.

此时两者同时可用！因为 `Option<i32>` 实现了 `Copy` 。

> Although `Copy` could be an auto trait the Rust language designers decided it's simpler and safer for types to explicitly opt into copy semantics rather than silently inheriting copy semantics whenever the type is eligible, as the latter can cause surprising confusing behavior which often leads to bugs.

或许你已经注意到，令 `Copy` 特性成为可自动实现的特性在理论上是可行的。但是 Rust 语言的设计者认为，比之于在恰当时隐式地继承拷贝语义，显示地声明为拷贝语义更加的简单和安全。前者可能会导致 Rust 语言产生十分反人类的行为，也更容易出现 bug 。



### Any

预备知识
- [Self](#self)
- [Generic Blanket Impls](#generic-blanket-impls)
- [Subtraits & Supertraits](#subtraits--supertraits)
- [Trait Objects](#trait-objects)

```rust
trait Any: 'static {
    fn type_id(&self) -> TypeId;
}
```

> Rust's style of polymorphism is parametric, but if we're looking to use a more ad-hoc style of polymorphism similar to dynamically-typed languages then we can emulate that using the `Any` trait. We don't have to manually impl this trait for our types because that's already covered by this generic blanket impl:

Rust 的多态性风格本身是参数化的，但如果我们希望临时使用一种更贴近于动态语言的多态性风格，可以借用 `Any` 特性来模拟。我们不需要手动实现 `Any` 特性，因为该特性通常由一揽子泛型实现所实现。

```rust
impl<T: 'static + ?Sized> Any for T {
    fn type_id(&self) -> TypeId {
        TypeId::of::<T>()
    }
}
```

> The way we get a `T` out of a `dyn Any` is by using the `downcast_ref::<T>()` and `downcast_mut::<T>()` methods:

对于 `dyn Any` 的特性对象，我们可以使用 `downcast_ref::<T>()` 或 `downcast_mut::<T>()` 来尝试解析出 `T` 。

```rust
use std::any::Any;

#[derive(Default)]
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn inc(&mut self) {
        self.x += 1;
        self.y += 1;
    }
}

fn map_any(mut any: Box<dyn Any>) -> Box<dyn Any> {
    if let Some(num) = any.downcast_mut::<i32>() {
        *num += 1;
    } else if let Some(string) = any.downcast_mut::<String>() {
        *string += "!";
    } else if let Some(point) = any.downcast_mut::<Point>() {
        point.inc();
    }
    any
}

fn main() {
    let mut vec: Vec<Box<dyn Any>> = vec![
        Box::new(0),
        Box::new(String::from("a")),
        Box::new(Point::default()),
    ];
    // vec = [0, "a", Point { x: 0, y: 0 }]
    vec = vec.into_iter().map(map_any).collect();
    // vec = [1, "a!", Point { x: 1, y: 1 }]
}
```

> This trait rarely _needs_ to be used because on top of parametric polymorphism being superior to ad-hoc polymorphism in most scenarios the latter can also be emulated using enums which are more type-safe and require less indirection. For example, we could have written the above example like this:

这个特性鲜少被使用，因为参数化的多态性时常要优于这样变通使用的多态性，且后者也可以使用更加类型安全和更加直接的枚举来模拟。如下例：

```rust
#[derive(Default)]
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn inc(&mut self) {
        self.x += 1;
        self.y += 1;
    }
}

enum Stuff {
    Integer(i32),
    String(String),
    Point(Point),
}

fn map_stuff(mut stuff: Stuff) -> Stuff {
    match &mut stuff {
        Stuff::Integer(num) => *num += 1,
        Stuff::String(string) => *string += "!",
        Stuff::Point(point) => point.inc(),
    }
    stuff
}

fn main() {
    let mut vec = vec![
        Stuff::Integer(0),
        Stuff::String(String::from("a")),
        Stuff::Point(Point::default()),
    ];
    // vec = [0, "a", Point { x: 0, y: 0 }]
    vec = vec.into_iter().map(map_stuff).collect();
    // vec = [1, "a!", Point { x: 1, y: 1 }]
}
```

> Despite `Any` rarely being _needed_ it can still be convenient to use sometimes, as we'll later see in the **Error Handling** section.

尽管 `Any` 特性鲜少是必须要被使用的，但有时它又是一种非常便捷的用法，我们将在 **错误处理** 一章中领会这一点。



## 文本格式化特性 Formatting Traits

> We can serialize types into strings using the formatting macros in `std::fmt`, the most well-known of the bunch being `println!`. We can pass formatting parameters to the `{}` placeholders used within format `str`s which are then used to select which trait impl to use to serialize the placeholder's argument.

我们可以使用 `std::fmt` 中提供的文本格式化宏来序列化结构体，例如我们最熟悉的 `println!` 。我们可以将文本格式化的参数传入 `{}` 占位符，以选择具体用哪个特性来序列化该结构。

| 特性 | 占位符 | 描述 |
|-------|-------------|-------------|
| `Display` | `{}` | 常规序列化 |
| `Debug` | `{:?}` | 调试序列化 |
| `Octal` | `{:o}` | 八进制序列化 |
| `LowerHex` | `{:x}` | 小写十六进制序列化 |
| `UpperHex` | `{:X}` | 大写十六进制序列化 |
| `Pointer` | `{:p}` | 内存地址 |
| `Binary` | `{:b}` | 二进制序列化 |
| `LowerExp` | `{:e}` | 小写指数序列化 |
| `UpperExp` | `{:E}` | 大写十六进制序列化 |



### Display & ToString

预备知识
- [Self](#self)
- [Methods](#methods)
- [Generic Blanket Impls](#generic-blanket-impls)

```rust
trait Display {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result;
}
```

> `Display` types can be serialized into `String`s which are friendly to the end users of the program. Example impl for `Point`:

实现 `Display` 特性的类型可以被序列化为 `String` 。这对于程序的用户来说非常的友好。例如：

```rust
use std::fmt;

#[derive(Default)]
struct Point {
    x: i32,
    y: i32,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    println!("origin: {}", Point::default());
    // prints "origin: (0, 0)"

    // get Point's Display representation as a String
    // Point 表达为可显示的 String
    let stringified_point = format!("{}", Point::default());
    assert_eq!("(0, 0)", stringified_point); // ✅
}
```

> Aside from using the `format!` macro to get a type's display representation as a `String` we can use the `ToString` trait:

除了使用 `format!` 宏来序列化结构体，我们也可以使用 `ToString` 特性：

```rust
trait ToString {
    fn to_string(&self) -> String;
}
```

> There's no need for us to impl this ourselves. In fact we can't, because of this generic blanket impl that automatically impls `ToString` for any type which impls `Display`:

我们不需要自己手动实现，事实上，我们也不能，因为对于实现了 `Display` 的类型来说，`ToString` 是由一揽子泛型实现所自动实现的。

```rust
impl<T: Display + ?Sized> ToString for T;
```

> Using `ToString` with `Point`:

对 `Point` 使用 `ToString` 特性：

```rust
#[test] // ✅
fn display_point() {
    let origin = Point::default();
    assert_eq!(format!("{}", origin), "(0, 0)");
}

#[test] // ✅
fn point_to_string() {
    let origin = Point::default();
    assert_eq!(origin.to_string(), "(0, 0)");
}

#[test] // ✅
fn display_equals_to_string() {
    let origin = Point::default();
    assert_eq!(format!("{}", origin), origin.to_string());
}
```



### Debug

预备知识
- [Self](#self)
- [Methods](#methods)
- [Derive Macros](#derive-macros)
- [Display & ToString](#display--tostring)

```rust
trait Debug {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result;
}
```

> `Debug` has an identical signature to `Display`. The only difference is that the `Debug` impl is called when we use the `{:?}` formatting specifier. `Debug` can be derived:

`Debug` 与 `Display` 具有相同的签名。唯一的区别在于我们使用 `{:?}` 文本格式化指令来调用 `Debug` 特性。 `Debug` 特性可以使用如下方法衍生：

```rust
use std::fmt;

#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

// derive macro generates impl below
// 衍生宏展开如下
impl fmt::Debug for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}
```

> Impling `Debug` for a type also allows it to be used within the `dbg!` macro which is superior to `println!` for quick and dirty print logging. Some of its advantages:
> 1. `dbg!` prints to stderr instead of stdout so the debug logs are easy to separate from the actual stdout output of our program.
> 2. `dbg!` prints the expression passed to it as well as the value the expression evaluated to.
> 3. `dbg!` takes ownership of its arguments and returns them so you can use it within expressions:

为特定类型实现 `Debug` 特性的同时，这也使得我们可以使用 `dbg!` 宏来快速地调试程序，这种方式要优于 `println!` 。其优点在于：
1. `dbg!` 输出到标准错误流而不是标准输出流，所以我们能够很容易地将调试信息提取出来。
2. `dbg!` 同时输出值和值的求值表达式。
3. `dbg!` 接管参数的属权，但不会吞掉参数，而是再抛出来，所以可以将它用在表达式中：

```rust
fn some_condition() -> bool {
    true
}

// no logging
// 没有日志
fn example() {
    if some_condition() {
        // some code
    }
}

// println! logging
// 使用 println! 打印日志
fn example_println() {
    // 🤦
    let result = some_condition();
    println!("{}", result); // just prints "true"
                            // 仅仅打印 "true"
    if result {
        // some code
    }
}

// dbg! logging
// 使用 dbg! 打印日志
fn example_dbg() {
    // 😍
    if dbg!(some_condition()) { // prints "[src/main.rs:22] some_condition() = true"
                                // 太棒了！打印出丰富的调试信息
        // some code
    }
}
```

> The only downside is that `dbg!` isn't automatically stripped in release builds so we have to manually remove it from our code if we don't want to ship it in the final executable.

`dbg!` 宏唯一的缺点是，它不能在构建最终发布的二进制文件时自动删除，我们不得不手动删除相关代码。



## 算符重载特性 Operator Traits

> All operators in Rust are associated with traits. If we'd like to impl operators for our types we have to impl the associated traits.

在 Rust 中，所有的算符都与相应的特性相关联。为特定类型实现相应特性，即为该类型实现了相应算符。

| 特性 | 类别 | 算符 | 描述 |
|----------|----------|-------------|-------------|
| `Eq`, `PartialEq` | 比较 | `==` | 相等 |
| `Ord`, `PartialOrd` | 比较 | `<`, `>`, `<=`, `>=` | 比较 |
| `Add` | 算数 | `+` | 加 |
| `AddAssign` | 算数 | `+=` | 加等于 |
| `BitAnd` | 算数 | `&` | 按位与 |
| `BitAndAssign` | 算数 | `&=` | 按位与等于 |
| `BitXor` | 算数 | `^` | 按位异或 |
| `BitXorAssign` | 算数 | `^=` | 按位异或等于 |
| `Div` | 算数 | `/` | 除 |
| `DivAssign` | 算数 | `/=` | 除等于 |
| `Mul` | 算数 | `*` | 乘 |
| `MulAssign` | 算数 | `*=` | 乘等于 |
| `Neg` | 算数 | `-` | 一元负 |
| `Not` | 算数 | `!` | 一元逻辑非 |
| `Rem` | 算数 | `%` | 求余 |
| `RemAssign` | 算数 | `%=` | 求余等于 |
| `Shl` | 算数 | `<<` | 左移 |
| `ShlAssign` | 算数 | `<<=` | 左移等于 |
| `Shr` | 算数 | `>>` | 右移 |
| `ShrAssign` | 算数 | `>>=` | 右移等于 |
| `Sub` | 算数 | `-` | 减 |
| `SubAssign` | 算数 | `-=` | 减等于 |
| `Fn` | 闭包 | `(...args)` | 不可变闭包调用 |
| `FnMut` | 闭包 | `(...args)` | 可变闭包调用 |
| `FnOnce` | 闭包 | `(...args)` | 一次性闭包调用 |
| `Deref` | 其它 | `*` | 不可变解引用 |
| `DerefMut` | 其它 | `*` | 可变解引用 |
| `Drop` | 其它 | - | 类型析构 |
| `Index` | 其它 | `[]` | 不可变索引 |
| `IndexMut` | 其它 | `[]` | 可变索引 |
| `RangeBounds` | 其它 | `..` | 范围迭代 |



### 比较特性 Comparison Traits

| 特性 | 类别 | 算符 | 描述 |
|----------|----------|-------------|-------------|
| `Eq`, `PartialEq` | 比较 | `==` | 相等 |
| `Ord`, `PartialOrd` | 比较 | `<`, `>`, `<=`, `>=` | 比较 |


#### PartialEq & Eq

预备知识
- [Self](#self)
- [Methods](#methods)
- [Generic Parameters](#generic-parameters)
- [Default Impls](#default-impls)
- [Generic Blanket Impls](#generic-blanket-impls)
- [Marker Traits](#marker-traits)
- [Subtraits & Supertraits](#subtraits--supertraits)
- [Sized](#sized)

```rust
trait PartialEq<Rhs = Self> 
where
    Rhs: ?Sized, 
{
    fn eq(&self, other: &Rhs) -> bool;

    // provided default impls
    // 提供默认实现
    fn ne(&self, other: &Rhs) -> bool;
}
```

> `PartialEq<Rhs>` types can be checked for equality to `Rhs` types using the `==` operator.

实现了 `PartialEq<Rhs>` 特性的类型可以使用 `==` 算符来检查与 `Rhs` 的相等性。

> All `PartialEq<Rhs>` impls must ensure that equality is symmetric and transitive. That means > for all `a`, `b`, and `c`:
> - `a == b` implies `b == a` (symmetry)
> - `a == b && b == c` implies `a == c` (transitivity) 

对 `PartialEq<Rhs>` 的实现须确保实现对称性与传递性。这意味着对于任意 `a` ， `b` 和 `c` 有：
- 若 `a == b` 则 `b == a` （对称性）
- 若 `a == b && b == c` 则 `a == c` （传递性）

> By default `Rhs = Self` because we almost always want to compare instances of a type to each other, and not to instances of different types. This also automatically guarantees our impl is symmetric and transitive.
>

默认情况下 `Rhs = Self` 是因为我们几乎总是在相同类型之间进行比较。这也自动地确保了我们的实现是对称的、可传递的。

```rust
struct Point {
    x: i32,
    y: i32
}

// Rhs == Self == Point
impl PartialEq for Point {
    // impl automatically symmetric & transitive
    // 该实现自动确保了对称性于传递性
    fn eq(&self, other: &Point) -> bool {
        self.x == other.x && self.y == other.y
    }
}
```

> If all the members of a type impl `PartialEq` then it can be derived:

如果特定类型的成员都实现了 `PartialEq` 特性，那么该类型也可衍生该特性：

```rust
#[derive(PartialEq)]
struct Point {
    x: i32,
    y: i32
}

#[derive(PartialEq)]
enum Suit {
    Spade,
    Heart,
    Club,
    Diamond,
}
```

> Once we impl `PartialEq` for our type we also get equality comparisons between references of our type for free thanks to these generic blanket impls:

多亏了一揽子泛型实现，一旦我们为特定类型实现了 `PartialEq` 特性，那么直接使用该类型的引用互相比较也是可以的：

```rust
// this impl only gives us: Point == Point
// 该衍生宏本身只允许我们在结构体之间进行比较
#[derive(PartialEq)]
struct Point {
    x: i32,
    y: i32
}

// all of the generic blanket impls below
// are provided by the standard library
// 以下的一揽子泛型实现由标准库提供

// this impl gives us: &Point == &Point
// 这个一揽子泛型实现允许我们通过不可变引用之间进行比较
impl<A, B> PartialEq<&'_ B> for &'_ A
where A: PartialEq<B> + ?Sized, B: ?Sized;

// this impl gives us: &mut Point == &Point
// 这个一揽子泛型实现允许我们通过可变引用与不可变引用进行比较
impl<A, B> PartialEq<&'_ B> for &'_ mut A
where A: PartialEq<B> + ?Sized, B: ?Sized;

// this impl gives us: &Point == &mut Point
// 这个一揽子泛型实现允许我们通过不可变引用与可变引用进行比较
impl<A, B> PartialEq<&'_ mut B> for &'_ A
where A: PartialEq<B> + ?Sized, B: ?Sized;

// this impl gives us: &mut Point == &mut Point
// 这个一揽子泛型实现允许我们通过可变引用之间进行比较
impl<A, B> PartialEq<&'_ mut B> for &'_ mut A
where A: PartialEq<B> + ?Sized, B: ?Sized;
```

> Since this trait is generic we can define equality between different types. The standard library leverages this to allow checking equality between the many string-like types such as `String`, `&str`, `PathBuf`, `&Path`, `OsString`, `&OsStr`, and so on.

由于该特性提供泛型，我们可以定义不同类型之间的可相等性。标准库正是利用这一点提供了不同类型字符串之间的比较功能，例如`String`， `&str`， `PathBuf`，`&Path`，`OsString` 和 `&OsStr`等等。

> Generally, we should only impl equality between different types _if they contain the same kind of data_ and the only difference between the types is how they represent the data or how they allow interacting with the data.

通常来说我们仅会实现相同类型之间的可相等性，除非两种类型虽然包含同一类数据，但又有表达形式或交互形式的差异，这时我们才会考虑实现不同类型之间的可相等性。

> Here's a cute but bad example of how someone might be tempted to impl `PartialEq` to check equality between different types that don't meet the above criteria:

以下是一个有趣但糟糕的例子，它尝试为不同类型实现 `PartialEq` 但又违背了上述要求：

```rust
#[derive(PartialEq)]
enum Suit {
    Spade,
    Club,
    Heart,
    Diamond,
}

#[derive(PartialEq)]
enum Rank {
    Ace,
    Two,
    Three,
    Four,
    Five,
    Six,
    Seven,
    Eight,
    Nine,
    Ten,
    Jack,
    Queen,
    King,
}

#[derive(PartialEq)]
struct Card {
    suit: Suit,
    rank: Rank,
}

// check equality of Card's suit
// 检查花色的相等性
impl PartialEq<Suit> for Card {
    fn eq(&self, other: &Suit) -> bool {
        self.suit == *other
    }
}

// check equality of Card's rank
// 检查牌序的相等性
impl PartialEq<Rank> for Card {
    fn eq(&self, other: &Rank) -> bool {
        self.rank == *other
    }
}

fn main() {
    let AceOfSpades = Card {
        suit: Suit::Spade,
        rank: Rank::Ace,
    };
    assert!(AceOfSpades == Suit::Spade); // ✅
    assert!(AceOfSpades == Rank::Ace); // ✅
}
```

> It works and kinda makes sense. A card which is an Ace of Spades is both an Ace and a Spade, and if we're writing a library to handle playing cards it's reasonable that we'd want to make it easy and convenient to individually check the suit and rank of a card. However, something's missing: symmetry! We can `Card == Suit` and `Card == Rank` but we cannot `Suit == Card` or `Rank == Card` so let's fix that:

上述代码有效且其逻辑有几分道理，黑桃 A 既是黑桃也是 A 。但如果我们真的去写一个处理扑克牌的库的话，最简单也最方便的方法莫过于独立地检查牌面的花色和牌序。而且，上述代码并不满足对称性！我们可以使用 `Card == Suit` 和 `Card == Rank` ，但却不能使用 `Suit == Card` 和 `Rank == Card`， 让我们来修复这一点：

```rust
// check equality of Card's suit
// 检查花色的相等性
impl PartialEq<Suit> for Card {
    fn eq(&self, other: &Suit) -> bool {
        self.suit == *other
    }
}

// added for symmetry
// 增加对称性
impl PartialEq<Card> for Suit {
    fn eq(&self, other: &Card) -> bool {
        *self == other.suit
    }
}

// check equality of Card's rank
// 检查牌序的相等性
impl PartialEq<Rank> for Card {
    fn eq(&self, other: &Rank) -> bool {
        self.rank == *other
    }
}

// added for symmetry
// 增加对称性
impl PartialEq<Card> for Rank {
    fn eq(&self, other: &Card) -> bool {
        *self == other.rank
    }
}
```

> We have symmetry! Great. Adding symmetry just broke transitivity! Oops. This is now possible:

我们实现了对称性！棒！但是实现对称性却破坏了传递性！糟tm大糕！考虑以下代码：

```rust
fn main() {
    // Ace of Spades
    // ♠A
    let a = Card {
        suit: Suit::Spade,
        rank: Rank::Ace,
    };
    let b = Suit::Spade;
    // King of Spades
    // ♠K
    let c = Card {
        suit: Suit::Spade,
        rank: Rank::King,
    };
    assert!(a == b && b == c); // ✅
    assert!(a == c); // ❌
}
```

> A good example of impling `PartialEq` to check equality between different types would be a program that works with distances and uses different types to represent different units of measurement.

关于对不同类型实现 `PartialEq` 特性的绝佳示例如下，本程序的功能在于处理空间上的距离，它使用不同的类型以表示不同的测量单位：

```rust
#[derive(PartialEq)]
struct Foot(u32);

#[derive(PartialEq)]
struct Yard(u32);

#[derive(PartialEq)]
struct Mile(u32);

impl PartialEq<Mile> for Foot {
    fn eq(&self, other: &Mile) -> bool {
        self.0 == other.0 * 5280
    }
}

impl PartialEq<Foot> for Mile {
    fn eq(&self, other: &Foot) -> bool {
        self.0 * 5280 == other.0
    }    
}

impl PartialEq<Mile> for Yard {
    fn eq(&self, other: &Mile) -> bool {
        self.0 == other.0 * 1760
    }
}

impl PartialEq<Yard> for Mile {
    fn eq(&self, other: &Yard) -> bool {
        self.0 * 1760 == other.0
    }    
}

impl PartialEq<Foot> for Yard {
    fn eq(&self, other: &Foot) -> bool {
        self.0 * 3 == other.0
    }
}

impl PartialEq<Yard> for Foot {
    fn eq(&self, other: &Yard) -> bool {
        self.0 == other.0 * 3
    }
}

fn main() {
    let a = Foot(5280);
    let b = Yard(1760);
    let c = Mile(1);
    
    // symmetry
    // 对称性
    assert!(a == b && b == a); // ✅
    assert!(b == c && c == b); // ✅
    assert!(a == c && c == a); // ✅

    // transitivity
    // 传递性
    assert!(a == b && b == c && a == c); // ✅
    assert!(c == b && b == a && c == a); // ✅
}
```

> `Eq` is a marker trait and a subtrait of `PartialEq<Self>`.

`Eq` 是仅用于标记的特性，也是 `PartialEq<Self>` 的子特性。

```rust
trait Eq: PartialEq<Self> {}
```

> If we impl `Eq` for a type, on top of the symmetry & transitivity properties required by `PartialEq`, we're also guaranteeing reflexivity, i.e. `a == a` for all `a`. In this sense `Eq` refines `PartialEq` because it represents a stricter version of equality. If all members of a type impl `Eq` then the `Eq` impl can be derived for the type.

鉴于 `PartialEq` 特性提供的对称性与传递性，一旦我们实现 `Eq` 特性，我们也就确保了该类型具有自反性，即对任意 `a` 有 `a == a` 。可以说， `Eq` 改良了 `PartialEq` ，因为它实现了一个比后者更加严格的可相等性。如果一个类型的全部成员都实现了 `Eq` 特性，那么该类型本身也可以衍生出该特性。

> Floats are `PartialEq` but not `Eq` because `NaN != NaN`. Almost all other `PartialEq` types are trivially `Eq`, unless of course if they contain floats.

所有的浮点类型都实现了 `PartialEq` 但是没有实现 `Eq` ，因为 `NaN != NaN` 。几乎所有其它实现 `PartialEq` 的类型也都自然地实现了 `Eq` ，除非它们包含了浮点数。

> Once a type impls `PartialEq` and `Debug` we can use it in the `assert_eq!` macro. We can also compare collections of `PartialEq` types.

对于实现了 `PartialEq` 和 `Debug` 的类型，我们也可以将它用于 `assert_eq!`  宏。并且，我们可以对实现 `PartialEq` 特性的类型组成的集合进行比较。

```rust
#[derive(PartialEq, Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn example_assert(p1: Point, p2: Point) {
    assert_eq!(p1, p2);
}

fn example_compare_collections<T: PartialEq>(vec1: Vec<T>, vec2: Vec<T>) {
    // if T: PartialEq this now works!
    if vec1 == vec2 {
        // some code
    } else {
        // other code
    }
}
```



#### Hash

预备知识
- [Self](#self)
- [Methods](#methods)
- [Generic Parameters](#generic-parameters)
- [Default Impls](#default-impls)
- [Derive Macros](#derive-macros)
- [PartialEq & Eq](#partialeq--eq)

```rust
trait Hash {
    fn hash<H: Hasher>(&self, state: &mut H);

    // provided default impls
    // 提供默认实现
    fn hash_slice<H: Hasher>(data: &[Self], state: &mut H);
}
```

> This trait is not associated with any operator, but the best time to talk about it is right after `PartialEq` & `Eq` so here it is. `Hash` types can be hashed using a `Hasher`.

本特性并未关联到任何算符，之所以在这里提及，是因为它与 `PartialEq` 与 `Eq` 密切的关系。实现 `Hash` 特性的类型可以通过 `Hasher` 作哈希运算。

```rust
use std::hash::Hasher;
use std::hash::Hash;

struct Point {
    x: i32,
    y: i32,
}

impl Hash for Point {
    fn hash<H: Hasher>(&self, hasher: &mut H) {
        hasher.write_i32(self.x);
        hasher.write_i32(self.y);
    }
}
```

> There's a derive macro which generates the same impl as above:

以下衍生宏展开与以上代码中相同的实现：

```rust
#[derive(Hash)]
struct Point {
    x: i32,
    y: i32,
}
```

> If a type impls both `Hash` and `Eq` those impls must agree with each other such that for all `a` and `b` if `a == b` then `a.hash() == b.hash()`. So we should always use the derive macro to impl both or manually impl both, but not mix the two, otherwise we risk breaking the above invariant.

如果一个类型同时实现了 `Hash` 和 `Eq` ，那么二者必须要实现步调一致，即对任意 `a` 与 `b` ， 若有 `a == b` ， 则必有 `a.hash() == b.hash()` 。所以，对于同时实现二者，要么都用衍生宏，要么都手动实现，不要一个用衍生宏，而另一个手动实现，否则我们将冒着步调不一致的极大风险。

> The main benefit of impling `Eq` and `Hash` for a type is that it allows us to store that type as keys in `HashMap`s and `HashSet`s.

实现`Eq` 和 `Hash` 特性的主要好处在于，这允许我们将该类型作为一个键存储于 `HashMap` 和 `HashSet` 中。

```rust
use std::collections::HashSet;

// now our type can be stored
// in HashSets and HashMaps!
// 现在我们的类型可以存储于 HashSet 和 HashMap 中了！
#[derive(PartialEq, Eq, Hash)]
struct Point {
    x: i32,
    y: i32,
}

fn example_hashset() {
    let mut points = HashSet::new();
    points.insert(Point { x: 0, y: 0 }); // ✅
}
```



#### PartialOrd & Ord

预备知识
- [Self](#self)
- [Methods](#methods)
- [Generic Parameters](#generic-parameters)
- [Default Impls](#default-impls)
- [Subtraits & Supertraits](#subtraits--supertraits)
- [Derive Macros](#derive-macros)
- [Sized](#sized)
- [PartialEq & Eq](#partialeq--eq)

```rust
enum Ordering {
    Less,
    Equal,
    Greater,
}

trait PartialOrd<Rhs = Self>: PartialEq<Rhs> 
where
    Rhs: ?Sized, 
{
    fn partial_cmp(&self, other: &Rhs) -> Option<Ordering>;

    // provided default impls
    // 提供默认实现
    fn lt(&self, other: &Rhs) -> bool;
    fn le(&self, other: &Rhs) -> bool;
    fn gt(&self, other: &Rhs) -> bool;
    fn ge(&self, other: &Rhs) -> bool;
}
```

> `PartialOrd<Rhs>` types can be compared to `Rhs` types using the `<`, `<=`, `>`, and `>=` operators.

实现 `PartialOrd<Rhs>` 的类型可以和 `Rhs` 的类型之间使用 `<`，`<=`，`>`，和 `>=` 算符。

> All `PartialOrd` impls must ensure that comparisons are asymmetric and transitive. That means for all `a`, `b`, and `c`:
> - `a < b` implies `!(a > b)` (asymmetry)
> - `a < b && b < c` implies `a < c` (transitivity)
>

实现 `PartialOrd` 时须确保比较的非对称性和传递性。这意味着对任意 `a`，`b`，`c`有：

- 若 `a < b` 则 `!(a > b)` （非对称性）
- 若 `a < b && b < c` 则 `a < c` （传递性）

> `PartialOrd` is a subtrait of `PartialEq` and their impls must always agree with each other.

`PartialOrd` 是 `PartialEq` 的子特性，二者必须要实现步调一致。

```rust
fn must_always_agree<T: PartialOrd + PartialEq>(t1: T, t2: T) {
    assert_eq!(t1.partial_cmp(&t2) == Some(Ordering::Equal), t1 == t2);
}
```

> `PartialOrd` refines `PartialEq` in the sense that when comparing `PartialEq` types we can check if they are equal or not equal, but when comparing `PartialOrd` types we can check if they are equal or not equal, and if they are not equal we can check if they are unequal because the first item is less than or greater than the second item.

`PartialOrd` 改良了 `PartialEq` ，后者仅能比较是否相等，而前者除了能比较是否相等，还能比较孰大孰小。

> By default `Rhs = Self` because we almost always want to compare instances of a type to each other, and not to instances of different types. This also automatically guarantees our impl is symmetric and transitive.

默认情况下 `Rhs = Self` ，因为我们几乎总是在相同类型的实例之间相比较，而不是不同类型之间。这一点自动保证了我们的实现的对称性和传递性。

```rust
use std::cmp::Ordering;

#[derive(PartialEq, PartialOrd)]
struct Point {
    x: i32,
    y: i32
}

// Rhs == Self == Point
impl PartialOrd for Point {
    // impl automatically symmetric & transitive
    // 该实现自动确保了对称性与传递性
    fn partial_cmp(&self, other: &Point) -> Option<Ordering> {
        Some(match self.x.cmp(&other.x) {
            Ordering::Equal => self.y.cmp(&other.y),
            ordering => ordering,
        })
    }
}
```

> If all the members of a type impl `PartialOrd` then it can be derived:

如果特定类型的全部成员都实现了 `PartialOrd` 特性，那么该类型也可以衍生出该特性：

```rust
#[derive(PartialEq, PartialOrd)]
struct Point {
    x: i32,
    y: i32,
}

#[derive(PartialEq, PartialOrd)]
enum Stoplight {
    Red,
    Yellow,
    Green,
}
```

> The `PartialOrd` derive macro orders types based on the lexicographical order of their members:

`PartialOrd` 衍生宏依据 **类型成员的定义顺序** 对类型进行排序：

```rust
// generates PartialOrd impl which orders
// Points based on x member first and
// y member second because that's the order
// they appear in the source code
// 宏展开的 PartialOrd 实现排序时
// 首先考虑 x 再考虑 y
// 因为这是它们在源代码中出现的顺序
#[derive(PartialOrd, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

// generates DIFFERENT PartialOrd impl
// which orders Points based on y member
// first and x member second
// 这里宏展开的 PartialOrd 实现排序时
// 首先考虑 y 再考虑 x
#[derive(PartialOrd, PartialEq)]
struct Point {
    y: i32,
    x: i32,
}
```

> `Ord` is a subtrait of `Eq` and `PartialOrd<Self>`:

`Ord` 是 `Eq` 和 `PartialOrd<Self>` 的子特性：

```rust
trait Ord: Eq + PartialOrd<Self> {
    fn cmp(&self, other: &Self) -> Ordering;

    // provided default impls
    // 提供默认实现
    fn max(self, other: Self) -> Self;
    fn min(self, other: Self) -> Self;
    fn clamp(self, min: Self, max: Self) -> Self;
}
```

> If we impl `Ord` for a type, on top of the asymmetry & transitivity properties required by `PartialOrd`, we're also guaranteeing that the asymmetry is total, i.e. exactly one of `a < b`, `a == b` or `a > b` is true for any given `a` and `b`. In this sense `Ord` refines `Eq` and `PartialOrd` because it represents a stricter version of comparisons. If a type impls `Ord` we can use that impl to trivially impl `PartialOrd`, `PartialEq`, and `Eq`:

鉴于 `PartialOrd` 提供的非对称性和传递性，对特定类型实现 `Ord` 特性的同时也就保证了其非对称性，即对于任意 `a` 与 `b` 有 `a < b` ，`a == b` ，`a < b` 。可以说， `Ord` 改良了 `Eq` 和 `PartialOrd` ，因为它提供了一种更加严格的比较。如果一个类型实现了 `Ord` ，那么 `PartialOrd` ，`PartialEq` 和 `Eq` 的实现也就微不足道了。

```rust
use std::cmp::Ordering;

// of course we can use the derive macros here
// 可以使用衍生宏
#[derive(Ord, PartialOrd, Eq, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

// note: as with PartialOrd, the Ord derive macro
// orders a type based on the lexicographical order
// of its members
// 注意：与 PatrialOrd 相同，Ord 衍生宏衍生宏依据
// 类型的成员的定义顺序 对类型进行排序

// but here's the impls if we wrote them out by hand
// 以下是我们手动的实现
impl Ord for Point {
    fn cmp(&self, other: &Self) -> Ordering {
        match self.x.cmp(&other.x) {
            Ordering::Equal => self.y.cmp(&other.y),
            ordering => ordering,
        }
    }
}
impl PartialOrd for Point {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}
impl PartialEq for Point {
    fn eq(&self, other: &Self) -> bool {
        self.cmp(other) == Ordering::Equal
    }
}
impl Eq for Point {}
```

> Floats impl `PartialOrd` but not `Ord` because both `NaN < 0 == false` and `NaN >= 0 == false` are simultaneously true. Almost all other `PartialOrd` types are trivially `Ord`, unless of course if they contain floats.

浮点数类型实现了 `PartialOrd` 但是没有实现 `Ord` ，因为 `NaN < 0 == false` 与 `NaN >= 0 == false` 同时为真。几乎所有其它实现 `PartialOrd` 的类型都实现了 `Ord` ，除非该类型包含浮点数。

> Once a type impls `Ord` we can store it in `BTreeMap`s and `BTreeSet`s as well as easily sort it using the `sort()` method on slices and any types which deref to slices such as arrays, `Vec`s, and `VecDeque`s.

对于实现了 `Ord` 特性的类型，我们可以将它存储于 `BTreeMap` 和 `BTreeSet` ，并且可以通过 `sort()` 方法对切片，或者任何可以自动解引用为切片的类型进行排序，例如 `Vec` 和 `VecDeque` 。

```rust
use std::collections::BTreeSet;

// now our type can be stored
// in BTreeSets and BTreeMaps!
// 现在我们的类型可以存储于 BTreeSet 和 BTreeMap 中了！
#[derive(Ord, PartialOrd, PartialEq, Eq)]
struct Point {
    x: i32,
    y: i32,
}

fn example_btreeset() {
    let mut points = BTreeSet::new();
    points.insert(Point { x: 0, y: 0 }); // ✅
}

// we can also .sort() Ord types in collections!
// 对于实现了 Ord 特性的类型，我们可以使用 .sort() 方法来对集合进行排序！
fn example_sort<T: Ord>(mut sortable: Vec<T>) -> Vec<T> {
    sortable.sort();
    sortable
}
```



### 算术特性 Arithmetic Traits

| 特性           | 类别 | 算符  | 描述         |
| -------------- | ---- | ----- | ------------ |
| `Add`          | 算数 | `+`   | 加           |
| `AddAssign`    | 算数 | `+=`  | 加等于       |
| `BitAnd`       | 算数 | `&`   | 按位与       |
| `BitAndAssign` | 算数 | `&=`  | 按位与等于   |
| `BitXor`       | 算数 | `^`   | 按位异或     |
| `BitXorAssign` | 算数 | `^=`  | 按位异或等于 |
| `Div`          | 算数 | `/`   | 除           |
| `DivAssign`    | 算数 | `/=`  | 除等于       |
| `Mul`          | 算数 | `*`   | 乘           |
| `MulAssign`    | 算数 | `*=`  | 乘等于       |
| `Neg`          | 算数 | `-`   | 一元负       |
| `Not`          | 算数 | `!`   | 一元逻辑非   |
| `Rem`          | 算数 | `%`   | 求余         |
| `RemAssign`    | 算数 | `%=`  | 求余等于     |
| `Shl`          | 算数 | `<<`  | 左移         |
| `ShlAssign`    | 算数 | `<<=` | 左移等于     |
| `Shr`          | 算数 | `>>`  | 右移         |
| `ShrAssign`    | 算数 | `>>=` | 右移等于     |
| `Sub`          | 算数 | `-`   | 减           |
| `SubAssign`    | 算数 | `-=`  | 减等于       |

> Going over all of these would be very redundant. Most of these only apply to number types anyway. We'll only go over `Add` and `AddAssign` since the `+` operator is commonly overloaded to do other stuff like adding items to collections or concatenating things together, that way we cover the most interesting ground and don't repeat ourselves.
>

详解以上所有算术特性未免显得多余，且其大多仅用于操作数字类型。本文仅就最常见被重载的 `Add` 和 `AddAssign` 特性，亦即 `+` 和 `+=` 算符，进行说明，其重载广泛用于为集合增加内容或对不同事物的连接。这样，我们多侧重于最有趣的地方，而不是无趣枯燥地重复。


#### Add & AddAssign

预备知识
- [Self](#self)
- [Methods](#methods)
- [Associated Types](#associated-types)
- [Generic Parameters](#generic-parameters)
- [Generic Types vs Associated Types](#generic-types-vs-associated-types)
- [Derive Macros](#derive-macros)

```rust
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```

> `Add<Rhs, Output = T>` types can be added to `Rhs` types and will produce `T` as output.

实现 `Add<Rhs, Output = T>` 特性的类型，与 `Rhs` 类型相加得到 `T` 类型的值。

> Example `Add<Point, Output = Point>` impl for `Point`:

下例对 `Point` 类型实现了 `Add<Rhs, Output = T>` ：

```rust
#[derive(Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Point;
    fn add(self, rhs: Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let p3 = p1 + p2;
    assert_eq!(p3.x, p1.x + p2.x); // ✅
    assert_eq!(p3.y, p1.y + p2.y); // ✅
}
```

> But what if we only had references to `Point`s? Can we still add them then? Let's try:

如果我们对 `Point` 的引用进行如上操作还能将他们加在一起吗？我们试试：

```rust
fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let p3 = &p1 + &p2; // ❌
}
```

> Unfortunately not. The compiler throws:

遗憾的是，并不可以。编译器出错了：

```none
error[E0369]: cannot add `&Point` to `&Point`
  --> src/main.rs:50:25
   |
50 |     let p3: Point = &p1 + &p2;
   |                     --- ^ --- &Point
   |                     |
   |                     &Point
   |
   = note: an implementation of `std::ops::Add` might be missing for `&Point`
```

> Within Rust's type system, for some type `T`, the types `T`, `&T`, and `&mut T` are all treated as unique distinct types which means we have to provide trait impls for each of them separately. Let's define an `Add` impl for `&Point`:
>

在 Rust 的类型系统中，对于特定类型 `T` 来讲，`T` ，`&T` ，`&mut T` 三者本身是具有不同类型的，这意味着我们需要对它们分别实现相应特性。下面我们对 `&Point` 实现 `Add` 特性：

```rust
impl Add for &Point {
    type Output = Point;
    fn add(self, rhs: &Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let p3 = &p1 + &p2; // ✅
    assert_eq!(p3.x, p1.x + p2.x); // ✅
    assert_eq!(p3.y, p1.y + p2.y); // ✅
}
```

> However, something still doesn't feel quite right. We have two separate impls of `Add` for `Point` and `&Point` and they _happen_ to do the same thing currently but there's no guarantee that they will in the future! For example, let's say we decide that when we add two `Point`s together we want to create a `Line` containing those two `Point`s instead of creating a new `Point`, we'd update our `Add` impl like this:

这是可行的，但是不觉得哪里怪怪的吗？我们对 `Point` 和 `&Point` 分别实现了 `Add` 特性，现在来看这两种实现能够保持步调一致，但是未来也能保证吗？例如，我们现在决定对两个 `Point` 相加要产生一个 `Line` 而不是 `Point` ，可以对 `Add` 特性的实现做出如下改动：

```rust
use std::ops::Add;

#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}

#[derive(Copy, Clone)]
struct Line {
    start: Point,
    end: Point,
}

// we updated this impl
// 我们更新了这个实现
impl Add for Point {
    type Output = Line;
    fn add(self, rhs: Point) -> Line {
        Line {
            start: self,
            end: rhs,
        }
    }
}

// but forgot to update this impl, uh oh!
// 但是忘记了更新这个实现，糟tm大糕！
impl Add for &Point {
    type Output = Point;
    fn add(self, rhs: &Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let line: Line = p1 + p2; // ✅

    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let line: Line = &p1 + &p2; // ❌ expected Line, found Point
                                // ❌ 期待得到 Line ，但是得到 Point
}
```

> Our current impl of `Add` for `&Point` creates an unnecessary maintenance burden, we want the impl to match `Point`'s impl without having to manually update it every time we change `Point`'s impl. We'd like to keep our code as DRY (Don't Repeat Yourself) as possible. Luckily this is achievable:

我们对 `&Point` 不可变引用类型的 `Add` 实现，给我们带来了不必要的维护困难。是否能够使得，当我们更改 `Point` 类型的实现时， `&Point` 类型的实现也能够自动发生匹配，而不需要我们手动维护呢？我们的愿望是尽可能写出 `DRY (Don't Repeat Yourself)` 的不重复的代码。幸运的是，我们可以如此实现这一点：

```rust
// updated, DRY impl
// 使用一种更“干”的实现
impl Add for &Point {
    type Output = <Point as Add>::Output;
    fn add(self, rhs: &Point) -> Self::Output {
        Point::add(*self, *rhs)
    }
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let line: Line = p1 + p2; // ✅

    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let line: Line = &p1 + &p2; // ✅
}
```

> `AddAssign<Rhs>` types allow us to add + assign `Rhs` types to them. The trait declaration:

实现 `AddAssign<Rhs>` 的类型，允许我们对 `Rhs` 的类型相加之并赋值到自身。该特性的声明为：

```rust
trait AddAssign<Rhs = Self> {
    fn add_assign(&mut self, rhs: Rhs);
}
```

> Example impls for `Point` and `&Point`:

对 `Point` 和 `&Point` 类型的实现示例：

```rust
use std::ops::AddAssign;

#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32
}

impl AddAssign for Point {
    fn add_assign(&mut self, rhs: Point) {
        self.x += rhs.x;
        self.y += rhs.y;
    }
}

impl AddAssign<&Point> for Point {
    fn add_assign(&mut self, rhs: &Point) {
        Point::add_assign(self, *rhs);
    }
}

fn main() {
    let mut p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    p1 += &p2;
    p1 += p2;
    assert!(p1.x == 7 && p1.y == 10);
}
```



### 闭包特性 Closure Traits

| 特性 | 类别 | 算符 | 描述 |
|----------|----------|-------------|-------------|
| `Fn` | 闭包 | `(...args)` | 不可变闭包调用 |
| `FnMut` | 闭包 | `(...args)` | 可变闭包调用 |
| `FnOnce` | 闭包 | `(...args)` | 一次性闭包调用 |



#### FnOnce, FnMut, & Fn

预备知识
- [Self](#self)
- [Methods](#methods)
- [Associated Types](#associated-types)
- [Generic Parameters](#generic-parameters)
- [Generic Types vs Associated Types](#generic-types-vs-associated-types)
- [Subtraits & Supertraits](#subtraits--supertraits)

```rust
trait FnOnce<Args> {
    type Output;
    fn call_once(self, args: Args) -> Self::Output;
}

trait FnMut<Args>: FnOnce<Args> {
    fn call_mut(&mut self, args: Args) -> Self::Output;
}

trait Fn<Args>: FnMut<Args> {
    fn call(&self, args: Args) -> Self::Output;
}
```

> Although these traits exist it's not possible to impl them for our own types in stable Rust. The only types we can create which impl these traits are closures. Depending on what the closure captures from its environment determines whether it impls `FnOnce`, `FnMut`, or `Fn`.

事实上，在 stable Rust 中我们并不能对我们自己的类型实现上述特性，唯一的例外是闭包。对于闭包从环境中捕获的值的不同，该闭包会实现不同的特性：`FnOnce` ，`FnMut` ，`Fn` 。

> An `FnOnce` closure can only be called once because it consumes some value as part of its execution:

对于实现 `FnOnce` 的闭包，仅可调用一次，因为它消耗掉了其执行中必须的值：

```rust
fn main() {
    let range = 0..10;
    let get_range_count = || range.count();
    assert_eq!(get_range_count(), 10); // ✅
    get_range_count(); // ❌
}
```

> The `.count()` method on iterators consumes the iterator so it can only be called once. Hence our closure can only be called once. Which is why when we try to call it a second time we get this error:

迭代器上的 `.count()` 方法会消耗掉整个迭代器，所以该方法仅能调用一次。所以我们的闭包也就是能调用一次了，这就是为什么当第二次调用该闭包时会出错：

```none
error[E0382]: use of moved value: `get_range_count`
 --> src/main.rs:5:5
  |
4 |     assert_eq!(get_range_count(), 10);
  |                ----------------- `get_range_count` moved due to this call
5 |     get_range_count();
  |     ^^^^^^^^^^^^^^^ value used here after move
  |
note: closure cannot be invoked more than once because it moves the variable `range` out of its environment
 --> src/main.rs:3:30
  |
3 |     let get_range_count = || range.count();
  |                              ^^^^^
note: this value implements `FnOnce`, which causes it to be moved when called
 --> src/main.rs:4:16
  |
4 |     assert_eq!(get_range_count(), 10);
  |                ^^^^^^^^^^^^^^^
```

> An `FnMut` closure can be called multiple times and can also mutate variables it has captured from its environment. We might say `FnMut` closures perform side-effects or are stateful. Here's an example of a closure that filters out all non-ascending values from an iterator by keeping track of the smallest value it has seen so far:

对于实现 `FnMut` 特性的闭包，我们可以多次调用，且其可以改变其从环境捕获的值。我们可以说实现 `FnMut` 的闭包的执行具有副作用，或者说它是具有状态的。下例展示了一个闭包，它通过跟踪最小值，来找到一个迭代器中所有非升序的值：

```rust
fn main() {
    let nums = vec![0, 4, 2, 8, 10, 7, 15, 18, 13];
    let mut min = i32::MIN;
    let ascending = nums.into_iter().filter(|&n| {
        if n <= min {
            false
        } else {
            min = n;
            true
        }
    }).collect::<Vec<_>>();
    assert_eq!(vec![0, 4, 8, 10, 15, 18], ascending); // ✅
}
```

> `FnMut` refines `FnOnce` in the sense that `FnOnce` requires taking ownership of its arguments and can only be called once, but `FnMut` requires only taking mutable references and can be called multiple times. `FnMut` can be used anywhere `FnOnce` can be used.

`FnMut` 改良了 `FnOnce` ，`FnOnce` 需要接管参数的属权因此只能调用一次，而 `FnMut` 只需要参数的可变引用即可并可调用多次。`FnMut` 可以在所有 `FnOnce` 可用的地方使用。

> An `Fn` closure can be called multiple times and does not mutate any variables it has captured from its environment. We might say `Fn` closures have no side-effects or are stateless. Here's an example closure that filters out all values less than some stack variable it captures from its environment from an iterator:

对于实现 `Fn ` 特性的闭包，我们可以调用多次，且其不改变任何从环境中捕获的变量。我们可以说实现 `Fn` 的闭包的执行不具有副作用，或者说它是不具有状态的。下例展示了一个闭包，它通过与栈上的值进行比较，过滤掉一个迭代器中所有比它小的值：

```rust
fn main() {
    let nums = vec![0, 4, 2, 8, 10, 7, 15, 18, 13];
    let min = 9;
    let greater_than_9 = nums.into_iter().filter(|&n| n > min).collect::<Vec<_>>();
    assert_eq!(vec![10, 15, 18, 13], greater_than_9); // ✅
}
```

> `Fn` refines `FnMut` in the sense that `FnMut` requires mutable references and can be called multiple times, but `Fn` only requires immutable references and can be called multiple times. `Fn` can be used anywhere `FnMut` can be used, which includes anywhere `FnOnce` can be used.

`Fn` 改良了 `FnMut` ，尽管它们都可以多次调用，但是 `FnMut` 需要参数的可变引用，而 `Fn` 仅需要参数的不可变引用。`Fn` 可以在所有 `FnMut` 和 `FnOnce` 可用的地方使用。

> If a closure doesn't capture anything from its environment it's technically not a closure, but just an anonymously declared inline function, and can be casted to, used, and passed around as a regular function pointer, i.e. `fn`. Function pointers can be used anywhere `Fn` can be used, which includes anywhere `FnMut` and `FnOnce` can be used.

如果一个闭包不从环境中捕获任何的值，那么从技术上讲它就不是闭包，而仅仅只是一个内联的匿名函数。并且它可以被转换为、用于或传递为一个常规函数指针，即 `fn`。函数指针可以用于任何 `Fn` ，`FnMut` ，`FnOnce` 可用的地方。

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

fn main() {
    let mut fn_ptr: fn(i32) -> i32 = add_one;
    assert_eq!(fn_ptr(1), 2); // ✅
    
    // capture-less closure cast to fn pointer
    // 不捕获环境的闭包可转换为普通函数指针
    fn_ptr = |x| x + 1; // same as add_one
    assert_eq!(fn_ptr(1), 2); // ✅
}
```

> Example of passing a regular function pointer in place of a closure:

以下示例中，将常规函数作为闭包而传入：

```rust
fn main() {
    let nums = vec![-1, 1, -2, 2, -3, 3];
    let absolutes: Vec<i32> = nums.into_iter().map(i32::abs).collect();
    assert_eq!(vec![1, 1, 2, 2, 3, 3], absolutes); // ✅
}
```



### 其它特性 Other Traits

| 特性 | 类别 | 算符 | 描述 |
|----------|----------|-------------|-------------|
| `Deref` | 其它 | `*` | 不可变解引用 |
| `DerefMut` | 其它 | `*` | 可变解引用 |
| `Drop` | 其它 | - | 类型析构 |
| `Index` | 其它 | `[]` | 不可变索引 |
| `IndexMut` | 其它 | `[]` | 可变索引 |
| `RangeBounds` | 其它 | `..` | 范围迭代 |



#### Deref & DerefMut

预备知识
- [Self](#self)
- [Methods](#methods)
- [Associated Types](#associated-types)
- [Subtraits & Supertraits](#subtraits--supertraits)
- [Sized](#sized)

```rust
trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}

trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

> `Deref<Target = T>` types can dereferenced to `T` types using the dereference operator `*`. This has obvious use-cases for smart pointer types like `Box` and `Rc`. However, we rarely see the dereference operator explicitly used in Rust code, and that's because of a Rust feature called _deref coercion_.

实现 `Deref<Target = T>` 的类型，可以通过 `*` 解引用算符，解引用到 `T` 类型。智能指针是该特性的著名实现者，例如 `Box` 和 `Rc` 。不过，我们很少在 Rust 编程中看到解引用算符，这是由于 Rust 的强制解引用的特性所导致的。

> Rust automatically dereferences types when they're being passed as function arguments, returned from a function, or used as part of a method call. This is the reason why we can pass `&String` and `&Vec<T>` to functions expecting `&str` and `&[T]` because `String` impls `Deref<Target = str>` and `Vec<T>` impls `Deref<Target = [T]>`.

当作为函数的参数、函数的返回值、方法的调用参数时，Rust 会自动地解引用。这就是为什么我们可以将 `&String` 或 `&Vec<T>` 类型的值作为参数传递给接受 `str` 或 `&[T]` 类型的参数的函数，因为 `String` 实现了 `Deref<Target = str>` ，`Vec<t>` 实现了 `Deref<Target = [T]>` 。

> `Deref` and `DerefMut` should only be implemented for smart pointer types. The most common way people attempt to misuse and abuse these traits is to try to shoehorn some kind of OOP-style data inheritance into Rust. This does not work. Rust is not OOP. Let's examine a few different situations where, how, and why it does not work. Let's start with this example:

`Deref` 和 `DerefMut` 仅应实现于智能指针类型。最常见的误用或滥用就是，人们经常希望强行把某种面向对象编程风格的数据继承塞到 Rust 编程中。这是不可能的，因为 Rust 不是面向对象的。让我们用一个例子来领会到底为什么这是不可以的：

```rust
use std::ops::Deref;

struct Human {
    health_points: u32,
}

enum Weapon {
    Spear,
    Axe,
    Sword,
}

// a Soldier is just a Human with a Weapon
// 士兵是手持武器的人类
struct Soldier {
    human: Human,
    weapon: Weapon,
}

impl Deref for Soldier {
    type Target = Human;
    fn deref(&self) -> &Human {
        &self.human
    }
}

enum Mount {
    Horse,
    Donkey,
    Cow,
}

// a Knight is just a Soldier with a Mount
// 骑士是胯骑坐骑的士兵
struct Knight {
    soldier: Soldier,
    mount: Mount,
}

impl Deref for Knight {
    type Target = Soldier;
    fn deref(&self) -> &Soldier {
        &self.soldier
    }
}

enum Spell {
    MagicMissile,
    FireBolt,
    ThornWhip,
}

// a Mage is just a Human who can cast Spells
// 法师是口诵咒语的人类
struct Mage {
    human: Human,
    spells: Vec<Spell>,
}

impl Deref for Mage {
    type Target = Human;
    fn deref(&self) -> &Human {
        &self.human
    }
}

enum Staff {
    Wooden,
    Metallic,
    Plastic,
}

// a Wizard is just a Mage with a Staff
// 巫师是腰别法宝的法师
struct Wizard {
    mage: Mage,
    staff: Staff,
}

impl Deref for Wizard {
    type Target = Mage;
    fn deref(&self) -> &Mage {
        &self.mage
    }
}

fn borrows_human(human: &Human) {}
fn borrows_soldier(soldier: &Soldier) {}
fn borrows_knight(knight: &Knight) {}
fn borrows_mage(mage: &Mage) {}
fn borrows_wizard(wizard: &Wizard) {}

fn example(human: Human, soldier: Soldier, knight: Knight, mage: Mage, wizard: Wizard) {
    // all types can be used as Humans
    borrows_human(&human);
    borrows_human(&soldier);
    borrows_human(&knight);
    borrows_human(&mage);
    borrows_human(&wizard);
    // Knights can be used as Soldiers
    borrows_soldier(&soldier);
    borrows_soldier(&knight);
    // Wizards can be used as Mages
    borrows_mage(&mage);
    borrows_mage(&wizard);
    // Knights & Wizards passed as themselves
    borrows_knight(&knight);
    borrows_wizard(&wizard);
}
```

> So at first glance the above looks pretty good! However it quickly breaks down to scrutiny. First of all, deref coercion only works on references, so it doesn't work when we actually want to pass ownership:

事实上，并不可以这么做。首先，强制解引用仅用于引用，所以我们不能移交属权：

```rust
fn takes_human(human: Human) {}

fn example(human: Human, soldier: Soldier, knight: Knight, mage: Mage, wizard: Wizard) {
    // all types CANNOT be used as Humans
    takes_human(human);
    takes_human(soldier); // ❌
    takes_human(knight); // ❌
    takes_human(mage); // ❌
    takes_human(wizard); // ❌
}
```

> Furthermore, deref coercion doesn't work in generic contexts. Let's say we impl some trait only on humans:

其次，强制解引用不可用于泛型编程。例如某特性仅对人类实现：

```rust
trait Rest {
    fn rest(&self);
}

impl Rest for Human {
    fn rest(&self) {}
}

fn take_rest<T: Rest>(rester: &T) {
    rester.rest()
}

fn example(human: Human, soldier: Soldier, knight: Knight, mage: Mage, wizard: Wizard) {
    // all types CANNOT be used as Rest types, only Human
    take_rest(&human);
    take_rest(&soldier); // ❌
    take_rest(&knight); // ❌
    take_rest(&mage); // ❌
    take_rest(&wizard); // ❌
}
```

> Also, although deref coercion works in a lot of places it doesn't work everywhere. It doesn't work on operands, even though operators are just syntax sugar for method calls. Let's say, to be cute, we wanted `Mage`s to learn `Spell`s using the `+=` operator:

强制解引用可以用于许多情况，但绝不是所有情况。例如对于算符的操作数而言就不行，即便算符仅是一种方法调用的语法糖。比如，我们希望使用 `+=` 算符来表达法师学习咒语。

```rust
impl DerefMut for Wizard {
    fn deref_mut(&mut self) -> &mut Mage {
        &mut self.mage
    }
}

impl AddAssign<Spell> for Mage {
    fn add_assign(&mut self, spell: Spell) {
        self.spells.push(spell);
    }
}

fn example(mut mage: Mage, mut wizard: Wizard, spell: Spell) {
    mage += spell;
    wizard += spell; // ❌ wizard not coerced to mage here
                     // ❌ 在这里，巫师不能强制转换为法师
    wizard.add_assign(spell); // oof, we have to call it like this 🤦
                              // 所以，我们必须要这样做 🤦
}
```

> In languages with OOP-style data inheritance the value of `self` within a method is always equal to the type which called the method but in the case of Rust the value of `self` is always equal to the type which implemented the method:

在带有面向对象风格的数据继承的语言中，方法中的 `self` 值的类型总是等同于调用该方法的类型。但是在 Rust 语言中，`self` 值的类型总是等同于实现该方法时的类型。

```rust
struct Human {
    profession: &'static str,
    health_points: u32,
}

impl Human {
    // self will always be a Human here, even if we call it on a Soldier
    // 该方法中的 self 的类型永远是 Human ，即便我们在 Soldier 类型上调用
    fn state_profession(&self) {
        println!("I'm a {}!", self.profession);
    }
}

struct Soldier {
    profession: &'static str,
    human: Human,
    weapon: Weapon,
}

fn example(soldier: &Soldier) {
    assert_eq!("servant", soldier.human.profession);
    assert_eq!("spearman", soldier.profession);
    soldier.human.state_profession(); // prints "I'm a servant!"
    soldier.state_profession(); // still prints "I'm a servant!" 🤦
}
```

> The above gotcha is especially damning when impling `Deref` or `DerefMut` on a newtype. Let's say we want to create a `SortedVec` type which is just a `Vec` but it's always in sorted order. Here's how we might do that:

上述特性常令人感到困惑，特别是在对新类型实现 `Deref` 和 `DerefMut` 的时候。例如我们想要设计一个 `SortedVec` 类型，相比于 `Vec` 类型，它总是处于已排序的状态。我们可能会这样做：

```rust
struct SortedVec<T: Ord>(Vec<T>);

impl<T: Ord> SortedVec<T> {
    fn new(mut vec: Vec<T>) -> Self {
        vec.sort();
        SortedVec(vec)
    }
    fn push(&mut self, t: T) {
        self.0.push(t);
        self.0.sort();
    }
}
```

> Obviously we cannot impl `DerefMut<Target = Vec<T>>` here or anyone using `SortedVec` would be able to trivially break the sorted order. However, impling `Deref<Target = Vec<T>>` surely must be safe, right? Try to spot the bug in the program below:

显然我们不能为其实现 `DerefMut<Target = Vec<T>>` ，因为这可能会破坏排序状态。实现 `Deref<Target = Vec<T>>` 必须要保证功能的正确性。尝试指出下列代码中的 bug ：

```rust
use std::ops::Deref;

struct SortedVec<T: Ord>(Vec<T>);

impl<T: Ord> SortedVec<T> {
    fn new(mut vec: Vec<T>) -> Self {
        vec.sort();
        SortedVec(vec)
    }
    fn push(&mut self, t: T) {
        self.0.push(t);
        self.0.sort();
    }
}

impl<T: Ord> Deref for SortedVec<T> {
    type Target = Vec<T>;
    fn deref(&self) -> &Vec<T> {
        &self.0
    }
}

fn main() {
    let sorted = SortedVec::new(vec![2, 8, 6, 3]);
    sorted.push(1);
    let sortedClone = sorted.clone();
    sortedClone.push(4);
}
```

> We never implemented `Clone` for `SortedVec` so when we call the `.clone()` method the compiler is using deref coercion to resolve that method call on `Vec` and so it returns a `Vec` and not a `SortedVec`!

鉴于我们从未对 `SortedVec` 实现 `Clone` 特性，所以当我们调用 `.clone()` 方法的时候，编译器会使用强制解引用将该方法调用解析为 `Vec` 的方法调用，所以该方法返回的是 `Vec` 而不是 `SortedVec` ！

```rust
fn main() {
    let sorted: SortedVec<i32> = SortedVec::new(vec![2, 8, 6, 3]);
    sorted.push(1); // still sorted

    // calling clone on SortedVec actually returns a Vec 🤦
    let sortedClone: Vec<i32> = sorted.clone();
    sortedClone.push(4); // sortedClone no longer sorted 💀
}
```

> Anyway, none of the above limitations, constraints, or gotchas are faults of Rust because Rust was never designed to be an OO language or to support any OOP patterns in the first place.

切记，Rust 并非设计为面向对象的语言，也并不将面向对象编程的模式作为一等公民，所以以上的限制、约束和令人困惑的特性并不被认为是在语言中是错误的。

> The main takeaway from this section is do not try to be cute or clever with `Deref` and `DerefMut` impls. They're really only appropriate for smart pointer types, which can only be implemented within the standard library for now as smart pointer types currently require unstable features and compiler magic to work. If we want functionality and behavior similar to `Deref` and `DerefMut` then what we're actually probably looking for is `AsRef` and `AsMut` which we'll get to later.

本节的主旨即是使读者领会为什么不要自作聪明地实现 `Deref` 和 `DerefMut` 特性。这类特性确仅适合于智能指针类的类型，目前来讲标准库中的智能指针的实现，确需要这样的不稳定特性以及一些编译器魔法才能工作。如果我们确需要一些类似于`Deref` 和 `DerefMut` 的特性，不妨使用 `AsRef` 和 `AsMut` 特性。我们将在后面的章节中对这类特性做出说明。



#### Index & IndexMut

预备知识
- [Self](#self)
- [Methods](#methods)
- [Associated Types](#associated-types)
- [Generic Parameters](#generic-parameters)
- [Generic Types vs Associated Types](#generic-types-vs-associated-types)
- [Subtraits & Supertraits](#subtraits--supertraits)
- [Sized](#sized)

```rust
trait Index<Idx: ?Sized> {
    type Output: ?Sized;
    fn index(&self, index: Idx) -> &Self::Output;
}

trait IndexMut<Idx>: Index<Idx> where Idx: ?Sized {
    fn index_mut(&mut self, index: Idx) -> &mut Self::Output;
}
```

> We can index `[]` into `Index<T, Output = U>` types with `T` values and the index operation will return `&U` values. For syntax sugar, the compiler auto inserts a deref operator `*` in front of any value returned from an index operation:

对于实现 `Index<T, Output = U>` 的类型，我们可以使用 `[]` 索引算符对 `T` 类型的值索引 `&U` 类型的值。作为语法糖，编译器也会为索引操作返回的值自动添加一个 `*` 解引用算符。

```rust
fn main() {
    // Vec<i32> impls Index<usize, Output = i32> so
    // indexing Vec<i32> should produce &i32s and yet...
    // 鉴于 Vec<i32> 实现了 Index<usize, Output = i32>
    // 所以对 Vec<i32> 的索引应当返回 &i32 类型的值，但是。。。
    let vec = vec![1, 2, 3, 4, 5];
    let num_ref: &i32 = vec[0]; // ❌ expected &i32 found i32
    
    // above line actually desugars to
    // 以上代码等价于
    let num_ref: &i32 = *vec[0]; // ❌ expected &i32 found i32

    // both of these alternatives work
    // 以下是建议使用的一对形式
    let num: i32 = vec[0]; // ✅
    let num_ref: &i32 = &vec[0]; // ✅
}
```

> It's kinda confusing at first, because it seems like the `Index` trait does not follow its own method signature, but really it's just questionable syntax sugar.

令人困惑的是，似乎 `Index` 特性没有遵循它自己的方法签名，但其实真正有问题的是语法糖。

> Since `Idx` is a generic type the `Index` trait can be implemented many times for a given type, and in the case of `Vec<T>` not only can we index into it using `usize` but we can also index into its using `Range<usize>`s to get slices.

鉴于 `Idx` 是泛型类型，`Index` 特性对多个给定类型可以多次实现。并且对于 `Vec<T>` ，我们不仅可以对 `usize` 索引，还可以对 `Range<usize>` 索引得到切片。

```rust
fn main() {
    let vec = vec![1, 2, 3, 4, 5];
    assert_eq!(&vec[..], &[1, 2, 3, 4, 5]); // ✅
    assert_eq!(&vec[1..], &[2, 3, 4, 5]); // ✅
    assert_eq!(&vec[..4], &[1, 2, 3, 4]); // ✅
    assert_eq!(&vec[1..4], &[2, 3, 4]); // ✅
}
```

> To show off how we might impl `Index` ourselves here's a fun example which shows how we can use a newtype and the `Index` trait to impl wrapping indexes and negative indexes on a `Vec`:

为了展示如何自己实现 `Index` 特性，以下是一个有趣的例子，它设计了一个 `Vec` 的包装结构，其使得循环索引和负数索引成为可能：

```rust
use std::ops::Index;

struct WrappingIndex<T>(Vec<T>);

impl<T> Index<usize> for WrappingIndex<T> {
    type Output = T;
    fn index(&self, index: usize) -> &T {
        &self.0[index % self.0.len()]
    }
}

impl<T> Index<i128> for WrappingIndex<T> {
    type Output = T;
    fn index(&self, index: i128) -> &T {
        let self_len = self.0.len() as i128;
        let idx = (((index % self_len) + self_len) % self_len) as usize;
        &self.0[idx]
    }
}

#[test] // ✅
fn indexes() {
    let wrapping_vec = WrappingIndex(vec![1, 2, 3]);
    assert_eq!(1, wrapping_vec[0_usize]);
    assert_eq!(2, wrapping_vec[1_usize]);
    assert_eq!(3, wrapping_vec[2_usize]);
}

#[test] // ✅
fn wrapping_indexes() {
    let wrapping_vec = WrappingIndex(vec![1, 2, 3]);
    assert_eq!(1, wrapping_vec[3_usize]);
    assert_eq!(2, wrapping_vec[4_usize]);
    assert_eq!(3, wrapping_vec[5_usize]);
}

#[test] // ✅
fn neg_indexes() {
    let wrapping_vec = WrappingIndex(vec![1, 2, 3]);
    assert_eq!(1, wrapping_vec[-3_i128]);
    assert_eq!(2, wrapping_vec[-2_i128]);
    assert_eq!(3, wrapping_vec[-1_i128]);
}

#[test] // ✅
fn wrapping_neg_indexes() {
    let wrapping_vec = WrappingIndex(vec![1, 2, 3]);
    assert_eq!(1, wrapping_vec[-6_i128]);
    assert_eq!(2, wrapping_vec[-5_i128]);
    assert_eq!(3, wrapping_vec[-4_i128]);
}
```

> There's no requirement that the `Idx` type has to be a number type or a `Range`, it could be an enum! Here's an example using basketball positions to index into a basketball team to retrieve players on the team:

`Idx` 的类型并不非得是数字类型或 `Range` 类型，甚至还可以是枚举！例如我们可以在一支篮球队中，对打什么位置索引从而得到队伍里打这个位置的队员：

```rust
use std::ops::Index;

enum BasketballPosition {
    PointGuard,
    ShootingGuard,
    Center,
    PowerForward,
    SmallForward,
}

struct BasketballPlayer {
    name: &'static str,
    position: BasketballPosition,
}

struct BasketballTeam {
    point_guard: BasketballPlayer,
    shooting_guard: BasketballPlayer,
    center: BasketballPlayer,
    power_forward: BasketballPlayer,
    small_forward: BasketballPlayer,
}

impl Index<BasketballPosition> for BasketballTeam {
    type Output = BasketballPlayer;
    fn index(&self, position: BasketballPosition) -> &BasketballPlayer {
        match position {
            BasketballPosition::PointGuard => &self.point_guard,
            BasketballPosition::ShootingGuard => &self.shooting_guard,
            BasketballPosition::Center => &self.center,
            BasketballPosition::PowerForward => &self.power_forward,
            BasketballPosition::SmallForward => &self.small_forward,
        }
    }
}
```



#### Drop

预备知识
- [Self](#self)
- [Methods](#methods)

```rust
trait Drop {
    fn drop(&mut self);
}
```

> If a type impls `Drop` then `drop` will be called on the type when it goes out of scope but before it's destroyed. We will rarely need to impl this for our types but a good example of where it's useful is if a type holds on to some external resources which needs to be cleaned up when the type is destroyed.

对于实现 `Drop` 特性的类型，在该类型脱离作用域并销毁前，其 `drop` 方法会被调用。通常，不必为我们的类型实现这一特性，除非该类型持有某种外部的资源，且该资源需要显式释放。

> There's a `BufWriter` type in the standard library that allows us to buffer writes to `Write` types. However, what if the `BufWriter` gets destroyed before the content in its buffer has been flushed to the underlying `Write` type? Thankfully that's not possible! The `BufWriter` impls the `Drop` trait so that `flush` is always called on it whenever it goes out of scope!

标准库中的 `BufWriter` 类型允许我们对向 `Write` 类型写入的时候进行缓存。显然，当 `BufWriter` 销毁前应当把缓存的内容写入 `Writer` 实例，这就是 `Drop` 所允许我们做到的！对于实现了 `Drop` 的 `BufWriter` 来说，其实例在销毁前会总会调用 `flush` 方法。

```rust
impl<W: Write> Drop for BufWriter<W> {
    fn drop(&mut self) {
        self.flush_buf();
    }
}
```

> Also, `Mutex`s in Rust don't have `unlock()` methods because they don't need them! Calling `lock()` on a `Mutex` returns a `MutexGuard` which automatically unlocks the `Mutex` when it goes out of scope thanks to its `Drop` impl:

并且，在 Rust 中 `Mutex` 类型之所以没有 `unlock()` 方法，就是因为它完全不需要！鉴于 `Drop` 特性的实现，调用 `Mutex` 的 `lock()` 方法返回的 `MutexGuard` 类型，在脱离作用域时会自动地释放 `Mutex` 。

```rust
impl<T: ?Sized> Drop for MutexGuard<'_, T> {
    fn drop(&mut self) {
        unsafe {
            self.lock.inner.raw_unlock();
        }
    }
}
```

> In general, if you're impling an abstraction over some resource that needs to be cleaned up after use then that's a great reason to make use of the `Drop` trait.

简而言之，如果你正在设计某种需要显示释放的资源的抽象包装，那么这正是 `Drop` 特性大显神威的地方。



## 转换特性 Conversion Traits



### From & Into

预备知识
- [Self](#self)
- [Functions](#functions)
- [Methods](#methods)
- [Generic Parameters](#generic-parameters)
- [Generic Blanket Impls](#generic-blanket-impls)

```rust
trait From<T> {
    fn from(T) -> Self;
}
```

> `From<T>` types allow us to convert `T` into `Self`.

实现 `From<T>` 特性的类型允许我们从 `T` 类型转换到自身的类型 `Self` 。

```rust
trait Into<T> {
    fn into(self) -> T;
}
```

> `Into<T>` types allow us to convert `Self` into `T`.

实现 `Into<T>` 特性的类型允许我们从自身的类型 `Self` 转换到 `T` 类型。

> These traits are two different sides of the same coin. We can only impl `From<T>` for our types because the `Into<T>` impl is automatically provided by this generic blanket impl:

这是一对恰好相反的特性，如同一枚硬币的两面。注意，我们只能手动实现 `From<T>` 特性，而不能手动实现 `Into<T>` 特性，因为 `Into<T>` 特性已经被一揽子泛型实现所自动实现。

```rust
impl<T, U> Into<U> for T
where
    U: From<T>,
{
    fn into(self) -> U {
        U::from(self)
    }
}
```

> The reason both traits exist is because it allows us to write trait bounds on generic types slightly differently:

这两个特性同时存在的一个好处在于，我们可以在为泛型类型添加约束的时候，使用两种稍有不同的记号：

```rust
fn function<T>(t: T)
where
    // these bounds are equivalent
    // 以下两种记号等价
    T: From<i32>,
    i32: Into<T>
{
    // these examples are equivalent
    // 以下两种记号等价
    let example: T = T::from(0);
    let example: T = 0.into();
}
```

> There are no hard rules about when to use one or the other, so go with whatever makes the most sense for each situation. Now let's look at some example impls on `Point`:

对于具体使用哪种记号并无一定之规，请根据实际情况做出最恰当的选择。接下来我们看看 `Point` 类型的例子：

```rust
struct Point {
    x: i32,
    y: i32,
}

impl From<(i32, i32)> for Point {
    fn from((x, y): (i32, i32)) -> Self {
        Point { x, y }
    }
}

impl From<[i32; 2]> for Point {
    fn from([x, y]: [i32; 2]) -> Self {
        Point { x, y }
    }
}

fn example() {
    // using From
    let origin = Point::from((0, 0));
    let origin = Point::from([0, 0]);

    // using Into
    let origin: Point = (0, 0).into();
    let origin: Point = [0, 0].into();
}
```

> The impl is not symmetric, so if we'd like to convert `Point`s into tuples and arrays we have to explicitly add those as well:

这样的转换并不是对称的，如果我们想将 `Point` 转换为元组或数组，那么我们需要显式地编写相应的代码：

```rust
struct Point {
    x: i32,
    y: i32,
}

impl From<(i32, i32)> for Point {
    fn from((x, y): (i32, i32)) -> Self {
        Point { x, y }
    }
}

impl From<Point> for (i32, i32) {
    fn from(Point { x, y }: Point) -> Self {
        (x, y)
    }
}

impl From<[i32; 2]> for Point {
    fn from([x, y]: [i32; 2]) -> Self {
        Point { x, y }
    }
}

impl From<Point> for [i32; 2] {
    fn from(Point { x, y }: Point) -> Self {
        [x, y]
    }
}

fn example() {
    // from (i32, i32) into Point
    let point = Point::from((0, 0));
    let point: Point = (0, 0).into();

    // from Point into (i32, i32)
    let tuple = <(i32, i32)>::from(point);
    let tuple: (i32, i32) = point.into();

    // from [i32; 2] into Point
    let point = Point::from([0, 0]);
    let point: Point = [0, 0].into();

    // from Point into [i32; 2]
    let array = <[i32; 2]>::from(point);
    let array: [i32; 2] = point.into();
}
```

> A popular use of `From<T>` is to trim down boilerplate code. Let's say we add a `Triangle` type to our program which contains three `Point`s, here's some of the many ways we can construct it:

借由 `From<T>` 特性，我们可以省却大量编写模板代码的麻烦。例如，我们现在具有一个包含三个 `Point` 的类型 `Triangle` 类型，以下是构造该类型的几种办法：

```rust
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn new(x: i32, y: i32) -> Point {
        Point { x, y }
    }
}

impl From<(i32, i32)> for Point {
    fn from((x, y): (i32, i32)) -> Point {
        Point { x, y }
    }
}

struct Triangle {
    p1: Point,
    p2: Point,
    p3: Point,
}

impl Triangle {
    fn new(p1: Point, p2: Point, p3: Point) -> Triangle {
        Triangle { p1, p2, p3 }
    }
}

impl<P> From<[P; 3]> for Triangle
where
    P: Into<Point>
{
    fn from([p1, p2, p3]: [P; 3]) -> Triangle {
        Triangle {
            p1: p1.into(),
            p2: p2.into(),
            p3: p3.into(),
        }
    }
}

fn example() {
    // manual construction
    let triangle = Triangle {
        p1: Point {
            x: 0,
            y: 0,
        },
        p2: Point {
            x: 1,
            y: 1,
        },
        p3: Point {
            x: 2,
            y: 2,
        },
    };

    // using Point::new
    let triangle = Triangle {
        p1: Point::new(0, 0),
        p2: Point::new(1, 1),
        p3: Point::new(2, 2),
    };

    // using From<(i32, i32)> for Point
    let triangle = Triangle {
        p1: (0, 0).into(),
        p2: (1, 1).into(),
        p3: (2, 2).into(),
    };

    // using Triangle::new + From<(i32, i32)> for Point
    let triangle = Triangle::new(
        (0, 0).into(),
        (1, 1).into(),
        (2, 2).into(),
    );

    // using From<[Into<Point>; 3]> for Triangle
    let triangle: Triangle = [
        (0, 0),
        (1, 1),
        (2, 2),
    ].into();
}
```

> There are no rules for when, how, or why we should impl `From<T>` for our types so it's up to us to use our best judgement for every situation.

对于 `From<T>` 特性的使用并无一定之规，运用你的智慧明智地使用它吧！

> One popular use of `Into<T>` is to make functions which need owned values generic over whether they take owned or borrowed values:

使用 `Into<T>` 特性的一个神奇之处在于，对于那些本来只能接受特定类型参数的函数，现在你可以有更多不同的选择：

```rust
struct Person {
    name: String,
}

impl Person {
    // accepts:
    // - String
    fn new1(name: String) -> Person {
        Person { name }
    }

    // accepts:
    // - String
    // - &String
    // - &str
    // - Box<str>
    // - Cow<'_, str>
    // - char
    // since all of the above types can be converted into String
    fn new2<N: Into<String>>(name: N) -> Person {
        Person { name: name.into() }
    }
}
```



## 错误处理 Error Handling

> The best time to talk about error handling and the `Error` trait is after going over `Display`, `Debug`, `Any`, and `From` but before getting to `TryFrom` hence why the **Error Handling** section awkwardly bisects the **Conversion Traits** section.

讲解错误处理与 `Error` 特性的最佳时机，莫过于在 `Display` ， `Debug` ， `Any` 和 `From` 之后， `TryFrom` 之前，这就是为什么我要将 **错误处理** 这一节硬塞在 **转换特性** 这一章里。

### Error

预备知识
- [Self](#self)
- [Methods](#methods)
- [Default Impls](#default-impls)
- [Generic Blanket Impls](#generic-blanket-impls)
- [Subtraits & Supertraits](#subtraits--supertraits)
- [Trait Objects](#trait-objects)
- [Display & ToString](#display--tostring)
- [Debug](#debug)
- [Any](#any)
- [From & Into](#from--into)

```rust
trait Error: Debug + Display {
    // provided default impls
    // 提供默认实现
    fn source(&self) -> Option<&(dyn Error + 'static)>;
    fn backtrace(&self) -> Option<&Backtrace>;
    fn description(&self) -> &str;
    fn cause(&self) -> Option<&dyn Error>;
}
```

> In Rust errors are returned, not thrown. Let's look at some examples.

在 Rust 中，错误是被返回的，而不是被抛出的。让我们看看下面的例子：

> Since dividing integer types by zero panics if we wanted to make our program safer and more explicit we could impl a `safe_div` function which returns a `Result` instead like this:

由于整数的除零操作会导致 panic ，为了程序的健壮性，我们显式地实现了安全的 `safe_div` 除法函数，它的返回值是 `Result` ：

```rust
use std::fmt;
use std::error;

#[derive(Debug, PartialEq)]
struct DivByZero;

impl fmt::Display for DivByZero {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "division by zero error")
    }
}

impl error::Error for DivByZero {}

fn safe_div(numerator: i32, denominator: i32) -> Result<i32, DivByZero> {
    if denominator == 0 {
        return Err(DivByZero);
    }
    Ok(numerator / denominator)
}

#[test] // ✅
fn test_safe_div() {
    assert_eq!(safe_div(8, 2), Ok(4));
    assert_eq!(safe_div(5, 0), Err(DivByZero));
}
```

> Since errors are returned and not thrown they must be explicitly handled, and if the current function cannot handle an error it should propagate it up to the caller. The most idiomatic way to propagate errors is to use the `?` operator, which is just syntax sugar for the now deprecated `try!` macro which simply does this:

由于错误是被返回的，而不是被抛出的，它们必须被显式地处理。如果当前函数没有处理该错误的能力，那么该错误应当原路返回到上一级调用函数。最慣用的返回错误的方法是使用 `?` 算符，它是现在已经过时的 `try!` 宏的语法糖：

```rust
macro_rules! try {
    ($expr:expr) => {
        match $expr {
            // if Ok just unwrap the value
            // 正常情况下直接解除 Result 的包装
            Ok(val) => val,
            // if Err map the err value using From and return
            // 否则将该错误进行适当转换后，返回到上级调用函数
            Err(err) => {
                return Err(From::from(err));
            }
        }
    };
}
```

> If we wanted to write a function which reads a file into a `String` we could write it like this, propagating the `io::Error`s using `?` everywhere they can appear:

例如，如果我们的函数其功能是将文件读为一个 `String` ，那么使用 `?` 算符来将可能的错误 `io::Error` 返回给上级调用函数就很方便：

```rust
use std::io::Read;
use std::path::Path;
use std::io;
use std::fs::File;

fn read_file_to_string(path: &Path) -> Result<String, io::Error> {
    let mut file = File::open(path)?; // ⬆️ io::Error
    let mut contents = String::new();
    file.read_to_string(&mut contents)?; // ⬆️ io::Error
    Ok(contents)
}
```

> But let's say the file we're reading is actually a list of numbers and we want to sum them together, we'd update our function like this:

又例如，如果我们的文件是一系列数字，我们想将它们加在一起，可以这样编写代码：

```rust
use std::io::Read;
use std::path::Path;
use std::io;
use std::fs::File;

fn sum_file(path: &Path) -> Result<i32, /* What to put here? */> {
                                        // 这里填写什么类型好呢？
    let mut file = File::open(path)?; // ⬆️ io::Error
    let mut contents = String::new();
    file.read_to_string(&mut contents)?; // ⬆️ io::Error
    let mut sum = 0;
    for line in contents.lines() {
        sum += line.parse::<i32>()?; // ⬆️ ParseIntError
    }
    Ok(sum)
}
```

> But what's the error type of our `Result` now? It can return either an `io::Error` or a `ParseIntError`. We're going to look at three approaches for solving this problem, starting with the most quick & dirty way and finishing with the most robust way.

现在 `Rusult` 的类型又如何？该函数内部可能产生 `io::Error` 或 `ParseIntError` 两种错误。我们将介绍三种解决此类问题的方法，从最简单但不优雅的方法，到最健壮的方法：

> The first approach is recognizing that all types which impl `Error` also impl `Display` so we can map all the errors to `String`s and use `String` as our error type:

方法一，我们注意到，所有实现了 `Error` 的类型同时也实现了 `Display` ，因此我们可以将错误映射到 `String` 并以此为错误类型：

```rust
use std::fs::File;
use std::io;
use std::io::Read;
use std::path::Path;

fn sum_file(path: &Path) -> Result<i32, String> {
    let mut file = File::open(path)
        .map_err(|e| e.to_string())?; // ⬆️ io::Error -> String
    let mut contents = String::new();
    file.read_to_string(&mut contents)
        .map_err(|e| e.to_string())?; // ⬆️ io::Error -> String
    let mut sum = 0;
    for line in contents.lines() {
        sum += line.parse::<i32>()
            .map_err(|e| e.to_string())?; // ⬆️ ParseIntError -> String
    }
    Ok(sum)
}
```

> The obvious downside of stringifying every error is that we throw away type information which makes it harder for the caller to handle the errors.

此方法的明显缺点在于，由于我们将所有的错误都序列化了，以至于丢弃了该错误的类型信息，这对于上级调用函数错误处理来讲，就不是那么方便了。

> One nonobvious upside to the above approach is we can customize the strings to provide more context-specific information. For example, `ParseIntError` usually stringifies to `"invalid digit found in string"` which is very vague and doesn't mention what the invalid string is or what integer type it was trying to parse into. If we were debugging this problem that error message would almost be useless. However we can make it significantly better by providing all the context relevant information ourselves:

但此方法也有一个不明显的优点，那就是我们可以使用自定义的字符串，来提供丰富的上下文错误信息。例如，`ParseIntError` 通常序列化为 `"invalid digit found in string"` 这样模棱两可的文本，既没有提及无效的字符串是什么，也没有提及它要转换到什么样的数字类型。这样的信息对于我们调试程序来讲几乎没有什么帮助。不过我们可以提供更有意义的，且上下文相关的信息来明显改善这一点：

```rust
sum += line.parse::<i32>()
    .map_err(|_| format!("failed to parse {} into i32", line))?;
```

> The second approach takes advantage of this generic blanket impl from the standard library:

方法二，利用标准库的一揽子泛型实现：

```rust
impl<E: error::Error> From<E> for Box<dyn error::Error>;
```

> Which means that any `Error` type can be implicitly converted into a `Box<dyn error::Error>` by the `?` operator, so we can set to error type to `Box<dyn error::Error>` in the `Result` return type of any function which produces errors and the `?` operator will do the rest of the work for us:

所有实现了 `Error` 特性的类型都可以隐式地使用 `?` 转换为 `Box<dyn error::Error>` 类型。所以我们可以将 `Rusult` 的错误类型设为 `Box<dyn error::Error>` 类型，然后 `?` 算符会帮我们实现这一隐式转换。

```rust
use std::fs::File;
use std::io::Read;
use std::path::Path;
use std::error;

fn sum_file(path: &Path) -> Result<i32, Box<dyn error::Error>> {
    let mut file = File::open(path)?; // ⬆️ io::Error -> Box<dyn error::Error>
    let mut contents = String::new();
    file.read_to_string(&mut contents)?; // ⬆️ io::Error -> Box<dyn error::Error>
    let mut sum = 0;
    for line in contents.lines() {
        sum += line.parse::<i32>()?; // ⬆️ ParseIntError -> Box<dyn error::Error>
    }
    Ok(sum)
}
```

> While being more concise, this seems to suffer from the same downside of the previous approach by throwing away type information. This is mostly true, but if the caller is aware of the impl details of our function they can still handle the different errors types using the `downcast_ref()` method on `error::Error` which works the same as it does on `dyn Any` types:

这看起来似乎有与第一种方法一样的缺点，丢弃了错误的类型信息。有时确实如此，但倘若上级调用函数知悉该函数的实现细节，那么它仍然可以通过 `error::Error` 特性的 `downcast_ref()` 方法来分辨错误的具体类型，这与实现了 `dyn Any` 特性的类型是一样的：

```rust
fn handle_sum_file_errors(path: &Path) {
    match sum_file(path) {
        Ok(sum) => println!("the sum is {}", sum),
        Err(err) => {
            if let Some(e) = err.downcast_ref::<io::Error>() {
                // handle io::Error
            } else if let Some(e) = err.downcast_ref::<ParseIntError>() {
                // handle ParseIntError
            } else {
                // we know sum_file can only return one of the
                // above errors so this branch is unreachable
                // 由于我们知道该函数只能返回以上两种错误，
                // 所以这一选择肢一般是不可能执行的
                unreachable!();
            }
        }
    }
}
```

> The third approach, which is the most robust and type-safe way to aggregate these different errors would be to build our own custom error type using an enum:

方法三，处理错误的最健壮和类型安全的方法，是通过枚举来构建我们自己的错误类型：

```rust
use std::num::ParseIntError;
use std::fs::File;
use std::io;
use std::io::Read;
use std::path::Path;
use std::error;
use std::fmt;

#[derive(Debug)]
enum SumFileError {
    Io(io::Error),
    Parse(ParseIntError),
}

impl From<io::Error> for SumFileError {
    fn from(err: io::Error) -> Self {
        SumFileError::Io(err)
    }
}

impl From<ParseIntError> for SumFileError {
    fn from(err: ParseIntError) -> Self {
        SumFileError::Parse(err)
    }
}

impl fmt::Display for SumFileError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            SumFileError::Io(err) => write!(f, "sum file error: {}", err),
            SumFileError::Parse(err) => write!(f, "sum file error: {}", err),
        }
    }
}

impl error::Error for SumFileError {
    // the default impl for this method always returns None
    // but we can now override it to make it way more useful!
    // 在默认实现中，该方法总是返回 None ，现在重写它！
    fn source(&self) -> Option<&(dyn error::Error + 'static)> {
        Some(match self {
            SumFileError::Io(err) => err,
            SumFileError::Parse(err) => err,
        })
    }
}

fn sum_file(path: &Path) -> Result<i32, SumFileError> {
    let mut file = File::open(path)?; // ⬆️ io::Error -> SumFileError
    let mut contents = String::new();
    file.read_to_string(&mut contents)?; // ⬆️ io::Error -> SumFileError
    let mut sum = 0;
    for line in contents.lines() {
        sum += line.parse::<i32>()?; // ⬆️ ParseIntError -> SumFileError
    }
    Ok(sum)
}

fn handle_sum_file_errors(path: &Path) {
    match sum_file(path) {
        Ok(sum) => println!("the sum is {}", sum),
        Err(SumFileError::Io(err)) => {
            // handle io::Error
        },
        Err(SumFileError::Parse(err)) => {
            // handle ParseIntError
        },
    }
}
```



## 转换特性深入 Conversion Traits Continued



### TryFrom & TryInto

预备知识
- [Self](#self)
- [Functions](#functions)
- [Methods](#methods)
- [Associated Types](#associated-types)
- [Generic Parameters](#generic-parameters)
- [Generic Types vs Associated Types](#generic-types-vs-associated-types)
- [Generic Blanket Impls](#generic-blanket-impls)
- [From & Into](#from--into)
- [Error](#error)

> `TryFrom` and `TryInto` are the fallible versions of `From` and `Into`.

`TryFrom` 和 `TryInto` 是可能失败版本的 `From` 和 `Into` 。

```rust
trait TryFrom<T> {
    type Error;
    fn try_from(value: T) -> Result<Self, Self::Error>;
}

trait TryInto<T> {
    type Error;
    fn try_into(self) -> Result<T, Self::Error>;
}
```

> Similarly to `Into` we cannot impl `TryInto` because its impl is provided by this generic blanket impl:

与 `Into` 相似地，我们不能手动实现 `TryInto` ，因为它已经为一揽子泛型实现所提供。

```rust
impl<T, U> TryInto<U> for T
where
    U: TryFrom<T>,
{
    type Error = U::Error;

    fn try_into(self) -> Result<U, U::Error> {
        U::try_from(self)
    }
}
```

> Let's say that in the context of our program it doesn't make sense for `Point`s to have `x` and `y` values that are less than `-1000` or greater than `1000`. This is how we'd rewrite our earlier `From` impls using `TryFrom` to signal to the users of our type that this conversion can now fail:

例如，我们的程序要求 `Point` 的 `x` 和 `y` 的值必须要处于 `-1000` 到 `1000` 之间，相较于 `From` ，使用 `TryFrom` 可以告知上级调用者，某些转换可能失败了。

```rust
use std::convert::TryFrom;
use std::error;
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

#[derive(Debug)]
struct OutOfBounds;

impl fmt::Display for OutOfBounds {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "out of bounds")
    }
}

impl error::Error for OutOfBounds {}

// now fallible
// TryFrom 的转换允许失败
impl TryFrom<(i32, i32)> for Point {
    type Error = OutOfBounds;
    fn try_from((x, y): (i32, i32)) -> Result<Point, OutOfBounds> {
        if x.abs() > 1000 || y.abs() > 1000 {
            return Err(OutOfBounds);
        }
        Ok(Point { x, y })
    }
}

// still infallible
// From 的转换不允许失败
impl From<Point> for (i32, i32) {
    fn from(Point { x, y }: Point) -> Self {
        (x, y)
    }
}
```

> And here's the refactored `TryFrom<[TryInto<Point>; 3]>` impl for `Triangle`:

现在，我们对 `Triangle` 使用 `TryFrom<[TryInto<Point>; 3]>` 进行重构：

```rust
use std::convert::{TryFrom, TryInto};
use std::error;
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

#[derive(Debug)]
struct OutOfBounds;

impl fmt::Display for OutOfBounds {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "out of bounds")
    }
}

impl error::Error for OutOfBounds {}

impl TryFrom<(i32, i32)> for Point {
    type Error = OutOfBounds;
    fn try_from((x, y): (i32, i32)) -> Result<Self, Self::Error> {
        if x.abs() > 1000 || y.abs() > 1000 {
            return Err(OutOfBounds);
        }
        Ok(Point { x, y })
    }
}

struct Triangle {
    p1: Point,
    p2: Point,
    p3: Point,
}

impl<P> TryFrom<[P; 3]> for Triangle
where
    P: TryInto<Point>,
{
    type Error = P::Error;
    fn try_from([p1, p2, p3]: [P; 3]) -> Result<Self, Self::Error> {
        Ok(Triangle {
            p1: p1.try_into()?,
            p2: p2.try_into()?,
            p3: p3.try_into()?,
        })
    }
}

fn example() -> Result<Triangle, OutOfBounds> {
    let t: Triangle = [(0, 0), (1, 1), (2, 2)].try_into()?;
    Ok(t)
}
```



### FromStr

预备知识
- [Self](#self)
- [Functions](#functions)
- [Associated Types](#associated-types)
- [Error](#error)
- [TryFrom & TryInto](#tryfrom--tryinto)

```rust
trait FromStr {
    type Err;
    fn from_str(s: &str) -> Result<Self, Self::Err>;
}
```

> `FromStr` types allow performing a fallible conversion from `&str` into `Self`. The idiomatic way to use `FromStr` is to call the `.parse()` method on `&str`s:

实现 `FromStr` 特性的类型允许可失败地从 `&str` 转换至 `Self` 。使用这一特性的慣用方式是，调用 `&str` 实例的 `.parse()` 方法：

```rust
use std::str::FromStr;

fn example<T: FromStr>(s: &'static str) {
    // these are all equivalent
    // 以下方法互相等价
    let t: Result<T, _> = FromStr::from_str(s);
    let t = T::from_str(s);
    let t: Result<T, _> = s.parse();
    let t = s.parse::<T>(); // most idiomatic
                            // 最慣用的使用方式
}
```

> Example impl for `Point`:

下例为 `Point` 实现了 `FromStr` 特性：

```rust
use std::error;
use std::fmt;
use std::iter::Enumerate;
use std::num::ParseIntError;
use std::str::{Chars, FromStr};

#[derive(Debug, Eq, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn new(x: i32, y: i32) -> Self {
        Point { x, y }
    }
}

#[derive(Debug, PartialEq)]
struct ParsePointError;

impl fmt::Display for ParsePointError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "failed to parse point")
    }
}

impl From<ParseIntError> for ParsePointError {
    fn from(_e: ParseIntError) -> Self {
        ParsePointError
    }
}

impl error::Error for ParsePointError {}

impl FromStr for Point {
    type Err = ParsePointError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let is_num = |(_, c): &(usize, char)| matches!(c, '0'..='9' | '-');
        let isnt_num = |t: &(_, _)| !is_num(t);

        let get_num =
            |char_idxs: &mut Enumerate<Chars<'_>>| -> Result<(usize, usize), ParsePointError> {
                let (start, _) = char_idxs
                    .skip_while(isnt_num)
                    .next()
                    .ok_or(ParsePointError)?;
                let (end, _) = char_idxs
                    .skip_while(is_num)
                    .next()
                    .ok_or(ParsePointError)?;
                Ok((start, end))
            };

        let mut char_idxs = s.chars().enumerate();
        let (x_start, x_end) = get_num(&mut char_idxs)?;
        let (y_start, y_end) = get_num(&mut char_idxs)?;

        let x = s[x_start..x_end].parse::<i32>()?;
        let y = s[y_start..y_end].parse::<i32>()?;

        Ok(Point { x, y })
    }
}

#[test] // ✅
fn pos_x_y() {
    let p = "(4, 5)".parse::<Point>();
    assert_eq!(p, Ok(Point::new(4, 5)));
}

#[test] // ✅
fn neg_x_y() {
    let p = "(-6, -2)".parse::<Point>();
    assert_eq!(p, Ok(Point::new(-6, -2)));
}

#[test] // ✅
fn not_a_point() {
    let p = "not a point".parse::<Point>();
    assert_eq!(p, Err(ParsePointError));
}
```

> `FromStr` has the same signature as `TryFrom<&str>`. It doesn't matter which one we impl for a type first as long as we forward the impl to the other one. Here's a `TryFrom<&str>` impl for `Point` assuming it already has a `FromStr` impl:

`FromStr` 与 `TryFrom<&str>` 具有相同的函数签名。先实现哪个特性无关紧要，因为我们可以利用先实现的特性实现后实现的特性。例如，我们假定 `Point` 类型已经实现了 `FromStr` 特性，再来实现 `TryFrom<&str>` 特性：

```rust
impl TryFrom<&str> for Point {
    type Error = <Point as FromStr>::Err;
    fn try_from(s: &str) -> Result<Point, Self::Error> {
        <Point as FromStr>::from_str(s)
    }
}
```


### AsRef & AsMut

预备知识
- [Self](#self)
- [Methods](#methods)
- [Sized](#sized)
- [Generic Parameters](#generic-parameters)
- [Sized](#sized)
- [Deref & DerefMut](#deref--derefmut)

```rust
trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}

trait AsMut<T: ?Sized> {
    fn as_mut(&mut self) -> &mut T;
}
```

> `AsRef` is for cheap reference to reference conversions. However, one of the most common ways it's used is to make functions generic over whether they take ownership or not:

`AsRef` 特性的存在很大程度上便捷了引用转换，其最常见的使用是为函数的引用类型的参数的传入提供方便：

```rust
// accepts:
//  - &str
//  - &String
fn takes_str(s: &str) {
    // use &str
}

// accepts:
//  - &str
//  - &String
//  - String
fn takes_asref_str<S: AsRef<str>>(s: S) {
    let s: &str = s.as_ref();
    // use &str
}

fn example(slice: &str, borrow: &String, owned: String) {
    takes_str(slice);
    takes_str(borrow);
    takes_str(owned); // ❌
    takes_asref_str(slice);
    takes_asref_str(borrow);
    takes_asref_str(owned); // ✅
}
```

> The other most common use-case is returning a reference to inner private data wrapped by a type which protects some invariant. A good example from the standard library is `String` which is just a wrapper around `Vec<u8>`:

另外一个常见的使用是，返回一个包装类型的内部私有数据的引用（该类型用于保证内部私有数据的不变性）。标准库中的 `String` 就是对 `Vec<u8>` 的这样一种包装：

```rust
struct String {
    vec: Vec<u8>,
}
```

> This inner `Vec` cannot be made public because if it was people could mutate any byte and break the `String`'s valid UTF-8 encoding. However, it's safe to expose an immutable read-only reference to the inner byte array, hence this impl:

之所以不公开内部的 `Vec` 数据，是因为一旦允许用户随意修改内部数据，就有可能破环 `String` 有效的 UTF-8 编码。但是，对外开放一个只读的字节数组的引用是安全的，所以有如下实现：

```rust
impl AsRef<[u8]> for String;
```

> Generally, it often only makes sense to impl `AsRef` for a type if it wraps some other type to either provide additional functionality around the inner type or protect some invariant on the inner type.

通常来讲我们不对类型实现 `AsRef` 特性，除非该类型包装了其它类型以提供额外的功能，或是对内部类型提供了不变性的保护。

> Let's examine a example of bad `AsRef` impls:

以下是实现 `AsRef` 特性的一个反例：

```rust
struct User {
    name: String,
    age: u32,
}

impl AsRef<String> for User {
    fn as_ref(&self) -> &String {
        &self.name
    }
}

impl AsRef<u32> for User {
    fn as_ref(&self) -> &u32 {
        &self.age
    }
}
```

> This works and kinda makes sense at first, but quickly falls apart if we add more members to `User`:

乍看起来这似乎有几分道理，但是当我们对 `User` 类型添加新的成员时，缺点就暴露出来了：

```rust
struct User {
    name: String,
    email: String,
    age: u32,
    height: u32,
}

impl AsRef<String> for User {
    fn as_ref(&self) -> &String {
        // uh, do we return name or email here?
        // 既然我们要返回一个字符串引用，那具体应该返回什么呢？
        // name 和 email 都是字符串，如何选择呢？
        // 出于返回类型的限制，似乎我们也难以返回一个混合的字符串。
    }
}

impl AsRef<u32> for User {
    fn as_ref(&self) -> &u32 {
        // uh, do we return age or height here?
        // 如上同理
    }
}
```

> A `User` is composed of `String`s and `u32`s but it's not really the same thing as a `String` or a `u32`. Even if we had much more specific types:

`User` 类型由多个 `String` 和 `u32` 类型的成员所组成，但我们也不能说 `User` 是 `String` 或 `u32` 吧？即便由更加具体的类型来构造也不行：

```rust
struct User {
    name: Name,
    email: Email,
    age: Age,
    height: Height,
}
```

> It wouldn't make much sense to impl `AsRef` for any of those because `AsRef` is for cheap reference to reference conversions between semantically equivalent things, and `Name`, `Email`, `Age`, and `Height` by themselves are not the same thing as a `User`.

对于 `User` 这样的类型来讲，实现 `AsRef` 特性并没有什么太多意义。因为 `AsRef` 的存在仅是为了做一种最简单的引用转换，这种转换最好存在于语义上相类似的事务之间。`Name`，`Email`，`Age` 和 `Height` 其本身和 `User` 就不是一回事，在逻辑上谈不上转换。

> A good example where we would impl `AsRef` would be if we introduced a new type `Moderator` that just wrapped a `User` and added some moderation specific privileges:

下例展示了 `AsRef` 特性的正确用法，我们实现了一个新的类型 `Moderator`，它仅仅是包装了 `User` 类型，并添加了对其权限的一些控制：

```rust
struct User {
    name: String,
    age: u32,
}

// unfortunately the standard library cannot provide
// a generic blanket impl to save us from this boilerplate
// 不幸的是，标准库并没有提供相应的一揽子泛型实现，我们不得不手动实现
impl AsRef<User> for User {
    fn as_ref(&self) -> &User {
        self
    }
}

enum Privilege {
    BanUsers,
    EditPosts,
    DeletePosts,
}

// although Moderators have some special
// privileges they are still regular Users
// and should be able to do all the same stuff
// 尽管主持人类具有一些特殊的权限，
// 但其仍然是普通的用户
// 所有用户类能做到的主持人类也应能做到
struct Moderator {
    user: User,
    privileges: Vec<Privilege>
}

impl AsRef<Moderator> for Moderator {
    fn as_ref(&self) -> &Moderator {
        self
    }
}

impl AsRef<User> for Moderator {
    fn as_ref(&self) -> &User {
        &self.user
    }
}

// this should be callable with Users
// and Moderators (who are also Users)
// 这个函数的参数可以是 User 也可以是 Moderator
// （ Moderator 也是 User ）
fn create_post<U: AsRef<User>>(u: U) {
    let user = u.as_ref();
    // etc
}

fn example(user: User, moderator: Moderator) {
    create_post(&user);
    create_post(&moderator); // ✅
}
```

> This works because `Moderator`s are just `User`s. Here's the example from the `Deref` section except using `AsRef` instead:

之所以可以这样做，是因为 `Moderator` 就是 `User` 。下例是将 `Deref` 一节中的例子使用 `AsRef` 做出替代：

```rust
use std::convert::AsRef;

struct Human {
    health_points: u32,
}

impl AsRef<Human> for Human {
    fn as_ref(&self) -> &Human {
        self
    }
}

enum Weapon {
    Spear,
    Axe,
    Sword,
}

// a Soldier is just a Human with a Weapon
// 士兵是手持武器的人类
struct Soldier {
    human: Human,
    weapon: Weapon,
}

impl AsRef<Soldier> for Soldier {
    fn as_ref(&self) -> &Soldier {
        self
    }
}

impl AsRef<Human> for Soldier {
    fn as_ref(&self) -> &Human {
        &self.human
    }
}

enum Mount {
    Horse,
    Donkey,
    Cow,
}

// a Knight is just a Soldier with a Mount
// 骑士是胯骑坐骑的士兵
struct Knight {
    soldier: Soldier,
    mount: Mount,
}

impl AsRef<Knight> for Knight {
    fn as_ref(&self) -> &Knight {
        self
    }
}

impl AsRef<Soldier> for Knight {
    fn as_ref(&self) -> &Soldier {
        &self.soldier
    }
}

impl AsRef<Human> for Knight {
    fn as_ref(&self) -> &Human {
        &self.soldier.human
    }
}

enum Spell {
    MagicMissile,
    FireBolt,
    ThornWhip,
}

// a Mage is just a Human who can cast Spells
// 法师是口诵咒语的人类
struct Mage {
    human: Human,
    spells: Vec<Spell>,
}

impl AsRef<Mage> for Mage {
    fn as_ref(&self) -> &Mage {
        self
    }
}

impl AsRef<Human> for Mage {
    fn as_ref(&self) -> &Human {
        &self.human
    }
}

enum Staff {
    Wooden,
    Metallic,
    Plastic,
}

// a Wizard is just a Mage with a Staff
// 巫师是腰别法宝的法师
struct Wizard {
    mage: Mage,
    staff: Staff,
}

impl AsRef<Wizard> for Wizard {
    fn as_ref(&self) -> &Wizard {
        self
    }
}

impl AsRef<Mage> for Wizard {
    fn as_ref(&self) -> &Mage {
        &self.mage
    }
}

impl AsRef<Human> for Wizard {
    fn as_ref(&self) -> &Human {
        &self.mage.human
    }
}

fn borrows_human<H: AsRef<Human>>(human: H) {}
fn borrows_soldier<S: AsRef<Soldier>>(soldier: S) {}
fn borrows_knight<K: AsRef<Knight>>(knight: K) {}
fn borrows_mage<M: AsRef<Mage>>(mage: M) {}
fn borrows_wizard<W: AsRef<Wizard>>(wizard: W) {}

fn example(human: Human, soldier: Soldier, knight: Knight, mage: Mage, wizard: Wizard) {
    // all types can be used as Humans
    borrows_human(&human);
    borrows_human(&soldier);
    borrows_human(&knight);
    borrows_human(&mage);
    borrows_human(&wizard);
    // Knights can be used as Soldiers
    borrows_soldier(&soldier);
    borrows_soldier(&knight);
    // Wizards can be used as Mages
    borrows_mage(&mage);
    borrows_mage(&wizard);
    // Knights & Wizards passed as themselves
    borrows_knight(&knight);
    borrows_wizard(&wizard);
}
```

> `Deref` didn't work in the prior version of the example above because deref coercion is an implicit conversion between types which leaves room for people to mistakenly formulate the wrong ideas and expectations for how it will behave. `AsRef` works above because it makes the conversion between types explicit and there's no room leftover to develop any wrong ideas or expectations.

之所以 `Deref` 在上例之前的版本中不可使用，是因为自动解引用是一种隐式的转换，这就为程序员错误地使用留下了巨大的空间。

而 `AsRef` 在上例中可以使用，是因为其实现的转换是显式的，这样很大程度上就消除了犯错误的空间。



### Borrow & BorrowMut

预备知识
- [Self](#self)
- [Methods](#methods)
- [Generic Parameters](#generic-parameters)
- [Subtraits & Supertraits](#subtraits--supertraits)
- [Sized](#sized)
- [AsRef & AsMut](#asref--asmut)
- [PartialEq & Eq](#partialeq--eq)
- [Hash](#hash)
- [PartialOrd & Ord](#partialord--ord)

```rust
trait Borrow<Borrowed> 
where
    Borrowed: ?Sized, 
{
    fn borrow(&self) -> &Borrowed;
}

trait BorrowMut<Borrowed>: Borrow<Borrowed> 
where
    Borrowed: ?Sized, 
{
    fn borrow_mut(&mut self) -> &mut Borrowed;
}
```

> These traits were invented to solve the very specific problem of looking up `String` keys in `HashSet`s, `HashMap`s, `BTreeSet`s, and `BTreeMap`s using `&str` values.

这类特性存在的意义旨在于解决特定领域的问题，例如在 `Hashset`，`HashMap`，`BTreeSet`，`BtreeMap` 中使用 `&str` 查询 `String` 类型的键。

> We can view `Borrow<T>` and `BorrowMut<T>` as stricter versions of `AsRef<T>` and `AsMut<T>`, where the returned reference `&T` has equivalent `Eq`, `Hash`, and `Ord` impls to `Self`. This is more easily explained with a commented example:

我们可以将 `Borrow<T>` 和 `BorrowMut<T>` 视作 `AsRef<T>` 和 `AsMut<T>` 的严格版本，其返回的引用 `&T` 具有与 `Self` 相同的 `Eq`，`Hash` 和 `Ord` 的实现。这一点在下例的注释中得到很好的解释：

```rust
use std::borrow::Borrow;
use std::hash::Hasher;
use std::collections::hash_map::DefaultHasher;
use std::hash::Hash;

fn get_hash<T: Hash>(t: T) -> u64 {
    let mut hasher = DefaultHasher::new();
    t.hash(&mut hasher);
    hasher.finish()
}

fn asref_example<Owned, Ref>(owned1: Owned, owned2: Owned)
where
    Owned: Eq + Ord + Hash + AsRef<Ref>,
    Ref: Eq + Ord + Hash
{
    let ref1: &Ref = owned1.as_ref();
    let ref2: &Ref = owned2.as_ref();
    
    // refs aren't required to be equal if owned types are equal
    // 值相等，不意味着其引用一定相等
    assert_eq!(owned1 == owned2, ref1 == ref2); // ❌
    
    let owned1_hash = get_hash(&owned1);
    let owned2_hash = get_hash(&owned2);
    let ref1_hash = get_hash(&ref1);
    let ref2_hash = get_hash(&ref2);
    
    // ref hashes aren't required to be equal if owned type hashes are equal
    // 值的哈希值相等，其引用不一定相等
    assert_eq!(owned1_hash == owned2_hash, ref1_hash == ref2_hash); // ❌
    
    // ref comparisons aren't required to match owned type comparisons
    // 值的比较，与其引用的比较没有必然联系
    assert_eq!(owned1.cmp(&owned2), ref1.cmp(&ref2)); // ❌
}

fn borrow_example<Owned, Borrowed>(owned1: Owned, owned2: Owned)
where
    Owned: Eq + Ord + Hash + Borrow<Borrowed>,
    Borrowed: Eq + Ord + Hash
{
    let borrow1: &Borrowed = owned1.borrow();
    let borrow2: &Borrowed = owned2.borrow();
    
    // borrows are required to be equal if owned types are equal
    // 值相等，借用值也必须相等
    assert_eq!(owned1 == owned2, borrow1 == borrow2); // ✅
    
    let owned1_hash = get_hash(&owned1);
    let owned2_hash = get_hash(&owned2);
    let borrow1_hash = get_hash(&borrow1);
    let borrow2_hash = get_hash(&borrow2);
    
    // borrow hashes are required to be equal if owned type hashes are equal
    // 值的哈希值相等，借用值的哈希值也必须相等
    assert_eq!(owned1_hash == owned2_hash, borrow1_hash == borrow2_hash); // ✅
    
    // borrow comparisons are required to match owned type comparisons
    // 值的比较，与借用值的比较必须步调一致
    assert_eq!(owned1.cmp(&owned2), borrow1.cmp(&borrow2)); // ✅
}
```

> It's good to be aware of these traits and understand why they exist since it helps demystify some of the methods on `HashSet`, `HashMap`, `BTreeSet`, and `BTreeMap` but it's very rare that we would ever need to impl these traits for any of our types because it's very rare that we would ever need create a pair of types where one is the "borrowed" version of the other in the first place. If we have some `T` then `&T` will get the job done 99.99% of the time, and `T: Borrow<T>` is already implemented for all `T` because of a generic blanket impl, so we don't need to manually impl it and we don't need to create some `U` such that `T: Borrow<U>`.


理解这类特性存在的意义，有助于我们揭开 `HashSet`，`HashMap`，`BTreeSet` 和 `BTreeMap` 中某些方法的实现的神秘面纱。但是在实际应用中，几乎没有什么地方需要我们去实现这样的特性，因为再难找到一个需要我们对一个值再创造一个“借用”版本的类型的场景了。对于某种类型 `T` ，`&T` 就能解决 99.9% 的问题了，且 `T: Borrow<T>` 已经被一揽子泛型实现对 `T` 实现了，所以我们无需手动实现它，也无需去实现某种的对 `U` 有 `T: Borrow<U>` 了。


### ToOwned

预备知识
- [Self](#self)
- [Methods](#methods)
- [Default Impls](#default-impls)
- [Clone](#clone)
- [Borrow & BorrowMut](#borrow--borrowmut)

```rust
trait ToOwned {
    type Owned: Borrow<Self>;
    fn to_owned(&self) -> Self::Owned;
    
    // provided default impls
    // 提供默认实现
    fn clone_into(&self, target: &mut Self::Owned);
}
```

> `ToOwned` is a more generic version of `Clone`. `Clone` allows us to take a `&T` and turn it into an `T` but `ToOwned` allows us to take a `&Borrowed` and turn it into a `Owned` where `Owned: Borrow<Borrowed>`.

`ToOwned` 特性是 `Clone` 特性的泛型版本。 `Clone` 特性允许我们由 `&T` 类型得到 `T` 类型，而 `ToOwned` 特性允许我们由 `&Borrow` 类型得到 `Owned` 类型，其中 `Owned: Borrow<Borrowed>` 。

> In other words, we can't "clone" a `&str` into a `String`, or a `&Path` into a `PathBuf`, or an `&OsStr` into an `OsString`, since the `clone` method signature doesn't support this kind of cross-type cloning, and that's what `ToOwned` was made for.

换句话讲，我们不能将 `&str` 克隆为 `String`，将 `&Path` 克隆为 `PathBuf` 或将 `&OsStr` 克隆为 `OsString` 。鉴于 `clone` 方法的签名不支持这样跨类型的克隆，这就是 `ToOwned` 特性存在的意义。

> For similar reasons as `Borrow` and `BorrowMut`, it's good to be aware of this trait and understand why it exists but it's very rare we'll ever need to impl it for any of our types.

与 `Borrow` 和 `BorrowMut` 相同地，理解此类特性存在的意义对我们或有帮助，但是鲜少需要我们手动为自己的类实现该特性。



## 迭代特性 Iteration Traits



### Iterator

预备知识
- [Self](#self)
- [Methods](#methods)
- [Associated Types](#associated-types)
- [Default Impls](#default-impls)

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;

    // provided default impls
    // 提供默认实现
    fn size_hint(&self) -> (usize, Option<usize>);
    fn count(self) -> usize;
    fn last(self) -> Option<Self::Item>;
    fn advance_by(&mut self, n: usize) -> Result<(), usize>;
    fn nth(&mut self, n: usize) -> Option<Self::Item>;
    fn step_by(self, step: usize) -> StepBy<Self>;
    fn chain<U>(
        self, 
        other: U
    ) -> Chain<Self, <U as IntoIterator>::IntoIter>
    where
        U: IntoIterator<Item = Self::Item>;
    fn zip<U>(self, other: U) -> Zip<Self, <U as IntoIterator>::IntoIter>
    where
        U: IntoIterator;
    fn map<B, F>(self, f: F) -> Map<Self, F>
    where
        F: FnMut(Self::Item) -> B;
    fn for_each<F>(self, f: F)
    where
        F: FnMut(Self::Item);
    fn filter<P>(self, predicate: P) -> Filter<Self, P>
    where
        P: FnMut(&Self::Item) -> bool;
    fn filter_map<B, F>(self, f: F) -> FilterMap<Self, F>
    where
        F: FnMut(Self::Item) -> Option<B>;
    fn enumerate(self) -> Enumerate<Self>;
    fn peekable(self) -> Peekable<Self>;
    fn skip_while<P>(self, predicate: P) -> SkipWhile<Self, P>
    where
        P: FnMut(&Self::Item) -> bool;
    fn take_while<P>(self, predicate: P) -> TakeWhile<Self, P>
    where
        P: FnMut(&Self::Item) -> bool;
    fn map_while<B, P>(self, predicate: P) -> MapWhile<Self, P>
    where
        P: FnMut(Self::Item) -> Option<B>;
    fn skip(self, n: usize) -> Skip<Self>;
    fn take(self, n: usize) -> Take<Self>;
    fn scan<St, B, F>(self, initial_state: St, f: F) -> Scan<Self, St, F>
    where
        F: FnMut(&mut St, Self::Item) -> Option<B>;
    fn flat_map<U, F>(self, f: F) -> FlatMap<Self, U, F>
    where
        F: FnMut(Self::Item) -> U,
        U: IntoIterator;
    fn flatten(self) -> Flatten<Self>
    where
        Self::Item: IntoIterator;
    fn fuse(self) -> Fuse<Self>;
    fn inspect<F>(self, f: F) -> Inspect<Self, F>
    where
        F: FnMut(&Self::Item);
    fn by_ref(&mut self) -> &mut Self;
    fn collect<B>(self) -> B
    where
        B: FromIterator<Self::Item>;
    fn partition<B, F>(self, f: F) -> (B, B)
    where
        F: FnMut(&Self::Item) -> bool,
        B: Default + Extend<Self::Item>;
    fn partition_in_place<'a, T, P>(self, predicate: P) -> usize
    where
        Self: DoubleEndedIterator<Item = &'a mut T>,
        T: 'a,
        P: FnMut(&T) -> bool;
    fn is_partitioned<P>(self, predicate: P) -> bool
    where
        P: FnMut(Self::Item) -> bool;
    fn try_fold<B, F, R>(&mut self, init: B, f: F) -> R
    where
        F: FnMut(B, Self::Item) -> R,
        R: Try<Ok = B>;
    fn try_for_each<F, R>(&mut self, f: F) -> R
    where
        F: FnMut(Self::Item) -> R,
        R: Try<Ok = ()>;
    fn fold<B, F>(self, init: B, f: F) -> B
    where
        F: FnMut(B, Self::Item) -> B;
    fn fold_first<F>(self, f: F) -> Option<Self::Item>
    where
        F: FnMut(Self::Item, Self::Item) -> Self::Item;
    fn all<F>(&mut self, f: F) -> bool
    where
        F: FnMut(Self::Item) -> bool;
    fn any<F>(&mut self, f: F) -> bool
    where
        F: FnMut(Self::Item) -> bool;
    fn find<P>(&mut self, predicate: P) -> Option<Self::Item>
    where
        P: FnMut(&Self::Item) -> bool;
    fn find_map<B, F>(&mut self, f: F) -> Option<B>
    where
        F: FnMut(Self::Item) -> Option<B>;
    fn try_find<F, R>(
        &mut self, 
        f: F
    ) -> Result<Option<Self::Item>, <R as Try>::Error>
    where
        F: FnMut(&Self::Item) -> R,
        R: Try<Ok = bool>;
    fn position<P>(&mut self, predicate: P) -> Option<usize>
    where
        P: FnMut(Self::Item) -> bool;
    fn rposition<P>(&mut self, predicate: P) -> Option<usize>
    where
        Self: ExactSizeIterator + DoubleEndedIterator,
        P: FnMut(Self::Item) -> bool;
    fn max(self) -> Option<Self::Item>
    where
        Self::Item: Ord;
    fn min(self) -> Option<Self::Item>
    where
        Self::Item: Ord;
    fn max_by_key<B, F>(self, f: F) -> Option<Self::Item>
    where
        F: FnMut(&Self::Item) -> B,
        B: Ord;
    fn max_by<F>(self, compare: F) -> Option<Self::Item>
    where
        F: FnMut(&Self::Item, &Self::Item) -> Ordering;
    fn min_by_key<B, F>(self, f: F) -> Option<Self::Item>
    where
        F: FnMut(&Self::Item) -> B,
        B: Ord;
    fn min_by<F>(self, compare: F) -> Option<Self::Item>
    where
        F: FnMut(&Self::Item, &Self::Item) -> Ordering;
    fn rev(self) -> Rev<Self>
    where
        Self: DoubleEndedIterator;
    fn unzip<A, B, FromA, FromB>(self) -> (FromA, FromB)
    where
        Self: Iterator<Item = (A, B)>,
        FromA: Default + Extend<A>,
        FromB: Default + Extend<B>;
    fn copied<'a, T>(self) -> Copied<Self>
    where
        Self: Iterator<Item = &'a T>,
        T: 'a + Copy;
    fn cloned<'a, T>(self) -> Cloned<Self>
    where
        Self: Iterator<Item = &'a T>,
        T: 'a + Clone;
    fn cycle(self) -> Cycle<Self>
    where
        Self: Clone;
    fn sum<S>(self) -> S
    where
        S: Sum<Self::Item>;
    fn product<P>(self) -> P
    where
        P: Product<Self::Item>;
    fn cmp<I>(self, other: I) -> Ordering
    where
        I: IntoIterator<Item = Self::Item>,
        Self::Item: Ord;
    fn cmp_by<I, F>(self, other: I, cmp: F) -> Ordering
    where
        F: FnMut(Self::Item, <I as IntoIterator>::Item) -> Ordering,
        I: IntoIterator;
    fn partial_cmp<I>(self, other: I) -> Option<Ordering>
    where
        I: IntoIterator,
        Self::Item: PartialOrd<<I as IntoIterator>::Item>;
    fn partial_cmp_by<I, F>(
        self, 
        other: I, 
        partial_cmp: F
    ) -> Option<Ordering>
    where
        F: FnMut(Self::Item, <I as IntoIterator>::Item) -> Option<Ordering>,
        I: IntoIterator;
    fn eq<I>(self, other: I) -> bool
    where
        I: IntoIterator,
        Self::Item: PartialEq<<I as IntoIterator>::Item>;
    fn eq_by<I, F>(self, other: I, eq: F) -> bool
    where
        F: FnMut(Self::Item, <I as IntoIterator>::Item) -> bool,
        I: IntoIterator;
    fn ne<I>(self, other: I) -> bool
    where
        I: IntoIterator,
        Self::Item: PartialEq<<I as IntoIterator>::Item>;
    fn lt<I>(self, other: I) -> bool
    where
        I: IntoIterator,
        Self::Item: PartialOrd<<I as IntoIterator>::Item>;
    fn le<I>(self, other: I) -> bool
    where
        I: IntoIterator,
        Self::Item: PartialOrd<<I as IntoIterator>::Item>;
    fn gt<I>(self, other: I) -> bool
    where
        I: IntoIterator,
        Self::Item: PartialOrd<<I as IntoIterator>::Item>;
    fn ge<I>(self, other: I) -> bool
    where
        I: IntoIterator,
        Self::Item: PartialOrd<<I as IntoIterator>::Item>;
    fn is_sorted(self) -> bool
    where
        Self::Item: PartialOrd<Self::Item>;
    fn is_sorted_by<F>(self, compare: F) -> bool
    where
        F: FnMut(&Self::Item, &Self::Item) -> Option<Ordering>;
    fn is_sorted_by_key<F, K>(self, f: F) -> bool
    where
        F: FnMut(Self::Item) -> K,
        K: PartialOrd<K>;
}
```

> `Iterator<Item = T>` types can be iterated and will produce `T` types. There's no `IteratorMut` trait. Each `Iterator` impl can specify whether it returns immutable references, mutable references, or owned values via the `Item` associated type.

实现 `Iterator<Item = T>` 的类型可以迭代产生 `T` 类型。注意：并不存在 `IteratorMut` 类型，因为可以通过在实现 `Iterator` 特性时指定 `Item` 关联类型，来选择其返回的是不可变引用、可变引用还是自有值。

| `Vec<T>` 方法 | 返回类型 |
|-----------------|-------------------|
| `.iter()` | `Iterator<Item = &T>` |
| `.iter_mut()` | `Iterator<Item = &mut T>` |
| `.into_iter()` | `Iterator<Item = T>` |

> Something that is not immediately obvious to beginner Rustaceans but that intermediate Rustaceans take for granted is that most types are not their own iterators. If a type is iterable we almost always impl some custom iterator type which iterates over it rather than trying to make it iterate over itself:

对于 Rust 的初学者而言可能有些费解，但是对于中级学习者而言则是顺理成章的一件事是 —— 绝大多数类型并不是自己的迭代器。这意味着，如果某种类型是可迭代的，那么应当实现某种额外的迭代器类型去迭代它，而不是让它自己迭代自己。

```rust
struct MyType {
    items: Vec<String>
}

impl MyType {
    fn iter(&self) -> impl Iterator<Item = &String> {
        MyTypeIterator {
            index: 0,
            items: &self.items
        }
    }
}

struct MyTypeIterator<'a> {
    index: usize,
    items: &'a Vec<String>
}

impl<'a> Iterator for MyTypeIterator<'a> {
    type Item = &'a String;
    fn next(&mut self) -> Option<Self::Item> {
        if self.index >= self.items.len() {
            None
        } else {
            let item = &self.items[self.index];
            self.index += 1;
            Some(item)
        }
    }
}
```

> For the sake of teaching the above example shows how to impl an `Iterator` from scratch but the idiomatic solution in this situation would be to just defer to `Vec`'s `iter` method:

出于教学的原因，我们在上例中从头手动实现了一个迭代器。而在这种情况下，最慣用的做法是直接调用 `Vec` 的 `iter` 方法。

```rust
struct MyType {
    items: Vec<String>
}

impl MyType {
    fn iter(&self) -> impl Iterator<Item = &String> {
        self.items.iter()
    }
}
```

> Also this is a good generic blanket impl to be aware of:

另外，最好了解这个一揽子泛型实现：

```rust
impl<I: Iterator + ?Sized> Iterator for &mut I;
```

> It says that any mutable reference to an iterator is also an iterator. This is useful to know because it allows us to use iterator methods with `self` receivers as if they had `&mut self` receivers.

任何迭代器的可变引用也是一个迭代器。了解这样的性质有助于我们理解，为什么可以将迭代器的某些参数为 `self` 的方法当作具有 `&mut self` 参数的方法来使用。

> As an example, imagine we have a function which processes an iterator of more than three items, but the first step of the function is to take out the first three items of the iterator and process them separately before iterating over the remaining items, here's how a beginner may attempt to write this function:

举个例子，想象我们有这样一个函数，它处理一个具有三个以上值的迭代器，这个函数首先要取得该迭代器的前三个值并分别地处理他们，然后再依次迭代剩余的值。初学者可能会这样实现该函数：

```rust
fn example<I: Iterator<Item = i32>>(mut iter: I) {
    let first3: Vec<i32> = iter.take(3).collect();
    for item in iter { // ❌ iter consumed in line above
                       // ❌ iter 在上一行就已经被消耗掉了
        // process remaining items
        // 处理剩余的值
    }
}
```

> Well that's annoying. The `take` method has a `self` receiver so it seems like we cannot call it without consuming the whole iterator! Here's what a naive refactor of the above code might look like:

糟糕，`take` 方法具有 `self` 参数，这意味着我们不能在不消耗掉整个迭代器的前提下调用该方法。以下可能是一个初学者的改进：

```rust
fn example<I: Iterator<Item = i32>>(mut iter: I) {
    let first3: Vec<i32> = vec![
        iter.next().unwrap(),
        iter.next().unwrap(),
        iter.next().unwrap(),
    ];
    for item in iter { // ✅
        // process remaining items
        // 处理剩余的值
    }
}
```

> Which is okay. However, the idiomatic refactor is actually:

这是可行的，但是慣用的改进方式莫过于：

```rust
fn example<I: Iterator<Item = i32>>(mut iter: I) {
    let first3: Vec<i32> = iter.by_ref().take(3).collect();
    for item in iter { // ✅
        // process remaining items
        // 处理剩余的值
    }
}
```

> Not very easy to discover. But anyway, now we know.

这真是一个很隐蔽的方法，但是被我们抓到了。

> Also, there are no rules or conventions on what can or cannot be an iterator. If the type impls `Iterator` then it's an iterator. Some creative examples from the standard library:

同样，对于什么可以是迭代器，什么不可以是，并无一定之规。实现了 `Iterator` 特性的就是迭代器。而在标准库中，确有一些具有创造性的用例：

```rust
use std::sync::mpsc::channel;
use std::thread;

fn paths_can_be_iterated(path: &Path) {
    for part in path {
        // iterate over parts of a path
        // 迭代 path 的不同部分
    }
}

fn receivers_can_be_iterated() {
    let (send, recv) = channel();

    thread::spawn(move || {
        send.send(1).unwrap();
        send.send(2).unwrap();
        send.send(3).unwrap();
    });

    for received in recv {
        // iterate over received values
        // 迭代接收到的值
    }
}
```



### IntoIterator

预备知识
- [Self](#self)
- [Methods](#methods)
- [Associated Types](#associated-types)
- [Iterator](#iterator)

```rust
trait IntoIterator 
where
    <Self::IntoIter as Iterator>::Item == Self::Item, 
{
    type Item;
    type IntoIter: Iterator;
    fn into_iter(self) -> Self::IntoIter;
}
```

> `IntoIterator` types can be converted into iterators, hence the name. The `into_iter` method is called on a type when it's used within a `for-in` loop:

闻弦歌而知雅意，实现 `IntoIterator` 特性的类型可以被转换为迭代器。当用于 `for-in` 循环时，将自动调用该类型的 `into_iter` 方法.

```rust
// vec = Vec<T>
for v in vec {} // v = T

// above line desugared
// 以上代码等价于
for v in vec.into_iter() {}
```

> Not only does `Vec` impl `IntoIterator` but so does `&Vec` and `&mut Vec` if we'd like to iterate over immutable or mutable references instead of owned values, respectively.

不仅 `Vec` 实现了 `IntoIterator` 特性，`&Vec` 与 `&mut Vec` 同样如此。因此我们可以相应的对可变与不可变的引用，以及自有值进行迭代。

```rust
// vec = Vec<T>
for v in &vec {} // v = &T

// above example desugared
// 以上代码等价于
for v in (&vec).into_iter() {}

// vec = Vec<T>
for v in &mut vec {} // v = &mut T

// above example desugared
// 以上代码等价于
for v in (&mut vec).into_iter() {}
```



### FromIterator

预备知识
- [Self](#self)
- [Functions](#functions)
- [Generic Parameters](#generic-parameters)
- [Iterator](#iterator)
- [IntoIterator](#intoiterator)

```rust
trait FromIterator<A> {
    fn from_iter<T>(iter: T) -> Self
    where
        T: IntoIterator<Item = A>;
}
```

> `FromIterator` types can be created from an iterator, hence the name. `FromIterator` is most commonly and idiomatically used by calling the `collect` method on `Iterator`:

顾叶落而晓秋至，实现 `FromIterator` 特性的类型可以由迭代器而构造。`FromIterator` 特性最常见和最慣用的使用方法是调用 `Iterator` 的 `collect` 方法：

```rust
fn collect<B>(self) -> B
where
    B: FromIterator<Self::Item>;
```

> Example of collecting an `Iterator<Item = char>` into a `String`:

下例展示了如何将 `Iterator<Item = char>` 迭代器的值收集为 `String` ：

```rust
fn filter_letters(string: &str) -> String {
    string.chars().filter(|c| c.is_alphabetic()).collect()
}
```

> All the collections in the standard library impl `IntoIterator` and `FromIterator` so that makes it easier to convert between them:

标准库中的全部集合类型都实现了 `IntoIterator` 和 `FromIterator` 特性，所以在它们之间进行转换是很方便的：

```rust
use std::collections::{BTreeSet, HashMap, HashSet, LinkedList};

// String -> HashSet<char>
fn unique_chars(string: &str) -> HashSet<char> {
    string.chars().collect()
}

// Vec<T> -> BTreeSet<T>
fn ordered_unique_items<T: Ord>(vec: Vec<T>) -> BTreeSet<T> {
    vec.into_iter().collect()
}

// HashMap<K, V> -> LinkedList<(K, V)>
fn entry_list<K, V>(map: HashMap<K, V>) -> LinkedList<(K, V)> {
    map.into_iter().collect()
}

// and countless more possible examples
// 还有数不胜数的例子
```



## 输入输出特性 I/O Traits



### Read & Write

预备知识
- [Self](#self)
- [Methods](#methods)
- [Scope](#scope)
- [Generic Blanket Impls](#generic-blanket-impls)

```rust
trait Read {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>;

    // provided default impls
    // 提供默认实现
    fn read_vectored(&mut self, bufs: &mut [IoSliceMut<'_>]) -> Result<usize>;
    fn is_read_vectored(&self) -> bool;
    unsafe fn initializer(&self) -> Initializer;
    fn read_to_end(&mut self, buf: &mut Vec<u8>) -> Result<usize>;
    fn read_to_string(&mut self, buf: &mut String) -> Result<usize>;
    fn read_exact(&mut self, buf: &mut [u8]) -> Result<()>;
    fn by_ref(&mut self) -> &mut Self
    where
        Self: Sized;
    fn bytes(self) -> Bytes<Self>
    where
        Self: Sized;
    fn chain<R: Read>(self, next: R) -> Chain<Self, R>
    where
        Self: Sized;
    fn take(self, limit: u64) -> Take<Self>
    where
        Self: Sized;
}

trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;

    // provided default impls
    // 提供默认实现
    fn write_vectored(&mut self, bufs: &[IoSlice<'_>]) -> Result<usize>;
    fn is_write_vectored(&self) -> bool;
    fn write_all(&mut self, buf: &[u8]) -> Result<()>;
    fn write_all_vectored(&mut self, bufs: &mut [IoSlice<'_>]) -> Result<()>;
    fn write_fmt(&mut self, fmt: Arguments<'_>) -> Result<()>;
    fn by_ref(&mut self) -> &mut Self
    where
        Self: Sized;
}
```

> Generic blanket impls worth knowing:

值得关注的一揽子泛型实现：

```rust
impl<R: Read + ?Sized> Read for &mut R;
impl<W: Write + ?Sized> Write for &mut W;
```

> These say that any mutable reference to a `Read` type is also `Read`, and same with `Write`. This is useful to know because it allows us to use any method with a `self` receiver as if it had a `&mut self` receiver. We already went over how to do this and why it's useful in the `Iterator` trait section so I'm not going to repeat it again here.

对于任何实现了 `Read` 特性的类型，其可变的引用类型也实现了 `Read` 特性。`Write` 也是如此。知晓这一点有助于我们理解为什么，对于具有 `self` 参数的函数可以如同那些具有 `&mut self` 参数的函数一般使用。鉴于我们已经在 `Iterator` 特性一节中做出了相近的说明，对此我不再赘述。

> I'd like to point out that `&[u8]` impls `Read` and that `Vec<u8>` impls `Write` so we can easily unit test our file handling functions using `String`s which are trivial to convert to `&[u8]` and from `Vec<u8>`:

我特别指出的是，在 `&[u8]` 实现 `Read` 的同时，`Vec<u8>` 实现了 `Write`，因此我们可以很方便地使用 `String` 来对我们的文件处理函数进行单元测试，因为它可以轻易地转换到 `&[u8]` 和转换自 `Vec<u8>` 。

```rust
use std::path::Path;
use std::fs::File;
use std::io::Read;
use std::io::Write;
use std::io;

// function we want to test
// 欲要测试此函数
fn uppercase<R: Read, W: Write>(mut read: R, mut write: W) -> Result<(), io::Error> {
    let mut buffer = String::new();
    read.read_to_string(&mut buffer)?;
    let uppercase = buffer.to_uppercase();
    write.write_all(uppercase.as_bytes())?;
    write.flush()?;
    Ok(())
}

// in actual program we'd pass Files
// 实际使用中我们传入文件
fn example(in_path: &Path, out_path: &Path) -> Result<(), io::Error> {
    let in_file = File::open(in_path)?;
    let out_file = File::open(out_path)?;
    uppercase(in_file, out_file)
}


// however in unit tests we can use Strings!
// 但是在单元测试中我们使用 String !
#[test] // ✅
fn example_test() {
    let in_file: String = "i am screaming".into();
    let mut out_file: Vec<u8> = Vec::new();
    uppercase(in_file.as_bytes(), &mut out_file).unwrap();
    let out_result = String::from_utf8(out_file).unwrap();
    assert_eq!(out_result, "I AM SCREAMING");
}
```



## 结语 Conclusion

> We learned a lot together! Too much in fact. This is us now:

我们真是学习了太多！太多了！可能这就是我们现在的样子：

![rust standard library traits](../../../assets/jason-jarvis-stdlib-traits.png)

_该漫画的创作者: [The Jenkins Comic](https://thejenkinscomic.wordpress.com/2020/05/06/memory/)_



## 讨论 Discuss

> Discuss this article on

可以在如下地点讨论本文
- [Github](https://github.com/pretzelhammer/rust-blog/discussions)
- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/ml9shl/tour_of_rusts_standard_library_traits/)
- [official Rust users forum](https://users.rust-lang.org/t/blog-post-tour-of-rusts-standard-library-traits/57974)
- [Twitter](https://twitter.com/pretzelhammer/status/1379561720176336902)
- [lobste.rs](https://lobste.rs/s/g27ezp/tour_rust_s_standard_library_traits)
- [rust subreddit](https://www.reddit.com/r/rust/comments/mmrao0/tour_of_rusts_standard_library_traits/)


## 更多资料 Further Reading

- [Sizedness in Rust](../../sizedness-in-rust.md)
- [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md)
- [Learning Rust in 2020](../../learning-rust-in-2020.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](../../too-many-brainfuck-compilers.md)



## 翻译 Translation

鉴于水平所限，

难免出现翻译错误，

如发现错误还请告知！



skanfd 译

2021年4月21日

