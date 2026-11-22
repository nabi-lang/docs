# 給 Python 使用者的 Nabi

這是寫給 Python 程式設計師的對照圖：哪些東西可以直接沿用、哪些地方 Nabi 的寫法不同，以及 Python 標準函式庫的各個模組在 Nabi 裡對應到哪裡。下面每個區塊都是完整的程式，用 `nabi run file.nabi` 執行即可。本指南的各章會完整說明每個部分。

## 可以沿用的部分

- 檔案和指令碼一樣由上往下執行，不需要 `main`：可以用 `nabi file.nabi` 執行、寫 `#!/usr/bin/env nabi` 這一行，也可以在檔案本身宣告相依套件（`nabi add json file.nabi`）。
- f-string，格式規範也相同：`f"{x:.2f}"`、`f"{name:>10}"`、`f"{x=}"`。
- 支援負索引和步長的切片：`xs[-1]`、`xs[1:3]`、`xs[::-1]`。
- 預設引數與具名引數：`fn f(a: i32, sep: str = " ")`，呼叫時寫 `f(1, sep: ", ")`。
- `in`、串連比較（`0 <= i < n`）、元組拆解與交換（`(a, b) = (b, a)`）。
- `try` / `catch` / `finally`、產生器（`yield fn`）、`async` / `await`，以及帶守衛條件的 `match`。
- 大整數：`int` 沒有大小限制。
- REPL（`nabi repl`）和 Jupyter 核心（`nabi jupyter`）。

## 一開始會意外的地方

- `print` 不會加上換行，`println` 才會。兩者都接受多個值和 `sep:`。
- 每個值都有編譯器知道的型別。沒有寫型別的參數是泛型的：它接受任何具備函式主體所呼叫方法的型別，也就是在編譯時檢查的鴨子型別（duck typing）。
- 區塊用 `{ }` 表示，而不是縮排；宣告新名稱要用 `let`。
- 沒有真值性（truthiness）：條件必須是 `bool`。請寫 `!xs.is_empty()`、`n != 0` 或 `x != none`。
- 缺失值是 `none`，可能持有它的型別寫成 `T | none`。
- 整數的 `/` 會向零取整（`7 / 2` 是 `3`）；向下取整除法是 `n.div_floor(2)`，`%%` 是向下取整的餘數。`//` 是註解的開頭。
- 整數超出其型別範圍時會觸發 panic。沒有後綴的整數是 `i64`，除非它的用法給了它其他型別；`int` 可以容納任意大小。
- 結構是值：`let q = p;` 會複製。`List`、`Dict` 和 `Set` 是共用的控制代碼，和 Python 一樣。
- `str` 的切片以位元組計算，不是以字元計算；要逐一走訪字元，請用 `s.chars()`。

```nabi
let name = "Ada";
let xs = [1, 2, 3];
print("no newline, ");
println("then one");
println("total:", xs.len(), xs);
println(1, 2, 3, sep: ", ");
println(f"{name} has {xs.len()} items: {xs:?}");
let x = 5;
println(f"{x=}");
```

```output
no newline, then one
total: 3 [1, 2, 3]
1, 2, 3
Ada has 3 items: [1, 2, 3]
x=5
```

## 控制流程

`if` 是運算式，所以也能取代 `a if c else b`。`elif` 寫成 `else if`，`for i in range(n)` 寫成 `for i in 0..n`。

```nabi
fn grade(score: i32) -> str {
    if score >= 90 { "A" } else if score >= 80 { "B" } else { "C" }
}

let people = [("ada", 92), ("bob", 78)];
for (name, score) in people {
    println(f"{name}: {grade(score)}");
}
for (i, (name, _)) in people.enumerate() {
    println(f"{i} {name}");
}
for i in 0..3 {
    if i == 1 { continue; }
    println(i);
}
let label = if people.len() > 1 { "many" } else { "one" };
println(label);
let c = 5;
if 0 < c < 10 && c != 7 { println("in range"); }
if 3 in [1, 2, 3] { println("member"); }
```

```output
ada: A
bob: C
0 ada
1 bob
0
2
many
in range
member
```

`and`、`or` 和 `not` 寫成 `&&`、`||` 和 `!`；`not in` 寫成 `!in`。

## 推導式就是迭代器鏈

Nabi 沒有串列、字典或集合推導式（comprehension）。集合型別都有迭代器方法，再用 `to_list()`、`to_dict()` 和 `to_set()` 收集結果。`sum`、`min`、`max`、`sorted`、`any` 和 `all` 也都是方法。

