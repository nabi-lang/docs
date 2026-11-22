# 2. 值與型別

本章介紹用來宣告變數的 `let`、數字和字串等基本型別，以及用來組合值的元組和串列。Nabi 是靜態型別語言，但大多數型別都能推論出來，所以你很少需要寫出型別。

## `let` 與重新指派

`let` 會建立新的變數；要寫入既有的變數，請使用不帶 `let` 的 `=`。變數預設是可變的（沒有 `mut` 標記）。

```nabi
let name = "nabi";
let count = 0;
count = count + 1;
count += 1;
println(f"{name}: {count}");
```

```output
nabi: 2
```

由於宣告和重新指派在語法上不同，打錯字永遠不會悄悄地建立新變數。指派給從未宣告過的名稱會造成編譯錯誤。

```text check=checks/02-undeclared.nabi
error[E0057]: `coutn` is not declared; a variable is created only by `let`
  --> main.nabi:2:1
   |
 2 | coutn = count + 1;
   | ^^^^^
   |
   = help: did you mean `count`? To declare a new variable, write `let coutn = count + 1;`
```

還有幾條規則：

- 必須提供初始值。`let x;` 是錯誤。如果還沒有值，請明確寫出「沒有值」：`let x: i32 | none = none;`。
- 變數的型別在宣告時就固定，之後不會改變。把字串指派給 `i32` 變數是錯誤。
- 對已經存在的名稱再寫一次 `let`，會建立一個新變數並遮蔽舊的變數，不論是在同一個作用域還是內層區塊中。新變數的初始值可以讀取舊的變數，型別也可以不同，所以一個值在每個處理步驟中都能沿用同一個名稱。
- 關鍵字不能當作名稱。其中六個保留給未來使用，目前還沒有語法：`macro`、`defer`、`dyn`、`move`、`gen` 和 `do`。如果一定要用關鍵字當名稱，請在前面加上 `r#`（`r#move`、`r#type`）；錯誤訊息會提示這個寫法。

```nabi
let a = 1;
{
    let a = 42;
    a = 10;
    println(f"inner a = {a}");
}
println(f"outer a = {a}");
```

```output
inner a = 10
outer a = 1
```

```nabi
let line = "  42  ";
let line = line.trim();          // 讀取舊的 `line`
let line: i32 = try! line.parse();
println(f"{line + 1}");
```

```output
43
```

