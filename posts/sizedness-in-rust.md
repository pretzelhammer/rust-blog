# Sizedness in Rust

_July 22nd, 2020 · 35 minute read · #rust · #sizedness_

**Table of Contents**

- [Intro](#intro)
- [Sizedness](#sizedness)
- [`Sized` Trait](#sized-trait)
- [`Sized` in Generics](#sized-in-generics)
- [Unsized Types](#unsized-types)
    - [Slices](#slices)
    - [Trait Objects](#trait-objects)
    - [Trait Object Limitations](#trait-object-limitations)
        - [Cannot Cast Unsized Types to Trait Objects](#cannot-cast-unsized-types-to-trait-objects)
        - [Cannot create Multi-Trait Objects](#cannot-create-multi-trait-objects)
    - [User-Defined Unsized Types](#user-defined-unsized-types)
- [Zero-Sized Types](#zero-sized-types)
    - [Unit Type](#unit-type)
    - [User-Defined Unit Structs](#user-defined-unit-structs)
    - [Never Type](#never-type)
    - [User-Defined Pseudo Never Types](#user-defined-pseudo-never-types)
    - [PhantomData](#phantomdata)
- [Conclusion](#conclusion)
- [Discuss](#discuss)
- [Notifications](#notifications)
- [Further Reading](#further-reading)



## Intro

Sizedness is lowkey one of the most important concepts to understand in Rust. It intersects a bunch of other language features in often subtle ways and only rears its ugly head in the form of _"x doesn't have size known at compile time"_ error messages which every Rustacean is all too familiar with. In this article we'll explore all flavors of sizedness from sized types, to unsized types, to zero-sized types while examining their use-cases, benefits, pain points, and workarounds.

Table of phrases I use and what they're suppose to mean:

| Phrase | Shorthand for |
|-|-|
| sizedness | property of being sized or unsized |
| sized type | type with a known size at compile time |
| 1) unsized type _or_<br>2) DST | dynamically-sized type, i.e. size not known at compile time |
| ?sized type | type that may or may not be sized |
| unsized coercion | coercing a sized type into an unsized type |
| ZST | zero-sized type, i.e. instances of the type are 0 bytes in size |
| width | single unit of measurement of pointer width |
| 1) thin pointer _or_<br>2) single-width pointer | pointer that is _1 width_ |
| 1) fat pointer _or_<br>2) double-width pointer | pointer that is _2 widths_ |
| 1) pointer _or_<br>2) reference | some pointer of some width, width will be clarified by context |
| slice | double-width pointer to a dynamically sized view into some array |



## Sizedness

In Rust a type is sized if its size in bytes can be determined at compile-time. Determining a type's size is important for being able to allocate enough space for instances of that type on the stack. Sized types can be passed around by value or by reference. If a type's size can't be determined at compile-time then it's referred to as an unsized type or a DST, Dynamically-Sized Type. Since unsized types can't be placed on the stack they can only be passed around by reference. Some examples of sized and unsized types:

```rust
use std::mem::size_of;

fn main() {
    // primitives
    assert_eq!(4, size_of::<i32>());
    assert_eq!(8, size_of::<f64>());

    // tuples
    assert_eq!(8, size_of::<(i32, i32)>());

    // arrays
    assert_eq!(0, size_of::<[i32; 0]>());
    assert_eq!(12, size_of::<[i32; 3]>());

    struct Point {
        x: i32,
        y: i32,
    }

    // structs
    assert_eq!(8, size_of::<Point>());

    // enums
    assert_eq!(8, size_of::<Option<i32>>());

    // get pointer width, will be
    // 4 bytes wide on 32-bit targets or
    // 8 bytes wide on 64-bit targets
    const WIDTH: usize = size_of::<&()>();

    // pointers to sized types are 1 width
    assert_eq!(WIDTH, size_of::<&i32>());
    assert_eq!(WIDTH, size_of::<&mut i32>());
    assert_eq!(WIDTH, size_of::<Box<i32>>());
    assert_eq!(WIDTH, size_of::<fn(i32) -> i32>());

    const DOUBLE_WIDTH: usize = 2 * WIDTH;

    // unsized struct
    struct Unsized {
        unsized_field: [i32],
    }

    // pointers to unsized types are 2 widths
    assert_eq!(DOUBLE_WIDTH, size_of::<&str>()); // slice
    assert_eq!(DOUBLE_WIDTH, size_of::<&[i32]>()); // slice
    assert_eq!(DOUBLE_WIDTH, size_of::<&dyn ToString>()); // trait object
    assert_eq!(DOUBLE_WIDTH, size_of::<Box<dyn ToString>>()); // trait object
    assert_eq!(DOUBLE_WIDTH, size_of::<&Unsized>()); // user-defined unsized type

    // unsized types
    size_of::<str>(); // compile error
    size_of::<[i32]>(); // compile error
    size_of::<dyn ToString>(); // compile error
    size_of::<Unsized>(); // compile error
}
```

How we determine the size of sized types is straight-forward: all primitives and pointers have known sizes and all structs, tuples, enums, and arrays are just made up of primitives and pointers or other nested structs, tuples, enums, and arrays so we can just count up the bytes recursively, taking into account extra bytes needed for padding and alignment. We can't determine the size of unsized types for similarly straight-forward reasons: slices can have any number of elements in them and can thus be of any size at run-time and trait objects can be implemented by any number of structs or enums and thus can also be of any size at run-time.

**Pro tips**
- pointers of dynamically sized views into arrays are called slices in Rust, e.g. a `&str` is a _"string slice"_, a `&[i32]` is an _"i32 slice"_
- slices are double-width because they store a pointer to the array and the number of elements in the array
- trait object pointers are double-width because they store a pointer to the data and a pointer to a vtable
- unsized structs pointers are double-width because they store a pointer to the struct data and the size of the struct
- unsized structs can only have 1 unsized field and it must be the last field in the struct

To really hammer home the point about double-width pointers for unsized types here's a commented code example comparing arrays to slices:

```rust
use std::mem::size_of;

const WIDTH: usize = size_of::<&()>();
const DOUBLE_WIDTH: usize = 2 * WIDTH;

fn main() {
    // data length stored in type
    // an [i32; 3] is an array of three i32s
    let nums: &[i32; 3] = &[1, 2, 3];

    // single-width pointer
    assert_eq!(WIDTH, size_of::<&[i32; 3]>());

    let mut sum = 0;

    // can iterate over nums safely
    // Rust knows it's exactly 3 elements
    for num in nums {
        sum += num;
    }

    assert_eq!(6, sum);

    // unsized coercion from [i32; 3] to [i32]
    // data length now stored in pointer
    let nums: &[i32] = &[1, 2, 3];

    // double-width pointer required to also store data length
    assert_eq!(DOUBLE_WIDTH, size_of::<&[i32]>());

    let mut sum = 0;

    // can iterate over nums safely
    // Rust knows it's exactly 3 elements
    for num in nums {
        sum += num;
    }

    assert_eq!(6, sum);
}
```

And here's another commented code example comparing structs to trait objects:

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
    // always prints "struct"
    // this is known at compile-time
    s.print();
    // single-width pointer
    assert_eq!(WIDTH, size_of::<&Struct>());
}

fn print_struct2(s2: &Struct2) {
    // always prints "struct2"
    // this is known at compile-time
    s2.print();
    // single-width pointer
    assert_eq!(WIDTH, size_of::<&Struct2>());
}

fn print_trait(t: &dyn Trait) {
    // print "struct" or "struct2" ?
    // this is unknown at compile-time
    t.print();
    // Rust has to check the pointer at run-time
    // to figure out whether to use Struct's
    // or Struct2's implementation of "print"
    // so the pointer has to be double-width
    assert_eq!(DOUBLE_WIDTH, size_of::<&dyn Trait>());
}

fn main() {
    // single-width pointer to data
    let s = &Struct; 
    print_struct(s); // prints "struct"
    
    // single-width pointer to data
    let s2 = &Struct2;
    print_struct2(s2); // prints "struct2"
    
    // unsized coercion from Struct to dyn Trait
    // double-width pointer to point to data AND Struct's vtable
    let t: &dyn Trait = &Struct;
    print_trait(t); // prints "struct"
    
    // unsized coercion from Struct2 to dyn Trait
    // double-width pointer to point to data AND Struct2's vtable
    let t: &dyn Trait = &Struct2;
    print_trait(t); // prints "struct2"
}
```

**Key Takeaways**
- only instances of sized types can be placed on the stack, i.e. can be passed around by value
- instances of unsized types can't be placed on the stack and must be passed around by reference
- pointers to unsized types are double-width because aside from pointing to data they need to do an extra bit of bookkeeping to also keep track of the data's length _or_ point to a vtable



## `Sized` Trait

The `Sized` trait in Rust is an auto trait and a marker trait.

Auto traits are traits that get automatically implemented for a type if it passes certain conditions. Marker traits are traits that mark a type as having a certain property. Marker traits do not have any trait items such as methods, associated functions, associated constants, or associated types. All auto traits are marker traits but not all marker traits are auto traits. Auto traits must be marker traits so the compiler can provide an automatic default implementation for them, which would not be possible if the trait had any trait items.

A type gets an auto `Sized` implementation if all of its members are also `Sized`. What "members" means depends on the containing type, for example: fields of a struct, variants of an enum, elements of an array, items of a tuple, and so on. Once a type has been "marked" with a `Sized` implementation that means its size in bytes is known at compile time.

Other examples of auto marker traits are the `Send` and `Sync` traits. A type is `Send` if it is safe to send that type across threads. A type is `Sync` if it's safe to share references of that type between threads. A type gets auto `Send` and `Sync` implementations if all of its members are also `Send` and `Sync`. What makes `Sized` somewhat special is that it's not possible to opt-out of unlike with the other auto marker traits which are possible to opt-out of.

```rust
#![feature(negative_impls)]

// this type is Sized, Send, and Sync
struct Struct;

// opt-out of Send trait
impl !Send for Struct {}

// opt-out of Sync trait
impl !Sync for Struct {}

impl !Sized for Struct {} // compile error
```

This seems reasonable since there might be reasons why we wouldn't want our type to be sent or shared across threads, however it's hard to imagine a scenario where we'd want the compiler to "forget" the size of our type and treat it as an unsized type as that offers no benefits and merely makes the type more difficult to work with.

Also, to be super pedantic `Sized` is not technically an auto trait since it's not defined using the `auto` keyword but the special treatment it gets from the compiler makes it behave very similarly to auto traits so in practice it's okay to think of it as an auto trait.

**Key Takeaways**
- `Sized` is an "auto" marker trait



## `Sized` in Generics

It's not immediately obvious that whenever we write any generic code every generic type parameter gets auto-bound with the `Sized` trait by default.

```rust
// this generic function...
fn func<T>(t: T) {}

// ...desugars to...
fn func<T: Sized>(t: T) {}

// ...which we can opt-out of by explicitly setting ?Sized...
fn func<T: ?Sized>(t: T) {} // compile error

// ...which doesn't compile since t doesn't have
// a known size so we must put it behind a pointer...
fn func<T: ?Sized>(t: &T) {} // compiles
fn func<T: ?Sized>(t: Box<T>) {} // compiles
```

**Pro tips**
- `?Sized` can be pronounced _"optionally sized"_ or _"maybe sized"_ and adding it to a type parameter's bounds allows the type to be sized or unsized
- `?Sized` in general is referred to as a _"widening bound"_ or a _"relaxed bound"_ as it relaxes rather than constrains the type parameter
- `?Sized` is the only relaxed bound in Rust

So why does this matter? Well, any time we're working with a generic type and that type is behind a pointer we almost always want to opt-out of the default `Sized` bound to make our function more flexible in what argument types it will accept. Also, if we don't opt-out of the default `Sized` bound we'll eventually get some surprising and confusing compile error messages.

Let me take you on the journey of the first generic function I ever wrote in Rust. I started learning Rust before the `dbg!` macro landed in stable so the only way to print debug values was to type out `println!("{:?}", some_value);` every time which is pretty tedious so I decided to write a `debug` helper function like this:

```rust
use std::fmt::Debug;

fn debug<T: Debug>(t: T) { // T: Debug + Sized
    println!("{:?}", t);
}

fn main() {
    debug("my str"); // T = &str, &str: Debug + Sized ✔️
}
```

So far so good, but the function takes ownership of any values passed to it which is kinda annoying so I changed the function to only take references instead:

```rust
use std::fmt::Debug;

fn dbg<T: Debug>(t: &T) { // T: Debug + Sized
    println!("{:?}", t);
}

fn main() {
    dbg("my str"); // &T = &str, T = str, str: Debug + !Sized ❌
}
```

Which now throws this error:

```rust
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
  = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
help: consider relaxing the implicit `Sized` restriction
  |
3 | fn dbg<T: Debug + ?Sized>(t: &T) {
  |   
```

When I first saw this I found it incredibly confusing. Despite making my function more restrictive in what arguments it takes than before it now somehow throws a compile error! What is going on?

I've already kinda spoiled the answer in the code comments above, but basically: Rust performs pattern matching when resolving `T` to its concrete types during compilation. Here's a couple tables to help clarify:

| Type | `T` | `&T` |
|------------|---|----|
| `&str` | `T` = `&str` | `T` = `str` |

| Type | `Sized` |
|-|-|
| `str` | ❌ |
| `&str` | ✔️ |
| `&&str` | ✔️ |

This is why I had to add a `?Sized` bound to make the function work as intended after changing it to take references. The working function below:

```rust
use std::fmt::Debug;

fn debug<T: Debug + ?Sized>(t: &T) { // T: Debug + ?Sized
    println!("{:?}", t);
}

fn main() {
    debug("my str"); // &T = &str, T = str, str: Debug + !Sized ✔️
}
```

**Key Takeaways**
- all generic type parameters are auto-bound with `Sized` by default
- if we have a generic function which takes an argument of some `T` behind a pointer, e.g. `&T`, `Box<T>`, `Rc<T>`, et cetera, then we almost always want to opt-out of the default `Sized` bound with `T: ?Sized`



## Unsized Types



### Slices

The most common slices are string slices `&str` and array slices `&[T]`. What's nice about slices is that many other types coerce to them, so leveraging slices and Rust's auto type coercions allow us to write flexible APIs.

Type coercions can happen in several places but most notably on function arguments and at method calls. The kinds of type coercions we're interested in are deref coercions and unsized coercions. A deref coercion is when a `T` gets coerced into a `U` following a deref operation, i.e. `T: Deref<Target = U>`, e.g. `String.deref() -> str`. An unsized coercion is when a `T` gets coerced into a `U` where `T` is a sized type and `U` is an unsized type, i.e. `T: Unsize<U>`, e.g. `[i32; 3] -> [i32]`.

```rust
trait Trait {
    fn method(&self) {}
}

impl Trait for str {
    // can now call "method" on
    // 1) str or
    // 2) String since String: Deref<Target = str>
}
impl<T> Trait for [T] {
    // can now call "method" on
    // 1) any &[T]
    // 2) any U where U: Deref<Target = [T]>, e.g. Vec<T>
    // 3) [T; N] for any N, since [T; N]: Unsize<[T]>
}

fn str_fun(s: &str) {}
fn slice_fun<T>(s: &[T]) {}

fn main() {
    let str_slice: &str = "str slice";
    let string: String = "string".to_owned();

    // function args
    str_fun(str_slice);
    str_fun(&string); // deref coercion

    // method calls
    str_slice.method();
    string.method(); // deref coercion

    let slice: &[i32] = &[1];
    let three_array: [i32; 3] = [1, 2, 3];
    let five_array: [i32; 5] = [1, 2, 3, 4, 5];
    let vec: Vec<i32> = vec![1];

    // function args
    slice_fun(slice);
    slice_fun(&vec); // deref coercion
    slice_fun(&three_array); // unsized coercion
    slice_fun(&five_array); // unsized coercion

    // method calls
    slice.method();
    vec.method(); // deref coercion
    three_array.method(); // unsized coercion
    five_array.method(); // unsized coercion
}
```

**Key Takeaways**
- leveraging slices and Rust's auto type coercions allows us to write flexible APIs



### Trait Objects

Traits are `?Sized` by default. This program:

```rust
trait Trait: ?Sized {}
```

Throws this error:

```rust
error: `?Trait` is not permitted in supertraits
 --> src/main.rs:1:14
  |
1 | trait Trait: ?Sized {}
  |              ^^^^^^
  |
  = note: traits are `?Sized` by default
```

We'll get into why traits are `?Sized` by default soon but first let's ask ourselves what are the implications of a trait being `?Sized`? Lets desugar the above example:

```rust
trait Trait where Self: ?Sized {}
```

Okay, so by default traits allow `self` to possibly be an unsized type. As we learned earlier we can't pass unsized types around by value, so that limits us in the kind of methods we can define in the trait. It should be impossible to write a method the takes or returns `self` by value and yet this surprisingly compiles:

```rust
trait Trait {
    fn method(self); // compiles
}
```

However the moment we try to implement the method, either by providing a default implementation or by implementing the trait for an unsized type, we get compile errors:

```rust
trait Trait {
    fn method(self) {} // compile error
}

impl Trait for str {
    fn method(self) {} // compile error
}
```

Throws:

```rust
error[E0277]: the size for values of type `Self` cannot be known at compilation time
 --> src/lib.rs:2:15
  |
2 |     fn method(self) {}
  |               ^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `Self`
  = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
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
  = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
  = note: all local variables must have a statically known size
  = help: unsized locals are gated as an unstable feature
```

If we're determined to pass `self` around by value we can fix the first error by explicitly binding the trait with `Sized`:

```rust
trait Trait: Sized {
    fn method(self) {} // compiles
}

impl Trait for str { // compile error
    fn method(self) {}
}
```

Now throws:

```rust
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
  = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
```

Which is okay, as we knew upon binding the trait with `Sized` we'd no longer be able to implement it for unsized types such as `str`. If on the other hand we really wanted to implement the trait for `str` an alternative solution would be to keep the trait `?Sized` and pass `self` around by reference:

```rust
trait Trait {
    fn method(&self) {} // compiles
}

impl Trait for str {
    fn method(&self) {} // compiles
}
```

Instead of marking the entire trait as `?Sized` or `Sized` we have the more granular and precise option of marking individual methods as `Sized` like so:

```rust
trait Trait {
    fn method(self) where Self: Sized {}
}

impl Trait for str {} // compiles!?

fn main() {
    "str".method(); // compile error
}
```

It's surprising that Rust compiles `impl Trait for str {}` without any complaints, but it eventually catches the error when we attempt to call `method` on an unsized type so all is fine. It's a little weird but affords us some flexibility in implementing traits with some `Sized` methods for unsized types as long as we never call the `Sized` methods:

```rust
trait Trait {
    fn method(self) where Self: Sized {}
    fn method2(&self) {}
}

impl Trait for str {} // compiles

fn main() {
    // we never call "method" so no errors
    "str".method2(); // compiles
}
```

Now back to the original question, why are traits `?Sized` by default? The answer is trait objects. Trait objects are inherently unsized because any type of any size can implement a trait, therefore we can only implement `Trait` for `dyn Trait` if `Trait: ?Sized`. To put it in code:

```rust
trait Trait: ?Sized {}

// the above is REQUIRED for

impl Trait for dyn Trait {
    // compiler magic here
}

// since `dyn Trait` is unsized

// and now we can use `dyn Trait` in our program

fn function(t: &dyn Trait) {} // compiles
```

If we try to actually compile the above program we get:

```rust
error[E0371]: the object type `(dyn Trait + 'static)` automatically implements the trait `Trait`
 --> src/lib.rs:5:1
  |
5 | impl Trait for dyn Trait {
  | ^^^^^^^^^^^^^^^^^^^^^^^^ `(dyn Trait + 'static)` automatically implements trait `Trait`
```

Which is the compiler telling us to chill since it automatically provides the implementation of `Trait` for `dyn Trait`. Again, since `dyn Trait` is unsized the compiler can only provide this implementation if `Trait: ?Sized`. If we bound `Trait` by `Sized` then `Trait` becomes _"object unsafe"_ which is a term that means we can't cast types which implement `Trait` to trait objects of `dyn Trait`. As expected this program does not compile:

```rust
trait Trait: Sized {}

fn function(t: &dyn Trait) {} // compile error
```

Throws:

```rust
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

Lets try to make an `?Sized` trait with a `Sized` method and see if we can cast it to a trait object:

```rust
trait Trait {
    fn method(self) where Self: Sized {}
    fn method2(&self) {}
}

fn function(arg: &dyn Trait) { // compiles
    arg.method(); // compile error
    arg.method2(); // compiles
}
```

As we saw before everything is okay as long as we don't call the `Sized` method on the trait object.

**Key Takeaways**
- all traits are `?Sized` by default
- `Trait: ?Sized` is required for `impl Trait for dyn Trait`
- we can require `Self: Sized` on a per-method basis
- traits bound by `Sized` can't be made into trait objects



### Trait Object Limitations

Even if a trait is object-safe there are still sizedness-related edge cases which limit what types can be cast to trait objects and how many and what kind of traits can be represented by a trait object.



#### Cannot Cast Unsized Types to Trait Objects

```rust
fn generic<T: ToString>(t: T) {}
fn trait_object(t: &dyn ToString) {}

fn main() {
    generic(String::from("String")); // compiles
    generic("str"); // compiles
    trait_object(&String::from("String")); // compiles, unsized coercion
    trait_object("str"); // compile error, unsized coercion impossible
}
```

Throws:

```rust
error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> src/main.rs:8:18
  |
8 |     trait_object("str"); // compile error
  |                  ^^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `str`
  = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
  = note: required for the cast to the object type `dyn std::string::ToString`
```

The reason why passing a `&String` to a function expecting a `&dyn ToString` works is because of type coercion. `String` implements `ToString` and we can convert a sized type such as `String` into an unsized type such as `dyn ToString` via an unsized coercion. `str` also implements `ToString` and converting `str` into a `dyn ToString` would also require an unsized coercion but `str` is already unsized! How do we unsize an already unsized type into another unsized type?

`&str` pointers are double-width, storing a pointer to the data and the data length. `&dyn ToString` pointers are also double-width, storing a pointer to the data and a pointer to a vtable. To coerce a `&str` into a `&dyn toString` would require a triple-width pointer to store a pointer to the data, the data length, and a pointer to a vtable. Rust does not support triple-width pointers so casting an unsized type to a trait object is not possible.

Previous 2 paragraphs summarized in a table:

| Type | Pointer to Data | Data Length | Pointer to VTable | Total Width |
|-|-|-|-|-|
| `&String` | ✔️ | ❌ | ❌ | 1 ✔️ |
| `&str` | ✔️ | ✔️ | ❌ | 2 ✔️ |
| `&String as &dyn ToString` | ✔️ | ❌ | ✔️ | 2 ✔️ |
| `&str as &dyn ToString` | ✔️ | ✔️ | ✔️ | 3 ❌ |



#### Cannot create Multi-Trait Objects

```rust
trait Trait {}
trait Trait2 {}

fn function(t: &(dyn Trait + Trait2)) {}
```

Throws:

```rust
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

Remember that a trait object pointer is double-width: storing 1 pointer to the data and another to the vtable, but there's 2 traits here so there's 2 vtables which would require the `&(dyn Trait + Trait2)` pointer to be 3 widths. Auto-traits like `Sync` and `Send` are allowed since they don't have methods and thus don't have vtables.

The workaround for this is to combine vtables by combining the traits using another trait like so:

```rust
trait Trait {
    fn method(&self) {}
}

trait Trait2 {
    fn method2(&self) {}
}

trait Trait3: Trait + Trait2 {}

// auto blanket impl Trait3 for any type that also impls Trait & Trait2
impl<T: Trait + Trait2> Trait3 for T {}

// from `dyn Trait + Trait2` to `dyn Trait3` 
fn function(t: &dyn Trait3) {
    t.method(); // compiles
    t.method2(); // compiles
}
```

One downside of this workaround is that Rust does not support supertrait upcasting. What this means is that if we have a `dyn Trait3` we can't use it where we need a `dyn Trait` or a `dyn Trait2`. This program does not compile:

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
    takes_trait(t); // compile error
    takes_trait2(t); // compile error
}
```

Throws:

```rust
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

This is because `dyn Trait3` is a distinct type from `dyn Trait` and `dyn Trait2` in the sense that they have different vtable layouts, although `dyn Trait3` does contain all the methods of `dyn Trait` and `dyn Trait2`. The workaround here is to add explicit casting methods:

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
    takes_trait(t.as_trait()); // compiles
    takes_trait2(t.as_trait2()); // compiles
}
```

This is a simple and straight-forward workaround that seems like something the Rust compiler could automate for us. Rust is not shy about performing type coercions as we have seen with deref and unsized coercions, so why isn't there a trait upcasting coercion? This is a good question with a familiar answer: the Rust core team is working on other higher-priority and higher-impact features. Fair enough.

**Key Takeaways**
- Rust doesn't support pointers wider than 2 widths so
    - we can't cast unsized types to trait objects
    - we can't have multi-trait objects, but we can work around this by coalescing multiple traits into a single trait



### User-Defined Unsized Types

```rust
struct Unsized {
    unsized_field: [i32],
}
```

We can define an unsized struct by giving the struct an unsized field. Unsized structs can only have 1 unsized field and it must be the last field in the struct. This is a requirement so that the compiler can determine the starting offset of every field in the struct at compile-time, which is important for efficient and fast field access. Furthermore, a single unsized field is the most that can be tracked using a double-width pointer, as more unsized fields would require more widths.

So how do we even instantiate this thing? The same way we do with any unsized type: by first making a sized version of it then coercing it into the unsized version. However, `Unsized` is always unsized by definition, there's no way to make a sized version of it! The only workaround is to make the struct generic so that it can exist in both sized and unsized versions:

```rust
struct MaybeSized<T: ?Sized> {
    maybe_sized: T,
}

fn main() {
    // unsized coercion from MaybeSized<[i32; 3]> to MaybeSized<[i32]>
    let ms: &MaybeSized<[i32]> = &MaybeSized { maybe_sized: [1, 2, 3] };
}
```

So what are the use-cases of this? There aren't any particularly compelling ones, user-defined unsized types are a pretty half-baked feature right now and their limitations outweigh any benefits. They're mentioned here purely for the sake of comprehensiveness.

**Fun fact:** `std::ffi::OsStr` and `std::path::Path` are 2 unsized structs in the standard library that you've probably used before without realizing!

**Key Takeaways**
- user-defined unsized types are a half-baked feature right now and their limitations outweigh any benefits



## Zero-Sized Types

ZSTs sound exotic at first but they're used everywhere.



### Unit Type

The most common ZST is the unit type: `()`. All empty blocks `{}` evaluate to `()` and if the block is non-empty but the last expression is discarded with a semicolon `;` then it also evaluates to `()`. Example:

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

Every function which doesn't have an explicit return type returns `()` by default.

```rust
// with sugar
fn function() {}

// desugared
fn function() -> () {}
```

Since `()` is zero bytes all instances of `()` are the same which makes for some really simple `Default`, `PartialEq`, and `Ord` implementations:

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

The compiler understands `()` is zero-sized and optimizes away interactions with instances of `()`. For example, a `Vec<()>` will never make any heap allocations, and pushing and popping `()` from the `Vec` just increments and decrements its `len` field:

```rust
fn main() {
    // zero capacity is all the capacity we need to "store" infinitely many ()
    let mut vec: Vec<()> = Vec::with_capacity(0);
    // causes no heap allocations or vec capacity changes
    vec.push(()); // len++
    vec.push(()); // len++
    vec.push(()); // len++
    vec.pop(); // len--
    assert_eq!(2, vec.len());
}
```

The above example has no practical applications, but is there any situation where we can take advantage of the above idea in a meaningful way? Surprisingly yes, we can get an efficient `HashSet<Key>` implementation from a `HashMap<Key, Value>` by setting the `Value` to `()` which is exactly how `HashSet` in the Rust standard library works:

```rust
// std::collections::HashSet
pub struct HashSet<T> {
    map: HashMap<T, ()>,
}
```

**Key Takeaways**
- all instances of a ZST are equal to each other
- Rust compiler knows to optimize away interactions with ZSTs



### User-Defined Unit Structs

A unit struct is any struct without any fields, e.g.

```rust
struct Struct;
```

Properties that make unit structs more useful than `()`:
- we can implement whatever traits we want on our own unit structs, Rust's trait orphan rules prevent us from implementing traits for `()` as it's defined in the standard library
- unit structs can be given meaningful names within the context of our program
- unit structs, like all structs, are non-Copy by default, which may be important in the context of our program



### Never Type

The second most common ZST is the never type: `!`. It's called the never type because it represents computations that never resolve to any value at all.

A couple interesting properties of `!` that make it different from `()`:
- `!` can be coerced into any other type
- it's not possible to create instances of `!`

The first interesting property is very useful for ergonomics and allows us to use handy macros like these:

```rust
// nice for quick prototyping
fn example<T>(t: &[T]) -> Vec<T> {
    unimplemented!() // ! coerced to Vec<T>
}

fn example2() -> i32 {
    // we know this parse call will never fail
    match "123".parse::<i32>() {
        Some(num) => num,
        None => unreachable!(), // ! coerced to i32
    }
}

fn example3(some_condition: bool) -> &'static str {
    if !some_condition {
        panic!() // ! coerced to &str
    } else {
        "str"
    }
}
```

`break`, `continue`, and `return` expressions also have type `!`:

```rust
fn example() -> i32 {
    // we can set the type of x to anything here
    // since the block never evaluates to any value
    let x: String = {
        return 123 // ! coerced to String
    };
}

fn example2(nums: &[i32]) -> Vec<i32> {
    let mut filtered = Vec::new();
    for num in nums {
        filtered.push(
            if *num < 0 {
                break // ! coerced to i32
            } else if *num % 2 == 0 {
                *num
            } else {
                continue // ! coerced to i32
            }
        );
    }
    filtered
}
```

The seconding interesting property of `!` allows us to mark certain states as impossible on a type level. Lets take this function signature as an example:

```rust
fn function() -> Result<Success, Error>;
```

We know that if the function returns and was successful the `Result` will contain some instance of type `Success` and if it errored `Result` will contain some instance of type `Error`. Now lets compare that to this function signature:

```rust
fn function() -> Result<Success, !>;
```

We know that if the function returns and was successful the `Result` will hold some instance of type `Success` and if it errored... but wait, it can never error, since it's impossible to create instances of `!`. Given the above function signature we know this function will never error. How about this function signature:

```rust
fn function() -> Result<!, Error>;
```

The inverse of the previous is now true: if this function returns we know it must have errored as success is impossible.

A practical application of the former example would be the `FromStr` implementation for `String` as it's impossible to fail converting a `&str` into a `String`:

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

A practical application of the latter example would be a function that runs an infinite loop that's never meant to return, like a server responding to client requests, unless there's some error:

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

The feature flag is necessary because while the never type exists and works within Rust internals using it in user-code is still considered experimental.

**Key Takeaways**
- `!` can be coerced into any other type
- it's not possible to create instances of `!` which we can use to mark certain states as impossible at a type level



### User-Defined Pseudo Never Types

While it's not possible to define a type that can coerce to any other type it is possible to define a type which is impossible to create instances of such as an `enum` without any variants:

```rust
enum Void {}
```

This allows us to remove the feature flag from the previous 2 examples and implement them using stable Rust:

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

This is the technique the Rust standard library uses, as the `Err` type for the `FromStr` implementation of `String` is `std::convert::Infallible` which is defined as:

```rust
pub enum Infallible {}
```



### PhantomData

The third most commonly used ZST is probably `PhantomData`. `PhantomData` is a zero-sized marker struct which can be used to "mark" a containing struct as having certain properties. It's similar in purpose to its auto marker trait cousins such as `Sized`, `Send`, and `Sync` but being a marker struct is used a little bit differently. Giving a thorough explanation of `PhantomData` and exploring all of its use-cases is outside the scope of this article so lets only briefly go over a single simple example. Recall this code snippet presented earlier:

```rust
#![feature(negative_impls)]

// this type is Send and Sync
struct Struct;

// opt-out of Send trait
impl !Send for Struct {}

// opt-out of Sync trait
impl !Sync for Struct {}
```

It's unfortunate that we have to use a feature flag, can we accomplish the same result using only stable Rust? As we've learned, a type is only `Send` and `Sync` if all of its members are also `Send` and `Sync`, so we can add a `!Send` and `!Sync` member to `Struct` like `Rc<()>`:

```rust
use std::rc::Rc;

// this type is not Send or Sync
struct Struct {
    // adds 8 bytes to every instance
    _not_send_or_sync: Rc<()>,
}
```

This is less than ideal because it adds size to every instance of `Struct` and we now also have to conjure a `Rc<()>` from thin air every time we want to create a `Struct`. Since `PhantomData` is a ZST it solves both of these problems:

```rust
use std::rc::Rc;
use std::marker::PhantomData;

type NotSendOrSyncPhantom = PhantomData<Rc<()>>;

// this type is not Send or Sync
struct Struct {
    // adds no additional size to instances
    _not_send_or_sync: NotSendOrSyncPhantom,
}
```

**Key Takeaways**
- `PhantomData` is a zero-sized marker struct which can be used to "mark" a containing struct as having certain properties



## Conclusion

- only instances of sized types can be placed on the stack, i.e. can be passed around by value
- instances of unsized types can't be placed on the stack and must be passed around by reference
- pointers to unsized types are double-width because aside from pointing to data they need to do an extra bit of bookkeeping to also keep track of the data's length _or_ point to a vtable
- `Sized` is an "auto" marker trait
- all generic type parameters are auto-bound with `Sized` by default
- if we have a generic function which takes an argument of some `T` behind a pointer, e.g. `&T`, `Box<T>`, `Rc<T>`, et cetera, then we almost always want to opt-out of the default `Sized` bound with `T: ?Sized`
- leveraging slices and Rust's auto type coercions allows us to write flexible APIs
- all traits are `?Sized` by default
- `Trait: ?Sized` is required for `impl Trait for dyn Trait`
- we can require `Self: Sized` on a per-method basis
- traits bound by `Sized` can't be made into trait objects
- Rust doesn't support pointers wider than 2 widths so
    - we can't cast unsized types to trait objects
    - we can't have multi-trait objects, but we can work around this by coalescing multiple traits into a single trait
- user-defined unsized types are a half-baked feature right now and their limitations outweigh any benefits
- all instances of a ZST are equal to each other
- Rust compiler knows to optimize away interactions with ZSTs
- `!` can be coerced into any other type
- it's not possible to create instances of `!` which we can use to mark certain states as impossible at a type level
- `PhantomData` is a zero-sized marker struct which can be used to "mark" a containing struct as having certain properties



## Discuss

Discuss this article on
- [official Rust users forum](https://users.rust-lang.org/t/blog-post-sizedness-in-rust/46293?u=pretzelhammer)
- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/hx2jd0/sizedness_in_rust/)
- [Twitter](https://twitter.com/pretzelhammer/status/1286669073137491973)
- [rust subreddit](https://www.reddit.com/r/rust/comments/hxips7/sizedness_in_rust/)


## Notifications

Get notified when the next blog post get published by
- [Following pretzelhammer on Twitter](https://twitter.com/pretzelhammer) or
- Watching this repo's releases (click on `Watch` dropdown and select `Releases only`)



## Further Reading

- [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md)
- [Learning Rust in 2020](./learning-rust-in-2020.md)