```nabi
let xs = [1, 2, 3, 4, 5];
// [x * x for x in xs if x % 2 == 1]
let odd_squares = xs.filter(|x| x % 2 == 1).map(|x| x * x).to_list();
println(f"{odd_squares:?}");
println(f"{xs.sum()} {xs.min() ?? 0} {xs.max() ?? 0}");

let words = ["pear", "fig", "apple"];
println(f"{words.sorted():?}");
println(f"{words.sorted_by_key(|w| w.len()):?}");
println(words.join(", "));
println(f"{xs.any(|x| x > 4)} {xs.all(|x| x > 0)}");

// {w: len(w) for w in words}
let lengths = words.map(|w| (w, w.len())).to_dict();
println(lengths["fig"]);
println([1, 1, 2].to_set().len());

let (a, b) = (1, 2);
(a, b) = (b, a);
println(f"{a} {b}");
```

```output
[1, 9, 25]
15 1 5
["apple", "fig", "pear"]
["fig", "pear", "apple"]
pear, fig, apple
true true
3
2
2 1
```

Python 寫 `lambda x: x + 1` 的地方，閉包寫成 `|x| x + 1`。

## 用 `none` 取代 `None`

可能什麼都找不到的函式會回傳 `T | none`。`??` 提供預設值，`x == none` 用來檢查，`if let` 則把值取出來。在 `x != none` 之後，或在 `if x == none { return; }` 下方，`x` 就是值本身。

```nabi
let xs = ["a", "b"];
let i = xs.position(|x| x == "b");
if let k = i { println(f"found at {k}"); }
println(xs.position(|x| x == "z") ?? 99);
if !xs.is_empty() { println(f"{xs.len()} items"); }
```

```output
found at 1
99
2 items
```

## 錯誤

可能失敗的函式會標明 `throws`，每次呼叫它都要用 `try` 標記。`throw "message"` 擲回一個單純的錯誤；`try .. catch` 處理錯誤，`try?` 則把失敗轉成 `none`。

```nabi
fn parse_port(s: str) throws -> i64 {
    let n = try s.parse::<i64>();
    if n > 65535 { throw "port out of range"; }
    n
}

println(try parse_port("8080") catch _ { 0 });
let p = try parse_port("99999") catch e {
    println(f"error: {e}");
    0
};
println(p);
println(try? parse_port("x") ?? -1);
```

```output
8080
error: port out of range
0
-1
```

`int("42")` 寫成 `try "42".parse::<int>()`，`float(s)` 寫成 `try s.parse::<f64>()`。

## 類別就是結構、trait 和 `impl`

結構持有欄位，`impl` 區塊持有方法，`#[derive(..)]` 會替你寫好 `__eq__`、`__repr__` 和 `__hash__`。會修改自身值的方法接受 `ref self`。

```nabi
#[derive(Debug, PartialEq)]
struct Point { x: f64; y: f64; }

impl Point {
    fn new(x: f64, y: f64) -> Point { Point { x: x, y: y } }
    fn norm(self) -> f64 { (self.x * self.x + self.y * self.y).sqrt() }
    fn scale(ref self, k: f64) { self.x *= k; self.y *= k; }
}

let p = Point::new(3.0, 4.0);
println(p.norm());
p.scale(2.0);
println(f"{p:?}");
println(p == Point { x: 6.0, y: 8.0 });
```

```output
5
Point { x: 6.0, y: 8.0 }
true
```

Nabi 沒有繼承。trait 列出一個型別提供的方法，需要的地方可以附上預設主體；`any Shape` 可以持有任何實作了它的型別。

```nabi
trait Shape {
    fn area(self) -> f64;
    fn describe(self) -> str { f"area {self.area():.1}" }
}
struct Square { side: f64; }
struct Circle { r: f64; }
impl Shape for Square { fn area(self) -> f64 { self.side * self.side } }
impl Shape for Circle { fn area(self) -> f64 { f64::PI * self.r * self.r } }

let shapes: List<any Shape> = [Square { side: 2.0 }, Circle { r: 1.0 }];
for s in shapes { println(s.describe()); }

fn loud(x) -> str { x.to_uppercase() }
println(loud("hi"));
```

```output
area 4.0
area 3.1
HI
```

## 值、控制代碼與數字

```nabi
struct P { x: i32; }
let a = [1];
let b = a;
b.push(2);
println(f"{a:?}");
let p = P { x: 1 };
let q = p;
q.x = 9;
println(f"{p.x} {q.x}");

println(7 / 2);
println((-7).div_floor(2));
println(-7 %% 2);
println(7.0 / 2.0);
println((7 as f64) / 2.0);
println(2 ** 10);
```

```output
[1, 2]
1 9
3
-4
1
3.5
3.5
1024
```

