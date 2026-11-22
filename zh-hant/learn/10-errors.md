# 10. 錯誤處理

會失敗的事情到處都是，例如找不到檔案或輸入有誤。Nabi 只有一種錯誤模型。可能擲回錯誤的函式在簽章中寫上 `throws`，呼叫端則用 `try` 標示這個呼叫。因此，錯誤可能流經的每個位置都在程式碼中清楚可見。

## `throws`、`throw` 與 `try`

`throws` 寫在參數列之後、`->` 之前。在函式主體中，`throw` 擲回一個錯誤值。大多數程式碼只需要一則訊息：`throw "msg"`（任何 `str`，f-string 也可以）會擲回標準函式庫的 `ValueError::new("msg")`（除非 `throws` 清單或 `catch e: str` 直接接受 `str`），所以你不必定義錯誤型別。

每次呼叫會擲回錯誤的函式，都必須加上 `try`。呼叫失敗時，錯誤會交給最近的 `catch`；如果沒有，錯誤就會傳播到外層函式之外，而外層函式也必須是 `throws`。

```nabi
fn parse_age(s: str) throws -> i32 {
    let n: i32 = try s.parse();
    if n < 0 { throw f"age {n} is negative"; }
    n
}

fn total_age(xs: List<str>) throws -> i32 {
    let sum = 0;
    for x in xs {
        sum += try parse_age(x);     // an error leaves total_age here
    }
    sum
}

println(f"{try total_age(["30", "12"])}");
let t = try total_age(["30", "-1"]) catch e { println(f"error: {e}"); 0 };
println(f"{t}");
```

```output
42
error: age -1 is negative
0
```

`catch e { .. }` 當場處理錯誤，並以其區塊的值取而代之。`try` 涵蓋它右邊的整個運算式：`try a(b())` 會同時標示兩個呼叫，`try { .. }` 則涵蓋區塊內的所有呼叫。

