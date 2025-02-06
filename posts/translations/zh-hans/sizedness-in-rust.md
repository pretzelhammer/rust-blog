# Rust中的Sizedness

_22 July 2020 · #rust · #sizedness_

**目录**

- [介绍](#介绍)
- [Sizedness(大小确定性)](#sizedness大小确定性)
- [`Sized` Trait](#sized-trait)
- [泛型中的`Sized`](#泛型中的sized)
- [不定大小类型(Unsized Types)](#不定大小类型unsized-types)
  - [Slices(切片)](#slices切片)
  - [Trait Objects(特性对象)](#trait-objects特性对象)
  - [trait object的限制](#trait-object的限制)
    - [不能将不定大小类型强转为trait object](#不能将不定大小类型强转为trait-object)
    - [不能创建多特性对象](#不能创建多特性对象)
  - [用户定义的不定大小类型](#用户定义的不定大小类型)
- [0大小类型](#0大小类型)
  - [单元类型](#单元类型)
  - [用户定义的单元大小结构](#用户定义的单元大小结构)
  - [不可能类型（Never Type）](#不可能类型never-type)
  - [用户定义的伪不可能类型](#用户定义的伪不可能类型)
  - [PhantomData(伪数据)](#phantomdata伪数据)
- [总结](#总结)
- [讨论](#讨论)
- [进一步阅读](#进一步阅读)
- [通知](#通知)

## 介绍

大小确定性(Sizedness)在Rust的众多重要概念中是相对不引人注目的一个。它以隐晦的方式与众多其它语言特性互相作用，并且往往只以错误提示的形式让我们了解到它的存在，这个错误我们Rustacean大概常遇到，就是"_x doesn't have size known at compile time_"(_x没有确定的编译时大小_)。本文将介绍与Sizeness相关的各种语言特性，包括: 确定大小类型(sized types)，不定大小类型(unsized types)，以及0大小类型(zero-sized types)，我们将探讨它们的使用场景，应用，痛点，以及在必要的时候如何绕过Sizeness带来的困扰。

名词解释:

| 名词 | 解释 |
|-|-|
| sizedness(大小确定性) | 表明特定类型有或没有确定的大小 |
| sized type(确定大小类型) | 编译时有确定大小的类型 |
| 1) unsized type(不定大小类型) _或_<br>2) DST | 动态大小类型，也就是说，该类型的大小不能在编译时确定 |
| ?sized type(未知大小类型) | 可能是sized type，也可能是unsized type |
| unsized coercion(不定大小强转) | 将一个确定大小类型强转为一个不定大小类型 |
| ZST | zero-sized type(0大小类型)，即，该类型的大小为0字节 |
| width(宽度) | 一个度量单位，用于描述指针宽度 |
| 1) 瘦指针 _或_<br>2) 单width指针 | 大小为 _1 width_ 的指针 |
| 1) 胖指针 _or_<br>2) 双width指针 | 大小为 _2 widths_ 的指针|
| 1) 指针 _or_<br>2) 引用 | 指针，其具体 _width_ 会在相应的上下文中说明 |
| slice(切片) | 指向某个数组动态大小视图的双width指针 |
| (译者补充: trait object(特性对象)) | 实现某个trait的对象([dyn SomeTrait](https://github.com/mercury-2025/rust-blog/blob/master/posts/tour-of-rusts-standard-library-traits.md#trait-objects)) |


## Sizedness(大小确定性)

在Rust中，所谓的Sizedness是指一个类型的具体大小是否可以在编译时确定。确定类型大小之所以重要，是因为这样一来，才有可能为类型的实例在栈上分配相应的空间。确定大小类型(Sized types)可以使用传值或传引用的方式到处传递。同样的，如果一个类型的大小不能在编译时确定，我们就称它为不定大小类型(unsized type)或者DST，又或者叫它动态大小类型。因为不定大小类型不能放到栈上，所以它们只能以引用的形式传递。 下面是一部分 _确定大小类型_ 和 _不定大小类型_ 的例子：

```rust
use std::mem::size_of;

fn main() {
    // 原始类型
    assert_eq!(4, size_of::<i32>());
    assert_eq!(8, size_of::<f64>());

    // 元组
    assert_eq!(8, size_of::<(i32，i32)>());

    // 数组
    assert_eq!(0, size_of::<[i32; 0]>());
    assert_eq!(12, size_of::<[i32; 3]>());

    struct Point {
        x: i32,
        y: i32,
    }

    // 结构体
    assert_eq!(8, size_of::<Point>());

    // 枚举
    assert_eq!(8, size_of::<Option<i32>>());

    // 取指针宽度:
    // 在32位目标上为4字节，或者
    // 在64位目标上为8字节
    const WIDTH: usize = size_of::<&()>();

    // 指向确定大小类型的指针，其width为1
    assert_eq!(WIDTH, size_of::<&i32>());
    assert_eq!(WIDTH, size_of::<&mut i32>());
    assert_eq!(WIDTH, size_of::<Box<i32>>());
    assert_eq!(WIDTH, size_of::<fn(i32) -> i32>());

    const DOUBLE_WIDTH: usize = 2 * WIDTH;

    // 不定大小结构体
    struct Unsized {
        unsized_field: [i32],
    }

    // 指向不定大小类型的指针，其width为2
    assert_eq!(DOUBLE_WIDTH, size_of::<&str>()); // 切片
    assert_eq!(DOUBLE_WIDTH, size_of::<&[i32]>()); // 切片
    assert_eq!(DOUBLE_WIDTH, size_of::<&dyn ToString>()); // trait object
    assert_eq!(DOUBLE_WIDTH, size_of::<Box<dyn ToString>>()); // trait object
    assert_eq!(DOUBLE_WIDTH, size_of::<&Unsized>()); // 用户定义的不定大小类型

    // 不定大小类型
    size_of::<str>(); // 编译报错
    size_of::<[i32]>(); // 编译报错
    size_of::<dyn ToString>(); // 编译报错
    size_of::<Unsized>(); // 编译报错
}
```

我们怎么知道某个类型是否为 _确定大小类型_ 呢? 很简单: 所有原始类型和指针都有确定的大小，同时所有的结构、元组、枚举和数组，它们或者直接由原始类型和指针构成，或者由它们嵌套得到，所以，只要把这些组成元素的大小加起来，那也能确定地计算出它们的大小（当然计算过程要考虑padding和对齐）。同样道理，我们也可以知道一个类型为 _不定大小类型_：切片可以有任意数量的成员，所以在运行时可能会有任意的大小，而trait object可能是任意实现了该特性的结构/枚举，因此其运行时大小也不确定。

**要点**
- 指向数组动态视图的指针在Rust中被称为slice(切片)，例如`&str`是 _"string slice"_，而`&[i32]`是 _"i32 slice"_
- 切片为双width，原因在于其存储了一个指向数组的指针，同时还要保存数组中元素的数量
- trait object为双width，原因在于其存储了指向原数据的指针，以及一个指向虚表的指针
- 不定大小结构的指针为双width，原因在于其存储了指向原数据的指针，同时也保存了该结构的大小
- 不定大小结构只能有一个不定大小的字段，并且该字段必须是该结构的最后一个字段
  

来看下面这个带注释的例子，该例子比较切片和数组，通过这个例子我们可以看到，为什么不定大小结构的指针为双width:

```rust
use std::mem::size_of;

const WIDTH: usize = size_of::<&()>();
const DOUBLE_WIDTH: usize = 2 * WIDTH;

fn main() {
    // 长度保存在类型中
    // [i32; 3] 是3个i32的数组
    let nums: &[i32; 3] = &[1, 2, 3];

    // 单width指针
    assert_eq!(WIDTH, size_of::<&[i32; 3]>());

    let mut sum = 0;

    // 可以安全地迭代nums
    // Rust知道它有3个元素
    for num in nums {
        sum += num;
    }

    assert_eq!(6, sum);

    // 将[i32; 3]不定大小强转(unsized coercion)为切片[i32]
    // 数据长度现在保存在指针内
    let nums: &[i32] = &[1, 2, 3];

    // 指针为双width，因为需要还要保存数据长度
    assert_eq!(DOUBLE_WIDTH, size_of::<&[i32]>());

    let mut sum = 0;

    // 也可以安全地迭代nums
    // 因为Rust知道它有3个元素(译者注: 通过指针中的第二个width保存)
    for num in nums {
        sum += num;
    }

    assert_eq!(6, sum);
}
```

下面是另一个例子，该例子比较结构与trait objects:

```rust
use std::mem::size_of;

const WIDTH: usize = size_of::<&()>();
const DOUBLE_WIDTH: usize = 2 * WIDTH;

trait Trait {
    fn print(&self);
}

struct Struct;
struct Struct2;

impl Trait for Struct {
    fn print(&self) {
        println!("struct");
    }
}

impl Trait for Struct2 {
    fn print(&self) {
        println!("struct2");
    }
}

fn print_struct(s: &Struct) {
    // 永远打印出"struct"
    // 这个信息在编译时已知
    s.print();
    // 单width pointer
    assert_eq!(WIDTH, size_of::<&Struct>());
}

fn print_struct2(s2: &Struct2) {
    // 永远打印出"struct2"
    // 这个信息在编译时已知
    s2.print();
    // 单width pointer
    assert_eq!(WIDTH, size_of::<&Struct2>());
}

fn print_trait(t: &dyn Trait) {
    // 会打印出"struct"还是"struct2"?
    // 这点在编译时不能确定
    t.print();
    // Rust必须在运行时检查指针内容
    // 以确定是使用Struct的
    // 还是Struct2的"print"实现
    // 所以这里pointer必须为双width(译者注: 第二个width用于保存虚表)
    assert_eq!(DOUBLE_WIDTH, size_of::<&dyn Trait>());
}

fn main() {
    // 指向数据的单width指针
    let s = &Struct; 
    print_struct(s); // 打印出"struct"
    
    // 指向数据的单width指针
    let s2 = &Struct2;
    print_struct2(s2); // 打印出"struct2"
    
    // 将Struct强转为不定长度的dyn Trait
    // 需要双width指针，以便同时指向数据和Struct的vtable
    let t: &dyn Trait = &Struct;
    print_trait(t); // 会打印出"struct"
    
    // 将Struct2强转为不定长度的dyn Trait
    // 需要双width指针，以便同时指向数据和Struct2的vtable
    let t: &dyn Trait = &Struct2;
    print_trait(t); // 会打印出"struct2"
}
```

**要点**
- 只有确定大小类型才能放在栈上，也只有它们才能在程序中按值传递
- 不定大小类型不能放在栈上，只能按引用传递
- 指向不定大小类型数据的指针是双width的，因为除了指向数据本身之外，还需要有额外的记录以追踪数据的长度或者结构的虚表


## `Sized` Trait

Rust中的`Sized`是auto trait(自动特性)，同时也是marker trait(标记特性)。

所谓的auto trait是指在满足一定条件的情况下，该特性会自动实现，而无需手动impl。而marker trait是指那些用于标记类型具备特定属性的特性。marker trait不需要有方法/关联函数/关联常量/关联类型之类的特性item。所有的auto traits都是marker traits，但不是所有的marker traits都是auto traits。之所以auto traits必须是marker traits，是因为只有这样，编译器才能为它们提供默认的实现，否则，如果特性有任何item，那编译器就无法知道怎么自动实现它们了。

对于特定类型来说，如果其所有成员都是`Sized`，那该类型也自动为`Sized`。这里的'成员'具体含义取决于该type的类别，比如：结构的成员字段，枚举的变量，数组的元素，元组的成员，等等。如果一个类型为`Sized`，那就意味着其大小在编译时是确定的。

还有一些其它的auto marker traits，比如`Send` 和 `Sync`。如果一个类型(的实例)可以安全地在线程间转移的话，则该类型为`Send`的，同样，如果一个类型可以安全地在多个线程间共享引用，则该类型为`Sync`的。如果一个类型的所有成员都是`Send` && `Sync`，则该型自身也为`Send` && `Sync`。对于特性`Sized`，有一点特殊的是，开发者不能自行改变或取消(opt-out)掉该特性，这和其它auto marker trait不同。

```rust
#![feature(negative_impls)]

// 该类型为Sized，Send，Sync
struct Struct;

// 取消(opt-out) Send特性
impl !Send for Struct {} // ✅

// 取消(opt-out) Sync特性
impl !Sync for Struct {} // ✅

// 不能取消(opt-out) Sized特性
impl !Sized for Struct {} // ❌
```

关于不能取消`Sized`这一点，也容易理解。因为可能有时我们不想某个类型在线程间传递，或者在不同线程中共享该类型，但很难想像在哪个场景下，我们会需要编译器"忘记"某个类型的确定大小性质，并把它当成不定大小类型来使用，这样做不会带来任何好处，只会使这个类型变得更难使用。

另外，如果非要严格地说的话，其实`Sized`不算auto trait，它没使用`auto`关键字，只不过编译器对它作了特殊处理，使得其表现上与其它auto traits一样，总之，在开发中把它当成auto trait没有任何问题。

**要点**
- `Sized`是auto marker trait —— 自动标记特性


## 泛型中的`Sized`

初看起来不明显，但实际上每次我们写泛型代码时，每个泛型参数都默认自动绑定了`Sized`特性(译者注: 除非我们手动指定了`?Sized`):

```rust
// 对于这个泛型函数...
fn func<T>(t: T) {}

// 它被展开后是这样的...
fn func<T: Sized>(t: T) {}

// 对于上面这个自动绑定的Sized，我们可以显式指定?Sized来取消掉它
fn func<T: ?Sized>(t: T) {} // ❌

// 一旦像上面这样做了，则上面的代码就编不过了，因为它可能没有确定的大小(译者注: 而不定大小类型不能以传值方式使用)
// 所以我们必须把它放到某个指针后面去，比如:
fn func<T: ?Sized>(t: &T) {} // ✅
fn func<T: ?Sized>(t: Box<T>) {} // ✅
```

**要点**
- `?Sized` 可以念成 _"可选 sized"_ 或者 _"可能 sized"_，把它加到到类型的限定中去后，表示这个类型可能是'确定大小类型'，也可能是'不定大小型'
- `?Sized` 通常也被称为 _"解除bound"_ 或者 _"宽松bound"_，因为它放宽了原有限制，而不是在为原类型添加更多的限制
- `?Sized` 是Rust中唯一的 _"解除bound"_

所以，上面说的这些有什么用呢？嗯，实际上只要我们写泛型类型，并且通过指针来使用这个类型，那基本上都会需要取消掉这个默认的`Sized`绑定，这样我们的函数才能更自由地接受各种参数类型，而如果我们不这样做，那最后，我们大概率会遇到麻烦，并被编译器的错误提示弄糊涂。

来看看我在Rust中写的第一个泛型函数。我在`dbg!`宏正式进入Rust前就开始学习Rust了，所以那时打印调试值的唯一办法是`println!("{:?}"，some_value);`，因为到处写这段代码有点麻烦，所以我决定写一个下面这样的`debug`辅助函数:

```rust
use std::fmt::Debug;

fn debug<T: Debug>(t: T) { // T: Debug + Sized
    println!("{:?}", t);
}

fn main() {
    debug("my str"); // T = &str, &str: Debug + Sized ✅
}
```

目前为止一切都很完美，但这个函数会拿走传入参数的所有权，这点有些烦人，所以我修改了下，将函数签名改为使用引用:

```rust
use std::fmt::Debug;

fn dbg<T: Debug>(t: &T) { // T: Debug + Sized
    println!("{:?}", t);
}

fn main() {
    dbg("my str"); // &T = &str, T = str, str: Debug + !Sized ❌
}
```

但是这段代码编不过:

```none
error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> src/main.rs:8:9
  |
3 | fn dbg<T: Debug>(t: &T) {
  |        - required by this bound in `dbg`
...
8 |     dbg("my str");
  |         ^^^^^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `str`
  = note: to learn more，visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
help: consider relaxing the implicit `Sized` restriction
  |
3 | fn dbg<T: Debug + ?Sized>(t: &T) {
  |   
```

第一次遇到这个错时，我真的被搞懵了。我只是给参数加了个比之前更严格的限制，过去能编通的代码，现在竟然编不过了？到底发生了什么？

其实在上面代码的注释里，我已经作了一定解释了，本质原因在于：Rust在编译时将`T`替换为具体类型时作了模式匹配。通过下面这个表格可能看得更清楚些：

| 类型 | `T` | `&T` |
|------------|---|----|
| `&str` | `T` = `&str` | `T` = `str` |

| 类型 | `Sized`（是否确定大小） |
|-|-|
| `str` | ❌ |
| `&str` | ✅ |
| `&&str` | ✅ |

所以，在把参数改成引用后，我需要同时加上`?Sized`(译者注:放宽对T的限制)，这样下面这段代码就可以正常工作了:

```rust
use std::fmt::Debug;

fn debug<T: Debug + ?Sized>(t: &T) { // T: Debug + ?Sized
    println!("{:?}"，t);
}

fn main() {
    debug("my str"); // &T = &str，T = str，str: Debug + !Sized ✅
}
```

**要点**
- 所有的泛型类型都有一个默认的`Sized`绑定
- 如果我们的泛型函数有指针形式的参数`T`，比如`&T`，`Box<T>`，`Rc<T>`， 等等，那我们几乎总是要通过`T: ?Sized`来取消默认的`Sized`


## 不定大小类型(Unsized Types)



### Slices(切片)

最常见的切片是字符串切片 `&str` 和数组切片 `&[T]`。切片有一个优点：很多其它类型可以强转为切片，利用这一点，以及Rust的自动类型强转，我们可以构造更加灵活的API。

类型强转可以在好些个不同的场景下发生，最常见的是在函数传参以及方法调用。这里，我们感兴趣的是deref（解引用）强转和不定大小强转（unsized coercions）。解引用强转是指`T`通过deref操作强转为`U`，也就是`T: Deref<Target = U>`，比如`String.deref() -> str`。 而不定大小强转则是指将`T`强转为`U`，这里的`T`是一个确定大小类型，而`U`为不定大小类型，也就是`T: Unsize<U>`，比如`[i32; 3] -> [i32]`。

```rust
trait Trait {
    fn method(&self) {}
}

impl Trait for str {
    // 现在能够调用下面这些类型的"method"
    // 1) str
    // 2) String (因为String实现了Deref<Target = str>)
}
impl<T> Trait for [T] {
    // 现在能够调用下面这些类型的"method"
    // 1) 任意 &[T]
    // 2) 任意 U where U: Deref<Target = [T]>，比如Vec<T>
    // 3) [T; N]，因为[T; N]: Unsize<[T]>
}

fn str_fun(s: &str) {}
fn slice_fun<T>(s: &[T]) {}

fn main() {
    let str_slice: &str = "str slice";
    let string: String = "string".to_owned();

    // 函数参数
    str_fun(str_slice);
    str_fun(&string); // 解引用强转

    // 方法调用
    str_slice.method();
    string.method(); // 解引用强转

    let slice: &[i32] = &[1];
    let three_array: [i32; 3] = [1, 2, 3];
    let five_array: [i32; 5] = [1, 2, 3, 4, 5];
    let vec: Vec<i32> = vec![1];

    // 函数参数
    slice_fun(slice);
    slice_fun(&vec); // 解引用强转
    slice_fun(&three_array); // 不定大小强转
    slice_fun(&five_array); // 不定大小强转

    // 方法调用
    slice.method();
    vec.method(); // 解引用强转
    three_array.method(); // 不定大小强转
    five_array.method(); // 不定大小强转
}
```

**关键点**
- 利用切片和Rust的自动类型强转，我们可以写出更灵活的API



### Trait Objects(特性对象)

特性默认是`?Sized`的。下面这段代码：

```rust
trait Trait: ?Sized {}
```

编译时会报错：

```none
error: `?Trait` is not permitted in supertraits
 --> src/main.rs:1:14
  |
1 | trait Trait: ?Sized {}
  |              ^^^^^^
  |
  = note: traits are `?Sized` by default
```

后面我们会讨论为什么traits默认是`?Sized`的，现在，我们先问下自己：trait为`?Sized`究竟是什么意思？我们把上面的代码展开来看：

```rust
trait Trait where Self: ?Sized {}
```

好的，所以默认的trait是指允许self为确定大小类型或者不定大小类型。而我们现在已经知道，不可以使用传值的方式传递不定大小类型，所以这个默认trait在效果上限制了我们可以在trait中定义的方法。具体说就是，我们不能写接受或返回传值形式的`self`。但是，奇怪的是，下面这个代码是可以通过编译的：

```rust
trait Trait {
    fn method(self); // ✅
}
```

不过，一旦我们真正开始实现这一方法，或者为这个方法提供默认实现，或者为不定大小类型实现这个trait的话，我们就会得到编译错误:

```rust
trait Trait {
    fn method(self) {} // ❌
}

impl Trait for str {
    fn method(self) {} // ❌
}
```

错误类似这样:

```none
error[E0277]: the size for values of type `Self` cannot be known at compilation time
 --> src/lib.rs:2:15
  |
2 |     fn method(self) {}
  |               ^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `Self`
  = note: to learn more，visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
  = note: all local variables must have a statically known size
  = help: unsized locals are gated as an unstable feature
help: consider further restricting `Self`
  |
2 |     fn method(self) where Self: std::marker::Sized {}
  |                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> src/lib.rs:6:15
  |
6 |     fn method(self) {}
  |               ^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `str`
  = note: to learn more，visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
  = note: all local variables must have a statically known size
  = help: unsized locals are gated as an unstable feature
```

如果我们可以确保只以传值的方式使用`self`，那可以通过给这个trait显式绑定`Sized`来解决上面的编译错误：

```rust
trait Trait: Sized {
    fn method(self) {} // ✅
}

impl Trait for str { // ❌
    fn method(self) {}
}
```

上面代码中，为str实现Trait时的编译报错为:

```none
error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> src/lib.rs:7:6
  |
1 | trait Trait: Sized {
  |              ----- required by this bound in `Trait`
...
7 | impl Trait for str {
  |      ^^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `str`
  = note: to learn more，visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
```

这是符合预期的，因为，既然我们为这个trait指定了`Sized`绑定，那自然就不能为不定大小的类型(比如`str`)实现该trait。另一方面，如果确实要为`str`实现该trait，那我们可以使用另一种方式，也就是保持trait的`?Sized`绑定，同时改用引用方式来传递`self`:

```rust
trait Trait {
    fn method(&self) {} // ✅
}

impl Trait for str {
    fn method(&self) {} // ✅
}
```

通过标记特性的方法而不是标记整个特性为`?Sized`或`Sized`，我们可以获得更细的控制粒度，比如：

```rust
trait Trait {
    fn method(self) where Self: Sized {}
}

impl Trait for str {} // ✅!?

fn main() {
    "str".method(); // ❌
}
```

有些让人吃惊的是，上面`impl Trait for str {}`一行可以通过编译，但最终，Rust会在我们试图在不定大小类型上调用`method`时捕捉到这一错误，所以一切都还好。虽然有一点奇怪，但它的确为我们提供了一些自由度：我们的trait可以有一些方法被绑定到`Sized`，同时我们还可以为不定大小类型实现该trait，只要我们永远不调用这些(译者注：绑定到`Sized`的)方法就不会有问题:

```rust
trait Trait {
    fn method(self) where Self: Sized {}
    fn method2(&self) {}
}

impl Trait for str {} // ✅

fn main() {
    // 我们永远不调用method，所以也就不会有报错
    "str".method2(); // ✅
}
```

现在回到前面提出的那个问题，为什么trait默认是`?Sized`（未知大小类型）? 答案在于trait objects。Trait objects是不定大小的，因为任意大小的类型都可以实现同一个trait，所以只有当`Trait: ?Sized`时，我们才有可能为`dyn Trait`实现`Trait`。看看实际代码的情况：

```rust
trait Trait: ?Sized {}

// 上面这一行对下面的`impl`是必须的

impl Trait for dyn Trait {
    // compiler magic here
}

// 因为`dyn Trait`是不定大小的

// 这样，我们在程序里才可以使用`dyn Trait`

fn function(t: &dyn Trait) {} // ✅
```

如果我们编译上面这段代码，会得到如下编译错误:

```none
error[E0371]: the object type `(dyn Trait + 'static)` automatically implements the trait `Trait`
 --> src/lib.rs:5:1
  |
5 | impl Trait for dyn Trait {
  | ^^^^^^^^^^^^^^^^^^^^^^^^ `(dyn Trait + 'static)` automatically implements trait `Trait`
```

这是编译器在告诉我们，它已经自动地为`dyn Trait`实现了`Trait`，所以我们不需要再手动显式这样做。再一次地，因为`dyn Trait`是不定大小的，所以编译器只能提供针对`Trait: ?Sized`的实现。反之，如果我们给`Trait`绑定`Sized`，那么`Trait`就会成为 _"object unsafe"_， 这是一个专有名词，它意味着我们不能将实现该trait的类型强转为`dyn Trait`的trait object。所以，下面的这段代码是编不过的：

```rust
trait Trait: Sized {}

fn function(t: &dyn Trait) {} // ❌
```

抛出的错误如下:

```none
error[E0038]: the trait `Trait` cannot be made into an object
 --> src/lib.rs:3:18
  |
1 | trait Trait: Sized {}
  |       -----  ----- ...because it requires `Self: Sized`
  |       |
  |       this trait cannot be made into an object...
2 | 
3 | fn function(t: &dyn Trait) {}
  |                ^^^^^^^^^^ the trait `Trait` cannot be made into an object
```

我们来写一个带`Sized`绑定方法的`?Sized`特性，然后试下看能否将其强转为trait object:

```rust
trait Trait {
    fn method(self) where Self: Sized {}
    fn method2(&self) {}
}

fn function(arg: &dyn Trait) { // ✅
    arg.method(); // ❌
    arg.method2(); // ✅
}
```

就像我们前面已经看到的那样，不调用这个`Sized`方法那一切都没有问题，反之则会报错。

**要点**
- 所有特性默认都是`?Sized`的
- `Trait: ?Sized`对于为`dyn Trait`实现`Trait`很关键
- 我们可以在方法粒度上指定`Self: Sized`
- `Sized`绑定的特性不能转成trait object


### trait object的限制

特性对象安全使得我们可以使用dyn Trait，不过，仍然有一些边缘的用例，限制了什么类型可以强转为trait object以及哪些特性可以用trait object来表示。


#### 不能将不定大小类型强转为trait object

```rust
fn generic<T: ToString>(t: T) {}
fn trait_object(t: &dyn ToString) {}

fn main() {
    generic(String::from("String")); // ✅
    generic("str"); // ✅
    trait_object(&String::from("String")); // ✅ - 不定大小强转
    trait_object("str"); // ❌ - 不能作不定大小强转
}
```

上面这段代码抛出的错误如下:

```none
error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> src/main.rs:8:18
  |
8 |     trait_object("str");
  |                  ^^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `str`
  = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
  = note: required for the cast to the object type `dyn std::string::ToString`
```

之所以可以将`String`强转为`&dyn ToString`，是因为`String`的确实现了`ToString`特性，并且，我们可以将一个确定大小类型(`String`)转为一个不定大小类型(`dyn ToString`)，Rust中支持这种强转，前文提到过，这种强转叫做不定大小强转。另一方面，`str`也实现了`ToString`特性，如果要将`str`转为`dyn ToString`，那也需要执行一次不定大小强转(译者注:因为dyn ToString是不定大小的)，但是这里的问题在于：`str`已经是不定大小的了，我们怎么样才能将一个不定大小的类型转为另一个不定大小类型呢？(译者注：答案是不支持，接下来是原因解释)

前文提到过，`&str`指针是双width的，它存储了指向数据的指针以及数据的长度。`&dyn ToString`指针也是双width的，保存了指向数据的指针以及指向虚表的指针。因此如果要将`&str`强转为`&dyn toString`，那就需要一个3倍width的指针，以同时保存指向数据的指针、数据长度以及指向虚表的指针。Rust不支持3倍width的指针，这就是我们不能将一个不定长度类型强转为trait object的原因。

将上面两段话总结成表格的话，就是下面这样:

| 类型 | 指向数据的指针 | 数据长度 | 指向虚表的指针 | 总Width |
|-|-|-|-|-|
| `&String` | ✅ | ❌ | ❌ | 1 ✅ |
| `&str` | ✅ | ✅ | ❌ | 2 ✅ |
| `&String as &dyn ToString` | ✅ | ❌ | ✅ | 2 ✅ |
| `&str as &dyn ToString` | ✅ | ✅ | ✅ | 3 ❌ |



#### 不能创建多特性对象

```rust
trait Trait {}
trait Trait2 {}

fn function(t: &(dyn Trait + Trait2)) {}
```

上面代码编译报错如下：

```none
error[E0225]: only auto traits can be used as additional traits in a trait object
 --> src/lib.rs:4:30
  |
4 | fn function(t: &(dyn Trait + Trait2)) {}
  |                      -----   ^^^^^^
  |                      |       |
  |                      |       additional non-auto trait
  |                      |       trait alias used in trait object type (additional use)
  |                      first non-auto trait
  |                      trait alias used in trait object type (first use)
```

让我们回想下：trait object的指针是双width的，它保存了一个指向数据的指针和一个指向虚表的指针，但上面的代码有两个trait，所以这里就有两个虚表，也就是说`&(dyn Trait + Trait2)`得是3倍width才行。（译者补充: 但我们经常在代码里看到`dyn Object + Send + Sync`这类写法是怎么回事？）自动特性之所以可以相加，是因为像`Sync`和`Send`这种自动特性它们没有方法，因此也就没有虚表，所以它们没有这个限制。

绕过这一限制的方法是，通过将不同trait组合成另一个trait来把虚表组合到一起：

```rust
trait Trait {
    fn method(&self) {}
}

trait Trait2 {
    fn method2(&self) {}
}

trait Trait3: Trait + Trait2 {}

// 为任何同时实现了Trait和Trait2的类型自动实现Trait3(译者注: 参见[Generic Blanket Impls](https://github.com/mercury-2025/rust-blog/blob/master/posts/tour-of-rusts-standard-library-traits.md#generic-blanket-impls))
impl<T: Trait + Trait2> Trait3 for T {}

// 将`dyn Trait + Trait2`改为`dyn Trait3` 
fn function(t: &dyn Trait3) {
    t.method(); // ✅
    t.method2(); // ✅
}
```

这种方法的缺点在于，Rust不支持到supertrait的向上强转。也就是我们不能将`dyn Trait3`用于需要`dyn Trait`或`dyn Trait2`的地方。下面这样的代码编不过：

```rust
trait Trait {
    fn method(&self) {}
}

trait Trait2 {
    fn method2(&self) {}
}

trait Trait3: Trait + Trait2 {}

impl<T: Trait + Trait2> Trait3 for T {}

struct Struct;
impl Trait for Struct {}
impl Trait2 for Struct {}

fn takes_trait(t: &dyn Trait) {}
fn takes_trait2(t: &dyn Trait2) {}

fn main() {
    let t: &dyn Trait3 = &Struct;
    takes_trait(t); // ❌
    takes_trait2(t); // ❌
}
```

上面代码编译报错如下：

```none
error[E0308]: mismatched types
  --> src/main.rs:22:17
   |
22 |     takes_trait(t);
   |                 ^ expected trait `Trait`, found trait `Trait3`
   |
   = note: expected reference `&dyn Trait`
              found reference `&dyn Trait3`

error[E0308]: mismatched types
  --> src/main.rs:23:18
   |
23 |     takes_trait2(t);
   |                  ^ expected trait `Trait2`, found trait `Trait3`
   |
   = note: expected reference `&dyn Trait2`
              found reference `&dyn Trait3`
```

报错原因在于，`dyn Trait3`是与`dyn Trait`/`dyn Trait2`完全不同的另一类型，因为他们有不同的虚表结构，虽然实际上`dyn Trait3`确实有`dyn Trait`和`dyn Trait2`的所有方法。要解决此问题，可以使用显式强转:

```rust
trait Trait {}
trait Trait2 {}

trait Trait3: Trait + Trait2 {
    fn as_trait(&self) -> &dyn Trait;
    fn as_trait2(&self) -> &dyn Trait2;
}

impl<T: Trait + Trait2> Trait3 for T {
    fn as_trait(&self) -> &dyn Trait {
        self
    }
    fn as_trait2(&self) -> &dyn Trait2 {
        self
    }
}

struct Struct;
impl Trait for Struct {}
impl Trait2 for Struct {}

fn takes_trait(t: &dyn Trait) {}
fn takes_trait2(t: &dyn Trait2) {}

fn main() {
    let t: &dyn Trait3 = &Struct;
    takes_trait(t.as_trait()); // ✅
    takes_trait2(t.as_trait2()); // ✅
}
```

这种方式简单而直观，看起来Rust编译器似乎应该帮我们自动化地做掉这个事情。Rust已经在deref或不定大小强转等情形下帮我们做过这类自动转换了，所以在这里为什么它就不能帮我们把这个向上强转也自动做掉呢? 这是一个很好的问题，答案你应该也很熟悉了: Rust核心团队正在做其它一些更高优先级以及影响更大的事情，很说得过去。

**要点**
- Rust不支持超过2倍宽度的指针，所以：
    - 我们不能将不定大小类型转为trait objects
    - 我们不能同时trait object定多个trait，但是我们可以通过将多个trait组合为单个trait来应对这一限制


### 用户定义的不定大小类型

```rust
struct Unsized {
    unsized_field: [i32],
}
```

我们可以通过为结构增加一个不定大小字段来定义一个不定大小结构。不定大小结构只能有一个不定大小字段，并且该字段只能是结构的最后一个字段。因为只有这样，编译器才能在编译期确定地知道结构中每个字段的起始偏移，这一点对于有效和快速的字段存取来说至关重要。并且，由于双width指针最多只能描述单个不定大小字段，所以我们不能有更多的不定大小字段，因为它们将会需要更大的width以存储这些信息。

那么，我们怎么才能初始化这样一个自定义的不定大小结构呢? 与我们构造系统原生不定大小类型时的做法一样: 先构造一个确定大小类型的版本，然后将其强转为不定大小的版本。但是，上面的`Unsized`类型就是不定大小类型啊，它的确定大小类型版本又是什么，我们又怎么能构造一个这样的确定大小版本`Unsized`呢? 唯一办法是：使用一个泛型参数来定义我们的不定大小字段成员，这样这个成员既可以是确定大小类型，也可以是不定大小类型:

```rust
struct MaybeSized<T: ?Sized> {
    maybe_sized: T,
}

fn main() {
    // 将MaybeSized<[i32; 3]>不定大小强转为MaybeSized<[i32]>
    let ms: &MaybeSized<[i32]> = &MaybeSized { maybe_sized: [1, 2, 3] };
}
```

好了，那这种东西具体有什么用处呢? 答案是，我们想不到有什么特别有说服力的场景需要它，用户定义的不定大小类型目前仅仅是个半成品，它们的限制超过了它们的收益。大家知道有这么回事就可以了。

**有趣的是:** `std::ffi::OsStr` 和 `std::path::Path`是标准库中的两个不定大小结构，你可能已经用过它们了，但没有意识到它们是不定大小的!

**要点**
- 用户自定义的不定大小类型目前还是半成品，它们的限制超出了它们的收益


## 0大小类型

ZST听起来有点怪，但他们实际上无处不在。



### 单元类型

最常见的ZST是单元类型: `()`。所有空的代码块`{}`都返回`()`，如果代码块非空，但最后一个表达式以分号`;`结尾，则它也返回`()`。比如：

```rust
fn main() {
    let a: () = {};
    let b: i32 = {
        5
    };
    let c: () = {
        5;
    };
}
```

所有没有显式返回值的函数默认也返回`()`。比如：

```rust
// 展开前
fn function() {}

// 展开后
fn function() -> () {}
```

由于`()`是0大小的，所以不同的`()`是相等的，这一点导致了部分相当简单的`Default`，`PartialEq`，以及 `Ord` 实现:

```rust
use std::cmp::Ordering;

impl Default for () {
    fn default() {}
}

impl PartialEq for () {
    fn eq(&self, _other: &()) -> bool {
        true
    }
    fn ne(&self, _other: &()) -> bool {
        false
    }
}

impl Ord for () {
    fn cmp(&self, _other: &()) -> Ordering {
        Ordering::Equal
    }
}
```

编译器知道`()`是0大小的，所以会优化掉与`()`实例相关的交互。比如，一个`Vec<()>`永远不会执行堆上的分配，在这个`Vec`中pushing和popping`()`只是增减它的`len`字段：

```rust
fn main() {
    // '存储'无穷多个()只需要0预留容量
    let mut vec: Vec<()> = Vec::with_capacity(0);
    // 因为这里不涉及任何堆分配或者vec的容量改变
    vec.push(()); // len++
    vec.push(()); // len++
    vec.push(()); // len++
    vec.pop(); // len--
    assert_eq!(2, vec.len());
}
```

上面这个例子没有什么实际用处，还有没有更实际有用的场景可以利用上面这一点的？回答是yes，我们可以通过 `HashMap<Key, Value>` 来得到高效的 `HashSet<Key>` 实现， 方法是将`Value`设为`()`，实际上这正是Rust标准库的做法：

```rust
// std::collections::HashSet
pub struct HashSet<T> {
    map: HashMap<T, ()>,
}
```

**要点**
- 特定ZST类型的所有实例都彼此相等
- Rust编译器会优化掉与ZST相关的交互


### 用户定义的单元大小结构

单元大小结构是不含任何成员字段的结构，也就是:

```rust
struct Struct;
```

单元大小结构比`()`更好的地方在于:
- 可以在其上实现我们需要的任何特性，Rust的特性隔离原则上阻止了我们为`()`实现trait的可能，因为`()`定义在标准库而不是我们的代码里
- 我们可以给单元大小结构赋一个在程序上下文中看起来更有意义的名字
- 与所有其它结构一样，单元大小结构默认是non-Copy的，这一点在我们程序的上下文中有可能会很重要


### 不可能类型（Never Type）

第二个常见的ZST是不可能类型: `!`。它之所以叫不可能类型，是因为它表示不可能计算为任何值的操作结果。

`!`不同于`()`的一些有趣特性:
- `!` 可以强转成任意其它类型(译者注: 编译器语法解析时)
- 我们无法在代码中创建`!`类型的实例

上述第一个属性很有用，有了这一属性，我们才可以使用下面这样的宏：

```rust
// 对于快速写原型很有用
fn example<T>(t: &[T]) -> Vec<T> {
    unimplemented!() // ! 强转为 Vec<T>
}

fn example2() -> i32 {
    // 我们知道parse操作永远不会失败
    match "123".parse::<i32>() {
        Ok(num) => num,
        Err(_) => unreachable!(), // ! 强转为 i32
    }
}

fn example3(some_condition: bool) -> &'static str {
    if !some_condition {
        panic!() // ! 强转为 &str
    } else {
        "str"
    }
}
```

`break`，`continue`，和 `return` 表达式类型也是 `!`:

```rust
fn example() -> i32 {
    // 在这里x可被设置为任何类型
    // 因为在这里代码块永远不会有返回值(译者注: 代码块中的return是example返回，而不是将return的值赋给x)
    let x: String = {
        return 123 // ! 强转为 String
    };
}

fn example2(nums: &[i32]) -> Vec<i32> {
    let mut filtered = Vec::new();
    for num in nums {
        filtered.push(
            if *num < 0 {
                break // ! 强转为 i32
            } else if *num % 2 == 0 {
                *num
            } else {
                continue // ! 强转为 i32
            }
        );
    }
    filtered
}
```

上面所说的第二个有趣特性，使得我们可以在类型这一层上标记代码中的不可能的情形。我们通过下面的函数签名例子来解释这具体是什么意思：

```rust
fn function() -> Result<Success, Error>;
```

我们知道，如果函数成功返回，则`Result`会包含类型`Success`的某个实例，如果函数失败，则`Result`会包含类型`Error`的某个实例。现在看下面这个函数签名：

```rust
fn function() -> Result<Success, !>;
```

同样的，如果函数成功返回，则`Result`会包含类型`Success`的某个实例，如果函数失败，嗯...， 但是等一下，它永远不能失败，因为根据上述第二个特性，我们不可能构造一个`!`的实例。所以，通过上面这个函数签名，我们知道这个函数永远不会失败。那下面这个函数呢：

```rust
fn function() -> Result<!, Error>;
```

反过来也一样: 既然不可能构造成功时的返回值，我们可以确定只要返回，则这个函数肯定失败了。

前一种形式的实际应用可以在`String`的`FromStr`实现中看到，因为将`&str`转为`String`不可能失败：

```rust
#![feature(never_type)]

use std::str::FromStr;

impl FromStr for String {
    type Err = !;
    fn from_str(s: &str) -> Result<String, Self::Err> {
        Ok(String::from(s))
    }
}
```

第二种形式的实际应用可以是一个执行无限循环永不结束的函数，比如一个处理客户端请求的server，除非遇到错误，否则这个函数永不返回：

```rust
#![feature(never_type)]

fn run_server() -> Result<!, ConnectionError> {
    loop {
        let (request, response) = get_request()?;
        let result = request.process();
        response.send(result);
    }
}
```

上面代码中的特性标记(feature flag)是必须的，因为不可能类型本身只存在和工作于Rust内部，在用户代码中使用仍然还只是实验性的。

**要点**
- `!`可以转成任何其它类型(译者注：语法上)
- `!`类型不可以创建任何实例这一特性，使得我们可以在类型层次上声明特定的不可能状态



### 用户定义的伪不可能类型

虽然我们不能定义一个可以转成任何其它类型的自定义类型，但我们可以定义一个不能创建任何实际实例的类型，比如一个不包含任何成员的`enum`：

```rust
enum Void {}
```

通过这种自定义类型，我们可以免去上面所说的特性标记(译者注： `#![feature(never_type)]`):

```rust
enum Void {}

// example 1
impl FromStr for String {
    type Err = Void;
    fn from_str(s: &str) -> Result<String, Self::Err> {
        Ok(String::from(s))
    }
}

// example 2
fn run_server() -> Result<Void, ConnectionError> {
    loop {
        let (request, response) = get_request()?;
        let result = request.process();
        response.send(result);
    }
}
```

这也是Rust标准库中使用的方法，在`String`的`FromStr`实现中，`Err`类型为`std::convert::Infallible`，它长这样：

```rust
pub enum Infallible {}
```



### PhantomData(伪数据)

第3种常见ZST大概是`PhantomData`。`PhantomData`是一个0大小标记结构，它可以用于标记所在的结构具有特定的属性。它有点像自动标记trait，比如`Sized`，`Send`，以及`Sync`，但使用方式上有一点区别。完整的`PhantomData`解释以及它的各种应用超出了本文的范围，所以我们只简单看一个例子。前文提到过这样一段代码：

```rust
#![feature(negative_impls)]

// 该结构为Send+Sync
struct Struct;

// 取消Send特性
impl !Send for Struct {}

// 取消Sync特性
impl !Sync for Struct {}
```

在上面代码中，我们得使用feature flag。那能不能只使用稳定Rust达到同样目的呢? 我们已经知道，当一个结构的所有成员都是`Send`/`Sync`时，该结构自动为`Send`/`Sync`，所以如果我们给`Struct`加一个`!Send`/`!Sync`成员，比如`Rc<()>`的话，那也可以达到取消自动特性的目的：

```rust
use std::rc::Rc;

// 该结构非Send或Sync
struct Struct {
    // 每个实例多了8字节
    _not_send_or_sync: Rc<()>,
}
```

但这种方式不够理想，因为它增加了`Struct`具体实例的大小，每次我们构造`Struct`时，都得凭空多造一个`Rc<()>`。不过，由于`PhantomData`为ZST，它可以在这里帮助我们解决此问题:

```rust
use std::rc::Rc;
use std::marker::PhantomData;

type NotSendOrSyncPhantom = PhantomData<Rc<()>>;

// 该结构非Send或Sync
struct Struct {
    // 不会增加实例的大小
    _not_send_or_sync: NotSendOrSyncPhantom,
}
```

**要点**
- `PhantomData`是0大小标记结构，它可以用于标记其所在结构的某些特定属性



## 总结

- 只有确定大小类型可以放在栈上，也就是说，只有它们可以通过值的方式传递
- 不定大小类型不能放在栈上，且只能以引用方式传递
- 指向不定大小类型的指针是双width的，因为除了指向数据的指针之外，它们还需要额外的比特来记录数据的大小 _或者_ 指向虚表的指针
- `Sized` 是一个"自动的"特性标记
- 所有泛型参数默认自动绑定`Sized`
- 如果我们有一个泛型函数，它的参数为某种指向`T`的指针，比如：`&T`，`Box<T>`，`Rc<T>`，等等，那么我们基本上都要取消默认的`Sized`绑定：`T: ?Sized`
- 利用切片和Rust的自动类型转换，我们可以写出更灵活的API
- 所有trait默认都是`?Sized`的
- `Trait: ?Sized`对于`impl Trait for dyn Trait`是必须的
- 我们可以在单方法的粒度上指定`Self: Sized`
- 绑定`Sized`的特性不能用作trait object
- Rust不支持大于2widths的指针，所以: 
    - 我们不能将不定大小类型强转为trait object
    - 我们不能定义多特性trait object，但可以通过组合多个特性至单个特性来绕过这一点
- 用户定义的不定大小类型目前是半成品，它们的限制大于它们可能有的任何好处
- 一个ZST的所有实例都彼此相等
- Rust编译器会优化掉与ZST的交互
- `!`可以转成任何其它类型(译者注: 语法上)
- 不可能构造`!`的实例，我们可以利用这一点在类型层次上声明不可能情形
- `PhantomData`是0大小标记结构，它可以用于"标记"所在的结构有特定的属性



## 讨论

可以到下列地点讨论本文
- [official Rust users forum](https://users.rust-lang.org/t/blog-post-sizedness-in-rust/46293?u=pretzelhammer)
- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/hx2jd0/sizedness_in_rust/)
- [rust subreddit](https://www.reddit.com/r/rust/comments/hxips7/sizedness_in_rust/)
- [Github](https://github.com/pretzelhammer/rust-blog/discussions)


## 进一步阅读

- [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md)
- [Tour of Rust's Standard Library Traits](./tour-of-rusts-standard-library-traits.md)
- [Beginner's Guide to Concurrent Programming: Coding a Multithreaded Chat Server using Tokio](./chat-server.md)
- [Learning Rust in 2024](./learning-rust-in-2024.md)
- [Using Rust in Non-Rust Servers to Improve Performance](./rust-in-non-rust-servers.md)
- [RESTful API in Sync & Async Rust](./restful-api-in-sync-and-async-rust.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](./too-many-brainfuck-compilers.md)


## 通知

在新blog发布时得到通知
- 订阅本仓库 [releases RSS feed](https://github.com/pretzelhammer/rust-blog/releases.atom) or
- 监控本仓库的发布 (click `Watch` → click `Custom` → select `Releases` → click `Apply`)
