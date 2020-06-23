# Common Rust Lifetime Misconceptions

_May 19th, 2020 · 37 minute read · #rust · #lifetimes_

**Table of Contents**
- [Intro](#intro)
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
    - [11) `'static` refs can always be coerced into `'a` refs](#11-static-refs-can-always-be-coerced-into-a-refs)
- [Conclusion](#conclusion)
- [Discuss](#discuss)
- [Follow](#follow)
- [Further Reading](#further-reading)



## Intro

I've held all of these misconceptions at some point and I see many beginners struggle with these misconceptions today. Some of my terminology might be non-standard, so here's a table of shorthand phrases I use and what I intend for them to mean.

| Phrase | Shorthand for |
|-|-|
| `T` | 1) a set containing all possible types _or_<br>2) some type within that set |
| owned type | some non-reference type, e.g. `i32`, `String`, `Vec`, etc |
| 1) borrowed type _or_<br>2) ref type | some reference type regardless of mutability, e.g. `&i32`, `&mut i32`, etc |
| 1) mut ref _or_<br>2) exclusive ref | exclusive mutable reference, i.e. `&mut T` |
| 1) immut ref _or_<br>2) shared ref | shared immutable reference, i.e. `&T` |



## The Misconceptions

In a nutshell: A variable's lifetime is how long the data it points to can be statically verified by the compiler to be valid at its current memory address. I'll now spend the next ~6500 words going into more detail about where people commonly get confused.



### 1) `T` only contains owned types

This misconception is more about generics than lifetimes but generics and lifetimes are tightly interwined in Rust so it's not possible to talk about one without also talking about the other. Anyway:

When I first started learning Rust I understood that `i32`, `&i32`, and `&mut i32` are different types. I also understood that some generic type variable `T` represents a set which contains all possible types. However, despite understanding both of these things separately, I wasn't able to understand them together. In my newbie Rust mind this is how I thought generics worked:

| | | | |
|-|-|-|-|
| **Type Variable** | `T` | `&T` | `&mut T` |
| **Examples** | `i32` | `&i32` | `&mut i32` |

`T` contains all owned types. `&T` contains all immutably borrowed types. `&mut T` contains all mutably borrowed types. `T`, `&T`, and `&mut T` are disjoint finite sets. Nice, simple, clean, easy, intuitive, and completely totally wrong. This is how generics actually work in Rust:

| | | | |
|-|-|-|-|
| **Type Variable** | `T` | `&T` | `&mut T` |
| **Examples** | `i32`, `&i32`, `&mut i32`, `&&i32`, `&mut &mut i32`, ... | `&i32`, `&&i32`, `&&mut i32`, ... | `&mut i32`, `&mut &mut i32`, `&mut &i32`, ... |

`T`, `&T`, and `&mut T` are all infinite sets, since it's possible to borrow a type ad-infinitum. `T` is a superset of both `&T` and `&mut T`. `&T` and `&mut T` are disjoint sets. Here's a couple examples which validate these concepts:

```rust
trait Trait {}

impl<T> Trait for T {}

impl<T> Trait for &T {} // compile error

impl<T> Trait for &mut T {} // compile error
```

The above program doesn't compile as expected:

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

The compiler doesn't allow us to define an implementation of `Trait` for `&T` and `&mut T` since it would conflict with the implementation of `Trait` for `T` which already includes all of `&T` and `&mut T`. The program below compiles as expected, since `&T` and `&mut T` are disjoint:

```rust
trait Trait {}

impl<T> Trait for &T {} // compiles

impl<T> Trait for &mut T {} // compiles
```

**Key Takeaways**
- `T` is a superset of both `&T` and `&mut T`
- `&T` and `&mut T` are disjoint sets


### 2) if `T: 'static` then `T` must be valid for the entire program

**Misconception Corollaries**
- `T: 'static` should be read as _"`T` has a `'static` lifetime"_
- `&'static T` and `T: 'static` are the same thing
- if `T: 'static` then `T` must be immutable
- if `T: 'static` then `T` can only be created at compile time

Most Rust beginners get introduced to the `'static` lifetime for the first time in a code example that looks something like this:

```rust
fn main() {
    let str_literal: &'static str = "str literal";
}
```

They get told that `"str literal"` is hardcoded into the compiled binary and is loaded into read-only memory at run-time so it's immutable and valid for the entire program and that's what makes it `'static`. These concepts are further reinforced by the rules surrounding defining `static` variables using the `static` keyword.

```rust
static BYTES: [u8; 3] = [1, 2, 3];
static mut MUT_BYTES: [u8; 3] = [1, 2, 3];

fn main() {
   MUT_BYTES[0] = 99; // compile error, mutating static is unsafe

    unsafe {
        MUT_BYTES[0] = 99;
        assert_eq!(99, MUT_BYTES[0]);
    }
}
```

Regarding `static` variables
- they can only be created at compile-time
- they should be immutable, mutating them is unsafe
- they're valid for the entire program

The `'static` lifetime was probably named after the default lifetime of `static` variables, right? So it makes sense that the `'static` lifetime has to follow all the same rules, right?

Well yes, but a type _with_ a `'static` lifetime is different from a type _bounded by_ a `'static` lifetime. The latter can be dynamically allocated at run-time, can be safely and freely mutated, can be dropped, and can live for arbitrary durations.

It's important at this point to distinguish `&'static T` from `T: 'static`.

`&'static T` is an immutable reference to some `T` that can be safely held indefinitely long, including up until the end of the program. This is only possible if `T` itself is immutable and does not move _after the reference was created_. `T` does not need to be created at compile-time. It's possible to generate random dynamically allocated data at run-time and return `'static` references to it at the cost of leaking memory, e.g.

```rust
use rand;

