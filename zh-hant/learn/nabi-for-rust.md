# 給 Rust 使用者的 Nabi

這是寫給 Rust 程式設計師的對照圖：哪些東西可以直接沿用、哪些習慣要放下，以及 Cargo 的各種工具在 Nabi 裡對應到哪裡。Nabi 的表面語法有很多地方和 Rust 相似，但記憶體模型不同：記憶體由垃圾回收管理，所以沒有生命週期（lifetime），也沒有借用檢查器（borrow checker）。下面每個區塊都是完整的程式，用 `nabi run file.nabi` 執行即可。本指南的各章會完整說明每個部分。

## 可以沿用的部分

- `fn`、`struct`、`enum`、`trait`、`impl Trait for Type`、帶約束和 `where` 子句的泛型、關聯型別與常數泛型。
- 帶守衛條件的 `match`、`if let`、`while let`、`let else`，以及能解構元組、結構和列舉變體的模式。
- `#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]`、`#[repr(C)]`、`#[cfg(..)]`、`#[test]` 和 `#[inline]`。
- `mod`、`use a::b::{c, d}`、`pub`、`pub use` 重新匯出，以及對應 `crate::` 的套件相對路徑（`pkg::`）。
- 整數型別 `i8` .. `i64`、`u8` .. `u64`、`isize`、`usize`、`f32`、`f64`；字面值後綴（`7u8`、`2.5f32`）、`_` 分隔符、`as` 強制轉型、`checked_add`、`wrapping_add`、`saturating_add`。
- 運算子優先順序、範圍 `a..b` 和 `a..=b`、向零截斷的 `/` 和 `%`，以及溢位時觸發 panic。
- 迭代器鏈（`map`、`filter`、`fold`、`zip`、`enumerate`、`windows`）、閉包 `|x| x + 1`、`async fn` 和 `await`、`unsafe` 區塊和 `extern "C"`。
- 一個能建置、執行、測試、格式化、lint 和產生文件的套件管理工具：`nabi` 扮演 Cargo 的角色。

## 首先要放下的習慣

- 沒有生命週期，沒有 `&` 或 `&mut`，也沒有借用檢查器。記憶體由 GC 釋放；只要還有東西參考某個值，它就會一直存在。
- 沒有 `mut`。每個變數都可以重新指派，而對已有的名稱再寫一次 `let` 會宣告一個新的變數，就像 Rust 的遮蔽（shadowing）。
- 結構、列舉和元組都是值，指派時會複製，不需要額外選用任何東西。`List`、`Dict` 和 `Set` 是指向同一個共用物件的控制代碼。`ref` 讓函式能修改呼叫端的結構。
- 錯誤不是你回傳的值。可能失敗的函式會標明 `throws`，每次呼叫它都要寫 `try`。錯誤沒有 `?` 可用。
- 沒有 `Option`。可能缺失的值的型別是 `T | none`，`T?` 是它的簡寫。
- 沒有後綴的整數字面值會從它在函式中的用法取得型別，和 Rust 一樣；沒有任何東西決定它的型別時則是 `i64`。數字永遠不會隱含轉換，連 `i32` 轉 `i64` 也不會。
- 只有帶 `Drop` 的型別（檔案、socket、通道的傳送端）才有單一擁有者並會移動。關閉可能失敗的檔案要明確關閉，用 `using` 或 `try f.close()`。
- 陳述式以 `;` 結尾，和 Rust 一樣，但結構的欄位也是：`struct P { x: i32; y: i32; }`。

```nabi
let x = 5;
x = x + 1;                 // no `mut` needed
let x = x * 2;             // a new `x`, as shadowing in Rust
println(x);

let words = ["a", "bb"];
let w = words;             // the same list, not a move
w.push("ccc");
println(f"{words:?} {words.len()}");
```

```output
12
["a", "bb", "ccc"] 3
```

如果還是寫了 `mut`，建置會停下來，help 會告訴你把它拿掉：

```text check=checks/rust-let-mut.nabi
error[E0012]: Nabi has no `let mut`: variables are mutable by default
  --> main.nabi:1:5
   |
 1 | let mut total = 0;
   |     ^^^
   |
   = help: drop `mut`: `let total = ..;`
```

