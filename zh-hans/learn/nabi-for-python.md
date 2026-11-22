# 写给 Python 用户的 Nabi

这是一份给 Python 程序员的对照图：哪些写法可以照搬，哪些写法在 Nabi 中不同，以及 Python 标准库的各个模块在 Nabi 中对应什么。下面每个代码块都是完整的程序，用 `nabi run file.nabi` 运行即可。本指南的各章会完整讲解每一部分。

## 可以照搬的写法

- 文件像脚本一样从上到下运行，不需要 `main`：`nabi file.nabi`、`#!/usr/bin/env nabi` 行，依赖也直接声明在文件中（`nabi add json file.nabi`）。
- f-string，格式说明符也相同：`f"{x:.2f}"`、`f"{name:>10}"`、`f"{x=}"`。
- 支持负索引和步长的切片：`xs[-1]`、`xs[1:3]`、`xs[::-1]`。
- 默认参数和具名参数：`fn f(a: i32, sep: str = " ")`，调用写作 `f(1, sep: ", ")`。
- `in`、链式比较（`0 <= i < n`）、元组解包和交换（`(a, b) = (b, a)`）。
- `try` / `catch` / `finally`、生成器（`yield fn`）、`async` / `await`、带守卫的 `match`。
- 大整数：`int` 没有大小限制。
- REPL（`nabi repl`）和 Jupyter 内核（`nabi jupyter`）。

## 最初的意外

- `print` 不加换行，`println` 才加。两者都接受多个值和 `sep:`。
- 每个值都有编译器知道的类型。没有写类型的参数是泛型的：它接受任何具有函数体所调用方法的类型，也就是在编译时检查的鸭子类型。
- 块用 `{ }` 而不是缩进，新名字用 `let` 声明。
- 没有真值判断（truthiness）：条件必须是 `bool`。写 `!xs.is_empty()`、`n != 0` 或 `x != none`。
- 缺失的值是 `none`，可能持有它的类型写作 `T | none`。
- 整数上的 `/` 向零取整（`7 / 2` 是 `3`）；向下取整除法是 `n.div_floor(2)`，`%%` 是向下取整的余数。`//` 开始一条注释。
- 整数超出其类型的范围时触发 panic。没有后缀的整数是 `i64`，除非它的用法给了它别的类型；`int` 可以容纳任意大小。
- 结构体是值：`let q = p;` 会复制。`List`、`Dict` 和 `Set` 是共享的句柄，与 Python 相同。
- `str` 切片按字节而不是字符计数；用 `s.chars()` 逐个遍历字符。

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

## 控制流

`if` 是表达式，所以它也可以代替 `a if c else b`。`elif` 写作 `else if`，`for i in range(n)` 写作 `for i in 0..n`。

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

`and`、`or` 和 `not` 分别是 `&&`、`||` 和 `!`；`not in` 是 `!in`。

## 推导式就是迭代器链

Nabi 没有列表、字典或集合推导式。集合类型都有迭代器方法，由 `to_list()`、`to_dict()` 和 `to_set()` 收集结果。`sum`、`min`、`max`、`sorted`、`any` 和 `all` 也都是方法。

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

Python 写 `lambda x: x + 1` 的地方，闭包写作 `|x| x + 1`。

## 用 `none` 代替 `None`

可能什么也找不到的函数返回 `T | none`。`??` 给出默认值，`x == none` 用于判断，`if let` 取出值。在 `x != none` 之后，或在 `if x == none { return; }` 下方，`x` 就是值本身。

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

## 错误

可能失败的函数要写 `throws`，对它的每次调用都用 `try` 标出。`throw "message"` 抛出一个普通错误；`try .. catch` 处理错误，`try?` 把失败变成 `none`。

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

`int("42")` 写作 `try "42".parse::<int>()`，`float(s)` 写作 `try s.parse::<f64>()`。

## 类就是结构体、trait 和 `impl`

结构体保存字段，`impl` 块保存方法，`#[derive(..)]` 负责写出 `__eq__`、`__repr__` 和 `__hash__`。修改自身值的方法接收 `ref self`。

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

