# Learn Assembly by Writing Entirely Too Many Brainfuck Compilers

_01 November 2020 · #assembly · #compilers_

**Table of Contents**
- [Intro](#intro)
- [What is brainfuck?](#what-is-brainfuck)
- [Interpreting brainfuck](#interpreting-brainfuck)
- [What is assembly?](#what-is-assembly)
- [Intro to x86](#intro-to-x86)
- [Compiling brainfuck to x86](#compiling-brainfuck-to-x86)
- [Intro to ARM](#intro-to-arm)
- [Compiling brainfuck to ARM](#compiling-brainfuck-to-arm)
- [Intro to WebAssembly](#intro-to-webassembly)
- [Compiling brainfuck to WebAssembly](#compiling-brainfuck-to-webassembly)
- [Intro to LLVM](#intro-to-llvm)
- [Compiling brainfuck to LLVM](#compiling-brainfuck-to-llvm)
- [Optimization opportunities](#optimization-opportunities)
- [Concluding thoughts](#concluding-thoughts)
- [Discuss](#discuss)
- [Further Reading](#further-reading)



## Intro

Hey you! Have you ever wanted to become a _CPU Whisperer_? Me too! I'm a frontend web developer by trade but low-level assembly code and compilers have always fascinated me. I've procrastinated on learning either for a long time but after I recently picked up Rust and have been hanging out in a lot of online Rust communities it's given me the kick in the butt to dive in. Rustaceans use fancy words and acronyms like _auto-vectorization_, _inlining_, _alignment_, _padding_, _linking_, _custom allocators_, _endianness_, _system calls_, _LLVM_, _SIMD_, _ABI_, _TLS_ and I feel bad for not being able to follow the discussions because I don't know what any of that stuff is. All I know is that it vaguely relates to low-level assembly code somehow so I decided I'd learn assembly by writing entirely too many brainfuck compilers in Rust. How many is too many? Four! My compile targets are going to be x86, ARM, WebAssembly, and LLVM.

The goal of this article is to be easily-digestible for anyone who has a modest amount of programming experience under their belt, even if they've never written a single line of assembly before.

So why x86? x86 is not just an ISA but it is _the_ ISA. Most servers, desktop PCs, laptops, and home gaming consoles use x86 CPUs.

Why ARM? ARM is not just an ISA but it is _the other_ ISA. Most mobile phones, tablets, mobile gaming consoles, and microcontrollers use ARM CPUs. Also Apple announced they will be switching all their laptops and desktops from x86 to ARM CPUs in 2021 which seems like a Pretty Big Deal.

Why WebAssembly? WebAssembly has the potential to be the future of the web and also the future of containerized applications in general! Solomon Hykes, the creator of Docker, has tweeted _"If WASM + WASI existed in 2008, we wouldn't have needed to create Docker. That's how important it is. WebAssembly on the server is the future of computing. A standardized system interface was the missing link. Let's hope WASI is up to the task!"_

Why LLVM? LLVM because it can compile to x86, ARM, or WebAssembly. Also because many modern and successful programming languages like Rust and Swift compile to LLVM instead of to assembly directly.

Since all of the above targets go by many names here's a quick list of their aliases:
- 64-bit x86 is also called: x86_64, x64, AMD64
- 64-bit ARM is also called: aarch64, ARM64
- 32-bit WebAssembly is also called: wasm32
- LLVM is short for LLVM IR (Intermediate Representation)

If you'd like to play around with the code in this article yourself then you're in luck! The article comes with a [companion code repository](https://github.com/pretzelhammer/brainfuck_compilers) which contains all the code and instructions on how to run it. Following along using the companion code repository is completely optional and the article can be easily read without it.



## What is brainfuck?

Brainfuck is, oxymoronically, the most well-known esoteric programming language. It's fame largely comes from the fact it has the word "fuck" in its name but hobbyist compiler developers like it because it's a tiny language which makes it easy to write compilers for. Fun fact: people have written more brainfuck compilers than actual brainfuck programs. Fun fact: I did zero research for that previous fun fact but it's probably true.

Overview of brainfuck:

- Brainfuck programs have an implicit pointer, "the pointer", which is free to move around an array of 30k unsigned bytes, all initially set to 0.
- Decrementing the pointer below 0 or incrementing the pointer above 30k is undefined behavior.
- Decrementing a byte below 0 or incrementing a byte above 255 wraps its value.
- The newline character is read and written as the value 10.
- The EOF (End of File) character is read as the value 0.

Brainfuck commands:

| Command | Description |
|-|-|
| `>` | increment pointer |
| `<` | decrement pointer |
| `+` | increment current byte |
| `-` | decrement current byte |
| `.` | write current byte to stdout |
| `,` | read byte from stdin and store value in current byte |
| `[` | jump past matching `]` if current byte is zero |
| `]` | jump back to matching `[` if current byte is nonzero |
| any other character | ignore, treat as comment |

As is customary in introducing any new programming language, here's _"Hello world!"_ in brainfuck:

```bf
++++++++++[>+++++++>++++++++++>+++>+<<<<-]>++.>+.+++++++..+++.>++.<<+++++++++++++++.>.+++.------.--------.>+.>.
```



## Interpreting brainfuck

Let's write a quick brainfuck interpreter first. We're going to parse brainfuck programs into an `Vec<Inst>` where `Inst` is defined as:

```rust
struct Inst {
    idx: usize,         // index of instruction
    kind: InstKind,     // kind of instruction
    times: usize,       // run-length encoding of instruction
}

enum InstKind {
    IncPtr,
    DecPtr,
    IncByte,
    DecByte,
    WriteByte,
    ReadByte,
    // end_idx = index of instruction after matching LoopEnd
    LoopStart { end_idx: usize },
    // start_idx = index of instruction after matching LoopStart
    LoopEnd { start_idx: usize },
}
```

Parsing brainfuck programs into the above format, namely: keeping track of every instruction's run-length encoding and calculating the `start_idx` and `end_idx` of loop instructions ahead of time, will allow us to write a much more efficient interpreter and also produce much more efficient assembly from our compilers.

We'll skip going over the remaining brainfuck interpreter code as it's very unexciting. Let's get to the fun part and try interpreting some brainfuck programs!

> If you're following along using the [companion code repository](https://github.com/pretzelhammer/brainfuck_compilers) the command we'll be using to interpret brainfuck programs is `just interpret {{name}}` where `{{name}}` is the name of the brainfuck source file in the `./input` directory.

```sh
# prints "Hello world!"
> just interpret hello_world
Hello World!

# prints fibonacci numbers under 100
> just interpret fibonacci
1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89

# encrypts lines from stdin using rot13 cipher
> just interpret rot13
unencrypted text
harapelcgrq grkg
```

Cool, we have a working brainfuck interpreter. Let's start digging into assembly.



## What is assembly?

A slightly better first question is what is an ISA? ISA stands for Instruction Set Architecture. An ISA is an interface which CPUs can implement. The most popular ISAs today are x86_64 and aarch64. If we write code using x86_64 instructions then any CPU which implements the x86_64 ISA will be able to run that code. So is "assembly" the same thing as an ISA? Well, not quite. The short answer is that "assembly" is any syntax understood by an assembler. An assembler is a utility program that allows people to write machine-code in a more human-friendly way, like with comments, whitespace, and symbolic names for machine instructions. "Assembly" therefore is a thin layer of abstraction over an ISA offered by an assembler. The assembler we will be using to assemble all of our x86_64 and aarch64 programs will be the GNU Assembler, often abbreviated to GAS. We'll be using Intel syntax instead of the default AT&T syntax for x86_64 assembly because it's closer to ARM syntax for aarch64 assembly which makes it less jarring to switch between the two. If that last sentence made no sense to you don't worry you're in good company. Also, we'll be executing all the compiled binaries in a Linux environment so we'll be making direct Linux system calls in our assembly programs when necessary.



## Intro to x86

x86_64 is a register-based ISA. A register is a container where we can store data. We can store data in RAM too but RAM is very far from the CPU whereas registers are directly _in_ the CPU and are where the CPU does all of its actual work. All of the instructions in x86_64 operate on registers directly or indirectly in some way. There are many different kinds of registers: some store integers, some store floats, some store vectors of integers, some are general purpose and some have a special purpose, and some we can modify directly and others we can only modify indirectly (as a byproduct of certain instructions). For the purposes of this article the only registers we'll be using are `rax`, `rdi`, `rsi`, `rdx`, and `r12` which all store 64-bit integers.

Let's learn some instructions.

```s
mov <dest>, <src>       # dest <- src
```

`mov` moves something from `<src>` to `<dest>` where `<src>` can be a literal value, register, or memory address and `<dest>` can be a register or memory address.

```s
mov rax, 5              # store 5 in rax
mov rsi, rdi            # copy value in rdi to rsi
mov [r12], 15           # store 15 at the memory address in r12
```

The last instruction is actually illegal because it's ambiguous. In the first 2 examples it's clear we're working with 64-bit integers since we're using 64-bit registers as operands, however in the last example we're trying to store the value 15 in the memory address in `r12` but how "big" is the value "15"? Does it take up 1, 2, 4, or 8 bytes? We need to know how many bytes to write to memory, after all. We can clear up ambiguities by suffixing the ambiguous instruction with `b` (byte), `w` (word, 2 bytes), `l` (longword, 4 bytes), or `q` (quadword, 8 bytes). So we can fix the last instruction in any number of these ways:

```s
movb [r12], 15           # write 15 as 1 byte to memory address in r12
movw [r12], 15           # write 15 as 2 bytes to memory address in r12
movl [r12], 15           # write 15 as 4 bytes to memory address in r12
movq [r12], 15           # write 15 as 8 bytes to memory address in r12
```

Also, although it may have already been made obvious, if we want to dereference a memory address stored in a register or label we wrap it with square brackets `[]`.

```s
mov rax, r12            # copy value from r12 to rax
mov rax, [r12]          # copy value from memory address stored in r12 to rax
```

Some arithmetic instructions:

```s
add <dest>, <src>       # dest <- dest + src
sub <dest>, <src>       # dest <- dest - src
```

Comparing values:

```s
cmp <op1>, <op2>        # compare op1 to op2, set flags in special rflags register
```

Control flow:

```s
jmp <label>             # unconditional jump to <label>

# all instructions below are conditional jumps which check the flags in the rflags register

je <label>              # jump to <label> if equal 
jne <label>             # jump to <label> if not equal
jg <label>              # jump to <label> if greater than
jge <label>             # jump to <label> if greater than or equal to
jl <label>              # jump to <label> if less than
jle <label>             # jump to <label> if less than or equal to
```

Putting all of the above together into an example:

```s
mov rax, 5              # store 5 in rax
mov r12, 10             # store 10 in r12
add rax, r12            # rax <- rax + r12 = 15
cmp rax, r12            # set flags in rflags
jge RAX_IS_LARGER       # read flags in rflags, jump to RAX_IS_LARGER

R12_IS_LARGER:
# some instructions
jmp END

RAX_IS_LARGER:
# some other instructions

END:
# more instructions
```

The rules for how to use registers before, during, and after a function call for both the caller and callee is called a _Calling Convention_. The problem with calling conventions is that it seems everyone and their grandma has one. ISAs, Operating Systems, and programming languages which compile to assembly can each define their own different calling conventions. Luckily for us it's not possible to define functions in brainfuck so we don't have to get into the nitty gritty details of any calling conventions in this article.

To make a system call we use the `syscall` instruction after setting the system call number in `rax` and the system call arguments in the `rdi`, `rsi`, and `rdx` registers.

```s
# direct Linux system calls

mov rax, 60             # syscall number for exit(code)
mov rdi, 0              # exit code, 0 for success
syscall                 # make system call

mov rax, 0              # syscall number for read(fd, buf_adr, buf_len)
mov rdi, 0              # file descriptor for stdin
mov rsi, 1234           # memory address to some buffer
mov rdx, 1              # buffer's length in bytes
syscall                 # make system call
# syscall returns number of bytes read in rax

mov rax, 1              # syscall number for write(fd, buf_adr, buf_len)
mov rdi, 1              # file descriptor for stdout
mov rsi, 1234           # memory address to some buffer
mov rdx, 1              # buffer's length in bytes
syscall                 # make system call
# syscall returns number of bytes written in rax
```

We now know a handful of x86_64 instructions, enough to write a brainfuck compiler actually, and yet we still haven't put together a single complete program yet. This is where the assembler comes in. As mentioned above we'll be using GNU Assembler for all our x86_64 code. Let's take a look at a simple x86_64 program that just exits.

```s
# ./examples/x86_64/exit.s

# GNU Assembler, Intel syntax, x86_64 Linux

.data

.equ SYS_EXIT, 60
.equ EXIT_CODE, 0

.text

.global _start

_start:
    # exit(code)
    mov rax, SYS_EXIT
    mov rdi, EXIT_CODE
    syscall
```

Unpacking the new stuff:
- Words prefixed with a dot `.` are called _assembler directives_ and they direct the assembler on how to assemble our assembly.
- `.data` means _"Everything below this directive is program data."_
- `.equ <symbol>, <literal>` means declare a constant `<symbol>` equal to value `<literal>`.
- `.text` means _"Everything below this directive is program instructions."_
- Words suffixed by a colon `:` are labels and they can point to data or instructions. The `_start` label points to the first instruction of our program.
- `.global <label>` means _"Make `<label>` visible to the linker."_ The linker is a program which converts the assembled output of our assembler into an actual executable program, and it needs to know where our program begins, hence the `_start` label.

To make our program a little more exciting let's read a character from stdin, and if it's lowercase we'll make it uppercase, and if it's uppercase we'll make it lowercase, and then we'll write the switched case character to stdout.

```s
# ./examples/x86_64/switch_case.s

# GNU Assembler, Intel syntax, x86_64 Linux

.data

# exit(code)
.equ SYS_EXIT, 60
.equ EXIT_CODE, 0

# write(fd, buf_adr, buf_len)
.equ SYS_WRITE, 1
.equ STDOUT, 1

# read(fd, buf_adr, buf_len)
.equ SYS_READ, 0
.equ STDIN, 0

# ASCII code for lowercase 'a'
.equ ASCII_A, 97

# Quick ASCII refresher:
# 65 - 91 = 'A' - 'Z'
# 97 - 123 = 'a' - 'z'

# e.g.
# 'A' + 32 = 'a'
# 'a' - 32 = 'A' 
.equ CASE_DIFF, 32

# single byte in memory
CHAR:
    .byte 0

.text

.global _start

_start:
    # read(STDIN, CHAR, 1)
    mov rax, SYS_READ
    mov rdi, STDIN
    mov rsi, offset CHAR
    mov rdx, 1
    syscall

    cmpb [CHAR], ASCII_A        # if byte at CHAR is lowercase
    jge MAKE_UPPERCASE          # make it uppercase

MAKE_LOWERCASE:                 # else make it lowercase
    addb [CHAR], CASE_DIFF      # lowercase byte at CHAR
    jmp WRITE                   # then write it to stdout

MAKE_UPPERCASE:
    subb [CHAR], CASE_DIFF      # uppercase byte at CHAR

WRITE:                          # write byte to stdout
    # write(STDOUT, CHAR, 1)
    mov rax, SYS_WRITE
    mov rdi, STDOUT
    mov rsi, offset CHAR
    mov rdx, 1
    syscall

    # exit(EXIT_CODE)
    mov rax, SYS_EXIT
    mov rdi, EXIT_CODE
    syscall
```

Unpacking the new stuff:
- `.byte` allows us to define an array of bytes by writing a comma-separated list of integer literals. In the above program we only needed 1 byte.
- By default, GAS dereferences labels, so `mov rsi, CHAR` would copy the value `0` into `rsi`. However, we don't want to copy the value at `CHAR` but we want to copy the literal value of `CHAR` itself, i.e. its memory address. We can do this using the `offset` keyword, which we do in `mov rsi, offset CHAR`.

> If you're following along using the [companion code repository](https://github.com/pretzelhammer/brainfuck_compilers) the command we'll be using to compile and run x86_64 example programs is `just carx {{name}}` where `{{name}}` is the name of the x86_64 source file in the `./examples/x86_64` directory.

```sh
# reads char from stdin, switches its case, prints to stdout

> just carx switch_case
a
A
Exit code: 0

> just carx switch_case
A
a
Exit code: 0
```



## Compiling brainfuck to x86

Alright, first thing's first, we need a zero-initialized array of 30k bytes. Given what we learned in the previous section we could generate the following code with our compiler:

```s
.data

ARRAY:
    .byte 0, 0, 0, 0, 0, ... (29,995 more times)
```

However, even for a compiler generated solution, it looks pretty dumb. Luckily for us there's an easier way to define large amounts of zero-initialized data:

```s
.bss

.lcomm ARRAY, 30000
```

`.bss` is similar to `.data` in the sense that we define data items below it, but the main difference is we don't initialize the data items, we just declare their size, and they are automatically zero initialized for us. `.lcomm ARRAY, 30000` means, _"Make symbol `ARRAY` point to a zero-initialized array of 30k bytes."_

One last tiny decision we have to make is which register we'll be using to store our array pointer. There's a lot to choose from, but let's go with `r12` because it's a general-purpose callee-saved register which means if we make any function or system calls we're guaranteed those calls won't overwrite `r12`.

Now that we have that out of the way we can generate the header and footer boilerplate for any compiled brainfuck program:

```s
### header boilerplate ###

# GNU Assembler, Intel syntax, x86_64 Linux

.data

.equ SYS_EXIT, 60
.equ SUCCESS, 9

.equ SYS_WRITE, 1
.equ STDOUT, 1

.equ SYS_READ, 0
.equ STDIN, 0 

.bss

.lcomm ARRAY, 30000

.text

.global _start

_start:
    mov r12, offset ARRAY

###############################################
# actual compiled brainfuck program goes here #
###############################################

### footer boilerplate ###

    mov rax, SYS_EXIT
    mov rdi, SUCCESS
    syscall
```

Let's now map brainfuck commands to x86_64 instructions. We should also consider how we can coalesce multiple repeating commands into single instructions.

```s
# increment array pointer

# >
add r12, 1

# >>
add r12, 2

# decrement array pointer

# <
sub r12, 1

# <<
sub r12, 2

# increment byte at pointer

# +
addb [r12], 1

# ++
addb [r12], 2

# decrement byte at pointer

# -
subb [r12], 1

# --
subb [r12], 2

# read byte from stdin & store at pointer

# ,
mov rax, SYS_READ
mov rdi, STDIN
mov rsi, r12
mov rdx, 1
syscall

# ,,
mov rax, SYS_READ
mov rdi, STDIN
mov rsi, r12
mov rdx, 1
syscall
mov rax, SYS_READ
mov rdi, STDIN
mov rsi, r12
mov rdx, 1
syscall

# write byte at pointer to stdout

# .
mov rax, SYS_WRITE
mov rdi, STDOUT
mov rsi, r12
mov rdx, 1
syscall

# ..
mov rax, SYS_WRITE
mov rdi, STDOUT
mov rsi, r12
mov rdx, 1
syscall
mov rax, SYS_WRITE
mov rdi, STDOUT
mov rsi, r12
mov rdx, 1
syscall
```

Unfortunately there's no simple way to coalesce multiple `,` or `.` commands into less instructions than it takes to execute a single `,` or `.` because the registers we set up for the system calls can be overwritten by the system call procedures so we have to reset the registers again before every call.

```s
# loops

# [
cmpb [r12], 0
je LOOP_END_1
LOOP_START_0:

# ]
cmpb [r12], 0
jne LOOP_START_0
LOOP_END_1:
```

Generating matching labels for matching loops is a problem we already solved in our parser. To examine the simplest case, our parser will parse the following brainfuck program `[-]` like so:

```rust
[
    Inst {
        idx: 0,
        kind: InstKind::LoopStart { end_idx: 3 },
        times: 1,
    },
    Inst {
        idx: 1,
        kind: InstKind::DecByte,
        times: 1,
    },
    Inst {
        idx: 2,
        kind: InstKind::LoopEnd { start_idx: 1 },
        times: 1,
    },
]
```

Using the data available to us inside the `LoopStart` instruction we can generate the following labels:

- `LOOP_START_<idx>`
- `LOOP_END_<end_idx-1>`

Using the data available to us inside the `LoopEnd` instruction we can generate the following labels:

- `LOOP_END_<idx>`
- `LOOP_START_<start_idx-1>`

Following this label generation scheme we're guaranteed matching labels for matching loops.

```s
# loops

# [[
cmpb [r12], 0
je LOOP_END_1
LOOP_START_0:

# ]]
cmpb [r12], 0
jne LOOP_START_0
LOOP_END_1:
```

Multiple stacked loops is interesting because it's not any different than a single loop. If the current byte is zero it doesn't matter how many `[` we have in a row because it will jump past all of them to the outermost matching `]`. Also, if the current byte is nonzero then it doesn't matter how many `]` we have in a row because it'll jump back to the innermost matching `[`. Our parser handles the hard work of figuring out which jumps to make so our label generation scheme stays the same regardless of how many stacked loops we have in the source code.

And we're done, time to give our compiler a test drive.

> If you're following along using the [companion code repository](https://github.com/pretzelhammer/brainfuck_compilers) the command we'll be using to compile brainfuck programs to x86_64 and run them is `just carbx {{name}}` where `{{name}}` is the name of the brainfuck source file in the `./input` directory.

```sh
# prints "Hello world!"
> just carbx hello_world
Hello World!

# prints fibonacci numbers under 100
> just carbx fibonacci
1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89

# encrypts lines from stdin using rot13 cipher
> just carbx rot13
unencrypted text
harapelcgrq grkg
```

Everything works as expected. I'm curious how much faster the compiled programs are compared to the interpreter so I'll run a very unscientific and informal benchmark by timing how long it takes to interpret the most CPU-intensive brainfuck program `./input/mandelbrot.b` vs how long the x86_64 compiled version takes to execute.

```sh
> just benchmark mandelbrot

# program outputs omitted

# interpreted mandelbrot.b
4.95s user 0.01s system 99% cpu 4.960 total

# x86_64 compiled mandelbrot.b
real    0m1.214s
user    0m1.149s
sys     0m0.041s
```

Wow, not bad! Our compiled version runs over 4x as fast as the interpreted version. In my opinion this is an impressive improvement given how simplistic brainfuck programs are, we're just moving a single pointer around in a single array and adding or subtracting some bytes.



## Intro to ARM

Like x86_64, aarch64 is a register-based ISA. We'll be using the following 64-bit registers for our examples and for our compiler `x0`, `x1`, `x2`, `x8`, `x19`, and `x20`.

Let's learn some instructions.

```s
mov <dest>, <src>       // dest <- src
```

`mov` moves something from `<src>` to `<dest>` where `<src>` can be a literal value or register and `<dest>` is a register. As you may have noticed, unlike the x86_64 version of `mov`, the aarch64 version of `mov` cannot operate directly on memory. This is not just true for the `mov` instruction but is generally true for all aarch64 instructions. The only aarch64 instructions that can operate on memory are `ldr` and  `str` where `ldr` loads data from memory into registers and `str` stores data from registers into memory. So for example, this single x86_64 instruction:

```s
addq [r12], 100         // add 100 to the 8-byte integer stored at the memory address in r12
```

Would take three aarch64 instructions to express:

```s
ldr x20, [x19]          // load 8-byte integer stored at memory address in x19 into x20
add x20, x20, 100       // add 100 to integer in x20 and store in x20
str x20, [x19]          // store 8-byte integer in x20 to memory address in x19
```

This does not mean that x86_64 is 3x "faster" or "more efficient" than aarch64 because under-the-hood the x86_64 CPU still has to load the value from memory into a register before it can add to it and then it still has to store the updated value back into memory. x86_64 simply offers us the convenience of expressing these 3 operations in a single instruction and aarch64 does not. Similarly to x86_64, both `ldr` and `str` instructions can be appended with suffixes to disambiguate data size: `b` (byte), and `h` (halfword, 2 bytes). To load or store 4 bytes of data we refer to registers with a `w` (word, 4 bytes) prefix instead of the usual `x` (extended word, 8 bytes) prefix. Here's some examples:

```s
ldr x20, [x19]          // load 8 bytes from [x19] into x20
ldr w20, [x19]          // load 4 bytes from [x19] into x20
ldrh w20, [x19]         // load 2 bytes from [x19] into x20
ldrb w20, [x19]         // load 1 byte from [x19] into x20

str x20, [x19]          // store 8 bytes from x20 into [x19]
str w20, [x19]          // store 4 bytes from x20 into [x19]
strh w20, [x19]         // store 2 bytes from x20 into [x19]
strb w20, [x19]         // store 1 byte from x20 to [x19]
```

Important note: `x20` and `w20` are the same register, it's just that referring to it by `x20` accesses all 64 bits of the register and referring to it by `w20` accesses only the lower 32 bits of the register.

Some arithmetic instructions:

```s
add <dest>, <op1>, <op2>    // dest <- op1 + op2
sub <dest>, <op1>, <op2>    // dest <- op1 - op2
```

Comparing values:

```s
cmp <op1>, <op2>        // compares op1 to op2 and sets flags in the special NZCV register 
```

Control flow:

```s
b <label>               // unconditionally branch to <label>

// all instructions below are conditional and check the flags in the NZCV register

b.eq <label>            // branch to <label> if equal
b.ne <label>            // branch to <label> if not equal
b.gt <label>            // branch to <label> if greater than
b.ge <label>            // branch to <label> if greater than or equal
b.lt <label>            // branch to <label> is less than
b.le <label>            // branch to <label> if less than or equal
```

To make a system call we use the `svc 0` instruction after setting the system call number in `x8` and the system call arguments in the `x0`, `x1`, and `x2` registers:

```s
// direct Linux system calls

mov x8, 93              // syscall number for exit(code)
mov x0, 0               // exit code, 0 for success
svc 0                   // make system call

mov x8, 63              // syscall number for read(fd, buf_adr, buf_len)
mov x0, 0               // file descriptor for stdin
mov x1, 1234            // memory address to some buffer
mov x2, 1               // buffer's length in bytes
svc 0                   // make system call

mov x8, 64              // syscall number for write(fd, buf_adr, buf_len)
mov x0, 1               // file descriptor for stdout
mov x1, 1234            // memory address to some buffer
mov x2, 1               // buffer's length in bytes
svc 0                   // make system call
```

As you may have noticed, the syscall numbers for the same Linux system calls are different between x86_64 and aarch64. As far as I'm aware there's no good reason for this. Also, `svc` is short for _Supervisor Call_ and the `0` just needs to be there.

Here's our `switch_case.s` program from before except ported to aarch64:

```s
// ./examples/aarch64/switch_case.s

// GNU Assembler, ARM syntax, aarch64 Linux

.data

// exit(code)
.equ SYS_EXIT, 93
.equ EXIT_CODE, 0

// write(fd, buf_adr, buf_len)
.equ SYS_WRITE, 64
.equ STDOUT, 1

// read(fd, buf_adr, buf_len)
.equ SYS_READ, 63
.equ STDIN, 0 

// ASCII code for lowercase 'a'
.equ ASCII_A, 97

// Quick ASCII refresher:
// 65 - 91 = 'A' - 'Z'
// 97 - 123 = 'a' - 'z'

// e.g.
// 'A' + 32 = 'a'
// 'a' - 32 = 'A' 
.equ CASE_DIFF, 32

CHAR:
    .byte 0

.text

.global _start

_start:
    // read(STDIN, CHAR, 1)
    mov x8, SYS_READ
    mov x0, STDIN
    ldr x1, =CHAR
    mov x2, 1
    svc 0

    ldr x19, =CHAR              // load CHAR memory address into x19
    ldrb w20, [x19]             // load byte at [x19] into w20
    cmp w20, ASCII_A            // if byte in w20 is lowercase
    b.ge MAKE_UPPERCASE         // make it uppercase

MAKE_LOWERCASE:                 // else make it lowercase
    add w20, w20, CASE_DIFF     // lowercase byte in w20
    b WRITE                     // then write it to stdout

MAKE_UPPERCASE:
    sub w20, w20, CASE_DIFF     // uppercase byte in w20

WRITE:
    strb w20, [x19]             // store byte in w20 in [x19]

    // write(STDOUT, CHAR, 1)
    mov x8, SYS_WRITE
    mov x0, STDOUT
    ldr x1, =CHAR
    mov x2, 1
    svc 0

    // exit(EXIT_CODE)
    mov x8, SYS_EXIT
    mov x0, EXIT_CODE
    svc 0
```

> If you're following along using the [companion code repository](https://github.com/pretzelhammer/brainfuck_compilers) the command we'll be using to compile and run aarch64 example programs is `just cara {{name}}` where `{{name}}` is the name of the aarch64 source file in the `./examples/aarch64` directory.

```sh
# reads char from stdin, switches its case, prints to stdout

> just cara switch_case
d
D
Exit code: 0

> just cara switch_case
G
g
Exit code: 0
```



## Compiling brainfuck to ARM

We declare a zero-initialized array of 30k bytes with:

```s
.bss

.lcomm ARRAY, 30000
```

As for our pointer and memory storage registers let's use `x19` and `x20` as they are general-purpose callee-saved registers so we're guaranteed they won't be overwritten if we need to make any function or system calls.

Our aarch64 boilerplate:

```s
// header boilerplate //

// GNU Assembler, ARM syntax, aarch64 Linux

.data

.equ SYS_EXIT, 93
.equ SUCCESS, 0

.equ SYS_WRITE, 64
.equ STDOUT, 1

.equ SYS_READ, 63
.equ STDIN, 0 

.bss

.lcomm ARRAY, 30000

.text

.global _start

_start:
    ldr x19, =ARRAY

/////////////////////////////////////////////////
// actual compiled brainfuck program goes here //
/////////////////////////////////////////////////

// footer boilerplate //

    mov x8, SYS_EXIT
    mov x0, SUCCESS
    svc 0
```

Mapping brainfuck commands to aarch64 instructions:

```s
// increment array pointer

// >
add x19, x19, 1

// >>
add x19, x19, 2

// decrement array pointer

// <
sub x19, x19, 1

// <<
sub x19, x19, 2

// increment byte at pointer

// +
ldrb w20, [x19]
add w20, w20, 1
strb w20, [x19]

// ++
ldrb w20, [x19]
add w20, w20, 2
strb w20, [x19]

// decrement byte at pointer

// -
ldrb w20, [x19]
sub w20, w20, 1
strb w20, [x19]

// --
ldrb w20, [x19]
sub w20, w20, 2
strb w20, [x19]

// read byte from stdin & store at pointer

// ,
mov x8, SYS_READ
mov x0, STDIN
mov x1, x19
mov x2, 1
svc 0

// ,,
mov x8, SYS_READ
mov x0, STDIN
mov x1, x19
mov x2, 1
svc 0
mov x8, SYS_READ
mov x0, STDIN
mov x1, x19
mov x2, 1
svc 0

// write byte at pointer to stdout

// .
mov x8, SYS_WRITE
mov x0, STDOUT
mov x1, x19
mov x2, 1
svc 0

// ..
mov x8, SYS_WRITE
mov x0, STDOUT
mov x1, x19
mov x2, 1
svc 0
mov x8, SYS_WRITE
mov x0, STDOUT
mov x1, x19
mov x2, 1
svc 0

// loops

// [
ldrb w20, [x19]
cmp w20, 0
b.eq LOOP_END_1
LOOP_START_0:

// ]
ldrb w20, [x19]
cmp w20, 0
b.ne LOOP_START_0
LOOP_END_1:
```

We're done with our compiler. Let's give it a test drive.

> If you're following along using the [companion code repository](https://github.com/pretzelhammer/brainfuck_compilers) the command we'll be using to compile brainfuck programs to aarch64 and run them is `just carba {{name}}` where `{{name}}` is the name of the brainfuck source file in the `./input` directory.

```sh
# prints "Hello world!"
> just carba hello_world
Hello World!

# prints fibonacci numbers under 100
> just carba fibonacci
1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89

# encrypts lines from stdin using rot13 cipher
> just carba rot13
unencrypted text
harapelcgrq grkg
```

And to perform another unscientific and informal benchmark:

```sh
> just benchmark mandelbrot

# program outputs omitted

# interpreted mandelbrot.b
4.95s user 0.01s system 99% cpu 4.960 total

# x86_64 compiled mandelbrot.b
real    0m1.214s
user    0m1.149s
sys     0m0.041s

# aarch64 compiled mandelbrot.b
real    0m4.206s
user    0m4.103s
sys     0m0.083s
```

The compiled aarch64 program is only a little faster than the interpreter, and much slower than the compiled x86_64 program, however that's to be expected since I'm running the aarch64 program on an x86_64 machine and using QEMU as an aarch64 CPU emulator. If I was running the aarch64 program on an aarch64 machine I imagine it'd be just as fast as the compiled x86_64 program.



## Intro to WebAssembly

Unlike both x86_64 and aarch64, wasm32 is a stack-based ISA. All wasm32 instructions operate by pushing and popping values from an implicit stack. Let's start with a simple example:

```wat
i32.const 4     ;; push value 4 onto stack
i32.const 5     ;; push value 5 onto stack
i32.add         ;; pop 2 values from stack, add them, push result onto stack
```

After `i32.const 4` the stack looks like this:

```
+---------+
|    4    |
+---------+
```

After `i32.const 5` the stack looks like this:

```
+---------+
|    4    |
+---------+
|    5    |
+---------+
```

After `i32.add` the stack looks like this:

```
+---------+
|    9    |
+---------+
```

WebAssembly Textual Format (WAT) supports writing instructions using S-expressions so we could also write the above example like this:

```wat
(i32.add (i32.const 4) (i32.const 5))
```

Both formats, a flat list of instructions and instructions in S-expressions, can be used in the same source file and we'll be using both interchangeably in our examples wherever each helps improve readability.

If we're not interested in a value on the stack we can discard it using the `drop` instruction.

```wat
i32.const 14        ;; pushes 14 onto stack
drop                ;; pops 14 from stack and discards it
```

Unlike in x86_64 and aarch64, all values and instructions in wasm32 are strongly typed. There are only four data types in wasm32: `i32`, `i64`, `f32`, and `f64`. If we try to mix types between values and instructions we will get a compile error:

```wat
i32.const 4
i32.const 5
i64.add             ;; type mismatch compile error, expected i64s found i32s
```

For simplicity and consistency we'll only be using `i32`s for all our examples and our compiler.

Similarly to aarch64, wasm32 does not allow us to operate on memory directly, so we have to load values from memory to the stack, operate on them, and then store the updated values from the stack back to memory.

```wat
;; all instructions below pop a memory address from the stack
;; then push the i32 value at that memory address onto the stack

i32.load8_u         ;; loads unsigned byte value from memory
i32.load8_s         ;; loads signed byte value from memory
i32.load16_u        ;; loads unsigned 2-byte value from memory
i32.load16_s        ;; loads signed 2-byte value from memory
i32.load            ;; loads 4-byte value from memory

;; example
i32.const 1000      ;; push memory address 1000 onto stack
i32.load            ;; loads 4-byte value from memory address 1000
                    ;; loaded result pushed onto stack

;; all instructions below pop a memory address & value from the stack
;; and then store that value at that memory address

i32.store8          ;; stores byte at memory address
i32.store16         ;; stores 2 bytes at memory address
i32.store           ;; stores 4 bytes at memory address

;; example
i32.const 1000      ;; push memory address 1000 onto stack
i32.const 123       ;; push value 123 onto stack
i32.store           ;; stores 123 at memory address 1000
```

Some arithmetic instructions:

```wat
;; both instructions pop 2 values from the stack then push their result onto stack
i32.add
i32.sub
```

Comparing values:

```wat
;; instructions below pop 2 values from the stack
;; then push the result of the comparison onto the stack

i32.eq              ;; equal
i32.ne              ;; not equal
i32.lt_u            ;; less than (unsigned)
i32.lt_s            ;; less than (signed)
i32.gt_u            ;; greater than (unsigned)
i32.gt_s            ;; greater than (signed)
i32.le_u            ;; less than or equal (unsigned)
i32.le_s            ;; less than or equal (signed)
i32.ge_u            ;; greater than or equal (unsigned)
i32.ge_s            ;; greater than or equal (signed)

;; instruction below pops 1 value from the stack
;; then pushes the result onto the stack

i32.eqz             ;; equal to zero
```

In wasm32 there are no booleans types. Any non-zero integer is interpreted as true and zero is interpreted as false.

Unlike in x86_64 and aarch64, wasm32 doesn't support branching to arbitrary labels. All control flow in wasm32 is "structured" and must terminate with an `end` instruction. Furthermore, the behavior of the branching instructions `br` and `br_if` depend on their context within the control flow structure.

```wat
;; simple if / else example

;; "if" pops a value from the stack
if
    ;; instructions to run if popped value was true, i.e. non-zero
else
    ;; instructions to run if popped value was false, i.e. zero
end

;; branching within blocks

block
    br 0            ;; unconditionally branches to end of block
    br_if 0         ;; conditionally branches to end of block 
end

;; branching within loops

loop
    br 0            ;; unconditionally branches to start of loop
    br_if 0         ;; conditionally branches to start of loop
end
```

Since control flow structures can be nested the number after the `br` and `br_if` instructions refers to which structure to perform the branch on, where `br 0` within a `block` would mean _"branch to the end of my current block"_ and `br 1` within a pair of nested `block`s would mean _"branch to the end of my parent block"_ and `br 2` within three nested `block`s would mean _"branch to the end of my grandparent block"_ and so on. This allows us to build a _"branch to the end of the loop"_ instruction by wrapping a `loop` with a `block` and using `br 1` or `br_if 1` like so:

```wat
block
    loop
        br_if 1     ;; conditionally branch to end of parent block (also end of loop)
        br_if 0     ;; conditionally branch to start of loop
    end
end
```

wasm32 instructions cannot exist on their own, they must be inside a function. Here's how we define a function:

```wat
(func $optional_name <optional params> <optional return> <optional locals>
    <list of instructions>
)
```

Functions begin with an empty stack, but can push their params onto the stack using `local.get <index or $label>` where `index` is the param's index in the parameter list and `$label` is the param's optional label (if it was given one).

```wat
;; without labels
(func $my_add (param i32 i32) (result i32)
    local.get 0
    local.get 1
    i32.add
)

;; with labels
(func $my_add (param $first i32) (param $second i32) (result i32)
    local.get $first
    local.get $second
    i32.add
)
```

Functions can return at most one value (although this restriction will be relaxed in the future). If the function returns a value then the stack at the end of the function must have exactly 1 value in it. If the function doesn't return a value then the stack at the end of the function must be empty.

Functions can use `local.set <index or $label>` to pop a value off the stack and store it in a local variable where `index` is the index of that variable within the list of local variables (including params) and `$label` is its optional label (if it was given one). Often we'd like to set a local variable while leaving its value on the stack, which is what the `local.tee` instruction is for, it's basically a shortcut for writing `local.set <index or $label>` immediately followed by `local.get <same index or $label>`.

```wat
;; without labels
(func $add_and_double (param i32 i32) (result i32) (local i32)
    local.get 0     ;; get 1st param
    local.get 1     ;; get 2nd param
    i32.add
    local.tee 2     ;; set & get 1st local var
    local.get 2     ;; get 1st local var again
    i32.add
)

;; with labels
(func $add_and_double (param $first i32) (param $second i32) (result i32) (local $local i32)
    local.get $first
    local.get $second
    i32.add
    local.tee $local
    local.get $local
    i32.add
)
```

We can set up a function call by pushing the function's arguments onto the stack and then calling the function with the `call $label` instruction.

```wat
i32.const 4
i32.const 5
call $my_add            ;; pops 4 & 5 off stack & pushes 9 onto stack
drop                    ;; discard result

i32.const 4
i32.const 5
call $add_and_double    ;; pops 4 & 5 off stack & pushes 18 onto stack
```

Like instructions, functions also cannot exist on their own and must be inside a module. A module is the fundamental unit of code in WebAssembly. Modules can define functions, memory segments, imports, and exports. Things that can be imported and exported include functions and memory segments. Here's a wasm32 module that defines a single function:

```wat
(module
    (func $_start
        ;; instructions
    )
)
```

Memory segments can be defined with `(memory <pages>)` where `<pages>` is how many pages large the memory segment should be. WebAssembly defines a page to be 65536 bytes in size so memory segments can only be multiples of this number. All memory segments are zero-initialized by default. Currently, modules can only define a single memory segment (but this restriction will be relaxed in the future). Here's a WebAssembly module that defines a single memory segment which is 1 page large:

```wat
(module
    ;; defines zero-initialized 65536 byte linear memory segment
    (memory 1)
)
```

Modules can also export functions and memory segments using `export "<exported name>" (<type> <index or $label>)`.

```wat
(module
    ;; zero-initialized 65536 byte linear memory segment
    (memory 1)

    (func $_start
        ;; instructions
    )

    ;; export our only defined memory segment by index as "memory"
    (export "memory" (memory 0))

    ;; export our $_start function as "_start"
    (export "_start" (func $_start))
)
```

WebAssembly modules can't make system calls on their own, that's where WASI comes in. WASI stands for WebAssembly System Interface and defines a set of functions that WASM VMs can implement and wasm32 modules can import to make system calls. To import functions into our module we have to use `(import "<namespace>" "<func name>" (func $<label> <params> <result>)`:

```wat
;; proc_exit(code)
(import "wasi_snapshot_preview1" "proc_exit"
    (func $proc_exit (param i32)))

;; fd_read(fd, iovec[]*, iovec_len, bytes_read*) -> error_number
(import "wasi_snapshot_preview1" "fd_read"
    (func $fd_read (param i32 i32 i32 i32) (result i32)))

;; fd_write(fd, iovec[]*, iovec_len, bytes_written*) -> error_number
(import "wasi_snapshot_preview1" "fd_write"
    (func $fd_write (param i32 i32 i32 i32) (result i32)))

;; An "iovec" is an 8-byte struct with two 4-byte members:
;; 1) buffer address (0-byte offset from start of iovec)
;; 2) buffer length (4-byte offset from start of iovec)

;; for example, if we had a 27-byte buffer at memory address
;; 1000 this is how we would construct an iovec struct stored
;; at memory address 3000 to point to that buffer:

;; store 1st iovec struct member
i32.const 3000      ;; 1st member at 3000 (3000 + 0 offset)
i32.const 1000      ;; buffer memory address
i32.store

;; store 2nd iovec struct member
i32.const 3004      ;; 2nd member at 3004 (3000 + 4 offset)
i32.const 27        ;; buffer length in bytes
i32.store

;; examples

;; proc_exit(code)
i32.const 0         ;; 0 exit code means success
call $proc_exit

;; fd_read(fd, iovec[]*, iovec_len, bytes_read*) -> error_number
i32.const 0         ;; file descriptor for stdin
i32.const 3000      ;; memory address to array of iovec structs
i32.const 1         ;; number of iovec structs in array
i32.const 5678      ;; memory address where to write bytes_read
call $fd_read       ;; pop 4 values from stack, pushes error_number to stack
drop                ;; discard error_number

;; fd_write(fd, iovec[]*, iovec_len, bytes_written*) -> error_number
i32.const 1         ;; file descriptor for stdout
i32.const 3000      ;; memory address to array of iovec structs
i32.const 1         ;; number of iovec structs in array
i32.const 5678      ;; memory address where to write bytes_written
call $fd_write      ;; pop 4 values from stack, pushes error_number to stack
drop                ;; discard error_number
```

Okay, we've _finally_ established enough context that we can now port `switch_case.s` to wasm32-wasi. As a refresher, this program reads 1 character from stdin, switches its case, writes the character to stdout, and then exits:

```wat
;; ./examples/wasm32-wasi/switch_case.wat

(module
    (import "wasi_snapshot_preview1" "proc_exit"
        (func $proc_exit (param i32)))

    (import "wasi_snapshot_preview1" "fd_write"
        (func $fd_write (param i32 i32 i32 i32) (result i32)))

    (import "wasi_snapshot_preview1" "fd_read"
        (func $fd_read (param i32 i32 i32 i32) (result i32)))

    (memory 1)

    (func $_start (local $char i32)
        ;; treat mem[0] as a 1 byte buffer

        ;; set up iovec at mem[4-12] to point to buffer
        i32.const 4     ;; memory address 4 (4 + 0 offset)
        i32.const 0     ;; memory address of buffer
        i32.store       ;; mem[4-8] = 0, 1st iovec member
        i32.const 8     ;; memory address 8 (4 + 4 offset)
        i32.const 1     ;; buffer length (1 byte)
        i32.store       ;; mem[8-12] = 1, 2nd iovec member

        ;; recap:
        ;; mem[0] = 1 byte buffer
        ;; mem[4-12] = iovec { buf_adr, buf_len }
        
        ;; $fd_read(STDIN, mem[4], 1, mem[12])
        i32.const 0         ;; 0 = STDIN
        i32.const 4         ;; mem[4] = iovec[]
        i32.const 1         ;; 1 = # of iovecs
        i32.const 12        ;; mem[12] = bytes_read address
        call $fd_read       ;; pop 4 values from stack, push error_num
        drop                ;; drop error_num (assume success)

        i32.const 0
        i32.load8_u         ;; load byte at mem[0]
        local.tee $char     ;; $char = mem[0]
        i32.const 97
        i32.ge_u            ;; is $char >= 97 ?

        if                  ;; if true make $char uppercase
            i32.const 0
            local.get $char
            i32.const 32
            i32.sub
            i32.store8      ;; mem[0] = $char - 32
        else                ;; if false make $char lowercase
            i32.const 0
            local.get $char
            i32.const 32
            i32.add
            i32.store8      ;; mem[0] = $char + 32
        end

        ;; $fd_write(STDOUT, mem[4], 1, mem[12])
        i32.const 1         ;; 1 = STDOUT
        i32.const 4         ;; mem[4] = iovec[]
        i32.const 1         ;; 1 = # of iovecs
        i32.const 12        ;; mem[12] = bytes_written address
        call $fd_write      ;; pop 4 values from stack, push error_num
        drop                ;; drop error_num (assume success)

        ;; $proc_exit(0)
        i32.const 0
        call $proc_exit
    )

    ;; export "memory" & "_start" to runtime
    (export "memory" (memory 0))
    (export "_start" (func $_start))
)
```

> If you're following along using the [companion code repository](https://github.com/pretzelhammer/brainfuck_compilers) the command we'll be using to compile and run WebAssembly examples is `just carw {{name}}` where `{{name}}` is the name of the WebAssembly source file in the `./examples/wasm32-wasi` directory.

```sh
# reads char from stdin, switches its case, prints to stdout

> just carw switch_case
f
F
Exit code: 0

> just carw switch_case
Q
q
Exit code: 0
```



## Compiling brainfuck to WebAssembly

Defining a zero-initialized memory segment in wasm32 is easy and we covered that in the previous section. What's not as easy is deciding where to store our iovec, which is a required in-memory struct that we need to use for `fd_read` and `fd_write` system calls. Since we're using the first 30k bytes of our memory segment for our brainfuck program let's store the iovec at memory address 30004. For our array pointer let's store it a function local variable and copy it over to the iovec struct before reads and writes. We've now made enough decisions to generate a wasm32-wasi boilerplate for our compiled brainfuck programs:

```wat
;; header boilerplate ;;

(module
    (import "wasi_snapshot_preview1" "fd_write"
        (func $fd_write (param i32 i32 i32 i32) (result i32)))

    (import "wasi_snapshot_preview1" "proc_exit"
        (func $proc_exit (param i32)))

    (import "wasi_snapshot_preview1" "fd_read"
        (func $fd_read (param i32 i32 i32 i32) (result i32)))

    (memory 1)

    (func $_start (local $ptr i32)
        ;; set up array pointer
        i32.const 0
        local.set $ptr

        ;; set up 1st iovec member
        i32.const 30004     ;; 30004 address (30004 + 0 offset)
        local.get $ptr      ;; initial index = 0
        i32.store           ;; mem[30004-30008] = 0

        ;; set up 2nd iovec member
        i32.const 30008     ;; 30008 address (30004 + 4 offset)
        i32.const 1         ;; buffer length is 1 byte
        i32.store           ;; mem[30008-30012] = 1

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;; actual compiled brainfuck program goes here ;;
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

;; footer boilerplate ;;

        i32.const 0
        call $proc_exit
    )

    (export "memory" (memory 0))
    (export "_start" (func $_start))
)
```

Alright let's map brainfuck commands to wasm32 instructions:

```wat
;; increment array pointer

;; >
local.get $ptr
i32.const 1
i32.add
local.set $ptr

;; >>
local.get $ptr
i32.const 2
i32.add
local.set $ptr

;; decrement array pointer

;; <
local.get $ptr
i32.const 1
i32.sub
local.set $ptr

;; <<
local.get $ptr
i32.const 2
i32.sub
local.set $ptr

;; increment byte at pointer

;; +
local.get $ptr
local.get $ptr
i32.load8_u
i32.const 1
i32.add
i32.store8

;; ++
local.get $ptr
local.get $ptr
i32.load8_u
i32.const 2
i32.add
i32.store8

;; decrement byte at pointer

;; -
local.get $ptr
local.get $ptr
i32.load8_u
i32.const 1
i32.sub
i32.store8

;; --
local.get $ptr
local.get $ptr
i32.load8_u
i32.const 2
i32.sub
i32.store8

;; read byte from stdin & store at pointer

;; ,
i32.const 30004
local.get $ptr
i32.store
i32.const 0
i32.const 30004
i32.const 1
i32.const 30012
call $fd_read
drop

;; ,,
i32.const 30004
local.get $ptr
i32.store
i32.const 0
i32.const 30004
i32.const 1
i32.const 30012
call $fd_read
drop
i32.const 0
i32.const 30004
i32.const 1
i32.const 30012
call $fd_read
drop

;; write byte at pointer to stdout

;; .
i32.const 30004
local.get $ptr
i32.store
i32.const 0
i32.const 30004
i32.const 1
i32.const 30012
call $fd_write
drop

;; ..
i32.const 30004
local.get $ptr
i32.store
i32.const 0
i32.const 30004
i32.const 1
i32.const 30012
call $fd_write
drop
i32.const 0
i32.const 30004
i32.const 1
i32.const 30012
call $fd_write
drop

;; loops

;; [
block
local.get $ptr
i32.load8_u
i32.eqz
br_if 0
loop

;; ]
local.get $ptr
i32.load8_u
br_if 0
end
end
```

That was surprisingly way more work than the x86_64 and aarch64 compilers! Thankfully we're done.

> If you're following along using the [companion code repository](https://github.com/pretzelhammer/brainfuck_compilers) the command we'll be using to compile brainfuck programs to WebAssembly and run them is `just carbw {{name}}` where `{{name}}` is the brainfuck source file in the `./input` directory.

```sh
# prints "Hello world!"
> just carbw hello_world
Hello World!

# prints fibonacci numbers under 100
> just carbw fibonacci
1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89

# encrypts lines from stdin using rot13 cipher
> just carbw rot13
unencrypted text
harapelcgrq grkg
```

Another unscientific and informal benchmark:

```sh
> just benchmark mandelbrot

# program outputs omitted

# interpreted mandelbrot.b
4.95s user 0.01s system 99% cpu 4.960 total

# x86_64 compiled mandelbrot.b
real    0m1.214s
user    0m1.149s
sys     0m0.041s

# aarch64 compiled mandelbrot.b
real    0m4.206s
user    0m4.103s
sys     0m0.083s

# wasm32-wasi compiled mandelbrot.b
real    0m1.480s
user    0m1.429s
sys     0m0.046s
```

The compiled wasm32-wasi version is 3.3x faster than the interpreted version and only 0.8x as fast as the x86_64 version. Pretty good! The promise that WASM programs execute at near-native speeds holds up!



## Intro to LLVM

Unlike x86_64, aarch64, and WebAssembly, LLVM IR isn't really a register-based or stack-based ISA. LLVM IR has variables like a high-level language. However, unlike high-level languages, LLVM IR variables must follow Static Single-Assignment (SSA) form which requires that every variable is assigned to exactly once. This is annoying because it means we can't reuse variable names.

```ll
%i = add i8 1, 2
%i = add i8 3, 4       ; illegal, %i assigned to twice
```

Unlike x86_64 and aarch64, but similarly to WebAssembly, all variables and instructions in LLVM IR are strongly typed. LLVM IR supports many types but the ones most relevant to us are integers, arrays, functions, and pointers:

```ll
; integer types

i<bit-width>            ; integer that is <bit-width> bits wide

; examples

i1                      ; 1 bit integer
i64                     ; 64 bit integer

; array types

[<quantity> x <type>]   ; array of <quantity> <type>

; examples

[4 x i32]               ; array of 4 32-bit integers
[12 x [2 x i8]]         ; array of 12 arrays of 2 bytes

; function types

<return_type> (<param_list>)

; examples

i32 (i32)               ; function taking and returning one i32
void (i8, i8)           ; function taking 2 bytes and returning void

; pointer types

<type>*                 ; pointer to <type>

; examples

i8*                     ; byte pointer
[10 x i32]*             ; pointer to array of 10 32-bit integers
i32 (i32)*              ; pointer to function taking and returning one i32
```

There's an LLVM IR instruction called `atomicrmw` that allows us to atomically modify memory values directly using a subset of the available arithmetic instructions but learning and using that instruction before we've learned the basics of `load` and `store` kinda feels like cheating so let's go over those instead. We'll explicitly load and store values for our examples and in our compiler.

```ll
; load

%val = load <type>, <type>* %ptr

; store

store <type> <value>, <type>* %ptr

; example

; assume %byte is some i8*
; below we add 1 to the value at %byte

%b.0 = load i8, i8* %byte
%b.1 = add i8 %b.0, 1
store i8 %b.1, i8* %byte
```

We can set global variables (outside of any function) using the following syntax:

```ll
@<name> = global <type> <value>

; where @<name> will have type <type>*

; example

@counter = global i32 0

; @counter has type i32*
```

Unfortunately setting local variables isn't as concise. There's this "shortcut" but it feels kinda hacky:

```ll
%var = add i32 0, 10        ; why am I adding 0 + 10 to set %var to 10 ?
```

The correct way to set a local variable is to do this:

```ll
%var = alloca i32           ; allocate space for an i32 on the stack
store i32 10, i32* %var     ; store 10 in %var
```

Which seems really inefficient but after an optimization pass LLVM transforms it into a single instruction where the literal value is set directly in a register (instead of on the stack).

Some arithmetic instructions:

```ll
%result = add <type> <op1>, <op2>
%result = sub <type> <op1>, <op2>
```

Comparing values:

```ll
%result = icmp <cond> <type> <op1>, <op2>

; where %result is an i1 and 1 = true, 0 = false

; where <cond> can be
;   - eq                ; equal
;   - ne                ; not equal
;   - ugt               ; unsigned greater than
;   - uge               ; unsigned greater than or equal
;   - ult               ; unsigned less than
;   - ule               ; unsigned less than or equal
;   - sgt               ; signed greater than
;   - sge               ; signed greater than or equal
;   - slt               ; signed less than
;   - sle               ; signed less than or equal

; examples

%true = icmp eq i8 0, 0
%false = icmp ugt i32 5, 9
%bool = icmp ne i64 %somevar, %othervar
```

Control flow:

```ll
; unconditional branch to <label>
br label %<label>

; conditional branch to <true> if <bool> is true, to <false> otherwise
br i1 <bool>, label %<true>, label %<false>

; return value <value> of type <type> from function
ret <type> <value>
```

Functions:

```ll
; declare external function
declare <return_type> @<name>(<args>)

; define function
define <return_type> @<name>(<args>) {
    <instructions>
}

; call function with call instruction
%result = call <return_type> @<name>(<args>)

; where <args> is a comma-separated list of <type> %<name>
```

LLVM refers to labeled blocks of instructions (note: the block of instructions inside a function body gets an implicit label) as _basic blocks_ and all _basic blocks_ must be terminated with a _terminator instruction_ that produces control flow to some other _basic block_ so _terminator instructions_ naturally include all the control flow instructions like `br` and `ret`. This is important to explain because there's no "fall through" between blocks in LLVM IR like there is in x86_64 and aarch64. Example:

```ll
define i32 @max(i32 %a, i32 %b) {
    %max = alloca i32
    %bool = icmp ugt i32 %a, %b
    br i1 %bool, label %A_IS_BIGGER, label %B_IS_BIGGER

A_IS_BIGGER:
    store i32 %a, i32* %max
    br label %RETURN

B_IS_BIGGER:
    store i32 %b, i32* %max
    br label %RETURN ; this seemingly pointless instruction is *required*

RETURN:
    %ret = load i32, i32* %max
    ret i32 %ret
}
```

There's a handy `select` instruction that allows us to condense the above example down to just a few lines:

```ll
; select i1 %<bool>, <type> <val>, <type> <val>

define i32 @max(i32 %a, i32 %b) {
    %bool = icmp ugt i32 %a, %b
    %ret = select i1 %bool, i32 %a, i32 %b
    ret i32 %ret
}
```

Okay, so we could make direct system calls by writing inline assembly in LLVM IR but that would beat the point of using LLVM IR in the first place so instead we're going to use functions from the C standard library, often just called libc, which abstract away having to deal with all the individual quirks of system calls across different platforms and targets. Here's `switch_case.s` ported to LLVM IR:

```ll
; ./examples/llvm_ir/switch_case.ll

; libc functions
declare i8 @putchar(i8)
declare i8 @getchar()

; main function called by libc
; the return value is set as program's exit code
define i8 @main() {
    %switched = alloca i8
    %char = call i8 @getchar()
    %bool = icmp uge i8 %char, 97 ; %char >= 97 ?
    br i1 %bool, label %MAKE_UPPERCASE, label %MAKE_LOWERCASE

MAKE_UPPERCASE:
    %upper = sub i8 %char, 32
    store i8 %upper, i8* %switched
    br label %WRITE

MAKE_LOWERCASE:
    %lower = add i8 %char, 32
    store i8 %lower, i8* %switched
    br label %WRITE
    
WRITE:
    %result = load i8, i8* %switched
    call i8 @putchar(i8 %result)
    ret i8 0
}
```

> If you're following along using the [companion code repository](https://github.com/pretzelhammer/brainfuck_compilers) the command we'll be using to compile and run LLVM IR examples is `just carl {{name}}` where `{{name}}` is the LLVM IR source file in the `./examples/llvm_ir` directory.

```sh
# reads char from stdin, switches its case, prints to stdout

> just carl switch_case
j
J
Exit code: 0

> just carl switch_case
T
t
Exit code: 0
```



## Compiling brainfuck to LLVM

We can create a global zero-initialized array of 30k bytes using the handy `zeroinitializer` keyword.

```ll
@array = global [ 300000 x i8 ] zeroinitializer
```

We could maintain a pointer into this array but LLVM IR doesn't make that easy on us because there are no pointer arithmetic instructions. To do pointer arithmetic in LLVM IR we have to convert the pointer to an integer with `ptrtoint`, do the arithmetic, and then convert it back with `inttoptr`. This is verbose and not fun. Instead let's maintain a global index variable and use the `getelementptr` instruction to get pointers into our global array.

```ll
@index = global i64 0

; generic template for getting a pointer into an array using an index:
; %ptr = getelementptr <type>, <type>* <array>, i64 0, i64 <index>

; example
%index = load i64, i64* @index
%ptr = getelementptr [ 30000 x i8 ], [ 30000 x i8 ]* @array, i64 0, i64 %index
```

It's not pretty, but the alternatives are even uglier, so it's the best we have to work with. But now with that out of the way we can generate an LLVM IR boilerplate for our compiled brainfuck programs:

```ll
; header boilerplate ;

@array = global [ 30000 x i8 ] zeroinitializer
@index = global i64 0

declare i8 @putchar(i8)
declare i8 @getchar()

define i8 @main() {
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; actual compiled brainfuck program goes here ;
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

; footer boilerplate ;

    ret i8 0
}
```

Mapping brainfuck commands to LLVM IR instructions:

```ll
; increment array pointer

; >
%idx.0 = load i64, i64* @index
%idx.1 = add i64 %idx.0, 1
store i64 %idx.1, i64* @index

; >>
%idx.2 = load i64, i64* @index
%idx.3 = add i64 %idx.2, 2
store i64 %idx.3, i64* @index

; decrement array pointer

; <
%idx.4 = load i64, i64* @index
%idx.5 = sub i64 %idx.4, 1
store i64 %idx.5, i64* @index

; <<
%idx.6 = load i64, i64* @index
%idx.7 = sub i64 %idx.6, 2
store i64 %idx.7, i64* @index

; increment byte at pointer

; +
%idx.8 = load i64, i64* @index
%ptr.0 = getelementptr [ 30000 x i8 ], [ 30000 x i8 ]* @array, i64 0, i64 %idx.8
%byte.0 = load i8, i8* %ptr.0
%byte.1 = add i8 %byte.0, 1
store i8 %byte.1, i8* %ptr.0

; ++
%idx.9 = load i64, i64* @index
%ptr.1 = getelementptr [ 30000 x i8 ], [ 30000 x i8 ]* @array, i64 0, i64 %idx.9
%byte.2 = load i8, i8* %ptr.1
%byte.3 = add i8 %byte.2, 2
store i8 %byte.3, i8* %ptr.1

; decrement byte at pointer

; -
%idx.10 = load i64, i64* @index
%ptr.2 = getelementptr [ 30000 x i8 ], [ 30000 x i8 ]* @array, i64 0, i64 %idx.10
%byte.4 = load i8, i8* %ptr.2
%byte.5 = sub i8 %byte.4, 1
store i8 %byte.5, i8* %ptr.2

; --
%idx.11 = load i64, i64* @index
%ptr.3 = getelementptr [ 30000 x i8 ], [ 30000 x i8 ]* @array, i64 0, i64 %idx.11
%byte.6 = load i8, i8* %ptr.3
%byte.7 = sub i8 %byte.6, 2
store i8 %byte.7, i8* %ptr.3

; read byte from stdin & store at pointer

; ,
%idx.12 = load i64, i64* @index
%ptr.4 = getelementptr [ 30000 x i8 ], [ 30000 x i8 ]* @array, i64 0, i64 %idx.12
%char.0 = call i8 @getchar()
%bool.0 = icmp eq i8 -1, %char.0
%char.1 = select i1 %bool.0, i8 0, i8 %char.0
store i8 %char.1, i8* %ptr.4

; ,,
%idx.13 = load i64, i64* @index
%ptr.5 = getelementptr [ 30000 x i8 ], [ 30000 x i8 ]* @array, i64 0, i64 %idx.13
call i8 @getchar()
%char.2 = call i8 @getchar()
%bool.1 = icmp eq i8 -1, %char.2
%char.3 = select i1 %bool.1, i8 0, i8 %char.2
store i8 %char.3, i8* %ptr.5

; write byte at pointer to stdout

; .
%idx.14 = load i64, i64* @index
%ptr.6 = getelementptr [ 30000 x i8 ], [ 30000 x i8 ]* @array, i64 0, i64 %idx.14
%char.4 = load i8, i8* %ptr.6
call i8 @putchar(i8 %char.4)

; ..
%idx.15 = load i64, i64* @index
%ptr.7 = getelementptr [ 30000 x i8 ], [ 30000 x i8 ]* @array, i64 0, i64 %idx.15
%char.5 = load i8, i8* %ptr.7
call i8 @putchar(i8 %char.5)
call i8 @putchar(i8 %char.5)

; loops

; [
%idx.16 = load i64, i64* @index
%ptr.8 = getelementptr [ 30000 x i8 ], [ 30000 x i8 ]* @array, i64 0, i64 %idx.16
%byte.8 = load i8, i8* %ptr.8
%bool.2 = icmp eq i8 0, %byte.8
br i1 %bool.2, label %LOOP_END_1, label %LOOP_START_0
LOOP_START_0:

; ]
%idx.18 = load i64, i64* @index
%ptr.10 = getelementptr [ 30000 x i8 ], [ 30000 x i8 ]* @array, i64 0, i64 %idx.18
%byte.11 = load i8, i8* %ptr.10
%bool.3 = icmp ne i8 0, %byte.11
br i1 %bool.3, label %LOOP_START_0, label %LOOP_END_0
LOOP_END_0:
```

Eagle-eyed readers may have noticed something slightly unusual in the implementation for `,`. The reason why we need to perform an `icmp` and `select` in our implementation of `,` is because the libc `getchar()` function returns `-1` on EOF but the semantics of our brainfuck compiler is that all EOFs should be read and stored as the value `0` so if we get a `-1` from `getchar()` we have to map it to a `0` before storing it in our array.

> If you're following along using the [companion code repository](https://github.com/pretzelhammer/brainfuck_compilers) the command we'll be using to compile brainfuck programs to LLVM IR and run them is `just carbl {{name}}` where `{{name}}` is the brainfuck source file in the `./input` directory.

```sh
# prints "Hello world!"
> just carbl hello_world
Hello World!

# prints fibonacci numbers under 100
> just carbl fibonacci
1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89

# encrypts lines from stdin using rot13 cipher
> just carbl rot13
unencrypted text
harapelcgrq grkg
```

Another totally unscientific and informal benchmark:

```sh
> just benchmark mandelbrot

# program outputs omitted

# interpreted mandelbrot.b
4.95s user 0.01s system 99% cpu 4.960 total

# x86_64 compiled mandelbrot.b
real    0m1.214s
user    0m1.149s
sys     0m0.041s

# aarch64 compiled mandelbrot.b
real    0m4.206s
user    0m4.103s
sys     0m0.083s

# wasm32-wasi compiled mandelbrot.b
real    0m1.480s
user    0m1.429s
sys     0m0.046s

# llvm-ir compiled mandelbrot.b
real    0m0.896s
user    0m0.887s
sys     0m0.001s
```

The LLVM IR compiled version is not only 5.5x faster than the interpreted version but it's also 1.3x faster than the x86_64 compiled version. What!? How!? That's nuts. Given how incredibly simplistic brainfuck programs are I'm surprised the LLVM IR optimizer still found so much room for improvement.



## Optimization opportunities

There's 3 ways we can massively improve the performance of all our compilers.


### Interpret during compilation and capture output

Brainfuck programs that don't have any `,` (read byte from stdin) commands have deterministic outputs so if a brainfuck program completes in a finite amount of time (as some brainfuck programs are written to execute indefinitely until they get a SIGKILL) then we can interpret it during compilation, capture its output, and then write a compiled program that just prints that output to stdout. This is "the ultimate" optimization as all compiled programs, regardless of how complex their source code was, will finish execution nearly instantly and no other optimizations are required. The following 2 optimizations are listed only to take into account brainfuck programs where this optimization cannot be applied (which are programs with `,` in their source code or programs that run indefinitely).


### Un-loop-ify simple loops

While looking through the source code of brainfuck programs I noticed this command pattern a lot: `[-]`. It's clear what it's doing, it's zeroing the byte at the current pointer. However the code we're generating for this simple case is wildly inefficient. Our x86_64 compiler will generate the following instructions:

```s
# [
cmpb [r12], 0
je LOOP_END_2
LOOP_START_0:

# - (subtract in loop until 0)
subb [r12], 1

# ]
cmpb [r12], 0
jne LOOP_START_0
LOOP_END_2:
```

When all it needs to generate is just this:

```s
# [-]
movb [r12], 0
```

A slightly more complex command pattern is this `[->+<]` which zeros the byte at the pointer _and_ adds its value to its neighbor in the array. Our x86_64 compiler would generate the following instructions for this command pattern:

```s
# [
cmpb [r12], 0
je LOOP_END_5
LOOP_START_0:

# - (decrement byte until 0 in loop)
subb [r12], 1

# > (move to neighbor)
add r12, 1

# + (increment neighbor)
addb [r12], 1

# < (move back to original byte)
sub r12, 1

# ]
cmpb [r12], 0
jne LOOP_START_0
LOOP_END_5:
```

However all it needs to generate is:

```s
# [->+<]
movb r13b, [r12]        # save current byte value
addb [r12 + 1], r13b    # add current byte value to neighbor
movb [r12], 0           # zero current byte
```

Most simple flat loops in brainfuck can be reduced to just a few instructions. A sufficiently smart compiler should be able to identify these scenarios, and it can maybe explain how the LLVM IR optimizer produces compiled brainfuck programs that run 1.3x faster than our naive x86_64 compiler.



### Buffer reads and writes

System calls are expensive, even without including the cost of switching from user-space to kernel-space they just take a lot of instructions to set up and invoke. Our compilers could all be made more efficient if they wrote bytes to an internal buffer and only invoked the system call to write the bytes to stdout when absolutely necessary: like when the buffer fills up, or an `,` command is reached, or the end of the program is reached.



## Concluding thoughts

I learned a lot and still far less than I thought I would. Remember that laundry list of fancy terms I mentioned back in the intro of this article? Yeah well, I still don't know what half of that stuff is.

While doing the research and programming for this project I finally learned what _auto-vectorization_, _inlining_, _endianness_, _system calls_, _LLVM_, _SIMD_, and _ABI_ are. I kinda get what _linking_ is on a very basic level but I still get lost whenever I read anything about _linking_ because it seems like the linker does a whole bunch of really crazy complicated code manipulations other than just playing connect-the-dots with some global symbols, so I don't feel like I "fully get" what a linker actually does. I get what _custom allocators_ are in concept but I don't get why, for example, Allocator X is more performant than Allocator Y for certain workloads. This project never forced me to figure out how heap allocations work in assembly so it makes sense that allocators are still a mystery to me. I know that _TLS_ stands for Thread Local Storage and people love talking about it but I don't know why. I know _padding_ is a thing that exists purely to serve _alignment_ but I have no clue why _alignment_ is so important. Apparently if the data in your program is _aligned_ everything is faster and if it's _unaligned_ it's either slow or completely unusable. But why? What is it with all this magical _alignment_ stuff?

I was very surprised by how much easier it was to write the x86_64 and aarch64 compilers compared to the WebAssembly and LLVM IR compilers. I think this mostly has to do with the fact that brainfuck is a super simple language that maps very cleanly to low-level assembly instructions and if I was writing compilers for a higher-level language it'd be easier to map higher-level constructs to WebAssembly and LLVM IR than x86_64 and aarch64 but I've never tried to do this so I can't say 100% for sure.

The documentation online for x86_64 and aarch64 is pretty terrible. It seems like if you seriously want to get into x86_64 or aarch64 programming you're probably better off buying some assembly books on Amazon and reading the 5000+ page programming manual for x86_64 on Intel's website or the 8000+ page programming manual for aarch64 on ARM's website. It's not very beginner-friendly.

The documentation online for WebAssembly is good _if you're writing a WASM VM_. If you're approaching WebAssembly as an application or compiler developer then it's terrible. There's no beginner-friendly tutorials on how to do anything and you have to figure everything out for yourself.

The documentation online for LLVM IR is by far the best. The [LLVM IR Language Reference](https://llvm.org/docs/LangRef.html) not only thoroughly explains every instruction but also shows example usages for all of the instructions! LLVM also maintains an official tutorial called [My First Language Frontend with LLVM](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/index.html) that shows how to implement an LLVM IR compiler for a simple programming language in C++. I don't know C++ so I didn't read the tutorial but the fact they have an official maintained tutorial is nice. Also there's a free online ebook called [Mapping High Level Constructs to LLVM IR](https://mapping-high-level-constructs-to-llvm-ir.readthedocs.io/en/latest/README.html) which is also pretty great.

If I had to write a compiler in the future I think I'll definitely stick with LLVM IR. It has the best documentation, it has a kick-ass optimizer, and it can compile down to x86_64, aarch64, or WebAssembly (plus a whole bunch of other targets)!


## Discuss

Discuss this article on
- [compilers subreddit](https://www.reddit.com/r/Compilers/comments/jpqnws/learn_assembly_with_entirely_too_many_brainfuck/)
- [asm subreddit](https://www.reddit.com/r/asm/comments/jqbq47/learn_assembly_with_entirely_too_many_brainfuck/)
- [coding subreddit](https://www.reddit.com/r/coding/comments/jqxl19/learn_assembly_with_entirely_too_many_brainfuck/)
- [ProgrammingLanguages subreddit](https://www.reddit.com/r/ProgrammingLanguages/comments/jsf9cr/learn_assembly_by_writing_entirely_too_many/)
- [programming subreddit](https://www.reddit.com/r/programming/comments/jrlljd/learn_assembly_with_entirely_too_many_brainfuck/)
- [Official Rust users forum](https://users.rust-lang.org/t/learn-assembly-by-writing-entirely-too-many-brainfuck-compilers-in-rust/51333?u=pretzelhammer)
- [rust subreddit](https://www.reddit.com/r/rust/comments/jsvdsy/learn_assembly_by_writing_entirely_too_many/)
- [Hackernews](https://news.ycombinator.com/item?id=25069243)
- [Github](https://github.com/pretzelhammer/rust-blog/discussions)

## Further Reading

Rust
- [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md)
- [Tour of Rust's Standard Library Traits](./tour-of-rusts-standard-library-traits.md)
- [Sizedness in Rust](./sizedness-in-rust.md)
- [RESTful API in Sync & Async Rust](./restful-api-in-sync-and-async-rust.md)
- [Learning Rust in 2020](./learning-rust-in-2020.md)

ISAs
- [What does RISC and CISC mean in 2020?](https://medium.com/swlh/what-does-risc-and-cisc-mean-in-2020-7b4d42c9a9de)

GNU Assembler
- [GNU Assembler Manual](https://sourceware.org/binutils/docs/as/)

x86
- [StackOverflow x86 Tag Wiki](https://stackoverflow.com/tags/x86/info)
- [CDOT x86_64 Quick Start](https://wiki.cdot.senecacollege.ca/wiki/X86_64_Register_and_Instruction_Quick_Start)
- [Intel's Software Developer Manuals](https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html)
- [AMD's Architecture Programmer's Manuals](https://developer.amd.com/resources/developer-guides-manuals/)

ARM
- [CDOT aarch64 Quick Start](https://wiki.cdot.senecacollege.ca/wiki/AArch64_Register_and_Instruction_Quick_Start)
- [Azeria Labs ARM Assembly Basics Tutorial](https://azeria-labs.com/writing-arm-assembly-part-1/)
- [modexp's Guide to aarch64 Assembly](https://modexp.wordpress.com/2018/10/30/arm64-assembly/)
- [ARM Programmer's Guide for ARMv8-A](https://developer.arm.com/documentation/den0024/a)
- [ARM Architecture Reference Manual for ARMv8-A](https://developer.arm.com/documentation/ddi0487/latest)

WebAssembly
- [WebAssembly Specification](https://webassembly.github.io/spec/core/index.html)
- [MDN's Understanding WebAssembly Text Format](https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format)
- [sunfishcode's WASM Reference Manual](https://github.com/sunfishcode/wasm-reference-manual/blob/master/WebAssembly.md)

LLVM IR
- [LLVM Language Reference Manual](https://llvm.org/docs/LangRef.html)
- [Mapping High Level Constructs to LLVM IR](https://mapping-high-level-constructs-to-llvm-ir.readthedocs.io/en/latest/README.html)
- [My First Language Frontend with LLVM Tutorial](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/index.html)