其他 Rust 寫法也會同樣讓建置停下來，而且每個 help 都會寫出對應的 Nabi 寫法：`&x` 和 `&mut x`、`&T` 和 `&mut self`、生命週期、`impl Trait`、`Box<dyn Trait>`、`Fn(i32) -> i32`、`move ||`、`println!`、`vec![..]`、`format!`、`macro_rules!`、`pub(crate)`、`crate::` 和 `super::`、`Vec`、`String` 和 `HashMap`、在 `throws` 函式中的 `Ok(x)` 和 `Err(e)`，以及對會擲回錯誤的呼叫使用 `.unwrap()`。一個錯誤只會產生一則錯誤訊息：`Rc::new(RefCell::new(x))` 只會回報一次，並附上完整的 Nabi 寫法（`Cell::new(x)`）。

## 記憶體：GC、值與控制代碼

結構是值：指派、傳遞或從串列中讀出時都會複製，原本的值仍然可以使用。只有帶 `Drop` 的型別會移動（見下一節），所以重複使用之前不需要先 clone。要修改呼叫端結構的函式接受 `ref`，呼叫端也要寫 `ref`，讓讀者看得出哪些東西可能被修改。會修改接收者的方法接受 `ref self`，對應 `&mut self`；呼叫端不需要標記。單獨的 `self` 是一份複本，就像 Rust 中 `Copy` 型別上的 `self`。

```nabi
#[derive(Debug)]
struct Point { x: i32; y: i32; }

impl Point {
    fn moved(self, dx: i32) -> Point { Point { x: self.x + dx, y: self.y } }
    fn shift(ref self, dx: i32) { self.x += dx; }
}

fn reset(ref p: Point) { p.x = 0; p.y = 0; }

let a = Point { x: 1, y: 2 };
let b = a;                 // a copy; `a` stays usable
b.shift(10);
println(f"{a:?} {b:?} {a.moved(5):?}");
reset(ref b);
println(f"{b:?}");

let ps = [Point { x: 1, y: 0 }, Point { x: 2, y: 0 }];
for ref p in ps { p.x *= 10; }      // `for p in ps.iter_mut()`
println(f"{ps:?}");
```

```output
Point { x: 1, y: 2 } Point { x: 11, y: 2 } Point { x: 6, y: 2 }
Point { x: 0, y: 0 }
[Point { x: 10, y: 0 }, Point { x: 20, y: 0 }]
```

