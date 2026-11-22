# 1. 入门

本章安装 nabi 工具链，创建并运行你的第一个包，再试试单文件脚本和 REPL。读完本章，你就能自己运行本指南后面的示例。

## 安装

### 前提条件

`nabic` 是自包含的：它有自己的代码生成器和链接器，所以不需要安装 C 编译器或链接器（[9.2.18](../spec/refs-lang/009-tools/902-aot-pipeline.html#9218-self-contained-toolchain)）。

- macOS：Xcode Command Line Tools（`xcode-select --install`），提供最终链接所需的 SDK。
- Linux：不需要任何东西。工具链及其构建的程序需要 glibc 2.34 或更高版本（Ubuntu 22.04、Debian 12、RHEL 9 及更新版本）。
- Windows：不需要任何东西。
- FreeBSD：不需要任何东西。工具链及其构建的程序需要 FreeBSD 14 或更高版本。

发布包为以下平台构建：macOS arm64，Linux x86_64 和 arm64（glibc 与静态 musl 两种），Windows x86_64 和 arm64，以及 FreeBSD x86_64。其他主机没有发布包，需要从源代码构建 Nabi。

### 安装脚本

安装脚本从 [GitHub releases](https://github.com/nabi-lang/nabi/releases) 下载适合你平台的工具链，用 SHA-256 校验后解压到 `~/.nabi`。

```bash
curl -fsSL https://nabi.sh | sh
```

在 Windows 上，在 PowerShell 中运行下面的命令。它安装到 `%USERPROFILE%\.nabi`，并把其中的 `bin` 加入用户的 `PATH`。

```powershell
powershell -ExecutionPolicy Bypass -c "irm https://nabi.sh/install.ps1 | iex"
```

`install.sh --version <v>` 和 `install.ps1 -Version <v>` 按确切版本（`0.1.0`）选择一个版本；不加时安装最新版本。

可以用环境变量 `NABI_HOME` 改变安装位置（默认为 `~/.nabi`）。脚本运行完后，目录结构如下。

```text
~/.nabi/
├── bin/                        # nabi, nabic, nabicc, nabils proxy
└── toolchains/
    └── 0.1.0-aarch64-apple-darwin/
        ├── bin/{nabi, nabic, nabicc, nabils}
        └── lib/                # runtime and standard library sources
```

刚安装的工具链成为默认工具链。`~/.nabi/bin` 中的四个可执行文件是**代理**：实际工作由当前工具链中同名的可执行文件完成。这样就可以并排安装多个工具链并从中选择一个（与 rustup 的做法相同）。代理始终复制自已安装的最新工具链，它了解所有旧工具链的布局，因此把旧工具链设为默认时，代理保持不变。

脚本还会把 `~/.nabi/bin` 加入 `PATH`。它写入 `~/.nabi/env`（以及 `~/.nabi/env.fish`），在 `~/.zshenv`、`~/.bashrc` 或 `~/.profile` 等 shell 启动文件中加载它，并为 fish 添加 `~/.config/fish/conf.d/nabi.fish`。它会列出修改过的每个文件：

```text
 Downloading nabi 0.1.0 (aarch64-apple-darwin)
   Installed nabi 0.1.0 to ~/.nabi

    Modified ~/.zshenv (PATH)

A new shell finds nabi on PATH; to use it in this one, run:
  . "$HOME/.nabi/env"            (sh, bash, zsh)
  source "$HOME/.nabi/env.fish"  (fish)

Next steps:
  nabi --version
  nabi new hello && cd hello && nabi run
```

再次运行脚本不会做任何改动。如果不想让脚本修改 shell 文件，传入 `--no-modify-path`（`install.ps1` 用 `-NoModifyPath`）或设置 `NABI_NO_MODIFY_PATH=1`；脚本会打印需要你自己加入 `PATH` 的那一行。工具链自带官方包（`json`、`http`、`regex`、`table` 等），所以使用它们的代码不联网也能构建；不需要它们时，传入 `--no-packages`（`-NoPackages`，或 `NABI_NO_PACKAGES=1`）。`nabi` 的 shell 补全也会一并安装（bash、zsh、fish、PowerShell）。想装到别处时，`nabi completions <shell>` 会打印补全脚本。

在新的 shell 中查看版本。

```bash
nabi --version
nabi toolchain list
```

`nabi --version -v` 还会列出实际运行的二进制文件、它选中的工具链及原因、它的 `nabic` 和主机，每项一行 `key: value`（`nabi version -v --format json` 把它们打印成一个 JSON 对象）。版本与预期不符时可以用它排查。

```text
$ nabi --version -v
nabi 0.1.0 (<commit> <date>)
binary:    /Users/me/.nabi/toolchains/0.1.0-aarch64-apple-darwin/bin/nabi
toolchain: 0.1.0-aarch64-apple-darwin (the default)
nabic:     /Users/me/.nabi/toolchains/0.1.0-aarch64-apple-darwin/bin/nabic
host:      aarch64-apple-darwin
```

共安装了四个可执行文件。

| 可执行文件 | 作用 |
|------------|--------------|
| `nabic`     | 编译器。构建、检查、测试和基准测试工具（[9.1](../spec/refs-lang/009-tools/901-nabic-cli.html)） |
| `nabi`      | 包与构建前端及各种工具。`nabi.nabion`、依赖、环境、运行、lint、格式化、文档、REPL、Jupyter（[9.10](../spec/refs-lang/009-tools/910-nabi-frontend.html)） |
| `nabicc`    | C 编译器，也可以作为 `nabi cc` 运行（[9.13](../spec/refs-lang/009-tools/913-c-compiler.html)，[第 14 章](14-ffi.html#c-源代码nabi-cc)） |
| `nabils`    | 语言服务器（LSP） |

### 管理工具链

`nabi toolchain` 管理已安装的工具链。当前工具链按以下顺序选定：命令行上的 `+version`（`nabi +0.1 build`）、环境变量 `NABI_TOOLCHAIN`、最近的 `nabi.nabion`（或脚本的 `//! nabion` 块）中的 `toolchain` 字段，最后是默认工具链。固定的版本可以是一个系列（`"0.1"`，即已安装的最新 0.1.x 版本；尚未安装任何 0.1 版本时，则是其最新的候选版本）、一个版本（`"0.1.0"`）或一个版本要求（`"^0.1.0"`）。一条命令从头到尾只在一个工具链上运行：即使按完整路径运行某个工具链自己的 `bin/nabi`，它也会把命令交给项目固定的工具链，而每个工具链的 `nabi` 只驱动它自己的 `nabic`。

```bash
nabi toolchain list                     # installed toolchains
nabi toolchain default 0.1.1            # change the default
nabi toolchain install 0.1.1            # download 0.1.1 beside the default
nabi toolchain update                   # download the newest release and make it the default
nabi toolchain update 0.1.1             # exactly 0.1.1 instead
```

`nabi toolchain update` 接受确切版本，而不是系列。`nabi toolchain install` 下载一个工具链但不改变默认值（`--from <archive>` 安装你已有的文件）。下载内容会用公布的 SHA-256 校验，任何一步失败，原来的默认工具链都保持不变。旧工具链会保留。更新跨入新系列时会附上一条提示：只有最新系列会得到修复，所以请把项目迁移过去。项目迁移之前，可以固定旧系列，例如 `toolchain: "0.1"` 固定到 0.1，或者运行 `nabi +0.1 ..`。不要把它和 `nabi update` 混淆，后者更新包的 `nabi.lock` 中固定的版本（[第 11 章](11-modules-packages.html)）。

同一系列中较新的工具链能构建较旧工具链能构建的代码。1.0 之前，系列是 `0.y`：0.1.7 能构建 0.1.3 构建过的代码，0.2.0 则不一定。从 1.0 起系列是 `x`：1.9.0 能构建 1.4.2 构建过的代码，2.0.0 则不一定。包的 `nabi_version` 遵循这条规则：`"0.1.3"` 表示 nabi 0.1.3 或更高的 0.1.x。

如果 `PATH` 中另一个 `nabi`（比如来自某个包管理器）排在 `~/.nabi/bin` 之前，新的默认工具链就不会生效，因此 `nabi toolchain update` 和 `nabi toolchain default` 会发出警告：

```text
nabi: warning: your shell runs /opt/homebrew/bin/nabi (nabi 0.1.3) before ~/.nabi/bin/nabi; put ~/.nabi/bin first on PATH or remove the other one
```

### 从源代码构建（可选）

你也可以自己构建工具链压缩包，再用同一个脚本安装。这需要 Rust 工具链（`rustup`）。

```bash
git clone https://github.com/nabi-lang/nabi && cd nabi
scripts/dist/native.sh                     # dist/nabi-<version>-<triple>.tar.gz
sh install.sh --archive dist/nabi-*.tar.gz  # into ~/.nabi
```

脚本还会用 `targets/build.sh` 构建每个目标的运行时和 C 库，这需要 `curl`、`python3` 和支持 xz 的 `tar`，下载的内容放在 `~/.cache/nabi`（可用 `NABI_CACHE` 改变位置）。在 Linux 上，`scripts/dist/gnu.sh`（podman 或 docker）在容器中构建同样的压缩包，使其能在 glibc 2.34 及更高版本上运行；发布包就是这样构建的。详情见仓库的 README。

如果你在开发编译器，可以不安装、直接使用构建输出。`cargo build` 生成可执行文件和主机的运行时。`targets/build.sh <triple>` 构建某个目标的 C 库和运行时文件：Linux 主机需要它自己的这些文件才能链接程序，每个交叉编译目标也都需要；在 macOS 和 Windows 上，主机不需要额外的东西。

```bash
cargo build --release
targets/build.sh "$(rustc -vV | sed -n 's/^host: //p')"
export PATH="$PWD/target/release:$PATH"
```

## 你的第一个包

`nabi new` 创建一个包目录，并初始化一个 git 仓库，其中的 `.gitignore` 排除 `target/`。包名由小写字母、数字、`-` 和 `_` 组成，以字母开头，最长 64 个字符。Windows 保留给设备的名字（`con`、`nul`、`com1` 等）会被拒绝，因为在那里没有文件能叫这个名字。`nabi` 这个名字被标准库占用，所以包不能叫 `nabi`。在代码中，`-` 变成 `_`（`nabi new my-lib` 以 `use my_lib::..` 导入）。

```bash
nabi new hello
cd hello
```

```text
     Created executable package (hello)
             cd hello && nabi run
```

第二行是接下来要运行的命令。包里有两个文件。清单 `nabi.nabion` 记录包名、版本、能构建它的最旧 nabi 版本（`nabi_version`）、创建它时所用的工具链系列，以及依赖。

```nabion
Package { name: "hello", version: "0.1.0", nabi_version: "0.1.0", toolchain: "0.1", deps: {} }
```

源代码放在 `src/` 中；`src/main.nabi` 只有一行，打印一句问候。

```nabi
println("Hello, world!");
```

```output
Hello, world!
```

`nabi run` 构建并运行它。输出放在 `target/debug/`。构建只编译程序用到的那部分标准库，工具链会保留这些结果，所以之后构建任何包时都会从缓存中取用。

```bash
nabi run
```

```text
   Compiling hello (.)
    Finished in 0.87s
     Running `target/debug/hello`
Hello, world!
```

`src/main.nabi` 中的顶层语句从上到下执行，构成整个程序。函数和类型声明可以放在顶层语句之间的任何位置。其他常用命令有 `nabi build`（只构建）、`nabi check`（只做类型检查）、`nabi test` 和 `nabi fmt`。

## 稍大一点的程序

把 `src/main.nabi` 改成下面的内容，再次运行 `nabi run`。

```nabi
let names = ["ada", "grace", "linus"];
for name in names {
    greet(name);
}
println(f"{names.len()} people greeted");

fn greet(name: str) {
    println(f"Hello, {name}!");
}
```

```output
Hello, ada!
Hello, grace!
Hello, linus!
3 people greeted
```

`f"..."` 是 f-string：`{}` 中的每个表达式都会被求值并插入原位。`greet` 在使用处之后才声明，调用却能正常工作，因为函数等声明不论顺序都可见。

`println` 还能接受多个值，并用空格分隔打印；`sep:` 和 `end:` 改变值之间和末尾输出的内容；调试时，`dbg(e)` 把表达式连同其位置和值打印到 stderr：

```nabi
let n = 3;
println("total:", n, [1, 2]);
println(1, 2, 3, sep: ", ");
println(f"{n=}");
```

```output
total: 3 [1, 2]
1, 2, 3
n=3
```

除了顶层语句，也可以用 `fn main()` 作为入口点。两者只能选其一。

```nabi
fn main() {
    let total = add(2, 3);
    println(f"2 + 3 = {total}");
}

fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

```output
2 + 3 = 5
```

## 单文件脚本

快速试验不需要包。把文件路径交给 `nabi run`，它就单独运行这个文件。`nabi build hi.nabi` 把可执行文件 `hi` 写到脚本旁边。

```bash
echo 'println("hi");' > hi.nabi
nabi run hi.nabi
```

```text
   Compiling hi (.)
    Finished in 0.05s
     Running hi.nabi
hi
```

脚本也可以使用依赖：`nabi add json hi.nabi` 把依赖写进文件开头的 `//!` 注释中（见[内联清单](../spec/refs-lang/009-tools/910-nabi-frontend.html#9107-inline-manifest-for-single-file-scripts)和[第 11 章](11-modules-packages.html#单文件脚本与环境)）。

## REPL

不带参数运行 `nabi` 会打开 REPL。输入一个表达式，它就打印其值；以 `?` 结尾则改为打印 `Debug` 形式。没有 `Display` 的值只打印其类型，`let` 只定义名字。

```text
$ nabi
Nabi 0.1.0 (<commit> <date>) on aarch64-apple-darwin
Type :help for help, :exit to quit.
>>> let xs = [1, 2, 3]
>>> xs.len() * 2
6
>>> xs?
[1, 2, 3]
>>> :type xs
List<i64>
>>> :exit
```

`:type expr` 报告表达式的类型而不运行它。阅读本指南时，凡是对某个类型有疑问，都可以在 REPL 中查看。单元格中最后一个 `let` 不需要 `;`。其余内容见[第 16 章](16-tools.html#repl-与-jupyter)（[JIT 单元格](../spec/refs-lang/009-tools/907-jit-cells.html)）。

## 小结

- 用 `curl -fsSL https://nabi.sh | sh`（Windows 上用 `install.ps1`）安装到 `~/.nabi`。脚本把 `~/.nabi/bin` 加入 `PATH`，并安装 shell 补全。
- `nabi new name` 创建 `nabi.nabion` 和 `src/main.nabi`；`nabi run` 构建并运行。
- `src/main.nabi` 的顶层语句就是程序。也可以使用 `fn main()`。
- `nabi run file.nabi` 运行脚本；单独运行 `nabi` 启动 REPL。

## 延伸阅读

- [安装布局](../spec/refs-lang/009-tools/902-aot-pipeline.html#9219-installation-layout)，[固定工具链](../spec/refs-lang/009-tools/910-nabi-frontend.html#9106-toolchain)
- [`nabi` 前端](../spec/refs-lang/009-tools/910-nabi-frontend.html)，[`nabic` CLI](../spec/refs-lang/009-tools/901-nabic-cli.html)
- [入口点](../spec/refs-lang/004-functions/416-entry-point.html)
- [JIT 单元格与 REPL](../spec/refs-lang/009-tools/907-jit-cells.html)
- 下一章：[2. 值与类型](02-values-and-types.html)
