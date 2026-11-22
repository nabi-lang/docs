# 11. 模块与包

程序变大以后，你需要拆分代码，并使用别人写的库。在 Nabi 中，划分代码的单位是**模块**，构建和分发的单位是**包**。

## `mod` 与路径

`mod name { .. }` 就地定义一个模块。模块内的项用 `module::item` 路径来引用，模块中还可以再包含模块。

```nabi
mod geom {
    pub struct Point { pub x: f64; pub y: f64; }

    pub fn origin() -> Point { Point { x: 0.0, y: 0.0 } }

    pub mod dist {
        use sup::Point;

        pub fn manhattan(a: Point, b: Point) -> f64 {
            abs(a.x - b.x) + abs(a.y - b.y)
        }

        fn abs(v: f64) -> f64 { if v < 0.0 { -v } else { v } }
    }
}

let p = geom::Point { x: 3.0, y: -4.0 };
println(f"{geom::dist::manhattan(geom::origin(), p)}");
```

```output
7
```

只有特定的几个词可以作为路径的开头。

| 开头     | 含义                                                         |
|----------|--------------------------------------------------------------|
| `pkg::`  | 从当前包的根开始                                             |
| `sup::`  | 从父模块开始（再往上用 `sup::sup::`）                        |
| `std::`、`core::`、`bare::` | 标准库的各个层级 |
| `::std::` | 即使存在本地的 `std`，也指标准库 |
| 一个名称 | 当前模块的项或子模块，或用 `use` 导入的名称 |

名称查找不会自动搜索父模块。所以在 `dist` 内部使用 `Point` 时，需要用 `use sup::Point;` 显式导入。得益于此，你总能在文件内部看出一个名称来自哪里。

`std`、`core` 和 `bare` 都是普通的名称。你自己的模块或依赖也可以叫 `std`；在拥有它的模块中，`std::` 指你的那个，而 `::std::` 仍然指标准库：

```nabi
mod std { pub fn hello() -> str { "mine" } }

fn main() {
    println(std::hello());                       // mine
    println(::std::path::Path::empty().is_relative());  // 标准库的
}
```

## 用 `use` 导入

不必每次都写长路径，用 `use` 导入名称即可。可以用 `{..}` 一次导入多个，用 `as` 重命名，用 `*` 导入一个模块的所有公开项。

```nabi
use report::show;
use shapes::*;

mod shapes {
    pub fn square(x: i32) -> i32 { x * x }
    pub fn cube(x: i32) -> i32 { x * x * x }
    pub const UNIT: i32 = 1;
}

mod report {
    use pkg::shapes::{square, cube as pow3};

    pub fn show(x: i32) {
        println(f"{x}: {square(x)} {pow3(x)}");
    }
}

show(3);
println(f"{cube(UNIT + 1)}");
```

```output
3: 9 27
8
```

按名称导入的名称不能在同一模块中再定义一次；否则定义会悄无声息地遮蔽导入。通配导入（`use shapes::*;`）则让位于模块自身的项。

```nabi norun
use shapes::square;
fn square(x: f64) -> f64 { x * x }
```

```text check=checks/11-import-clash.nabi
error[E0032]: `square` is both imported from `shapes::square` and defined in this module
  --> main.nabi:1:5
   |
 1 | use shapes::square;
   |     ^^^^^^ imported here
 2 | fn square(x: f64) -> f64 { x * x }
   | -- defined here, which would hide the import
   |
   = help: import it under another name, `use shapes::square as shapes_square;`, or remove this `use` if the module does not need it
```

到目前为止不写 `use` 就能使用的 `List`、`Dict`、`println` 等名称来自 *prelude*：每个文件都以一句隐式的 `use core::prelude::*;` 开头。prelude 是一份固定的短列表：常用类型（`List`、`Dict`、`Result`、`Duration`、`Mutex`……），写在 `impl` 头部和约束中的 trait（`Clone`、`Hash`、`Display`、`Iterator`、`From`……），打印和 panic 函数，以及 `core` 的顶层模块（`task`、`par`、`ops`、`num`、`sync`、`iter`……）。`core` 中的其他内容要通过这些模块之一访问，或者导入：