`**` 是乘冪運算子，和 Python 一樣：`2 ** 10`，而 `-x ** 2` 是 `-(x ** 2)`（[第 21 章](21-arrays-and-numerics.html)）。整數底數接受 `u32` 指數，所以 `2 ** -1` 是錯誤，錯誤訊息會告訴你改寫成 `2.0 ** -1.0`。

用重複建立的網格是兩者不同的地方。Python 的 `[[0] * 3] * 2` 會在兩個位置放入同一個列串列，所以 `g[0][1] = 5` 會同時改到兩列。Nabi 的 `[[0; 3]; 2]` 只對列求值一次，再複製到每個位置（固定長度陣列，[第 2 章](02-values-and-types.html)），所以 `g[0][1] = 5` 只會改到第一列。持有串列的元素會用 `clone()` 複製：`[List::<i32>::new(); 3]` 是三個各自獨立的串列。

## `with` 就是 `using`

`using` 繫結一個值，並在其作用域結束時關閉它，就像 `with` 區塊會關閉它的檔案。

```nabi
struct Conn { name: str; }
impl Drop for Conn {
    fn drop(ref self) { println(f"closed {self.name}"); }
}
fn work() {
    using c = Conn { name: "db" };
    println(f"using {c.name}");
}
work();
println("done");
```

```output
using db
closed db
done
```

## 檔案與文字

```nabi
use std::{env, fs, process};

let path = env::temp_dir() / f"pynabi_{process::id()}.txt";
try fs::write(path, "one\ntwo\n");
let text = try fs::read_to_str(path);
for line in text.lines() { println(line.to_uppercase()); }
try fs::remove_file(path);

println(42.to_str() + "px");
println(f"{3.14159:.2}");
println("ab".repeat(3));
println("a,b".split(",").to_list().len());
```

```output
ONE
TWO
42px
3.14
ababab
2
```

## Nabi 中的 Python 慣用寫法

| Python | Nabi |
|---|---|
| `print(x)` / `print(x, end="")` | `println(x)` / `print(x)` |
| `print(a, b, sep=", ")` | `println(a, b, sep: ", ")` |
| `f"{x!r}"` | `f"{x:?}"` |
| `a and b`, `a or b`, `not a` | `a && b`, `a \|\| b`, `!a` |
| `x not in xs` | `x !in xs` |
| `x is None`, `x is not None` | `x == none`, `x != none` |
| `None`, `True`, `False` | `none`, `true`, `false` |
| `a if c else b` | `if c { a } else { b }` |
| `elif` | `else if` |
| `def f(x):` / `pass` | `fn f(x) { }`（空區塊不需要 `pass`） |
| `lambda x: x + 1` | `\|x\| x + 1` |
| `if xs:` | `if !xs.is_empty()` |
| `len(xs)`, `sorted(xs)`, `sum(xs)` | `xs.len()`, `xs.sorted()`, `xs.sum()` |
| `min(a, b)`, `abs(x)`, `round(x)` | `a.min(b)`, `x.abs()`, `x.round()` |
| `str(x)` | `x.to_str()` 或 `f"{x}"` |
| `int(s)`, `float(s)` | `try s.parse::<int>()`, `try s.parse::<f64>()` |
| `", ".join(xs)` | `xs.join(", ")` |
| `s * 3` | `s.repeat(3)` |
| `[[0] * 3] * 2`（各列共用） | `[[0; 3]; 2]`（每列各為一份複本） |
| `range(n)`, `range(a, b)` | `0..n`, `a..b` |
| `for i, x in enumerate(xs)` | `for (i, x) in xs.enumerate()` |
| `for k, v in d.items()` | `for (k, v) in d` |
| `[f(x) for x in xs if c]` | `xs.filter(\|x\| c).map(\|x\| f(x)).to_list()` |
| `sorted(xs, key=f, reverse=True)` | `xs.sorted_by_key(f)`, `xs.sorted(reverse: true)` |
| `d.get(k, 0)`、`defaultdict(int)` 和 `d[k] += 1` | `d[k, default: 0]`, `d[k, default: 0] += 1` |
| `Counter(xs)`, `Counter(xs).most_common(n)` | `xs.counts()`, `xs.counts().most_common(n)` |
| `print(f"{x=}")`、`breakpoint()` 式的傾印 | `f"{x=}"`, `dbg(x)` |
| `x ** 2`, `n // 2` | `x ** 2`, `n.div_floor(2)` |
| `raise ValueError("m")` / `except E as e` | `throw "m"` / `catch e: E` |
| `with open(p) as f:` | `using f = try File::open(p);` |
| `assert x == 2, "m"` | `assert(x == 2, "m");` |
| 帶 `self` 的 `class C:` | `struct C { .. }` 和 `impl C { fn m(self) { .. } }` |
| `C("h", 1)` | `C { host: "h", port: 1 }` |
| `import m` / `from m import x` | `use m;` / `use m::x;` |

