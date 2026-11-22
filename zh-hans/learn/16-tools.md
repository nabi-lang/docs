# 16. 工具

本章介绍写代码时常用的工具：格式化工具、lint 工具、测试与基准测试、文档生成、编辑器支持（LSP）、REPL 与 Jupyter，以及用来查看编译器内部的调试选项。它们都从同一个 `nabi` 命令启动。

本章的输出来自实际运行下面这个包。它由 `nabi new mathx` 创建，并把 `src/main.nabi` 改成了这样：

```nabi build
//! Small math helpers.

/// Greatest common divisor of `a` and `b`.
pub fn gcd(a: i64, b: i64) -> i64 {
    if b == 0 { a } else { gcd(b, a % b) }
}

/// The `n`-th Fibonacci number.
pub fn fib(n: i64) -> i64 {
    if n < 2 { return n; }
    fib(n - 1) + fib(n - 2)
}

println(f"gcd(12, 18) = {gcd(12, 18)}, fib(10) = {fib(10)}");

#[test]
fn gcd_works() {
    assert_eq(gcd(12, 18), 6);
    assert_eq(gcd(7, 0), 7);
}

#[test]
fn fib_works() {
    assert_eq(fib(10), 55);
}

#[test]
fn fib_wrong() {
    assert_eq(fib(5), 6, "deliberately wrong");
}

#[bench]
fn fib_20(b: bench::Bench) { b.iter(|| fib(20)); }
```

## `nabi test`：测试

标有 `#[test]` 的函数就是测试。普通构建不包含测试函数；`nabi test` 会收集它们并单独构建。检查使用 `assert`、`assert_eq` 和 `assert_ne`，失败时触发 panic。可以把消息作为最后一个参数传入。

每个测试在各自的进程中运行，所以一个测试触发 panic 时，其余测试会继续运行。上面包中的 `fib_wrong` 是故意写错的。

```text
$ nabi test
   Compiling mathx (.)
    Finished in 0.69s

running 3 tests
test fib_works ... ok
test fib_wrong ... FAILED
test gcd_works ... ok

failures:

---- fib_wrong ----
thread 'main' panicked at src/main.nabi:29:5:
assertion `left == right` failed: deliberately wrong
  left: 5
 right: 6
(panicked (exit 101))

failures:
    fib_wrong

test result: FAILED. 2 passed; 1 failed; 0 ignored; 0 filtered out; finished in 0.22s
```

只要有测试失败，退出码就是 1，所以可以直接用于 CI。把名字的一部分（或用 `^`、`$`、`.`、`*`、`+`、`?` 写的简单正则表达式）作为参数传入，就只运行匹配的测试。

```text
$ nabi test gcd
   Compiling mathx (.)
    Finished in 0.37s

running 1 test
test gcd_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 2 filtered out; finished in 0.22s
```

用 `--show-output` 可以同时看到通过的测试的输出，用 `--threads=N` 设置同时运行多少个测试。对于 CI 仪表盘和其他工具，`--format json` 在标准输出上每行写一个 JSON 对象，每个测试一个，最后是一个汇总，并把给人看的文本移到标准错误：

```text
$ nabi test --format json 2>/dev/null
{"schema":1,"type":"case","kind":"test","name":"fib_works","result":"ok","ms":87.057}
{"schema":1,"type":"case","kind":"test","name":"fib_wrong","result":"failed","ms":87.144,"reason":"panicked (exit 101)"}
{"schema":1,"type":"case","kind":"test","name":"gcd_works","result":"ok","ms":87.05}
{"schema":1,"type":"summary","kind":"test","result":"failed","passed":2,"failed":1,"ignored":0,"filtered_out":0,"ms":87.276}
```

`--exact` 让过滤器只匹配完整的名字，`--skip <pat>` 排除匹配的测试（可以给多次以跳过多个）。`--list` 只打印名字，不运行任何测试。

### 预期 panic、忽略的测试与超时

应该触发 panic 的测试用 `#[should_panic]` 声明；带上 `expected` 时，panic 消息还必须包含该文本。运行很慢的测试，或者需要 CI 中没有的东西的测试，标上 `#[ignore]`，可以附带原因：

```nabi norun
#[test]
#[should_panic(expected = "deliberately")]
fn fib_wrong_panics() {
    assert_eq(fib(5), 6, "deliberately wrong");
}

#[test]
#[ignore = "takes a minute"]
fn fib_large() {
    assert(fib(90) > 0);
}
```

```text
test fib_large ... ignored, takes a minute
test fib_wrong_panics ... ok
```

`nabi test --ignored` 只运行被忽略的测试，`--include-ignored` 把它们和其余测试一起运行。卡住的测试不会卡住整个运行：600 秒后它会被终止并报告为失败。用 `--timeout=SECS`（或 `NABI_TEST_TIMEOUT`）设置其他时限；`--timeout=0` 关闭时限。

### 文档测试

在库中，`nabi test` 还会运行文档中的代码示例：`///` 和 `//!` 注释以及 `README.md` 中的每个 ```` ```nabi ```` 块，都会构建成一个使用该库的小程序并运行。库的公开项可以直接按名字使用，就像写了 `use mathx::*;` 一样。示例无法再编译，或者其中的 `assert` 失败，都会让测试运行失败，所以文档始终保持正确。

````text
/// The greatest common divisor.
///
/// ```nabi
/// # let (a, b) = (12, 18);
/// assert(mathx::gcd(a, b) == 6);
/// ```
pub fn gcd(a: i64, b: i64) -> i64 {
    // ...
}
````