```nabi norun
task::sleep(Duration::ms(1));          // the module `task` is in the prelude
impl ops::Add for V { .. }             // operator traits live in `ops`
use core::sync::Frozen;           // or import the name
```

你自己的名称总是优先于 prelude 中的名称。枚举变体不在 prelude 中；在期望类型能确定枚举的地方，直接写变体名即可（[第 6 章](06-structs-enums-traits.html)）。名称不在作用域内时，错误会给出应写的路径和导入它的 `use`。`std` 中的文件、网络、线程等模块需要显式导入，如 `use std::fs;`。

trait 也是如此：只有在 trait 处于作用域内时才能找到它的方法，所以来自其他模块或包的 trait 需要先 `use`（`use zoo::Animal;`），`x.name()` 才能解析。标准库（`core`、`std`）中 trait 的方法在任何地方都可用，不需要 `use`。

宏、derive 和元属性属于定义它们的包。你自己的包和标准库中的直接写名称（`log!(..)`、`#[derive(Inspect)]`）；其他包的要带上包名，`json::log!(..)`、`#[derive(json::Json)]`，或者用 `use json::log!;` 导入一次。直接写其他包中宏的名称是错误，错误的帮助信息会给出这两种写法。

## 可见性

默认情况下，一个项只在它自己的模块内可见。要让它在外部可见，加上可见性关键字。

| 关键字     | 可见范围                                |
|------------|-----------------------------------------|
| （无）     | 同一模块                                |
| `pub(sub)` | 同一模块及其子模块                      |
| `pub(pkg)` | 同一包内的任何地方                      |
| `pub`      | 其他包也可见                            |

结构体的字段也必须逐个设为公开。所以上面的 `Point` 给每个字段都加了 `pub`，如 `pub x: f64`。从外部使用私有项时，编译器会告诉你在哪里加 `pub`。

```nabi
mod bank {
    pub struct Account { pub owner: str; balance: i64; }

    pub fn open(owner: str) -> Account { Account { owner: owner, balance: 0 } }

    pub fn deposit(a: Account, amount: i64) -> Account {
        Account { owner: a.owner, balance: a.balance + check(amount) }
    }

    pub fn balance(a: Account) -> i64 { a.balance }

    fn check(amount: i64) -> i64 { if amount < 0 { 0 } else { amount } }
}

let a = bank::deposit(bank::open("kim"), 500);
println(f"{a.owner}: {bank::balance(a)}");
```

```output
kim: 500
```

`balance` 字段和 `check` 函数在 `bank` 外部不可见，所以余额只能通过 `bank` 的函数改变。出于同样的原因，`bank` 外部的代码不能写 `bank::Account { owner: "kim" }`：结构体字面量只有在私有字段有默认值时才能省略它。所以由 `bank::open` 来构造账户。

标准库中有少数 `pub` 项标有 `#[internal]`：工具链自己的包共享它们，其他任何包使用它们都是错误，即 “internal to the Nabi toolchain”。API 文档和编辑器不会列出它们。

## 拆分到文件

在真实的包中，模块会拆分到各个文件。像 `mod shapes;` 这样声明一个没有主体的模块，同一目录下的 `shapes.nabi` 就成为该模块的主体。如果它有子模块，在 `shapes.nabi` 旁边放一个同名的 `shapes/` 目录（没有 `mod.nabi` 这样的特殊文件名）。

```text
geom/
├── nabi.nabion
└── src/
    ├── lib.nabi          // mod shapes;
    ├── shapes.nabi       // shapes module body, declares mod circle;
    └── shapes/
        └── circle.nabi   // shapes::circle
```

包的源代码放在 `src/` 中。包的根模块对可执行程序是 `src/main.nabi`，对库是 `src/lib.nabi`。可执行的顶层语句只能出现在 `src/main.nabi` 中；其他模块只包含项。

