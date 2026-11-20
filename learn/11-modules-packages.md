# 11. Modules and Packages

As a program grows, you need to split the code up and use libraries written by other people. In Nabi, the unit for dividing code is the **module**, and the unit for building and distributing is the **package**.

## `mod` and paths

`mod name { .. }` defines a module in place. Items inside a module are referred to with a `module::item` path, and modules can contain further modules.

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

Only certain words can start a path.

| Start    | Meaning                                                      |
|----------|--------------------------------------------------------------|
| `pkg::`  | From the root of the current package                         |
| `sup::`  | From the parent module (`sup::sup::` to go further up)       |
| `std::`, `core::`, `bare::` | The standard library's tiers |
| `::std::` | The standard library even where a local `std` exists |
| a name   | An item or child module of the current module, or a name brought in with `use` |

Name lookup does not search parent modules automatically. That is why using `Point` inside `dist` requires bringing it in explicitly with `use sup::Point;`. Thanks to this, you can always tell from within a file where a name comes from.

`std`, `core` and `bare` are ordinary names. A module or dependency of yours may be called `std`; in the module that has it, `std::` means yours, and `::std::` still means the standard library:

```nabi
mod std { pub fn hello() -> str { "mine" } }

fn main() {
    println(std::hello());                       // mine
    println(::std::path::Path::empty().is_relative());  // the library's
}
```

## Importing with `use`

Instead of writing long paths every time, bring names in with `use`. You can import several with `{..}`, rename with `as`, and import all public items of a module with `*`.

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

A name you import by name must not also be defined in the same module; the definition would hide the import without a word. A glob (`use shapes::*;`) gives way to the module's own items.

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

Names like `List`, `Dict`, and `println`, which you have used so far without `use`, come from the *prelude*: every file starts with an implicit `use core::prelude::*;`. The prelude is a fixed, short list: the common types (`List`, `Dict`, `Result`, `Duration`, `Mutex`, ...), the traits written in `impl` headers and bounds (`Clone`, `Hash`, `Display`, `Iterator`, `From`, ...), the printing and panic functions, and the top-level modules of `core` (`task`, `par`, `ops`, `num`, `sync`, `iter`, ...). Everything else in `core` is reached through one of those modules or imported:

```nabi norun
task::sleep(Duration::ms(1));          // the module `task` is in the prelude
impl ops::Add for V { .. }             // operator traits live in `ops`
use core::sync::Frozen;           // or import the name
```

Your own names always win over the prelude's. Enum variants are not in the prelude; they are written bare where the expected type picks the enum ([chapter 6](06-structs-enums-traits.html)). When a name is not in scope, the error gives the path to write and the `use` that imports it. Modules in `std` such as files, networking, and threads are imported explicitly, as in `use std::fs;`.

A trait works the same way: its methods are found only where the trait is in scope, so a trait from another module or package needs a `use` (`use zoo::Animal;`) before `x.name()` resolves. The methods of the standard library's traits (`core`, `std`) work everywhere, without a `use`.

Macros, derives, and meta attributes belong to the package that defines them. Your own package and the standard library write them bare (`log!(..)`, `#[derive(Inspect)]`); another package's are written with its name, `json::log!(..)`, `#[derive(json::Json)]`, or imported once with `use json::log!;`. A bare name of another package's macro is an error whose help gives both forms.

## Visibility

By default, an item is visible only within its own module. To make it visible outside, add a visibility keyword.

| Keyword    | Visible from                            |
|------------|-----------------------------------------|
| (none)     | The same module                         |
| `pub(sub)` | The same module and its submodules      |
| `pub(pkg)` | Anywhere in the same package            |
| `pub`      | Other packages too                      |

Struct fields must also be made public individually. That is why `Point` above marks each field with `pub`, as in `pub x: f64`. If you use a private item from outside, the compiler tells you where to add `pub`.

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

The `balance` field and the `check` function are not visible outside `bank`, so the balance changes only through `bank`'s functions. For the same reason, code outside `bank` cannot write `bank::Account { owner: "kim" }`: a struct literal may leave out a private field only when the field has a default. That is why `bank::open` builds the account.

A few `pub` items of the standard library are marked `#[internal]`: the toolchain's own packages share them, and a use from any other package is an error, "internal to the Nabi toolchain". The API docs and the editor leave them out.

## Splitting into files

In a real package, modules are split into files. Declaring a module without a body, as in `mod shapes;`, makes `shapes.nabi` in the same directory the body of that module. If it has submodules, put a `shapes/` directory with the same name next to `shapes.nabi` (there is no special file name like `mod.nabi`).

```text
geom/
├── nabi.nabion
└── src/
    ├── lib.nabi          // mod shapes;
    ├── shapes.nabi       // shapes module body, declares mod circle;
    └── shapes/
        └── circle.nabi   // shapes::circle
```

A package's sources live in `src/`. The package root module is `src/main.nabi` for an executable and `src/lib.nabi` for a library. Executable top-level statements may appear only in `src/main.nabi`; other modules contain only items.

