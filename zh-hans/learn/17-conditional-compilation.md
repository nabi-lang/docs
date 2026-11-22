# 17. 条件编译

同一棵源代码树常常要构建成几种形态：按操作系统区分、调试版与发布版、带或不带某个可选特性。Nabi 在编译期用 `#[cfg(..)]` 决定这些。条件为假的代码在解析之后立即被移除，就像从未写过一样，所以它在运行时没有任何开销，甚至可以引用在该次构建中不存在的名字。

## 条件

条件是一个标志、一个 `key = "value"` 对，或者用 `all(..)`、`any(..)` 和 `not(..)` 组合起来的它们。

| 条件                       | 为真的情况                                              |
|---------------------------------|--------------------------------------------------------|
| `target_os = "linux"`           | 为 Linux 构建（`"macos"`、`"windows"`、`"wasi"`，浏览器为 `"unknown"` 等） |
| `target_arch = "aarch64"`       | 为该 CPU 构建（`"x86_64"` 等）                 |
| `target_family = "unix"`        | 类 Unix 目标（`"windows"`、`"wasm"`）              |
| `target_pointer_width = "64"`   | 64 位目标                                         |
| `feature = "tls"`               | 启用了包的特性 `tls`（[第 18 章](18-features-and-publishing.html)） |
| `debug_assertions`              | 以 `-O0` 构建（`nabi run` 的默认值）          |
| `test`                          | 为 `nabi test` 构建                                |

`target_*` 跟随 `--target`，所以交叉构建看到的是目标的值，而不是主机的值。

## `#[cfg]` 放在哪里

`#[cfg]` 可以附加在项上（函数、类型、`mod`、`use`、`const`、`static`、`impl` 块及其成员），也可以附加在代码内部更小的部分上：语句、`match` 分支、结构体字段、结构体字面量中的字段，以及函数参数。

```nabi
enum Op { Add, Trace, Quit }

struct Build {
    name: str;
    #[cfg(debug_assertions)]
    checks: bool;                         // a field only debug builds have
}

fn describe(op: Op) -> str {
    match op {
        Op::Add => "add",
        #[cfg(debug_assertions)]
        Op::Trace => "trace (debug build)", // an arm
        _ => "unsupported",
    }
}

let b = Build {
    name: "demo",
    #[cfg(debug_assertions)]
    checks: true,                         // a struct literal field
};

#[cfg(debug_assertions)]
println(f"debug build of {b.name}");      // a statement
#[cfg(not(debug_assertions))]
println(f"release build of {b.name}");

println(describe(Op::Trace));
```

```output
debug build of demo
trace (debug build)
```

用 `-O2` 构建时，同一个文件打印 `release build of demo` 和 `unsupported`：`checks` 字段、它的初始化以及 `Trace` 分支都消失了。

条件为假的参数会从签名中移除，调用时只传入剩下的实参。这样可选特性就能增加一个参数，而无需再写一份函数。

```nabi
fn connect(host: str, #[cfg(feature = "tls")] verify: bool) -> str {
    #[cfg(feature = "tls")]
    if verify { return f"{host} (verified)"; }
    host
}

// Without the `tls` feature, `connect` takes one argument.
println(connect("example.com"));
```

```output
example.com
```

闭包参数不能带属性。

## 两个定义，一次构建

要让函数在不同平台上有不同的函数体，就用互相排斥的条件把它写两遍。任何一次构建中都只会保留其中一个；如果两个都保留下来，就是普通的重复定义错误。

```nabi
#[cfg(target_family = "windows")]
fn path_sep() -> str { "\\" }

#[cfg(not(target_family = "windows"))]
fn path_sep() -> str { "/" }

println(["usr", "local", "bin"].join(path_sep()));
```

在 Linux 和 macOS 上，它打印 `usr/local/bin`。

整个模块也可以是有条件的。`#[cfg(target_os = "linux")] mod linux;` 在其他构建中甚至不会读取 `linux.nabi`。