以 `# ` 开头的行会运行，但在 `nabi doc` 中隐藏，所以示例可以做准备工作而不显示出来。围栏中 `nabi` 后面的词会改变检查的内容：`nabi,norun` 只构建示例（用于读取文件或等待网络的代码），`nabi,should_panic` 预期 panic，`nabi,compile_fail` 预期示例无法构建（`compile_fail,E0040` 指定某个错误），`nabi,ignore` 跳过它。其他语言（```` ```text ````、```` ```nabion ````）从不运行。

```text
running 2 tests
test doc README.md:9 ... ok
test doc src/lib.nabi:14 (gcd) ... ok
```

每个文档测试以其文件和行号命名，再加上它所记录的项。示例中的错误会指向你源文件中的那一行。`nabi test --doc` 只运行文档测试，`--no-doc` 跳过它们。

## `nabi bench`：基准测试

`#[bench]` 函数接受一个 `Bench`，把要计时的代码交给 `b.iter`。`nabi bench` 以优化方式（`-O2`）构建它们，多次调用每个闭包，并打印每次调用的中位时间及其离散程度。从不调用 `b.iter` 的基准测试会收到警告。数字因机器和每次运行而异。

```text
$ nabi bench
   Compiling mathx (.)
    Finished in 0.42s

benchmarking 1 bench
  fib_20      12.99 us +/- 120.52 ns

bench result: ok. 1 run; 0 failed; 0 filtered out; finished in 2.63s
```

`--iters=N` 是每个样本的调用次数（默认取足够约 100 ms 的次数），`--samples=N` 是样本数（默认 10）。过滤器的用法与 `nabi test` 相同。输出在终端和管道中看起来一样；读取结果的工具可以传入 `--format=nabibench`，每个基准测试输出一行 `#nabibench`，或者传入 `--format json`，输出与 `nabi test` 相同的 JSON 行，两种格式都带 `median_ns`、`mad_ns` 和 `samples`。

## 垃圾回收器停顿

垃圾回收器会让所有线程短暂停顿。两个环境变量可以显示这些停顿：`NABIR_GC_LOG=1` 在标准错误上为每次回收打印一行，`NABIR_GC_STATS=1` 在程序退出时打印汇总。

```text
$ NABIR_GC_LOG=1 ./target/release/server
nabi gc: #41 minor: 0.802 ms pause, 21046 objects marked, 141312 -> 104160 KiB; ttsp 9 us, mark 410 us, sweep 352 us, evac 0 us, threads 3, at 1520.114 ms
```

停顿从回收器要求线程停下的那一刻开始计时。`ttsp`（time to safepoint，到达安全点的时间）是等待线程停下所花的部分：`ttsp` 很长说明有线程一直在运行，而不是回收器慢。长字符串的搜索与计数、UTF-8 检查和大块复制不会拖住停顿；它们让回收器在旁边同时运行。`NABIR_GC_STATS=1` 的汇总最后按停顿种类（minor、full，以及并发标记的完整回收的开始和结束）各列一行，给出 p50、p90、p99 和最长值。

程序可以用 `bare::gc::stats()` 读取同样的计数器，用于指标端点或帧预算。它会短暂获取回收器的锁，所以偶尔读取即可，不要放在热循环里。

```nabi
use bare::gc;

let s = gc::stats();
println(f"{s.collections} collections, longest pause {s.pause_max_ns / 1000} us");
```

`bare::gc::collect()` 运行一次完整回收，并在整个过程中让所有线程停下：在帧之间调用它来“清理”只会让每一帧变得更长。要让停顿不落在帧内，请使用 `collect_young`、停顿目标、`no_collect` 和 arena（[第 25 章](25-gc-pauses.html)）。

## 其他目标

带上 `--target` 时，`nabi test`、`nabi bench` 和 `nabi run` 会通过一个运行器在本机上运行构建出的程序：`wasm32-wasip2` 和 `wasm32-wasip1-threads` 用 `wasmtime`，`wasm32-unknown-unknown` 用 `node`，在 Linux 上，其他 CPU 的 Linux 程序用 `qemu-<cpu>`（qemu-user）。它们都必须在 `PATH` 上；找不到时，错误信息会说明要安装什么。本机自己就能运行的程序不需要运行器：x86-64 Linux 上的 `i686-unknown-linux-gnu`，以及 arm64 Linux 上的 `armv7-unknown-linux-gnueabihf`，后者要求 CPU 也能运行 32 位 ARM 代码（Neoverse N1 和 Cortex-A72 可以，Neoverse N2 和 Apple 芯片不行），并且装有 armhf C 库（Debian、Ubuntu：先 `dpkg --add-architecture armhf`，再 `apt install libc6:armhf`）；`armv7-unknown-linux-musleabihf` 程序是静态的，在那里不需要 C 库。

```text
$ nabi test --target wasm32-wasip2 gcd
   Compiling mathx (.)
    Finished in 0.61s

running 1 test
test gcd_works ... ok

test result: ok. 1 passed; 0 failed; 2 filtered out; finished in 0.05s
```

要使用其他运行器，或者为没有运行器的目标指定一个，把 `NABI_RUNNER` 设为该命令；程序及其参数放在它后面：

```text
$ NABI_RUNNER="qemu-aarch64 -L /opt/aarch64-sysroot" nabi test --target aarch64-unknown-linux-gnu
```

在装有 Xcode 的 Mac 上，`aarch64-apple-ios-sim` 程序在已启动的 iOS 模拟器上运行（`NABI_SIM_DEVICE` 指定其他设备）；请见[第 28 章](28-ios.html)。

WASI 程序没有工作目录：相对路径从 `/` 开始查找，所以在 wasm 上运行的测试要使用绝对路径。

### Android

