# 23. Plugins

A plugin is a package your program loads while it runs: game levels, editor tools, filters a user drops into a folder. In Nabi a plugin is an ordinary library package built with one more line in its manifest. It runs on your program's runtime, so everything it returns, strings, lists, closures, `any` values, is a normal value of your program, and you call its functions like any other function.

This chapter builds a program that loads a plugin, calls it, and unloads it.

## A plugin package

A plugin is a library: `src/lib.nabi`, no `src/main.nabi`. `lib: { kind: "plugin" }` in its `nabi.nabion` makes `nabi build` link it as a plugin.

```nabion
Package {
    name: "greeter",
    version: "0.1.0",
    lib: { kind: "plugin" },
    deps: {},
}
```

Its `pub fn`s are what programs get from it:

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

The plugin is `target/debug/libgreeter.dylib` on macOS, `libgreeter.so` on Linux and `greeter.dll` on Windows. `std::plugin::file("greeter")` gives that name for the platform you build for.

## Loading it

A program loads the file with `plugin::load` and asks for each function by its path and its type. The type is checked: asking for `fn(i32) -> str` when the plugin declares `fn(str) -> str` is an error, not a crash.

```nabi norun
use std::plugin;

fn main() throws {
    let file = plugin::file("greeter");
    let p = try plugin::load(f"../greeter/target/debug/{file}");
    let greet = try p.get::<fn(str) -> str>("greeter::greet");
    println(greet("Ada"));

    let squares = try p.get::<fn(i32) -> List<i32>>("greeter::squares");
    println(squares(5));

    // A closure the plugin made keeps its state.
    let next = try p.get::<fn() -> fn() -> i32>("greeter::counter")();
    println(f"{next()} {next()} {next()}");
}
```

```text
hello, Ada
[0, 1, 4, 9, 16]
1 2 3
```

Nothing has to be declared on the program's side: a program that uses `std::plugin` is built to host plugins.

Instead of a path, let `nabi` build the plugin for you: a dependency with `plugin: true` is built as a plugin next to your program (same profile, same target), and `plugin::beside(name)` finds it there.

```nabion
Package {
    name: "app",
    deps: { "greeter": Dep::path("../greeter", plugin: true) },
}
```

```nabi norun
let p = try plugin::load(plugin::beside("greeter"));
```

## Sharing types

The program and the plugin can exchange values of a type both know: put it in a package both depend on.

```nabion
// game/nabi.nabion: the program
Package { name: "game", deps: { "engine": Dep::path("../engine") } }

// enemies/nabi.nabion: the plugin
Package { name: "enemies", lib: { kind: "plugin" }, deps: { "engine": Dep::path("../engine") } }
```

`engine::Entity` is then the same type in both, and `p.get::<fn(engine::World) -> List<engine::Entity>>("enemies::spawn")` passes them back and forth as they are. The plugin takes `engine` from the program, so `engine`'s statics have one copy. A dependency only the plugin uses can go in with it: `Dep::version("^1.2", bundle: true)` links it into the plugin.

`any` and `any Trait` values cross too: a plugin can return an `any Display` holding one of its own types, and `{v}` in the program prints it with the plugin's `fmt`. Reflection sees its types as well: `core::reflect::info_of(v)` describes a value the plugin made, and `p.types()` lists the types the plugin describes.

## When it does not load

Every error says what to do. A missing file, a file that is not a plugin, a plugin built by another toolchain, a package the program lacks, a function that is not there or has another type, all come back as a `PluginError`:

```nabi norun
let p = try plugin::load("plugins/missing.so") catch e {
    println(e);
    return;
};
```

```text
no plugin at plugins/missing.so; build its package with `nabi build` and give the file it writes under target/<profile>/ (`plugin::file(name)` gives the file name)
```

A plugin and the program must come from the same toolchain: after `nabi toolchain` changes, rebuild your plugins. There is no plugin ABI that stays stable between toolchain versions.

## Unloading

`p.unload()` takes the plugin out of reach at once, but it stays in memory while anything of it is alive: an object of one of its types, a closure it made, a call into it still running. The answer says which happened:

```nabi norun
match p.unload() {
    plugin::Unload::Done => println("gone"),
    plugin::Unload::Pending(why) => println(f"later: {why}"),
    plugin::Unload::Kept(why) => println(f"stays: {why}"),
}
```

A plugin that gave a C library a callback of its own code is kept for good, because C may still call it.

## Where plugins work

Plugins load on macOS (arm64), Linux (glibc, x86-64 and arm64) and Windows (x64 and arm64). On WebAssembly and on the static musl targets a program cannot load code while it runs, so building a plugin, or a program that uses `std::plugin`, for them stops with an error that says to build one program instead: make the plugin's package an ordinary dependency.

## Summary

- `lib: { kind: "plugin" }` builds a library package as a plugin; `plugin: true` on a dependency builds it beside your program.
- `plugin::load(path)` loads it, `p.get::<F>("pkg::name")` gets a `pub fn` of exactly type `F`.
- Values cross as they are: one runtime, one heap. Share types through a package both depend on.
- `unload` waits until nothing of the plugin is alive.

## Further reading

- Spec: [Plugins](../spec/refs-lang/009-tools/912-plugins.html), [Module descriptor ABI](../spec/refs-lang/008-abi/812-module-abi.html), [`std::plugin`](../spec/refs-standard-libraries/300-std/311-plugin.html)
- API: [std::plugin](../std/std/plugin.html)
- Previous chapter: [22. Science with Nabi](22-science.html)