没有继承。trait 列出一个类型提供的方法，可以在需要的地方带上默认实现，`any Shape` 可以持有任何实现了它的类型。

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

## 值、句柄和数字

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

`**` 是幂运算符，与 Python 相同：`2 ** 10`，`-x ** 2` 是 `-(x ** 2)`（[第 21 章](21-arrays-and-numerics.html)）。整数底数要求 `u32` 指数，所以 `2 ** -1` 是一个错误，错误信息会提示改写为 `2.0 ** -1.0`。

两者的区别体现在通过重复构造的网格上。Python 的 `[[0] * 3] * 2` 把同一个行列表放进两个位置，所以 `g[0][1] = 5` 会同时改变两行。Nabi 的 `[[0; 3]; 2]` 只对行求值一次，再把它复制到每个位置（定长数组，[第 2 章](02-values-and-types.html)），所以 `g[0][1] = 5` 只改变第一行。持有列表的元素用 `clone()` 复制：`[List::<i32>::new(); 3]` 是三个独立的列表。

## `with` 就是 `using`

`using` 绑定一个值，并在其作用域结束时关闭它，就像 `with` 块会关闭文件一样。

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

## 文件和文本

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

## Nabi 中的 Python 惯用写法

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
| `def f(x):` / `pass` | `fn f(x) { }`（空块不需要 `pass`） |
| `lambda x: x + 1` | `\|x\| x + 1` |
| `if xs:` | `if !xs.is_empty()` |
| `len(xs)`, `sorted(xs)`, `sum(xs)` | `xs.len()`, `xs.sorted()`, `xs.sum()` |
| `min(a, b)`, `abs(x)`, `round(x)` | `a.min(b)`, `x.abs()`, `x.round()` |
| `str(x)` | `x.to_str()` 或 `f"{x}"` |
| `int(s)`, `float(s)` | `try s.parse::<int>()`, `try s.parse::<f64>()` |
| `", ".join(xs)` | `xs.join(", ")` |
| `s * 3` | `s.repeat(3)` |
| `[[0] * 3] * 2`（各行共享） | `[[0; 3]; 2]`（每行一个副本） |
| `range(n)`, `range(a, b)` | `0..n`, `a..b` |
| `for i, x in enumerate(xs)` | `for (i, x) in xs.enumerate()` |
| `for k, v in d.items()` | `for (k, v) in d` |
| `[f(x) for x in xs if c]` | `xs.filter(\|x\| c).map(\|x\| f(x)).to_list()` |
| `sorted(xs, key=f, reverse=True)` | `xs.sorted_by_key(f)`, `xs.sorted(reverse: true)` |
| `d.get(k, 0)`、`defaultdict(int)` 和 `d[k] += 1` | `d[k, default: 0]`, `d[k, default: 0] += 1` |
| `Counter(xs)`, `Counter(xs).most_common(n)` | `xs.counts()`, `xs.counts().most_common(n)` |
| `print(f"{x=}")`、`breakpoint()` 式的转储 | `f"{x=}"`, `dbg(x)` |
| `x ** 2`, `n // 2` | `x ** 2`, `n.div_floor(2)` |
| `raise ValueError("m")` / `except E as e` | `throw "m"` / `catch e: E` |
| `with open(p) as f:` | `using f = try File::open(p);` |
| `assert x == 2, "m"` | `assert(x == 2, "m");` |
| 带 `self` 的 `class C:` | `struct C { .. }` 和 `impl C { fn m(self) { .. } }` |
| `C("h", 1)` | `C { host: "h", port: 1 }` |
| `import m` / `from m import x` | `use m;` / `use m::x;` |