以下每一種 Python 寫法，如果直接寫在 Nabi 裡，都會讓建置停在一個錯誤上，錯誤的 help 會顯示應該改寫成的 Nabi 寫法：`and` / `or` / `not`、`is not`、`a if c else b`、`lambda`、推導式（help 會寫出迭代器鏈）、`elif`、`except T as e`、`with open(..) as f`、`assert c, m`、`for i, x in ..`、`""".."""`、`import math` 和 `from m import x`、`pass`、`f(end="")`、`f"{x!r}"`、當成函式呼叫的內建函式（`len(xs)`、`str(x)`、`int(s)`）、`Counter(xs)` 和 `xs.most_common(n)`、`C("h", 1)`、`", ".join(xs)`、`s * 3`、真值性（`if xs:`），以及沒有 `self` 的方法。

## Nabi 中的 Python 模組

| Python | Nabi |
|---|---|
| `math` | 數字上的方法（`x.sqrt()`、`x.sin()`、`x.floor()`、`x.is_close(y)`、`a.gcd(b)`）和 `f64::PI`；`num::fsum`、`num::comb`、`num::perm` |
| `os`, `sys` | `std::env`（`env::args()`、`env::var(..)`）、`std::process`（`process::exit(..)`、`process::id()`） |
| `os.path`, `pathlib` | `std::path::Path`（`Path::new("a") / "b"`）、`std::fs` |
| `open`, `shutil` | `std::fs`（`fs::read_to_str`、`fs::write`、`File::open`） |
| `subprocess` | `std::process`（取得輸出用 `process::run("git", ["status"])`，管線用 `Command`） |
| `glob` | `fs::glob("*.txt")` |
| `random` | `std::random`（`random::range(1..7)`、`random::choice(xs)`、`random::shuffle(xs)`；`Xoshiro256PlusPlus::seed(42)` 可取得指定種子的 `Rng`，提供 `normal`、`gamma`、`poisson`、`weighted` 等） |
| `cmath`, `complex` | `Complex` 和 `i` 後綴：`3.0 + 4.0i` |
| `numpy.linspace`, `numpy.arange` | `num::linspace(0.0, 1.0, 5)`、`num::arange(0.0, 1.0, 0.25)`；n 維資料用 `Array`（[第 21 章](21-arrays-and-numerics.html)） |
| `numpy` 歸約（`a.sum(axis=0)`、`np.nanmax`、`np.where`、`np.allclose`） | `a.sum_along(0)`、`a.nanmax()`、`array::where(m, a, b)`、`a.all_close(b)` |
| `numpy.linalg`, `scipy.sparse`, `scipy.stats`, `scipy.signal`, `scipy.optimize`, `pandas`, `matplotlib` | 套件 `linalg`、`sparse`、`stats`、`signal`、`optimize`、`table`、`plot` 等（[第 22 章](22-science.html)） |
| `time` | `core::time`（`Instant::now()`、`Duration`） |
| `datetime`, `zoneinfo` | `std::time`（`try Date::new(2024, 1, 31)`、`DateTime`、`Zoned::now(TimeZone::local())`、`try TimeZone::get("Europe/Paris")`、`d.format("%Y-%m-%d")`、`try Date::parse_with(..)`） |
| `threading` | `std::thread`（`thread::spawn(..)`、`join`） |
| `asyncio` | 內建的 `async fn` 和 `await` |
| `collections` | 內建的 `Dict`、`Set`、`Deque`、`BinaryHeap`；`Counter` 是 `xs.counts()` 加上 `most_common(n)`，`defaultdict` 是 `d[k, default: v]` |
| `sortedcontainers` | `core::collections` 中的 `SortedDict`、`SortedSet` |
| `itertools`, `functools` | 迭代器方法：`zip`、`chain`、`windows`、`chunks`、`pairwise`、`combinations`、`permutations`、`group_by`、`fold`、`reduce`、`scan` |
| `json` | `json` 套件：`nabi add json` |
| `input()` | `try io::input("prompt")` |
| `unittest`, `pytest` | `#[test]` 函式和 `nabi test` |
| `black`, `ruff` | `nabi fmt`、`nabi lint` |
| `pip`, `venv` | `nabi add`、`nabi env` |

`re` 對應官方套件 `regex`（`nabi add regex`）。`logging` 和 `argparse` 目前還沒有對應的套件，預計會以 `log` 和 `cli` 套件提供。