`x86_64-linux-android` 为 Android 10 及以后的版本构建程序和共享库，它们在 Android 模拟器（x86_64 系统镜像）中运行。面向 arm64 手机和平板（`aarch64-linux-android`）的构建仍在进行中：该目标尚未在设备上测试，会在以后的版本中提供。链接需要 Android NDK，可以从 Android Studio 的 SDK Manager（SDK Tools、NDK）或 NDK 下载页面获取；`nabi` 会在 Android SDK 中找到它，也可以把 `ANDROID_NDK_HOME` 设为它的位置：

```text
$ export ANDROID_NDK_HOME=~/Android/Sdk/ndk/30.0.16248370
$ nabi build --target x86_64-linux-android --release
$ adb push target/x86_64-linux-android/release/hello /data/local/tmp/
$ adb shell /data/local/tmp/hello
Hello, world!
```

供应用的 Java 或 Kotlin 代码使用的库是一个 `cdylib`，其导出函数使用 JNI 的命名；把 `.so` 放到应用的 `jniLibs/x86_64/` 下（设备构建推出后则是 `arm64-v8a/`），并用 `System.loadLibrary` 加载：

```nabi
use bare::ptr::CPtr;

#[export_name("Java_com_example_app_Native_add")]
pub extern "C" fn add(env: CPtr<c_void>, class: CPtr<c_void>, a: i32, b: i32) -> i32 { a + b }
```

```text
$ nabi build --target x86_64-linux-android --release --crate-type=cdylib
```