## 包与 `nabi.nabion`

`nabi new` 创建一个包：可执行程序得到 `src/main.nabi`，加 `--lib` 则得到包含一个示例函数和一个 `#[test]` 的 `src/lib.nabi`。清单 `nabi.nabion` 采用 [NABION](../spec/refs-standard-libraries/300-std/308-nabion.html) 格式。除了名称和版本，它还记录 `nabi_version`，即创建它的 nabi，也就是已知能构建它的最旧版本（发布的包的用户需要这一信息，见[第 18 章](18-features-and-publishing.html)），以及 `toolchain: "0.1"`，即创建该包时所用的工具链系列，这样另一个系列的较新工具链就不会意外地构建它（[第 1 章](01-getting-started.html#管理工具链)）。标准库不是依赖：`std`、`core` 和 `bare` 随工具链提供，在 `deps` 中写 `"std": ..` 一行是错误，错误会提示你删除它。

```bash
nabi new --lib geom
nabi new app
```

```text
     Created library package (geom)
             cd geom && nabi test
     Created executable package (app)
             cd app && nabi run
```

我们在 `geom` 中创建两个模块。如果在包的根用 `pub use` 重新导出子模块的项，用户就不需要了解内部结构。

```nabi norun
// geom/src/lib.nabi
mod shapes;

pub use shapes::Circle;

pub(pkg) const PI: f64 = 3.14159;

pub fn area_sum(cs: List<Circle>) -> f64 {
    cs.map(|c| c.area()).sum()
}
```

```nabi norun
// geom/src/shapes.nabi
pub struct Circle { pub r: f64; }

impl Circle {
    pub fn area(self) -> f64 { pkg::PI * self.r * self.r }
}
```

在 `app` 中用 `nabi add` 添加依赖时，它会写入 `nabi.nabion` 的 `deps`。依赖包的名称成为它的顶层路径（`-` 变为 `_`）。

```bash
cd app
nabi add geom --path ../geom
```

```text
       Added geom v0.1.0
```

```nabion
Package {
    name: "app",
    version: "0.1.0",
    nabi_version: "0.1.0",
    toolchain: "0.1",
    deps: { "geom": Dep::path("../geom") },
}
```

```nabi norun
// app/src/main.nabi
use geom::{Circle, area_sum};

println(f"{area_sum([Circle { r: 1.0 }, Circle { r: 2.0 }])}");
```

```text
$ nabi run
   Compiling geom (../geom)
   Compiling app (.)
    Finished in 0.22s
     Running `target/debug/app`
15.70795
```

`PI` 是 `pub(pkg)`，所以在 `geom` 内部的任何地方都能使用，但在 `app` 中不可见。git 仓库可以用 `nabi add parser --git <url> --tag v1.2` 这样的命令添加。对于 git 和注册表依赖，确切的提交固定在 `nabi.lock` 中，所以也要把 `nabi.lock` 提交到版本控制。只有 `nabi update` 会让固定的版本前进。

## 包注册表

默认的包注册表是 [packages.nabi-lang.org](https://packages.nabi-lang.org)。按名称搜索，并连同版本要求一起添加。版本要求的规则与 Cargo 相同：`^1.2` 表示 `>=1.2.0, <2.0.0`。省略要求时，会以 `^` 记录最新版本。

```text
$ nabi search json
    Updating index https://github.com/nabi-lang/packages-index
json 0.2.0 - JSON (RFC 8259): Value tree, strict parser, streaming tokenizer, typed encode/decode, JSON Pointer
$ nabi add json@^0.2
 Downloading json 0.2.0
       Added json v0.2.0
```

```nabion
Package {
    name: "app",
    version: "0.1.0",
    nabi_version: "0.1.0",
    toolchain: "0.1",
    deps: { "json": Dep::version("^0.2") },
}
```

拼错的名称会得到最接近的包：

```text
$ nabi add jsn
    Updating index https://github.com/nabi-lang/packages-index
nabi: error: no package named `jsn` in https://github.com/nabi-lang/packages-index
help: did you mean `json`? (or search: `nabi search jsn`)
```

单独的字符串是 `Dep::version` 的简写，没有运算符的要求表示 `^`，所以下面三行要求的是同一件事。包的 README 常用简写形式（`"json": "0.2"`，表示 `>=0.2.0, <0.3.0`）。

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "textwrap": "1.2",
    // "textwrap": "^1.2",
    // "textwrap": Dep::version("^1.2"),
} }
```

路径不是版本：要写 `Dep::path("../geom")`，而不是 `"../geom"`。

依赖还有两个键：`package:` 给它重命名，于是代码以 `j5` 来称呼 `json5` 包；`target:` 把它限制在构建匹配目标时使用。

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "j5": Dep::version("^0.2", package: "json5"),
    "nix": Dep::version("^0.4", target: "cfg(target_family = \"unix\")"),
} }
```

