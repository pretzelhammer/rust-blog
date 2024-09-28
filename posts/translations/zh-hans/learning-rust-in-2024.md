# 2024 年的 Rust 学习指南

_2024 年 9 月 21 日 · #rust · #coding · #exercises_

**目录**

- [TL; DR](#tl-dr)
- [0）参考资料](#0参考资料)
- [1）阅读 《半小时学会 Rust》](#1阅读-半小时学会-rust)
- [2）完成 rustlings](#2完成-rustlings)
- [3）花 10 小时编写 Rust 代码](#3花-10-小时编写-rust-代码)
  - [100 个练习学会 Rust](#100-个练习学会-rust)
  - [Exercism 上的 Rust 课程](#exercism-上的-rust-课程)
  - [Advent of Code](#advent-of-code)
    - [设置](#设置)
    - [2020 年](#2020-年)
    - [2022 年](#2022-年)
  - [教程](#教程)
- [4）阅读 《常见的 Rust 生命周期误解》](#4阅读-常见的-rust-生命周期误解)
- [5）再花 10 小时用 Rust 编程](#5再花-10-小时用-rust-编程)
- [6）阅读 《导览 Rust 标准库特征》](#6阅读-导览-rust-标准库特征)
- [接下来做什么？](#接下来做什么)
- [荣誉提名](#荣誉提名)
  - [Tour of Rust](#tour-of-rust)
  - [Comprehensive Rust](#comprehensive-rust)
  - [Rust 模块系统的清晰解释](#rust-模块系统的清晰解释)
- [讨论](#讨论)
- [进一步阅读](#进一步阅读)

这是我的个人指南，教你如何尽快从对 Rust 一无所知到不错的水平。如果我是今天开始学 Rust，那我也会告诉自己这些东西。

## TL; DR

1. 阅读 [《半小时学会 Rust》](https://fasterthanli.me/articles/a-half-hour-to-learn-rust)（30 - 60 分钟）
2. 完成 [rustlings](https://github.com/rust-lang/rustlings)（2 - 3 小时）
3. 花 10 小时用 Rust 编程（8 - 12 小时）
4. 阅读 [《常见的 Rust 生命周期误解》](./common-rust-lifetime-misconceptions.md)（30 - 60 分钟）
5. 再花 10 小时用 Rust 编程（8 - 12 小时）
6. 阅读 [《导览 Rust 标准库特征》](./tour-of-rusts-standard-library-traits.md)（2 - 4 小时）

按这样的过程，经过短短的 19 到 30 小时，你将从 Rust 初学者变成 Rust 高级初学者 😎

## 0）参考资料

学习 Rust 时，建议在浏览器中打开 [Rust by Example](https://doc.rust-lang.org/rust-by-example/index.html) 和 [Rust 标准库文档](https://doc.rust-lang.org/std/)。如果你遇到问题，可以在这两个资源中搜索答案。它们的顶部都有搜索栏，方便你快速查找。

其它不错的选择还有 StackOverflow 和 ChatGPT（或者其它你更喜欢的大型语言模型）。几乎每个 Rust 初学者的问题，都在 StackOverflow 上被提问过并得到回答。它们都可以通过 Google 搜索到。至于 ChatGPT，尽管有一半的时间可能给出错误答案，但至少能为你指明大致方向，让你知道下一步去哪里搜索答案。

如果你尝试了以上所有方式，但仍无法解决问题，那可以去一些适合初学者的 Rust 在线社区：[r/rust](https://www.reddit.com/r/rust/) 子版块，[r/learnrust](https://www.reddit.com/r/learnrust/) 子版块，以及 [官方 Rust 用户论坛](https://users.rust-lang.org/)。

另外，最好在浏览器标签页中打开 [Rust Playground](https://play.rust-lang.org/)。这是一个在线的 Rust 代码编辑器和编译器。它非常适合用来调试小的示例，以巩固你在阅读时学到的知识。

## 1）阅读 [《半小时学会 Rust》](https://fasterthanli.me/articles/a-half-hour-to-learn-rust)

相比其他资源，我更推荐 [《半小时学会 Rust》](https://fasterthanli.me/articles/a-half-hour-to-learn-rust)。它以一种非常系统和循循渐进的方法介绍 Rust 的语法和概念。它从最简单的例子开始，并在后续的例子中逐渐增加复杂性。这种安排非常符合逻辑，也易于理解。

## 2）完成 [rustlings](https://github.com/rust-lang/rustlings)

[Rustlings](https://github.com/rust-lang/rustlings) 是一组按序编排的 Rust 编程小练习。每个练习都是一段无法编译或未能通过单元测试的 Rust 代码。你的任务是修复这些代码，使其能够编译并通过单元测试。

开始 rustlings 之前，你需要安装 Rust。我建议使用 [rustup](https://rustup.rs/) 来管理 Rust 的安装。一旦安装了 `rustup`，你只需要运行 `rustup update`，就能随时更新你机器上的 Rust。

用 Rust 编程时，你可以使用任何你喜欢的代码编辑器。但我建议使用支持 [rust-analyzer](https://rust-analyzer.github.io/) 的编辑器。我使用 [VS Code](https://code.visualstudio.com/)，并安装了 [这个扩展](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust-analyzer) 来启用 rust-analyzer。

使用 rustlings 非常简单，可以用 `cargo` 来安装它。Cargo 是 Rust 的依赖管理器和构建工具，随 Rust 一起安装。如果你使用 `rustup` 安装 Rust，那么你也会同时会安装 Cargo。

运行 `cargo install rustlings` 来安装 rustlings。默认情况下，rustlings 的二进制文件会被放在 `$HOME/.cargo/bin` 中。因此，如果这个路径不在你的 `$PATH` 变量中，那你应该添加进去，例如：

```sh
export PATH="$HOME/.cargo/bin:$PATH"
```

现在运行 `rustlings`，它将为你创建一个 `rustlings/` 目录，里面包含了所有的练习。在代码编辑器中打开该目录，然后在目录内运行命令 `rustlings`，并按照其指示操作，它会告诉你应该做哪个练习。完成当前的练习后，命令会告诉你接下来应该做哪个练习。重复这个过程，直到你全部完成。

## 3）花 10 小时编写 Rust 代码

你可以编写任何代码。但对于你的前 10 小时，我建议使用单线程同步的 Rust。这里面有足够新颖且具有挑战性的 Rust 概念\*，足以让你至少忙上 10 小时。很多人甚至需要更长的时间才能完全理解。多线程异步 Rust 引入了更多复杂概念\*\*，在基础知识不牢固之前就开始学习，很容易让人感到不知所措。

\*：所有权（ownership）、借用（borrowing）、生命周期（lifetimes）、生命周期省略（lifetime elision）、可变与不可变引用（mutable vs immutable references）、固定大小（sizedness）、解引用强制转换（deref coercion）、特征（traits）、特征对象（trait objects）、泛型（generics）、常量泛型（const generics）、静态与动态分派（static vs dynamic dispatch）、错误处理（error handling）、迭代器（iterators）、模块（modules）、声明宏（declarative macros）、过程宏（procedural macros）...

\*\*：futures、固定（pinning）、轮询（polling）、异步等待（awaiting）、send、sync、取消（cancellation）、流（streams）、sinks、锁（locks）、原子数（atomics）、通道（channels）、actors ...

如果你不知道要写什么代码，那么我有一些建议。

### [100 个练习学会 Rust](https://rust-exercises.com/100-exercises/)

如果你喜欢 rustlings 的难度和节奏，那么你会喜欢《100 个练习学会 Rust》（100 Exercises To Learn Rust，100ELR）。尽管由不同的作者制作，但 100ELR 显然从 rustlings 获得了大量的灵感，更像是后者的延续，。

开始 100 ELR，首先要克隆 [mainmatter/100-exercises-to-learn-rust](https://github.com/mainmatter/100-exercises-to-learn-rust) 仓库到你的机器上，接着使用 `cargo install --locked workshop-runner` 安装测试套件的运行程序，然后就能开始阅读 [配套书籍](https://rust-exercises.com/100-exercises/) 了。它有 100 章，每章结束时，都会告诉你在仓库中做哪个练习。完成一个练习后，在项目根目录运行 `wr` 来检查你的解答是否通过了单元测试，如果通过了，就继续下一章。重复这个过程，直到你完成所有练习。

虽然 100 个章节听起来很多，但它们都很短，所以你会比想象中更快地完成它们。大多数人来说。完成整本书可能需要 15 到 25 小时。

然而，如果你觉得 rustlings 太简单，并且想尝试更具挑战性的东西，那么你可能想要跳过 100ELR。

### Exercism 上的 [Rust 课程](https://exercism.org/tracks/rust)

除了 rustlings 和 100ELR 之外，Exercism 上的 [Rust 课程](https://exercism.org/tracks/rust) 是互联网上最好的 Rust 小练习。它的优秀在于，许多练习是由 Rustaceans（指 Rust 开发者，译者注）制作的，旨在教你 Rust 相较于其它编程语言的独特之处。有一个练习要求你编写一个声明式宏！甚至有一个练习要求你编写不安全的代码！

为了开始该课程，你需要在 Exercism 上注册账户，并加入 Rust 课程（两者都是免费的）。然后，你可以用 Exercism 的在线代码编辑器完成练习，或者将练习下载到你的机器上，使用自己的代码编辑器完成它们，并通过 `exercism` cli 工具将解答提交到 Exercism 上。

我建议使用 `exercism` cli 工具。它有很多优点。首先，你的代码编辑器有 rust-analyzer，而 Exercism 的在线代码编辑器没有。其次，你可以在编辑器中使用暗黑模式，但在 Exercism 上只有付费后才能使用（这是高级功能）。第三，你可以使用 git 跟踪你的解答。第四，在编写答案时，你可以选择运行哪些单元测试，而不是每次都运行完整的测试套件。[这里](https://exercism.org/docs/using/solving-exercises/working-locally) 是如何在本地机器上进行练习的官方说明。

Exercism 上的每个练习都有一个「学习模式」和「练习模式」。目前，由于 Exercism 团队正在重新设计，Rust 课程的「学习模式」已被禁用。但不管如何，我认为「练习模式」更好，因为它允许你按任意顺序进行练习。虽然现在没有选择，但如果将来 Rust 课程的「学习模式」和「练习模式」重新开放，我建议选择「练习模式」。

虽然 Rust 课程上的练习是按序的，但这个顺序是任意的，是否遵循并不重要。除了 [Luhn](https://exercism.org/tracks/rust/exercises/luhn)，[Luhn From](https://exercism.org/tracks/rust/exercises/luhn-from)，和 [Luhn Trait](https://exercism.org/tracks/rust/exercises/luhn-trait) 以外，所有其他的练习都彼此无关，可以按任何顺序完成。

完成练习后最好的地方，是去浏览和点赞其他人的解答。我建议完成练习后看看它们，尤其是那些点赞最多的解答，可以教你一些你没有发现的优雅和惯用的 Rust 写法。

练习按难度分为：简单、中等和困难。目前 Rust 课程共有 99 个练习，我认为有 30 个非常好，其他的 59 个只是从其他课程中复制粘贴过来的普通练习。完成整个课程的人占比不到 0.1%。如果完成一半的人不到 1%，我也丝毫不会吃惊。我已经完成了所有练习。考虑到你可能不会完成所有练习，所以我将推荐我认为最好的那些。

你应该从简单练习开始。难度方面，它们比你在 rustlings 或 100ELR 中的稍微更具挑战性。质量方面，它们都还不错。完成任何一个简单练习都不会超过 20 分钟。另外，如果完成简单练习不再让你感到有收获，那就转到中等练习。不要强迫自己完成所有的简单练习。

就中等难度而言，以下是我个人认为的最好练习（排名不分先后）：

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

根据你选择的练习，完成时间可能需要 10 到 45 分钟。

当你准备好再次提高难度时，以下是我认为最好的困难练习（排名不分先后）：

- [Xorcism](https://exercism.org/tracks/rust/exercises/xorcism)
- [React](https://exercism.org/tracks/rust/exercises/react)
- [Circular Buffer](https://exercism.org/tracks/rust/exercises/circular-buffer)
- [Forth](https://exercism.org/tracks/rust/exercises/forth)
- [Doubly Linked List](https://exercism.org/tracks/rust/exercises/doubly-linked-list)

根据你选择的练习，完成时间可能需要 30 到 90 分钟。

完成一系列简单练习，再加上我推荐的中等和困难练习，大概会花费约 20 小时。完成整个课程可能需要两倍的时间，约 40 小时。话虽如此，如果你觉得自己开始失去兴趣，就不要勉强自己完成整个课程。此时最好调整方向，尝试 Rust 的其它东西。学习过程中最重要的事，是保持乐趣。

### [Advent of Code](https://adventofcode.com/)

[Advent of Code](https://adventofcode.com/)（AoC）是一组编程谜题。自 2015 年起，每年从 12 月 1 日到 25 日，每天都会发布一个新的谜题，每个谜题有两个部分。每年的前 10-15 个谜题往往是简单或中等难度，最后 10-15 个谜题通常很难。谜题被设计成与语言无关，意味着你可以使用任何编程语言来解决。如果你的目标是学习 Rust，那么仅完成谜题并不会给你太多帮助。但如果完成一个谜题后，再看看一位经验丰富的 Rustacean 的解决方案，就可能会帮你学到 Rust 的很多知识。

在这里，这位经验丰富的 Rustacean 就是 fasterthanlime。他不仅发布了 2020 年和 2022 年 AoC 谜题的解答，还写了一些面向初学者的博客文章，解释每个谜题的解决思路。如果你打算通过 AoC 学习 Rust，那么你应该从 2020 年的谜题开始，在完成每个谜题后，阅读 fasterthanlime 对应的博客文章。无论是哪一年，我都不建议做第 10 个之后的谜题，因为它们可能会变得极具挑战性，你会绞尽脑汁，花费大部分时间去想如何解决问题，而不是学习 Rust。

#### 设置

首先，在 [AoC](https://adventofcode.com/) 上创建账号。其次，由于 AoC 与语言无关，你必须设置你自己的 cargo 项目并搭建脚手架来解决这些谜题。这样做可能非常有教育意义，但也相当枯燥。我建议跳过这一步，使用这个 Github 模板：[fspoettel/advent-of-code-rust](https://github.com/fspoettel/advent-of-code-rust)：点击 `Use this template` -> `Create a new repository`，然后 `git clone` 克隆仓库到你的机器上。在 `.cargo/config.toml` 中将 `AOC_YEAR` 设置为 `2020`。

你还可以安装 `aoc-cli` 工具，从你的 AoC 账户获取指令和输入到你的机器上。为此，要运行 `cargo install aoc-cli --version 0.12.0`，然后创建文件 `$HOME/.adventofcode.session`，并将你的 AoC 的 cookie 粘贴进去。获取 cookie，需要在 AoC 网站中，按 `F12` 打开浏览器开发者工具，然后在 `Application` 或 `Storage` 选项卡下找到 `Cookies`，复制 `session` cookie 的值。

完成这些步骤后，你可以在项目目录中运行 `cargo download {day}`，生成脚手架并下载当天的谜题。一旦你认为已经完成，就可以运行 `cargo solve {day}`，通过 AoC 网站提交解答并进行验证。例如，对于第 1 天，命令将是 `cargo download 1` 和 `cargo solve 1`。

#### 2020 年

正如前面提到的，一定要在 `.cargo/config.toml` 中将 `AOC_YEAR` 设置为 `2020`。完成一个谜题后，去阅读 fasterthanlime 在他的博客上发布的解决方案。这是他前 10 个谜题解决方案的链接：

1. [AoC 2020 Day 1](https://fasterthanli.me/series/advent-of-code-2020/part-1)
2. [AoC 2020 Day 2](https://fasterthanli.me/series/advent-of-code-2020/part-2)
3. [AoC 2020 Day 3](https://fasterthanli.me/series/advent-of-code-2020/part-3)
4. [AoC 2020 Day 4](https://fasterthanli.me/series/advent-of-code-2020/part-4)
5. [AoC 2020 Day 5](https://fasterthanli.me/series/advent-of-code-2020/part-5)
6. [AoC 2020 Day 6](https://fasterthanli.me/series/advent-of-code-2020/part-6)
7. [AoC 2020 Day 7](https://fasterthanli.me/series/advent-of-code-2020/part-7)
8. [AoC 2020 Day 8](https://fasterthanli.me/series/advent-of-code-2020/part-8)
9. [AoC 2020 Day 9](https://fasterthanli.me/series/advent-of-code-2020/part-9)
10. [AoC 2020 Day 10](https://fasterthanli.me/series/advent-of-code-2020/part-10)

#### 2022 年

如果你已经完成了至少 10 个 2020 年的谜题，那么推荐接下来做 2022 年的谜题。再克隆一份 [fspoettel/advent-of-code-rust](https://github.com/fspoettel/advent-of-code-rust)，并在 `.cargo/config.toml` 中将 `AOC_YEAR` 设置为 `2022`。这是 fasterthanlime 对 2022 年谜题的解答的链接：

1. [AoC 2022 Day 1](https://fasterthanli.me/series/advent-of-code-2022/part-1)
2. [AoC 2022 Day 2](https://fasterthanli.me/series/advent-of-code-2022/part-2)
3. [AoC 2022 Day 3](https://fasterthanli.me/series/advent-of-code-2022/part-3)
4. [AoC 2022 Day 4](https://fasterthanli.me/series/advent-of-code-2022/part-4)
5. [AoC 2022 Day 5](https://fasterthanli.me/series/advent-of-code-2022/part-5)
6. [AoC 2022 Day 6](https://fasterthanli.me/series/advent-of-code-2022/part-6)
7. [AoC 2022 Day 7](https://fasterthanli.me/series/advent-of-code-2022/part-7)
8. [AoC 2022 Day 8](https://fasterthanli.me/series/advent-of-code-2022/part-8)
9. [AoC 2022 Day 9](https://fasterthanli.me/series/advent-of-code-2022/part-9)
10. [AoC 2022 Day 10](https://fasterthanli.me/series/advent-of-code-2022/part-10)

### 教程

大多数 Rust 教程的目标，是向你展示如何用 Rust 实现某个软件，而不一定是教你 Rust 语言本身。因此，许多教程作者都假设读者已经对这门语言有一定的了解，为了保持教程的重点和简洁，会省略许多针对初学者的解释。

话虽如此，编写一个有一定难度的软件，比完成一堆小练习要更有趣和更有成就感。所以这种方法可能更适合保持你的学习动力，特别是当你在构建一个你真正感兴趣和兴奋的东西时。

我希望在这里推荐一些教程。但是已经有太多教程，很难一一评估和按质量排名。不过，质量可能并不是那么重要，重要的是你在编写 Rust 代码并享受这个过程。所以，选择任何看起来有趣的教程并跟着学习，不要过分纠结细节。

不过，我要特别提一下 [Codecrafters](https://codecrafters.io)。他们的教程质量非常高，而且所有教程都提供了 Rust 模板。在教程的每一步，他们都会给出指导和提示，告诉你如何实现程序的下一部分。你完成并提交代码后，它会运行一个测试套件，来检查是否所有内容都正确实现。

Codecrafters 的最大缺点是费用，每月要 40 美元。他们偶尔会有促销活动，免费开放某个教程一个月。你在浏览他们网站时，可能会幸运地发现一个你感兴趣的教程是免费的。然而，这个定价表明，他们的目标是那些希望培训其工程师的软件公司，而不是个人工程师。如果你在这样的公司工作，可以查看你的公司是否为类似 Codecrafters 的课程提供培训津贴，这样你就可以用这些津贴来支付费用。

## 4）阅读 [《常见的 Rust 生命周期误解》](./common-rust-lifetime-misconceptions.md)

免责声明：这是我写的。

阅读 [《常见的 Rust 生命周期误解》](./common-rust-lifetime-misconceptions.md) 将帮助你理解，为什么在过去的 10 个小时的 Rust 编程中，你一半的尝试都不起作用 🙂

玩笑归玩笑，理解生命周期可能是许多初学者在学习 Rust 时遇到的最大绊脚石。这篇文章澄清了关于生命周期的最常见误解，它们常常引起人们的困惑和挫败感。

我之所以建议在至少有 10 小时的 Rust 编码实践经验后再阅读它，是因为对纯粹的初学者来说，这篇文章可能包含了过多的技术细节，会让人感到不知所措，而不是提供帮助。

## 5）再花 10 小时用 Rust 编程

如果你仍然不知道要编写什么代码，我的建议保仍然是：[100 个练习学会 Rust](https://rust-exercises.com/100-exercises/)，Exercism 上的 [Rust 课程](https://exercism.org/tracks/rust)，[Advent of Code](https://adventofcode.com/)，或其它教程。

现在你已经有了一些 Rust 经验。如果你喜欢冒险，可以尝试用 Rust 编写一些多线程异步代码。

## 6）阅读 [《导览 Rust 标准库特征》](./tour-of-rusts-standard-library-traits.md)

免责声明：这是我写的。

特征（Trait）是 Rust 中编写多态代码的主要方式。它们无处不在，尤其是对来自标准库的特征来说。本文对最常见的标准库特征进行了导览，介绍了它们的方法、如何使用它们以及何时为你的自定义类型实现它们。这篇文章相当详尽，并分享了大量实用的技巧。尽管篇幅较长，但你不必读阅读全文就能从中受益，所以不要被它的长度吓到。

## 接下来做什么？

恭喜你！你做到了。你肯定已经掌握了足够的 Rust 知识，可以自行探索前进的道路。祝你好运并玩得开心。

## 荣誉提名

这些是我非常喜欢的其他的 Rust 初学者资源，但我不知道应该将它们放在本指南的何处。

### [Tour of Rust](https://tourofrust.com/)

这是一个非常好的资源。然而，它涉及很多其他资源已经覆盖的内容，所以为了保持指南的简洁性，我没有将其包括在内。

### [Comprehensive Rust](https://google.github.io/comprehensive-rust/)

这是 Google 的 Android 团队开发的 Rust 课程。它是一个幻灯片集合，本应由一位具有 Rust 经验的讲者来讲解，但书籍版本在每页底部有讲者注释，因此也可以作为独立资源来学习。

这门课程非常简洁，节奏很快，同时还涵盖了其他初学者资源中通常忽略的 Rust 部分，例如 [裸机 Rust](https://google.github.io/comprehensive-rust/bare-metal.html) 和 [Rust 中的并发性](https://google.github.io/comprehensive-rust/concurrency/welcome.html)。然而，因为它更适合由讲者来呈现，所以不太适合放入面向自学者的指南中。

### [Rust 模块系统的清晰解释](https://www.sheshbabu.com/posts/rust-module-system/)

这是一篇很棒的文章。它可以放在该指南中的任何地方，所以我不确定将它放在哪里。理想情况下，学习者开始将 Rust 代码分散到多个文件时，应该阅读这篇文章。在每个人的 Rust 编码旅程中，这个时间点会有所不同。无论如何，当你到了那一步时，请阅读这篇文章。

## 讨论

在以下平台讨论本文：

- [Github](https://github.com/pretzelhammer/rust-blog/discussions/84)

## 进一步阅读

- [常见的 Rust 生命周期误解](./common-rust-lifetime-misconceptions.md)
- [Rust 标准库特性指南](./tour-of-rusts-standard-library-traits.md)
- [并发编程初学者指南：使用 Tokio 编写多线程聊天服务器](./chat-server.md)
- [Sizedness in Rust](../../sizedness-in-rust.md)
- [RESTful API in Sync & Async Rust](../../restful-api-in-sync-and-async-rust.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](../../too-many-brainfuck-compilers.md)
