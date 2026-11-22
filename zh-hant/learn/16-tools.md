# 16. 工具

本章介紹撰寫程式碼時隨手會用到的工具：格式化工具、lint 工具、測試與基準測試、文件產生、編輯器支援（LSP）、REPL 與 Jupyter，以及檢視編譯器內部的偵錯選項。這些工具全都從同一個 `nabi` 指令開始。

本章的輸出都來自實際執行下面這個套件。它以 `nabi new mathx` 建立，並把 `src/main.nabi` 改成如下內容：

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

## `nabi test`：測試

標上 `#[test]` 的函式就是測試。測試函式不會進入一般建置；`nabi test` 會收集它們並另外建置。檢查使用 `assert`、`assert_eq` 和 `assert_ne`，失敗時會觸發 panic。最後一個引數可以傳入一段訊息。

每個測試都在自己的行程中執行，所以即使其中一個觸發 panic，其餘的也會繼續執行。上面套件中的 `fib_wrong` 是刻意寫錯的。

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

只要有任何測試失敗，結束碼就是 1，所以可以直接用在 CI 中。把名稱的一部分（或使用 `^`、`$`、`.`、`*`、`+`、`?` 的簡單正規表示式）當作引數傳入，就只會執行符合的測試。

```text
$ nabi test gcd
   Compiling mathx (.)
    Finished in 0.37s

running 1 test
test gcd_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 2 filtered out; finished in 0.22s
```

用 `--show-output` 可以連通過的測試的輸出也一併顯示，用 `--threads=N` 可以設定同時執行多少個測試。給 CI 儀表板和其他工具使用時，`--format json` 會在 stdout 上每行寫一個 JSON 物件，每個測試一行，最後再加一行摘要，並把給人看的文字移到 stderr：

```text
$ nabi test --format json 2>/dev/null
{"schema":1,"type":"case","kind":"test","name":"fib_works","result":"ok","ms":87.057}
{"schema":1,"type":"case","kind":"test","name":"fib_wrong","result":"failed","ms":87.144,"reason":"panicked (exit 101)"}
{"schema":1,"type":"case","kind":"test","name":"gcd_works","result":"ok","ms":87.05}
{"schema":1,"type":"summary","kind":"test","result":"failed","passed":2,"failed":1,"ignored":0,"filtered_out":0,"ms":87.276}
```

`--exact` 讓篩選條件只比對完整名稱，`--skip <pat>` 則排除符合的測試（要略過好幾個時可以重複指定）。`--list` 只印出名稱，不執行任何測試。

### 預期的 panic、忽略的測試與逾時

應該觸發 panic 的測試以 `#[should_panic]` 標明；加上 `expected` 時，panic 訊息還必須包含該段文字。很慢的測試，或需要 CI 沒有的東西的測試，可以加上 `#[ignore]`，並可附上原因：

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

`nabi test --ignored` 只執行被忽略的測試，`--include-ignored` 則把它們和其他測試一起執行。卡住的測試不會讓整次執行卡住：600 秒後它會被終止並回報為失敗。用 `--timeout=SECS`（或 `NABI_TEST_TIMEOUT`）設定其他上限；`--timeout=0` 會關閉逾時。

### 文件測試

在函式庫中，`nabi test` 也會執行文件裡的程式碼範例：`///` 與 `//!` 註解以及 `README.md` 中的每個 ```` ```nabi ```` 區塊，都會建置成使用該函式庫的小程式並執行。函式庫的公開項目可以直接以名稱使用，就像寫了 `use mathx::*;` 一樣。範例不再能編譯，或其中的 `assert` 失敗時，整次測試就會失敗，因此文件始終保持正確。

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

以 `# ` 開頭的行會執行，但在 `nabi doc` 中隱藏，所以範例可以做準備工作而不顯示出來。圍欄中 `nabi` 後面的字會改變檢查的內容：`nabi,norun` 只建置範例（用於讀取檔案或等待網路的程式碼），`nabi,should_panic` 預期會觸發 panic，`nabi,compile_fail` 預期範例無法建置（要指定特定錯誤時寫 `compile_fail,E0040`），`nabi,ignore` 則略過它。其他語言（```` ```text ````、```` ```nabion ````）一律不執行。

```text
running 2 tests
test doc README.md:9 ... ok
test doc src/lib.nabi:14 (gcd) ... ok
```

每個文件測試以其檔案與行號命名，再加上它所說明的項目。範例內的錯誤會指向你原始檔中的那一行。`nabi test --doc` 只執行文件測試，`--no-doc` 則略過它們。

## `nabi bench`：基準測試

`#[bench]` 函式接收一個 `Bench`，並把要計時的程式碼交給 `b.iter`。`nabi bench` 以最佳化（`-O2`）建置它們，多次呼叫每個閉包，並印出每次呼叫時間的中位數及其離散程度。從未呼叫 `b.iter` 的基準測試會收到警告。數字會因機器與每次執行而不同。

