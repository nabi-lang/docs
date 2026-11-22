# 2. 值与类型

本章介绍用于声明变量的 `let`、数字和字符串等基本类型，以及用来组合值的元组和列表。Nabi 是静态类型语言，但大多数类型都能推断出来，所以你很少需要写出类型。

## `let` 与重新赋值

`let` 创建一个新变量；要写入已有的变量，使用不带 `let` 的 `=`。变量默认可变（没有 `mut` 标记）。

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

由于声明和重新赋值在语法上不同，拼写错误永远不会悄悄创建一个新变量。给从未声明过的名字赋值是编译错误。

```text check=checks/02-undeclared.nabi
error[E0057]: `coutn` is not declared; a variable is created only by `let`
  --> main.nabi:2:1
   |
 2 | coutn = count + 1;
   | ^^^^^
   |
   = help: did you mean `count`? To declare a new variable, write `let coutn = count + 1;`
```

还有几条规则：

- 必须提供初始值。`let x;` 是错误。如果暂时没有值，就显式写出“没有值”：`let x: i32 | none = none;`。
- 变量的类型在声明时确定，之后不会改变。把字符串赋给 `i32` 变量是错误。
- 对已有的名字再写一次 `let`，会创建一个遮蔽旧变量的新变量，无论是在同一作用域还是内层块中。它的初始值可以读取旧变量，类型也可以不同，因此一个值可以在每一步处理中保持同一个名字。
- 关键字不能用作名字。其中六个保留给将来使用，目前还没有语法：`macro`、`defer`、`dyn`、`move`、`gen` 和 `do`。如果一定要把关键字用作名字，在前面加 `r#`（`r#move`、`r#type`）；错误信息会给出这个建议。

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
let line = line.trim();          // 读取之前的 `line`
let line: i32 = try! line.parse();
println(f"{line + 1}");
```

```output
43
```

要改变变量的值，使用不带 `let` 的赋值。重新声明不会改动旧变量（[5.1.4](../spec/refs-lang/005-modules/501-variable-declaration.html#514-在同一作用域中重新声明)）：已打开的文件不会因此关闭，所以要先关闭它，或者把它移入新值，例如 `let f = BufReader::new(f);`（第 13 章）。

## 数值类型与字面量

整数类型有 `i8` `i16` `i32` `i64` `isize` 以及无符号的 `u8` `u16` `u32` `u64` `usize`；浮点数类型有 `f32` `f64`，`f16` / `bf16` 则为大数组存储 16 位浮点数（[数组与数值计算](21-arrays-and-numerics.html#16-位浮点数f16-与-bf16)）。带类型名后缀的字面量就是该类型；不带后缀时，它遵循**期望类型**。用 `let` 绑定的字面量，其类型取决于该变量在函数中之后的用法；只有在没有任何线索时，整数才默认为 `i64`，浮点数默认为 `f64`。

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

由于类型来自用法，你很少需要写出它：

```nabi
fn count_big(xs: List<i32>) -> i32 {
    let c = 0;                 // i32：函数返回它
    for x in xs { if x > 100 { c += 1; } }
    c
}
let hits = Atomic::new(0);     // 计数器的类型来自下面的 fetch_add
let step: u8 = 2;
hits.fetch_add(step);
println(f"{count_big([5, 500, 900])} {hits.load()}");
```

```output
2 2
```

Nabi 仍然从不做转换：一个变量只有一种类型。把 `let t = 0;` 当作 `f64` 使用是错误，错误信息会让你在声明处写成 `let t = 0.0;`。

类型参数声明的默认值也算在内。对于 `struct Vec3<T = f32>`，`Vec3::new(1.0, 2.0, 3.0)` 中的字面量默认为 `f32`，所以这个向量是 `Vec3<f32>`；期望类型、之后的用法或有类型的实参仍然优先（[泛型](07-generics.html#默认值)）。

`_` 是数字分隔符，`0x` / `0b` / `0o` 分别表示十六进制 / 二进制 / 八进制。前导零不会让数字变成八进制，`0755` 是编译错误：八进制写 `0o755`，十进制写 `755`。整数除法向零截断。超出类型范围的字面量（`let x: u8 = 256;`）是编译错误。

## 没有隐式转换：`as`

Nabi 从不自动混用不同的数值类型。把 `i32` 和 `i64` 相加是错误。

```text check=checks/02-mixed-int.nabi
error[E0054]: operator `+` cannot be applied to `i32` and `i64`
  --> main.nabi:3:12
   |
 3 | println(f"{a + b}");
   |            ^^^^^
   |
   = help: Nabi never converts numbers implicitly; convert one side: `(a as i64) + b`