要改變變數的值，請不帶 `let` 直接指派。重新宣告不會動到舊的變數（[5.1.4](../spec/refs-lang/005-modules/501-variable-declaration.html#514-redeclaration-in-the-same-scope)）：你開啟的檔案不會因此被關閉，所以請先關閉它，或把它移入新的值中，例如 `let f = BufReader::new(f);`（第 13 章）。

## 數值型別與字面值

整數有 `i8` `i16` `i32` `i64` `isize`，以及無號的 `u8` `u16` `u32` `u64` `usize`；浮點數有 `f32` `f64`，而 `f16`／`bf16` 用於在大型陣列中儲存 16 位元浮點數（[陣列與數值運算](21-arrays-and-numerics.html#16-位元浮點數f16-與-bf16)）。帶有型別名稱後綴的字面值就是該型別；沒有後綴時，則依照**預期型別**（expected type）決定。用 `let` 綁定的字面值，會從該變數在函式中後續的用法取得型別；只有在沒有任何線索時，整數才預設為 `i64`，浮點數預設為 `f64`。

```nabi
let a = 42;            // i64
let b: u8 = 200;       // u8, from the annotation
let c = 7i64;          // suffix
let d = 2.5f32;
let e = 3.14;          // f64
let big = 1_000_000u64;
println(f"{a} {b} {c} {d} {e} {big}");
println(f"{0xFF} {0b1010} {0o17} {1e3}");
println(f"{7 / 2} {-7 / 2} {7.0 / 2.0}");
```

```output
42 200 7 2.5 3.14 1000000
255 10 15 1000
3 -3 3.5
```

因為型別來自用法，你很少需要寫出它：

```nabi
fn count_big(xs: List<i32>) -> i32 {
    let c = 0;                 // i32: 函式回傳它
    for x in xs { if x > 100 { c += 1; } }
    c
}
let hits = Atomic::new(0);     // 計數器型別來自下方的 fetch_add
let step: u8 = 2;
hits.fetch_add(step);
println(f"{count_big([5, 500, 900])} {hits.load()}");
```

```output
2 2
```

Nabi 仍然從不做轉換：一個變數只有一種型別。把 `let t = 0;` 當作 `f64` 使用是錯誤，錯誤訊息會告訴你在宣告處寫成 `let t = 0.0;`。

型別參數宣告的預設型別也算在內。有 `struct Vec3<T = f32>` 時，`Vec3::new(1.0, 2.0, 3.0)` 中的字面值預設為 `f32`，所以這個向量是 `Vec3<f32>`；預期型別、後續用法或有型別的引數仍然優先（[泛型](07-generics.html#預設值)）。

`_` 是數字分隔符，`0x`／`0b`／`0o` 分別表示十六進位／二進位／八進位。開頭的零不會讓數字變成八進位，`0755` 是編譯錯誤：八進位請寫 `0o755`，十進位請寫 `755`。整數除法會向零截斷。超出型別範圍的字面值（`let x: u8 = 256;`）是編譯錯誤。

## 沒有隱含轉換：`as`

Nabi 從不自動混用不同的數值型別。把 `i32` 和 `i64` 相加是錯誤。

```text check=checks/02-mixed-int.nabi
error[E0054]: operator `+` cannot be applied to `i32` and `i64`
  --> main.nabi:3:12
   |
 3 | println(f"{a + b}");
   |            ^^^^^
   |
   = help: Nabi never converts numbers implicitly; convert one side: `(a as i64) + b`
```

這是刻意的設計，用來避免悄悄失去精確度。轉換需要明確寫出。`as` 一定會成功，但可能截斷或改變值；`.into()` 只允許無損轉換：在每個目標上都能保留所有值的擴大轉換，例如 `i32` 到 `i64`、`u8` 到 `isize` 或 `u32` 到 `f64`（完整的表格請見 [stdlib 2.11.8](../spec/refs-standard-libraries/200-core/211-conversion.html#2118-from-impl-matrix---between-primitives)）。

```nabi
let x: i32 = 10;
let y: i64 = 3;
println(f"{(x as i64) + y}");
let w: i64 = x.into();
println(f"{w}");
println(f"{2.9 as i32} {300 as u8} {x as f64 / 4.0}");
```

```output
13
10
2 44 2.5
```

`2.9 as i32` 會捨去小數部分，`300 as u8` 只保留低 8 位元。對於不能失去任何資訊的轉換，`.try_into::<T>()` 會檢查範圍，超出時擲回錯誤；處理錯誤的方式請見[第 10 章](10-errors.html)。

數字永遠不能強制轉型為 `bool`：`n as bool` 是錯誤。請寫出你要的比較，`n != 0`（浮點數則是 `x != 0.0`）。

## 整數的陷阱

有幾條整數規則會讓從 Python、Go 或 JavaScript 轉過來的人感到意外。

**`len()` 是 `usize`。** 長度和索引都是 `usize`。字面值的型別來自它的用法，所以 `let total = 6;` 若除以 `xs.len()`，也會是 `usize`；但型別已經固定的值（`i64` 欄位、參數或型別標註）不會被轉換：`total` 是 `i64` 時，`total / xs.len()` 是錯誤。請用 `as` 轉換其中一邊。

```text check=checks/02-len-usize.nabi
error[E0054]: operator `/` cannot be applied to `i64` and `usize`
```

```nabi
let xs: List<i64> = [10, 20, 30];
let total = 0;
for x in xs { total += x; }
let avg = total / (xs.len() as i64);
println(f"average {avg}");

for i in 0..xs.len() {       // i is a usize, so xs[i] just works
    print(f"{i}:{xs[i]} ");
}
println("");
```

```output
average 20
0:10 1:20 2:30 
```

**`/` 和 `%` 向零截斷。** `-7 / 2` 是 `-3`，`-7 % 2` 是 `-1`：餘數的正負號跟隨左邊，和 C、Go、Rust 相同。若要 Python 的 `%`（結果的正負號跟隨除數），請使用 `%%`（向下取整的模數）。它很適合用來讓索引繞回開頭。

```nabi
println(f"{-7 / 2} {-7 % 2} {-7 %% 2}");
let len = 5;
let i = -1;
println(f"{i % len} {i %% len}");
```

```output
-3 -1 1
-1 4
```

**運算子的結合方式和 Rust 相同。** `*` `/` `%` 比 `+` `-` 優先，`+` `-` 又比位移 `<<` `>>` 優先，接著是 `&`、`^`、`|`，而它們全都比比較運算優先。所以 `1 + 2 << 3` 是 `(1 + 2) << 3`，`x & mask == 0` 是 `(x & mask) == 0`，而不是 C 的 `x & (mask == 0)`。乘冪運算子 `**` 的優先順序最高，比前置的 `-` 還高，並且和 Python 一樣向右結合：`-2 ** 2` 是 `-4`，`2 ** 3 ** 2` 是 `2 ** 9`。整數底數的指數是 `u32`。

```nabi
println(f"{1 + 2 << 3} {6 & 3 == 2}");
println(f"{2 ** 10} {-2 ** 2} {2 ** 3 ** 2} {1.5 ** 2.0}");
```

```output
24 true
1024 -4 512 2.25
```

**溢位會觸發 panic。** 超出固定位元寬度整數的範圍時，不論是偵錯建置還是 `--release`，每一種建置都會觸發 panic，所以錯誤的值永遠不會悄悄溜過。

```nabi build
let x: i32 = 2147483647;
println(f"{x + 1}");
```

```text
$ nabi run --release
thread 'main' panicked at src/main.nabi:2:12:
attempt to add with overflow
```

在密集的整數迴圈中，這些檢查會花掉幾個百分點的效能。需要找回這些效能的程式，可以用 `-C overflow-checks=no` 關閉檢查，或在 `nabi.nabion` 中針對某個建置設定檔寫 `profile: { "release": { overflow_checks: false } }`；這時溢位會繞回。預期會溢位時，請用 `checked_add`（得到 `none`）、`wrapping_add` 或 `saturating_add` 寫明你要的行為，或使用永遠不會溢位的 `int`（見下文）。

```nabi
let x: i32 = 2147483647;
println(f"{x.checked_add(1):?} {x.wrapping_add(1)} {x.saturating_add(1)}");
```

```output
none -2147483648 2147483647
```

## bool、char、str

`bool` 是 `true`／`false`，用 `&&`、`||` 和 `!` 組合。同一方向的比較可以串接：`0 <= x < 10` 就是 `0 <= x && x < 10`，而 `x` 只求值一次（`a < b > c` 是錯誤）。`x in xs` 和 `x !in xs` 檢查是否屬於某個串列、集合、字典（它的鍵）、字串或範圍。`char` 是單一 Unicode 字元，用單引號表示；`b'A'` 是該 ASCII 字元的位元組，型別為 `u8`。`str` 是 UTF-8 字串，用雙引號表示。

```nabi
let ok = 3 < 5 && !false;
let x = 7;
println(f"{ok} {0 <= x < 10} {x in [1, 7]} {x !in 0..5}");
let ch = '가';
println(f"{ch} {ch as u32} {65u8 as char} {b'A'}");

let s = "héllo";
println(f"{s.len()} bytes, {s.char_count()} chars");
let t = s + ", nabi";
println(f"{t} {t.to_uppercase()} {t.contains("nabi")}");
println("tab:\tend");
println(r"raw: C:\path\n");
```

```output
true true true true
가 44032 A 65
6 bytes, 5 chars
héllo, nabi HÉLLO, NABI true
tab:	end
raw: C:\path\n
```

韓文音節 `'가'`（U+AC00，碼位 44032）顯示 `char` 可以容納任何 Unicode 字元，而不只是 ASCII。`len()` 計算的是位元組數；需要字元數時請用 `char_count()`。`r"..."` 是不處理跳脫序列的原始字串。其他 `str` 方法（`trim`、`split`、`replace`、`parse` 等）請見[標準函式庫文件](../../std/core/str.html)。

## 元組、陣列與串列

**元組**（tuple）把幾個不同型別的值組合在一起。用 `.0`、`.1` 存取，或用 `let` 解構。

```nabi
let p = (1, "one", true);
println(f"{p.0} {p.1} {p:?}");
let (n, word, _) = p;
println(f"{word} = {n}");
```

```output
1 one (1, "one", true)
one = 1
```

`[...]` 字面值預設是 **`List`**。它是可以增長的串列，所有指向同一個串列的變數都會看到它的變更。`List` 不能用 `{}` 印出，請使用 Debug 格式 `{:?}`。

```nabi
let names = ["ada", "grace"];
names.push("linus");
println(f"{names:?} {names.len()} {names[0]}");

let empty: List<i32> = [];
println(f"{empty.is_empty()}");
```

```output
["ada", "grace", "linus"] 3 ada
true
```

如果變數的型別是 `[T; N]`，同樣的字面值就會成為固定長度的**陣列**。它的大小在編譯時期就已知，所以適合用於矩陣、緩衝區這類形狀固定的資料。

```nabi
let grid: [[i32; 2]; 2] = [[1, 2], [3, 4]];
println(f"{grid:?} {grid[1][0]} {grid.len()}");
```

```output
[[1, 2], [3, 4]] 3 2
```

`[v; n]` 會用一個值建立有 `n` 個元素的陣列。Nabi 只對 `v` 求值一次，並給每個元素各自的複本，所以改變一個元素永遠不會影響另一個，即使元素是串列也一樣：

```nabi
let g = [[0; 3]; 2];
g[0][1] = 5;
let rows = [List::<i32>::new(); 3];
rows[0].push(7);
println(f"{g:?} {rows:?}");
```

```output
[[0, 5, 0], [0, 0, 0]] [[7], [], []]
```

數字和其他單純的值會原樣複製。保存集合型別的元素會用 `clone()` 複製，所以它的型別必須是 `Clone`。不是 `Clone` 的型別，或只有一個擁有者的值（例如開啟中的 `File`），會造成編譯錯誤；錯誤的說明會寫出 `array::from_fn::<T, n>(|_| ..)`，它會為每個元素建立各自的值。

空串列會從後續的用法取得元素型別：在 `let xs = []; xs.push(1u8);` 之後，`xs` 是 `List<u8>`。若函式中沒有任何線索，請寫出型別：`let xs: List<i32> = [];`。

## 大整數：`int`

`i32` 這類固定位元寬度的型別很快，但有範圍限制。需要不必擔心範圍的精確整數時，請使用 `int`。小的值會像固定位元寬度的整數一樣儲存，變大時會自動切換成大整數。`int` 只會出現在明確寫出型別的地方，和固定位元寬度型別混用時也需要明確轉換。

```nabi
let big: int = 1;
for _ in 0..100 {
    big = big * 2;
}
println(f"2^100 = {big}");
```

```output
2^100 = 1267650600228229401496703205376
```

`int` 可以用 `big.to_f64()` 轉成 `f64`，結果會捨入到最接近的值。捨入並非無損，所以 `into()` 和 `as f64` 對它無法編譯；值超過 `f64::MAX` 時，`try f64::try_from(big)` 會擲回錯誤，而不是得到 `inf`。

> [!NOTE]
> 進階：`int` 也提供 `isqrt`、`modpow`、`gcd`、`lcm` 等數論輔助函式。它們都列在 [`int` API](../../std/core/num/bigint.html) 中。

## 常數

`const` 是在編譯時期就固定的值。和 `let` 不同，它必須寫出型別，而且可以出現在模組的頂層。它的值會內嵌到每一個使用處，所以每次使用都是一個新的值：修改它（對 `const` 串列執行 `XS.push(4)`）是錯誤；請先把它複製到變數中（`let xs = XS;`）。`const` 只能讀取常數：在有 `let n` 的情況下寫 `const HALF: i32 = n / 2;` 是錯誤，錯誤訊息會告訴你改寫成 `let half = n / 2;`。

```nabi
const MAX_USERS: i32 = 100;
const GREETING: str = "welcome";

fn capacity_left(used: i32) -> i32 {
    MAX_USERS - used
}

println(f"{GREETING}: {capacity_left(42)} seats left");
```

```output
welcome: 58 seats left
```

## 小結

- `let` 宣告，`=` 重新指派。對同一個名稱再寫 `let` 會建立新的變數，而且必須提供初始值。
- 沒有後綴的字面值依照預期型別決定，預設為 `i64`／`f64`，或它所填入的型別參數的預設型別。
- 數值型別之間沒有自動轉換。請使用 `as`、`.into()` 或 `.try_into::<T>()`。
- `len()` 和索引是 `usize`；要和 `i64` 混用時，請用 `as` 轉換。
- `/` 和 `%` 向零截斷；`%%` 是向下取整的模數。除非關閉溢位檢查，否則溢位在每一種建置中都會觸發 panic。
- `[...]` 是 `List`；型別為 `[T; N]` 時則是固定陣列。`[v; n]` 是元素各為 `v` 獨立複本的固定陣列。`List` 請用 `{:?}` 印出。
- `int` 是沒有上限的整數；`const` 是編譯時期常數。

## 延伸閱讀

- [變數宣告](../spec/refs-lang/005-modules/501-variable-declaration.html)、[`const`](../spec/refs-lang/005-modules/502-const.html)
- [數值字面值](../spec/refs-lang/001-lexical/106-numeric-literals.html)、[字串字面值](../spec/refs-lang/001-lexical/107-string-literals.html)、[字元字面值](../spec/refs-lang/001-lexical/108-character-literals.html)
- [強制轉型與轉換](../spec/refs-lang/002-types/214-cast-and-conversion.html)、[型別運算式](../spec/refs-lang/002-types/201-type-expressions.html)
- [數值型別與 `int`](../spec/refs-standard-libraries/200-core/201-numbers.html)、[格式化輸出](../spec/refs-standard-libraries/200-core/210-fmt.html)
- 標準函式庫 API：[`str`](../../std/core/str.html)、[`List`](../../std/core/collections/list.html)
- 下一章：[3. 函式](03-functions.html)