```text
$ nabi bench
   Compiling mathx (.)
    Finished in 0.42s

benchmarking 1 bench
  fib_20      12.99 us +/- 120.52 ns

bench result: ok. 1 run; 0 failed; 0 filtered out; finished in 2.63s
```

`--iters=N` 是每個樣本的呼叫次數（預設為足以花費約 100 ms 的次數），`--samples=N` 是樣本數（預設 10）。篩選條件的用法與 `nabi test` 相同。在終端機和管線中，輸出看起來都一樣；讀取結果的工具可以傳入 `--format=nabibench`，每個基準測試得到一行 `#nabibench`，或傳入 `--format json`，得到與 `nabi test` 相同的 JSON 行，每筆都帶有 `median_ns`、`mad_ns` 和 `samples`。

## 垃圾回收器的暫停

垃圾回收器會以短暫的暫停讓所有執行緒停下。有兩個環境變數可以顯示這些暫停：`NABIR_GC_LOG=1` 會在每次回收時於 stderr 印出一行，`NABIR_GC_STATS=1` 則在程式結束時印出摘要。

```text
$ NABIR_GC_LOG=1 ./target/release/server
nabi gc: #41 minor: 0.802 ms pause, 21046 objects marked, 141312 -> 104160 KiB; ttsp 9 us, mark 410 us, sweep 352 us, evac 0 us, threads 3, at 1520.114 ms
```

暫停從回收器要求執行緒停下的那一刻開始計算。`ttsp`（time to safepoint，抵達安全點的時間）是等待它們停下所花的部分：`ttsp` 很長，代表某個執行緒一直在執行，而不是回收器很慢。長字串的搜尋與計數、UTF-8 檢查和大量複製都不會拖住暫停；它們會讓回收器在旁同時執行。`NABIR_GC_STATS=1` 的摘要最後，每種暫停各有一行（minor、full，以及並行標記的完整回收的開始與結束），列出其 p50、p90、p99 與最長值。

程式可以用 `bare::gc::stats()` 讀取同樣的計數器，供指標端點或影格預算使用。它會短暫取得回收器的鎖，所以偶爾讀一次即可，不要在熱迴圈中讀取。

```nabi
use bare::gc;

let s = gc::stats();
println(f"{s.collections} collections, longest pause {s.pause_max_ns / 1000} us");
```

`bare::gc::collect()` 會執行一次完整回收，並在整個過程中讓所有執行緒停下：在影格之間呼叫它來「清理」，只會讓每個影格更長。要讓暫停不落在影格中，請使用 `collect_young`、暫停目標、`no_collect` 與 arena（[第 25 章](25-gc-pauses.html)）。

## 其他目標

搭配 `--target` 時，`nabi test`、`nabi bench` 和 `nabi run` 會透過執行器（runner）在這台機器上執行建置出的程式：`wasm32-wasip2` 和 `wasm32-wasip1-threads` 使用 `wasmtime`，`wasm32-unknown-unknown` 使用 `node`，在 Linux 上執行其他 CPU 的 Linux 程式則使用 `qemu-<cpu>`（qemu-user）。這些都必須在 `PATH` 上；找不到時，錯誤訊息會說明要安裝什麼。機器本身就能執行的程式不需要執行器：x86-64 Linux 上的 `i686-unknown-linux-gnu`，以及 arm64 Linux 上的 `armv7-unknown-linux-gnueabihf`，前提是 CPU 也能執行 32 位元 ARM 程式碼（Neoverse N1 和 Cortex-A72 可以；Neoverse N2 和 Apple silicon 不行），並且已安裝 armhf 的 C 函式庫（Debian、Ubuntu：先 `dpkg --add-architecture armhf`，再 `apt install libc6:armhf`）；`armv7-unknown-linux-musleabihf` 的程式是靜態的，在那裡不需要 C 函式庫。

```text
$ nabi test --target wasm32-wasip2 gcd
   Compiling mathx (.)
    Finished in 0.61s

running 1 test
test gcd_works ... ok

test result: ok. 1 passed; 0 failed; 2 filtered out; finished in 0.05s
```

要使用其他執行器，或替沒有執行器的目標指定一個，請把 `NABI_RUNNER` 設為該指令；程式與它的引數會接在後面：

```text
$ NABI_RUNNER="qemu-aarch64 -L /opt/aarch64-sysroot" nabi test --target aarch64-unknown-linux-gnu
```

在裝有 Xcode 的 Mac 上，`aarch64-apple-ios-sim` 的程式會在已啟動的 iOS 模擬器上執行（`NABI_SIM_DEVICE` 可指定其他裝置）；請見[第 28 章](28-ios.html)。

WASI 程式沒有工作目錄：相對路徑會從 `/` 開始查找，所以在 wasm 上執行的測試請使用絕對路徑。

### Android

`x86_64-linux-android` 為 Android 10 以上建置程式與共用函式庫，可在 Android 模擬器（x86_64 系統映像檔）中執行。給 arm64 手機與平板的建置（`aarch64-linux-android`）仍在進行中：該目標尚未在實體裝置上測試，會在之後的版本推出。連結需要 Android NDK，可以從 Android Studio 的 SDK Manager（SDK Tools、NDK）或 NDK 下載頁面取得；`nabi` 會在 Android SDK 中找到它，或者把 `ANDROID_NDK_HOME` 設為它的位置：

