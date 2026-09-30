# JS-is-bloated

## jsib — JavaScript Is Bloated? Not anymore.

**jsib** compiles JavaScript and TypeScript into **sub-100KB standalone, statically linked Linux ELF binaries** with:
* **Zero Node.js runtime**
* **Zero V8 JIT engine**
* **Zero dynamic `.so` dependencies**
* **Hardened W^X memory layout & non-executable stack**
* **Sub-millisecond cold startup (~302 µs / 34,600 CPU cycles)**
* **Persistent caching (< 0.5s recompilations)**
* **Native coroutine engine (`async/await` without VM overhead)**
* **Full standard battery: Regular Expressions, zlib, mbedTLS, and File System**

Built by [TheMaster1127](https://github.com/TheMaster1127).

---

## Table of Contents
- [The Problem](#the-problem)
- [What is scriptc? (And Why Does jsib Exist?)](#what-is-scriptc-and-why-does-jsib-exist)
- [Runtime Coverage (What Actually Works?)](#runtime-coverage-what-actually-works)
- [The Philosophy & The Scaling Curve](#the-philosophy--the-scaling-curve)
- [Live Silicon Benchmarks](#live-silicon-benchmarks)
  - [1. Micro-Benchmark: Cold Startup & Hardware Counters](#1-micro-benchmark-cold-startup--hardware-counters)
  - [2. Mega-Scale Synthetic Stress Test (1,000,000 Lines)](#2-mega-scale-synthetic-stress-test-1000000-lines)
- [How It Works: Step-by-Step Architecture](#how-it-works-step-by-step-architecture)
  - [1. AOT IR Lowering (scriptc Interception)](#1-aot-ir-lowering-scriptc-interception)
  - [2. Naked Musl Splicing & Decapitation](#2-naked-musl-splicing--decapitation)
  - [3. The Naked AMD64 Assembly Stack Pivot](#3-the-naked-amd64-assembly-stack-pivot)
  - [4. Vendor Engine Integration (Regex, zlib & mbedTLS)](#4-vendor-engine-integration-regex-zlib--mbedtls)
  - [5. Section-Level Isolation & Dead-Code Elimination](#5-section-level-isolation--dead-code-elimination)
  - [6. The Hardened W^X Linker Script](#6-the-hardened-wx-linker-script)
  - [7. Section Decapitation (`sstrip`)](#7-section-decapitation-sstrip)
  - [8. Persistent Precompiled Runtime Cache](#8-persistent-precompiled-runtime-cache)
- [Scope & Limitations](#scope--limitations)
- [Requirements & Clean NPM Setup](#requirements--clean-npm-setup)
- [Installation](#installation)
- [Usage & CLI Reference](#usage--cli-reference)
- [Author & Ecosystem](#author--ecosystem)
- [License](#license)

---

## The Problem

Modern JavaScript deployment toolchains suffer from staggering bloat. When you want to ship a single script or CLI tool as a standalone executable, current industry tools bundle an entire browser-grade operating environment:

| Runtime / Tool | Packaging Mechanism | Binary Size | Startup Latency |
| :--- | :--- | :--- | :--- |
| **Node.js (SEA / pkg)** | Bundles V8 JIT + Node Core + ICU tables | **~85 MB – 110 MB** | ~90 – 120 ms |
| **Bun compile** | Bundles JavaScriptCore VM + Zig runtime | **~60 MB – 90 MB** | ~25 – 40 ms |
| **Deno compile** | Bundles V8 Engine + Rust runtime | **~75 MB – 100 MB** | ~35 – 50 ms |
| **Docker container** | Full Linux userland just to run 1 JS file | **~150 MB – 400 MB** | ~500 – 1500 ms |
| **jsib (Static ELF)** | **AOT C translation + Naked Musl + DCE** | **92 KB – 122 KB** | **0.30 ms (~302 µs)** |

For simple utilities, micro-services, system scripts, and background daemons, paying an **85-megabyte tax** for a loop that prints five lines of text is unacceptable.

---

## What is scriptc? (And Why Does jsib Exist?)

To understand `jsib`, you have to understand **[scriptc](https://github.com/vercel-labs/scriptc)**:

`scriptc` is an official **Vercel Labs experiment** that compiles JavaScript and TypeScript directly into readable C and textual LLVM IR. It uses the official TypeScript compiler front-end for AST parsing, semantic analysis, and type checking. Unlike Node or Bun, it does **not** embed the V8 engine, QuickJS, or an interpreter for static code.

### The Catch with Standard `scriptc`:
While `scriptc` is a brilliant compiler, its default pipeline is designed around the heavy LLVM/Clang ecosystem:
* It defaults to Clang, LLVM backends, or heavy platform SDKs.
* It targets standard dynamic glibc environments on Linux, resulting in external `.so` dependencies or glibc static linking issues.
* It does not perform aggressive naked-runtime dead-code elimination out of the box.

### The `jsib` Intervention:
`jsib` acts as a high-performance **hijack wrapper**:
1. We invoke `scriptc` strictly as an **AST-to-C transpiler** (`--backend c --emit-ir`), extracting pure C code and completely bypassing Clang and the LLVM backend.
2. We raid the `@scriptc/runtime` C pack and discard glibc entirely.
3. We inject our own **bare-metal AMD64 assembly coroutine engine** (`swapcontext` / `makecontext`) so async fibers run on naked Musl.
4. We compile the bundled vendor engines (**QuickJS `libregexp`**, **`zlib`**, and **`mbedTLS`**) directly into the static archive.
5. We pass the whole pipeline through a custom **W^X single-page linker script** with GNU `ld --gc-sections` and `sstrip`.

The result: You get the language ergonomics of TypeScript/JavaScript with the binary footprint and startup speed of a hand-rolled C program.

---

## Runtime Coverage (What Actually Works?)

`jsib` is not a toy Hello-World runner. The precompiled static runtime cache (`libscruntime.a`) includes the core operational surface of modern Node.js and TypeScript CLI programs:

| Feature / API | Status | Implementation Details |
| :--- | :---: | :--- |
| **Async / Await & Promises** | ✅ Full | Powered by our 15-instruction AMD64 naked stack pivot |
| **Regular Expressions (`RegExp`)** | ✅ Full | QuickJS's native `libregexp.c` & `libunicode.c` |
| **Compression (`node:zlib`)** | ✅ Full | Built-in standard `zlib` (`deflate`, `inflate`, `gzip`) |
| **HTTPS & TLS Encryption** | ✅ Full | Built-in embedded `mbedTLS` library (TLS 1.2 / 1.3) |
| **File System (`node:fs`)** | ✅ Full | `readFile`, `writeFile`, `stat`, directories via Musl POSIX syscalls |
| **Child Processes (`child_process`)** | ✅ Full | `spawn`, `execSync`, pipes via `fork`, `execve`, `pipe2` |
| **JSON Engine** | ✅ Full | Pure C recursive-descent parser (`scr_json.c`) |
| **Buffers & Bytes I/O** | ✅ Full | `Buffer`, `Uint8Array`, raw memory manipulation |
| **Console & Strings** | ✅ Full | Formatted output, string concatenation, slicing |
| **Maps & Sets** | ✅ Full | Built-in C hash map tables (`scr_map.c`) |
| **URL Parsing** | ✅ Full | WHATWG-compliant pure C URL parser (`scr_url.c`) |
| **Crypto (`arc4random_buf`)**| ✅ Full | Hardware entropy via direct Linux kernel `SYS_getrandom` |

---

## The Philosophy & The Scaling Curve

Let's be completely transparent: **The 90 KB footprint is the baseline for micro-workloads** (CLI utilities, scripts, algorithms, async loops, and simple tools).

`jsib` does not emit a static binary with fixed size. Because it uses section-level Dead-Code Elimination (`--gc-sections`), you only pay for the exact runtime modules your JavaScript actually executes:

```text
┌────────────────────────────────────────────────────────┐
│  Baseline (Async/Await + Console + Strings):  ~90 KB   │
├────────────────────────────────────────────────────────┤
│  + Regular Expressions (QuickJS libregexp):   ~122 KB  │
├────────────────────────────────────────────────────────┤
│  + File System I/O & Buffers:                 ~180 KB  │
├────────────────────────────────────────────────────────┤
│  + Full JSON Parser & String Utilities:       ~260 KB  │
├────────────────────────────────────────────────────────┤
│  + Compression & HTTPS (zlib + mbedTLS):      ~380 KB  │
├────────────────────────────────────────────────────────┤
│  + Kitchen Sink (Every single feature linked):~550 KB  │
└────────────────────────────────────────────────────────┘
```

Even at the absolute maximum with every single runtime feature pulled in, the entire static executable is **~550 KB**—still **99.3% smaller** than a standard Node.js Single Executable Application (~85 MB) or Bun compile (~60 MB).

---

## Live Silicon Benchmarks

Measured on **Artix Linux x86-64 physical hardware**:

### 1. Micro-Benchmark: Cold Startup & Hardware Counters

Measured directly on physical silicon using Linux `perf stat -r 100` executing the asynchronous loop:

```text
$ perf stat -r 100 ./hello >/dev/null

 Performance counter stats for './hello' (100 runs):

                 0      context-switches:u               #      0.0 cs/sec
                 0      cpu-migrations:u                 #      0.0 migrations/sec
                11      page-faults:u                    #  46,492 faults/sec      ( +-  0.45% )
              0.24 msec task-clock:u                     #  Active execution time  ( +-  1.55% )
             4,736      branches:u                       #  20.0 M/sec             ( +-  0.00% )
            34,616      cpu-cycles:u                     #  Total silicon cycles   ( +-  0.89% )

       0.000302593 +- 0.000004500 seconds time elapsed   ( +-  1.49% )
```

* **Cold Startup Latency:** **302 microseconds** (0.0003s elapsed wall-clock).
* **Demand Paging:** Only **11 page faults** to bootstrap the entire static runtime, allocate the async fiber stack, format strings, and flush to `stdout`.
* **Zero Context Switches:** Async fibers switch stacks strictly in user-space registers via our naked assembly shims without kernel scheduling traps.
* **Dependencies:** Zero. Verified with `ldd hello` -> `not a dynamic executable`.

---

### 2. Mega-Scale Synthetic Stress Test (1,000,000 Lines)

We stressed the pipeline with an extreme **1,000,000-line JavaScript program** (17 MB source text, ~250,000 functions):

```text
$ time jsib mega.js
⚡ [1/3] Transpiling mega.js...
🔨 [2/3] Compiling mega.c...
📦 [3/3] Linking static binary with --gc-sections...
✅ Done!
-rwxr-xr-x 1 user user 86K Sep 27 18:18 mega

real    0m36.372s
user    0m40.411s
sys     0m5.499s

$ ./mega
1M line test accumulator: 180
```

> **The Architectural Framing:**
> `jsib` is not designed to compete with sub-millisecond single-pass C compilers like [HT-Speed](https://github.com/TheMaster1127/HT-Speed). The goal of `jsib` is to be **the only toolchain in existence that can compile a 1,000,000-line JavaScript program into an 86 KB hardened static binary**.
> 
> Taking 36 seconds to traverse a 1-million-line TypeScript AST, emit 17 MB of C IR, run GCC instruction selection across 250,000 independent ELF sections with `-ffunction-sections`, and execute graph-traversal DCE in `ld`—where the industry alternative is shipping an **85 MB Node.js runtime that burns 100 ms of CPU warm-up on every launch**—is an extraordinary reduction in software weight.

---

## How It Works: Step-by-Step Architecture

```text
[ JavaScript / TypeScript ]
            │
            ▼
   ┌─────────────────┐
   │    scriptc      │  Lowers AST & TS types directly to typed C IR
   └────────┬────────┘
            ▼
   ┌─────────────────┐
   │   Musl Splicing │  Bypasses glibc; links to naked Musl CRT (crt1.o)
   └────────┬────────┘
            ▼
   ┌─────────────────┐
   │ Assembly Fibers │  Injects naked x86-64 swapcontext/makecontext shims
   └────────┬────────┘
            ▼
   ┌─────────────────┐
   │ GCC (-Os -ffs)  │  Compiles each function into an isolated ELF section
   └────────┬────────┘
            ▼
   ┌─────────────────┐
   │ GNU ld (--gc)   │  Discards unreferenced sections & enforces W^X
   └────────┬────────┘
            ▼
   ┌─────────────────┐
   │     sstrip      │  Obliterates Section Header Table
   └────────┬────────┘
            ▼
   [ ~90 KB Hardened Static ELF ]
```

### 1. AOT IR Lowering (scriptc Interception)
The source `.js` or `.ts` file is passed to `scriptc` with `--backend c --emit-ir`. The TypeScript compiler parses the syntax tree, verifies types, and emits a readable `.c` file that implements the program logic using `@scriptc/runtime` primitives.

### 2. Naked Musl Splicing & Decapitation
Standard glibc static libraries contain conflicting `.note.GNU-stack` section flags and complex dynamic hooks that trigger linker errors when building minimal binaries. `jsib` bypasses glibc completely, wiring directly into Musl's naked C runtime (`crt1.o`, `crti.o`, `libc.a`, `crtn.o`).

### 3. The Naked AMD64 Assembly Stack Pivot
Musl deliberately does not implement the deprecated POSIX `<ucontext.h>` functions (`swapcontext`, `getcontext`, `makecontext`) which ScriptC relies on for `async/await` execution. `jsib` dynamically synthesizes pure x86-64 assembly routines:
* **The Stack Pivot:** Saves callee-saved registers (`rbx, rbp, r12-r15`), steals the caller's return address from `(%rsp)`, updates the target's RIP, pivots `%rsp` to the heap-allocated fiber buffer, and executes `ret`.
* **Zero Overhead:** Context switches occur entirely in user-space in **~15 CPU instructions** without triggering a single kernel context-switch trap.

### 4. Vendor Engine Integration (Regex, zlib & mbedTLS)
Rather than pulling in dynamic shared objects or stripping functionality, `jsib` indexes the bundled vendor libraries in `@scriptc/runtime/vendor`:
* QuickJS's `libregexp.c` provides full ECMAScript Regular Expression matching.
* Standard `zlib` provides compression/decompression.
* `mbedTLS` provides TLS 1.2/1.3 cryptographic socket communications.

### 5. Section-Level Isolation & Dead-Code Elimination
The runtime files, vendor engines, and generated C code are compiled with `-ffunction-sections` and `-fdata-sections`. This instructs GCC to place every single C function and global variable into its own unique ELF section (e.g. `.text.scr_crypto_sha256` or `.text.lre_compile`).

### 6. The Hardened W^X Linker Script
When GNU `ld` links the binary with `--gc-sections`, it performs a graph traversal starting at `_start`. Any section not reachable from the execution path is permanently deleted. Furthermore, the custom linker script strictly separates code from data:
* **Code Segment (`PT_LOAD`):** Marked `PF_R | PF_X` (Read + Execute ONLY). Writing is strictly forbidden.
* **Data Segment (`PT_LOAD`):** Marked `PF_R | PF_W` (Read + Write ONLY). Execution is strictly forbidden.
* **Stack Segment (`PT_GNU_STACK`):** Non-executable stack.
* **The `/DISCARD/` Black Hole:** Purges all `.comment`, `.note.*`, and DWARF `.eh_frame` exception unwinding tables.

### 7. Section Decapitation (`sstrip`)
Standard `strip` only removes symbol and debugging entries, leaving the Section Header Table intact. `jsib` passes the binary through `sstrip` (from ELFkickers), which physically deletes the Section Header Table from the end of the file. The Linux kernel ELF loader requires only Program Headers (`Phdr`) to map and execute the binary.

### 8. Persistent Precompiled Runtime Cache
Compiling all core runtime modules and vendor libraries takes ~5 seconds. `jsib` precompiles the entire runtime once into an archive (`~/.cache/jsib/libscruntime.a`). Subsequent compilations only compile your application file, dropping compilation latency to **~0.4 seconds**.

---

## Scope & Limitations

* **Best For:** Standalone CLI tools, background workers, utility scripts, algorithmic computations, and embedded Linux environments where cold start latency and memory footprint are critical.
* **Dynamic Features:** JavaScript code that relies on runtime `eval()`, `vm.runInContext()`, or dynamic code generation is not supported statically.
* **Native C++ Addons:** NPM packages that require compilation via `node-gyp` (native C++ bindings) cannot be linked statically through this pipeline.
* **Architecture:** Currently optimized for **x86-64 Linux**. (ARM64 support is in development).

---

## Requirements & Clean NPM Setup

### 1. System Packages

#### Arch / Artix Linux:
```bash
sudo pacman -S gcc binutils musl elfkickers
```

#### Debian / Ubuntu:
```bash
sudo apt install build-essential musl musl-dev musl-tools
# Install sstrip from ELFkickers
git clone https://github.com/BR903/ELFkickers.git
cd ELFkickers/sstrip && make && sudo make install
```

#### Fedora / RHEL:
```bash
sudo dnf install gcc binutils musl-libc musl-devel
# Install sstrip from ELFkickers
git clone https://github.com/BR903/ELFkickers.git
cd ELFkickers/sstrip && make && sudo make install
```

---

### 2. Global NPM Setup (Recommended: No Sudo)

ScriptC compiles internal vendor cache prerequisites on demand. Installing global npm packages with `sudo` makes those directories read-only, which will cause permission errors.

The standard Linux recommendation is to configure npm to install global packages into your user directory:

```bash
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

npm install -g scriptc
```

*(If you previously installed scriptc with `sudo npm -g`, simply reclaim ownership with: `sudo chown -R $USER:$USER $(npm root -g)/scriptc`)*

---

## Installation

```bash
git clone https://github.com/TheMaster1127/JS-is-bloated.git
cd JS-is-bloated
sudo cp jsib /usr/local/bin/jsib
sudo chmod +x /usr/local/bin/jsib
```

---

## Usage & CLI Reference

### 1. Basic Async Example (`hello.js`):

```javascript
function print(val) {
    console.log(val);
}

async function main() {
    for (let i = 0; i < 5; i++) {
        print("Hello from jsib! Count: " + i);
    }
}

main();
```

Compile and run:
```bash
$ jsib hello.js
⚡ [1/3] Transpiling hello.js...
🔨 [2/3] Compiling hello.c...
📦 [3/3] Linking static binary with --gc-sections...
✅ Done!
-rwxr-xr-x 1 user user 90K Sep 27 18:00 hello

$ ./hello
Hello from jsib! Count: 0
Hello from jsib! Count: 1
Hello from jsib! Count: 2
Hello from jsib! Count: 3
Hello from jsib! Count: 4
```

---

### 2. Regular Expressions Example (`regex.js`):

```javascript
function test() {
    const text = "Found 1337 items in the database!";
    const match = text.match(/[0-9]+/);
    console.log("Matched numbers: " + match[0]);
}

test();
```

Compile and run:
```bash
$ jsib regex.js
✅ Done!
-rwxr-xr-x 1 user user 122K Sep 27 18:49 regex

$ ./regex
Matched numbers: 1337
```

---

### CLI Options:

```text
Usage:
  jsib <file.js|file.ts> [options]
  jsib -c                          # (re)build full cache only

Options:
  -o <name>         Output binary name (default: basename without .js/.ts)
  -c, --recache     Force rebuild the static runtime cache
  -k, --keep        Keep temporary C/ASM/LD artifacts in .jsib_build/
  -h, --help        Show this help
```

---

## Author & Ecosystem

Developed by **TheMaster1127** (aka *Mr. Compiler*), a low-level systems programmer and reverse engineer.

* **GitHub:** [@TheMaster1127](https://github.com/TheMaster1127)
* **Related Projects:**
  * [cib (C-Is-Bloated)](https://github.com/TheMaster1127/C-is-bloated) — Strips C binaries down to 169 bytes with zero libc.
  * [HT-Speed](https://github.com/TheMaster1127/HT-Speed) — Sub-millisecond x86-64 native compiler with zero libc.
  * [HT-RE](https://github.com/TheMaster1127/HT-RE) — Web-based reverse engineering & binary analysis environment.
  * [binpatch](https://github.com/TheMaster1127/binpatch) — Binary patching utility.

---

## License

This project is open-source software licensed under the **GNU General Public License v3.0 (GPLv3)**.