## Packages and `nabi.nabion`

`nabi new` creates a package: `src/main.nabi` for an executable, or with `--lib`, `src/lib.nabi` holding an example function and one `#[test]`. The manifest `nabi.nabion` is in [NABION](../spec/refs-standard-libraries/300-std/308-nabion.html) format. Besides the name and version it records `nabi_version`, the nabi that made it and so the oldest one known to build it (what a published package's users need, [chapter 18](18-features-and-publishing.html)), and `toolchain: "0.1"`, the toolchain series the package was made with, so a later toolchain from another series does not build it by surprise ([chapter 1](01-getting-started.html#managing-toolchains)). The standard library is no dependency: `std`, `core` and `bare` come with the toolchain, and a `"std": ..` line in `deps` is an error that says to remove it.

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

Let's create two modules in `geom`. If you re-export a submodule's items from the package root with `pub use`, users do not need to know the internal structure.

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

When you add a dependency in `app` with `nabi add`, it goes into `deps` in `nabi.nabion`. The name of the dependency package becomes its top-level path (`-` turns into `_`).

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

`PI` is `pub(pkg)`, so it can be used anywhere inside `geom` but is not visible from `app`. A git repository is added with something like `nabi add parser --git <url> --tag v1.2`. For git and registry dependencies, the exact commit is pinned in `nabi.lock`, so commit `nabi.lock` to version control as well. Only `nabi update` moves pinned versions forward.

## The package registry

The default package registry is [packages.nabi-lang.org](https://packages.nabi-lang.org). You search by name and add it together with a version requirement. Version requirements work as in Cargo: `^1.2` means `>=1.2.0, <2.0.0`. If you omit the requirement, the newest version is recorded with a `^`.

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

A misspelled name gets the closest package:

```text
$ nabi add jsn
    Updating index https://github.com/nabi-lang/packages-index
nabi: error: no package named `jsn` in https://github.com/nabi-lang/packages-index
help: did you mean `json`? (or search: `nabi search jsn`)
```

A plain string is shorthand for `Dep::version`, and a requirement with no operator means `^`, so these three lines ask for the same thing. Package READMEs often use the short form (`"json": "0.2"`, which means `>=0.2.0, <0.3.0`).

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "textwrap": "1.2",
    // "textwrap": "^1.2",
    // "textwrap": Dep::version("^1.2"),
} }
```

A path is not a version: write `Dep::path("../geom")`, not `"../geom"`.

Two more keys of a dependency: `package:` renames it, so code calls the package `json5` as `j5`, and `target:` limits it to builds for matching targets.

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "j5": Dep::version("^0.2", package: "json5"),
    "nix": Dep::version("^0.4", target: "cfg(target_family = \"unix\")"),
} }
```

A bare requirement always means the default registry's index. To fetch that index from a mirror, put the mirror in `~/.nabi/sources.nabion`; `nabi.lock` still names the default registry's index, so the lock stays the same with or without the mirror.

```nabion
Sources { replace: { "https://github.com/nabi-lang/packages-index": "https://mirror.example.com/packages-index" } }
```

The registry keeps an archive of every published version, and `nabi` downloads that (`Downloading json 0.2.0`) and checks its SHA-256 against the index, so a package keeps working even if its git repository goes away. Only when the registry has no archive does `nabi` fetch the package's git repository.

The packages the Nabi project publishes (`json`, `toml`, `http`, `markdown`, ...) are **official**, and you can skip `nabi add` for them: use one, and the next `nabi run` (or build, check, test, REPL cell) adds it for you and says so.

```text
$ cat src/main.nabi
use json::parse;
...
$ nabi run
 Downloading json 0.2.0
       Added json ^0.2 (official)
```

`"json": "^0.2"` is now in `nabi.nabion` and pinned in `nabi.lock`, just as if you had run `nabi add json`. Only official packages come in this way; any other name still asks you to run `nabi add <name>`. With `--locked` (or `--frozen`), or in CI (the `CI` variable set), `nabi` never edits your manifest and tells you to add the package yourself.

The toolchain carries the official packages, and when a version it carries meets your requirement, that is the one picked: no index and no download, so this works without a network. `nabi update` still moves to the newest version, and `nabi component update packages` downloads the newest of every official package ahead of time.

Every requirement on a package counts. If `app` needs `json` `>=0.1` and `table` needs `json` `^0.1`, `nabi` picks the newest `0.1.x`, which both accept, and backs up to older versions of the packages in between when that is what it takes. When nothing fits, the error shows who asks for what and what to change:

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

A library can offer **features** that its users turn on and off, and you can publish your own library as a package with `nabi login` and `nabi publish`. Both are covered in [chapter 18](18-features-and-publishing.html), together with the packages the Nabi project maintains (`json`, `toml`, `csv`, `http`, `markdown`, `spatial`, `nabi-syntax`).

## Workspaces

Packages you develop together, say an app and the libraries it uses, can live in one **workspace**. The top directory gets a `nabi.nabion` that lists the members instead of describing a package:

```nabion
Workspace { members: ["app", "libs/*"] }
```

```text
shop/
|-- nabi.nabion          Workspace { members: [..] }
|-- nabi.lock            one lock for every member
|-- target/              one build directory
|-- app/                 Package { name: "app", deps: { "cart": Dep::path("../libs/cart") } }
`-- libs/cart/           Package { name: "cart", .. }
```

`libs/*` means every package directly inside `libs/`. The members share `nabi.lock` and `target/`, so they all build against the same versions of their dependencies and a library is built once for all of them. Members use each other through `Dep::path`.

Run a command at the top to run it on every member, or pick members with `-p`; inside a member, `--workspace` reaches all of them:

```text
$ nabi test                 # at the top: every member
$ nabi run -p app           # one member
$ cd libs/cart && nabi test --workspace
```

## Single-file scripts and environments

For a small script that does not warrant a package, its dependencies live in a ```` ```nabion ```` block inside the `//!` comment at the very top of the file. `nabi add` with the script as the last argument writes the block for you; a path is written relative to the script, and the lockfile is kept in the cache. An official package the script uses is added to its block on the first run.

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

A script can have file modules too: `mod helper;` loads `helper.nabi` beside it, just as in a package's `src/`. A module file may carry its own `//!` block (`nabi add <name> helper.nabi`), and the blocks of all the files are merged. Scripts never see libraries installed with `nabi install`; only their blocks count. When a script outgrows one file, `nabi init --script tool.nabi` turns it into a package: the blocks become `nabi.nabion`, `tool.nabi` becomes `src/main.nabi`, and its module files move into `src/`.

Named environments are for the REPL and Jupyter. Install libraries into one with `nabi install`, and pick it with `--env` or change the default with `nabi env use`; there is no activation step. With no environment yet, `nabi env list` says how to make one. An environment is a directory under `~/.nabi/envs`, so its name must work as a file name on every system: no `/`, `\` or leading `.`, at most 64 characters, and not a Windows device name such as `con`. Names that differ only in case count as the same, and `current` is reserved for the one `nabi env use` chose.

```text
$ nabi env create sci
     Created environment `sci` (~/.nabi/envs/sci)
$ nabi install --env sci geom --path ~/work/geom
   Installed geom into environment `sci` (for `nabi repl` / `nabi jupyter`; a script gets it with `nabi add geom <file.nabi>`)
$ nabi env use sci
$ nabi env list
* sci (active)
```

A package never uses an environment: inside a package, its own `nabi.nabion` is all that counts.

## Installing programs

`nabi install` also installs programs, like `cargo install`: a package with `src/main.nabi` (from the registry, git or a path) or a script file is built with `--release` into `~/.nabi/bin`. Installing the same registry version or git commit again does nothing, and says how to rebuild; a new version replaces the old one. A path or a script file is always rebuilt.

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

`nabi install --list` shows what is installed, and `nabi uninstall tool` removes it.

## Summary

- Create modules with `mod`, and refer to names with `pkg::` / `sup::` / `std::` paths and `use`.
- Names used without `use` come from the prelude, a fixed list of common `core` names and modules; the rest is reached through a module (`task::sleep`, `ops::Add`) or imported.
- Items are private by default. Widen their scope with `pub(sub)`, `pub(pkg)`, and `pub`.
- `mod x;` maps to `x.nabi` (and `x/`). The root is `main.nabi` or `lib.nabi`.
- Create packages with `nabi new [--lib]` and add dependencies (path, git, registry) with `nabi add`. Commit `nabi.lock`.
- A workspace (`Workspace { members: [..] }`) gives packages developed together one lock and one `target/`; `-p <name>` and `--workspace` pick members.
- Work with registry packages using `nabi search` / `nabi add name@^1.2`; features and publishing are in [chapter 18](18-features-and-publishing.html).
- Scripts use their `//! ```nabion` blocks (`nabi add <name> <file.nabi>`), and `nabi init --script` turns one into a package; `nabi env` is for the REPL / Jupyter.

## Further reading

- Spec: [Modules](../spec/refs-lang/005-modules/504-modules.html), [`use`](../spec/refs-lang/005-modules/505-use.html), [Visibility](../spec/refs-lang/005-modules/506-visibility.html), [Paths](../spec/refs-lang/005-modules/507-paths.html), [Re-exports](../spec/refs-lang/005-modules/508-re-export.html)
- Tools: [`nabi` front-end](../spec/refs-lang/009-tools/910-nabi-frontend.html), [`nabi.nabion` / `nabi.lock`](../spec/refs-lang/009-tools/910-nabi-frontend.html#9103-nabinabion--nabilock), [Registry packages](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104a-registry-packages), [Environments](../spec/refs-lang/009-tools/910-nabi-frontend.html#9105-environments), [Inline manifests](../spec/refs-lang/009-tools/910-nabi-frontend.html#9107-inline-manifest-for-single-file-scripts)
- [1. Getting Started](01-getting-started.html)
- Next chapter: [12. Concurrency](12-concurrency.html)