在 Nabi 中写下以下任何一种 Python 形式，构建都会停在一个错误上，错误的帮助信息会给出应该写的 Nabi 形式：`and` / `or` / `not`、`is not`、`a if c else b`、`lambda`、推导式（帮助信息会写出对应的迭代器链）、`elif`、`except T as e`、`with open(..) as f`、`assert c, m`、`for i, x in ..`、`""".."""`、`import math` 和 `from m import x`、`pass`、`f(end="")`、`f"{x!r}"`、当作函数调用的内置函数（`len(xs)`、`str(x)`、`int(s)`）、`Counter(xs)` 和 `xs.most_common(n)`、`C("h", 1)`、`", ".join(xs)`、`s * 3`、真值判断（`if xs:`），以及不带 `self` 的方法。

## Nabi 中的 Python 模块

| Python | Nabi |
|---|---|
| `math` | 数字上的方法（`x.sqrt()`、`x.sin()`、`x.floor()`、`x.is_close(y)`、`a.gcd(b)`）和 `f64::PI`；`num::fsum`、`num::comb`、`num::perm` |
| `os`, `sys` | `std::env`（`env::args()`、`env::var(..)`）、`std::process`（`process::exit(..)`、`process::id()`） |
| `os.path`, `pathlib` | `std::path::Path`（`Path::new("a") / "b"`）、`std::fs` |
| `open`, `shutil` | `std::fs`（`fs::read_to_str`、`fs::write`、`File::open`） |
| `subprocess` | `std::process`（获取输出用 `process::run("git", ["status"])`，管道用 `Command`） |
| `glob` | `fs::glob("*.txt")` |
| `random` | `std::random`（`random::range(1..7)`、`random::choice(xs)`、`random::shuffle(xs)`；`Xoshiro256PlusPlus::seed(42)` 得到一个带种子的 `Rng`，提供 `normal`、`gamma`、`poisson`、`weighted` 等） |
| `cmath`, `complex` | `Complex` 和 `i` 后缀：`3.0 + 4.0i` |
| `numpy.linspace`, `numpy.arange` | `num::linspace(0.0, 1.0, 5)`、`num::arange(0.0, 1.0, 0.25)`；多维数据用 `Array`（[第 21 章](21-arrays-and-numerics.html)） |
| `numpy` 归约（`a.sum(axis=0)`、`np.nanmax`、`np.where`、`np.allclose`） | `a.sum_along(0)`、`a.nanmax()`、`array::where(m, a, b)`、`a.all_close(b)` |
| `numpy.linalg`, `scipy.sparse`, `scipy.stats`, `scipy.signal`, `scipy.optimize`, `pandas`, `matplotlib` | `linalg`、`sparse`、`stats`、`signal`、`optimize`、`table`、`plot` 等包（[第 22 章](22-science.html)） |
| `time` | `core::time`（`Instant::now()`、`Duration`） |
| `datetime`, `zoneinfo` | `std::time`（`try Date::new(2024, 1, 31)`、`DateTime`、`Zoned::now(TimeZone::local())`、`try TimeZone::get("Europe/Paris")`、`d.format("%Y-%m-%d")`、`try Date::parse_with(..)`） |
| `threading` | `std::thread`（`thread::spawn(..)`、`join`） |
| `asyncio` | 内置的 `async fn` 和 `await` |
| `collections` | 内置的 `Dict`、`Set`、`Deque`、`BinaryHeap`；`Counter` 是带 `most_common(n)` 的 `xs.counts()`，`defaultdict` 是 `d[k, default: v]` |
| `sortedcontainers` | `core::collections` 中的 `SortedDict`、`SortedSet` |
| `itertools`, `functools` | 迭代器方法：`zip`、`chain`、`windows`、`chunks`、`pairwise`、`combinations`、`permutations`、`group_by`、`fold`、`reduce`、`scan` |
| `json` | `json` 包：`nabi add json` |
| `input()` | `try io::input("prompt")` |
| `unittest`, `pytest` | `#[test]` 函数和 `nabi test` |
| `black`, `ruff` | `nabi fmt`, `nabi lint` |
| `pip`, `venv` | `nabi add`, `nabi env` |

`re` 对应官方的 `regex` 包（`nabi add regex`）。`logging` 和 `argparse` 目前还没有对应物，计划作为 `log` 和 `cli` 包提供。
