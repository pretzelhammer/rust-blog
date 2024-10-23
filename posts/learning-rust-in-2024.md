# Learning Rust in 2024

_21 September 2024 · #rust · #coding · #exercises_

**Table of contents**
- [TL;DR](#tldr)
- [0\) Reference material](#0-reference-material)
- [1\) Read A half hour to learn Rust](#1-read-a-half-hour-to-learn-rust)
- [2\) Complete rustlings](#2-complete-rustlings)
- [3\) Spend 10 hours coding in Rust](#3-spend-10-hours-coding-in-rust)
  - [100 Exercises to Learn Rust](#100-exercises-to-learn-rust)
  - [The Rust track on Exercism](#the-rust-track-on-exercism)
  - [Advent of Code](#advent-of-code)
  - [Tutorials](#tutorials)
- [4\) Read Common Rust Lifetime Misconceptions](#4-read-common-rust-lifetime-misconceptions)
- [5\) Spend another 10 hours coding in Rust](#5-spend-another-10-hours-coding-in-rust)
- [6\) Read Tour of Rust's Standard Library Traits](#6-read-tour-of-rusts-standard-library-traits)
- [What's next?](#whats-next)
- [Honorable mentions](#honorable-mentions)
- [Discuss](#discuss)
- [Further Reading](#further-reading)

This is my opinionated guide on how to go from knowing nothing about Rust to being kinda okay at Rust quickly as possible. This is what I would tell myself if I was starting today.



## TL;DR

1. Read [A half hour to learn Rust](https://fasterthanli.me/articles/a-half-hour-to-learn-rust) (30 - 60 mins)
2. Complete [rustlings](https://github.com/rust-lang/rustlings) (2 - 3 hours)
3. Spend 10 hours coding in Rust (8 - 12 hours)
4. Read [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md) (30 - 60 mins)
5. Spend another 10 hours coding in Rust (8 - 12 hours)
6. Read [Tour of Rust's Standard Library Traits](./tour-of-rusts-standard-library-traits.md) (2 - 4 hours)

And that's it, after a short 19 to 30 hours you'll go from being a Rust beginner to a Rust advanced beginner 😎



## 0\) Reference material

While learning Rust it's useful to have [Rust by Example](https://doc.rust-lang.org/rust-by-example/index.html) and the [Rust Standard Library docs](https://doc.rust-lang.org/std/) open in a couple browser tabs. If you get stuck on something you can probably find the answer by searching in either of those two resources, as both have a search bar at the top if you need to quickly find something.

Other good alternatives are StackOverflow and ChatGPT (or whatever LLM you prefer). Almost every Rust beginner question has been asked and answered on StackOverflow and is indexed on Google. As for ChatGPT, it may get the answer wrong like half the time, but it will at least point you in the general direction of where you should search next to find a good answer.

If you've tried all of those things but still cannot get unstuck, then here are some beginner-friendly online Rust communities where you can ask questions: the [/r/rust](https://www.reddit.com/r/rust/) subreddit, the [/r/learnrust](https://www.reddit.com/r/learnrust/) subreddit, and the [official Rust users forum](https://users.rust-lang.org/).

Also it's a good idea to have [Rust Playground](https://play.rust-lang.org/) open in a browser tab. It's an online code editor and compiler for Rust code. It's great for tinkering with small examples to reinforce what you're learning while reading.



## 1\) Read [A half hour to learn Rust](https://fasterthanli.me/articles/a-half-hour-to-learn-rust)

I recommend [A half hour to learn Rust](https://fasterthanli.me/articles/a-half-hour-to-learn-rust) above all other resources similar to it because it introduces Rust syntax and concepts in a very systematic and ground-up approach. It starts with the simplest possible example and then incrementally adds a little bit more complexity in every following example. The sequencing is very logical and easy to follow.



## 2\) Complete [rustlings](https://github.com/rust-lang/rustlings)

[Rustlings](https://github.com/rust-lang/rustlings) is an ordered collection of tiny Rust programming exercises. Each exercise is a tiny Rust program that doesn't compile or is failing its unit tests. Your job for every exercise is to fix the program so it compiles and passes its unit tests.

Before starting rustlings you need to have Rust installed. I recommend using [rustup](https://rustup.rs/) to manage Rust installations. Once you have `rustup` you just have to run `rustup update` to update Rust on your machine any time.

When coding Rust, you can use whatever code editor you want, but I recommend using one that supports [rust-analyzer](https://rust-analyzer.github.io/). I use [VS Code](https://code.visualstudio.com/) with [this extension](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust-analyzer) to enable rust-analyzer.

Getting started with rustlings is very easy because we can install it using `cargo`. Cargo is Rust's dependency manager and build tool, and comes with all Rust installations. If you installed Rust using `rustup` then you already have it.

Run `cargo install rustlings`. By default, it will put the rustlings binary in `$HOME/.cargo/bin`, so you should add that path to your `$PATH` variable if it's not in there already, e.g.

```sh
export PATH="$HOME/.cargo/bin:$PATH"
```

Now run `rustlings`, which will create a `rustlings/` directory for you with all of the exercises in it. Open that directory in a code editor and then run the `rustlings` command within the directory and follow its instructions. It will tell you which exercise to work on. Complete the exercise and then the command will tell which exercise to work on next. Rinse and repeat until you're done.



## 3\) Spend 10 hours coding in Rust

This can be whatever you want, but for your first 10 hours I would suggest sticking to single-threaded sync Rust. There's enough novel and challenging Rust concepts\* to learn within those parameters to easily keep you busy for at least 10 hours, and many people take much longer to fully wrap their heads around them. Multithreaded async Rust adds a bunch of additional complex concepts\*\*, so starting with it before you have the basics down will likely be overwhelming.

\*: ownership, borrowing, lifetimes, lifetime elision, mutable vs immutable references, sizedness, deref coercion, traits, trait objects, generics, const generics, static vs dynamic dispatch, error handling, iterators, modules, declarative macros, procedural macros, just to name a few.

\*\*: futures, pinning, polling, awaiting, send, sync, cancellation, streams, sinks, locks, atomics, channels, actors, just to name a few.

If you have no ideas on what to code then I have a few suggestions.



### [100 Exercises to Learn Rust](https://rust-exercises.com/100-exercises/)

If you enjoyed the difficulty and pacing of rustlings then you will enjoy 100 Exercises to Learn Rust (100ELR). Although made by different authors, 100ELR feels like a natural extension of rustlings and clearly drew a lot of inspiration from it.

To get started clone the [mainmatter/100-exercises-to-learn-rust](https://github.com/mainmatter/100-exercises-to-learn-rust) repo to your machine. Install their test suite runner using `cargo install --locked workshop-runner`. Then start reading the [accompanying book](https://rust-exercises.com/100-exercises/). It has 100 chapters and at the end of each chapter it tells you what exercise to do in the repo. Once you complete an exercise run `wr` in the project root to check that your solution passes the unit tests, and if it does continue on to the next chapter. Rinse and repeat until you're done.

Although 100 chapters sounds like a lot they're all quite short so you'll get through them faster than you think. Completing the entire book likely take most people about 15 to 25 hours.

However, if you found rustlings too easy or too slow and would like to try something more challenging, then you may want to skip 100ELR.



### The [Rust track](https://exercism.org/tracks/rust) on Exercism

The [Rust track](https://exercism.org/tracks/rust) on Exercism is the best collection of small Rust exercises on the internet aside from rustlings and 100ELR. What makes it so good is that many of the exercises were crafted by Rustaceans, and are meant to teach you the aspects of Rust that make it unique and interesting compared to other programming languages. There's an exercise where you have to write a declarative macro! There's even exercise where you have to write unsafe code!

To get started you have to make an account on Exercism and enroll in the Rust track (both are free). You can then complete the exercises online using Exercism's online code editor or by downloading the exercises to your machine, completing them using your code editor, and submitting your solutions to Exercism using the `exercism` cli tool.

I recommend using the `exercism` cli tool. There are many advantages. First, your code editor has rust-analyzer and Exercism's online code editor doesn't. Second, you can use dark mode in your editor but you can't use dark mode on Exercism unless you pay (it's a premium feature). Third, you can track your solutions using git. Fourth, you can choose which unit tests to run while you're developing your solution, instead of running the full test suite every time. The official Exercism instructions on how to do the exercises locally on your machine are [here](https://exercism.org/docs/using/solving-exercises/working-locally).

Each track on Exercism has a "learning mode" and a "practice mode." At the moment the Rust track's "learning mode" is disabled while the Exercism team reworks it. Regardless, I think the "practice mode" is better because it lets you do the exercises in whatever order you want. So while you may not have the option right now, if the option between a "learning mode" and a "practice mode" comes back to the Rust track in the future I'd recommend picking practice mode.

While the exercises in the Rust track are ordered, the order is arbitrary and following it isn't important. With the exception of the [Luhn](https://exercism.org/tracks/rust/exercises/luhn), [Luhn From](https://exercism.org/tracks/rust/exercises/luhn-from), and [Luhn Trait](https://exercism.org/tracks/rust/exercises/luhn-trait) exercises, all of the other exercises are totally unrelated to each other and can be done in whatever order.

The best part of completing an exercise is that it gives you the ability to see and star other people's solutions. After completing an exercise I recommend checking these out, especially the most starred solutions, as they can teach you elegant and idiomatic parts of Rust that you may have not discovered on your own.

Furthermore, the exercises are rated by difficulty: easy, medium, or hard. Currently there's 99 total exercises in the Rust track and I would say that about 30 are really good and the other 59 are just generic exercises that were copy & pasted from other tracks. Less than 0.1% of people complete the entire track. I wouldn't be surprised if less than 1% of people even complete half of the track. I have done all of the exercises so I'll recommend which ones I think are the best, since you're likely not going to do all of them anyway.

You should start with the easy exercises. Difficulty-wise they're slightly more challenging than what you'd find in rustlings or 100ELR. Quality-wise all of them are okay. None of them will take you longer than 20 minutes to complete. Also, if completing the easy exercises stops being rewarding then move on to the medium exercises, don't push yourself to complete all the easy ones if you don't feel like you're getting anything out of them anymore.

Anyway, as for the medium exercises, these are the best ones in my opinion (in no particular order):
- [PaaS I/O](https://exercism.org/tracks/rust/exercises/paasio)
- [Simple Linked List](https://exercism.org/tracks/rust/exercises/simple-linked-list)
- [Fizzy](https://exercism.org/tracks/rust/exercises/fizzy)
- [Parallel Letter Frequency](https://exercism.org/tracks/rust/exercises/parallel-letter-frequency)
- [Macros](https://exercism.org/tracks/rust/exercises/macros)
- [Robot Name](https://exercism.org/tracks/rust/exercises/robot-name)
- [Triangle](https://exercism.org/tracks/rust/exercises/triangle)
- [Robot Simulator](https://exercism.org/tracks/rust/exercises/robot-simulator)
- [Accumulate](https://exercism.org/tracks/rust/exercises/accumulate)
- [Word Count](https://exercism.org/tracks/rust/exercises/word-count)
- [Grep](https://exercism.org/tracks/rust/exercises/grep)
- [Luhn](https://exercism.org/tracks/rust/exercises/luhn)
- [Luhn From](https://exercism.org/tracks/rust/exercises/luhn-from)
- [Luhn Trait](https://exercism.org/tracks/rust/exercises/luhn-trait)
- [Clock](https://exercism.org/tracks/rust/exercises/clock)
- [Space Age](https://exercism.org/tracks/rust/exercises/space-age)
- [Sublist](https://exercism.org/tracks/rust/exercises/sublist)
- [Binary Search](https://exercism.org/tracks/rust/exercises/binary-search)
- [ETL](https://exercism.org/tracks/rust/exercises/etl)
- [Grade School](https://exercism.org/tracks/rust/exercises/grade-school)
- [Hamming](https://exercism.org/tracks/rust/exercises/hamming)
- [Isogram](https://exercism.org/tracks/rust/exercises/isogram)
- [Nucleotide Count](https://exercism.org/tracks/rust/exercises/nucleotide-count)

Depending on which ones you do they can take you anywhere from 10 to 45 minutes.

Once you're ready to bump up the difficulty again, these are the best hard exercises in my opinion (in no particular order):
- [Xorcism](https://exercism.org/tracks/rust/exercises/xorcism)
- [React](https://exercism.org/tracks/rust/exercises/react)
- [Circular Buffer](https://exercism.org/tracks/rust/exercises/circular-buffer)
- [Forth](https://exercism.org/tracks/rust/exercises/forth)
- [Doubly Linked List](https://exercism.org/tracks/rust/exercises/doubly-linked-list)

Depending on which ones you do they can take anywhere from 30 to 90 minutes.

If you do a bunch of easy exercises, plus the medium and hard ones I recommend, it will probably take you around 20 hours. I think completing the entire track will probably be double that, at around 40 hours. With that said don't push yourself to finish the entire track if you feel yourself starting to lose interest, it'd be better to change gears and continue your Rust coding journey trying something else next, because the most important thing is that you keep having fun while learning.



### [Advent of Code](https://adventofcode.com/)

[Advent of Code](https://adventofcode.com/) (AoC) is a collection of programming puzzles. Every year since 2015 a new puzzle has been released every day from December 1st to 25th. Each puzzle has 2 parts. The first 10-15 puzzles of each year are usually easy to medium difficulty, and the last 10-15 puzzles of each year are usually hard. The puzzles are designed to be language agnostic, meaning you can solve them using any programming language. So if your goal is to learn Rust then just doing the puzzles is not going to help you very much, but if you complete a puzzle and then review an experienced Rustacean's solution that could help teach you a lot of Rust.

The experienced Rustacean in this case would be fasterthanlime. On top of publishing his solutions for the 2020 and 2022 AoC puzzles he also wrote beginner-friendly blog posts explaining his thought process while solving each one. If you're going to do AoC to learn Rust then you should start with the 2020 puzzles and after completing each one read the corresponding fasterthanlime blog post on it. I wouldn't recommend going far past puzzle 10 for any given year because they can start to get really challenging and you'll be spending most of your time busting your brain trying to figure out how to solve the puzzle instead of learning Rust.



#### Setting up

First, create an account on [AoC](https://adventofcode.com/). Second, since AoC is language agnostic you have to set up your own cargo project with your own scaffolding in order to solve the puzzles. Doing this can be very educational but it's also pretty boring. I'd suggest skipping that and using this Github template instead: [fspoettel/advent-of-code-rust](https://github.com/fspoettel/advent-of-code-rust). Click on `Use this template` -> `Create a new repository`, then `git clone` the created repository to your machine. In `.cargo/config.toml` set `AOC_YEAR` to `2020`.

Also you can install the `aoc-cli` tool to fetch all instructions and inputs from your AoC account to your machine. To do that run `cargo install aoc-cli --version 0.12.0`, then create the file `$HOME/.adventofcode.session` and paste your AoC session cookie into it. To get your session cookie, press `F12` while anywhere on the AoC website to open your browser's developer tools, then look for `Cookies` under the `Application` or `Storage` tabs and copy the `session` cookie value.

Once all of that is done you can run `cargo download {day}` within your project directory to generate the scaffolding and download the puzzle for that day. Once you think you've solved it you can run `cargo solve {day}` and then submit your final result via the AoC website to verify. For day 1 the commands would be `cargo download 1` and `cargo solve 1`.



#### Year 2020

As mentioned before, make sure to set `AOC_YEAR` to `2020` in `.cargo/config.toml`. After completing a puzzle read fasterthanlime's solution on his blog, here's direct links to his solutions to the first 10 puzzles:

1. [AoC 2020 Day 1](https://fasterthanli.me/series/advent-of-code-2020/part-1)
1. [AoC 2020 Day 2](https://fasterthanli.me/series/advent-of-code-2020/part-2)
1. [AoC 2020 Day 3](https://fasterthanli.me/series/advent-of-code-2020/part-3)
1. [Aoc 2020 Day 4](https://fasterthanli.me/series/advent-of-code-2020/part-4)
1. [AoC 2020 Day 5](https://fasterthanli.me/series/advent-of-code-2020/part-5)
1. [AoC 2020 Day 6](https://fasterthanli.me/series/advent-of-code-2020/part-6)
1. [AoC 2020 Day 7](https://fasterthanli.me/series/advent-of-code-2020/part-7)
1. [AoC 2020 Day 8](https://fasterthanli.me/series/advent-of-code-2020/part-8)
1. [AoC 2020 Day 9](https://fasterthanli.me/series/advent-of-code-2020/part-9)
1. [AoC 2020 Day 10](https://fasterthanli.me/series/advent-of-code-2020/part-10)



#### Year 2022

If you completed at least 10 of the 2020 puzzles then year 2022 is the next best year to do. Clone another copy of [fspoettel/advent-of-code-rust](https://github.com/fspoettel/advent-of-code-rust) and set `AOC_YEAR` to `2022` in `.cargo/config.toml` in the project directory. Then here's direct links to fasterthanlime's 2022 solutions:

1. [AoC 2022 Day 1](https://fasterthanli.me/series/advent-of-code-2022/part-1)
1. [AoC 2022 Day 2](https://fasterthanli.me/series/advent-of-code-2022/part-2)
1. [AoC 2022 Day 3](https://fasterthanli.me/series/advent-of-code-2022/part-3)
1. [AoC 2022 Day 4](https://fasterthanli.me/series/advent-of-code-2022/part-4)
1. [AoC 2022 Day 5](https://fasterthanli.me/series/advent-of-code-2022/part-5)
1. [AoC 2022 Day 6](https://fasterthanli.me/series/advent-of-code-2022/part-6)
1. [AoC 2022 Day 7](https://fasterthanli.me/series/advent-of-code-2022/part-7)
1. [AoC 2022 Day 8](https://fasterthanli.me/series/advent-of-code-2022/part-8)
1. [AoC 2022 Day 9](https://fasterthanli.me/series/advent-of-code-2022/part-9)
1. [AoC 2022 Day 10](https://fasterthanli.me/series/advent-of-code-2022/part-10)



### Tutorials

The goal of most Rust tutorials is to show you how some piece of software could be implemented in Rust, not to necessarily to teach you Rust, and so many tutorial authors assume their audience will already be somewhat competent with the language and will omit a bunch of beginner explanations for the sake of keeping the tutorial focused and concise.

With that said, writing a non-trivial piece of software can be a lot more fun and rewarding than completing a bunch of small exercises, so this approach can be preferable for keeping your motivations high, especially if you're building something you're genuinely interested and excited about.

I wish I could recommend some tutorials here but there's so many of them out there that it would be hard for me to go through all of them and rank them by their quality. The quality probably isn't that important though, what's important is that you're writing Rust code and enjoying your time, so pick any tutorial that looks fun and follow along without sweating the details.

I will give a shoutout to [Codecrafters](https://codecrafters.io) though. Their tutorials are clearly very high quality and they have Rust starter templates for all of them. In every step of every tutorial they give you instructions and hints on how to implement the next part of the program, and once you're done you submit your code and it runs against a test suite to check if everything was implemented correctly.

A big downside of Codecrafters is that it costs $40/month. They occasionally run promotions where a tutorial will be free for a month, so depending on when you check their site you may get lucky and a tutorial that catches your eye will be free. However, their pricing suggests they target software companies looking to train their engineers, and not individual engineers themselves. If you're employed at such a company, check if your company provides you a discretionary training allowance for courses like Codecrafters, which you could use to cover the cost.



## 4\) Read [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md)

Disclaimer: I wrote this.

Reading [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md) will help you understand why half of the stuff you tried in your last 10 hours of Rust coding didn't work 🙂

Jokes aside, understanding lifetimes is probably the biggest stumbling block that many beginners struggle to get over when learning Rust, and this article dispels all of the most common misconceptions that people have about lifetimes that cause them confusion and frustration.

The reason I recommend it after at least 10 hours of practical Rust coding experience is because I think it might be too much technical information too soon for absolute beginners, who may find it more overwhelming than helpful.



## 5\) Spend another 10 hours coding in Rust

As before if you're not sure what to code my suggestions remain the same: [100 Exercises to Learn Rust](https://rust-exercises.com/100-exercises/), the [Rust track](https://exercism.org/tracks/rust) on Exercism, [Advent of Code](https://adventofcode.com/), or tutorials.

Also since you now have some Rust experience under your belt feel free to try coding something in multithreaded async Rust if you're feeling adventurous.



## 6\) Read [Tour of Rust's Standard Library Traits](./tour-of-rusts-standard-library-traits.md)

Disclaimer: I wrote this.

Traits are the main way to write polymorphic code in Rust, so they're used everywhere, especially the ones from the standard library. This article gives a guided tour of the most popular standard library traits, their methods, how to use them, and when to implement them for your own types. It's pretty thorough, and shares a ton of useful tips. And although the article is lengthy you don't have to read the entire thing to get value out of it, so don't let its size daunt you.



## What's next?

Congrats. You made it. You definitely know enough Rust to forge your own path forward. Good luck and have fun.



## Honorable mentions

These are other Rust beginner resources that I really like but wasn't able to find a spot for them in my Rust learning guide.



### [Tour of Rust](https://tourofrust.com/)

It's really good. However, it covers a lot of the same ground that other resources in the guide already cover, so it was omitted for the sake of keeping the guide streamlined.



### [Comprehensive Rust](https://google.github.io/comprehensive-rust/)

This is a Rust course developed by the Android team at Google. It's a collection of slides which are meant to be presented by a speaker experienced with Rust, but the book version has the speaker notes at the bottom of each page so that it can be learned from as a standalone resource.

It's very concise and fast-paced, and also has sections covering parts of Rust that are typically omitted from other beginner resources, such as [bare metal Rust](https://google.github.io/comprehensive-rust/bare-metal.html) and [concurrency in Rust](https://google.github.io/comprehensive-rust/concurrency/welcome.html). However, since it's better presented by a speaker it didn't make sense to put it into my guide for self-learners.



### [Clear explanation of Rust's module system](https://www.sheshbabu.com/posts/rust-module-system/)

This is a great article. I wasn't sure where to put it in the guide because it could go anywhere, but ideally it comes right when the learner starts breaking their Rust code across multiple files, which comes at a different point for every person in their Rust coding journey. Anyway, whenever that point comes for you, read this article then.



## Discuss

Discuss this article on
- [Github](https://github.com/pretzelhammer/rust-blog/discussions/84)
- [official Rust users forum](https://users.rust-lang.org/t/learning-rust-in-2024)
- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/1fnlvd8/learning_rust_in_2024/)
- [rust subreddit](https://www.reddit.com/r/rust/comments/1fod8u9/learning_rust_in_2024/)


## Further reading

- [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md)
- [Tour of Rust's Standard Library Traits](./tour-of-rusts-standard-library-traits.md)
- [Beginner's Guide to Concurrent Programming: Coding a Multithreaded Chat Server using Tokio](./chat-server.md)
- [Using Rust in Non-Rust Servers to Improve Performance](./rust-in-non-rust-servers.md)
- [Sizedness in Rust](./sizedness-in-rust.md)
- [RESTful API in Sync & Async Rust](./restful-api-in-sync-and-async-rust.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](./too-many-brainfuck-compilers.md)
