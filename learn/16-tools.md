# 16. Tools

This chapter tours the tools you keep at hand while writing code: the formatter, the linter, tests and benchmarks, documentation generation, editor support (LSP), the REPL and Jupyter, and debug options for looking inside the compiler. All of them start from the single `nabi` command.

The output in this chapter comes from actually running the package below. It was created with `nabi new mathx`, and `src/main.nabi` was changed to this:

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

## `nabi test`: tests

A function marked `#[test]` is a test. Test functions are left out of normal builds; `nabi test` collects them and builds them separately. Checks use `assert`, `assert_eq`, and `assert_ne`, which panic on failure. You can pass a message as the last argument.

Each test runs in its own process, so if one panics the rest keep running. `fib_wrong` in the package above is deliberately wrong.

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

If anything fails, the exit code is 1, so it works in CI as is. Passing part of a name (or a small regex using `^`, `$`, `.`, `*`, `+`, `?`) as an argument runs only the matching tests.

```text
$ nabi test gcd
   Compiling mathx (.)
    Finished in 0.37s

running 1 test
test gcd_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 2 filtered out; finished in 0.22s
```

Use `--show-output` to also see output from passing tests, and `--threads=N` to set how many tests run concurrently. For CI dashboards and other tools, `--format json` writes one JSON object per line on stdout, one per test and a final summary, and moves the human-readable text to stderr:

```text
$ nabi test --format json 2>/dev/null
{"schema":1,"type":"case","kind":"test","name":"fib_works","result":"ok","ms":87.057}
{"schema":1,"type":"case","kind":"test","name":"fib_wrong","result":"failed","ms":87.144,"reason":"panicked (exit 101)"}
{"schema":1,"type":"case","kind":"test","name":"gcd_works","result":"ok","ms":87.05}
{"schema":1,"type":"summary","kind":"test","result":"failed","passed":2,"failed":1,"ignored":0,"filtered_out":0,"ms":87.276}
```

`--exact` makes the filter match whole names only, and `--skip <pat>` leaves matching tests out (give it more than once to skip several). `--list` prints the names without running anything.

### Expected panics, ignored tests and timeouts

A test that should panic says so with `#[should_panic]`; with `expected`, the panic message must also contain that text. A slow test, or one that needs something CI lacks, gets `#[ignore]`, with an optional reason:

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

`nabi test --ignored` runs only the ignored tests, and `--include-ignored` runs them with the rest. A test that hangs does not hang the run: after 600 seconds it is killed and reported as failed. Set another limit with `--timeout=SECS` (or `NABI_TEST_TIMEOUT`); `--timeout=0` turns it off.

### Doc tests

In a library, `nabi test` also runs the code examples in its docs: each ```` ```nabi ```` block in `///` and `//!` comments and in `README.md` is built as a small program that uses the library, and run. The library's public items are in scope by name, as after `use mathx::*;`. An example that no longer compiles, or whose `assert` fails, fails the test run, so the docs stay correct.

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

Lines starting with `# ` run but are hidden from `nabi doc`, so an example can set things up without showing it. Words after `nabi` in the fence change what is checked: `nabi,norun` only builds the example (for code that reads files or waits on the network), `nabi,should_panic` expects a panic, `nabi,compile_fail` expects the example not to build (`compile_fail,E0040` for a specific error), and `nabi,ignore` skips it. Other languages (```` ```text ````, ```` ```nabion ````) are never run.

```text
running 2 tests
test doc README.md:9 ... ok
test doc src/lib.nabi:14 (gcd) ... ok
```

Each doc test is named by its file and line, plus the item it documents. An error inside an example points at the line in your source file. `nabi test --doc` runs only the doc tests, and `--no-doc` skips them.

## `nabi bench`: benchmarks

A `#[bench]` function takes a `Bench` and hands the code to time to `b.iter`. `nabi bench` builds them optimized (`-O2`), calls each closure many times, and prints the median time per call and its spread. A benchmark that never calls `b.iter` gets a warning. The numbers vary by machine and by run.

```text
$ nabi bench
   Compiling mathx (.)
    Finished in 0.42s

benchmarking 1 bench
  fib_20      12.99 us +/- 120.52 ns

bench result: ok. 1 run; 0 failed; 0 filtered out; finished in 2.63s
```

