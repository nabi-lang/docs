# 4. 控制流

本章介绍分支（`if`、`match`）、循环（`while`、`for`、`loop`）和解构。关键在于，Nabi 中的 `if`、`match` 和 `loop` 都是产生值的表达式。

## `if` 是表达式

`if` 的条件不需要括号，主体总是写在花括号中。每个分支的最后一个表达式成为整个 `if` 的值，所以没有单独的三元运算符。

```nabi
let temp = 23;
let feel = if temp >= 28 {
    "hot"
} else if temp >= 15 {
    "mild"
} else {
    "cold"
};
println(f"{temp} degrees: {feel}");
```

```output
23 degrees: mild
```

作为值使用时，所有分支的类型必须相同，或者都属于某个分支的和类型（当 `x: i32?` 时，`if c { x } else { none }` 的类型是 `i32?`）。要混用不同的类型，给接收的变量标注 `i32 | str` 这样的和类型。

## `match` 与模式

`match` 从上到下把值与各个模式比较，执行第一个匹配的分支。可以使用字面量、用 `|` 连接的多个模式、区间（`4..=9`）、绑定（`k @ pattern`）、守卫（`if condition`）和通配符 `_`。

```nabi
fn describe(n: i32) -> str {
    match n {
        0 => "zero",
        1 | 2 | 3 => "small",
        k @ 4..=9 => f"digit {k}",
        k if k < 0 => "negative",
        _ => "large",
    }
}

for n in [0, 2, 7, -5, 42] {
    println(f"{n}: {describe(n)}");
}
```

```output
0: zero
2: small
7: digit 7
-5: negative
42: large
```

编译器会检查每种情况是否都已处理。`i32` 的值多得无法一一列出，所以最后必须有 `_`。元组也可以用模式解构和匹配。

```nabi
fn locate(p: (i32, i32)) -> str {
    match p {
        (0, 0) => "origin",
        (x, 0) => f"on the x-axis at {x}",
        (0, y) => f"on the y-axis at {y}",
        (a, b) if a == b => "on the diagonal",
        _ => "somewhere else",
    }
}

println(locate((0, 0)));
println(locate((5, 0)));
println(locate((2, 2)));
println(locate((1, 9)));
```

```output
origin
on the x-axis at 5
on the diagonal
somewhere else
```

模式中单独的名字（`x`、`a`）会绑定该位置上的任何值。所以模式不能通过写出常量名来与常量比较：`LIMIT => ..` 会匹配所有值，因此这样写是错误。请改写成守卫，`n if n == LIMIT => ..`。

列表、切片和定长数组用 `[..]` 模式按元素匹配。`[]` 是空的，`[a, b]` 恰好有两个元素，`..` 表示中间任意数量的元素；`..rest` 把它们绑定为一个切片。

```nabi
fn sum(xs: Slice<i32>) -> i32 {
    match xs {
        [] => 0,
        [first, ..rest] => first + sum(rest),
    }
}

fn ends(xs: List<str>) -> str {
    match xs {
        [] => "nothing",
        [only] => f"just {only}",
        [first, .., last] => f"{first} to {last}",
    }
}

let xs: List<i32> = [1, 2, 3, 4];
println(sum(xs[:]));
println(ends(["a"]));
println(ends(["a", "b", "c"]));
let arr: [i32; 3] = [10, 20, 30];
let [x, y, z] = arr;
println(x + y + z);
```

```output
10
just a
a to c
60
```

编译器也会检查长度：没有 `[first, .., last]` 分支时，`ends` 会漏掉有两个或更多元素的列表，错误信息会用模式 `[_, _, ..]` 指出这一点。普通的 `let` 可以拆开长度已知的定长数组；列表的长度未知，所以要写 `let [first, ..rest] = xs else { .. };` 或者 `match`。

当值可能是几种类型之一（`i32 | str | none`）时，`match` 用 `n: i32` 这样的模式按类型区分。这类模式以及 `is`、`if let` 和 `let else`，将在[下一章](05-sum-types.html)介绍和类型时一并讲解。

## `while` 与 `for`

`while` 在条件为真时一直重复。`for` 从区间、列表或迭代器中逐个取出元素。`a..b` 不包含 `b`；`a..=b` 包含 `b`。区间也可以用于 `in`：`if i in 0..n { .. }` 不需要括号。`continue` 进入下一次迭代，`break` 离开循环。

```nabi
let i = 0;
while i < 3 {
    print(f"{i} ");
    i += 1;
}
println("");

for k in 0..10 {
    if k % 2 == 1 { continue; }
    if k > 6 { break; }
    print(f"{k} ");
}
println("");

for (idx, name) in ["ada", "grace"].enumerate() {
    println(f"{idx}: {name}");
}
```

