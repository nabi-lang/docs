# 17. 條件式編譯

同一份原始碼樹常常需要建置成好幾種樣子：依作業系統、debug 或 release、有沒有某個選用功能。Nabi 以 `#[cfg(..)]` 在編譯時決定這些。條件為假的程式碼在剖析後立刻被移除，就像從來沒寫過一樣，所以它在執行時不會有任何成本，甚至可以參照在該次建置中不存在的名稱。

## 條件

條件是一個旗標、一組 `key = "value"`，或是以 `all(..)`、`any(..)` 和 `not(..)` 組合起來的條件。

| 條件                            | 何時為真                                               |
|---------------------------------|--------------------------------------------------------|
| `target_os = "linux"`           | 為 Linux 建置時（`"macos"`、`"windows"`、`"wasi"`，瀏覽器為 `"unknown"` 等） |
| `target_arch = "aarch64"`       | 為該 CPU 建置時（`"x86_64"` 等）                       |
| `target_family = "unix"`        | 類 Unix 目標（`"windows"`、`"wasm"`）                  |
| `target_pointer_width = "64"`   | 64 位元目標                                            |
| `feature = "tls"`               | 套件的功能特性 `tls` 已啟用（[第 18 章](18-features-and-publishing.html)） |
| `debug_assertions`              | 以 `-O0` 建置時（`nabi run` 的預設值）                  |
| `test`                          | 為 `nabi test` 建置時                                   |

`target_*` 依循 `--target`，所以交叉建置看到的是目標的值，而不是主機的值。

## `#[cfg]` 可以放在哪裡

`#[cfg]` 可以附加在項目上（函式、型別、`mod`、`use`、`const`、`static`、`impl` 區塊及其成員），也可以附加在程式碼內部較小的部分：陳述式、`match` 分支、結構欄位、結構字面值的欄位，以及函式參數。

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

以 `-O2` 建置時，同一個檔案會印出 `release build of demo` 和 `unsupported`：`checks` 欄位、它的初始化和 `Trace` 分支全都消失了。

條件為假的參數會從簽章中移除，呼叫時只傳入其餘的引數。這讓選用功能可以新增參數，而不必再寫一份函式。

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

閉包參數不能加屬性。

## 兩份定義，一次建置

要讓函式在不同平台有不同的主體，就用互斥的條件把它寫兩次。任何一次建置中都只會留下一份；如果兩份都留下，就是一般的重複定義錯誤。

```nabi
#[cfg(target_family = "windows")]
fn path_sep() -> str { "\\" }

#[cfg(not(target_family = "windows"))]
fn path_sep() -> str { "/" }

println(["usr", "local", "bin"].join(path_sep()));
```

在 Linux 和 macOS 上，這會印出 `usr/local/bin`。

整個模組也可以是條件式的。`#[cfg(target_os = "linux")] mod linux;` 在其他建置中連 `linux.nabi` 都不會讀取。

編譯器會記住條件移除了什麼。呼叫只存在於另一種建置中的東西，會得到一個錯誤，告訴你它存在於哪裡，以及如何保護這個呼叫：

```text
error[E0046]: no method `pidfd` found for type `Child`
   = note: `pidfd` is only available on linux (`#[cfg(target_os = "linux")]`)
   = help: put the code that uses it under the same `#[cfg(target_os = "linux")]`
```

## std 中與作業系統相關的部分

標準函式庫也以同樣的方式拆分。只有部分系統才有的東西放在 `std::os::unix`、`os::linux` 和 `os::windows` 中，每個模組只存在於它的目標上。它們的擴充 trait 在那些目標上位於預載模組中，所以在 Linux 和 macOS 上，`Command::new("sh").process_group(0)` 不需要 `use`。在 Windows 上，同樣的呼叫是錯誤，錯誤會告訴你這個方法存在於哪裡：

```text
error[E0046]: no method `process_group` found for type `Command`
   = note: `process_group` is only available on unix (`#[cfg(target_family = "unix")]`)
   = help: put the code that uses it under the same `#[cfg(target_family = "unix")]`
```

所以使用它的程式碼要放在相同的條件下：

```nabi
use std::process::Command;

#[cfg(target_family = "unix")]
fn detached(cmd: Command) -> Command { cmd.process_group(0) }

#[cfg(not(target_family = "unix"))]
fn detached(cmd: Command) -> Command { cmd }
```

給 C 用的原始作業系統控制代碼也放在那裡。`as_raw_fd()`（在 Windows 上是 `as_raw_handle()` / `as_raw_socket()`）會借用 `File`、socket 或子行程管道的描述符：原本的值仍然擁有它，並負責關閉它。`into_raw_fd()`（`into_raw_handle()` / `into_raw_socket()`）則會交出描述符：原本的值被消耗，不再關閉它，所以現在要由你關閉。`unsafe` 的 `from_raw_fd(fd)` 會建立一個重新擁有 `fd` 的值。

## `cfg!`：編譯時的 `bool`

當兩個分支在任何地方都能編譯、只有選擇不同時，`cfg!(..)` 寫起來比較短。它是常數 `bool`，所以編譯器會丟掉不會執行的分支；與 `#[cfg]` 不同的是，兩邊仍然都必須通過型別檢查。

```nabi
const DEBUG: bool = cfg!(debug_assertions);

let level = if cfg!(feature = "verbose") { 2 } else { 1 };
println(f"debug={DEBUG} level={level}");
```

```output
debug=true level=1
```

`#[cfg_attr(cond, attr)]` 只在條件成立時附加另一個屬性，例如 `#[cfg_attr(test, derive(Debug))]`。

## 小結

- `#[cfg(..)]` 在編譯時保留或移除程式碼。條件有 `key = "value"`、旗標（`test`、`debug_assertions`）以及 `all` / `any` / `not`。
- 它可以附加在項目、欄位、變體、陳述式、match 分支、結構字面值欄位和參數上。被移除的參數會從簽章中消失。
- 為各平台寫不同主體時，使用互斥的定義。
- `cfg!(..)` 是給一般 `if` 用的常數 `bool`。
- 與作業系統相關的 std 位於 `std::os::{unix, linux, windows}`；它們的擴充 trait 在各自的目標上位於預載模組中。
- `feature = ".."` 條件由套件的功能特性設定，這是下一章的主題。

## 延伸閱讀

- 規格：[6.3 `#[cfg]`](../spec/refs-lang/006-meta-system/603-cfg.html)、[6.3.3 附加位置](../spec/refs-lang/006-meta-system/603-cfg.html#633-附加位置)、[6.3.5 `cfg!`](../spec/refs-lang/006-meta-system/603-cfg.html#635-cfg-編譯期的-bool-值)、[平台目標](../spec/refs-lang/000-meta/006-platform-targets.html)
- 下一章：[18. 功能特性與發布](18-features-and-publishing.html)