```text
$ export ANDROID_NDK_HOME=~/Android/Sdk/ndk/30.0.16248370
$ nabi build --target x86_64-linux-android --release
$ adb push target/x86_64-linux-android/release/hello /data/local/tmp/
$ adb shell /data/local/tmp/hello
Hello, world!
```

給 app 的 Java 或 Kotlin 程式碼使用的函式庫是一個 `cdylib`，其匯出的函式採用 JNI 的命名；把 `.so` 放在 app 的 `jniLibs/x86_64/` 下（裝置建置推出後則是 `arm64-v8a/`），再以 `System.loadLibrary` 載入：

```nabi
use bare::ptr::CPtr;

#[export_name("Java_com_example_app_Native_add")]
pub extern "C" fn add(env: CPtr<c_void>, class: CPtr<c_void>, a: i32, b: i32) -> i32 { a + b }
```

```text
$ nabi build --target x86_64-linux-android --release --crate-type=cdylib
```

程式可在 Android 10（API level 29）以上執行；`ANDROID_PLATFORM=34` 會為較新的版本建置。程式與函式庫可以在 4 KB 或 16 KB 分頁的裝置上載入，這正是 Google Play 對 Android 15 以上 app 的要求。由於核心是 Linux，Linux 程式大多可以照原樣執行。不同之處請見[平台說明](../spec/refs-lang/000-meta/006-platform-targets.html#android)。

## `nabi fmt`：格式化工具

`nabi fmt` 會把套件中的 `.nabi` 與 `.nabion` 檔案改寫成標準格式。Nabi 的格式化工具刻意保守：它**不動語彙單元、行內間距和註解**，只修正縮排（每層括號 4 個空格）、行尾空白，以及連續的空行。它唯一會移動的是模組開頭的 `mod` / `use` 行：依 `pub mod`、`mod`、`pub use`、`use` 分組，各組排序，組與組之間空一行。因此格式化工具永遠不會改變程式碼的意義，也不會破壞有人手動排好的對齊。

格式化工具讀取運算子語彙單元的方式與編譯器相同：包括套件本身的 `#[infix]` 語彙單元，以及其相依套件所匯出的語彙單元（取自 `nabi.lock`，不會下載任何東西），所以使用 `a ⊛ b` 或 `x |> f` 的測試、基準測試和相依於它的套件也都能格式化。

用 `--check` 檢查一個縮排混亂、混用 tab 且有行尾空白的 `fib`，檔案不會被改動，會顯示將要變更的行，並以結束碼 1 結束。這個模式很適合用在 CI。

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

不加 `--check` 時，它會改寫檔案並印出改了幾個；`-q` 會省略那一行。在編輯器中設定每次存檔時執行，會很方便。

在套件內，從任何子目錄執行 `nabi fmt` 都會格式化整個套件，並略過 `target/`、隱藏目錄、`node_modules/` 以及 `.gitignore` 中列出的路徑。在套件外，則要指明格式化什麼：一個檔案（`nabi fmt hi.nabi`）或一個目錄（`nabi fmt scripts/`）。沒有指定路徑時，它會停下來並說明原因：

```text
$ nabi fmt
nabi: error: not inside a package; pass a file or directory: `nabi fmt <path>`
```

## `nabi lint` 與 `nabi check`

`nabi lint` 不會編譯；它只執行型別檢查與 lint。它會抓到上面留下的 `let unused = 1;`。

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

每個 lint 的預設等級都是 `warn`，但有例外：`nan_comparison` 和 `self_assignment` 一定是錯誤，預設為 `deny`；`missing_docs`（沒有 `///` 註解的 `pub` 項目）則是 `allow`，直到你開啟它為止。等級可以在三個地方變更，後面的優先：

1. `nabi.nabion` 中的 `lints` 欄位：`lints: { "unused": "deny", "dead_code": "allow" }`
2. 命令列上的 `-A` / `-W` / `-D`（allow / warn / deny）
3. 原始碼中的屬性：項目上的 `#[allow(dead_code)]`、檔案開頭的 `#![deny(unused)]`

除了單一 lint（`unused_variables`），也可以指定一個群組（`unused`、`nonstandard_style`、`warnings`、`all`）。只要有任何 `deny` lint 觸發，結束碼就是 1。

`unused_result` 會對丟棄值的陳述式發出警告，例如 `xs.len();` 或 `parse(s);`，因為這通常是錯誤。如果你本來就是要丟棄它，請寫 `_ = expr;`。呼叫端經常用不到結果的函式，例如 `Dict::insert`、`List::pop` 和 `Atomic::fetch_add`，都標有 `#[discardable]`，呼叫它們的陳述式不會產生警告；你自己的函式也可以這樣標記。

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

要讓 CI 在有警告時失敗，請把 `warnings` 設為 deny（`lints: { "warnings": "deny" }` 或 `-D warnings`）。與 rustc 一樣，它涵蓋所有警告等級的 lint，包括之後的版本新增的 lint，所以當新版本不能改變 CI 接受的內容時，請在 `nabi.nabion` 中固定確切的工具鏈版本（`toolchain: "=0.1.0"`）。`all` 是所有 lint。在 `nabi.nabion` 中設定 `"warnings": "deny"` 時，最後一則 note 會指出等級的來源：

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

有些檢查結果會附帶解決它的修改：未使用的 `let` 會加上開頭的 `_`，未使用的 `use` 行會被刪除，`while true` 變成 `loop`，`x == true` 變成 `x`，多餘的 `as` 或 `.clone()` 會被移除，含有 `{name}` 的一般字串會加上 `f`。你的編輯器會把每一項都當作快速修正提供；`nabi fix` 則會在整個套件中套用所有修正，並反覆檢查直到沒有剩下的為止：

```text
$ nabi fix
       Fixed src/main.nabi (2 fixes)
```

`nabi fix` 只會修改你的套件中的檔案，而且在工作樹有未提交的變更時會拒絕執行，所以每項修正都會出現在 `git diff` 中，也都能復原（`--allow-dirty` 會略過這項檢查）。`nabi fix --diff` 只印出修正而不寫入。lint 等級同樣適用：`nabi fix -A unused_imports` 不會動到匯入。遇到編譯錯誤時，它會在做任何修改之前停下；請先手動修正那個錯誤。

`deprecated` 會對任何使用標有 `#[deprecated]` 的項目之處發出警告：函式、方法、欄位、型別、常數或靜態變數，不論來自你的程式碼、相依套件或標準函式庫；編輯器會以刪除線顯示這些使用處。完整清單請用 `nabi lint --list` 查看，它也會列出每個 lint 是在哪個版本加入的（把它導向 `head` 也沒問題：當讀取輸出的一方離開時，`nabi` 和 `nabic` 會安靜地停止，而不是回報錯誤）。`nabi check` 更進一步，也會執行程式碼產生，但不會連結或產生執行檔。它比建置快，所以要回答「能不能編譯？」時就用它。從這裡開始的範例使用一個用完即丟的 `hello` 套件，讓錯誤和底層輸出不會干擾 `mathx`。

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

`nabi build` 和 `nabi run` 會印出相同的診斷，建置失敗時最後會以計數取代 `Finished`：

```text
$ nabi build
   Compiling hello (.)
...
error: could not build hello (1 error, 0 warnings)
```

要取得機器可讀的輸出，請加上 `--message-format=json`。每則診斷都會印成一行 JSON（`"schema": 1`）：包含它的錯誤碼與種類、訊息、帶有起訖行號與欄號的範圍、`notes` 與 `help` 行、它建議的 `fixes`，以及 lint 警告的 `lint` 名稱。

方括號中的錯誤碼是錯誤的名稱，每種錯誤一個錯誤碼（所有 lint 共用 `E0131`，並在 note 中寫出自己的名稱）。`nabic --explain E0040`（或 `nabi --explain E0040`）會印出該錯誤碼的意義與修正方法，[錯誤碼索引](../errors/index.html)則列出所有錯誤碼。如果 nabic 本身當掉（編譯器內部錯誤），它會以結束碼 70 結束，與觸發 panic 的程式的 101 區分開來；請回報這個問題。

## `nabi doc`：文件註解與 API 文件

`///` 說明緊接在它下方的項目，`//!` 則說明包含該註解的模組（檔案）。在上面的套件中，第一行 `//! Small math helpers.` 會成為模組的說明，`gcd` 和 `fib` 上方的 `///` 則成為函式的說明。一般的 `//` 註解永遠不是文件，即使位在檔案開頭也一樣：當函式庫的根檔案或 `pub` 模組的檔案以 `//` 行開頭卻沒有 `//!` 時，`nabi lint` 會發出警告（`plain_module_doc`），help 會顯示應該寫的 `//!` 行。文件文字可以使用 markdown，例如標題、清單、行內程式碼、程式碼區塊和連結。其中的 ```` ```nabi ```` 範例會由 `nabi test` 測試（請見[文件測試](#文件測試)）。

```text
$ nabi doc
 Documenting mathx (.)
    Finished in 0.10s
   Generated ./target/doc/mathx/index.html
```

`nabi doc --open` 還會在瀏覽器中開啟該頁面。

`target/doc/` 會產生套件及其相依套件的公開（`pub`）項目的 HTML，外加一份搜尋索引：套件本身是 `target/doc/<package>/index.html`，每個公開模組各一頁（`<package>/geo.html`、`<package>/geo/shapes.html`）。每頁的側邊欄會列出套件的相依套件，並連結到它們的頁面，搜尋則能找到所有已產生文件的套件中的項目。透過 `pub use` 從私有模組引入的項目，會顯示在重新匯出它的頁面上，就像是在那裡宣告的一樣；重新匯出某個公開模組已顯示的項目時，則會連結到該模組的頁面。這些頁面的外觀與本站的[標準函式庫 API](../../std/core/index.html) 相同，也遵循相同的規則，而且可以離線使用，直接從磁碟開啟即可。

## 編輯器：`nabils`

`nabils` 是 LSP（Language Server Protocol）伺服器。它提供診斷、懸停提示、跳到定義、尋找參照、重新命名、文件符號、自動完成、簽章說明、快速修正和格式化。自動完成會提供作用域中的區域變數、值的成員（`xs.` 會列出 `List` 的公開方法），以及 `use` 後面的路徑。懸停提示和跳到定義也能深入標準函式庫。快速修正會套用診斷建議的修正、補上缺少的 `;` 或 `use`，並為缺少的套件執行 `nabi add`。重新命名會拒絕函式庫項目、關鍵字和來自其他檔案的名稱，並說明原因。診斷跟著你的輸入走，而不是跟著存檔：每次編輯都會在背景重新檢查，在這期間，懸停提示、自動完成和跳到定義會立即以上一次成功的檢查結果回應，所以從網頁貼上的字元（零寬度空格、用 `−` 代替 `-`、彎引號）、單獨的組合符號、格式錯誤的數字、錯誤的跳脫序列，或沒有任何處理器接受的字面值後綴（已註冊的是 `hz` 時寫成 `10Hz`），都會立刻被標示出來，並附上編譯器本身的訊息；修正方式很明確時，還會提供寫入修正的快速修正。每個檔案都以它所屬套件的相依套件來檢查，所以包含多個套件的資料夾也能正常運作，而透過快速修正新增、或編輯並儲存 `nabi.nabion` 而新增的相依套件，不必重新啟動伺服器就能找到。`nabi lsp` 是同一個伺服器。它透過標準 IO 通訊，所以任何支援 LSP 的編輯器只需要知道要執行的指令（編輯器自行加上的 `--stdio` 也能接受）。檔案有型別錯誤時，懸停提示仍會繼續回應：依據該次檢查已推得的型別，否則依據上一次通過的檢查。在終端機中手動啟動時，`nabils` 只會說明它應由編輯器啟動。

Neovim（0.11 以上）請在 `init.lua` 中加入：

```text
vim.filetype.add({ extension = { nabi = "nabi", nabion = "nabion" } })
vim.lsp.config("nabils", {
  cmd = { "nabils" },
  filetypes = { "nabi", "nabion" },
  root_markers = { "nabi.nabion" },
})
vim.lsp.enable("nabils")
```

Helix 請在 `languages.toml` 中加入：

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

在 VS Code 等其他編輯器中，請用通用的 LSP 用戶端擴充功能註冊 `nabils`。`nabils` 必須在你的 `PATH` 上。

## REPL 與 Jupyter

進一步看看第 1 章介紹過的 REPL（`nabi` 或 `nabi repl`）。在終端機中，它啟動時會顯示兩行：工具鏈的版本與主機，以及一行提示。`nabi repl -q` 會省略這兩行，從管線輸入時則一律不顯示。

```text
$ nabi repl
Nabi 0.1.0 on aarch64-apple-darwin
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

提示符號是 `>>>`，儲存格尚未結束時則是 `...`。儲存格在行尾結束，除非輸入顯然還沒完成：括號、字串或區塊註解仍未關閉，該行以運算子、`.`、`,` 或 `=` 結尾，或者是像 `fn f(x: i32)` 這樣仍在等待主體的項目開頭。當儲存格的最後一個陳述式是 `let` 或指派時，可以省略它的 `;`。

每個儲存格只執行一次。頂層的 `let` 繫結以及 `fn` / `struct` / `impl` 定義會延續到之後的儲存格，儲存格也可以在頂層使用 `await`。運算式儲存格會以 `Display` 印出它的值，沒有就用 `Debug`，再沒有就只印出它的型別。在結尾加上 `?` 則改為印出 `Debug` 形式。`let` 儲存格什麼都不顯示。由多個陳述式組成的儲存格，若以運算式結尾，會顯示最後那個運算式的值。儲存格顯示的值會保存為 `it`，讓下一個儲存格可以使用；一旦你自己定義了 `it`，就以你的為準。

與一般原始碼不同，你可以再次 `let` 同一個名稱，甚至換成另一個型別。再次定義 `fn` 或 `struct` 會取代原本的定義，先前儲存格中的程式碼會呼叫新版本。編譯失敗、觸發 panic 或擲回錯誤的儲存格，會讓工作階段保持原樣。它已經寫出的輸出則會保留。`:reset` 會從空的工作階段重新開始。

錯誤會以 `<repl>:line:col` 指向你輸入的儲存格。panic 以及先前儲存格中 `fn` 的錯誤，則以 `<repl cell N>:line:col` 指出那個儲存格，其中 N 是到目前為止輸入的儲存格數。在 Jupyter 中，N 是筆記本儲存格的執行次數，而一個拆成多個 REPL 儲存格的筆記本儲存格，會依序編號為 `N.1`、`N.2` 等。

```text
>>> fn boom(n: i32) {
...     panic(f"boom {n}");
... }
>>> boom(5)
thread 'main' panicked at <repl cell 1>:2:5:
boom 5
error: the cell panicked; the session is unchanged
```

| 指令 | 作用 |
|------|---------|
| `expr?` | 印出 `Debug` 形式，而不是 `Display` |
| `:type expr`（`:t`） | 印出型別而不執行 |
| `:doc name` | 印出某個名稱的簽章與文件註解；方法用 `:doc xs.len` |
| `:time expr` | 執行運算式，並印出每個階段花費的時間 |
| `:vars` | 以 `name: type` 列出繫結，接著列出項目 |
| `:load file`、`:reload` | 把程式載入工作階段；重新載入上一個。由 `:save` 產生的指令碼會逐一重播每個儲存格 |
| `:edit` | 在 `$VISUAL` / `$EDITOR` 中撰寫儲存格，然後執行 |
| `:save file` | 把工作階段寫成指令碼，一個個標記好的儲存格依序排列 |
| `:opt [level]` | 顯示或設定之後儲存格的最佳化等級（`0` 到 `3`、`s`） |
| `:reset` | 忘掉所有儲存格並重新開始 |
| `:add name ..` | 新增相依套件：在環境中執行 `nabi install`，在套件中則執行 `nabi add`。儲存格使用的官方套件不需要這個指令就會自動加入 |
| `:help`（`:h`） | 列出所有指令 |
| `:exit`、`:quit`（`:q`） | 結束；單獨輸入 `exit` 或 `quit`，或輸入結束時也一樣 |

擷取了變數的閉包無法保存在儲存格頂層的 `let` 中，因為每個閉包都有自己的無名型別。這樣的儲存格會被拒絕：

```text
>>> let x = 3
>>> let f = |n: i64| n * x
error: `f: closure(i64) -> i64` cannot be kept for later cells (its type has no name)
```

請為該繫結指定函式型別，`let f: fn(i64) -> i64 = |n: i64| n * x`，這樣就能延續下去。（`fn` 也能延續，但它看不到 `x` 這類儲存格變數；請把它們當作引數傳入。）在儲存格內部使用的閉包，例如 `xs.map(|n| n * x)`，則照常運作。

在終端機中，REPL 自己負責行編輯：

- Tab 會補全名稱：作用域中的區域變數、工作階段的繫結與項目、core 函式庫、關鍵字，在 `x.` 之後是 `x` 的公開欄位與方法，在 `a::` 之後則是模組 `a` 的項目。在行首按 Tab 則是縮排。
- 歷史記錄會保存完整的儲存格，並跨工作階段保留（`$NABI_HOME/repl_history`）。上、下方向鍵在其中移動；已輸入部分文字時，只會在以該文字開頭的儲存格之間切換。Ctrl-R 會向前搜尋。
- 貼上的區塊在按下 Enter 時會成為一個儲存格，而不是逐行執行。
- 輸入會像本指南一樣上色。設定 `NO_COLOR` 可以關閉顏色。

讀取標準輸入的儲存格會從終端機讀取。當 REPL 本身的輸入來自管線時，儲存格看到的是空的輸入。

`nabi repl path` 會把一個套件或檔案載入工作階段。若是函式庫，該套件會被加為相依套件，並先執行 `use name::*;`，`:reset` 之後也會再執行一次。在開發函式庫時用來試用函式很方便。若是程式，它會執行該程式，然後把程式的繫結與項目保留在工作階段中。在套件內，`nabi repl .` 會載入目前的套件；單純的 `nabi repl` 則從空的工作階段開始。

同一套儲存格引擎也可以在 Jupyter 筆記本中使用。只要註冊一次核心，就能在 Jupyter、VS Code 筆記本等環境中選擇 `nabi` 核心。

```text
$ nabi jupyter install
   Installed kernel `nabi` (/Users/me/Library/Jupyter/kernels/nabi/kernel.json)
```

它在 macOS 上安裝於 `~/Library/Jupyter`，在 Windows 上安裝於 `%APPDATA%\jupyter`，在其他系統上則安裝於 `$XDG_DATA_HOME/jupyter`（預設為 `~/.local/share/jupyter`）；`JUPYTER_DATA_DIR` 可以變更位置。`nabi jupyter install --env name` 會為具名的環境安裝核心。

筆記本儲存格遵循 REPL 的規則，並顯示最後一個運算式的值。型別實作了 `RichDisplay` 的值，例如圖表的 figure，會依其 `mime_bundle()` 提供的內容顯示（SVG 圖片、HTML），`display(x)` 則可以在儲存格中途顯示一個值。輸出會在儲存格執行期間陸續出現，而不是只在結束時才出現。讀取標準輸入的儲存格會在筆記本中得到一個輸入框。Tab 補全和 Shift-Tab 說明使用與 REPL 相同的名稱。在筆記本前端認得 Nabi 之前，程式碼會以 Rust 的規則標示語法。

## 最佳化等級

`nabi build` 建置 `debug` 設定檔，`--release` 建置 `release` 設定檔。`nabi` 和 `nabic` 也可以直接指定等級：

| 等級 | 用途 |
|-------|----------------|
| `-O0` | 預設值：不做最佳化，建置最快，每個值都在偵錯工具預期的位置 |
| `-O1` | 執行速度快的偵錯建置：迴圈得到與 `-O2` 相同的處理（走訪串列的 `for` 會變成計數迴圈，可以時加以向量化），其他地方很少內嵌，建置也保留偵錯資訊 |
| `-O2` | `--release`：快速的程式碼 |
| `-O3` | 以較多程式碼換取多一點速度：迴圈中內嵌得更多，純量迴圈（浮點數加總、主體中有呼叫）會被展開 |
| `-Os` | 小的程式碼：只內嵌小函式，迴圈仍是計數迴圈，較短時會向量化 |

`-O2`、`-O3` 和 `-Os` 建置 `release` 設定檔，`-O0` 和 `-O1` 建置 `debug` 設定檔。`-g` 會為 release 建置加入偵錯資訊，而不改變它的程式碼。

## 建置快取

建置會編譯程式用到的標準函式庫函式，並把編譯好的程式碼保存在工具鏈的快取中，之後任何套件的建置都會從那裡取用。快取最多保存 5 GiB：超過時，建置會移除最久以前使用的內容，直到剩下 4 GiB 為止，也會移除一個月內沒有任何建置使用過的內容。設定 `NABI_CACHE_MAX_SIZE` 可以變更上限（`NABI_CACHE_MAX_SIZE=20G`，或用 `off` 表示不設上限）。`nabi cache` 可以顯示並修剪快取：

```bash
nabi cache dir                  # 位置
nabi cache prune                # 現在就縮減到上限
nabi cache prune --max-size 1G  # 最多只保留 1 GiB
nabi cache clean                # 清空
```

工具鏈也以唯讀快取的形式附帶官方套件，解析相依套件時會先嘗試它。`nabi component list` 會說明它們是否存在，`nabi component remove packages` 會刪除它們；`nabi component add packages` 則會重新下載。

## 檢視編譯器內部

要查看每個建置階段花費多久，請加上 `-v`。

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

`--emit` 會把中間產物寫到 `target/debug/build/`。種類有 `source`（經過 meta 展開與語法糖去除後的整個套件，以 Nabi 原始碼寫出）、`hir`、`mir`（程式碼產生器所編譯的內容；`-O0` 以上為最佳化後的結果）、`obj`（目的檔）、`asm`（以組合語言原始碼表示的目的檔，見下文）、`listing`（帶有偏移量與位元組的機器碼，見下文），以及 `expansions`（每次 meta 展開的輸出）。單獨執行 `nabi emit` 會寫出 `mir`。

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

`asm` 會像 `clang -S` 一樣，把目的檔寫成組合語言原始碼：區段、符號、每個函式的指令（附有區域標籤，並在重定位處寫出符號名稱，例如 `call nabi_panic_at@PLT`），以及資料。把它組譯（`clang -c`）會得到相同的程式碼、重定位與符號，所以結果可以像原本的目的檔一樣連結與執行。組譯器從文字不會以那種方式編碼的指令，會被強制指定（`{disp32}`、`mov.s`），或寫成位元組並在註解中附上指令。回溯表（unwind table）以組譯器據以建立它們的指示詞表示（`.cfi_*`；x64 Windows 上是 `.seh_*`；32 位元 ARM 上是 `.fnstart` .. `.fnend`；arm64 Windows 上則是資料），偵錯資訊則是資料。在 wasm32 上，它是 LLVM 的 WebAssembly 組合語言，可以用 `clang --target=wasm32-wasi -msimd128 -c` 組譯：

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

`listing` 會像 `objdump -d` 那樣列出每個函式的機器碼（在 x86_64 和 x86 上使用 Intel 語法）：偏移量、位元組和指令，附有程式碼產生器的標籤（`.L0:`），並在指令下方列出填入它的重定位。這些位元組與 `obj` 的完全相同。由於輸出時不做連結，它適用於所有目標，交叉編譯時也一樣：

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

在目的檔中，因而在偵錯工具或效能分析工具中，函式都帶有以 `_Y` 開頭的修飾後符號名稱。`nabi demangle` 會把它還原成原本的名稱（`-v` 會顯示它所編碼的各個部分）；不給符號時，它會過濾文字，讓效能分析工具或 `nm` 的輸出以 Nabi 的名稱呈現。

```text
$ nabi demangle _Y957555df__square_
square
$ perf report --stdio | nabi demangle
$ nm target/debug/hello | nabi demangle
```

以 `-g` 建置會包含 DWARF 偵錯資訊，所以你可以在 `lldb` / `gdb` 中逐行執行原始碼（Windows 建置則以 PDB 中的 CodeView 取得相同的資訊，供 Visual Studio 偵錯工具與 WinDbg 讀取：原始碼行、參數、區域變數及其型別）。`-g` 不會改變最佳化建置（`-O1` 以上）的程式碼：偵錯工具會把內嵌的函式顯示為各自的框架，最佳化後的程式碼不再保存其值的變數則顯示為 optimized out。

工具鏈附帶給 lldb、gdb 和 natvis（Visual Studio、WinDbg）使用的格式化器，把 `str` 顯示為它的文字，把 `List`、`Dict`、`Set`、`SortedDict`、`Slice` 或 `Array` 顯示為大小與元素，並（在 lldb 中）把和型別顯示為目前作用中的變體。`nabic --print lldbinit`、`--print gdbinit` 和 `--print natvis` 會印出載入它們所需的內容：

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

在 cdb 或 WinDbg 中，以 `.nvload <path>` 載入 natvis 檔案（路徑即 `nabic --print natvis` 印出的路徑）；在 Visual Studio 中，則把它複製到 `Documents\Visual Studio 2022\Visualizers`。

在 WinDbg 或 cdb 中，被最佳化器內嵌的函式沒有自己的位址：`bm hello!sq` 會在它被內嵌的每個位置設定中斷點，`k` 則把它列為內嵌框架。以 `-g` 建置的 wasm32 模組可以透過 wasmtime 偵錯，wasmtime 會把它的偵錯資訊轉換成 gdb 或 lldb 可用的原生偵錯資訊；wasmtime 自身的最佳化可能會讓某些值遺失，`-O opt-level=0` 可以保留它們。

```text
$ gdb --args wasmtime run -D debug-info -O opt-level=0 hello.wasm
```

## 取得說明

`nabi help` 會依群組列出指令，`nabi help <command>`（或 `nabi <command> --help`）則顯示單一指令的用法、範例與選項。

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

拼錯的指令或選項會得到最接近的名稱：

```text
$ nabi tset
nabi: unknown command `tset`; did you mean `test`?
run `nabi help` for usage
```

`-q`、`-v` 和 `--color` 等全域選項放在指令之前或之後都可以。

## 小結

- `nabi test` / `nabi bench` 會另外建置並執行 `#[test]` / `#[bench]` 函式。每個測試都有自己的行程，並有時間限制。`#[should_panic]` 和 `#[ignore]` 會改變測試檢查的內容與執行的時機。在函式庫中，`nabi test` 也會執行文件與 README 中的 ```` ```nabi ```` 範例。搭配 `--target` 時，它們會透過 `wasmtime`、`node` 或 `qemu-<cpu>`，或 `NABI_RUNNER` 執行。
- `nabi fmt` 只修正縮排和空行。在 CI 中請使用 `nabi fmt --check`。在套件外請給它一個路徑。
- `nabi lint` 執行 lint，`nabi fix` 套用它們的修正；`nabi check` 會一路做到程式碼產生。lint 等級的解析順序為 `nabi.nabion` < `-A/-W/-D` < 原始碼屬性。
- `nabi doc` 從 `///` 和 `//!` 文件註解在 `target/doc/` 中產生 HTML。
- 在編輯器中把 `nabils` 註冊為 LSP 伺服器。
- REPL 以 `>>>` 提示，把最後的值保存在 `it` 中，並有 `:type`、`:doc`、`:vars` 和 `:reset` 等指令。`nabi repl .` 會載入目前的套件；筆記本則用 `nabi jupyter install`。
- 要深入調查問題：`-v`、`--emit`、`nabi demangle`、`-g`（搭配 `nabic --print lldbinit` / `gdbinit` / `natvis` 所載入的格式化器）。
- `nabi help <command>` 會顯示單一指令的用法與選項。
- `NABIR_GC_LOG=1` / `NABIR_GC_STATS=1` 會顯示回收器的暫停及其抵達安全點的時間；`bare::gc::stats()` 則在程式碼中提供這些計數器。

## 延伸閱讀

- 規格：[nabic CLI](../spec/refs-lang/009-tools/901-nabic-cli.html)、[標準格式](../spec/refs-lang/009-tools/910-nabi-frontend.html#標準格式)、[lint 控制](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#629-allow--deny--warn-lint-控制)、[Nabi 前端](../spec/refs-lang/009-tools/910-nabi-frontend.html)、[JIT 儲存格與 REPL](../spec/refs-lang/009-tools/907-jit-cells.html)、[REPL 行編輯](../spec/refs-lang/009-tools/907-jit-cells.html#行編輯)、[REPL 指令](../spec/refs-lang/009-tools/909-debug-tools.html#999-repl--jupyter-偵錯)、[Jupyter](../spec/refs-lang/009-tools/908-jupyter-integration.html)、[偵錯工具](../spec/refs-lang/009-tools/909-debug-tools.html)、[診斷](../spec/refs-lang/009-tools/909-debug-tools.html#994-診斷--錯誤訊息)、[符號修飾](../spec/refs-lang/009-tools/905-symbol-mangling.html)
- [錯誤碼](../errors/index.html)
- 在執行時載入程式碼：[23. 外掛](23-plugins.html)
- 在程式執行期間套用修改：[26. 熱重載](26-hot-reload.html)
- API：[core 中的 assert 函式](../../std/core/panic.html)
- 下一章：[17. 條件式編譯](17-conditional-compilation.html)