程序在 Android 10（API level 29）及以后的版本上运行；`ANDROID_PLATFORM=34` 为更新的版本构建。程序和库可以在 4 KB 或 16 KB 页面的设备上加载，这正是 Google Play 对 Android 15 及以后版本的应用的要求。由于内核就是 Linux，Linux 程序大多可以原样运行。不同之处请见[平台说明](../spec/refs-lang/000-meta/006-platform-targets.html#android)。

## `nabi fmt`：格式化工具

`nabi fmt` 把包中的 `.nabi` 和 `.nabion` 文件改写成标准格式。Nabi 的格式化工具刻意保守：它**不改动词法单元、行内空白和注释**，只修正缩进（每层括号 4 个空格）、行尾空白和连续的空行。它唯一会移动的是模块顶部的 `mod` / `use` 行：按 `pub mod`、`mod`、`pub use`、`use` 分组，每组内排序，组与组之间空一行。所以格式化工具绝不会改变代码的含义，也不会破坏人手工设置的对齐。

格式化工具像编译器一样读取运算符词法单元：包自己的 `#[infix]` 词法单元以及其依赖导出的词法单元（从 `nabi.lock` 读取，不获取任何东西），所以使用 `a ⊛ b` 或 `x |> f` 的测试、基准测试和依赖方也能格式化。

用 `--check` 检查一个缩进混乱、混用制表符并带行尾空白的 `fib`，文件不会被改动，它会显示将要改变的行，并以退出码 1 退出。这种模式很适合 CI。

```text
$ nabi fmt --check
Diff in ./src/main.nabi:
-   10   if n < 2 { return n; }
-   11   let unused = 1;
-   12 	fib(n - 1) + fib(n - 2)
+   10     if n < 2 { return n; }
+   11     let unused = 1;
+   12     fib(n - 1) + fib(n - 2)
        note: 1 file needs formatting; `nabi fmt` without `--check` rewrites it
$ nabi fmt
   Formatted 1 file, 1 already formatted
$ nabi fmt --check
        note: 2 files already formatted
```

不带 `--check` 时，它会改写文件并打印改了多少个；`-q` 省略这一行。在编辑器中每次保存时运行它很方便。

在包内，`nabi fmt` 从任何子目录都会格式化整个包，跳过 `target/`、隐藏目录、`node_modules/` 以及 `.gitignore` 中列出的路径。在包外，要指明格式化什么：一个文件（`nabi fmt hi.nabi`）或一个目录（`nabi fmt scripts/`）。不给路径时，它会停下并说明原因：

```text
$ nabi fmt
nabi: error: not inside a package; pass a file or directory: `nabi fmt <path>`
```

## `nabi lint` 与 `nabi check`

`nabi lint` 不编译，只运行类型检查和 lint。它能发现上面留下的 `let unused = 1;`。

```text
$ nabi lint
    Checking mathx (.)
warning[E0131]: unused variable `unused`
   --> src/main.nabi:11:9
    |
 11 |     let unused = 1;
    |         ^^^^^^
    |
    = note: if this is intentional, prefix it with an underscore: `_unused`
    = note: `#[warn(unused_variables)]` on by default
     Checked mathx: 0 errors, 1 warning
    Finished in 0.05s
```

每个 lint 默认都是 `warn`，例外是 `nan_comparison` 和 `self_assignment`，它们总是错误，默认为 `deny`；还有 `missing_docs`（没有 `///` 注释的 `pub` 项），在你开启之前它是 `allow`。级别可以在三个地方修改，后面的优先：

1. `nabi.nabion` 中的 `lints` 字段：`lints: { "unused": "deny", "dead_code": "allow" }`
2. 命令行上的 `-A` / `-W` / `-D`（allow / warn / deny）
3. 源代码中的属性：项上的 `#[allow(dead_code)]`，文件顶部的 `#![deny(unused)]`

除了单个 lint（`unused_variables`），也可以指定一个组（`unused`、`nonstandard_style`、`warnings`、`all`）。只要有 `deny` 级别的 lint 触发，退出码就是 1。

`unused_result` 会对丢弃了值的语句发出警告，比如 `xs.len();` 或 `parse(s);`，因为这通常是失误。确实要丢弃时，写 `_ = expr;`。调用方常常不需要其结果的函数，比如 `Dict::insert`、`List::pop` 和 `Atomic::fetch_add`，标有 `#[discardable]`，调用它们的语句不会产生警告；你自己的函数也可以这样标记。

```text
$ nabi lint -D unused
    Checking mathx (.)
error[E0131]: unused variable `unused`
   --> src/main.nabi:11:9
    |
 11 |     let unused = 1;
    |         ^^^^^^
    |
    = note: if this is intentional, prefix it with an underscore: `_unused`
    = note: `-D unused` on the command line
     Checked mathx: 1 error, 0 warnings
    Finished in 0.04s
```

要让 CI 在有警告时失败，就 deny `warnings`（`lints: { "warnings": "deny" }`，或 `-D warnings`）。和 rustc 一样，它涵盖所有警告级别的 lint，包括以后的版本新增的 lint，所以当新版本不能改变 CI 接受的内容时，请在 `nabi.nabion` 中固定确切的工具链（`toolchain: "=0.1.0"`）。`all` 是所有 lint。在 `nabi.nabion` 中写了 `"warnings": "deny"` 时，最后一条注记会说明级别来自哪里：

```text
$ nabi lint
    Checking mathx (.)
error[E0131]: unused variable `unused`
   --> src/main.nabi:11:9
    |
 11 |     let unused = 1;
    |         ^^^^^^
    |
    = note: if this is intentional, prefix it with an underscore: `_unused`
    = note: `"warnings": "deny"` in `lints` of nabi.nabion
     Checked mathx: 1 error, 0 warnings
    Finished in 0.11s
```

有些发现会附带解决它的修改：未使用的 `let` 加上前导 `_`，未使用的 `use` 行被删除，`while true` 变成 `loop`，`x == true` 变成 `x`，多余的 `as` 或 `.clone()` 被去掉，含有 `{name}` 的普通字符串加上 `f`。编辑器会把每一项作为快速修复提供；`nabi fix` 在整个包中应用所有修复，并反复检查，直到没有剩余：

```text
$ nabi fix
       Fixed src/main.nabi (2 fixes)
```

`nabi fix` 只修改你的包中的文件，并且在工作树有未提交的更改时拒绝运行，所以每处修复都会出现在 `git diff` 中，可以撤销（`--allow-dirty` 跳过这项检查）。`nabi fix --diff` 打印修复而不写入。lint 级别同样适用：`nabi fix -A unused_imports` 不动导入。遇到编译错误时，它会在做任何修改之前停下；请先手工修好。

`deprecated` 会对标有 `#[deprecated]` 的项的任何使用发出警告：函数、方法、字段、类型、常量或静态变量，无论来自你的代码、依赖还是标准库；编辑器会把这些使用显示为删除线。用 `nabi lint --list` 查看完整列表，其中还给出了每个 lint 是在哪个版本加入的（把它通过管道传给 `head` 没有问题：当读取输出的一方离开时，`nabi` 和 `nabic` 会安静地停止，而不是报错）。`nabi check` 更进一步，还会运行代码生成，但不链接，也不产生可执行文件。它比构建快，所以用它来回答“能编译吗？”。从这里开始的示例使用一个临时的 `hello` 包，这样错误和底层输出就不会干扰 `mathx`。

```text
$ nabi check
    Checking hello (.)
error[E0040]: mismatched types: `let s` expects `str`, found `i32`
  --> src/main.nabi:2:14
   |
 2 | let s: str = square(7);
   |              ^^^^^^^^^
   |
   = help: to make text from it, write `f"{square(7)}"`
     Checked hello: 1 error, 0 warnings
    Finished in 0.04s
```

`nabi build` 和 `nabi run` 打印同样的诊断，构建失败时以计数结尾，而不是 `Finished`：

```text
$ nabi build
   Compiling hello (.)
...
error: could not build hello (1 error, 0 warnings)
```

要得到机器可读的输出，加上 `--message-format=json`。每条诊断打印为一行 JSON（`"schema": 1`）：它的代码和种类、消息、带起止行列的范围、`notes` 和 `help` 行、它建议的 `fixes`，以及 lint 警告的 `lint` 名。

方括号中的代码指明错误，每种错误一个代码（所有 lint 共用 `E0131`，并在注记中写出自己的名字）。`nabic --explain E0040`（或 `nabi --explain E0040`）打印该代码的含义和修复方法，[错误码索引](../errors/index.html)列出了所有代码。如果 nabic 本身崩溃（内部编译器错误），它以退出码 70 退出，与触发 panic 的程序的 101 不同；请报告这个问题。

## `nabi doc`：文档注释与 API 文档

`///` 为紧接在它下面的项编写文档，`//!` 为包含该注释的模块（文件）编写文档。在上面的包中，第一行 `//! Small math helpers.` 成为模块说明，`gcd` 和 `fib` 上方的 `///` 成为函数说明。普通的 `//` 注释从来不是文档，即使在文件顶部也不是：当库的根文件或 `pub` 模块的文件以 `//` 行开头却没有 `//!` 时，`nabi lint` 会发出警告（`plain_module_doc`），帮助信息会给出应该写的 `//!` 行。文档文本可以使用 markdown，比如标题、列表、行内代码、代码块和链接。其中的 ```` ```nabi ```` 示例由 `nabi test` 测试（请见[文档测试](#文档测试)）。

```text
$ nabi doc
 Documenting mathx (.)
    Finished in 0.10s
   Generated ./target/doc/mathx/index.html
```

`nabi doc --open` 还会在浏览器中打开该页面。

`target/doc/` 中会生成包及其依赖的公开（`pub`）项的 HTML，以及一个搜索索引：包的页面是 `target/doc/<package>/index.html`，每个公开模块一个页面（`<package>/geo.html`、`<package>/geo/shapes.html`）。每个页面的侧栏列出包的依赖，并链接到它们的页面；搜索能找到所有已生成文档的包中的项。通过 `pub use` 从私有模块引入的项，显示在重新导出它的页面上，就像在那里声明的一样；重新导出公开模块中某个项时，会链接到那个模块的页面。这些页面的外观与本站的[标准库 API](../../std/core/index.html) 相同，遵循同样的规则，并且可以离线使用，直接从磁盘打开即可。

## 编辑器：`nabils`

`nabils` 是一个 LSP（Language Server Protocol）服务器。它提供诊断、悬停提示、跳转到定义、查找引用、重命名、文档符号、补全、签名帮助、快速修复和格式化。补全会提供作用域中的局部变量、值的成员（`xs.` 列出 `List` 的公开方法）以及 `use` 之后的路径。悬停提示和跳转到定义也能深入标准库。快速修复会应用诊断建议的修复，补上缺少的 `;` 或 `use`，并为缺少的包运行 `nabi add`。重命名会拒绝库中的项、关键字和来自其他文件的名字，并说明原因。诊断跟随你的输入，而不是你的保存：每次编辑都会在后台重新检查，在此期间，悬停提示、补全和跳转到定义会立即根据上一次成功的检查作答。所以从网页粘贴来的字符（零宽空格、代替 `-` 的 `−`、弯引号）、孤立的组合标记、格式错误的数字、错误的转义，或者没有处理器接受的字面量后缀（注册的是 `hz` 时写的 `10Hz`），都会立即被标出，附带编译器自己的消息；在修复方法明确时，还会提供写好修复的快速修复。每个文件都用它所在包的依赖来检查，所以包含多个包的文件夹也能正常工作；通过快速修复添加的依赖，或者编辑并保存 `nabi.nabion` 添加的依赖，不用重启服务器就能找到。`nabi lsp` 是同一个服务器。它通过标准 IO 通信，所以任何支持 LSP 的编辑器只需告诉它要运行的命令（编辑器附加的 `--stdio` 也能接受）。文件有类型错误时，悬停提示仍会作答：使用那次检查得出的类型，否则使用上一次通过的检查。在终端中手动启动时，`nabils` 只会说明它应由编辑器启动。

对于 Neovim（0.11 或更高版本），把下面的内容加到 `init.lua`：

```text
vim.filetype.add({ extension = { nabi = "nabi", nabion = "nabion" } })
vim.lsp.config("nabils", {
  cmd = { "nabils" },
  filetypes = { "nabi", "nabion" },
  root_markers = { "nabi.nabion" },
})
vim.lsp.enable("nabils")
```

对于 Helix，把下面的内容加到 `languages.toml`：

```text
[language-server.nabils]
command = "nabils"

[[language]]
name = "nabi"
scope = "source.nabi"
file-types = ["nabi"]
roots = ["nabi.nabion"]
language-servers = ["nabils"]
```

在 VS Code 等其他编辑器中，用通用的 LSP 客户端扩展注册 `nabils`。`nabils` 必须在你的 `PATH` 上。

## REPL 与 Jupyter

进一步看看第 1 章中的 REPL（`nabi` 或 `nabi repl`）。在终端上，它启动时显示两行：工具链版本和主机，以及一条提示。`nabi repl -q` 省略这两行，管道输入时则从不显示。

```text
$ nabi repl
Nabi 0.1.0 on aarch64-apple-darwin
Type :help for help, :exit to quit.
>>> let xs = [3, 1, 2]
>>> xs.len()
3
>>> it * 10
30
>>> xs.sort(); xs?
[1, 2, 3]
>>> :type xs
List<i64>
>>> fn twice(x: i32) -> i32 {
...     x * 2
... }
>>> twice(21)
42
>>> exit
```

提示符是 `>>>`，单元尚未结束时是 `...`。单元在行尾结束，除非输入明显没有写完：括号、字符串或块注释还没有闭合，行以运算符、`.`、`,` 或 `=` 结尾，或者是 `fn f(x: i32)` 这样仍在等待主体的项头。单元的最后一条语句是 `let` 或赋值时，可以省略它的 `;`。

每个单元只运行一次。顶层的 `let` 绑定以及 `fn` / `struct` / `impl` 定义会延续到后面的单元，单元也可以在顶层 `await`。表达式单元用 `Display` 打印它的值，没有则用 `Debug`，再没有就只打印类型。以 `?` 结尾则改为打印 `Debug` 形式。`let` 单元不显示任何内容。由多条语句组成的单元，如果以表达式结尾，就显示该表达式的值。单元显示的值会保存为 `it`，下一个单元可以使用它；一旦你自己定义了 `it`，就以你的为准。

与普通源代码不同，你可以再次 `let` 同一个名字，甚至换成另一个类型。再次定义 `fn` 或 `struct` 会替换它，之前单元的代码会调用新版本。编译失败、触发 panic 或抛出错误的单元会让会话保持原样。它已经写出的输出不会消失。`:reset` 从空会话重新开始。

错误以 `<repl>:line:col` 指向你输入的单元。panic，以及之前单元中某个 `fn` 里的错误，以 `<repl cell N>:line:col` 指明那个单元，其中 N 是到目前为止输入的单元数。在 Jupyter 中，N 是笔记本单元的执行计数，一个笔记本单元拆分成多个 REPL 单元时，编号为 `N.1`、`N.2`，以此类推。

```text
>>> fn boom(n: i32) {
...     panic(f"boom {n}");
... }
>>> boom(5)
thread 'main' panicked at <repl cell 1>:2:5:
boom 5
error: the cell panicked; the session is unchanged
```

| 命令 | 作用 |
|------|---------|
| `expr?` | 打印 `Debug` 形式而不是 `Display` |
| `:type expr`（`:t`） | 打印类型，不运行 |
| `:doc name` | 打印某个名字的签名和文档注释；方法用 `:doc xs.len` |
| `:time expr` | 运行表达式，并打印每个阶段花了多长时间 |
| `:vars` | 以 `name: type` 列出绑定，然后列出各项 |
| `:load file`，`:reload` | 把程序加载到会话中；重新加载上一个。`:save` 生成的脚本会逐个单元重放 |
| `:edit` | 在 `$VISUAL` / `$EDITOR` 中编写单元，然后运行 |
| `:save file` | 把会话写成脚本，标记好的单元一个接一个 |
| `:opt [level]` | 显示或设置后续单元的优化级别（`0` 到 `3`，`s`） |
| `:reset` | 忘掉所有单元，重新开始 |
| `:add name ..` | 添加依赖：在环境中 `nabi install`，在包中 `nabi add`。单元用到的官方包无需它也会添加 |
| `:help`（`:h`） | 列出所有命令 |
| `:exit`，`:quit`（`:q`） | 退出；单独输入 `exit` 或 `quit`，或者输入结束，也会退出 |

捕获了变量的闭包不能保存在单元的顶层 `let` 中，因为每个闭包都有自己的匿名类型。这个单元会被拒绝：

```text
>>> let x = 3
>>> let f = |n: i64| n * x
error: `f: closure(i64) -> i64` cannot be kept for later cells (its type has no name)
```

给绑定一个函数类型，`let f: fn(i64) -> i64 = |n: i64| n * x`，它就能延续下去。（`fn` 也能延续，但它看不到 `x` 这样的单元变量；请把它们作为参数传入。）在单元内部使用的闭包，比如 `xs.map(|n| n * x)`，照常工作。

在终端上，REPL 自己编辑输入行：

- Tab 补全名字：作用域中的局部变量、会话的绑定和项、core 库、关键字，`x.` 之后是 `x` 的公开字段和方法，`a::` 之后是模块 `a` 的项。在行首，Tab 用于缩进。
- 历史记录保存整个单元，并跨会话保留（`$NABI_HOME/repl_history`）。上下方向键在其中移动；已经输入了一些文本时，只在以它开头的单元之间切换。Ctrl-R 向后搜索。
- 粘贴的一段代码在按下 Enter 时成为一个单元，而不是逐行运行。
- 输入的着色与本指南相同。设置 `NO_COLOR` 可以关闭颜色。

读取标准输入的单元读的是终端。REPL 自身的输入来自管道时，单元看到的是空输入。

`nabi repl path` 把一个包或文件加载到会话中。对于库，该包作为依赖添加，并先运行 `use name::*;`，`:reset` 之后也会再运行一次。在构建库时用来试用函数很方便。对于程序，它会运行程序，然后把程序的绑定和项保留在会话中。在包内，`nabi repl .` 加载当前包；单纯的 `nabi repl` 从空会话开始。

同样的单元引擎也可用于 Jupyter 笔记本。注册一次内核，就可以在 Jupyter、VS Code 笔记本等环境中选择 `nabi` 内核。

```text
$ nabi jupyter install
   Installed kernel `nabi` (/Users/me/Library/Jupyter/kernels/nabi/kernel.json)
```

它在 macOS 上安装到 `~/Library/Jupyter`，在 Windows 上安装到 `%APPDATA%\jupyter`，在其他系统上安装到 `$XDG_DATA_HOME/jupyter`（默认 `~/.local/share/jupyter`）；`JUPYTER_DATA_DIR` 可以改变位置。`nabi jupyter install --env name` 为指定名字的环境安装内核。

笔记本单元遵循 REPL 的规则，显示最后一个表达式的值。类型实现了 `RichDisplay` 的值（比如绘图的 figure）按其 `mime_bundle()` 给出的内容显示（SVG 图像、HTML），`display(x)` 可以在单元中间显示一个值。输出在单元运行时就会出现，而不只是在结束时。读取标准输入的单元会在笔记本中得到一个输入框。Tab 补全和 Shift-Tab 帮助使用与 REPL 相同的名字。在笔记本前端认识 Nabi 之前，代码按 Rust 的规则高亮。

## 优化级别

`nabi build` 构建 `debug` 构建配置，`--release` 构建 `release` 构建配置。`nabi` 和 `nabic` 也直接接受级别：

| 级别 | 用途 |
|-------|----------------|
| `-O0` | 默认：不优化，构建最快，每个值都在调试器预期的位置 |
| `-O1` | 运行较快的调试构建：循环得到 `-O2` 给的优化（遍历列表的 `for` 变成计数循环，能向量化时就向量化），其他地方内联很少，构建保留调试信息 |
| `-O2` | `--release`：快速的代码 |
| `-O3` | 用更多代码换取一点额外速度：循环中内联更多，标量循环（浮点求和、主体中有调用）会被展开 |
| `-Os` | 小代码：只内联小函数，循环仍是计数循环，较短时会向量化 |

`-O2`、`-O3` 和 `-Os` 构建 `release` 构建配置，`-O0` 和 `-O1` 构建 `debug` 构建配置。`-g` 为 release 构建添加调试信息，而不改变其代码。

## 构建缓存

构建时会编译程序用到的标准库函数，并把编译好的代码保存在工具链的缓存中，这样下一次构建任何包都可以直接从那里取用。缓存最多占用 5 GiB：超过后，构建会删除最久未使用的内容，直到剩下 4 GiB，还会删除一个月内没有被任何构建使用的内容。设置 `NABI_CACHE_MAX_SIZE` 可以改变上限（`NABI_CACHE_MAX_SIZE=20G`，或用 `off` 表示不设上限）。`nabi cache` 用于查看和清理缓存：

```bash
nabi cache dir                  # 位置
nabi cache prune                # 立即缩减到上限
nabi cache prune --max-size 1G  # 最多保留 1 GiB
nabi cache clean                # 清空
```

工具链还以只读缓存的形式附带官方包，解析依赖时会先尝试它。`nabi component list` 显示它们是否存在，`nabi component remove packages` 删除它们；`nabi component add packages` 重新下载。

## 查看编译器内部

要查看每个构建阶段花了多长时间，加上 `-v`。

```text
$ nabi build -v
   Compiling hello (.)
   Stage: front end (111ms)
      Reused: 168 of 169 files parsed, 0 of 1 bodies lowered, the library's items collected
    Building ./target/debug/hello
       Units 26 reused, 2 compiled
   Stage: codegen + link (22ms)
    Finished in 0.14s
```

`--emit` 把中间产物写到 `target/debug/build/`。种类有 `source`（元展开和脱糖之后的整个包，以 Nabi 源代码写出）、`hir`、`mir`（代码生成器编译的内容；高于 `-O0` 时是优化后的）、`obj`（目标文件）、`asm`（目标文件的汇编源代码形式，见下文）、`listing`（带偏移和字节的机器码，见下文）以及 `expansions`（每次元展开的输出）。单独的 `nabi emit` 写出 `mir`。

```text
$ nabi emit
   Compiling hello (.)
       Wrote ./target/debug/build/hello.mir
    Finished in 0.04s
$ grep -A 6 '^fn square' target/debug/build/hello.mir
fn square(_0: i32) -> i32 {
    let _1: i32;

    bb0:
        _1 = Mul<i32>(_0, _0);
        return _1
}
```

`asm` 像 `clang -S` 一样把目标文件写成汇编源代码：节、符号、每个函数的指令（带局部标签，在重定位处写出符号名，如 `call nabi_panic_at@PLT`）以及数据。对它进行汇编（`clang -c`）会得到同样的代码、重定位和符号，所以结果可以像目标文件一样链接和运行。汇编器根据文本不会那样编码的指令，会被强制指定编码（`{disp32}`、`mov.s`），或者写成字节，并在注释中给出指令。展开表写成汇编器用来生成它们的伪指令（`.cfi_*`；x64 Windows 上是 `.seh_*`；32 位 ARM 上是 `.fnstart` .. `.fnend`；arm64 Windows 上是数据），调试信息则是数据。在 wasm32 上，它是 LLVM 的 WebAssembly 汇编，可以用 `clang --target=wasm32-wasi -msimd128 -c` 汇编：

```text
$ nabi emit --emit=asm --target x86_64-unknown-linux-gnu
   Compiling hello (.)
       Wrote ./target/x86_64-unknown-linux-gnu/debug/build/hello.s
    Finished in 0.60s
$ grep -A 17 '^_Y.*square_:' target/x86_64-unknown-linux-gnu/debug/build/hello.s
_Y957555df__square_:
	.cfi_startproc
	push	rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	mov	rbp, rsp
	.cfi_def_cfa_register 6
	.byte	0x48, 0x81, 0xec, 0x30, 0x00, 0x00, 0x00	# sub rsp, 0x30
	mov	dword ptr [rsp], edi
.LBB0_0:
	movsxd	rcx, dword ptr [rsp]
	mov.s	rax, rcx
	imul	eax, ecx
	{disp32} jo	.LBB0_1
	movsxd	rax, eax
	mov	rsp, rbp
	pop	rbp
	ret
$ clang --target=x86_64-linux-gnu -c target/x86_64-unknown-linux-gnu/debug/build/hello.s
```

`listing` 按 `objdump -d` 的方式列出每个函数的机器码（在 x86_64 和 x86 上使用 Intel 语法）：偏移、字节和指令，带有代码生成器的标签（`.L0:`），在指令下方给出填充它的重定位。字节与 `obj` 中的完全相同。它适用于每个目标，交叉编译时也可以，因为输出时不需要链接：

```text
$ nabi emit --emit=listing --target x86_64-unknown-linux-gnu
   Compiling hello (.)
       Wrote ./target/x86_64-unknown-linux-gnu/debug/build/hello.lst
    Finished in 0.85s
$ grep -A 12 'square_>:' target/x86_64-unknown-linux-gnu/debug/build/hello.lst
<_Y957555df__square_>:
       0: 55                           	push	rbp
       1: 48 89 e5                     	mov	rbp, rsp
       4: 48 81 ec 30 00 00 00         	sub	rsp, 0x30
       b: 89 3c 24                     	mov	dword ptr [rsp], edi
.L0:
       e: 48 63 0c 24                  	movsxd	rcx, dword ptr [rsp]
      12: 48 8b c1                     	mov	rax, rcx
      15: 0f af c1                     	imul	eax, ecx
      18: 0f 80 08 00 00 00            	jo	0x26 <.L1>
      1e: 48 63 c0                     	movsxd	rax, eax
      21: 48 89 ec                     	mov	rsp, rbp
      24: 5d                           	pop	rbp
```

在目标文件中，因而在调试器或性能分析器中，函数带有以 `_Y` 开头的修饰后的符号名。`nabi demangle` 把一个符号还原为它的名字（`-v` 显示它编码的各个部分）；不给符号时，它会过滤文本，让性能分析器或 `nm` 的输出以 Nabi 名字显示。

```text
$ nabi demangle _Y957555df__square_
square
$ perf report --stdio | nabi demangle
$ nm target/debug/hello | nabi demangle
```

用 `-g` 构建会包含 DWARF 调试信息，所以可以在 `lldb` / `gdb` 中逐行单步执行源代码（Windows 构建会在 PDB 中得到同样内容的 CodeView，Visual Studio 调试器和 WinDbg 可以读取：源代码行、参数、局部变量及其类型）。`-g` 不会改变优化构建（`-O1` 及以上）的代码：调试器把内联的函数显示为独立的帧，把优化后的代码不再保留其值的变量显示为 optimized out。

工具链附带 lldb、gdb 和 natvis（Visual Studio、WinDbg）的格式化器，它们把 `str` 显示为其文本，把 `List`、`Dict`、`Set`、`SortedDict`、`Slice` 或 `Array` 显示为大小和元素，并且（lldb 中）把和类型显示为当前的变体。`nabic --print lldbinit`、`--print gdbinit` 和 `--print natvis` 打印加载它们的内容：

```text
$ lldb -O "$(nabic --print lldbinit)" ./target/debug/hello
(lldb) frame variable xs m
(core::collections::list::List<i64>) xs = size=3 {
  [0] = 1
  [1] = 2
  [2] = 3
}
(core::collections::dict::Dict<str, i64, RandomState>) m = size=1 {
  ["a"] = 1
}
$ gdb -iex "$(nabic --print gdbinit)" ./target/debug/hello
(gdb) print xs
$1 = size=3 = {1, 2, 3}
$ nabic --print lldbinit >> ~/.lldbinit      # or load them every time
$ nabic --print gdbinit >> ~/.gdbinit
```

在 cdb 或 WinDbg 中，用 `.nvload <path>` 加载 natvis 文件（路径为 `nabic --print natvis` 打印出的路径）；在 Visual Studio 中，把它复制到 `Documents\Visual Studio 2022\Visualizers`。

在 WinDbg 或 cdb 中，被优化器内联的函数没有自己的地址：`bm hello!sq` 在它被内联的每个位置设置断点，`k` 把它列为内联帧。用 `-g` 构建的 wasm32 模块可以通过 wasmtime 调试，wasmtime 会把它的调试信息转成供 gdb 或 lldb 使用的本机调试信息；wasmtime 自身的优化可能丢失一些值，用 `-O opt-level=0` 可以保留。

```text
$ gdb --args wasmtime run -D debug-info -O opt-level=0 hello.wasm
```

## 获取帮助

`nabi help` 按组列出命令，`nabi help <command>`（或 `nabi <command> --help`）显示某个命令的用法、示例和选项。

```text
$ nabi help fmt
usage: nabi fmt [path] [options]

Format .nabi and .nabion files in place: <path> is one file, or a directory
whose files are all formatted (default: the package the current directory
is in).

examples:
  nabi fmt                       format everything here
  nabi fmt --check               exit 1 if a file would change (CI)
  nabi fmt --diff src/main.nabi  show the change, write nothing

options:
  --check  exit 1 if a file would change
  --diff   show the changes, write nothing

global options (-q, -v, --color, ...): `nabi help`
```

拼错的命令或选项会得到最接近的名字：

```text
$ nabi tset
nabi: unknown command `tset`; did you mean `test`?
run `nabi help` for usage
```

`-q`、`-v` 和 `--color` 等全局选项写在命令之前或之后都可以。

## 小结

- `nabi test` / `nabi bench` 单独构建并运行 `#[test]` / `#[bench]` 函数。每个测试有自己的进程，并有时间限制。`#[should_panic]` 和 `#[ignore]` 改变测试检查的内容和运行的时机。在库中，`nabi test` 还会运行文档和 README 中的 ```` ```nabi ```` 示例。带上 `--target` 时，它们通过 `wasmtime`、`node`、`qemu-<cpu>` 或 `NABI_RUNNER` 运行。
- `nabi fmt` 只修正缩进和空行。在 CI 中用 `nabi fmt --check`。在包外要给它一个路径。
- `nabi lint` 运行 lint，`nabi fix` 应用 lint 的修复；`nabi check` 一直执行到代码生成。lint 级别按 `nabi.nabion` < `-A/-W/-D` < 源代码属性的顺序决定。
- `nabi doc` 根据 `///` 和 `//!` 文档注释在 `target/doc/` 中生成 HTML。
- 在编辑器中把 `nabils` 注册为 LSP 服务器。
- REPL 的提示符是 `>>>`，把最后的值保存在 `it` 中，并有 `:type`、`:doc`、`:vars` 和 `:reset` 等命令。`nabi repl .` 加载当前包；笔记本则用 `nabi jupyter install`。
- 深入排查问题：`-v`、`--emit`、`nabi demangle`、`-g`（以及 `nabic --print lldbinit` / `gdbinit` / `natvis` 加载的格式化器）。
- `nabi help <command>` 显示某个命令的用法和选项。
- `NABIR_GC_LOG=1` / `NABIR_GC_STATS=1` 显示回收器的停顿及其到达安全点的时间；`bare::gc::stats()` 在代码中提供这些计数器。

## 延伸阅读

- 规范：[nabic CLI](../spec/refs-lang/009-tools/901-nabic-cli.html)、[标准格式](../spec/refs-lang/009-tools/910-nabi-frontend.html#standard-format)、[lint 控制](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#629-allow--deny--warn-lint-control)、[Nabi 前端](../spec/refs-lang/009-tools/910-nabi-frontend.html)、[JIT 单元与 REPL](../spec/refs-lang/009-tools/907-jit-cells.html)、[REPL 行编辑](../spec/refs-lang/009-tools/907-jit-cells.html#line-editing)、[REPL 命令](../spec/refs-lang/009-tools/909-debug-tools.html#999-repl--jupyter-debugging)、[Jupyter](../spec/refs-lang/009-tools/908-jupyter-integration.html)、[调试工具](../spec/refs-lang/009-tools/909-debug-tools.html)、[诊断](../spec/refs-lang/009-tools/909-debug-tools.html#994-diagnostics--error-messages)、[符号修饰](../spec/refs-lang/009-tools/905-symbol-mangling.html)
- [错误码](../errors/index.html)
- 在运行时加载代码：[23. 插件](23-plugins.html)
- 在程序运行时应用修改：[26. 热重载](26-hot-reload.html)
- API：[core 中的 assert 函数](../../std/core/panic.html)
- 下一章：[17. 条件编译](17-conditional-compilation.html)
