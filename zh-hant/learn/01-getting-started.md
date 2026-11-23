# 1. 入門

本章會安裝 nabi 工具鏈，建立並執行你的第一個套件，再試試單檔指令碼和 REPL。讀完之後，你就能自己執行本指南其餘部分的範例。

## 安裝

### 事前準備

`nabic` 是自給自足的：它有自己的程式碼產生器和連結器，所以不需要另外安裝 C 編譯器或連結器（[9.2.18](../spec/refs-lang/009-tools/902-aot-pipeline.html#9218-自給自足的工具鏈)）。

- macOS：Xcode Command Line Tools（`xcode-select --install`），用於最後連結時需要的 SDK。
- Linux：不需要任何東西。工具鏈和它建置的程式需要 glibc 2.34 以上（Ubuntu 22.04、Debian 12、RHEL 9 及更新版本）。
- Windows：不需要任何東西。
- FreeBSD：不需要任何東西。工具鏈和它建置的程式需要 FreeBSD 14 以上。

發行封存檔提供 macOS arm64、Linux x86_64 與 arm64（glibc 和靜態 musl 兩種）、Windows x86_64 與 arm64，以及 FreeBSD x86_64 版本。其他主機沒有發行封存檔，請從原始碼建置 Nabi。

### 安裝指令碼

安裝指令碼會從 [GitHub releases](https://github.com/nabi-lang/nabi/releases) 下載適合你平台的工具鏈，用該版本的 `SHA256SUMS` 驗證，再解壓縮到 `~/.nabi`。如果 `PATH` 上有 `minisign`，它會先用發行金鑰驗證 `SHA256SUMS.minisig`；沒有的話，`SHA256SUMS` 只經由 HTTPS 取得，指令碼會說明如何手動驗證簽章。任何一項不符都不會安裝。

```bash
curl -fsSL https://nabi.sh | sh
```

在 Windows 上，請在 PowerShell 中執行下面的指令。它會安裝到 `%USERPROFILE%\.nabi`，並把其中的 `bin` 加入你的使用者 `PATH`。

```powershell
powershell -ExecutionPolicy Bypass -c "irm https://nabi.sh/install.ps1 | iex"
```

`install.sh --version <v>` 和 `install.ps1 -Version <v>` 會以確切版本（`0.1.0`）選擇一個發行版本；不加的話會安裝最新的版本。

安裝位置可以用環境變數 `NABI_HOME` 變更（預設為 `~/.nabi`）。指令碼執行完後，會得到下面的目錄結構。

```text
~/.nabi/
├── bin/                        # nabi, nabic, nabicc, nabils proxy
└── toolchains/
    └── 0.1.3-aarch64-apple-darwin/
        ├── bin/{nabi, nabic, nabicc, nabils}
        └── lib/                # runtime and standard library sources
```

剛安裝的工具鏈會成為預設工具鏈。`~/.nabi/bin` 裡的四個執行檔是**代理程式**（proxy）：實際的工作由目前使用中的工具鏈裡同名的執行檔完成。這樣你就能並存安裝多個工具鏈，並從中選擇一個（和 rustup 的做法相同）。代理程式永遠複製自最新安裝的工具鏈，它知道每個舊版工具鏈的結構，所以把舊版工具鏈設為預設時，代理程式維持不變。

指令碼也會把 `~/.nabi/bin` 加入你的 `PATH`。它會寫入 `~/.nabi/env`（以及 `~/.nabi/env.fish`），從 shell 的啟動檔（例如 `~/.zshenv`、`~/.bashrc` 或 `~/.profile`）載入它，並為 fish 新增 `~/.config/fish/conf.d/nabi.fish`。它會列出每一個修改過的檔案：

```text
 Downloading nabi 0.1.3 (aarch64-apple-darwin)
   Installed nabi 0.1.3 to ~/.nabi

    Modified ~/.zshenv (PATH)

A new shell finds nabi on PATH; to use it in this one, run:
  . "$HOME/.nabi/env"            (sh, bash, zsh)
  source "$HOME/.nabi/env.fish"  (fish)

Next steps:
  nabi --version
  nabi new hello && cd hello && nabi run
```

再執行一次指令碼不會改變任何東西。若要讓 shell 檔案保持原樣，請傳入 `--no-modify-path`（`install.ps1` 用 `-NoModifyPath`）或設定 `NABI_NO_MODIFY_PATH=1`；這時指令碼會印出你需要自己加入 `PATH` 的那一行。工具鏈附帶官方套件（`json`、`http`、`regex`、`table` 等），所以使用它們的程式碼不需要網路就能建置；若不想安裝，請傳入 `--no-packages`（`-NoPackages`，或 `NABI_NO_PACKAGES=1`）。`nabi` 的 shell 自動補全也會一併安裝（bash、zsh、fish、PowerShell）。如果想安裝到其他地方，`nabi completions <shell>` 會印出該指令碼。

在新的 shell 中確認版本。

```bash
nabi --version
nabi toolchain list
```

`nabi --version -v` 還會列出實際執行的二進位檔、它選擇的工具鏈和原因、它的 `nabic`，以及主機，每項一行 `key: value`（`nabi version -v --format json` 會把它們印成一個 JSON 物件）。版本和你預期的不同時，可以用它來查看。

```text
$ nabi --version -v
nabi 0.1.3 (<commit> <date>)
binary:    /Users/me/.nabi/toolchains/0.1.3-aarch64-apple-darwin/bin/nabi
toolchain: 0.1.3-aarch64-apple-darwin (the default)
nabic:     /Users/me/.nabi/toolchains/0.1.3-aarch64-apple-darwin/bin/nabic
host:      aarch64-apple-darwin
```

會安裝四個執行檔。

| 執行檔 | 用途 |
|------------|--------------|
| `nabic`     | 編譯器。建置、檢查、測試和基準測試框架（[9.1](../spec/refs-lang/009-tools/901-nabic-cli.html)） |
| `nabi`      | 套件／建置前端與工具。`nabi.nabion`、相依套件、環境、執行、lint、格式化、文件、REPL、Jupyter（[9.10](../spec/refs-lang/009-tools/910-nabi-frontend.html)） |
| `nabicc`    | C 編譯器，也可用 `nabi cc` 執行（[9.13](../spec/refs-lang/009-tools/913-c-compiler.html)、[第 14 章](14-ffi.html#c-原始檔nabi-cc)） |
| `nabils`    | 語言伺服器（LSP） |

### 管理工具鏈

`nabi toolchain` 管理已安裝的工具鏈。使用中的工具鏈依下列順序決定：命令列上的 `+version`（`nabi +0.1 build`）、環境變數 `NABI_TOOLCHAIN`、最近的 `nabi.nabion`（或指令碼的 `//! nabion` 區塊）中的 `toolchain` 欄位，最後是預設工具鏈。固定的版本可以是一個系列（`"0.1"`，即已安裝的最新 0.1.x 版本；若尚未安裝任何 0.1 正式版本，則為其最新的候選版本）、一個版本（`"0.1.0"`）或一個版本需求（`"^0.1.0"`）。一個指令從頭到尾都在同一個工具鏈上執行：即使以完整路徑執行某個工具鏈自己的 `bin/nabi`，它也會把指令交給專案所固定的工具鏈，而每個工具鏈的 `nabi` 只驅動它自己的 `nabic`。

```bash
nabi toolchain list                     # installed toolchains
nabi toolchain default 0.1.1            # change the default
nabi toolchain install 0.1.1            # download 0.1.1 beside the default
nabi toolchain update                   # download the newest release and make it the default
nabi toolchain update 0.1.1             # exactly 0.1.1 instead
```

`nabi toolchain update` 接受確切版本，而不是系列。`nabi toolchain install` 只下載而不變更預設工具鏈（`--from <archive>` 會安裝你已經有的檔案）。下載的檔案會以公布的 SHA-256 驗證，任何步驟失敗時，原本的預設工具鏈都會保留。舊版工具鏈不會被刪除。更新跨到新的系列時，會附上一則提示：只有最新的系列會收到修正，所以請把專案移到新系列。在專案移過去之前，可以固定舊的系列，例如 `toolchain: "0.1"` 固定在 0.1，或執行 `nabi +0.1 ..`。請不要和 `nabi update` 混淆，後者是更新套件 `nabi.lock` 中固定的版本（[第 11 章](11-modules-packages.html)）。

同一系列中較新的工具鏈，能建置較舊工具鏈能建置的東西。1.0 之前，系列是 `0.y`：0.1.7 能建置 0.1.3 建置的東西，0.2.0 則不一定。從 1.0 起系列是 `x`：1.9.0 能建置 1.4.2 建置的東西，2.0.0 則不一定。套件的 `nabi_version` 也遵循這個規則：`"0.1.3"` 表示 nabi 0.1.3 或更新的 0.1.x。

如果 `PATH` 上有另一個 `nabi`（例如來自套件管理器）排在 `~/.nabi/bin` 之前，新的預設工具鏈就不會生效，所以 `nabi toolchain update` 和 `nabi toolchain default` 會發出警告：

```text
nabi: warning: your shell runs /opt/homebrew/bin/nabi (nabi 0.1.3) before ~/.nabi/bin/nabi; put ~/.nabi/bin first on PATH or remove the other one
```

### 從原始碼建置（選用）

你也可以自己建置工具鏈封存檔，再用同一個指令碼安裝。這需要 Rust 工具鏈（`rustup`）。

```bash
git clone https://github.com/nabi-lang/nabi && cd nabi
scripts/dist/native.sh                     # dist/nabi-<version>-<triple>.tar.gz
sh install.sh --archive dist/nabi-*.tar.gz  # into ~/.nabi
```

指令碼也會用 `targets/build.sh` 建置每個目標的執行環境和 C 函式庫，這需要 `curl`、`python3` 以及支援 xz 的 `tar`，下載的檔案會放在 `~/.cache/nabi`（可用 `NABI_CACHE` 變更位置）。在 Linux 上，`scripts/dist/gnu.sh`（podman 或 docker）會在容器中建置同樣的封存檔，讓它能在 glibc 2.34 以上執行；發行封存檔就是這樣建置的。詳細說明請見儲存庫的 README。

如果你在開發編譯器，可以不安裝而直接使用建置輸出。`cargo build` 會產生執行檔和主機的執行環境。`targets/build.sh <triple>` 會建置某個目標的 C 函式庫和執行環境檔案：Linux 主機需要自己的這些檔案才能連結程式，每個交叉編譯目標也都需要；在 macOS 和 Windows 上，主機不需要其他東西。

```bash
cargo build --release
targets/build.sh "$(rustc -vV | sed -n 's/^host: //p')"
export PATH="$PWD/target/release:$PATH"
```

## 你的第一個套件

`nabi new` 會建立一個套件目錄，並初始化一個 git 儲存庫，附上排除 `target/` 的 `.gitignore`。套件名稱使用小寫字母、數字、`-` 和 `_`，必須以字母開頭，最長 64 個字元。Windows 保留給裝置的名稱（`con`、`nul`、`com1` 等）會被拒絕，因為在 Windows 上無法用這些名稱命名檔案。`nabi` 這個名稱已被標準函式庫使用，所以套件不能命名為 `nabi`。在程式碼中，`-` 會變成 `_`（`nabi new my-lib` 以 `use my_lib::..` 匯入）。

```bash
nabi new hello
cd hello
```

```text
     Created executable package (hello)
             cd hello && nabi run
```

第二行是接下來要執行的指令。套件裡有兩個檔案。清單 `nabi.nabion` 記錄套件名稱、版本、能建置它的最舊 nabi 版本（`nabi_version`）、建立時使用的工具鏈系列，以及相依套件。

```nabion
Package { name: "hello", version: "0.1.0", nabi_version: "0.1.3", toolchain: "0.1", deps: {} }
```

原始碼放在 `src/` 中；`src/main.nabi` 只有一行，會印出一句問候。

```nabi
println("Hello, world!");
```

```output
Hello, world!
```

`nabi run` 會建置並執行它。輸出放在 `target/debug/`。建置時只會編譯程式用到的標準函式庫部分，工具鏈會保留這些結果，所以之後建置任何套件時都會從快取中取用。

```bash
nabi run
```

```text
   Compiling hello (.)
    Finished in 0.87s
     Running `target/debug/hello`
Hello, world!
```

`src/main.nabi` 中的頂層陳述式由上而下執行，構成整個程式。函式和型別宣告可以放在頂層陳述式之間的任何位置。其他常用的指令有 `nabi build`（只建置）、`nabi check`（只做型別檢查）、`nabi test` 和 `nabi fmt`。

## 稍大一點的程式

把 `src/main.nabi` 改成下面的內容，再執行一次 `nabi run`。

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

`f"..."` 是 f-string：`{}` 中的每個運算式都會被求值，並插入到該位置。`greet` 宣告在使用處的下方，呼叫依然有效，因為函式等宣告不論順序都看得到。

`println` 也能接受多個值，並以空格分隔印出；`sep:` 和 `end:` 可以改變值之間和最後印出的內容；偵錯時，`dbg(e)` 會把運算式連同位置和值印到 stderr：

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

除了頂層陳述式，你也可以用 `fn main()` 作為進入點。兩者擇一使用。

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

## 單檔指令碼

快速實驗不需要套件。把檔案路徑交給 `nabi run`，它就會單獨執行那個檔案。`nabi build hi.nabi` 會在指令碼旁邊產生執行檔 `hi`。

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

指令碼也能使用相依套件：`nabi add json hi.nabi` 會把它們寫進檔案開頭的 `//!` 註解（請見[內嵌清單](../spec/refs-lang/009-tools/910-nabi-frontend.html#9107-單一檔案腳本的行內資訊清單)和[第 11 章](11-modules-packages.html#單檔指令碼與環境)）。

## REPL

不帶引數執行 `nabi` 會開啟 REPL。輸入一個運算式，它就會印出值；以 `?` 結尾則改印 `Debug` 形式。沒有 `Display` 的值只會印出它的型別，而 `let` 只會定義名稱。

```text
$ nabi
Nabi 0.1.3 (<commit> <date>) on aarch64-apple-darwin
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

`:type expr` 會回報運算式的型別而不執行它。閱讀本指南時，只要對某個型別有疑問，就可以在 REPL 中確認。一個儲存格最後的 `let` 不需要 `;`。其餘內容請見[第 16 章](16-tools.html#repl-與-jupyter)（[JIT 儲存格](../spec/refs-lang/009-tools/907-jit-cells.html)）。

## 小結

- 用 `curl -fsSL https://nabi.sh | sh` 安裝到 `~/.nabi`（Windows 上用 `install.ps1`）。指令碼會把 `~/.nabi/bin` 加入 `PATH`，並安裝 shell 自動補全。
- `nabi new name` 會建立 `nabi.nabion` 和 `src/main.nabi`；`nabi run` 會建置並執行。
- `src/main.nabi` 的頂層陳述式就是程式。也可以使用 `fn main()`。
- `nabi run file.nabi` 執行指令碼；單獨執行 `nabi` 會啟動 REPL。

## 延伸閱讀

- [安裝結構](../spec/refs-lang/009-tools/902-aot-pipeline.html#9219-安裝配置)、[固定工具鏈](../spec/refs-lang/009-tools/910-nabi-frontend.html#9106-toolchain)
- [`nabi` 前端](../spec/refs-lang/009-tools/910-nabi-frontend.html)、[`nabic` CLI](../spec/refs-lang/009-tools/901-nabic-cli.html)
- [進入點](../spec/refs-lang/004-functions/416-entry-point.html)
- [JIT 儲存格與 REPL](../spec/refs-lang/009-tools/907-jit-cells.html)
- 下一章：[2. 值與型別](02-values-and-types.html)
