# 在非 Rust 服务器中逐步引入 Rust 以提高性能

_2024 年 10 月 22 日 · #rust · #wasm · #ffi · #performance_

**目录**

- [介绍](#介绍)
- [策略](#策略)
  - [第 0 层：不使用 Rust](#第-0-层不使用-Rust)
  - [第 1 层：Rust CLI 工具](#第-1-层Rust-CLI-工具)
  - [第 2 层：Rust Wasm 模块](#第-2-层Rust-Wasm-模块)
    - [手动编写 Wasm 绑定](#手动编写-Wasm-绑定)
  - [第 3 层：Rust 原生函数](#第-3-层Rust-原生函数)
  - [第 4 层：Rust 重写](#第-4-层Rust-重写)
- [总结](#总结)
- [讨论](#讨论)
- [延伸阅读](#延伸阅读)
- [通知](#通知)

## 介绍

在本文中，我将讨论如何逐步将 Rust 引入到用其他语言（如 JavaScript、Python、Java、Go、PHP、Ruby 等）编写的服务器中。主要原因是，当你对服务器进行性能分析后，发现某个热点函数由于 CPU 瓶颈无法满足性能要求，而通常的优化手段（如函数记忆化或改进算法）在这种情况下不可行或无效时，你可能会考虑将该函数的实现替换为用更高效的 CPU 语言（如 Rust）编写的版本。如果你正面临这种情况，那么这篇文章非常适合你。

这些策略按“层级”排序，其中“层级”指的是“Rust 采用的层级”。第一层是完全不使用 Rust，最后一层则是将整个服务器重写为 Rust。

我们将在一个用 JS 编写的 Node.js 示例服务器上应用和基准测试这些策略。不过，这些策略可以推广到任何其他语言或运行时。

> [!NOTE]
> 本文中所有示例的完整源代码可以在 [此仓库](https://github.com/pretzelhammer/using-rust-in-non-rust-servers) 中找到。

## 策略

### 第 0 层：不使用 Rust

假设我们有一个 Node.js 服务器，它有一个 HTTP 端点，该端点接收一个文本字符串作为查询参数，并返回一个 200px × 200px 的 PNG 图像，图像内容是该文本的二维码。

以下是服务器代码：

```js
const express = require("express");
const generateQrCode = require("./generate-qr.js");

const app = express();
app.get("/qrcode", async (req, res) => {
  const { text } = req.query;

  if (!text) {
    return res.status(400).send('missing "text" query param');
  }

  if (text.length > 512) {
    return res.status(400).send("text must be <= 512 bytes");
  }

  try {
    const qrCode = await generateQrCode(text);
    res.setHeader("Content-Type", "image/png");
    res.send(qrCode);
  } catch (err) {
    res.status(500).send("failed generating QR code");
  }
});

app.listen(42069, "127.0.0.1");
```

以下是热点函数的代码：

```js
const QRCode = require("qrcode");

/**
 * @param {string} text - 要编码的文本
 * @returns {Promise<Buffer>|Buffer} - 二维码
 */
module.exports = function generateQrCode(text) {
  return QRCode.toBuffer(text, {
    type: "png",
    errorCorrectionLevel: "L",
    width: 200,
    rendererOpts: {
      // 这些选项在测试中提供了速度和压缩之间的最佳平衡
      deflateLevel: 9, // 0 - 9
      deflateStrategy: 3, // 1 - 4
    },
  });
};
```

我们可以通过以下方式调用该端点：

```
http://localhost:42069/qrcode?text=https://www.reddit.com/r/rustjerk/top/?t=all
```

这将正确生成以下二维码 PNG：

![QR code for rustjerk subreddit](../../../assets/rustjerk-subreddit-qr-code.png)

现在，让我们在 30 秒内向该服务器发送数万个请求，看看它的表现如何：

| 层级    | 吞吐量       | 平均延迟 | p99 延迟 | 平均响应大小 | 内存    |
| ------- | ------------ | -------- | -------- | ------------ | ------- |
| 第 0 层 | 1464 请求/秒 | 68 毫秒  | 96 毫秒  | 1506 字节    | 1353 MB |

由于我还没有描述我的基准测试方法，这些结果本身是没有意义的，我们无法判断这是“好”还是“坏”的性能。这没关系，因为我们不关心具体的数字，我们将使用这些结果作为基线，与所有后续实现进行比较。每个服务器都在相同的环境中进行测试，因此相对比较是准确的。

关于异常高的内存使用量，这是因为我在“集群模式”下运行 Node.js，该模式根据测试机器上的 12 个 CPU 核心生成了 12 个进程，每个进程都是一个独立的 Node.js 实例，这就是为什么即使我们有一个非常简单的服务器，它也会占用 1300+ MB 的内存。JS 是单线程的，所以如果我们希望 Node.js 服务器充分利用多核 CPU，就必须这样做。

### 第 1 层：Rust CLI 工具

对于此策略，我们将热点函数重写为 Rust，将其编译为独立的 CLI 工具，然后从主服务器调用它。

首先，我们将函数重写为 Rust：

```rust
/** qr_lib/lib.rs **/

use qrcode::{QrCode, EcLevel};
use image::Luma;
use image::codecs::png::{CompressionType, FilterType, PngEncoder};

pub type StdErr = Box<dyn std::error::Error>;

pub fn generate_qr_code(text: &str) -> Result<Vec<u8>, StdErr> {
    let qr = QrCode::with_error_correction_level(text, EcLevel::L)?;
    let img_buf = qr.render::<Luma<u8>>()
        .min_dimensions(200, 200)
        .build();
    let mut encoded_buf = Vec::with_capacity(512);
    let encoder = PngEncoder::new_with_quality(
        &mut encoded_buf,
        // 这些选项在测试中提供了速度和压缩之间的最佳平衡
        CompressionType::Default,
        FilterType::NoFilter,
    );
    img_buf.write_with_encoder(encoder)?;
    Ok(encoded_buf)
}
```

然后我们将其制作成 CLI 工具：

```rust
/** qr_cli/main.rs **/

use std::{env, process};
use std::io::{self, BufWriter, Write};
use qr_lib::StdErr;

fn main() -> Result<(), StdErr> {
    let mut args = env::args();
    if args.len() != 2 {
        eprintln!("Usage: qr-cli <text>");
        process::exit(1);
    }

    let text = args.nth(1).unwrap();
    let qr_png = qr_lib::generate_qr_code(&text)?;

    let stdout = io::stdout();
    let mut handle = BufWriter::new(stdout.lock());
    handle.write_all(&qr_png)?;

    Ok(())
}
```

我们可以像这样使用这个 CLI：

```bash
qr-cli https://youtu.be/cE0wfjsybIQ?t=74 > crab-rave.png
```

这将正确生成以下二维码 PNG：

![QR code for crab rave youtube video](../../../assets/crab-rave-qr-code.png)

现在，我们更新主服务器中的热点函数以调用此 CLI：

```js
const { spawn } = require("child_process");
const path = require("path");
const qrCliPath = path.resolve(__dirname, "./qr-cli");

/**
 * @param {string} text - 要编码的文本
 * @returns {Promise<Buffer>} - 二维码
 */
module.exports = function generateQrCode(text) {
  return new Promise((resolve, reject) => {
    const qrCli = spawn(qrCliPath, [text]);
    const qrCodeData = [];
    qrCli.stdout.on("data", (data) => {
      qrCodeData.push(data);
    });
    qrCli.stderr.on("data", (data) => {
      reject(new Error(`error generating qr code: ${data}`));
    });
    qrCli.on("error", (err) => {
      reject(new Error(`failed to start qr-cli ${err}`));
    });
    qrCli.on("close", (code) => {
      if (code === 0) {
        resolve(Buffer.concat(qrCodeData));
      } else {
        reject(new Error("qr-cli exited unsuccessfully"));
      }
    });
  });
};
```

现在让我们看看这个更改对性能的影响：

绝对测量值

| 层级    | 吞吐量          | 平均延迟   | p99 延迟   | 平均响应大小 | 内存       |
| ------- | --------------- | ---------- | ---------- | ------------ | ---------- |
| 第 0 层 | 1464 请求/秒    | 68 毫秒    | 96 毫秒    | 1506 字节    | 1353 MB    |
| 第 1 层 | 2572 请求/秒 🥇 | 39 毫秒 🥇 | 78 毫秒 🥇 | 778 字节 🥇  | 1240 MB 🥇 |

相对测量值

| 层级    | 吞吐量   | 平均延迟 | p99 延迟 | 平均响应大小 | 内存     |
| ------- | -------- | -------- | -------- | ------------ | -------- |
| 第 0 层 | 1.00x    | 1.00x    | 1.00x    | 1.00x        | 1.00x    |
| 第 1 层 | 1.76x 🥇 | 0.57x 🥇 | 0.82x 🥇 | 0.52x 🥇     | 0.92x 🥇 |

哇，我没想到吞吐量会增加 76%！这是一个非常粗糙的策略，所以看到它如此有效真是有趣。平均响应大小也从 1506 字节减少到 778 字节，Rust 库中的压缩算法一定比 JS 库中的更好。我们每秒处理的请求数显著增加，返回的响应大小显著减小，所以我认为这是一个很好的结果。

### 第 2 层：Rust Wasm 模块

对于此策略，我们将 Rust 函数编译为 Wasm 模块，然后使用 Wasm 运行时从主服务器加载并运行它。以下是一些不同语言的 Wasm 运行时链接：

| 语言       | Wasm 运行时                                              | GitHub 星数 |
| ---------- | -------------------------------------------------------- | ----------- |
| JavaScript | 内置                                                     | -           |
| 多种语言   | [Wasmer](https://github.com/wasmerio/wasmer)             | 19.2K+      |
| 多种语言   | [Wasmtime](https://github.com/bytecodealliance/wasmtime) | 15.7K+      |
| 多种语言   | [WasmEdge](https://github.com/WasmEdge/WasmEdge)         | 8.7K+       |
| 多种语言   | [wasm3](https://github.com/wasm3/wasm3)                  | 7.4k+       |
| Go         | [Wazero](https://github.com/tetratelabs/wazero)          | 5.1k+       |
| 多种语言   | [Extism](https://github.com/extism/extism)               | 4.6k+       |
| Java       | [Chicory](https://github.com/dylibso/chicory)            | 560+        |

由于我们正在集成到 Node.js 服务器中，让我们使用 `wasm-bindgen` 来生成 Rust Wasm 代码和 JS 代码之间交互的粘合代码。

以下是更新后的 Rust 代码：

```rust
/** qr_wasm_bindgen/lib.rs **/

use wasm_bindgen::prelude::*;

#[wasm_bindgen(js_name = generateQrCode)]
pub fn generate_qr_code(text: &str) -> Result<Vec<u8>, JsError> {
    qr_lib::generate_qr_code(text)
        .map_err(|e| JsError::new(&e.to_string()))
}
```

使用 `wasm-pack` 编译该代码后，我们可以将构建的资产复制到 Node.js 服务器中，并在热点函数中使用它们，如下所示：

```js
const wasm = require("./qr_wasm_bindgen.js");

/**
 * @param {string} text - 要编码的文本
 * @returns {Buffer} - 二维码
 */
module.exports = function generateQrCode(text) {
  return Buffer.from(wasm.generateQrCode(text));
};
```

更新后的基准测试：

绝对测量值

| 层级    | 吞吐量          | 平均延迟   | p99 延迟   | 平均响应大小 | 内存       |
| ------- | --------------- | ---------- | ---------- | ------------ | ---------- |
| 第 0 层 | 1464 请求/秒    | 68 毫秒    | 96 毫秒    | 1506 字节    | 1353 MB    |
| 第 1 层 | 2572 请求/秒    | 39 毫秒    | 78 毫秒    | 778 字节 🥇  | 1240 MB 🥇 |
| 第 2 层 | 2978 请求/秒 🥇 | 34 毫秒 🥇 | 63 毫秒 🥇 | 778 字节 🥇  | 1286 MB    |

相对测量值

| 层级    | 吞吐量   | 平均延迟 | p99 延迟 | 平均响应大小 | 内存     |
| ------- | -------- | -------- | -------- | ------------ | -------- |
| 第 0 层 | 1.00x    | 1.00x    | 1.00x    | 1.00x        | 1.00x    |
| 第 1 层 | 1.76x    | 0.57x    | 0.82x    | 0.52x 🥇     | 0.92x 🥇 |
| 第 2 层 | 2.03x 🥇 | 0.50x 🥇 | 0.66x 🥇 | 0.52x 🥇     | 0.95x    |

使用 Wasm 使我们的吞吐量比基线翻了一番！然而，与之前调用 CLI 工具的原始策略相比，性能提升比我预期的要小。

总之，虽然 `wasm-bindgen` 是一个优秀的 JS 到 Rust Wasm 绑定生成器，但对于其他语言（如 Python、Java、Go、PHP、Ruby 等）没有等效的工具。我不想让这些人失望，所以我会解释如何手动编写绑定。免责声明：代码会变得很丑陋，所以除非你真的对底层实现感兴趣，否则你可以跳过下一节。

#### 手动编写 Wasm 绑定

Wasm 的一个有趣之处在于它只支持四种数据类型：`i32`、`i64`、`f32` 和 `f64`。然而，对于我们的用例，我们需要将字符串从主机传递给 Wasm 函数，并且 Wasm 函数需要返回一个数组给主机。Wasm 没有字符串或数组。那么我们该如何解决这个问题呢？

答案在于以下几点：

- Wasm 模块的内存由 Wasm 实例和主机共享，两者都可以读取和修改它。
- Wasm 模块最多只能请求 4GB 的内存，因此每个可能的内存地址都可以编码为 `i32`，因此该数据类型也用作内存地址指针。

如果我们想将字符串从主机传递给 Wasm 函数，主机必须直接将字符串写入 Wasm 模块的内存，然后传递两个 `i32` 给 Wasm 函数：一个指向字符串的内存地址，另一个指定字符串的字节长度。

如果我们想将数组从 Wasm 函数传递给主机，主机首先需要为 Wasm 函数提供一个 `i32`，指向数组应写入的内存地址，然后当 Wasm 函数完成时，它返回一个 `i32`，表示写入的字节数。

然而，现在我们有一个新问题：当主机写入 Wasm 模块的内存时，如何确保它不会覆盖 Wasm 模块正在使用的内存？为了让主机能够安全地写入内存，它必须首先请求 Wasm 模块为其分配空间。

好的，现在有了所有这些背景知识，我们终于可以看这段代码并真正理解它了：

```rust
/** qr_wasm/lib.rs **/

use std::{alloc::Layout, mem, slice, str};

// 主机调用此函数以分配空间，以便安全地写入数据
#[no_mangle]
pub unsafe extern "C" fn alloc(size: usize) -> *mut u8 {
    let layout = Layout::from_size_align_unchecked(
        size * mem::size_of::<u8>(),
        mem::align_of::<usize>(),
    );
    std::alloc::alloc(layout)
}

// 在分配文本缓冲区和输出缓冲区后，主机调用此函数以生成二维码 PNG
#[no_mangle]
pub unsafe extern "C" fn generateQrCode(
    text_ptr: *const u8,
    text_len: usize,
    output_ptr: *mut u8,
    output_len: usize,
) -> usize {
    // 从内存中读取文本，主机已将其写入
    let text_slice = slice::from_raw_parts(text_ptr, text_len);
    let text = str::from_utf8_unchecked(text_slice);

    let qr_code = match qr_lib::generate_qr_code(text) {
        Ok(png_data) => png_data,
        // 错误：无法生成二维码
        Err(_) => return 0,
    };

    if qr_code.len() > output_len {
        // 错误：输出缓冲区太小
        return 0;
    }

    // 将生成的二维码 PNG 写入输出缓冲区，主机将在函数返回后从中读取
    let output_slice = slice::from_raw_parts_mut(output_ptr, qr_code.len());
    output_slice.copy_from_slice(&qr_code);

    // 返回写入的 PNG 数据长度
    qr_code.len()
}
```

编译此 Wasm 模块后，我们可以从 JS 中使用它，如下所示：

```js
const path = require("path");
const fs = require("fs");

// 获取 Wasm 文件
const qrWasmPath = path.resolve(__dirname, "./qr_wasm.wasm");
const qrWasmBinary = fs.readFileSync(qrWasmPath);

// 实例化 Wasm 模块
const qrWasmModule = new WebAssembly.Module(qrWasmBinary);
const qrWasmInstance = new WebAssembly.Instance(qrWasmModule, {});

// JS 字符串是 UTF16，但我们需要将其重新编码为 UTF8 再传递给 Wasm 模块
const textEncoder = new TextEncoder();

// 告诉 Wasm 模块为我们分配两个缓冲区：
// - 第一个缓冲区：输入缓冲区，我们将写入 UTF8 字符串，generateQrCode 函数将从中读取
// - 第二个缓冲区：输出缓冲区，generateQrCode 函数将写入二维码 PNG 字节，我们将从中读取
const textMemLen = 1024;
const textMemOffset = qrWasmInstance.exports.alloc(textMemLen);
const outputMemLen = 4096;
const outputMemOffset = qrWasmInstance.exports.alloc(outputMemLen);

/**
 * @param {string} text - 要编码的文本
 * @returns {Buffer} - 二维码
 */
module.exports = function generateQrCode(text) {
  // 将 UTF16 JS 字符串转换为 Uint8Array
  let encodedText = textEncoder.encode(text);
  let encodedTextLen = encodedText.length;

  // 将字符串写入 Wasm 内存
  qrWasmMemory = new Uint8Array(qrWasmInstance.exports.memory.buffer);
  qrWasmMemory.set(encodedText, textMemOffset);

  const wroteBytes = qrWasmInstance.exports.generateQrCode(
    textMemOffset,
    encodedTextLen,
    outputMemOffset,
    outputMemLen
  );

  if (wroteBytes === 0) {
    throw new Error("failed to generate qr");
  }

  // 从 Wasm 内存中读取二维码 PNG 字节并返回
  return Buffer.from(
    qrWasmInstance.exports.memory.buffer,
    outputMemOffset,
    wroteBytes
  );
};
```

这就是当我们使用像 `wasm-bindgen` 这样的库时，底层生成的代码。总之，我对其进行了基准测试，手写绑定的性能与生成的绑定在此情况下的性能几乎相同。

所以，编写主机和 Wasm 模块之间的粘合代码显然不是一件有趣的事情。幸运的是，积极参与 Wasm 规范制定的人们已经意识到这一点，他们目前正在制定“组件模型 Component Model”提案，该提案将标准化一种称为 WIT（Wasm 接口类型, Wasm Interface Type）的 IDL（接口定义语言, Interface Definition Language），绑定生成器和 Wasm 运行时可以围绕它构建。

目前，有一个名为 `wit-bindgen` 的 Rust 项目，它可以根据 WIT 文件为用 Rust 编写的 Wasm 模块生成粘合代码，但你需要一个单独的工具来生成主机粘合代码，比如 `jco`，它可以根据 Wasm 和 WIT 文件生成 JS 粘合代码。

使用 `wit-bingen` + `jco` 将给你类似于使用 `wasm-bindgen` 的结果，但希望未来会有更多针对其他语言的 WIT 主机绑定生成器，这样 Python、Java、Go、PHP、Ruby 等程序员就能拥有像 `wasm-bindgen` 对 JS 程序员那样方便易用的解决方案。

### 第 3 层：Rust 原生函数

对于此策略，我们将用 Rust 编写函数，将其编译为原生代码，然后从主机运行时加载并执行它。以下是各种语言的 Rust 绑定生成器库表：

| 语言       | Rust 绑定生成器                                           | GitHub 星数 |
| ---------- | --------------------------------------------------------- | ----------- |
| Python     | [pyo3](https://github.com/pyo3/pyo3)                      | 12.7k+      |
| JavaScript | [napi-rs](https://github.com/napi-rs/napi-rs)             | 6.3k+       |
| Erlang     | [rustler](https://github.com/rusterlium/rustler)          | 4.4k+       |
| 多种语言   | [uniffi-rs](https://github.com/mozilla/uniffi-rs)         | 3k+         |
| Java       | [jni-rs](https://github.com/jni-rs/jni-rs)                | 1.3k+       |
| Ruby       | [rutie](https://github.com/danielpclark/rutie)            | 970+        |
| PHP        | [ext-php-rs](https://github.com/davidcole1340/ext-php-rs) | 610+        |
| 多种语言   | [diplomat](https://github.com/rust-diplomat/diplomat)     | 560+        |

由于我们的示例服务器是用 JS 编写的，我们将使用 `napi-rs`。以下是 Rust 代码：

```rust
use napi::bindgen_prelude::*;
use napi_derive::napi;

#[napi]
pub fn generate_qr_code(text: String) -> Result<Vec<u8>, Status> {
    qr_lib::generate_qr_code(&text)
        .map_err(|e| Error::from_reason(e.to_string()))
}
```

我喜欢它的简单性。在前一节中从头开始编写 Wasm 模块后，我对实现和维护绑定生成器库的人们有了新的欣赏和尊重。

构建上述代码后，我们可以从 Node.js 中使用它，如下所示：

```js
const native = require("./qr_napi.node");

/**
 * @param {string} text - 要编码的文本
 * @returns {Buffer} - 二维码
 */
module.exports = function generateQrCode(text) {
  return Buffer.from(native.generateQrCode(text));
};
```

现在让我们看看这个家伙能否飞起来：

绝对测量值

| 层级    | 吞吐量          | 平均延迟   | p99 延迟   | 平均响应大小 | 内存       |
| ------- | --------------- | ---------- | ---------- | ------------ | ---------- |
| 第 0 层 | 1464 请求/秒    | 68 毫秒    | 96 毫秒    | 1506 字节    | 1353 MB    |
| 第 1 层 | 2572 请求/秒    | 39 毫秒    | 78 毫秒    | 778 字节 🥇  | 1240 MB 🥇 |
| 第 2 层 | 2978 请求/秒    | 34 毫秒    | 63 毫秒    | 778 字节 🥇  | 1286 MB    |
| 第 3 层 | 5490 请求/秒 🥇 | 18 毫秒 🥇 | 37 毫秒 🥇 | 778 字节 🥇  | 1309 MB    |

相对测量值

| 层级    | 吞吐量   | 平均延迟 | p99 延迟 | 平均响应大小 | 内存     |
| ------- | -------- | -------- | -------- | ------------ | -------- |
| 第 0 层 | 1.00x    | 1.00x    | 1.00x    | 1.00x        | 1.00x    |
| 第 1 层 | 1.76x    | 0.57x    | 0.82x    | 0.52x 🥇     | 0.92x 🥇 |
| 第 2 层 | 2.03x    | 0.50x    | 0.66x    | 0.52x 🥇     | 0.95x    |
| 第 3 层 | 3.75x 🥇 | 0.26x 🥇 | 0.39x 🥇 | 0.52x 🥇     | 0.97x    |

事实证明，原生代码非常快！我们的吞吐量几乎比基线增加了四倍，比 Wasm 实现增加了一倍。

### 第 4 层：Rust 重写

在此策略中，我们将用 Rust 重写主机服务器。诚然，这对于大多数现实世界的情况来说是不切实际的，因为服务器代码库通常有 10 万行以上的代码。在这些情况下，我们可以只重写主机服务器的一部分。如今，大多数人在后端运行的所有内容都位于反向代理之后，因此部署一个新的 Rust 服务器并修改反向代理配置以将一些请求路由到 Rust 服务器并不会给许多人的后端设置带来太多额外的操作开销。

所以，这是用 Rust 重写的服务器：

```rust
/** qr-server/main.rs **/

use std::process;
use axum::{
    extract::Query,
    http::{header, StatusCode},
    response::{IntoResponse, Response},
    routing::get,
    Router,
};

#[derive(serde::Deserialize)]
struct TextParam {
    text: String,
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/qrcode", get(handler));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:42069")
        .await
        .unwrap();
    println!(
        "server {} listening on {}",
        process::id(),
        listener.local_addr().unwrap(),
    );
    axum::serve(listener, app).await.unwrap();
}

async fn handler(
    Query(param): Query<TextParam>
) -> Result<Response, (StatusCode, &'static str)> {
    if param.text.len() > 512 {
        return Err((
            StatusCode::BAD_REQUEST,
            "text must be <= 512 bytes"
        ));
    }
    match qr_lib::generate_qr_code(&param.text) {
        Ok(bytes) => Ok((
            [(header::CONTENT_TYPE, "image/png"),],
            bytes,
        ).into_response()),
        Err(_) => Err((
            StatusCode::INTERNAL_SERVER_ERROR,
            "failed to generate qr code"
        )),
    }
}
```

让我们看看它是否名副其实：

绝对测量值

| 层级    | 吞吐量          | 平均延迟   | p99 延迟   | 平均响应大小 | 内存     |
| ------- | --------------- | ---------- | ---------- | ------------ | -------- |
| 第 0 层 | 1464 请求/秒    | 68 毫秒    | 96 毫秒    | 1506 字节    | 1353 MB  |
| 第 1 层 | 2572 请求/秒    | 39 毫秒    | 78 毫秒    | 778 字节 🥇  | 1240 MB  |
| 第 2 层 | 2978 请求/秒    | 34 毫秒    | 63 毫秒    | 778 字节 🥇  | 1286 MB  |
| 第 3 层 | 5490 请求/秒    | 18 毫秒    | 37 毫秒    | 778 字节 🥇  | 1309 MB  |
| 第 4 层 | 7212 请求/秒 🥇 | 14 毫秒 🥇 | 27 毫秒 🥇 | 778 字节 🥇  | 13 MB 🥇 |

相对测量值

| 层级    | 吞吐量   | 平均延迟 | p99 延迟 | 平均响应大小 | 内存     |
| ------- | -------- | -------- | -------- | ------------ | -------- |
| 第 0 层 | 1.00x    | 1.00x    | 1.00x    | 1.00x        | 1.00x    |
| 第 1 层 | 1.76x    | 0.57x    | 0.82x    | 0.52x 🥇     | 0.92x    |
| 第 2 层 | 2.03x    | 0.50x    | 0.66x    | 0.52x 🥇     | 0.95x    |
| 第 3 层 | 3.75x    | 0.26x    | 0.39x    | 0.52x 🥇     | 0.97x    |
| 第 4 层 | 4.93x 🥇 | 0.21x 🥇 | 0.28x 🥇 | 0.52x 🥇     | 0.01x 🥇 |

这不是打字错误。Rust 服务器在处理 7200+ 请求/秒时，真的只使用了 13 MB 的内存。我认为它确实名副其实！

## 总结

我认为所有这些策略都很好，但第 3 层脱颖而出，性价比最高。如果你可以使用现成的绑定生成器库，那么用 Rust 编写原生函数非常容易，并且它可以对性能产生深远的影响。

第 3 层最困难的部分可能是如果你还不了解 Rust，那么学习 Rust 可能会有些困难，但如果你处于这种情况，你应该阅读 [2024 年学习 Rust](./learning-rust-in-2024.md)，它将帮助你弄清楚如何开始。

## 讨论

在以下平台讨论本文：

- [Github](https://github.com/pretzelhammer/rust-blog/discussions/87)
- [official Rust users forum](https://users.rust-lang.org/t/using-rust-in-non-rust-servers-to-improve-performance/120121)
- [rust subreddit](https://www.reddit.com/r/rust/comments/1gabrdh/using_rust_in_nonrust_servers_to_improve/)
- [Hackernews](https://news.ycombinator.com/item?id=41941451)

## 延伸阅读

- [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md)
- [Tour of Rust's Standard Library Traits](./tour-of-rusts-standard-library-traits.md)
- [Beginner's Guide to Concurrent Programming: Coding a Multithreaded Chat Server using Tokio](./chat-server.md)
- [Learning Rust in 2024](./learning-rust-in-2024.md)
- [Sizedness in Rust](./sizedness-in-rust.md)
- [RESTful API in Sync & Async Rust](./restful-api-in-sync-and-async-rust.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](./too-many-brainfuck-compilers.md)

## 通知

通过以下方式在新博客文章发布时获得通知：

- 订阅此仓库的 [releases RSS feed](https://github.com/pretzelhammer/rust-blog/releases.atom) 或
- 关注此仓库的 releases (点击 `Watch` → 点击 `Custom` → 选择 `Releases` → 点击 `Apply`)