编译器会记住条件移除了什么。调用只存在于另一种构建中的东西是一个错误，错误信息会说明它在哪里存在，以及如何为这次调用加上条件：

```text
error[E0046]: no method `pidfd` found for type `Child`
   = note: `pidfd` is only available on linux (`#[cfg(target_os = "linux")]`)
   = help: put the code that uses it under the same `#[cfg(target_os = "linux")]`
```

## std 中特定于操作系统的部分

标准库也以同样的方式划分。只有部分系统才有的内容位于 `std::os::unix`、`os::linux` 和 `os::windows` 中，每个模块只存在于对应的目标上。它们的扩展 trait 在那些目标上位于 prelude 中，所以在 Linux 和 macOS 上，`Command::new("sh").process_group(0)` 不需要 `use`。在 Windows 上，同样的调用是一个错误，错误信息会说明该方法在哪里存在：

```text
error[E0046]: no method `process_group` found for type `Command`
   = note: `process_group` is only available on unix (`#[cfg(target_family = "unix")]`)
   = help: put the code that uses it under the same `#[cfg(target_family = "unix")]`
```

所以用到它的代码要放在同样的条件下：

```nabi
use std::process::Command;

#[cfg(target_family = "unix")]
fn detached(cmd: Command) -> Command { cmd.process_group(0) }

#[cfg(not(target_family = "unix"))]
fn detached(cmd: Command) -> Command { cmd }
```

供 C 使用的原始操作系统句柄也在那里。`as_raw_fd()`（Windows 上是 `as_raw_handle()` / `as_raw_socket()`）借用 `File`、套接字或子进程管道的描述符：值仍然拥有它，并负责关闭它。`into_raw_fd()`（`into_raw_handle()` / `into_raw_socket()`）交出描述符：值被消耗，不再关闭它，所以关闭它的责任就在你了。`unsafe` 的 `from_raw_fd(fd)` 重新创建一个拥有 `fd` 的值。

## `cfg!`：编译期的 `bool`

当两个分支在所有平台上都能编译、只是选择不同时，`cfg!(..)` 更简短。它是一个常量 `bool`，所以编译器会丢弃无用的分支；与 `#[cfg]` 不同，两边仍然都必须通过类型检查。

```nabi
const DEBUG: bool = cfg!(debug_assertions);

let level = if cfg!(feature = "verbose") { 2 } else { 1 };
println(f"debug={DEBUG} level={level}");
```

```output
debug=true level=1
```

`#[cfg_attr(cond, attr)]` 仅在条件成立时附加另一个属性，例如 `#[cfg_attr(test, derive(Debug))]`。

## 小结

- `#[cfg(..)]` 在编译期保留或移除代码。条件有 `key = "value"`、标志（`test`、`debug_assertions`）以及 `all` / `any` / `not`。
- 它可以附加在项、字段、变体、语句、match 分支、结构体字面量字段和参数上。被移除的参数会从签名中消失。
- 为不同平台的函数体编写互相排斥的定义。
- `cfg!(..)` 是一个常量 `bool`，用于普通的 `if`。
- 特定于操作系统的 std 位于 `std::os::{unix, linux, windows}` 中；其扩展 trait 在对应目标上位于 prelude 中。
- `feature = ".."` 条件由包的特性设定，这是下一章的主题。

## 延伸阅读

- 规范：[6.3 `#[cfg]`](../spec/refs-lang/006-meta-system/603-cfg.html)、[6.3.3 附加位置](../spec/refs-lang/006-meta-system/603-cfg.html#633-attachment-positions)、[6.3.5 `cfg!`](../spec/refs-lang/006-meta-system/603-cfg.html#635-cfg-compile-time-bool-value)、[平台目标](../spec/refs-lang/000-meta/006-platform-targets.html)
- 下一章：[18. 特性与发布](18-features-and-publishing.html)