在 Rust 要你在 `&` 和 `&mut` 之間做選擇的地方，Nabi 改為把錯誤用法變成編譯錯誤：對一份之後沒有人讀取的複本設定欄位（`fn inc(self) { self.n += 1; }` 或 `for p in ps { p.x = 0; }`）無法通過編譯，help 會指出要用 `ref self` 或 `for ref`（[第 6 章](06-structs-enums-traits.html#changing-the-callers-value-ref)）。

和鎖或其他執行緒共用的控制代碼也會受到檢查。`Mutex` 守衛的 `g.get()` 回傳的是受保護物件的檢視，所以它不能離開持有鎖的 `using` 區塊（[E0106](../errors/E0106.html)）；請改用 `g.get_clone()`。像 `r.lines()` 這樣的串流檢視會經由 `r` 讀取，並停留在 `r` 的作用域內（[E0139](../errors/E0139.html)）；請回傳擁有資料的形式 `r.into_lines()`。Nabi 只在這少數幾個地方追蹤類似借用的東西，而且每個錯誤都會告訴你該怎麼寫。

## 資源：用 `using` 取代作用域結束時的 `Drop`

GC 會釋放記憶體，但從不關閉檔案。實作 `Drop` 的型別只有一個擁有者，就像每個 Rust 值一樣：`let b = a` 會移動它，之後 `a` 就不存在了。`using x = e { .. }` 不論以何種方式離開區塊都會關閉 `x`；陳述式形式 `using x = e;` 則涵蓋外層區塊的其餘部分，並依相反順序關閉，就像 Rust 丟棄區域變數的順序。沒有使用 `using` 的 `Drop` 值會在其擁有者離開作用域時釋放。

```nabi
struct Conn { name: str; }

impl Drop for Conn {
    fn drop(ref self) { println(f"close {self.name}"); }
}

fn work() throws -> i32 {
    using a = Conn { name: "db" };
    using b = Conn { name: "cache" };
    println(f"using {a.name} and {b.name}");
    throw "query failed";
}

try work() catch e { println(f"work failed: {e}"); };
```

```output
using db and cache
close cache
close db
work failed: query failed
```

關閉檔案可能失敗（緩衝到關閉時才寫入的資料可能沒有寫到磁碟上），而 Rust 的 `Drop` 會吞掉這個錯誤。Nabi 不會替你關閉這種型別：請寫 `try f.close()`，或用 `using` 繫結它，`using` 會擲回關閉時的錯誤。在 `File` 仍開啟的情況下離開作用域是編譯錯誤（[E0134](../errors/E0134.html)）。

```nabi
use std::{env, fs, process};

let path = env::temp_dir() / f"rustnabi_{process::id()}.txt";
using f = try fs::File::create(path) {
    try f.write_all("hello\n".as_bytes());
}
println(try fs::read_to_str(path));
try fs::remove_file(path);
```

```output
hello

```

## 錯誤：用 `throws` 和 `try` 取代 `Result` 和 `?`

`fn f() -> Result<T, E>` 寫成 `fn f() throws -> T`，`f()?` 寫成 `try f()`。`try` 會標記每個可能失敗的呼叫，讓讀者看得到函式可能提前離開的每個位置。`throw "message"` 會擲回 `ValueError`；你自己的錯誤型別要實作 `Display` 和 `Error`，並列為 `throws (A | B)`。

```nabi
fn parse_port(s: str) throws -> i32 {
    let n = try s.parse::<i32>();
    if n < 1 || n > 65535 { throw f"port {n} out of range"; }
    n
}

println(try parse_port("8080") catch _ { 0 });             // match on Err(_)
let p = try parse_port("99999") catch e {
    println(f"error: {e}");
    0
};
println(p);
println(try? parse_port("x") ?? -1);                       // .ok().unwrap_or(-1)
println(try! parse_port("443"));                           // .unwrap()
```

```output
8080
error: port 99999 out of range
0
-1
443
```

`catch e: IoError { .. }` 依型別選擇，`finally` 一定會執行。`Result<T, E>` 仍然存在，用來把錯誤保留為值，以收集多個結果：`Result::catching(|| try f())`。`panic(..)`、超出範圍的索引和失敗的 `try!` 都是 panic，`catch` 不會攔截它們，和 Rust 一樣。

## 用 `T | none` 和和型別取代 `Option`

`Option<T>` 寫成 `T | none`，簡寫為 `T?`。`Some(x)` 就是 `x`，`None` 就是 `none`。`??` 對應 `unwrap_or`，`?.` 對應對欄位或方法的 `map`；後置的 `?` 會在回傳 `T?` 的函式中回傳 `none`，就像 Rust 的 `?` 用在 `Option` 上。

`A | B` 是任意型別的和型別（sum type），不需要宣告列舉：用 `match` 分支或 `is` 選出型別。`enum` 會為每個情況命名，並像 Rust 一樣攜帶資料。

```nabi
struct User { name: str; nick: str?; }

fn nick_len(u: User?) -> usize? {
    let user = u?;
    user.nick?.len()
}

fn show(v: i32 | str | none) -> str {
    match v {
        n: i32 if n < 0 => "negative",
        n: i32 => f"int {n}",
        s: str => f"str {s}",
        none => "nothing",
    }
}

let u = User { name: "ada", nick: "bob" };
println(f"{nick_len(u):?} {nick_len(none):?}");
println(u.nick ?? "-");
if let n = u.nick { println(n.to_uppercase()); }
println(f"{show(-1)}, {show(7)}, {show("x")}, {show(none)}");

enum Shape { Circle(f64), Rect { w: f64; h: f64; }, Empty }
let s = Shape::Rect { w: 2.0, h: 3.0 };
let area = match s {
    Circle(r) => 3.0 * r * r,
    Rect { w, h } => w * h,
    Empty => 0.0,
};
println(area);
```

```output
3 none
bob
BOB
negative, int 7, str x, nothing
6
```

`match` 所比對的列舉，其變體直接寫名稱即可（寫 `Circle(r)`，而不是 `Shape::Circle(r)`）。`if let x = e` 會從 `T?` 中取出值，Rust 則寫成 `if let Some(x) = e`；`if let x: T = e` 會從更大的和型別中選出一個成員。

## Trait、泛型與 `impl`

trait、`impl Trait for Type`、泛型約束、`where` 子句、預設方法和關聯型別的運作方式都和 Rust 相同，孤兒規則（orphan rule）也一樣：`impl` 需要 trait 或型別來自你的套件，而 trait 引數中的本地型別也算數，且優先於任何單純的型別參數（`impl Mul<Matrix> for f64`）（[E0076](../errors/E0076.html)）。不同之處如下：

- trait 不是型別。`dyn Trait` 寫成 `any Trait`，不需要 `Box`：`List<any Shape>` 可以持有任何實作了 `Shape` 的型別的值。`impl Trait` 寫成 `some Trait`，可用在引數和回傳位置：`fn area(s: some Shape)` 是沒有名稱的型別參數，`fn evens() -> some Iterator<Item = i32>` 隱藏了它回傳的型別。Rust 的 `type X = impl Trait;`（型別別名 `impl Trait`）寫成 `type X = some Trait;`：X 所在模組中回傳 `X` 的函式決定它的型別，其他模組只看得到約束。
- 沒有型別的參數是泛型的：`fn loud(x) -> str { x.to_uppercase() }` 接受任何具備函式主體所呼叫方法的型別。
- `Fn`、`FnMut` 和 `FnOnce` 合為一個型別 `fn(i32) -> i32`，可以持有函式或閉包。`fn() + Send` 則是可以移動到另一個執行緒的函式。
- 閉包以複製值的方式擷取。`[ref x]` 擷取變數本身，讓閉包可以修改它；`[move x]` 則把值交出去：建立閉包的程式碼之後就不能再使用 `x`。
- `#[derive(..)]` 會像 Rust 一樣替你寫好 `impl`。`Copy` 只是一個標記，因為每個結構在 `let y = x;` 時本來就會複製。

```nabi
trait Shape {
    fn area(self) -> f64;
    fn describe(self) -> str { f"area {self.area():.1}" }
}
struct Square { side: f64; }
struct Circle { r: f64; }
impl Shape for Square { fn area(self) -> f64 { self.side * self.side } }
impl Shape for Circle { fn area(self) -> f64 { f64::PI * self.r * self.r } }

fn largest<T: Shape>(xs: List<T>) -> f64 {
    xs.map(|s| s.area()).fold(0.0, |a, b| a.max(b))
}

let shapes: List<any Shape> = [Square { side: 2.0 }, Circle { r: 1.0 }];
for s in shapes { println(s.describe()); }
println(largest([Square { side: 1.0 }, Square { side: 3.0 }]));

fn apply(f: fn(i32) -> i32, x: i32) -> i32 { f(x) }
fn adder(n: i32) -> fn(i32) -> i32 { |x| x + n }
println(apply(adder(5), 10));

let count = 0;
let snapshot = || count + 1;
let inc = [ref count] || { count += 1; };
inc();
inc();
println(f"{snapshot()} {count}");
```

```output
area 4.0
area 3.1
9
15
1 2
```

## 數字

沒有後綴的整數字面值會從它所在的位置（`let b: u8 = 200;`、引數、另一個型別的運算元）或之後的用法取得型別，沒有任何東西給它型別時則是 `i64`。浮點數字面值同樣預設為 `f64`。長度和索引是 `usize`，所以當 `total` 是 `i32` 時，`total / xs.len()` 需要 `as`。`int` 是任意大小的整數。

沒有任何隱含轉換：`i32 + i64` 是錯誤，help 會替你寫出 `as`。`as` 會截斷，`.into()` 只允許用在不會遺失資訊的地方，而 `.try_into::<T>()` 在值放不下時會擲回錯誤，所以 `try_from` 的位置需要加上 `try`。

```nabi
let a = 7;                    // i64：沒有其他東西給它型別
let b: u8 = 200;
let n = 3i32;
println(f"{a + 1} {(n as i64) + a} {b as i32 + 100} {300 as u8}");
let w: i64 = n.into();
let small: u8 = try 255i32.try_into::<u8>();
println(f"{w} {small}");
println(try? 256i32.try_into::<u8>() ?? 0);
println(f"{-7 / 2} {-7 % 2} {-7 %% 2} {7.pow(2)}");
let big: int = 2;
println(big.pow(100));
```

```output
8 10 300 44
3 255
0
-3 -1 1 49
1267650600228229401496703205376
```

`%%` 是向下取整的餘數，在除數為正時等同 Rust 的 `rem_euclid`。`x.pow(n)` 是方法，和 Rust 一樣，`x ** n` 則是運算子形式（整數底數接受 `u32` 指數）。

## 運算子

算術、位元和比較運算子及其優先順序都和 Rust 相同，運算子 trait 是 `Add`、`Mul`、`Neg`、`Index` 等。Nabi 另外加了幾個：

- 串連比較：`0 <= i < n`。
- 用 `in` 和 `!in` 檢查成員：`x in xs`、`k !in dict`。
- 逐元素的「點」運算子 `.+`、`.*`、`.>` 等，會把運算子套用到串列或 `Array` 的每個元素上，還有就地形式 `.=` 和 `.+=`（[第 8 章](08-collections-iterators.html#advanced-broadcast-operators)、[第 21 章](21-arrays-and-numerics.html)）。
- 用 `**` 計算乘冪：`2 ** 10`、`x ** 0.5`。
- `??` 提供預設值，`?.` 可穿過 `none` 串連。
- 在 `Path` 上的 `/` 會連接路徑：`dir / "file.txt"`。

```nabi
let a = [1.0, 2.0, 3.0];
let b = [10.0, 20.0, 30.0];
println(f"{a .+ b:?} {a .* 2.0:?}");
let i = 3;
println(0 <= i < 5 && i !in [1, 2]);
```

```output
[11.0, 22.0, 33.0] [2.0, 4.0, 6.0]
true
```

## 模組與可見性

`mod name;` 會載入同一位置的 `name.nabi`，`name/` 則放它的子模組，和沒有 `mod.rs` 的 Rust 2018 一樣。從套件根目錄開始的路徑以 `pkg::` 開頭，對應 Rust 的 `crate::`；`sup::` 對應 `super::`。項目預設對其模組私有；`pub(sub)` 開放給子模組，`pub(pkg)` 開放給整個套件（`pub(crate)`），`pub` 則開放給所有人。標準函式庫中有少數 `pub` 項目標有 `#[internal]`：只有工具鏈本身的套件可以使用它們。

```nabi
use geo::{Point, unit};

mod geo {
    pub struct Point { pub x: f64; pub y: f64; }
    pub fn dist_sq(a: Point, b: Point) -> f64 { sq(a.x - b.x) + sq(a.y - b.y) }
    fn sq(v: f64) -> f64 { v * v }
    pub mod unit {
        pub fn origin() -> sup::Point { sup::Point { x: 0.0, y: 0.0 } }
    }
}

println(geo::dist_sq(unit::origin(), Point { x: 3.0, y: 4.0 }));
```

```output
25
```

`println`、`List`、`Dict`、`Mutex`、`channel`、`task` 和 `par` 都在 prelude 中；`std` 中與作業系統互動的模組（`fs`、`env`、`process`、`thread`、`net`）則用 `use std::fs;` 匯入。

## 屬性與巨集

屬性的寫法和 Rust 相同。原本會用到 `macro_rules!` 或過程式巨集（procedural macro）的程式，在 Nabi 中只有一種機制：元定義（meta definition），也就是帶有 `@` 記號的 Nabi 程式碼樣板。`#[impl_derive(T)]` 定義一個 derive，`#[impl_attribute(name)]` 定義一個屬性，`#[impl_macro(name)]` 則定義一個 `name!(..)` 呼叫（[6.8](../spec/refs-lang/006-meta-system/608-meta-blocks.html)）。沒有在編譯時對語彙單元串流執行的程式碼。`println`、`assert` 和 `format` 式的字串都是一般函式和 f-string，所以不需要加 `!`。

```nabi
#[impl_macro(twice)] @macro {
    ( @e: expr ) => { { let v = @e; (v, v) } }
}

let (a, b) = twice!(21 * 2);
println(f"{a} {b}");
assert(a == b, "same value");
```

```output
42 42
```

## 並行

執行緒和 Rust 相同：`thread::spawn` 接受一個閉包，`join` 回傳它的值（若該執行緒觸發 panic 則擲回錯誤）。`Send` 和 `Sync` 都存在，而集合型別不是 `Sync`：要把串列交給另一個執行緒的閉包，需用 `[move rows]` 取得它。`Mutex`、`RwLock`、`Atomic` 和 `channel` 都在 prelude 中，交給鎖或通道的值之後就不能再使用，這項交付檢查取代了 Rust 中移入 `Mutex` 的做法。

`async fn` 和 `await` 內建了執行環境，所以不需要 Tokio：`task::spawn` 啟動一個任務，`task::join` 等待多個任務，而含有頂層 `await` 的指令碼會以 `async fn main` 的形式執行。`await` 是前置運算子：寫 `await f()`，而不是 `f().await`。`par::map`、`par::filter` 和 `par::for_each` 會把工作分散到執行緒池上，就像 Rayon。

```nabi
use std::thread;

let rows = [3, 1, 2];
let h = thread::spawn([move rows] || {
    rows.sort();
    rows[0]
});
println(f"smallest: {try h.join()}");

let m = Mutex::new(0);
par::for_each(1..=100, |n| { using g = m.lock() { g.set(g.get() + n); } });
println(using g = m.lock() { g.get() });

async fn fetch(id: i32) -> str {
    await task::sleep(Duration::ms(5));
    f"item-{id}"
}
let tasks = [task::spawn(fetch(1)), task::spawn(fetch(2))];
println(f"{await task::join(tasks):?}");
println(f"{par::map(1..=4, |x| x * x):?}");
```

```output
smallest: 1
5050
["item-1", "item-2"]
[1, 4, 9, 16]
```

## `unsafe` 與 FFI

`extern "C" { .. }` 宣告 C 函式，每次呼叫都要寫在 `unsafe { .. }` 中。`#[repr(C)]`、原始指標（`bare::ptr` 中的 `Ptr<T>`、`ConstPtr<T>`）、`#[link("m")]` 和 `extern "C" fn` 回呼的用法都和 Rust 相同。`Pod`（`bare::mem::Pod`，plain old data）是和 `bytemuck::Pod` 一樣的 `unsafe trait`：使用 `#[derive(Pod)]` 或 `unsafe impl Pod for T {}`，編譯器仍會檢查其配置。`List`、`Dict` 或閉包不能傳進 C，因為回收器可能會移動它們；請傳遞指標和長度。

```nabi
extern "C" {
    fn abs(x: i32) -> i32;
    fn strlen(s: str) -> usize;
}

println(unsafe { abs(-42) });
println(unsafe { strlen("hello") });
```

```output
42
5
```

## Cargo、crate 與 `nabi.nabion`

`nabi` 是套件管理工具，清單是 `nabi.nabion`，以 Nabi 的資料格式 NABION 撰寫，而不是 TOML。套件發布到 [packages.nabi-lang.org](https://packages.nabi-lang.org)，就像 crate 發布到 crates.io。`nabi.lock` 像 `Cargo.lock` 一樣固定版本。`nabi publish` 不會上傳原始碼封存檔：它會在索引中新增一行，指向你的 git 儲存庫中的某個提交。單一檔案也可以是一個程式：`nabi file.nabi` 不需要套件和 `main` 就能執行它，而 `nabi add json file.nabi` 會把相依套件記錄在檔案本身。

```nabion
Package { name: "app", version: "0.1.0", toolchain: "0.1", deps: {
    "json": "^0.1",
    "geom": Dep::path("../geom"),
} }
```

| Cargo | Nabi |
|---|---|
| `cargo new app`, `cargo new --lib geom` | `nabi new app`, `nabi new --lib geom` |
| `cargo build`, `cargo run`, `cargo check` | `nabi build`, `nabi run`, `nabi check` |
| `cargo test`, `cargo bench` | `nabi test`, `nabi bench` |
| `cargo fmt`, `cargo clippy` | `nabi fmt`, `nabi lint` |
| `cargo doc` | `nabi doc` |
| `cargo add serde`, `cargo update` | `nabi add json`, `nabi update` |
| `cargo install`, `cargo publish` | `nabi install`, `nabi publish` |
| `rustup toolchain install` | `nabi toolchain install` |
| `[features]`, `--features` | `nabi.nabion` 中的 `features:`、`--features`（[第 18 章](18-features-and-publishing.html)） |

`nabi fmt` 會修正縮排、行尾空白和空行，並把模組開頭的 `mod` / `use` 行依 `pub mod`、`mod`、`pub use`、`use` 分組排序；一行之內的語彙單元和空格則保留你原本的寫法。測試是套件中任何位置的 `#[test]` 函式，外面不需要包 `#[cfg(test)] mod tests`：一般建置會排除它，而 `nabi test` 會在各自獨立的行程中執行每個測試，所以一個 panic 不會讓其他測試停下來。

```nabi
fn gcd(a: i32, b: i32) -> i32 { if b == 0 { a } else { gcd(b, a % b) } }

#[test]
fn gcd_works() {
    assert_eq(gcd(12, 18), 6);
}

println(gcd(12, 18));
```

```output
6
```

## Nabi 中的 Rust 慣用寫法

| Rust | Nabi |
|---|---|
| `let mut x = 0;` | `let x = 0;` |
| `let x = x.trim();`（遮蔽） | `let x = x.trim();` |
| `&T`、`&mut T` 參數 | `T`（結構是一份複本，`List` 則是同一個物件）、`ref p: T` |
| `&mut self` | `ref self` |
| `for x in xs.iter_mut()` | `for ref x in xs` |
| 重複使用前的 `x.clone()` | 不需要：結構會複製；要取得獨立的 `List` 才用 `.clone()` |
| `Box<dyn Trait>`, `Vec<Box<dyn Trait>>` | `any Trait`, `List<any Trait>` |
| `fn f(x: impl Trait)` | `fn f(x: some Trait)` |
| `fn f() -> impl Trait` | `fn f() -> some Trait` |
| `type X = impl Trait;` | `type X = some Trait;` |
| `impl Fn(i32) -> i32`, `Box<dyn FnMut()>` | `some fn(i32) -> i32`, `fn(i32) -> i32`, `fn()` |
| `move \|\| ..` | `[move x] \|\| ..` |
| 修改所擷取區域變數的閉包 | `[ref x] \|\| { x += 1; }` |
| `Option<T>`, `Some(x)`, `None` | `T?`, `x`, `none` |
| `opt.unwrap_or(d)`, `opt.map(\|v\| v.f())` | `opt ?? d`, `opt?.f()` |
| `if let Some(x) = opt` | `if let x = opt` |
| `fn f() -> Result<T, E>` | `fn f() throws -> T` |
| `f()?`, `f().unwrap()`, `f().ok()` | `try f()`, `try! f()`, `try? f()` |
| `Err("msg".into())`, `anyhow!("msg")` | `throw "msg"` |
| `match r { Ok(v) => v, Err(e) => .. }` | `try f() catch e { .. }` |
| `impl Drop` + 作用域結束 | `using x = e;`，檔案則用 `try f.close()` |
| `Vec<T>`, `HashMap<K, V>`, `HashSet<T>` | `List<T>`, `Dict<K, V>`, `Set<T>` |
| `vec![1, 2]`, `String`, `&str` | `[1, 2]`、`str`（只有一種字串型別） |
| `println!("{x:?}")`, `format!(..)` | `println(f"{x:?}")`, `f".."` |
| `assert_eq!(a, b)` | `assert_eq(a, b)` |
| `x as i64`, `i64::from(x)` | `x as i64`, `x.into()` |
| `u8::try_from(x)?` | `try x.try_into::<u8>()` |
| `x.rem_euclid(n)` | `x %% n` |
| `crate::a::b`, `super::c`, `pub(crate)` | `pkg::a::b`, `sup::c`, `pub(pkg)` |
| `macro_rules! m` | `#[impl_macro(m)] @macro { .. }` |
| `fut.await`, `tokio::spawn` | `await fut`, `task::spawn` |
| `rayon` `par_iter().map(..)` | `par::map(xs, ..)` |
| `struct P { x: i32, y: i32 }` | `struct P { x: i32; y: i32; }` |