一个包只能看到它自己的依赖，并且用的是它给这些依赖起的名称。这里 `j5::..` 是 `json5`，而 `json5::..` 什么也不是，即使构建中的另一个包以这个名称依赖 `json5`。依赖的名称同时也是某个子模块的名称时，`name::..` 会有两种含义，因此是错误：依赖写 `::name::..`，模块写 `pkg::name::..`，或者用 `package:` 重命名依赖。

单独的版本要求总是指默认注册表的索引。要从镜像获取该索引，把镜像写进 `~/.nabi/sources.nabion`；`nabi.lock` 仍然记录默认注册表的索引，所以无论是否使用镜像，锁文件都保持不变。

```nabion
Sources { replace: { "https://github.com/nabi-lang/packages-index": "https://mirror.example.com/packages-index" } }
```

注册表保存每个已发布版本的压缩包，`nabi` 下载它（`Downloading json 0.2.0`）并对照索引检查其 SHA-256，所以即使包的 git 仓库消失了，包也能继续使用。只有注册表没有压缩包时，`nabi` 才会获取包的 git 仓库。

Nabi 项目发布的包（`json`、`toml`、`http`、`markdown`……）是**官方包**，使用它们时可以省去 `nabi add`：直接使用，下一次 `nabi run`（或构建、检查、测试、REPL 单元）会替你添加，并给出提示。

```text
$ cat src/main.nabi
use json::parse;
...
$ nabi run
 Downloading json 0.2.0
       Added json ^0.2 (official)
```

现在 `"json": "^0.2"` 已写入 `nabi.nabion` 并固定在 `nabi.lock` 中，与运行了 `nabi add json` 的效果一样。只有官方包会以这种方式加入；其他任何名称仍会要求你运行 `nabi add <name>`。使用 `--locked`（或 `--frozen`），或者在 CI 中（设置了 `CI` 变量）时，`nabi` 绝不修改你的清单，而是提示你自己添加该包。

工具链自带官方包，当它自带的版本满足你的要求时，就选用这个版本：不查索引，也不下载，所以没有网络也能工作。`nabi update` 仍然会移到最新版本，`nabi component update packages` 则会提前下载每个官方包的最新版本。

对一个包的每条要求都会被考虑。如果 `app` 需要 `json` `>=0.1`，而 `table` 需要 `json` `^0.1`，`nabi` 就选两者都接受的最新 `0.1.x`，必要时还会把中间的包退回到较旧的版本。没有任何版本满足时，错误会列出谁要求什么，以及要修改什么：

```text
nabi: error: no version of package `b` meets every requirement on it:
  `^2` (required by `a` 1.4.0)
  `^1` (required by `c` 0.3.0)
... has b 1.2.0, 2.0.1; none of them meets all of these
how those packages came in:
  `a` 1.4.0: the root package requires `a` `^1.0`
  `c` 0.3.0: the root package requires `c` `~0.3`
one build has one version of a package
help: `c` 0.4.0 gets along with the other requirements on `b`: change the root package's requirement on it, `"c": "^0.4.0"`
```

