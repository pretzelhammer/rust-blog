# 2020 年的 Rust 学习指南

_2020年 5月 9日 · #rust · #programming · #exercises_

**目录**
- [前言](#前言)
- [省流](#省流)
- [Rust 练习平台比较](#practical-rust-resource-reviews)
    - [HackerRank](#hackerrank)
    - [Project Euler](#project-euler)
    - [LeetCode](#leetcode)
    - [Codewars](#codewars)
    - [Advent of Code](#advent-of-code)
    - [Rustlings](#rustlings)
    - [Exercism](#exercism)
- [结论](#conclusion)
- [讨论](#discuss)
- [参阅](#further-reading)



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
- 无法看到 WA 测试点。如果这个测试点刚好是一个边界情况，题目又没有讲清楚，就很烦人了。

优点：
- 可以使用以下第三方库解 Rust 题目：rand、chrono、regex、serde、itertools 和 lazy_static，使 Rust 的功能赶上其他语言。
- 可以按语言筛选题目。
- 答案通过时会自动发布为题解。可以查看他人的题解。可以按赞数排序题解，方便看简洁聪明（而且很多时候也很优雅）的题解。
- 问题难度的设计十分优雅！与 LeetCode 不同，Codewars 用 8 kyu 到 1 kyu 从简单到困难排序。我完成了很多 8～4 kyu 的题，每一 kyu 的题都刚好比上一 kyu 难一点，符合我的预期。



### [Advent of Code](https://adventofcode.com/)

Advent of Code 的题目语言无关。也许刚开始这像一个扣分项，
但是和前面几个做一下对比，你就知道这多么先进了。
Advent of Code 的题目也得到了上面一些平台的收录，因为它们确实又有趣有优秀。

槽点：
- 不提供题解，想看得自己搜，搜了也不知道好不好。

建议完成 2018 Calendar 然后把答案和 [BurntSushi 的](https://github.com/BurntSushi/advent-of-code)
做对比。BurntSushi 的代码干净整洁，可读性强。
如果想做 2019 Calendar，可以用 [bcmyers 的答案](https://github.com/bcmyers/aoc2019)
做参考。
推荐他的原因是他做了 [一系列关于这些题目解法的视频](https://www.youtube.com/playlist?list=PLQXBtq4j4Ozkx3r4eoMstdkkOG98qpBfg)，
而且讲得很好。

优点：
- 高质量、有趣且精心策划的题目；
- 语言无关，所以虽然不会有高质量代码，但至少不会冒出来不规范的代码。



### [Rustlings](https://github.com/rust-lang/rustlings)

Rustlings 非常好。Rustling 上的所有题目都专门为 Rust 所编写。
而且，它真的是奔着教会你优雅的代码去的！

如果你是一个完全的 Rust 萌新，你肯定要做一做 [Rustlings](https://github.com/rust-lang/rustlings)
上的练习。极力推荐 [fasterthanlime 的 《A half hour to learn Rust》](https://fasterthanli.me/blog/2020/a-half-hour-to-learn-rust)，
可以帮你快速了解很多 Rust 的语法和概念。

只有一个小缺陷：在“error-handling”和“conversions”几节难度忽然飙升，
让很多人不知所措。希望大家都能挺过去。

也有一个小建议（不是批评）：太短了。之所以不是批评，是因为 Rustlings 的目标就是
快速帮人入门 Rust。但是它真的写得太好了，希望多来点儿。



### [Exercism](https://exercism.io/tracks/rust)

Exercism 有一个 “Rust track”，就是一列题目，大致按难度排序。
Rust track 和其他 tracks 共享很多题目，但是所有题目均由专业人员译为 Rust 题目，
不会出现像 LeetCode 和 Codewars 那样的问题。
有十几道题需要实现标准库 trait，写几个宏，写点多线程并发，或者 `unsafe {}`。
这些题目是这个 track 的亮点，希望多来点。
Exercism 仅次于 Rustlings。将它排在 Rustlings 后面的唯一原因是 Rustlings 一晚上就可以完成，
但 Exercism 的 Rust track 需要至少一个月。

Rust 特供槽点：
- 因为很多指导员不活跃，“指导模式”基本没用，不如用“练习模式”。
- 大约有 92 道题，但其中有很多题并没有新东西，像在做无用功，不如删掉二十几题。

优点：
- 所有题目均有专业人士译为 Rust 题目。
- 有专门讲 Rust 引入的新东西的题目。
- 题目分级准确。
- 可以随意引入第三方库。
- 所有测试点均公开，可以知道错在哪了；
- 可以浏览其他用户的题解，并按 star 数排序。



## 结论

同 [TL;DR](#tldr) :)



## 讨论

在以下区域讨论本文：
- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/ggj8tf/learning_rust_in_2020/)
- [official Rust users forum](https://users.rust-lang.org/t/blog-post-learning-rust-in-2020/42373)
- [rust subreddit](https://www.reddit.com/r/rust/comments/gie64f/learning_rust_in_2020/)
- [Hackernews](https://news.ycombinator.com/item?id=23160975)
- [Github](https://github.com/pretzelhammer/rust-blog/discussions)


## 参阅

- [Rust 中常见的有关生命周期的误解](./common-rust-lifetime-misconceptions.md)
- [Rust 标准库特性指南](./tour-of-rusts-standard-library-traits.md)
- [Sizedness in Rust](./../../sizedness-in-rust.md)
- [RESTful API in Sync & Async Rust](./restful-api-in-sync-and-async-rust.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](./too-many-brainfuck-compilers.md)