`--iters=N` is the number of calls per sample (by default, enough for about 100 ms), and `--samples=N` is the number of samples (default 10). Filters work as in `nabi test`. The output looks the same in a terminal and in a pipe; tools that read the results pass `--format=nabibench` for one `#nabibench` line per benchmark, or `--format json` for the same JSON lines as `nabi test`, with `median_ns`, `mad_ns` and `samples` in each case.

## Garbage collector pauses

The garbage collector stops every thread for short pauses. Two environment variables show them: `NABIR_GC_LOG=1` prints one line per collection on stderr, and `NABIR_GC_STATS=1` a summary when the program exits.

```text
$ NABIR_GC_LOG=1 ./target/release/server
nabi gc: #41 minor: 0.802 ms pause, 21046 objects marked, 141312 -> 104160 KiB; ttsp 9 us, mark 410 us, sweep 352 us, evac 0 us, threads 3, at 1520.114 ms
```

A pause counts from the moment the collector asks the threads to stop. `ttsp` (time to safepoint) is the part spent waiting for them: a long `ttsp` means a thread kept running, not that the collector was slow. Long string searches and counts, UTF-8 checks and large copies do not hold a pause back; they let the collector run beside them. The `NABIR_GC_STATS=1` summary ends with a line per kind of pause (minor, full, and the start and finish of a full collection marked concurrently) giving its p50, p90, p99 and longest.

A program reads the same counters with `bare::gc::stats()`, for a metrics endpoint or a frame budget. It takes the collector's lock for a moment, so read it now and then, not in a hot loop.

```nabi
use bare::gc;

let s = gc::stats();
println(f"{s.collections} collections, longest pause {s.pause_max_ns / 1000} us");
```

`bare::gc::collect()` runs a full collection and stops every thread for all of it: calling it to "clean up" between frames makes each frame longer. To keep pauses out of a frame, use `collect_young`, a pause goal, `no_collect` and arenas ([chapter 25](25-gc-pauses.html)).

## Other targets

With `--target`, `nabi test`, `nabi bench` and `nabi run` run the built program on this machine through a runner: `wasmtime` for `wasm32-wasip2` and `wasm32-wasip1-threads`, `node` for `wasm32-unknown-unknown`, and on Linux `qemu-<cpu>` (qemu-user) for another CPU's Linux. Each must be on `PATH`; when it is not, the error names what to install. A program the machine runs itself needs no runner: `i686-unknown-linux-gnu` on x86-64 Linux, and `armv7-unknown-linux-gnueabihf` on arm64 Linux when the CPU also runs 32-bit ARM code (Neoverse N1 and Cortex-A72 do; Neoverse N2 and Apple silicon do not), with the armhf C library installed (Debian, Ubuntu: `dpkg --add-architecture armhf`, then `apt install libc6:armhf`); `armv7-unknown-linux-musleabihf` programs are static and need no C library there.

```text
$ nabi test --target wasm32-wasip2 gcd
   Compiling mathx (.)
    Finished in 0.61s

running 1 test
test gcd_works ... ok

test result: ok. 1 passed; 0 failed; 2 filtered out; finished in 0.05s
```

To use another runner, or one for a target with none, set `NABI_RUNNER` to the command; the program and its arguments go after it:

```text
$ NABI_RUNNER="qemu-aarch64 -L /opt/aarch64-sysroot" nabi test --target aarch64-unknown-linux-gnu
```

On a Mac with Xcode, `aarch64-apple-ios-sim` programs run on the booted iOS simulator (`NABI_SIM_DEVICE` names another device); see [chapter 28](28-ios.html).

A WASI program has no working directory: a relative path is looked up from `/`, so use absolute paths in tests that run on wasm.

### Android

`x86_64-linux-android` builds programs and shared libraries for Android 10 and later, which run in the Android emulator (an x86_64 system image). Builds for arm64 phones and tablets (`aarch64-linux-android`) are in progress: that target is not tested on a device yet and comes in a later release. The link needs the Android NDK, from Android Studio's SDK Manager (SDK Tools, NDK) or from the NDK downloads page; `nabi` finds it in the Android SDK, or set `ANDROID_NDK_HOME` to it:

