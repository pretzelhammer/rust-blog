
> 原文链接: https://github.com/pretzelhammer/rust-blog/blob/master/posts/learning-rust-in-2020.md
> 
> 翻译：[Asura](https://github.com/asur4s)
> 
> 选题：[Asura](https://github.com/asur4s)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# Rust 大佬给初学者的学习建议

2020 年 5 月 9 日 · #rust · #programming · #exercises

**目录**

- [Rust 大佬给初学者的学习建议](#rust-大佬给初学者的学习建议)
  - [简介](#简介)
  - [摘要](#摘要)
  - [Rust 练习场评测](#rust-练习场评测)
    - [HackerRank](#hackerrank)
    - [欧拉计划（Project Euler）](#欧拉计划project-euler)
    - [力扣（LeetCode）](#力扣leetcode)
    - [CodeWars](#codewars)
    - [Advent of Code](#advent-of-code)
    - [Rustlings](#rustlings)
    - [Exercism](#exercism)
  - [结论](#结论)
  - [讨论](#讨论)
  - [通知](#通知)
  - [延申阅读](#延申阅读)


## 简介

当我开始学习 Rust 的时候，我犯了一个错误，那就是先读[《The Rust Programming Language》](https://doc.rust-lang.org/book/title-page.html)。虽然这是一本非常好的资料，但让新手一开始就阅读这本 20 个章节的书籍，简直令人望而生畏，大多数人还没开始就放弃了。没有人会让一个刚开始学习 JavaScript 或者 Python 的人去阅读一本 20 个章节的书籍。Rust 学习曲线非常陡峭的，但只要循序渐进的学习一定也能学有所成。


## 摘要

如果你是一个完完全全的 Rust 小白，想要在一天中尽可能多的学习 Rust，那我推荐你去阅读 fasterthanlime 的[《半小时快速了解 Rust》](https://fasterthanli.me/articles/a-half-hour-to-learn-rust)，然后完成 [Rustlings](https://github.com/rust-lang/rustlings) 项目中的练习。

如果你已经学过 Rust 的基本语法，你可以试着做一下 Exercism.io 网站上的 [Rust](https://exercism.org/tracks/rust) 部分。如果你遇到了问题，你可以在 Google 或者 StackOverflow 上寻求帮助。我推荐你花点时间来简单的阅读和浏览一下[《Rust Standard Library Docs》](https://doc.rust-lang.org/std/)，它是一个很棒的学习资料，里面有一些简单且实用例子去帮助你更好的使用 Rust 的标准库。[《Rust by Example》](https://doc.rust-lang.org/rust-by-example/)也是一本高质量的参考资料，你可以通过他快速的学习 Rust 的语法和特性。如果你想要更深入的理解 Rust 的某一个概念，那么我推荐你在[《The Rust Programming Language》](https://doc.rust-lang.org/book/title-page.html)这本书中寻找相关的章节去阅读。尤其推荐在 Exercism.io 上进行练习。在完成每个题目之后，你可以查看其他所有人的题解，可以按点赞数排序来找到通俗易懂并且巧妙的题解。这是一种很棒的学习方式。

此时，你可能已经是一个高级的初学者，能够找到属于自己的学习路线。但，如果你还需要更多的指导并想要尝试用 Rust 来写一些简单的程序，我推荐你试一着做一下 [Advent of Code 2018 Calendar](https://adventofcode.com/2018) 上的练习。为什么推荐你做 2018 年的题目呢？因为当你做完了这个练习，你可以和 BurntSushi 提供的答案（ [BurntSushi's Advent of Code 2018 Rust solutions](https://github.com/BurntSushi/advent-of-code)）进行对比。BurntSushi 写的 Rust 代码整洁、可读性强、通俗易懂。阅读一个有经验的 Rustacean 的代码将会使你受益无穷。

现在可以退出了，文章最精彩的部分已经结束了。

## Rust 练习场评测

*别名：适合 Rust 新人的免费练习场评测*

大多数的练习场都不是专门用于练习 Rust 的，但都可以用于学习和练习 Rust。某些练习场还明确的表示可以提交 Rust 代码，并支持指定 Rust 版本。

下面的练习场是按照**从最坏到最好**进行排序的。

### [HackerRank](https://www.hackerrank.com/)

虽然 HackerRank 支持 Rust 语言，但大多数题目并不能提交题解。当我上传题解的时候被直接拒绝了。

![](https://github.com/pretzelhammer/rust-blog/blob/master/assets/hackerrank-more-like-failrank.png)

迷惑。我能浏览其他用户上传的 Rust 题解，按道理说，我也可以上传 Rust 题解。我尝试在 Google 上搜索这个问题，并没有找到有用的信息。我没办法将 HackerRank 和其他网站对比，因此我只想告诉你，别像我这样浪费时间，试试其他网站吧。

### [欧拉计划（Project Euler）](https://projecteuler.net/)

我是从 2012 年开始学习编程的，当时我常常听到“如果你想要快速学习一门编程语言，来做几个 Project Euler 的题目吧！”，这在当时是个不错的建议，因为当时没有其他更好的选择。但在我看来，Project Euler 和编程关系不大。Project Euler 的题目大多是数学问题，而不是编程问题。这些题目需要大量的数学推导才能做出来，相比之下，所需要的编程能力是微不足道的。我不推荐你在 Project Euler 上学习 Rust，除非你数学很强或对这个网站有一些情结。

### [力扣（LeetCode）](https://leetcode.com/problemset/all/)

力扣上支持使用 Rust 来编写题解。力扣上的每个题目，你都会得到一个模板，模板通常包含一个没有被实现函数，你需要实现它并使其完成题目要求的功能。对于某个比较复杂的问题，模板可能会包含一些 struct 结构体和 impl 代码块、几个未被实现的方法。不幸的是，这些模板不是由人工撰写的，而是由机器自动生成的，这会导致你写出许多不规范 Rust 代码。让我们来对比一下力扣生成的 Rust 代码和优雅的 Rust 代码

| 力扣生成的 Rust 代码 | 优雅的 Rust 代码 |
| ---                 | ---             |
| 使用 `Option<Rc<RefCell<Node>>>` 来表示链表 | 使用 `Option<Rc<RefCell<Node>>>` 来表示链表太复杂。`Option<Box<Node>>` 同样能表示链表，并且使用起来更简单。  |
| insert 方法需要改变 self，但生成的代码依然使用不可改变的借用。例如：`fn insert(&self, val: i32)` | 将 self 需要改成可变借用。例如：`fn insert(&mut self, val: i32)` |
| 即使程序不会使用负数，依然将所有的数字都声明为 32 位的整型。例如：`fn nth_fib(n: i32) -> i32` | 如果程序不使用负数，我们应该使用无符号整数。例如：`fn nth_fib(n: u32) -> u32` |
| 即使不需要获得参数的所有权，依然，。例如：`fn sum(nums: Vec<i32>) -> i32` | 如果不需要参数的所有权，那么应该使用借用。例如：`fn sum(nums: &[i32]) -> i32` |

在力扣上练习 Rust 的问题：
- 力扣不允许你在做题时引入第三方的依赖。对于大多数语言来说，我认为这是合理的，但 Rust 的标准库非常小，甚至不支持正则表达式，使用 Rust 来做复杂字符串解析意义不大。由于其他语言的标准库自带正则表达式，所以做起来很简单。
- 所有的并发类问题，都不能提交题解。无法理解！安全且高效的并发是 Rust 的一大亮点。
- 在做完题目后，你能在问题的评论区查看其他用户的题解（许多用户喜欢讲题解发布在评论区），但由于 Rust 太小众，有时你会找不到 Rust 的题解。

力扣的常见问题：
- 力扣上有大量低质量的题目。虽然用户可以选择喜欢或是不喜欢，但题目不会被删除，即使这个题目被绝大多数讨厌。我看过许多 100+ 投票的问题，其中 80% 都是不喜欢，我不理解为什么还要保留这些题目。
- 题目的难度等级不准确。力扣上的题目共有三个难度等级，分别是简单、中等、困难，但有很多简单题的解题率比难题更低。
- 不是所有的题目都能用任意语言提交。你没办法直观的判断题目支持哪些语言。例如，所有关于图的题目都不支持使用 Rust 提交。

力扣的优点：
- 提交的题解会使用一组未知的数据来进行测试，如果程序在某个特殊数据出错了，它会把这个特殊数据显示给你，帮助你完善题解。
- 生成的所有 Rust 代码都符合 rustfmt 格式规范。



### [CodeWars](https://www.codewars.com/join?language=rust)

Codewars 这个名字经常被误解，这里并没有竞赛。Codewar 的题目没有时间限制，也不会用你的执行速度和内存使用情况来评价你的题解。值得注意的是，没有任何人和你一决高下并不是什么坏事。

Codewars 支持使用 Rust 来编写题解。类似力扣，Codewars 上的每个题目，你都会得到一个模板，模板通常会包含一个没有被实现函数，你需要实现它并使其完成题目要求的功能。这些模板是由人工撰写的，不是所有撰写模板的人都擅长 Rust，所以你可能偶然会碰到糟糕的 Rust 题目。例如：

| Codewars 的模板 | 优雅的 Rust 代码 |
| ---                 | ---             |
| 有时不遵循 rustfmt 格式规范，例如：`fn makeUppercase(s:&str)->String` | 应该尽量遵循 rustfmt 格式规范的写法，例如：`fn make_uppercase(s:&str)->String`  |
| 即使程序不会使用负数，依然将所有的数字都声明为 32 位的整型。例如：`fn nth_fib(n: i32) -> i32` | 如果程序没有负数，我们应该使用无符号整数。例如：`fn nth_fib(n: u32) -> u32` |
| 当程序需要将 `-1` 作为返回结果时，例如：`fn get_index(needle:i32, haystack: &[i32])->i32` | 应该将返回类型包装成 `Option` 类型。 例如：`fn get_index(needle:i32, haystack: &[i32]) -> Option<usize>`|
| 有时不充分利用解引用强制转换。例如：`fn do_stuff(s: &String, list: &Vec<i32>)` | 充分利用解引用强制转换。例如：`fn do_stuff(s: &str, list: &[i32])` |

上面的问题只是偶然会碰到，因为撰写模板的人的技能水品不同。相比于力扣，这是一个很大的进步，力扣上的 Rust 题目普遍都比较糟糕。在 Codewars 上的 Rust 学习者整体趋向于经验不足，因为我经常在高赞题解中看见非常普通的代码。例如：
| Codewars 的高赞题解 | 优雅的 Rust 写法 |
|--|--|
| 有时在函数末尾使用 return。例如：`return result;` | 代码块可以被作为表达式，函数末尾的 return 通常省略不写。例如：`result` |
| 总是使用紧凑的格式来让题解显得更简洁。 | 应该遵循 rustfmt 格式规范  |
| 有时调用不必要的函数。例如：`str_slice.to_string().chars()` | 如果不需要，就不调用。例如：`str_slice.chars()` |
| 总是使用迭代器来编写题解，忽视了其他方面的问题。 | 迭代器很好用，也很常用。但如果你把 15 个迭代器写在一行，或是将迭代器多层嵌套在一起，你应该考虑重构一些函数。|

上面的问题也只是偶然会碰到。有经验的 Rust 开发者能很轻松的发现这些问题，但大部分的 Rust 初学者并不清楚它们正在学习不规范的代码。

Codewars 在 Rust 方面的其他问题：
- Rust 的题目非常少。这个网站一共 9000 多个题目，但只有 300 个支持使用 Rust 提交题解。

Codewars 的常见问题：
- 当题目提交后，系统会使用一组未知的数据来测试你的题解。如果测试没有全部通过，系统并不会告诉你哪个数据出现问题。如果边界值没有在题目描述中清楚的交代，这会让人非常恼火。
  
Codewars 的优点：
- Codewars 有一个很小的 Rust 第三方依赖库列表，你可以用这些依赖库来解决问题。支持的第三方依赖库有：`rand`、`chrono`、`regex`、`serde`、`itertools`、`lazy_static`，它们能让 Rust 更接近其他语言。
- 你可以直接筛选出支持 Rust 的语言。
- 题解被提交后，会被自动发布到答案区。你也可以查看其他人的题解，并对其进行投票。你可以按点赞数对题解进行排序，前面的题解通常都是简单而高效的，但有时也会不符合规范。
- 题目的难度等级非常准确。Codewars 并不是像力扣一样将问题从易到难分为三个阶段（简单、中等、困难），而是将难度分为八个等级，它们由易到难分别是：8kyu、7kyu、6kyu、5kyu、4kyu、3kyu、2kyu、1kyu。在 8kyu 到 4 kyu 之间，我共计完成了 60 个题目，每一层难度都比只上一层难一点，非常适合训练。

### [Advent of Code](https://adventofcode.com/)

Advent of Code（简称 AoC）和编程语言完全没有关系。HackerRank、LeetCode、CodeWars 对 Rust 的支持并不好，相比之下，使用 AoC 可能效果更好。 在我在看，AoC 的题目是有趣的、多样化的、高质量的。

AoC 的常见问题：
- 当你完成练习后，没办法看到其他人的题解，并且无法判断在 Google 上搜索的题解质量。

为了解决上面的问题，我推荐去做 2018 年的题目，然后将你的题解和 BurntSushi 提供的答案（ [BurntSushi's Advent of Code 2018 Rust solutions](https://github.com/BurntSushi/advent-of-code)）进行对比。BurntSushi 写的 Rust 代码整洁、可读性强、通俗易懂。如果你想要完成 2019 年的题目，我推荐参考 bcmyers 的题解（[bcmyers' AoC 2019 Rust solutions](https://github.com/bcmyers/aoc2019)），bcmyers 还在 Youtube 上制作了一个[ Youtube 播放列表](https://www.youtube.com/playlist?list=PLQXBtq4j4Ozkx3r4eoMstdkkOG98qpBfg)来讲解这些题目，他很好的解释了他的思考过程以及他为什么的代码要做这些事情。

### [Rustlings](https://github.com/rust-lang/rustlings)

极力推荐 Rustlings，因为它太棒了。这是一套能教会你 Rust 常规用法的练习。

如果你是一个完完全全的 Rust 小白，你必须试试 [Rustlings](https://github.com/rust-lang/rustlings)。不过，我极力推荐你先阅读 fasterthanlime 的[《半小时快速了解 Rust》](https://fasterthanli.me/blog/2020/a-half-hour-to-learn-rust/)，它会帮助你迅速了解很多的 Rust 语法和概念。

Rustlings 的一点小缺陷：在“错误处理”（Error handling）和“类型转换”（Type conversions）部分突然难度飙升，很多人在这不知所措。

Rustlings 太短了，这并不是在批评它，Rustlings 设计的目的就是简单快速的介绍 Rust。但它写得太好了，我希望这样的内容能再多一点。

### [Exercism](https://exercism.org/tracks/rust)

Exercism 有一套 Rust 的练习集，大致按主题和难度系数进行排序。虽然这套 Rust 的练习集类似于其他练习集，但所有的练习题都是由经验丰富的 Rustacean 来改写的，不会遇到在 LeetCode 或 Codewars 上的问题。上面由很多 Rust 的经典问题，实现一个标准库中的 Trait，或是写一个 Macro，或是使用多线程写一个并发问题，或是使用 unsafe 编写 Rust 代码。Exercism 是仅次于 Rustlings 的 Rust 学习资料。Rustlings 被放在 Exercism 之上，仅仅是因为 Rustlings 可以在一个晚上完成，而 Exercism 的 Rust 练习集需要花费至少一个月。 

在 Exercism 上练习 Rust 的问题：
- 不能使用“指导模式”（Mentored mode）。因为网站上大部分的 Rust 导师都并不活跃，所以最好使用“练习模式”（Practice mode）
- 一共有 92 个练习题，但其中很大一部分不会教你任何新知识，从中砍掉 20 个左右的练习题，可以让内容更紧凑一点。

Exercism 的优点：
- 所有的练习题都是由经验丰富的 Rustacean 来改写的。
- 内容包括 Rust 的习惯用法、设计模式、语言特性。
- 由易到难，难度等级准确。
- 你可以在题解中引用任何第三方依赖库。
- 所有的测试案例都是公开的，如果测试样例不通过，你可以知道为什么没有通过。
- 提交题解之后，你可以查看别人的题解，并且题解可以按质量来排序。

## 结论

同[摘要部分](#摘要)。

## 讨论

可以在下面这些地方讨论本文：
- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/ggj8tf/learning_rust_in_2020/)
- [official Rust users forum](https://users.rust-lang.org/t/blog-post-learning-rust-in-2020/42373)
- [Twitter](https://twitter.com/pretzelhammer/status/1259897499122360322)
- [rust subreddit](https://www.reddit.com/r/rust/comments/gie64f/learning_rust_in_2020/)
- [Hackernews](https://news.ycombinator.com/item?id=23160975)
- [Github](https://github.com/pretzelhammer/rust-blog/discussions)

## 通知

通过这些渠道获取最新消息
- [Following pretzelhammer on Twitter](https://twitter.com/pretzelhammer) or
- [Subscribing to this repo's release RSS feed](https://github.com/pretzelhammer/rust-blog/releases.atom) or
- Watching this repo's releases (click Watch -> click Custom -> select Releases -> click Apply)

## 延申阅读

- [Common Rust Lifetime Misconceptions](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md)
- [Sizedness in Rust](https://github.com/pretzelhammer/rust-blog/blob/master/posts/sizedness-in-rust.md)
- [Tour of Rust's Standard Library Traits](https://github.com/pretzelhammer/rust-blog/blob/master/posts/tour-of-rusts-standard-library-traits.md)
- [RESTful API in Sync & Async Rust](https://github.com/pretzelhammer/rust-blog/blob/master/posts/restful-api-in-sync-and-async-rust.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](https://github.com/pretzelhammer/rust-blog/blob/master/posts/too-many-brainfuck-compilers.md)

