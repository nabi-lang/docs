# 1. Getting Started

In this chapter you install the nabi toolchain, create and run your first package, and try a single-file script and the REPL. By the end you can run the examples in the rest of the guide yourself.

## Installation

### Prerequisites

`nabic` is self-contained: it has its own code generator and linker, so you do not install a C compiler or linker ([9.2.18](../spec/refs-lang/009-tools/902-aot-pipeline.html#9218-self-contained-toolchain)).

- macOS: the Xcode Command Line Tools (`xcode-select --install`), for the SDK that the final link needs.
- Linux: nothing. The toolchain and the programs it builds need glibc 2.34 or later (Ubuntu 22.04, Debian 12, RHEL 9 and newer).
- Windows: nothing.
- FreeBSD: nothing. The toolchain and the programs it builds need FreeBSD 14 or later.

Release archives are built for macOS arm64, Linux x86_64 and arm64 (both glibc and static musl), Windows x86_64 and arm64, and FreeBSD x86_64. Other hosts have no release archive; build Nabi from source.

### Install script

The install script downloads the toolchain for your platform from the [GitHub releases](https://github.com/nabi-lang/nabi/releases), checks it against its SHA-256, and unpacks it into `~/.nabi`.

```bash
curl -fsSL https://nabi.sh | sh
```

On Windows, run this in PowerShell. It installs into `%USERPROFILE%\.nabi` and adds its `bin` to your user `PATH`.

```powershell
powershell -ExecutionPolicy Bypass -c "irm https://nabi.sh/install.ps1 | iex"
```

`install.sh --version <v>` and `install.ps1 -Version <v>` pick a release by its exact version (`0.1.0`); without them you get the newest release.

You can change the install location with the `NABI_HOME` environment variable (default `~/.nabi`). When the script finishes, you get the following layout.

```text
~/.nabi/
├── bin/                        # nabi, nabic, nabicc, nabils proxy
└── toolchains/
    └── 0.1.0-aarch64-apple-darwin/
        ├── bin/{nabi, nabic, nabicc, nabils}
        └── lib/                # runtime and standard library sources
```

The toolchain you just installed becomes the default. The four executables in `~/.nabi/bin` are **proxies**: the real work is done by the executable of the same name in the active toolchain. This lets you install several toolchains side by side and pick one (the same approach as rustup). The proxies are always copies from the newest installed toolchain, which knows how every older one is laid out, so making an older toolchain the default leaves them as they are.

The script also puts `~/.nabi/bin` on your `PATH`. It writes `~/.nabi/env` (and `~/.nabi/env.fish`), loads it from your shell's startup files such as `~/.zshenv`, `~/.bashrc` or `~/.profile`, and adds `~/.config/fish/conf.d/nabi.fish` for fish. It names every file it changed:

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

Running the script again changes nothing. To keep your shell files as they are, pass `--no-modify-path` (`-NoModifyPath` for `install.ps1`) or set `NABI_NO_MODIFY_PATH=1`; the script then prints the line to add to `PATH` yourself. The toolchain comes with the official packages (`json`, `http`, `regex`, `table` and the rest), so code that uses them builds without a network; to leave them out, pass `--no-packages` (`-NoPackages`, or `NABI_NO_PACKAGES=1`). Shell completion for `nabi` is installed too (bash, zsh, fish, PowerShell). `nabi completions <shell>` prints the script if you want to install it elsewhere.

Check the version in a new shell.

```bash
nabi --version
nabi toolchain list
```

`nabi --version -v` also names the binary that ran, the toolchain it picked and why, its `nabic`, and the host, one `key: value` line each (`nabi version -v --format json` prints them as one JSON object). Use it when the version is not the one you expect.

```text
$ nabi --version -v
nabi 0.1.0 (<commit> <date>)
binary:    /Users/me/.nabi/toolchains/0.1.0-aarch64-apple-darwin/bin/nabi
toolchain: 0.1.0-aarch64-apple-darwin (the default)
nabic:     /Users/me/.nabi/toolchains/0.1.0-aarch64-apple-darwin/bin/nabic
host:      aarch64-apple-darwin
```

Four executables are installed.

| Executable | What it does |
|------------|--------------|
| `nabic`     | The compiler. Build, check, test and bench harnesses ([9.1](../spec/refs-lang/009-tools/901-nabic-cli.html)) |
| `nabi`      | Package / build front-end and tools. `nabi.nabion`, dependencies, environments, run, lint, format, docs, REPL, Jupyter ([9.10](../spec/refs-lang/009-tools/910-nabi-frontend.html)) |
| `nabicc`    | The C compiler, also run as `nabi cc` ([9.13](../spec/refs-lang/009-tools/913-c-compiler.html), [chapter 14](14-ffi.html#c-sources-nabi-cc)) |
| `nabils`    | Language server (LSP) |

### Managing toolchains

`nabi toolchain` manages installed toolchains. The active toolchain is chosen, in order, by `+version` on the command line (`nabi +0.1 build`), the `NABI_TOOLCHAIN` environment variable, the `toolchain` field of the nearest `nabi.nabion` (or of a script's `//! nabion` block), and finally the default. A pin is a series (`"0.1"`, the newest installed 0.1.x release, or its newest release candidate while no 0.1 release is installed), a version (`"0.1.0"`) or a requirement (`"^0.1.0"`). One command runs on one toolchain from start to finish: even a toolchain's own `bin/nabi`, run by its full path, hands the command to the toolchain the project pins, and each toolchain's `nabi` drives only its own `nabic`.

```bash
nabi toolchain list                     # installed toolchains
nabi toolchain default 0.1.1            # change the default
nabi toolchain install 0.1.1            # download 0.1.1 beside the default
nabi toolchain update                   # download the newest release and make it the default
nabi toolchain update 0.1.1             # exactly 0.1.1 instead
```

`nabi toolchain update` takes an exact version, not a series. `nabi toolchain install` downloads one without changing the default (`--from <archive>` installs a file you already have). The download is checked against its published SHA-256, and if anything fails the previous default stays in place. Older toolchains are kept. When an update moves to a new series, it adds a note: only the newest series gets fixes, so move your projects to it. Until a project moves, it can pin the old series, as `toolchain: "0.1"` pins 0.1, or run `nabi +0.1 ..`. Do not confuse it with `nabi update`, which moves the versions pinned in a package's `nabi.lock` ([chapter 11](11-modules-packages.html)).

A newer toolchain of the same series builds what an older one did. Below 1.0 the series is `0.y`: 0.1.7 builds what 0.1.3 built, while 0.2.0 may not. From 1.0 it is `x`: 1.9.0 builds what 1.4.2 built, while 2.0.0 may not. A package's `nabi_version` follows this rule: `"0.1.3"` means nabi 0.1.3 or a later 0.1.x.

If another `nabi` (from a package manager, say) comes before `~/.nabi/bin` on your `PATH`, the new default would not take, so `nabi toolchain update` and `nabi toolchain default` warn:

```text
nabi: warning: your shell runs /opt/homebrew/bin/nabi (nabi 0.1.3) before ~/.nabi/bin/nabi; put ~/.nabi/bin first on PATH or remove the other one
```

### Building from source (optional)

You can also build a toolchain archive yourself and install it with the same script. This needs a Rust toolchain (`rustup`).

```bash
git clone https://github.com/nabi-lang/nabi && cd nabi
scripts/dist/native.sh                     # dist/nabi-<version>-<triple>.tar.gz
sh install.sh --archive dist/nabi-*.tar.gz  # into ~/.nabi
```

The script also builds each target's runtime and C library with `targets/build.sh`, which needs `curl`, `python3` and `tar` with xz, and keeps its downloads in `~/.cache/nabi` (`NABI_CACHE` moves them). On Linux, `scripts/dist/gnu.sh` (podman or docker) builds the same archive in a container so that it runs on glibc 2.34 and later; the release archives are built this way. The repository's README has the details.

If you are working on the compiler, you can use the build output directly without installing. `cargo build` makes the executables and the host's runtime. `targets/build.sh <triple>` builds the C library and runtime files of a target: a Linux host needs its own to link programs, and every cross target needs them; on macOS and Windows the host needs nothing more.

```bash
cargo build --release
targets/build.sh "$(rustc -vV | sed -n 's/^host: //p')"
export PATH="$PWD/target/release:$PATH"
```

## Your first package

`nabi new` creates a package directory and initializes a git repository with a `.gitignore` that excludes `target/`. Package names use lowercase letters, digits, `-`, and `_`, start with a letter, and are at most 64 characters long. A name Windows keeps for a device (`con`, `nul`, `com1`, ...) is refused, because no file can be called that there. The name `nabi` is taken by the standard library, so a package can't be called `nabi`. In code, `-` becomes `_` (`nabi new my-lib` is imported as `use my_lib::..`).

```bash
nabi new hello
cd hello
```

```text
     Created executable package (hello)
             cd hello && nabi run
```

The second line is the command to run next. There are two files. The manifest `nabi.nabion` holds the package name, version, the oldest nabi it builds with (`nabi_version`), the toolchain series it was made with, and dependencies.

```nabion
Package { name: "hello", version: "0.1.0", nabi_version: "0.1.0", toolchain: "0.1", deps: {} }
```

The source lives in `src/`; `src/main.nabi` is a single line that prints a greeting.

```nabi
println("Hello, world!");
```

```output
Hello, world!
```

`nabi run` builds and runs it. The output goes to `target/debug/`. A build compiles only the parts of the standard library the program uses, and the toolchain keeps them, so later builds of any package take them from its cache.

```bash
nabi run
```

```text
   Compiling hello (.)
    Finished in 0.87s
     Running `target/debug/hello`
Hello, world!
```

The top-level statements in `src/main.nabi` run from top to bottom and form the program. Function and type declarations can go anywhere between top-level statements. Other common commands are `nabi build` (build only), `nabi check` (type check only), `nabi test`, and `nabi fmt`.

## A slightly bigger program

Change `src/main.nabi` to the following and run `nabi run` again.

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

`f"..."` is an f-string: each expression inside `{}` is evaluated and inserted in place. `greet` is declared below its use, yet the call works, because declarations such as functions are visible regardless of order.

`println` also takes several values and prints them with a space between, `sep:` and `end:` change what goes between and after them, and `dbg(e)` prints an expression with its place and value to stderr while you debug:

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

Instead of top-level statements, you can use `fn main()` as the entry point. Use one or the other.

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

## Single-file scripts

Quick experiments don't need a package. Give `nabi run` a file path and it runs that file on its own. `nabi build hi.nabi` writes the executable `hi` next to the script.

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

Scripts can use dependencies too: `nabi add json hi.nabi` writes them into a `//!` comment at the top of the file (see [inline manifests](../spec/refs-lang/009-tools/910-nabi-frontend.html#9107-inline-manifest-for-single-file-scripts) and [chapter 11](11-modules-packages.html#single-file-scripts-and-environments)).

## REPL

Running `nabi` with no arguments opens the REPL. Enter an expression and it prints the value; end it with `?` to print the `Debug` form instead. A value with no `Display` prints only its type, and a `let` just defines the name.

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

`:type expr` reports the type of an expression without running it. Whenever you wonder about a type while reading this guide, check it in the REPL. A cell's last `let` needs no `;`. [Chapter 16](16-tools.html#repl-and-jupyter) covers the rest ([JIT cells](../spec/refs-lang/009-tools/907-jit-cells.html)).

## Summary

- Install into `~/.nabi` with `curl -fsSL https://nabi.sh | sh` (`install.ps1` on Windows). The script puts `~/.nabi/bin` on `PATH` and installs shell completion.
- `nabi new name` creates `nabi.nabion` and `src/main.nabi`; `nabi run` builds and runs.
- The top-level statements of `src/main.nabi` are the program. You can also use `fn main()`.
- `nabi run file.nabi` runs a script; `nabi` alone starts the REPL.

## Further reading

- [Install layout](../spec/refs-lang/009-tools/902-aot-pipeline.html#9219-installation-layout), [pinning the toolchain](../spec/refs-lang/009-tools/910-nabi-frontend.html#9106-toolchain)
- [`nabi` front-end](../spec/refs-lang/009-tools/910-nabi-frontend.html), [`nabic` CLI](../spec/refs-lang/009-tools/901-nabic-cli.html)
- [Entry point](../spec/refs-lang/004-functions/416-entry-point.html)
- [JIT cells and the REPL](../spec/refs-lang/009-tools/907-jit-cells.html)
- Next chapter: [2. Values and Types](02-values-and-types.html)
