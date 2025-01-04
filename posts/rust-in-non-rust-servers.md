

# Using Rust in Non-Rust Servers to Improve Performance
_22 October 2024 · #rust · #wasm · #ffi · #performance_



**Table of contents**
- [Intro](#intro)
- [The strategies](#the-strategies)
    - [Tier 0: No Rust](#tier-0-no-rust)
    - [Tier 1: Rust CLI Tool](#tier-1-rust-cli-tool)
    - [Tier 2: Rust Wasm Module](#tier-2-rust-wasm-module)
        - [Wasm bindings by hand](#wasm-bindings-by-hand)
    - [Tier 3: Rust Native Function](#tier-3-rust-native-function)
    - [Tier 4: Rust Rewrite](#tier-4-rust-rewrite)
- [Concluding thoughts](#concluding-thoughts)
- [Discuss](#discuss)
- [Further reading](#further-reading)
- [Notifications](#notifications)



## Intro

In this article I'll discuss different strategies for incrementally adding Rust into a server written in another language, such as JavaScript, Python, Java, Go, PHP, Ruby, etc. The main reason why you'd want to do this is because you've profiled your server, identified a hot function which is not meeting your performance requirements because it's bottlenecked by the CPU, and the usual techniques of memoizing the function or improving its algorithm wouldn't be feasible or effective in this situation for whatever reason. You have come to the conclusion that it would be worth investigating swapping out the function implementation for something written in a more CPU-efficient language, like Rust. Great, then this is definitely the article for you.

The strategies are ordered in tiers, where "tier" is short for "tier of Rust adoption." The first tier would be not using Rust at all. The last tier would be rewriting the entire server in Rust.

The example server which we'll be applying and benchmarking the strategies on will be implemented in JS, running on the Node.js runtime. The strategies can be generalized to any other language or runtime though.

> [!NOTE]
> The full source code for every example in this article can be found in [this repository](https://github.com/pretzelhammer/using-rust-in-non-rust-servers).



## The strategies



### Tier 0: No Rust

Let's say we have a Node.js server with an HTTP endpoint that takes a string of text as a query parameter and returns a 200px by 200px PNG image of the text encoded as a QR code.

Here's what the server code would look like:

```js
const express = require('express');
const generateQrCode = require('./generate-qr.js');

const app = express();
app.get('/qrcode', async (req, res) => {
    const { text } = req.query;

    if (!text) {
        return res.status(400).send('missing "text" query param');
    }

    if (text.length > 512) {
        return res.status(400).send('text must be <= 512 bytes');
    }

    try {
        const qrCode = await generateQrCode(text);
        res.setHeader('Content-Type', 'image/png');
        res.send(qrCode);
    } catch (err) {
        res.status(500).send('failed generating QR code');
    }
});

app.listen(42069, '127.0.0.1');
```

And here's what the hot function would look like:

```js
const QRCode = require('qrcode');

/**
 * @param {string} text - text to encode
 * @returns {Promise<Buffer>|Buffer} - qr code
 */
module.exports = function generateQrCode(text) {
    return QRCode.toBuffer(text, {
        type: 'png',
        errorCorrectionLevel: 'L',
        width: 200,
        rendererOpts: {
            // these options were chosen since
            // they offered the best balance
            // between speed and compression
            // during testing
            deflateLevel: 9, // 0 - 9
            deflateStrategy: 3, // 1 - 4
        },
    });
};
```

We can hit that endpoint by calling:

```
http://localhost:42069/qrcode?text=https://www.reddit.com/r/rustjerk/top/?t=all
```

Which will correctly produce this QR code PNG:

![QR code for rustjerk subreddit](../assets/rustjerk-subreddit-qr-code.png)

Anyway, let's throw tens of thousands of requests at this server for 30 seconds and see how it performs:

| Tier | Throughput | Avg Latency | p99 Latency | Avg Response | Memory |
|-|-|-|-|-|-|
| Tier 0 | 1464 req/sec | 68 ms | 96 ms | 1506 bytes | 1353 MB |

Since I haven't described my benchmarking methodology these results are meaningless on their own, we can't say whether this is "good" or "bad" performance. That's okay, since we don't care about the absolute numbers, we're going to use these results as a baseline to compare all of the following implementations against. Every server is tested in the same environment so relative comparisons will be accurate.

Regarding the abnormally high memory usage, it's because I'm running Node.js in "cluster mode", which spawns 12 processes for each of the 12 CPU cores on my test machine, and each process is a standalone Node.js instance which is why it takes up 1300+ MB of memory even though we have a very simple server. JS is single-threaded so this is what we have to do if we want a Node.js server to make full use of a multi-core CPU.



### Tier 1: Rust CLI Tool

For this strategy we rewrite the hot function in Rust, compile it as a standalone CLI tool, and then call it from our host server.

Let's start by rewriting the function in Rust:

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
        // these options were chosen since
        // they offered the best balance
        // between speed and compression
        // during testing
        CompressionType::Default,
        FilterType::NoFilter,
    );
    img_buf.write_with_encoder(encoder)?;
    Ok(encoded_buf)
}
```

Then let's make it a CLI tool:

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

We can use this CLI like so:

```bash
qr-cli https://youtu.be/cE0wfjsybIQ?t=74 > crab-rave.png
```

Which correctly produces this QR code PNG:

![QR code for crab rave youtube video](../assets/crab-rave-qr-code.png)

Now let's update the hot function in our host server to call this CLI:

```js
const { spawn } = require('child_process');
const path = require('path');
const qrCliPath = path.resolve(__dirname, './qr-cli');

/**
 * @param {string} text - text to encode
 * @returns {Promise<Buffer>} - qr code
 */
module.exports = function generateQrCode(text) {
    return new Promise((resolve, reject) => {
        const qrCli = spawn(qrCliPath, [text]);
        const qrCodeData = [];
        qrCli.stdout.on('data', (data) => {
            qrCodeData.push(data);
        });
        qrCli.stderr.on('data', (data) => {
            reject(new Error(`error generating qr code: ${data}`));
        });
        qrCli.on('error', (err) => {
            reject(new Error(`failed to start qr-cli ${err}`));
        });
        qrCli.on('close', (code) => {
            if (code === 0) {
                resolve(Buffer.concat(qrCodeData));
            } else {
                reject(new Error('qr-cli exited unsuccessfully'));
            }
        });
    });
};
```

Now let's see how this change affected performance:

Absolute measurements

| Tier | Throughput | Avg Latency | p99 Latency | Avg Response | Memory |
|-|-|-|-|-|-|
| Tier 0 | 1464 req/sec | 68 ms | 96 ms | 1506 bytes | 1353 MB |
| Tier 1 | 2572 req/sec 🥇 | 39 ms 🥇 | 78 ms 🥇 | 778 bytes 🥇 | 1240 MB 🥇 |

Relative measurements

| Tier | Throughput | Avg Latency | p99 Latency | Avg Response | Memory |
|-|-|-|-|-|-|
| Tier 0 | 1.00x | 1.00x | 1.00x | 1.00x | 1.00x |
| Tier 1 | 1.76x 🥇 | 0.57x 🥇 | 0.82x 🥇 | 0.52x 🥇 | 0.92x 🥇 |

Wow, I was not expecting throughput to increase by 76%! This is a very caveman-brain strategy so it's funny to see that it was that effective. Average response size also halved from 1506 bytes to 778 bytes, the compression algo in the Rust library must be better than the one in the JS library. We're serving significantly more requests per second and returning significantly smaller responses, so I'd say this is a great result.



### Tier 2: Rust Wasm Module

For this strategy we'll compile the Rust function into a Wasm module, and then load and run it from the host server using a Wasm runtime. Some links to Wasm runtimes across different languages:

| Language | Wasm runtime | Github stars |
|-|-|-|
| JavaScript | built-in | - |
| Multiple | [Wasmer](https://github.com/wasmerio/wasmer) | 19.2K+ |
| Multiple | [Wasmtime](https://github.com/bytecodealliance/wasmtime) | 15.7K+ |
| Multiple | [WasmEdge](https://github.com/WasmEdge/WasmEdge) | 8.7K+ |
| Multiple | [wasm3](https://github.com/wasm3/wasm3) | 7.4k+ |
| Go | [Wazero](https://github.com/tetratelabs/wazero) | 5.1k+ |
| Multiple | [Extism](https://github.com/extism/extism) | 4.6k+ |
| Java | [Chicory](https://github.com/dylibso/chicory) | 560+ |

Since we're integrating into a Node.js server let's use `wasm-bindgen` to generate the glue code that our Rust Wasm code and our JS code will use to interact with each other.

Here's the updated Rust code:

```rust
/** qr_wasm_bindgen/lib.rs **/

use wasm_bindgen::prelude::*;

#[wasm_bindgen(js_name = generateQrCode)]
pub fn generate_qr_code(text: &str) -> Result<Vec<u8>, JsError> {
    qr_lib::generate_qr_code(text)
        .map_err(|e| JsError::new(&e.to_string()))
}
```

After compiling that code using `wasm-pack`, we can copy the built assets over to our Node.js server and use them in the hot function like this:

```js
const wasm = require('./qr_wasm_bindgen.js');

/**
 * @param {string} text - text to encode
 * @returns {Buffer} - QR code
 */
module.exports = function generateQrCode(text) {
    return Buffer.from(wasm.generateQrCode(text));
};
```

Updated benchmarks:

Absolute measurements

| Tier | Throughput | Avg Latency | p99 Latency | Avg Response | Memory |
|-|-|-|-|-|-|
| Tier 0 | 1464 req/sec | 68 ms | 96 ms | 1506 bytes | 1353 MB |
| Tier 1 | 2572 req/sec | 39 ms | 78 ms | 778 bytes 🥇 | 1240 MB 🥇 |
| Tier 2 | 2978 req/sec 🥇 | 34 ms 🥇 | 63 ms 🥇 | 778 bytes 🥇 | 1286 MB |

Relative measurements

| Tier | Throughput | Avg Latency | p99 Latency | Avg Response | Memory |
|-|-|-|-|-|-|
| Tier 0 | 1.00x | 1.00x | 1.00x | 1.00x | 1.00x |
| Tier 1 | 1.76x | 0.57x | 0.82x | 0.52x 🥇 | 0.92x 🥇 |
| Tier 2 | 2.03x 🥇 | 0.50x 🥇 | 0.66x 🥇 | 0.52x 🥇 | 0.95x |

Using Wasm doubled our throughput compared to the baseline! However the jump in performance compared to the earlier caveman-brain strategy of calling a CLI tool is smaller than I would have expected.

Anyway, while `wasm-bindgen` is an excellent JS to Rust Wasm binding generator there's no equivalent of it for other languages such as Python, Java, Go, PHP, Ruby, etc. I don't want to leave those folks out to dry, so I'll explain how to write the bindings by hand. Disclaimer: the code is going to get ugly, so unless you're really interested in seeing how the sausage is made you can just skip over the next section.



#### Wasm bindings by hand

The funny thing about Wasm is that it only supports four data types: `i32`, `i64`, `f32`, and `f64`. Yet for our use-case we need to pass a string from the host to a Wasm function, and the Wasm function needs to return an array to the host. Wasm doesn't have strings or arrays. So how are we supposed to solve this problem?

The answer hinges on having a couple insights:
- The Wasm module's memory is shared between the Wasm instance and the host, both can read and modify it.
- A Wasm module can only request up to 4GB of memory, so every possible memory address can be encoded as an `i32`, so this data type is also used as a memory address pointer.

If we want to pass a string from the host to a Wasm function the host has to directly write the string into the Wasm module's memory, and then pass two `i32`s to the Wasm function: one pointing to the string's memory address and another specifying the string's byte length.

And if we want to pass an array from a Wasm function to the host, the host first needs to provide the Wasm function an `i32` pointing to the memory address where the array should be written, and then when the Wasm function completes it returns an `i32` which represents the number of bytes that were written.

However, now we have a new problem: when the host writes to the Wasm module's memory, how can it ensure it doesn't overwrite memory that the Wasm module is using? For the host to be able to safely write to memory, it must first ask the Wasm module to allocate space for it.

Okay, now with all of that context out of the way we can finally look at this code and actually understand it:

```rust
/** qr_wasm/lib.rs **/

use std::{alloc::Layout, mem, slice, str};

// host calls this function to allocate space where
// it can safely write data to
#[no_mangle]
pub unsafe extern "C" fn alloc(size: usize) -> *mut u8 {
    let layout = Layout::from_size_align_unchecked(
        size * mem::size_of::<u8>(),
        mem::align_of::<usize>(),
    );
    std::alloc::alloc(layout)
}

// after allocating a text buffer and output buffer,
// host calls this function to generate the QR code PNG
#[no_mangle]
pub unsafe extern "C" fn generateQrCode(
    text_ptr: *const u8,
    text_len: usize,
    output_ptr: *mut u8,
    output_len: usize,
) -> usize {
    // read text from memory, where it was written to by the host
    let text_slice = slice::from_raw_parts(text_ptr, text_len);
    let text = str::from_utf8_unchecked(text_slice);

    let qr_code = match qr_lib::generate_qr_code(text) {
        Ok(png_data) => png_data,
        // error: unable to generate QR code
        Err(_) => return 0,
    };

    if qr_code.len() > output_len {
        // error: output buffer is too small
        return 0;
    }

    // write generated QR code PNG to output buffer,
    // where the host will read it from after this
    // function returns
    let output_slice = slice::from_raw_parts_mut(output_ptr, qr_code.len());
    output_slice.copy_from_slice(&qr_code);

    // return written length of PNG data
    qr_code.len()
}
```

After compiling this Wasm module here's how we'd use it from JS:

```js
const path = require('path');
const fs = require('fs');

// fetch Wasm file
const qrWasmPath = path.resolve(__dirname, './qr_wasm.wasm');
const qrWasmBinary = fs.readFileSync(qrWasmPath);

// instantiate Wasm module
const qrWasmModule = new WebAssembly.Module(qrWasmBinary);
const qrWasmInstance = new WebAssembly.Instance(
    qrWasmModule,
    {},
);

// JS strings are UTF16, but we need to re-encode them
// as UTF8 before passing them to our Wasm module
const textEncoder = new TextEncoder();

// tell Wasm module to allocate two buffers for us:
// - 1st buffer: an input buffer which we'll
//               write UTF8 strings into that
//               the generateQrCode function
//               will read
// - 2nd buffer: an output buffer that the
//               generateQrCode function will
//               write QR code PNG bytes into
//               and that we'll read
const textMemLen = 1024;
const textMemOffset = qrWasmInstance.exports.alloc(textMemLen);
const outputMemLen = 4096;
const outputMemOffset = qrWasmInstance.exports.alloc(outputMemLen);

/**
 * @param {string} text - text to encode
 * @returns {Buffer} - QR code
 */
module.exports = function generateQrCode(text) {
    // convert UTF16 JS string to Uint8Array
    let encodedText = textEncoder.encode(text);
    let encodedTextLen = encodedText.length;

    // write string into Wasm memory
    qrWasmMemory = new Uint8Array(qrWasmInstance.exports.memory.buffer);
    qrWasmMemory.set(encodedText, textMemOffset);

    const wroteBytes = qrWasmInstance.exports.generateQrCode(
        textMemOffset,
        encodedTextLen,
        outputMemOffset,
        outputMemLen,
    );

    if (wroteBytes === 0) {
        throw new Error('failed to generate qr');
    }

    // read QR code PNG bytes from Wasm memory & return
    return Buffer.from(
        qrWasmInstance.exports.memory.buffer,
        outputMemOffset,
        wroteBytes,
    );
};
```

This is what is generated under-the-hood when we use a library like `wasm-bindgen`. Anyway, I benchmarked it and the performance of the handwritten bindings were virtually identical to the performance of the generated bindings in this case.

So writing Wasm glue code between the host and the guest is obviously not fun. Fortunately, the people actively contributing to the Wasm specification are aware of this and they are currently working on the "Component Model" proposal that will standardize an IDL (Interface Definition Language) called WIT (Wasm Interface Type) that binding generators and Wasm runtimes can be built around.

At the moment there's a Rust project called `wit-bindgen` that will generate the glue code for Wasm modules written in Rust given a WIT file, however you'd need a separate tool to generate the host glue code, like `jco`, which can generate JS glue code given a Wasm and WIT file.

Using `wit-bingen` + `jco` will give you a similar result to just using `wasm-bindgen`, but the hope is that more WIT host binding generators for other languages will be written in the future, so that Python, Java, Go, PHP, Ruby, etc programmers have a solution that's as convenience and easy-to-use as `wasm-bindgen` is today for JS programmers.



### Tier 3: Rust Native Function

For this strategy we're going to write the function in Rust, compile it to native code, and then load and execute it from the host runtime. Table of Rust bindgen libraries for various languages:

| Language | Rust bindgen | Github stars |
|-|-|-|
| Python | [pyo3](https://github.com/pyo3/pyo3) | 12.7k+ |
| JavaScript | [napi-rs](https://github.com/napi-rs/napi-rs) | 6.3k+ |
| Erlang | [rustler](https://github.com/rusterlium/rustler) | 4.4k+ |
| Multiple | [uniffi-rs](https://github.com/mozilla/uniffi-rs) | 3k+ |
| Java | [jni-rs](https://github.com/jni-rs/jni-rs) | 1.3k+ |
| Ruby | [rutie](https://github.com/danielpclark/rutie) | 970+ |
| PHP | [ext-php-rs](https://github.com/davidcole1340/ext-php-rs) | 610+ |
| Multiple | [diplomat](https://github.com/rust-diplomat/diplomat) | 560+ |

Since our example server is written in JS we're going to use `napi-rs`. Here's the Rust code:

```rust
use napi::bindgen_prelude::*;
use napi_derive::napi;

#[napi]
pub fn generate_qr_code(text: String) -> Result<Vec<u8>, Status> {
    qr_lib::generate_qr_code(&text)
        .map_err(|e| Error::from_reason(e.to_string()))
}
```

I love how easy it is. After writing a Wasm module from scratch in Rust in the preceding section I have a newfound appreciation and respect for people who implement and maintain binding generator libraries.

After building the code above here's how we'd use it from Node.js:

```js
const native = require('./qr_napi.node');

/**
 * @param {string} text - text to encode
 * @returns {Buffer} - QR code
 */
module.exports = function generateQrCode(text) {
    return Buffer.from(native.generateQrCode(text));
};
```

Now let's see if this puppy can fly:

Absolute measurements

| Tier | Throughput | Avg Latency | p99 Latency | Avg Response | Memory |
|-|-|-|-|-|-|
| Tier 0 | 1464 req/sec | 68 ms | 96 ms | 1506 bytes | 1353 MB |
| Tier 1 | 2572 req/sec | 39 ms | 78 ms | 778 bytes 🥇 | 1240 MB 🥇 |
| Tier 2 | 2978 req/sec | 34 ms | 63 ms | 778 bytes 🥇 | 1286 MB |
| Tier 3 | 5490 req/sec 🥇 | 18 ms 🥇 | 37 ms 🥇 | 778 bytes 🥇 | 1309 MB |

Relative measurements

| Tier | Throughput | Avg Latency | p99 Latency | Avg Response | Memory |
|-|-|-|-|-|-|
| Tier 0 | 1.00x | 1.00x | 1.00x | 1.00x | 1.00x |
| Tier 1 | 1.76x | 0.57x | 0.82x | 0.52x 🥇 | 0.92x 🥇 |
| Tier 2 | 2.03x | 0.50x | 0.66x | 0.52x 🥇 | 0.95x |
| Tier 3 | 3.75x 🥇 | 0.26x 🥇 | 0.39x 🥇 | 0.52x 🥇 | 0.97x |

It turns out native code is pretty fast! We've almost quadrupled our throughput compared to the baseline, and doubled it compared to the Wasm implementations.



### Tier 4: Rust Rewrite

In this strategy we're going to rewrite the host server in Rust. Admittedly, this is impractical for most real-world cases, where it's not unusual to see server codebases with 100k+ lines of code. In those situations we could instead only rewrite a subset of the host server. Nowadays most people run everything on their backend behind a reverse proxy anyway, so deploying a new Rust server and modifying the reverse proxy config to route some requests to the Rust server doesn't introduce that much additional operational overhead to many people's backend setups.

So here's the server rewritten in Rust:

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

Let's see if it lives up to the hype:

Absolute measurements

| Tier | Throughput | Avg Latency | p99 Latency | Avg Response | Memory |
|-|-|-|-|-|-|
| Tier 0 | 1464 req/sec | 68 ms | 96 ms | 1506 bytes | 1353 MB |
| Tier 1 | 2572 req/sec | 39 ms | 78 ms | 778 bytes 🥇 | 1240 MB |
| Tier 2 | 2978 req/sec | 34 ms | 63 ms | 778 bytes 🥇 | 1286 MB |
| Tier 3  | 5490 req/sec | 18 ms | 37 ms | 778 bytes 🥇 | 1309 MB |
| Tier 4 | 7212 req/sec 🥇 | 14 ms 🥇 | 27 ms 🥇 | 778 bytes 🥇 | 13 MB 🥇 |

Relative measurements

| Tier  | Throughput | Avg Latency | p99 Latency | Avg Response | Memory |
|-|-|-|-|-|-|
| Tier 0 | 1.00x | 1.00x | 1.00x | 1.00x | 1.00x |
| Tier 1 | 1.76x | 0.57x | 0.82x | 0.52x 🥇 | 0.92x |
| Tier 2 | 2.03x | 0.50x | 0.66x | 0.52x 🥇 | 0.95x |
| Tier 3 | 3.75x | 0.26x | 0.39x | 0.52x 🥇 | 0.97x |
| Tier 4 | 4.93x 🥇 | 0.21x 🥇 | 0.28x 🥇 | 0.52x 🥇 | 0.01x 🥇 |

That's not a typo. The Rust server really only used 13 MB of memory while serving 7200+ requests per second. I'd say it lived up to the hype for sure!



## Concluding thoughts

I think all of the strategies are good, but Tier 3 stands out as being the best bang for the buck. If you can use an off-the-shelf binding generator library then writing a native function in Rust is super easy and it can have a profound effect on performance.

The hardest part of Tier 3 is probably learning Rust if you don't know it already, but if you're in that boat you should read [Learning Rust in 2024](./learning-rust-in-2024.md) which will help you figure out how to begin.



## Discuss

Discuss this article on
- [Github](https://github.com/pretzelhammer/rust-blog/discussions/87)
- [official Rust users forum](https://users.rust-lang.org/t/using-rust-in-non-rust-servers-to-improve-performance/120121)
- [rust subreddit](https://www.reddit.com/r/rust/comments/1gabrdh/using_rust_in_nonrust_servers_to_improve/)
- [Hackernews](https://news.ycombinator.com/item?id=41941451)



## Further reading

- [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md)
- [Tour of Rust's Standard Library Traits](./tour-of-rusts-standard-library-traits.md)
- [Beginner's Guide to Concurrent Programming: Coding a Multithreaded Chat Server using Tokio](./chat-server.md)
- [Learning Rust in 2024](./learning-rust-in-2024.md)
- [Sizedness in Rust](./sizedness-in-rust.md)
- [RESTful API in Sync & Async Rust](./restful-api-in-sync-and-async-rust.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](./too-many-brainfuck-compilers.md)



## Notifications

Get notified when a new blog post gets published by
- Subscribing to this repo's [releases RSS feed](https://github.com/pretzelhammer/rust-blog/releases.atom) or
- Watching this repo's releases (click `Watch` → click `Custom` → select `Releases` → click `Apply`)