```

这是有意为之，为的是防止悄无声息地丢失精度。转换要显式写出。`as` 总是成功，但可能截断或改变值；`.into()` 只允许无损转换：在每个目标上都保留所有值的拓宽转换，例如 `i32` 到 `i64`、`u8` 到 `isize` 或 `u32` 到 `f64`（完整的表见[标准库 2.11.8](../spec/refs-standard-libraries/200-core/211-conversion.html#2118-from-实现矩阵原始类型之间)）。

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

`2.9 as i32` 丢弃小数部分，`300 as u8` 只保留低 8 位。对于不能丢失任何信息的转换，`.try_into::<T>()` 会检查范围，超出时抛出错误；[第 10 章](10-errors.html)介绍如何处理错误。

数字永远不能强制转换为 `bool`：`n as bool` 是错误。写出你想要的比较，`n != 0`（浮点数写 `x != 0.0`）。

## 整数的陷阱

有几条整数规则会让从 Python、Go 或 JavaScript 转过来的人感到意外。

**`len()` 是 `usize`**。长度和索引都是 `usize`。字面量的类型来自它的用法，所以用 `xs.len()` 去除的 `let total = 6;` 也是 `usize`；但类型已经确定的值（`i64` 字段、参数或标注）不会被转换：当 `total` 是 `i64` 时，`total / xs.len()` 是错误。用 `as` 转换其中一边。

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

**`/` 和 `%` 向零截断**。`-7 / 2` 是 `-3`，`-7 % 2` 是 `-1`：余数的符号与左操作数相同，和 C、Go、Rust 一样。如果想要 Python 的 `%`（结果符号与除数相同），使用 `%%`（向下取整的取模）。它很适合让索引循环回绕。

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

**运算符的结合方式与 Rust 相同**。`*` `/` `%` 比 `+` `-` 结合得更紧，`+` `-` 又比移位 `<<` `>>` 更紧，然后依次是 `&`、`^`、`|`，它们都比比较运算更紧。因此 `1 + 2 << 3` 是 `(1 + 2) << 3`，`x & mask == 0` 是 `(x & mask) == 0`，而不是 C 中的 `x & (mask == 0)`。幂运算符 `**` 结合得最紧，比前置的 `-` 还紧，并且右结合，和 Python 一样：`-2 ** 2` 是 `-4`，`2 ** 3 ** 2` 是 `2 ** 9`。整数底数接受 `u32` 指数。

```nabi
println(f"{1 + 2 << 3} {6 & 3 == 2}");
println(f"{2 ** 10} {-2 ** 2} {2 ** 3 ** 2} {1.5 ** 2.0}");
```

```output
24 true
1024 -4 512 2.25
```

**溢出会触发 panic**。超出定宽整数的范围时，无论是调试构建还是 `--release`，都会触发 panic，所以错误的值永远不会悄悄混过去。

```nabi build
let x: i32 = 2147483647;
println(f"{x + 1}");
```

```text
$ nabi run --release
thread 'main' panicked at src/main.nabi:2:12:
attempt to add with overflow
```

在紧凑的整数循环中，这些检查会带来几个百分点的开销。需要找回这部分性能的程序，可以用 `-C overflow-checks=no` 关闭检查，或者在 `nabi.nabion` 中针对某个构建配置写 `profile: { "release": { overflow_checks: false } }`；此时溢出会回绕。预期会溢出时，用 `checked_add`（得到 `none`）、`wrapping_add` 或 `saturating_add` 写明你想要的行为，或者使用永远不会溢出的 `int`（见下文）。

```nabi
let x: i32 = 2147483647;
println(f"{x.checked_add(1):?} {x.wrapping_add(1)} {x.saturating_add(1)}");
```

```output
none -2147483648 2147483647
```

## bool、char、str

`bool` 的值是 `true` / `false`，用 `&&`、`||` 和 `!` 组合。同方向的比较可以连写：`0 <= x < 10` 就是 `0 <= x && x < 10`，`x` 只求值一次（`a < b > c` 是错误）。`x in xs` 和 `x !in xs` 检查元素是否属于列表、集合、字典（检查键）、字符串或区间。`char` 是单个 Unicode 字符，用单引号书写；`b'A'` 是该 ASCII 字符的字节，类型为 `u8`。`str` 是 UTF-8 字符串，用双引号书写。

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

韩文音节 `'가'`（U+AC00，码点 44032）说明 `char` 能容纳任何 Unicode 字符，而不只是 ASCII。`len()` 计算字节数；需要字符数时用 `char_count()`。`r"..."` 是不处理转义的原始字符串。其他 `str` 方法（`trim`、`split`、`replace`、`parse` 等）见[标准库文档](../../std/core/str.html)。

## 元组、数组与列表

**元组**把几个不同类型的值组合在一起。用 `.0`、`.1` 访问，或者用 `let` 解构。

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

`[...]` 字面量默认是 **`List`**。它是可增长的列表，引用同一个列表的每个变量都能看到对它的修改。`List` 不能用 `{}` 打印，请使用 Debug 格式 `{:?}`。

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

如果变量的类型是 `[T; N]`，同样的字面量就成为定长**数组**。它的大小在编译时已知，适合矩阵、缓冲区等形状固定的数据。

```nabi
let grid: [[i32; 2]; 2] = [[1, 2], [3, 4]];
println(f"{grid:?} {grid[1][0]} {grid.len()}");
```

```output
[[1, 2], [3, 4]] 3 2
```

`[v; n]` 用一个值构造含 `n` 个元素的数组。Nabi 只对 `v` 求值一次，然后给每个元素一份独立的副本，所以修改一个元素永远不会改变另一个，即使元素是列表也是如此：

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

数字和其他普通值按原样复制。持有集合的元素用 `clone()` 复制，所以其类型必须是 `Clone`。不满足的类型，或者只有一个所有者的值（例如已打开的 `File`），会导致编译错误；错误的帮助信息会写出 `array::from_fn::<T, n>(|_| ..)`，它为每个元素构造各自的值。

空列表的元素类型来自之后的用法：执行 `let xs = []; xs.push(1u8);` 之后，`xs` 是 `List<u8>`。如果函数中没有任何线索，就写出类型：`let xs: List<i32> = [];`。

## 大整数：`int`

`i32` 这样的定宽类型速度快，但有范围限制。需要不必担心范围的精确整数时，使用 `int`。较小的值像定宽整数一样存储，随着增长自动切换为大整数。`int` 只出现在显式写出该类型的地方，与定宽类型混用时同样需要显式转换。

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

`int` 可以用 `big.to_f64()` 转为 `f64`，结果舍入到最接近的值。舍入不是无损的，所以 `into()` 和 `as f64` 对它无法编译；值超过 `f64::MAX` 时，`try f64::try_from(big)` 会抛出错误，而不是给出 `inf`。

> [!NOTE]
> 进阶：`int` 还有 `isqrt`、`modpow`、`gcd`、`lcm` 等数论辅助函数，列在 [`int` API](../../std/core/num/bigint.html) 中。

## 常量

`const` 是在编译时确定的值。与 `let` 不同，它必须写出类型，并且可以出现在模块的顶层。值在每个使用处内联，所以每次使用都是一个新值：修改它（对 `const` 列表调用 `XS.push(4)`）是错误；请先把它复制到变量中（`let xs = XS;`）。`const` 只能读取常量：在有 `let n` 的情况下写 `const HALF: i32 = n / 2;` 是错误，错误信息会让你写成 `let half = n / 2;`。

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

## 小结

- `let` 声明，`=` 重新赋值。对同一个名字再写 `let` 会创建新变量，并且必须提供初始值。
- 不带后缀的字面量遵循期望类型，默认为 `i64` / `f64`，或者它所填入的类型参数的默认值。
- 数值类型之间没有自动转换。使用 `as`、`.into()` 或 `.try_into::<T>()`。
- `len()` 和索引是 `usize`；要与 `i64` 混用，用 `as` 转换。
- `/` 和 `%` 向零截断；`%%` 是向下取整的取模。除非关闭溢出检查，否则溢出在所有构建中都会触发 panic。
- `[...]` 是 `List`；类型为 `[T; N]` 时是定长数组。`[v; n]` 是元素为 `v` 的独立副本的定长数组。用 `{:?}` 打印 `List`。
- `int` 是无界整数；`const` 是编译时常量。

## 延伸阅读

- [变量声明](../spec/refs-lang/005-modules/501-variable-declaration.html)，[`const`](../spec/refs-lang/005-modules/502-const.html)
- [数值字面量](../spec/refs-lang/001-lexical/106-numeric-literals.html)，[字符串字面量](../spec/refs-lang/001-lexical/107-string-literals.html)，[字符字面量](../spec/refs-lang/001-lexical/108-character-literals.html)
- [强制转换与转换](../spec/refs-lang/002-types/214-cast-and-conversion.html)，[类型表达式](../spec/refs-lang/002-types/201-type-expressions.html)
- [数值类型与 `int`](../spec/refs-standard-libraries/200-core/201-numbers.html)，[格式化输出](../spec/refs-standard-libraries/200-core/210-fmt.html)
- 标准库 API：[`str`](../../std/core/str.html)，[`List`](../../std/core/collections/list.html)
- 下一章：[3. 函数](03-functions.html)