```output
0 1 2 
0 2 4 6 
0: ada
1: grace
```

`while let` 在模式匹配时一直重复。`pop()` 返回 `T | none`，所以把列表清空的写法很简单。

```nabi
let stack = [1, 2, 3];
while let top = stack.pop() {
    print(f"{top} ");
}
println("");
```

```output
3 2 1 
```

## `loop` 与带值的 `break`

`loop` 一直重复，直到遇到 `break`。用 `break value` 退出时，这个值就是 `loop` 表达式的结果。这适合那些在结束时才确定结果的计算。

```nabi
let n = 27;
let steps = 0;
let total = loop {
    if n == 1 { break steps; }
    n = if n % 2 == 0 { n / 2 } else { 3 * n + 1 };
    steps += 1;
};
println(f"Collatz(27) reaches 1 in {total} steps");
```

```output
Collatz(27) reaches 1 in 111 steps
```

## 标签

要从嵌套的循环中直接结束外层循环，给它加上 `'name:` 标签，然后使用 `break 'name` 或 `continue 'name`。`loop` 和块上的标签也可以把值带出来。

```nabi
let pair = 'search: loop {
    for a in 1..10 {
        for b in a..10 {
            if a * b == 42 { break 'search (a, b); }
        }
    }
    break (0, 0);
};
println(f"{pair:?}");

let limit = 50;
let checked = 'check: {
    if limit > 100 { break 'check -1; }
    limit * 2
};
println(f"{checked}");
```

```output
(6, 7)
100
```

## 解构

`let` 和 `for` 的左边可以是任何总能匹配的模式（不可反驳的模式）。可以拆开嵌套的元组，用 `..` 收集其余部分，或者用 `_` 丢弃。

```nabi
let ((a, b), c) = ((1, 2), 3);
let (first, ..rest) = (10, 20, 30, 40);
let (_, second) = ("skip", "keep");
println(f"{a} {b} {c} {first} {rest:?} {second}");

let points = [(1, 2), (3, 4)];
for (x, y) in points {
    println(f"({x}, {y}) -> {x + y}");
}
```

```output
1 2 3 10 (20, 30, 40) keep
(1, 2) -> 3
(3, 4) -> 7
```

```nabi
let a = 0;
let b = 1;
for _ in 0..10 {
    (a, b) = (b, a + b);
}
println(f"{a} {b}");

let xs = [1, 2, 3];
(xs[0], xs[2]) = (xs[2], xs[0]);
println(f"{xs:?}");
```

```output
55 89
[3, 2, 1]
```

不带 `let` 时，`=` 左边由已有位置组成的元组会给其中每个位置赋值：变量、字段、元素、嵌套元组和 `_`。右边会先完整求值，所以 `(a, b) = (b, a + b)` 不需要临时变量，`(xs[i], xs[j]) = (xs[j], xs[i])` 则交换两个元素。闭包和函数的参数接受同样的模式：`|(k, v)| ..`、`fn f((x, y): (i32, i32))`（[第 3 章](03-functions.html)）。

## 小结

- `if`、`match`、`loop` 和块都是产生值的表达式。各分支的类型必须相同。
- `match` 支持字面量、`|`、区间、`@` 绑定、守卫和切片模式（`[first, ..rest]`），并检查穷尽性。
- `while let` 在模式匹配时一直重复。
- `for x in a..b` / `a..=b` 遍历区间；`xs.enumerate()` 带索引遍历；`x in a..b` 检查是否在区间内。
- `(a, b) = (b, a + b)` 在右边求值之后一次给多个位置赋值。
- `break value` 和 `'label:` 让你跳出嵌套循环并返回结果。

## 延伸阅读

- [`if` 表达式](../spec/refs-lang/003-expressions/310-if-expression.html)，[`match` 表达式](../spec/refs-lang/003-expressions/312-match-expression.html)，[模式](../spec/refs-lang/003-expressions/311-patterns.html)
- [循环表达式](../spec/refs-lang/003-expressions/313-loop-expressions.html)，[区间表达式](../spec/refs-lang/003-expressions/307-range-expressions.html)，[带标签的块](../spec/refs-lang/003-expressions/309-block-expressions.html)
- [解构赋值](../spec/refs-lang/003-expressions/315-destructuring.html)，[`let-else`](../spec/refs-lang/003-expressions/316-let-else.html)
- [`is` 与槽的类型检查](../spec/refs-lang/002-types/202-any-slot.html)，[和类型](../spec/refs-lang/002-types/206-sum-types.html)
- 下一章：[5. 和类型与 none](05-sum-types.html)
