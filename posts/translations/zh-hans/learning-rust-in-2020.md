# Learning Rust in 2020

_2020年 5月 9日 · #rust · #programming · #exercises_

**目录**
- [前言](#前言)
- [省流](#省流)
- [Practical Rust Resource Reviews](#practical-rust-resource-reviews)
    - [HackerRank](#hackerrank)
    - [Project Euler](#project-euler)
    - [LeetCode](#leetcode)
    - [Codewars](#codewars)
    - [Advent of Code](#advent-of-code)
    - [Rustlings](#rustlings)
    - [Exercism](#exercism)
- [Conclusion](#conclusion)
- [Discuss](#discuss)
- [Further Reading](#further-reading)



## 前言

我开始学习 Rust 时犯了一个错误：
完全按照 [Rust 程序设计语言](https://rustwiki.org/zh-CN/book/) 的指导学习。
虽然这本书确实不错，但是跟一个初学者讲“先啃下这本只有 20 章的小书”还是太劝退了，
很多人因此刚入门就入土了。可没人跟 Javascript 或 Python 初学者说什么
“先啃下这本只有 20 章的小书”。
Rust 确实很难入门，但是比起读20章的“小书”，大家更希望先试试。
编程很有趣，但是读书很无聊，尤其是20章的“小书”。

本文的前 10% 是一点建议，帮助你在 2020 年通过 *练习* 上手 Rust。
读完之后你就可以退出了（下文会告诉你在哪里退出）。
之后是关于各大 OJ 平台对 Rust 支持程度的意见和个人排名。



## 省流

如果你之前完全没了解过Rust，想要尽可能多学点，
可以读 fasterthanlime 的[《A half-hour to learn Rust》](https://fasterthanli.me/blog/2020/a-half-hour-to-learn-rust/)，
然后看看 [Rustlings](https://github.com/rust-lang/rustlings) 并完成练习。

如果你刚开始学习，应该去看看 [Exercism.io 的 Rust 练习](https://exercism.io/tracks/rust)。
如果遇到了困难，可以问问 Google 或者 StatckOverflow。
建议花点儿时间简单看看 [Rust 标准库文档](https://doc.rust-lang.org/std),
里面有很多简单的例子，帮助你更好地学习标准库。
[《Rust by Example》](https://doc.rust-lang.org/rust-by-example/) 
也很好地阐述了 Rust 的语法和特性。
如果想要深入理解 Rust 的某个概念，
可以在[Rust 程序设计语言](https://rustwiki.org/zh-CN/book/)中检索相关章节。
Exercism.io 的好处是可以看到其他人的解法，并通过 Star 数找到优秀的题解加以学习，
这个方法很好！

现在，你已经初步进入了 Rust 抽象之塔的大门，并且可以自己继续前进。
如果想尝试编写一些小程序，需要更多指导，建议完成 [Advent of Code 2018 Calendar](https://adventofcode.com/2018) 
推荐 2018 Calendar 的原因是完成之后可以参考 [BurntSushi 的 Advent of Code 2018 Calendar Rust 题解](https://github.com/BurntSushi/advent-of-code)。
他的代码简洁干净，易于阅读。
阅读优秀的 Rust 代码可以让你学到和自己写一遍不相上下的经验。

可以退出了。

（译注：中文学习者可以考虑 [Rust 圣经](https://course.rs/about-book.html) ）



## Rust 练习平台比较

_又名：对供 Rust 初学者练习编写简单代码的免费在线资源的比较_

以下大部分平台都不是专门为 Rust 搭建的，但是仍然可以用于学习 Rust，
且大部分平台明确地支持 Rust 并提供题目的 Rust 变体。

以下平台从差到好排序。



### [HackerRank](https://www.hackerrank.com)

支持 Rust，但大部分题目不支持 Rust 题解。
尝试提交题解，被当场拒绝：

![建 议 改 为： FailRank](../assets/hackerrank-more-like-failrank.png)

6。可以看到别人的，但是不能上传自己的。
Google 了一圈，未能发现有用的信息，建议直接放弃。


### [Project Euler](https://projecteuler.net/archives)

当年我开始学习编程时（2012年），我经常听到“快速学编程，上Project Euler！”之类的话。
当年确实没多少选择，但我认为 Projet Euler 更侧重于数学，而不是编程。
不建议，除非数学很好。



### [LeetCode](https://leetcode.com/problemset/all/)

LeetCode 支持 Rust。
在 LeetCode 上，每题有一个答案模板，包含一个待实现函数，
需要实现所有函数，然后提交。
更复杂的题目可能会包含一个 `struct` 和一个 `impl`，包含几个待实现方法。
可惜，这些模板由机器生成，而不是人类。这导致这些模板良莠不齐，比如：

| LeetCode 生成的 Rust | 一般的 Rust |
|-|-|
| 树相关问题用 `Option<Rc<RefCell<Node>>>` 表示边。 |  `Option<Box<Node>>` 更简单好用。 |
| 借用可变性错误： `fn insert(&self, val: i32)` | 应用可变借用： `fn insert(&mut self, val: i32)` |
| 总是使用有符号数，即使题目不关心负数。 `fn nth_fib(n: i32) -> i32` | 应用无符号数： `fn nth_fib(n: u32) -> u32` |
| 总是吃掉参数的所有权： `fn sum(nums: Vec<i32>) -> i32` | 只需要借用就好： `fn sum(nums: &[i32]) -> i32` |
| 忽略特殊情况：`nums` 为空时 `fn get_max(nums: Vec<i32>) -> i32` 的返回值未定义 | 应用 `Option` 包裹返回值： `fn get_max(nums: &[i32]) -> Option<i32>` |

Rust 特供槽点：
- 禁止引入依赖，即使是像 Rust 这样标准库里连正则表达式都没有的语言；
- 禁止向 `concurrency` 分区里的题提交题解。众所周知，安全高效的并发是 Rust 的一大亮点。
- 很少有 Rust 题解。（因为过于小众？）

统一槽点：
- 有很多低质量题目。用户可以为题解点赞或点踩，但题目不会被删除。
  有这么一道题，有 100 多个评价，八成都是踩，却像在数据库里长了根一样。
- 题目难度不准。有三等：Easy（简单）、Medium（中等）和 Hard（困难），
  但是很多 Easy 题 AC 率比 Hard 题还低。
- 不是所有题都支持用所有语言提交，且无法通过支持的语言筛选题目。
  比如，所有图论题都不支持 Rust。
- 需要按月付费才能查看“premium”题，但是没有任何试用期，
  所以没有办法提前判断是否有必要交钱。

优点：
- 可以查看 WA 测试点。
- 所有 Rust 代码模板均符合 rustfmt 的要求。



### [Codewars](https://www.codewars.com/join?language=rust)

Codewars 这个名字容易让人误解。Codewars 上没有什么 war（*n.* 战争）。
答案不会 TLE，也不会用时间或内存用量排名。人与人之间没有竞争——这并不是什么坏事。

Codewars 支持 Rust。所有题目都提供一个答案模板，通常包含一个待实现函数。
答案模板是人工编写的，但编写模板的人不一定熟悉 Rust，
所以时不时会冒出一些没那么好的模板，例如：

| Codewars 的部分 Rust 答案模板 | 一般的 Rust |
|-|-|
| 不遵守 rustfmt 要求： `fn makeUppercase(s:&str)->String` | 用 rustfmt 格式化：`fn make_uppercase(s: &str) -> String` |
| 使用有符号数，即使题目不关心负数。 `fn nth_fib(n: i32) -> i32` | 应用无符号数： `fn nth_fib(n: u32) -> u32` |
| 用 `-1` 表示未找到：`fn get_index(needle: i32, haystack: &[i32]) -> i32` | 应用 `Option::None`：`fn get_index(needle: i32, haystack: &[i32]) -> Option<usize>` |
| 使用一般的引用：`fn do_stuff(s: &String, list: &Vec<i32>)` | 使用切片：`fn do_stuff(s: &str, list: &[i32])` |

因为 Codewars 上编写模板的人水平良莠不齐，所以时不时会有以上问题，至少比 LeetCode 好多了（后者是“时不时没有问题”）。
但是，写 Rust 的 Codewars 用户普遍经验不足，时不时会有质量较低的高赞题解，例如：

| Codewars 上题目的高赞 Rust 题解 | 一般的 Rust |
|-|-|
| 在代码块最后使用 `return result;` | `return` 和分号可省略：`result` |
| 通过删减空格使题解显得更加“简洁” | 遵守 rustfmt 规范 |
| 无用内存分配：`str_slice.to_string().chars()` | 不分配：`str_slice.chars()` |
| 迭代器上瘾 | 当迭代器太多时应该重构 |

重复：只是“时不时”。有经验的 Rustacean 可以一眼看穿以上问题，但有很多 Rust 新手不能。

Rust 特供槽点：
- 在 Codewars 上 Rust 还很小众，一共九千多题，只有三百多题有 Rust 版本 :(

统一槽点：
- Your solution is tested against a suite of secret unit tests, if you fail one of the secret unit tests you aren't shown the failed test case. This is especially annoying if the test case tests for an edge case that wasn't clearly communicated in the problem description.

Things Codewars does right:
- There's a small whitelist of 3rd-party dependencies you can use to help solve problems with Rust. This whitelist includes: rand, chrono, regex, serde, itertools, and lazy_static which helps round out Rust's standard library and puts it more on par with other languages.
- You can filter problems by language.
- Submitting a solution to a problem also automatically publishes the solution. You can view and upvote other members' solutions. You can sort solutions by most upvotes to see particularly concise and clever solutions, which sometimes will also be very idiomatic (but sometimes not, as explained above).
- Problem difficulty grading is pretty good! Instead of grading problems as Easy, Medium, or Hard like LeetCode, Codewars chooses to grade problems from easiest to hardest as: 8 kyu, 7 kyu, 6 kyu, 5 kyu, 4 kyu, 3 kyu, 2 kyu, 1 kyu. I completed 60 problems in the 8 kyu - 4 kyu range and every level felt a little more difficult than the last, which aligned with my expectations.



### [Advent of Code](https://adventofcode.com/)

Advent of Code is totally language-agnostic. This would seem like a minus at first but seeing how horribly HackerRank, LeetCode, and Codewars handle their support for Rust on their sites it's actually a plus. Advent of Code also gets placed above the previously mentioned sites because AoC's exercises are really interesting, diverse, and high quality in my opinion.

General AoC issues:
- After you finish an exercise there's no way to see other people's Rust solutions unless you search from them on Google, and even after you find some there's no telling how good or idiomatic they are.

To solve the above issue I recommend going through the 2018 Calendar problems and comparing your solutions to [BurntSushi's AoC 2018 Rust solutions](https://github.com/BurntSushi/advent-of-code). BurntSushi writes really clean, readable, idiomatic Rust code. If you want to go through the 2019 Calendar then I recommend comparing your solutions to [bcmyers' AoC 2019 Rust solutions](https://github.com/bcmyers/aoc2019). The reason I specifically suggest bcmyers' is because he made a [youtube playlist of him coding up the solutions](https://www.youtube.com/playlist?list=PLQXBtq4j4Ozkx3r4eoMstdkkOG98qpBfg) and he does a great job of explaining his thought process and why he's doing what he's doing while he's coding.

Things AoC got right:
- High quality, interesting, curated exercises that are tied together with a narrative.
- Language agnostic, so while it doesn't teach you any Rust patterns it at least doesn't teach you any Rust anti-patterns either.



### [Rustlings](https://github.com/rust-lang/rustlings)

Rustlings is sooo good. All Rustlings exercises are hand-crafted for Rust with love and it's a wonderful breath of fresh air. Finally, a set of exercises that really teach you idiomatic Rust!

If you're a total Rust newbie you should absolutely checkout [Rustlings](https://github.com/rust-lang/rustlings) and get started on the exercises. I highly recommend reading fasterthanlime's [A half-hour to learn Rust](https://fasterthanli.me/blog/2020/a-half-hour-to-learn-rust/) first as it'll get you up to speed on a lot of Rust syntax and concepts super quickly.

I have only 1 tiny Rustlings criticism: there are some sudden difficulty spikes in the "error-handling" and "conversions" exercises that I could see some users getting overwhelmed by. I assume most probably make it through, or at least I hope.

I also have 1 tiny non-criticism: it's too short. This is a non-criticism because it's one of Rustlings design goals to be a quick and gentle introduction to Rust but it's so good that of course I wish it was somehow longer.



### [Exercism](https://exercism.io/tracks/rust)

Exercism has a Rust track, which is a collection of exercises roughly ordered by subject and difficulty. The Rust track shares a lot of exercises in common with other tracks, but all of the exercises were translated to Rust by experienced Rustaceans and don't suffer from any of the awkward unidiomatic Rust issues that are common on LeetCode and Codewars. There are about a dozen Rust-specific problems that require you to implement a standard library trait, or write a macro, or write a parallel solution using multiple threads, or write unsafe Rust code. These exercises are by far the highlights of the track and I wish there were more of them. Exercism is second only to Rustlings as a resource for learning Rust. The only reason I placed it above Rustlings is Rustlings can be completed in an evening and Exercism's Rust track will take at least a month to complete so it just has a lot more content.

Exercism issues, specific to the Rust track:
- "Mentored mode" is useless, as most of the Rust mentors on the site are inactive, and the students heavily outnumber them, so it's much better to go through a track in "practice mode".
- There are 92 exercises but a good chunk of them don't really teach you anything new so they kinda feel like busywork. They could probably cut ~20 exercises from the track to make it feel a lot tighter.

Things Exercism does right:
- All problems are translated to Rust or written for Rust by experienced Rustaceans.
- There are problems which specifically teach Rust's idioms, design patterns, and unique features.
- Problem difficulties are fairly graded, easy problems are easy, medium problems are medium, hard problems are hard.
- You can include whatever 3rd-party dependencies that you want in your solutions.
- All unit tests are public, if you're failing a test you know exactly why.
- After you submit a solution you can browse other user's solutions, and you can sort solutions by which received the most stars.



## Conclusion

Same as the [TL;DR](#tldr) :)



## Discuss

Discuss this article on
- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/ggj8tf/learning_rust_in_2020/)
- [official Rust users forum](https://users.rust-lang.org/t/blog-post-learning-rust-in-2020/42373)
- [rust subreddit](https://www.reddit.com/r/rust/comments/gie64f/learning_rust_in_2020/)
- [Hackernews](https://news.ycombinator.com/item?id=23160975)
- [Github](https://github.com/pretzelhammer/rust-blog/discussions)


## Further Reading

- [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md)
- [Tour of Rust's Standard Library Traits](./tour-of-rusts-standard-library-traits.md)
- [Sizedness in Rust](./sizedness-in-rust.md)
- [RESTful API in Sync & Async Rust](./restful-api-in-sync-and-async-rust.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](./too-many-brainfuck-compilers.md)
