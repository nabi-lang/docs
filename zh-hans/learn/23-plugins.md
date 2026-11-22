# 23. 插件

插件是程序在运行时加载的包：游戏关卡、编辑器工具、用户放进某个文件夹的过滤器。在 Nabi 中，插件就是普通的库包，只在清单中多写一行。它运行在你的程序的运行时上，因此它返回的一切，包括字符串、列表、闭包和 `any` 值，都是你的程序中的普通值，调用它的函数也和调用其他函数一样。

本章构建一个程序，加载一个插件，调用它，再卸载它。

## 插件包

插件是一个库：有 `src/lib.nabi`，没有 `src/main.nabi`。在它的 `nabi.nabion` 中写上 `lib: { kind: "plugin" }`，`nabi build` 就会把它链接为插件。

```nabion
Package {
    name: "greeter",
    version: "0.1.0",
    lib: { kind: "plugin" },
    deps: {},
}
```

程序从它那里得到的是它的 `pub fn`：

```nabi norun
pub fn greet(name: str) -> str {
    f"hello, {name}"
}

pub fn squares(n: i32) -> List<i32> {
    let xs: List<i32> = List::new();
    for i in 0..n {
        xs.push(i * i);
    }
    xs
}

pub fn counter() -> fn() -> i32 {
    let n = [0];
    || {
        n[0] += 1;
        n[0]
    }
}
```

```bash
nabi build
```

插件在 macOS 上是 `target/debug/libgreeter.dylib`，在 Linux 上是 `libgreeter.so`，在 Windows 上是 `greeter.dll`。`std::plugin::file("greeter")` 给出目标平台上的文件名。

## 加载插件

程序用 `plugin::load` 加载文件，再按路径和类型请求每个函数。类型会被检查：插件声明的是 `fn(str) -> str`，却请求 `fn(i32) -> str`，得到的是错误，而不是崩溃。

```nabi norun
use std::plugin;

fn main() throws {
    let file = plugin::file("greeter");
    let p = try plugin::load(f"../greeter/target/debug/{file}");
    let greet = try p.get::<fn(str) -> str>("greeter::greet");
    println(greet("Ada"));

    let squares = try p.get::<fn(i32) -> List<i32>>("greeter::squares");
    println(squares(5));

    // 插件创建的闭包会保留自己的状态。
    let next = try p.get::<fn() -> fn() -> i32>("greeter::counter")();
    println(f"{next()} {next()} {next()}");
}
```

```text
hello, Ada
[0, 1, 4, 9, 16]
1 2 3
```

程序一侧不需要任何声明：使用 `std::plugin` 的程序在构建时就能承载插件。

也可以不写路径，让 `nabi` 替你构建插件：带 `plugin: true` 的依赖会作为插件构建在你的程序旁边（相同的构建配置、相同的目标），`plugin::beside(name)` 会在那里找到它。

```nabion
Package {
    name: "app",
    deps: { "greeter": Dep::path("../greeter", plugin: true) },
}
```

```nabi norun
let p = try plugin::load(plugin::beside("greeter"));
```

## 共享类型

程序和插件可以交换双方都知道的类型的值：把该类型放进两者都依赖的包中。

```nabion
// game/nabi.nabion: 程序
Package { name: "game", deps: { "engine": Dep::path("../engine") } }

// enemies/nabi.nabion: 插件
Package { name: "enemies", lib: { kind: "plugin" }, deps: { "engine": Dep::path("../engine") } }
```

这样 `engine::Entity` 在两边就是同一个类型，`p.get::<fn(engine::World) -> List<engine::Entity>>("enemies::spawn")` 可以原样来回传递这些值。插件从程序那里取得 `engine`，所以 `engine` 的静态变量只有一份。只有插件使用的依赖可以随插件一起带上：`Dep::version("^1.2", bundle: true)` 把它链接进插件。

`any` 和 `any Trait` 值也能跨越边界：插件可以返回一个装着它自己类型的 `any Display`，程序中的 `{v}` 会用插件的 `fmt` 打印它。反射也能看到插件的类型：`core::reflect::info_of(v)` 描述插件创建的值，`p.types()` 列出插件描述的类型。

## 加载失败时

每个错误都会说明该怎么做。文件不存在、文件不是插件、插件由其他工具链构建、程序缺少某个包、函数不存在或类型不符，都会以 `PluginError` 返回：

```nabi norun
let p = try plugin::load("plugins/missing.so") catch e {
    println(e);
    return;
};
```

```text
no plugin at plugins/missing.so; build its package with `nabi build` and give the file it writes under target/<profile>/ (`plugin::file(name)` gives the file name)
```

插件和程序必须来自同一个工具链：`nabi toolchain` 变更之后，请重新构建插件。不存在在工具链版本之间保持稳定的插件 ABI。

## 卸载

`p.unload()` 会立即让插件无法再被访问，但只要插件的任何部分还存活，它就留在内存中：它的某个类型的对象、它创建的闭包、仍在执行的对它的调用。返回结果会说明是哪种情况：

```nabi norun
match p.unload() {
    plugin::Unload::Done => println("gone"),
    plugin::Unload::Pending(why) => println(f"later: {why}"),
    plugin::Unload::Kept(why) => println(f"stays: {why}"),
}
```

如果插件把自己代码中的回调交给了 C 库，它会被永久保留，因为 C 可能仍会调用它。

## 插件在哪些平台可用

插件可以在 macOS（arm64）、Linux（glibc，x86-64 和 arm64）和 Windows（x64 和 arm64）上加载。在 WebAssembly 和静态 musl 目标上，程序无法在运行时加载代码，因此为它们构建插件或使用 `std::plugin` 的程序时，会以错误停止，错误会提示改为构建单个程序：把插件的包作为普通依赖。

## 小结

- `lib: { kind: "plugin" }` 把库包构建为插件；依赖上的 `plugin: true` 把它构建在你的程序旁边。
- `plugin::load(path)` 加载插件，`p.get::<F>("pkg::name")` 取得类型恰好为 `F` 的 `pub fn`。
- 值原样跨越边界：一个运行时，一个堆。通过两者都依赖的包共享类型。
- `unload` 会等到插件的任何部分都不再存活。

## 延伸阅读

- 规范：[插件](../spec/refs-lang/009-tools/912-plugins.html)、[模块描述符 ABI](../spec/refs-lang/008-abi/812-module-abi.html)、[`std::plugin`](../spec/refs-standard-libraries/300-std/311-plugin.html)
- API：[std::plugin](../../std/std/plugin.html)
- 上一章：[22. 用 Nabi 做科学计算](22-science.html)