### 安全公告：`nabi audit`

注册表发布[安全公告](https://packages.nabi-lang.org/advisories)：一个包的哪些版本有已知漏洞，以及哪个版本修复了它。`nabi audit` 用这些公告检查 `nabi.lock` 中的包，并告诉你该升级到哪个版本：

```sh
nabi audit
```

没有适用的公告时以 0 退出，有适用的公告时以 1 退出（于是 CI 任务失败），无法检查时以 2 退出。`--ignore <id>` 在你读过某条公告后接受它，`--json` 为工具输出记录。详细信息以及如何报告漏洞，见[安全公告](packages/04-security-advisories.html)。

库可以提供用户能开启和关闭的**特性**，你也可以用 `nabi login` 和 `nabi publish` 把自己的库作为包发布。特性与 Nabi 项目维护的包（`json`、`toml`、`csv`、`http`、`markdown`、`spatial`、`nabi-syntax`）一起在[第 18 章](18-features-and-publishing.html)中介绍，发布则见[发布包](packages/index.html)。

## 工作区

一起开发的多个包，比如一个应用和它使用的库，可以放在一个**工作区**中。顶层目录中放一个 `nabi.nabion`，它列出成员，而不是描述一个包：

```nabion
Workspace { members: ["app", "libs/*"] }
```

```text
shop/
|-- nabi.nabion          Workspace { members: [..] }
|-- nabi.lock            所有成员共用一个锁文件
|-- target/              一个构建目录
|-- app/                 Package { name: "app", deps: { "cart": Dep::path("../libs/cart") } }
`-- libs/cart/           Package { name: "cart", .. }
```

`libs/*` 表示 `libs/` 下的每个直接子包。成员共享 `nabi.lock` 和 `target/`，所以它们都基于相同版本的依赖构建，一个库也只为所有成员构建一次。成员之间通过 `Dep::path` 相互使用。

在顶层运行命令，会在每个成员上运行，也可以用 `-p` 挑选成员；在某个成员内部，`--workspace` 会覆盖所有成员：

```text
$ nabi test                 # 在顶层：所有成员
$ nabi run -p app           # 一个成员
$ cd libs/cart && nabi test --workspace
```

## 单文件脚本与环境

对于不值得建一个包的小脚本，它的依赖写在文件最顶部 `//!` 注释中的一个 ```` ```nabion ```` 块里。以脚本作为最后一个参数运行 `nabi add`，会替你写好这个块；路径相对于脚本书写，锁文件保存在缓存中。脚本使用的官方包会在第一次运行时加入它的块中。

```bash
nabi add geom --path ../geom tool.nabi
```

```text
       Added geom to tool.nabi
```

````nabi norun
//! ```nabion
//! Package {
//!     deps: {
//!         "geom": Dep::path("../geom"),
//!     },
//! }
//! ```
mod helper;

use geom::{Circle, area_sum};
println(f"{area_sum([Circle { r: helper::radius() }])}");
````

```bash
nabi run tool.nabi
```

脚本也可以有文件模块：`mod helper;` 加载它旁边的 `helper.nabi`，与包的 `src/` 中一样。模块文件可以带有自己的 `//!` 块（`nabi add <name> helper.nabi`），所有文件的块会合并在一起。脚本永远看不到用 `nabi install` 安装的库；只有它们的块才算数。脚本大到一个文件放不下时，`nabi init --script tool.nabi` 把它变成一个包：块变为 `nabi.nabion`，`tool.nabi` 变为 `src/main.nabi`，它的模块文件移入 `src/`。

具名环境用于 REPL 和 Jupyter。用 `nabi install` 把库安装进某个环境，用 `--env` 选择它，或用 `nabi env use` 更改默认环境；不需要激活步骤。还没有环境时，`nabi env list` 会告诉你如何创建。环境是 `~/.nabi/envs` 下的一个目录，所以它的名称必须在每个系统上都能用作文件名：不含 `/`、`\` 或开头的 `.`，最多 64 个字符，并且不能是 `con` 这样的 Windows 设备名。只有大小写不同的名称视为相同，`current` 保留给 `nabi env use` 所选的那个环境。

```text
$ nabi env create sci
     Created environment `sci` (~/.nabi/envs/sci)
$ nabi install --env sci geom --path ~/work/geom
   Installed geom into environment `sci` (for `nabi repl` / `nabi jupyter`; a script gets it with `nabi add geom <file.nabi>`)
$ nabi env use sci
$ nabi env list
* sci (active)
```

包从不使用环境：在包内部，只有它自己的 `nabi.nabion` 才算数。

## 安装程序

`nabi install` 也可以安装程序，类似 `cargo install`：带有 `src/main.nabi` 的包（来自注册表、git 或路径）或脚本文件，会以 `--release` 构建到 `~/.nabi/bin` 中。再次安装相同的注册表版本或 git 提交不会做任何事，并会提示如何重新构建；新版本会替换旧版本。路径或脚本文件总是会重新构建。

```text
$ nabi install tool
   Installed tool 0.1.0 (~/.nabi/bin/tool)
$ nabi install tool
`tool` 0.1.0 is installed already (~/.nabi/bin/tool); to rebuild it, run `nabi install tool --force`
$ nabi install tool --force
 Reinstalled tool 0.1.0 (~/.nabi/bin/tool)
$ nabi install tool@0.2
    Upgraded tool from 0.1.0 to 0.2.0 (~/.nabi/bin/tool)
```

`nabi install --list` 显示已安装的内容，`nabi uninstall tool` 将其删除。

## 小结

- 用 `mod` 创建模块，用 `pkg::` / `sup::` / `std::` 路径和 `use` 引用名称。
- 不写 `use` 就能使用的名称来自 prelude，这是一份由 `core` 中常用名称和模块组成的固定列表；其余内容通过模块访问（`task::sleep`、`ops::Add`）或导入。
- 项默认是私有的。用 `pub(sub)`、`pub(pkg)` 和 `pub` 扩大其范围。
- `mod x;` 对应 `x.nabi`（以及 `x/`）。根是 `main.nabi` 或 `lib.nabi`。
- 用 `nabi new [--lib]` 创建包，用 `nabi add` 添加依赖（路径、git、注册表）。提交 `nabi.lock`。
- 工作区（`Workspace { members: [..] }`）让一起开发的包共用一个锁文件和一个 `target/`；`-p <name>` 和 `--workspace` 用于挑选成员。
- 用 `nabi search` / `nabi add name@^1.2` 使用注册表中的包；特性和发布见[第 18 章](18-features-and-publishing.html)。
- 脚本使用它们的 `//! ```nabion` 块（`nabi add <name> <file.nabi>`），`nabi init --script` 把脚本变成包；`nabi env` 用于 REPL / Jupyter。

## 延伸阅读

- 规范：[模块](../spec/refs-lang/005-modules/504-modules.html)、[`use`](../spec/refs-lang/005-modules/505-use.html)、[可见性](../spec/refs-lang/005-modules/506-visibility.html)、[路径](../spec/refs-lang/005-modules/507-paths.html)、[重新导出](../spec/refs-lang/005-modules/508-re-export.html)
- 工具：[`nabi` 前端](../spec/refs-lang/009-tools/910-nabi-frontend.html)、[`nabi.nabion` / `nabi.lock`](../spec/refs-lang/009-tools/910-nabi-frontend.html#9103-nabinabion--nabilock)、[注册表包](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104a-注册表包)、[环境](../spec/refs-lang/009-tools/910-nabi-frontend.html#9105-环境)、[内联清单](../spec/refs-lang/009-tools/910-nabi-frontend.html#9107-单文件脚本的内联清单)
- [1. 入门](01-getting-started.html)
- 下一章：[12. 并发](12-concurrency.html)