```text
$ export ANDROID_NDK_HOME=~/Android/Sdk/ndk/30.0.16248370
$ nabi build --target x86_64-linux-android --release
$ adb push target/x86_64-linux-android/release/hello /data/local/tmp/
$ adb shell /data/local/tmp/hello
Hello, world!
```

A library for an app's Java or Kotlin code is a `cdylib` whose exported functions have JNI's names; put the `.so` under the app's `jniLibs/x86_64/` (`arm64-v8a/` once device builds come) and load it with `System.loadLibrary`:

```nabi
use bare::ptr::CPtr;

#[export_name("Java_com_example_app_Native_add")]
pub extern "C" fn add(env: CPtr<c_void>, class: CPtr<c_void>, a: i32, b: i32) -> i32 { a + b }
```

```text
$ nabi build --target x86_64-linux-android --release --crate-type=cdylib
```

Programs run on Android 10 (API level 29) and later; `ANDROID_PLATFORM=34` builds for a later one. Programs and libraries load on devices with 4 KB or 16 KB pages, as Google Play asks of apps for Android 15 and later. A Linux program mostly runs as it is, as the kernel is Linux. What differs is in the [platform notes](../spec/refs-lang/000-meta/006-platform-targets.html#android).

## `nabi fmt`: the formatter

`nabi fmt` rewrites the package's `.nabi` and `.nabion` files into the standard format. Nabi's formatter is deliberately conservative: it **leaves tokens, spacing within a line, and comments alone**, and fixes indentation (4 spaces per bracket depth), trailing whitespace, and runs of blank lines. The one thing it moves is the `mod` / `use` lines at the top of a module: it groups them as `pub mod`, `mod`, `pub use`, `use`, sorts each group, and puts one blank line between groups. So the formatter never changes the meaning of code or alignment a person set up by hand.

The formatter reads operator tokens as the compiler does: a package's own `#[infix]` tokens and those its dependencies export (from `nabi.lock`, without fetching anything), so tests, benchmarks and dependents that use `a ⊛ b` or `x |> f` format too.

Checking a `fib` with messed-up indentation, mixed tabs, and trailing whitespace using `--check` leaves the file untouched, shows the lines that would change, and exits with code 1. This mode is a good fit for CI.

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

Without `--check`, it rewrites the files and prints how many it changed; `-q` leaves out that line. Running it on every save in your editor is convenient.

Inside a package, `nabi fmt` formats the whole package from any subdirectory, skipping `target/`, hidden directories, `node_modules/` and paths listed in `.gitignore`. Outside a package, name what to format: a file (`nabi fmt hi.nabi`) or a directory (`nabi fmt scripts/`). Without a path it stops and says so:

```text
$ nabi fmt
nabi: error: not inside a package; pass a file or directory: `nabi fmt <path>`
```

## `nabi lint` and `nabi check`

`nabi lint` does not compile; it runs only type checking and lints. It catches the `let unused = 1;` left in above.

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

Every lint defaults to `warn`, except `nan_comparison` and `self_assignment`, which are always a mistake and default to `deny`, and `missing_docs` (a `pub` item without a `///` comment), which is `allow` until you turn it on. The level can be changed in three places, and later ones win:

1. The `lints` field in `nabi.nabion`: `lints: { "unused": "deny", "dead_code": "allow" }`
2. `-A` / `-W` / `-D` (allow / warn / deny) on the command line
3. Attributes in source: `#[allow(dead_code)]` on an item, `#![deny(unused)]` at the top of a file

Instead of a single lint (`unused_variables`), you can name a group (`unused`, `nonstandard_style`, `warnings`, `all`). If any `deny` lint fires, the exit code is 1.

`unused_result` warns about a statement that throws a value away, such as `xs.len();` or `parse(s);`, since that is usually a mistake. Write `_ = expr;` when dropping it is what you mean. Functions whose result callers often do not need, such as `Dict::insert`, `List::pop` and `Atomic::fetch_add`, are marked `#[discardable]`, and a statement that calls one does not warn; mark your own functions the same way.

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

To fail CI on warnings, deny `warnings` (`lints: { "warnings": "deny" }`, or `-D warnings`). As in rustc, it covers every warning-level lint, including those a later release adds, so pin an exact toolchain in `nabi.nabion` (`toolchain: "=0.1.0"`) when a new release must not change what CI accepts. `all` is every lint. With `"warnings": "deny"` in `nabi.nabion`, the last note names where the level came from:

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

Some findings come with the edit that resolves them: an unused `let` gets a leading `_`, an unused `use` line is deleted, `while true` becomes `loop`, `x == true` becomes `x`, a redundant `as` or `.clone()` goes away, and a plain string holding `{name}` gets its `f`. Your editor offers each one as a quick fix; `nabi fix` applies them all across the package and checks again until nothing is left:

```text
$ nabi fix
       Fixed src/main.nabi (2 fixes)
```

`nabi fix` changes only your package's files, and it refuses while the work tree has uncommitted changes, so every fix shows up in `git diff` and can be undone (`--allow-dirty` skips that check). `nabi fix --diff` prints the fixes without writing them. Lint levels apply: `nabi fix -A unused_imports` leaves the imports alone. A compile error stops it before anything changes; fix that by hand first.

`deprecated` warns about any use of an item marked `#[deprecated]`: a function, method, field, type, constant or static, from your code, a dependency or the standard library; editors show such uses struck through. See the full list with `nabi lint --list`, which also gives the release that added each lint (piping it into `head` is fine: when the reader of its output goes away, `nabi` and `nabic` stop quietly instead of reporting an error). `nabi check` goes further and also runs code generation, but does not link or produce an executable. It is faster than a build, so use it to answer "does it compile?" The examples from here on use a throwaway `hello` package, so the errors and low-level output do not disturb `mathx`.

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

`nabi build` and `nabi run` print the same diagnostics, and a failed build ends with a count instead of `Finished`:

```text
$ nabi build
   Compiling hello (.)
...
error: could not build hello (1 error, 0 warnings)
```

For machine-readable output, add `--message-format=json`. Each diagnostic is printed as one line of JSON (`"schema": 1`): its code and kind, its message, spans with start and end line and column, `notes` and `help` lines, the `fixes` it suggests, and the `lint` name of a lint warning.

The code in brackets names the error, one code per kind of error (every lint shares `E0131` and names itself in a note). `nabic --explain E0040` (or `nabi --explain E0040`) prints what that code means and how to fix it, and the [error code index](../errors/index.html) lists every code. If nabic itself crashes (an internal compiler error), it exits with code 70, apart from a panicking program's 101; please report it.

## `nabi doc`: doc comments and API docs

`///` documents the item directly below it, and `//!` documents the module (file) containing the comment. In the package above, the first line `//! Small math helpers.` becomes the module description, and the `///` above `gcd` and `fib` become the function descriptions. A plain `//` comment is never documentation, even at the top of a file: `nabi lint` warns (`plain_module_doc`) when a library's root file or a `pub` module's file starts with `//` lines but has no `//!`, and the help shows the `//!` line to write. Doc text can use markdown such as headings, lists, inline code, code blocks, and links. The ```` ```nabi ```` examples in it are tested by `nabi test` (see [Doc tests](#doc-tests)).

```text
$ nabi doc
 Documenting mathx (.)
    Finished in 0.10s
   Generated ./target/doc/mathx/index.html
```

`nabi doc --open` also opens that page in your browser.

`target/doc/` gets HTML for the public (`pub`) items of the package and its dependencies, plus a search index: `target/doc/<package>/index.html` for the package, and one page per public module (`<package>/geo.html`, `<package>/geo/shapes.html`). Each page's sidebar lists the package's dependencies, linked to their pages, and the search finds items in all the documented packages. An item that a `pub use` brings in from a private module is shown on the page that re-exports it, as if it were declared there; a re-export of an item a public module shows links to that module's page. The pages look like this site's [standard library API](../std/core/index.html) and follow the same rules, and they work offline, opened straight from the disk.

## Editors: `nabils`

`nabils` is an LSP (Language Server Protocol) server. It provides diagnostics, hover, go to definition, find references, rename, document symbols, completion, signature help, quick fixes, and formatting. Completion offers the locals in scope, the members of a value (`xs.` lists the public methods of `List`), and paths after `use`. Hover and go to definition reach into the standard library too. Quick fixes apply the fix a diagnostic suggests, add a missing `;` or `use`, and run `nabi add` for a missing package. Rename refuses library items, keywords and names from other files, and says why. Diagnostics follow your typing, not your saves: each edit is checked again in the background, and hover, completion and go to definition answer at once from the last good check meanwhile, so a character pasted from a web page (a zero-width space, `−` for `-`, curly quotes), a lone combining mark, a malformed number, a bad escape or a literal suffix no handler takes (`10Hz` where `hz` is registered) is marked at once, with the compiler's own message and, where the fix is plain, a quick fix that writes it. Each file is checked with the dependencies of its own package, so a folder that holds several packages works, and a dependency added by the quick fix or by editing and saving `nabi.nabion` is found without restarting the server. `nabi lsp` is the same server. It communicates over standard IO, so any editor with LSP support only needs to be told the command to run (a `--stdio` the editor adds is accepted). Hover keeps answering while the file has a type error: from what that check typed, else from the last check that passed. Started by hand in a terminal, `nabils` only says that an editor starts it.

Official extensions start `nabils` for you and add syntax highlighting:

- VS Code: [nabi-lang/nabi.vscode](https://github.com/nabi-lang/nabi.vscode). Install `nabi-lang.nabi` from the [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=nabi-lang.nabi) (search for Nabi in the Extensions view, or `code --install-extension nabi-lang.nabi`); VSCodium, Cursor and other editors that use [Open VSX](https://open-vsx.org/extension/nabi-lang/nabi) find it there under the same id.
- Vim and Neovim: [nabi-lang/nabi.vim](https://github.com/nabi-lang/nabi.vim). Point your plugin manager at it (`Plug 'nabi-lang/nabi.vim'` with vim-plug); on Neovim it also starts `nabils`.
- Zed: [nabi-lang/nabi.zed](https://github.com/nabi-lang/nabi.zed). It is not in Zed's extension registry yet: install Rust with rustup, clone the repository, run `zed: install dev extension` from the command palette and pick the clone.

For Neovim (0.11 or later) without the plugin, add this to `init.lua`:

```text
vim.filetype.add({ extension = { nabi = "nabi", nabion = "nabion" } })
vim.lsp.config("nabils", {
  cmd = { "nabils" },
  filetypes = { "nabi", "nabion" },
  root_markers = { "nabi.nabion" },
})
vim.lsp.enable("nabils")
```

For Helix, add this to `languages.toml`:

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

In other editors, register `nabils` with the editor's LSP client or a generic LSP client extension. `nabils` must be on your `PATH`.

## REPL and Jupyter

A closer look at the REPL (`nabi` or `nabi repl`) from chapter 1. On a terminal it starts with two lines: the toolchain version and host, and a hint. `nabi repl -q` leaves them out, and piped input never gets them.

```text
$ nabi repl
Nabi 0.1.3 (<commit> <date>) on aarch64-apple-darwin
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

The prompt is `>>>`, and `...` while a cell goes on. A cell ends at the end of a line unless the input is clearly unfinished: a bracket, string or block comment is still open, the line ends in an operator, `.`, `,` or `=`, or it is an item header such as `fn f(x: i32)` still waiting for its body. When the cell's last statement is a `let` or an assignment, you can leave out its `;`.

Each cell runs once. Top-level `let` bindings and `fn` / `struct` / `impl` definitions carry over to later cells, and a cell may `await` at the top level. An expression cell prints its value with `Display`, else with `Debug`, else just its type. End it with `?` to print the `Debug` form instead. A `let` cell shows nothing. A cell of several statements shows the value of its last expression, if it ends in one. The value a cell shows is kept as `it`, so the next cell can use it; once you define your own `it`, yours wins.

Unlike regular source, you can `let` the same name again, even with another type. Defining an `fn` or a `struct` again replaces it, and code from earlier cells calls the new version. A cell that fails to compile, panics or throws leaves the session as it was. Output it already wrote stays written. `:reset` starts over from an empty session.

Errors point into the cell you typed as `<repl>:line:col`. Panics, and errors in an `fn` from an earlier cell, name that cell as `<repl cell N>:line:col`, where N counts the cells entered so far. In Jupyter, N is the notebook cell's execution count, and a notebook cell that splits into several REPL cells numbers them `N.1`, `N.2` and so on.

```text
>>> fn boom(n: i32) {
...     panic(f"boom {n}");
... }
>>> boom(5)
thread 'main' panicked at <repl cell 1>:2:5:
boom 5
error: the cell panicked; the session is unchanged
```

| Command | What it does |
|------|---------|
| `expr?` | Print the `Debug` form instead of `Display` |
| `:type expr` (`:t`) | Print the type without running it |
| `:doc name` | Print the signature and doc comment of a name; `:doc xs.len` for a method |
| `:time expr` | Run the expression and print how long each stage took |
| `:vars` | List the bindings as `name: type`, then the items |
| `:load file`, `:reload` | Load a program into the session; load the last one again. A script from `:save` is replayed cell by cell |
| `:edit` | Write cells in `$VISUAL` / `$EDITOR`, then run them |
| `:save file` | Write the session out as a script, one marked cell after another |
| `:opt [level]` | Show or set the optimization level for later cells (`0` to `3`, `s`) |
| `:reset` | Forget all cells and start over |
| `:add name ..` | Add dependencies: `nabi install` into the environment, or `nabi add` in a package. An official package a cell uses is added without it |
| `:help` (`:h`) | List every command |
| `:exit`, `:quit` (`:q`) | Quit; so does `exit` or `quit` alone, or end of input |

A closure that captures variables can't be kept in a cell's top-level `let`, because each closure has its own unnamed type. The cell is rejected:

```text
>>> let x = 3
>>> let f = |n: i64| n * x
error: `f: closure(i64) -> i64` cannot be kept for later cells (its type has no name)
```

Give the binding a function type, `let f: fn(i64) -> i64 = |n: i64| n * x`, which carries over. (An `fn` carries over too, but it cannot see cell variables such as `x`; pass them as arguments.) Closures used inside a cell, as in `xs.map(|n| n * x)`, work as usual.

On a terminal the REPL edits the line itself:

- Tab completes names: the locals in scope, the session's bindings and items, the core library, keywords, after `x.` the public fields and methods of `x`, and after `a::` the items of module `a`. At the start of a line, Tab indents.
- History keeps whole cells, also across sessions (`$NABI_HOME/repl_history`). Up and Down walk it; with some text typed, they step only through cells that start with it. Ctrl-R searches backwards.
- A pasted block becomes one cell when you press Enter, instead of running line by line.
- Input is coloured as in this guide. Set `NO_COLOR` to turn colours off.

A cell that reads standard input reads the terminal. When the REPL's own input is piped, a cell sees an empty input.

`nabi repl path` loads a package or a file into the session. For a library, the package is added as a dependency and `use name::*;` runs first, again after `:reset`. Handy for trying out functions while you build a library. For a program, it runs the program, then keeps its bindings and items in the session. Inside a package, `nabi repl .` loads the current one; a plain `nabi repl` starts empty.

The same cell engine is available in Jupyter notebooks. Register the kernel once, and you can pick the `nabi` kernel in Jupyter, VS Code notebooks, and so on.

```text
$ nabi jupyter install
   Installed kernel `nabi` (/Users/me/Library/Jupyter/kernels/nabi/kernel.json)
```

It is installed under `~/Library/Jupyter` on macOS, `%APPDATA%\jupyter` on Windows, and `$XDG_DATA_HOME/jupyter` (default `~/.local/share/jupyter`) elsewhere; `JUPYTER_DATA_DIR` changes the location. `nabi jupyter install --env name` installs a kernel for a named environment.

A notebook cell follows the REPL's rules and shows the value of its last expression. A value whose type implements `RichDisplay`, such as a plot's figure, shows as what its `mime_bundle()` gives (an SVG image, HTML), and `display(x)` shows one in the middle of a cell. Output appears while the cell runs, not only at the end. A cell that reads standard input gets an input box in the notebook. Tab completion and Shift-Tab help use the same names as the REPL. Code is highlighted with Rust rules until notebook front ends know Nabi.

## Optimization levels

`nabi build` builds the `debug` profile, `--release` the `release` one. `nabi` and `nabic` also take the level itself:

| Level | What it is for |
|-------|----------------|
| `-O0` | the default: no optimization, the quickest build, every value where the debugger expects it |
| `-O1` | debug builds that run fast: loops get what `-O2` gives them (a `for` over a list becomes a counted loop, vectorized where it can be), little is inlined elsewhere, and the build keeps its debug info |
| `-O2` | `--release`: fast code |
| `-O3` | a little more speed for more code: more is inlined in loops, and a scalar loop (a float sum, a call in the body) is unrolled |
| `-Os` | small code: only small functions are inlined, loops are still counted, and vectorized when short |

`-O2`, `-O3` and `-Os` build the `release` profile, `-O0` and `-O1` the `debug` one. `-g` adds debug info to a release build without changing its code.

## The build cache

A build compiles the standard library functions the program reaches and keeps the compiled code in the toolchain's cache, so the next build of any package takes it from there. The cache holds at most 5 GiB: past that, a build removes what was used longest ago until 4 GiB is left, and it also removes what no build used for a month. Set `NABI_CACHE_MAX_SIZE` to change the cap (`NABI_CACHE_MAX_SIZE=20G`, or `off` for none). `nabi cache` shows and trims it:

```bash
nabi cache dir                  # where it is
nabi cache prune                # down to the cap now
nabi cache prune --max-size 1G  # keep at most 1 GiB
nabi cache clean                # empty it
```

The toolchain also carries the official packages as a read-only cache, which resolution tries first. `nabi component list` says whether they are there, and `nabi component remove packages` deletes them; `nabi component add packages` downloads them again.

## Looking inside the compiler

To see how long each build stage takes, add `-v`.

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

`--emit` writes intermediate artifacts to `target/debug/build/`. The kinds are `source` (the whole package after meta expansion and desugaring, written as Nabi source), `hir`, `mir` (what the code generator compiles; optimized above `-O0`), `obj` (the object file), `asm` (the object as assembly source, see below), `listing` (the machine code with offsets and bytes, see below), `expansions` (the output of each meta expansion), and `c-header` (the C header of the `#[export]` functions, see [chapter 14](14-ffi.html#shared-libraries-for-c---crate-typecdylib)). `nabi emit` alone writes `mir`.

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

`asm` writes the object as assembly source, like `clang -S`: sections, symbols, each function's instructions with local labels and symbol names where relocations go (`call nabi_panic_at@PLT`), and the data. Assembling it (`clang -c`) gives back the same code, relocations and symbols, so the result links and runs like the object. An instruction no assembler would encode that way from its text is forced (`{disp32}`, `mov.s`) or written as its bytes with the instruction in a comment. Unwind tables are the directives an assembler builds them from (`.cfi_*`; `.seh_*` on x64 Windows; `.fnstart` .. `.fnend` on 32-bit ARM; data on arm64 Windows), and debug info is data. On wasm32 it is LLVM's WebAssembly assembly, which `clang --target=wasm32-wasi -msimd128 -c` assembles:

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

`listing` lists each function's machine code the way `objdump -d` shows it (Intel syntax on x86_64 and x86): offset, bytes and instruction, with the code generator's labels (`.L0:`) and, under an instruction, the relocation that fills it. The bytes are exactly those of `obj`. It works for every target, also when cross-compiling, since emitting does not link:

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

In the object file, and so in a debugger or profiler, functions have mangled symbol names starting with `_Y`. `nabi demangle` turns one into its name (`-v` shows the parts it encodes); with no symbol it filters text, so profiler or `nm` output reads with Nabi names.

```text
$ nabi demangle _Y957555df__square_
square
$ perf report --stdio | nabi demangle
$ nm target/debug/hello | nabi demangle
```

Building with `-g` includes DWARF debug info, so you can step through source lines in `lldb` / `gdb` (Windows builds get the same as CodeView in a PDB, which the Visual Studio debugger and WinDbg read: source lines, parameters, locals and their types). `-g` does not change the code of an optimized build (`-O1` and up): the debugger shows inlined functions as frames of their own, and a variable whose value the optimized code no longer keeps as optimized out.

The toolchain ships formatters for lldb, gdb and natvis (Visual Studio, WinDbg) that show a `str` as its text and a `List`, `Dict`, `Set`, `SortedDict`, `Slice` or `Array` as its size and elements, and (lldb) a sum as its active variant. `nabic --print lldbinit`, `--print gdbinit` and `--print natvis` print what loads them:

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

In cdb or WinDbg, load the natvis file with `.nvload <path>` (the path `nabic --print natvis` prints); in Visual Studio, copy it into `Documents\Visual Studio 2022\Visualizers`.

In WinDbg or cdb, a function the optimizer inlined has no address of its own: `bm hello!sq` sets a breakpoint at every place it was inlined, and `k` lists it as an inline frame. A wasm32 module built with `-g` can be debugged through wasmtime, which turns its debug info into native debug info for gdb or lldb; wasmtime's own optimizations can lose some values, which `-O opt-level=0` keeps.

```text
$ gdb --args wasmtime run -D debug-info -O opt-level=0 hello.wasm
```

## Getting help

`nabi help` lists the commands by group, and `nabi help <command>` (or `nabi <command> --help`) shows one command's usage, examples and options.

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

A misspelled command or option gets the closest name:

```text
$ nabi tset
nabi: unknown command `tset`; did you mean `test`?
run `nabi help` for usage
```

Global options such as `-q`, `-v` and `--color` work before or after the command.

## Summary

- `nabi test` / `nabi bench` build and run `#[test]` / `#[bench]` functions separately. Each test gets its own process, with a time limit. `#[should_panic]` and `#[ignore]` change what a test checks and when it runs. In a library, `nabi test` also runs the ```` ```nabi ```` examples in its docs and README. With `--target`, they run through `wasmtime`, `node` or `qemu-<cpu>`, or `NABI_RUNNER`.
- `nabi fmt` only fixes indentation and blank lines. In CI, use `nabi fmt --check`. Outside a package, give it a path.
- `nabi lint` runs lints, `nabi fix` applies their fixes; `nabi check` goes as far as code generation. Lint levels resolve in the order `nabi.nabion` < `-A/-W/-D` < source attributes.
- From `///` and `//!` doc comments, `nabi doc` generates HTML in `target/doc/`.
- Install the official extension for VS Code, Vim/Neovim or Zed, or register `nabils` as the LSP server in your editor.
- The REPL prompts with `>>>`, keeps the last value in `it`, and has commands such as `:type`, `:doc`, `:vars` and `:reset`. `nabi repl .` loads the current package; for notebooks, `nabi jupyter install`.
- To dig into problems: `-v`, `--emit`, `nabi demangle`, `-g` (with the formatters `nabic --print lldbinit` / `gdbinit` / `natvis` load).
- `nabi help <command>` shows one command's usage and options.
- `NABIR_GC_LOG=1` / `NABIR_GC_STATS=1` show the collector's pauses and their time to safepoint; `bare::gc::stats()` gives the counters in code.

## Further reading

- Spec: [nabic CLI](../spec/refs-lang/009-tools/901-nabic-cli.html), [Standard format](../spec/refs-lang/009-tools/910-nabi-frontend.html#standard-format), [Lint control](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#629-allow--deny--warn-lint-control), [Nabi front-end](../spec/refs-lang/009-tools/910-nabi-frontend.html), [JIT cells and the REPL](../spec/refs-lang/009-tools/907-jit-cells.html), [REPL line editing](../spec/refs-lang/009-tools/907-jit-cells.html#line-editing), [REPL commands](../spec/refs-lang/009-tools/909-debug-tools.html#999-repl--jupyter-debugging), [Jupyter](../spec/refs-lang/009-tools/908-jupyter-integration.html), [Debug tools](../spec/refs-lang/009-tools/909-debug-tools.html), [Diagnostics](../spec/refs-lang/009-tools/909-debug-tools.html#994-diagnostics--error-messages), [Symbol mangling](../spec/refs-lang/009-tools/905-symbol-mangling.html)
- [Error codes](../errors/index.html)
- Loading code at run time: [23. Plugins](23-plugins.html)
- Applying edits while the program runs: [26. Hot Reload](26-hot-reload.html)
- API: [assert functions in core](../std/core/panic.html)
- Next chapter: [17. Conditional Compilation](17-conditional-compilation.html)