單純的 `throws` 表示函式可以擲回任何錯誤型別（是 `throws (any Error)` 的簡寫）；`s.parse()` 擲回 `ParseError`，而 `parse_age` 原封不動地把它傳下去。如何明確列出錯誤型別，請見[下文](#自訂錯誤型別)。

### 指令碼頂層的錯誤

程式的頂層陳述式構成一個隱含的 `main`，而它是 `throws` 的。所以在指令碼中，只寫 `try` 就夠了：沒有人攔截的錯誤會在 stderr 印出訊息，並以結束碼 1 結束程式。

```nabi build
let n: i32 = try "abc".parse();
println(f"{n}");
```

```text
$ nabi run
error: `main` returned an uncaught error: cannot parse "abc": invalid digit
  --> src/main.nabi:1:18
$ echo $?
1
```

沒有回傳型別的明確 `fn main()` 也以同樣的方式擲回錯誤，不必寫 `throws`：

```nabi build
fn main() {
    let n: i32 = try "42".parse();
    if n > 40 { throw f"{n} is too big"; }
    println("unreached");
}
```

```text
error: `main` returned an uncaught error: 42 is too big
```

頂層陳述式中單獨的 `return;` 會以結束碼 0 結束指令碼。搭配 `let .. else`，可以在缺少輸入時停止指令碼：

```nabi
let first = ["a", "b"].first() else {
    return;
};
println(first);
```

```output
a
```

在那裡寫 `return v;` 是錯誤（E0018）：要使用其他結束碼，請用 `process::exit(v)`。

## `try?` 與 `??`：用預設值取代錯誤

很多時候你只想要一個備用值。`try? e` 丟棄錯誤並產生 `none`，`??` 則把 `none` 換成預設值。`??` 會結束 `try?`（以及 `try!`）的運算元，所以 `try? f() ?? d` 就是 `(try? f()) ?? d`：發生錯誤時得到 `d`。`try` 仍然涵蓋它右邊的所有內容，包括 `??`。

```nabi
fn parse_port(s: str) -> i32 {
    try? s.parse::<i32>() ?? 8080
}

println(f"{parse_port("3000")} {parse_port("abc")}");
```

```output
3000 8080
```

如果 `f()` 本身回傳 `T | none`，`try? f()` 就是兩層的 `(T | none)?`：外層的 `none` 表示擲回了錯誤，內層的則是 `f` 自己的 `none`，而 `??` 只會去掉外層。希望兩者代表同一件事時，請寫 `try? f() ?? none`。

要刻意忽略錯誤，請寫 `let _ = try? f();`。`try! e` 會把錯誤轉成 panic；只有在那裡失敗就代表程式有 bug 時才使用它。

| 形式     | 發生錯誤時                            |
|----------|---------------------------------------|
| `try e`  | 交給最近的 `catch`，或向外傳播        |
| `try? e` | 丟棄錯誤並產生 `none`                 |
| `try! e` | 觸發 panic（視為程式 bug）            |

```nabi
fn save(n: i32) throws {
    if n < 0 { throw "negative"; }
    println(f"saved {n}");
}

let _ = try? save(1);
let _ = try? save(-1);         // the error is dropped
let n: i32 = try! "42".parse();
println(f"{n}");
```

```output
saved 1
42
```

## `catch`：依型別選擇

`catch` 模式由上而下依序檢查，執行第一個符合的區塊。`e: T` 依型別選擇，在該分支中 `e` 就具有那個型別，因此可以讀取它的欄位。最後的 `catch e` 接住其餘所有錯誤。

```nabi
fn read_age(s: str) throws -> i32 {
    let n: i32 = try s.parse();
    if n < 0 { throw f"age {n} is negative"; }
    n
}

for s in ["30", "x", "-1"] {
    let age = try read_age(s)
    catch e: ValueError { println(f"  {e.message()}"); -1 }
    catch e { println(f"  {e}"); -1 };
    println(f"{s} -> {age}");
}
```

```output
30 -> 30
  cannot parse "x": invalid digit
x -> -1
  age -1 is negative
-1 -> -1
```

## 區塊形式與 `finally`

要一次包住好幾個步驟，請用 `try { .. } catch .. { .. }` 形式。不論控制流程以何種方式離開——正常完成、錯誤、`return` 等——`finally` 區塊都會在最後執行。

```nabi
fn half(n: i32) throws -> i32 {
    if n % 2 != 0 { throw f"{n} is odd"; }
    n / 2
}

for input in [8, 6] {
    let r = try {
        let a = try half(input);
        let b = try half(a);
        a + b
    } catch e {
        println(f"failed: {e}");
        0
    } finally {
        println(f"done with {input}");
    };
    println(f"result {r}");
}
```

```output
done with 8
result 6
failed: 3 is odd
done with 6
result 0
```

從 `finally` 區塊本身跳出的 `return`、`break` 或 `continue` 是錯誤：它會丟掉正在向外傳遞的錯誤。請把跳躍放在 `try` 運算式之後。

## `try for`：步驟可能失敗的迴圈

有些迭代器的每一步都可能失敗，例如會擲回錯誤的產生器，或從串流讀取位元組。這種迭代器是 `TryIterator`，走訪它的 `for` 要寫成 `try for`。這個 `try` 涵蓋被走訪的物件和每一步，但不涵蓋迴圈主體：主體中可能擲回錯誤的呼叫需要自己的 `try`。

```nabi
yield fn numbers(words: List<str>) throws (ParseError) -> i32 {
    for w in words { yield try w.parse::<i32>(); }
}

fn total(words: List<str>) throws -> i32 {
    let sum = 0;
    try for n in numbers(words) {   // a failed step propagates
        sum += n;
    }
    sum
}

println(f"{try! total(["1", "2", "3"])} {try? total(["1", "x"]) ?? -1}");
```

```output
6 -1
```

存在變數中的 `TryIterator` 會就地前進：用 `break` 提前離開的迴圈，會讓變數停在迴圈停下的位置，再用第二個迴圈走訪它時，就從那裡繼續。`try! for` 在步驟失敗時觸發 panic，`try? for` 則在那裡結束迴圈。要略過錯誤繼續進行，請走訪 `it.results()`，它把每一步都以 `Result` 的形式給出。

## 自訂錯誤型別

呼叫端需要區分不同的失敗時，就定義一個錯誤型別：一個具有 `Debug` 和 `Display`、並實作 `Error` trait 的結構或列舉。`#[derive(Debug, Error)]` 提供其中兩者，`#[display("..")]` 則從一個由欄位組成的 f-string 寫出 `Display`；手寫的 `impl Display` 形式請見[第 6 章](06-structs-enums-traits.html#運算子-trait)。這樣一來，簽章就能明確列出它擲回的型別 `throws (E)`，或以和型別列出好幾種 `throws (A | B)`。編譯器會檢查每個列出的型別都有被處理或傳下去。

```nabi
#[derive(Debug, Error)]
#[display("no key {self.key}")]
struct NotFound { key: str; }

#[derive(Debug, Error)]
#[display("{self.reason}")]
struct Invalid { reason: str; }

fn lookup(key: str) throws (NotFound | Invalid) -> i32 {
    if key == "" { throw Invalid { reason: "empty key" }; }
    if key != "a" { throw NotFound { key: key }; }
    1
}

fn describe(key: str) -> str {
    try f"found {lookup(key)}"
    catch e: NotFound { f"missing: {e.key}" }
    catch e: Invalid { f"invalid: {e}" }
}

fn main() {
    for k in ["a", "b", ""] {
        println(describe(k));
    }
}
```

```output
found 1
missing: b
invalid: empty key
```

`lookup` 的錯誤種類封閉為 `NotFound | Invalid`，所以涵蓋這兩種型別的 `describe` 就成為不再擲回錯誤的函式。編譯器會檢查這一點。當錯誤種類的集合難以封閉時，例如在公開 API 中，請用單純的 `throws`；接收端用 `catch e` 接住所有錯誤，或先收窄型別，例如 `catch e: ValueError`（[4.14.3](../spec/refs-lang/004-functions/414-throws.html#4143-untyped-throws)）。

`catch` 後面可以接任何模式，所以結構模式可以直接取出欄位。

```nabi
#[derive(Debug, Error)]
#[display("no key {self.key}")]
struct NotFound { key: str; }

fn lookup(key: str) throws (NotFound) -> i32 {
    if key != "a" { throw NotFound { key: key }; }
    1
}

for key in ["a", "b"] {
    let n = try lookup(key)
    catch NotFound { key } { println(f"missing: {key}"); 0 };
    println(f"{n}");
}
```

```output
1
missing: b
0
```

## `rethrows`：只在引數會擲回時才擲回

接受函式作為引數的高階函式可以宣告為 `rethrows`。如果傳入的函式不會擲回錯誤，呼叫時就不需要 `try`；只有在傳入會擲回錯誤的函式時才加上 `try`。

```nabi
fn apply_all(xs: List<i32>, f: fn(i32) throws -> i32) rethrows -> List<i32> {
    let out = [];
    for x in xs { out.push(try f(x)); }
    out
}

fn half(n: i32) throws -> i32 {
    if n % 2 != 0 { throw f"{n} is odd"; }
    n / 2
}

let doubled = apply_all([1, 2, 3], |x| x * 2);
println(f"{doubled:?}");
let halves = try apply_all([2, 4], half) catch _ { [] };
println(f"{halves:?}");
let bad = try apply_all([2, 3], half) catch e { println(f"{e}"); [] };
println(f"{bad:?}");
```

```output
[2, 4, 6]
[1, 2]
3 is odd
[]
```

## `using`：清理資源

GC 會回收記憶體，但檔案、socket 和鎖這類資源必須在確定的時間點關閉。`using x = acquire_expr { .. }` 不論區塊以何種方式離開（正常結束、經由 `return`，或因錯誤傳播），都會呼叫 `x` 的 `Drop::drop`。多個資源以逗號分隔，並以相反的順序清理。

```nabi
struct Conn { name: str; }

impl Drop for Conn {
    fn drop(ref self) { println(f"close {self.name}"); }
}

fn open(name: str) -> Conn {
    println(f"open {name}");
    Conn { name: name }
}

fn work() throws -> i32 {
    using a = open("db"), b = open("cache") {
        println("working");
        throw "query failed";
    }
}

try work() catch e { println(f"work failed: {e}"); };
```

```output
open db
open cache
working
close cache
close db
work failed: query failed
```

`using` 是運算式，所以也能產生值，例如 `let text = using f = try fs::open(path) { try f.read_to_end() };`。

當資源應該存活到外層區塊結束時，陳述式形式 `using x = e;` 可以省去多餘的區塊。它相當於套在其後所有陳述式上的區塊形式，所以釋放發生在區塊結束時，以相反的順序進行，不論從哪條路徑離開都一樣：

```nabi
struct Conn { name: str; }

impl Drop for Conn {
    fn drop(ref self) { println(f"close {self.name}"); }
}

fn report() {
    using a = Conn { name: "db" };
    using b = Conn { name: "cache" };
    println(f"using {a.name} and {b.name}");
}

report();
```

```output
using db and cache
close cache
close db
```

型別實作了 `Drop` 的值只有一個擁有者。`let b = a` 會移動它，之後再使用 `a` 就是錯誤；`a.clone()` 會產生第二個值，它有自己的擁有者。不使用 `using` 時，編譯器會在擁有者離開作用域的地方釋放該值，不論從哪條路徑離開都一樣：

```nabi
struct Conn { name: str; }

impl Drop for Conn {
    fn drop(ref self) { println(f"close {self.name}"); }
}

fn short_job() {
    let c = Conn { name: "tmp" };
    let d = c;               // moves: `c` is gone
    println("working");
}                            // the compiler releases `d` here

short_job();
```

```output
working
close tmp
```

關閉時可能失敗的型別，例如 `File`，不會自動替你關閉，因為它的錯誤會遺失：請用 `try f.close()` 關閉它，或用 `using` 繫結它。在它仍然開啟的狀態下離開作用域是編譯錯誤。基於同樣的理由，`try?` 不能丟棄可能持有這種值的錯誤，例如 `tx.send(conn)` 失敗時的 `SendError<TcpStream>`，它會把串流交還給你：請用 `try .. catch e { .. }` 攔截它，取出值並關閉。

## 錯誤與 panic

`throws` 產生的錯誤是「可能發生的失敗」。相對地，索引超出範圍、除以零、`try!` 失敗或呼叫 `panic(..)` 則是 **panic**，視為程式的 bug。panic 無法用 `catch` 攔截；它會在 stderr 寫出訊息，並以結束碼 101 結束。`finally` 和 `using` 中的清理也不會執行。

```nabi build
let xs = [1, 2, 3];
let i = 7;
println(f"{xs[i]}");
```

```text
thread 'main' panicked at main.nabi:3:12:
index 7 out of bounds for length 3
```

要了解程式是怎麼走到那裡的，請用 `NABI_BACKTRACE=1` 執行：panic 也會列出呼叫堆疊上的函式，最內層的排在最前面（`NABI_BACKTRACE=full` 會再加上它們的位址）。不需要 `-g`，但被最佳化器內嵌的函式不會有自己的一行。

```text
stack backtrace:
   0: core::index::index_oob
   1: core::collections::list::<impl>::index
   2: main
note: some details are omitted, run with `NABI_BACKTRACE=full` for a verbose backtrace.
```

只有在需要把錯誤當成值保存時（收集多個結果，或稍後再處理），才使用 `Result<T, E>`。規則是：「要擲回就用 `throws`，要保存就用 `Result`」。

```nabi
fn half(n: i32) throws -> i32 {
    if n % 2 != 0 { throw f"{n} is odd"; }
    n / 2
}

for n in [4, 5] {
    let r = Result::catching(|| try half(n));
    println(f"{n}: ok={r.is_ok()} value={r.unwrap_or(-1)}");
}
```

```output
4: ok=true value=2
5: ok=false value=-1
```

## 小結

- 可能擲回錯誤的函式宣告 `throws`（在 `->` 之前），並用 `throw` 擲回；`throw "msg"` 擲回 `ValueError`，足以應付大多數需求。`fn main()` 不必宣告就能擲回錯誤。
- 呼叫以 `try` 標示。未被攔截的錯誤會傳播到外層函式；在指令碼頂層，它會以結束碼 1 結束程式。
- `try? e ?? default` 提供備用值；`let _ = try? f();` 忽略錯誤；`try!` 把錯誤轉成 panic。單獨的 `return;` 結束指令碼。
- `catch` 是依模式選擇的運算式，`finally` 一定會執行。
- 呼叫端必須區分失敗時，定義自己的錯誤型別（`Display` + `Error`），並用 `throws (A | B)` 列出。
- `rethrows` 只在收到的函式會擲回時才擲回。
- 以 `using` 繫結的資源不論區塊以何種方式離開都會被清理；`using x = e;` 涵蓋其所在區塊的其餘部分。其他 `Drop` 值在擁有者離開作用域時釋放；`File` 必須關閉，或以 `using` 繫結。
- `try for` 走訪步驟可能失敗的迭代器；迴圈主體要自行標示 `try`。
- panic 是無法攔截的 bug 訊號。只有在需要把錯誤當成值保存時，才使用 `Result`。

## 延伸閱讀

- 規格：[錯誤控制流程](../spec/refs-lang/003-expressions/314-error-control-flow.html)、[`throws` 簽章](../spec/refs-lang/004-functions/414-throws.html)、[`using` 運算式](../spec/refs-lang/003-expressions/317-using-expression.html)、[panic 與致命錯誤](../spec/refs-lang/003-expressions/314-error-control-flow.html#3148-panics-and-fatal-errors)
- 標準函式庫規格：[error](../spec/refs-standard-libraries/200-core/214-error.html)
- API：[`Error`](../../std/core/error.html#trait.Error)、[`ValueError`](../../std/core/error.html#struct.ValueError)、[`Result`](../../std/core/error.html#enum.Result)、[`Drop`](../../std/core/marker.html#trait.Drop)
- 下一章：[11. 模組與套件](11-modules-packages.html)