// generate random 'static str refs at run-time
fn rand_str_generator() -> &'static str {
    let rand_string = rand::random::<u64>().to_string();
    Box::leak(rand_string.into_boxed_str())
}
```

`T: 'static` is some `T` that can be safely held indefinitely long, including up until the end of the program. `T: 'static` includes all `&'static T` however it also includes all owned types, like `String`, `Vec`, etc. The owner of some data is guaranted that data will never get invalidated as long as the owner holds onto it, therefore the owner can safely hold onto the data indefinitely long, including up until the end of the program. `T: 'static` should be read as _"`T` is bounded by a `'static` lifetime"_ not _"`T` has a `'static` lifetime"_. A program to help illustrate these concepts:

```rust
use rand;

fn drop_static<T: 'static>(t: T) {
    std::mem::drop(t);
}

fn main() {
    let mut strings: Vec<String> = Vec::new();
    for _ in 0..10 {
        if rand::random() {
            // all the strings are randomly generated
            // and dynamically allocated at run-time
            let string = rand::random::<u64>().to_string();
            strings.push(string);
        }
    }

    // strings are owned types so they're bounded by 'static
    for mut string in strings {
        // all the strings are mutable
        string.push_str("a mutation");
        // all the strings are droppable
        drop_static(string); // compiles
    }

    // all the strings have been invalidated before the end of the program
    println!("i am the end of the program");
}
```

**Key Takeaways**
- `T: 'static` should be read as _"`T` is bounded by a `'static` lifetime"_
- if `T: 'static` then `T` can be a borrowed type with a `'static` lifetime _or_ an owned type
- since `T: 'static` includes owned types that means `T`
    - can be dynamically allocated at run-time
    - does not have to be valid for the entire program
    - can be safely and freely mutated
    - can be dynamically dropped at run-time
    - can have lifetimes of different durations



### 3) `&'a T` and `T: 'a` are the same thing

This misconception is a generalized version of the one above.

`&'a T` requires and implies `T: 'a` since a reference to `T` of lifetime `'a` cannot be valid for `'a` if `T` itself is not valid for `'a`. For example, the Rust compiler will never allow the construction of the type `&'static Ref<'a, T>` because if `Ref` is only valid for `'a` we can't make a `'static` reference to it.

`T: 'a` includes all `&'a T` but the reverse is not true.

```rust
// only takes ref types bounded by 'a
fn t_ref<'a, T: 'a>(t: &'a T) {}

// takes any types bounded by 'a
fn t_bound<'a, T: 'a>(t: T) {}

// owned type which contains a reference
struct Ref<'a, T: 'a>(&'a T);

fn main() {
    let string = String::from("string");

    t_bound(&string); // compiles
    t_bound(Ref(&string)); // compiles
    t_bound(&Ref(&string)); // compiles

    t_ref(&string); // compiles
    t_ref(Ref(&string)); // compile error, expected ref, found struct
    t_ref(&Ref(&string)); // compiles

    // string var is bounded by 'static which is bounded by 'a
    t_bound(string); // compiles
}
```

**Key Takeaways**
- `T: 'a` is more general and more flexible than `&'a T`
- `T: 'a` accepts owned types, owned types which contain references, and references
- `&'a T` only accepts references
- if `T: 'static` then `T: 'a` since `'static` >= `'a` for all `'a`



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

It throws this compile error:

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


### 11) `'static` refs can always be coerced into `'a` refs

I presented this code example earlier:

```rust
fn get_str<'a>() -> &'a str; // generic version
fn get_str() -> &'static str; // 'static version
```

Several readers contacted me to ask if there was a practical difference between the two. At first I wasn't sure but after some investigation it unforuntately turns out that the answer is yes, there is a practical difference between these two functions.

So ordinarily, when working with values, we can use a `'static` ref in place of an `'a` ref because Rust automatically coerces `'static` refs into `'a` refs. Intuitively this makes sense, since using a ref with a long lifetime where only a short lifetime is required will never cause any memory safety issues. The program below compiles as expected:

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

However this coercion does not take place when the references are part of a function's type signature, so this does not compile:

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

Throws this error:

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

It's debatable whether or not this is a Rust Gotcha, since it's not a simple straight-forward case of coercing a `&'static str` into a `&'a str` but coercing a `for<T> Fn() -> &'static T` into a `for<'a, T> Fn() -> &'a T`. The former is a coercion between values and the latter is a coercion between types.

**Key Takeaways**
- functions with `for<'a, T> fn() -> &'a T` signatures are more flexible and work in more scenarios than functions with `for<T> fn() -> &'static T` signatures



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
- functions with `for<'a, T> fn() -> &'a T` signatures are more flexible and work in more scenarios than functions with `for<T> fn() -> &'static T` signatures



## Discuss

Discuss this article on
- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/gmrcrq/common_rust_lifetime_misconceptions/)
- [official Rust users forum](https://users.rust-lang.org/t/blog-post-common-rust-lifetime-misconceptions/42950)
- [Twitter](https://twitter.com/pretzelhammer/status/1263505856903163910)
- [rust subreddit](https://www.reddit.com/r/rust/comments/golrsx/common_rust_lifetime_misconceptions/)
- [Hackernews](https://news.ycombinator.com/item?id=23279731)



## Follow

[Follow pretzelhammer on Twitter](https://twitter.com/pretzelhammer) to get notified of future blog posts!



## Further Reading

[Learning Rust in 2020](./learning-rust-in-2020.md)
