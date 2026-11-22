# 5. 和类型与 none

当一个值是“要么这个，要么那个”时，Nabi 用 `|` 把类型连起来，写作 `A | B`。本章介绍和类型（sum type），以及处理最常见的和类型 `T | none`（“可能没有值”）的各种工具。

## 和类型 `A | B`

和类型的值恰好属于所列变体中的一个。值上带有记录是哪个变体的标签，所以 `match` 可以用类型检查模式 `name: Type` 把它取出来。

```nabi
fn describe(v: i32 | str | bool) -> str {
    match v {
        n: i32 => f"integer {n}",
        s: str => f"string {s}",
        b: bool => f"boolean {b}",
    }
}

println(describe(42));
println(describe("nabi"));
println(describe(true));
```

```output
integer 42
string nabi
boolean true
```

和类型是**封闭的**。编译器知道所有可能的变体，所以从上面的 `match` 中删掉 `bool` 分支，就会得到一个“未处理所有情况”的编译错误。添加新变体时，编译器会指出每一处需要修改的地方。

和类型的行为像集合。`i32 | str` 和 `str | i32` 是同一个类型，`i32 | i32` 就是 `i32`。较长的和类型可以用 `type` 命名。

```nabi
type Number = i32 | f64;

fn half(n: Number) -> f64 {
    match n {
        i: i32 => i as f64 / 2.0,
        f: f64 => f / 2.0,
    }
}

println(f"{half(7)} {half(1.5)}");
```

```output
3.5 0.75
```

## `none` 与 `T?`

`none` 是表示“没有值”的字面量。在其他语言使用 `Option<T>` 或 `null` 的地方，Nabi 使用和类型 `T | none`，简写为 `T?`。没有 `Some(..)` 这样的包装，所以直接返回值即可。

```nabi
fn find(xs: List<i32>, target: i32) -> usize? {
    for i in 0..xs.len() {
        if xs[i] == target { return i; }
    }
    none
}

let xs = [3, 1, 4];
println(f"{find(xs, 4):?}");
println(f"{find(xs, 7):?}");
```

```output
2
none
```

标准库遵循同样的约定。可能失败的查找，例如 `List::get`、`Dict::get` 和 `List::pop`，都返回 `T | none`。

> [!NOTE]
> 进阶：`T | none` 几乎没有开销。对于有未使用位模式的类型，例如 `bool`、指针和没有字段的 `enum`，`none` 存放在其中一个位模式中，不需要单独的标签（[niche 优化](../spec/refs-lang/002-types/207-niche-optimization.html)）。所以 `bool?` 只占 1 字节。

## 用 `is` 收窄类型

`x is T` 是一个 `bool` 表达式，检查值当前是否为 `T` 变体。当 `if` 条件检查的是局部变量时，该变量在这个分支中会被**收窄**为 `T`，因此可以直接使用 `T` 的操作。用 `&&` 连接的后续条件也是如此。

```nabi
let v: i32 | str = 7;
if v is i32 && v > 5 {
    println(f"big number: {v + 1}");
}

let w: i32 | str = "hello";
if w is str {
    println(f"length: {w.len()}");
}
println(f"{w is i32}");
```

```output
big number: 8
length: 5
false
```

对于类型既不是和类型也不是 `any` 的值，`x is T` 在编译时就能确定。这里不带后缀的字面量取其默认类型：执行 `let n = 3;` 之后，`n is i64` 为 `true`。

在收窄后的分支中，仍然可以给变量赋值。赋予收窄后类型的值，收窄保持不变；赋予声明类型中其他成员的值，收窄就此结束，从那里到分支末尾，变量重新具有其声明的类型。

```nabi
let v: i32 | str = 5;
if v is i32 {
    v = v + 1;              // still i32
    println(f"{v * 2}");
    v = "six";              // ends the narrowing: v is i32 | str again
    println(f"{v is str} {v:?}");
}
```

```output
12
true "six"
```

执行 `v = "six"` 之后再写 `v + 1` 就是类型错误，因为 `i32 | str` 上没有定义 `+`。

## `if let` 与 `match`

要同时检查值并把它绑定到新名字，使用 `if let name = expr`：对于 `T | none` 值，`name` 是去掉 `none` 之后的 `T`。只有在要从更宽的和类型中选出一个成员时，才写 `if let name: T = expr`。在 `match` 中，`none` 写作字面量模式。

```nabi
fn lookup(id: i32) -> str? {
    if id == 1 { "kim" } else { none }
}

if let name = lookup(1) {
    println(f"found: {name}");
} else {
    println("not found");
}

for id in [1, 2] {
    match lookup(id) {
        name: str => println(f"{id} → {name}"),
        none => println(f"{id} → (none)"),
    }
}
```

```output
found: kim
1 → kim
2 → (none)
```

与 `none` 的比较也会收窄类型。在 `x != none` 之后、在 `if x == none` 的 `else` 中，以及在 `if x == none { return ..; }` 之下，`x` 都是去掉 `none` 的值，直到你给它赋一个可能为 `none` 的值。

```nabi
fn shout(name: str?) -> str {
    if name == none { return "nobody"; }
    name.to_uppercase()       // 这里 name: str
}
println(shout("kim"));
println(shout(none));
```

```output
KIM
nobody
```

## `let else`：提前退出

`let pattern = expr else { ... };` 在模式匹配时绑定名字，否则执行 `else` 块，该块必须用 `return`、`break`、`continue` 或 `throw` 离开当前作用域。这样，“缺失就提前退出”的写法不需要额外缩进，之后该名字具有收窄后的类型。

```nabi
fn first_even(xs: List<i32>) -> i32 | none {
    for x in xs {
        if x % 2 == 0 { return x; }
    }
    none
}

fn report(xs: List<i32>) -> str {
    let n = first_even(xs) else {
        return "no even number";
    };
    f"first even is {n}, half is {n / 2}"
}

println(report([1, 3, 8, 10]));
println(report([1, 3]));
```

```output
first even is 8, half is 4
no even number
```

## 用 `??` 提供默认值，用 `?.` 链式访问

每次都写 `match` 会很繁琐。`a ?? b` 在 `a` 为 `none` 时返回 `b`，否则返回 `a` 的值。`x?.field` 和 `x?.method()` 在 `x` 为 `none` 时求值为 `none`，只有存在值时才执行访问。两者结合，“有就取，没有就用默认值”一行就能写完。

```nabi
struct User { name: str; nick: str?; }

let u: User? = User { name: "kim", nick: none };
let nobody: User? = none;

println(u?.name ?? "anonymous");
println(nobody?.name ?? "anonymous");
println(f"{u?.name?.len():?}");

let scores: Dict<str, i32> = {"kim": 90};
println(f"{scores.get("kim") ?? 0} {scores.get("lee") ?? 0}");
```

```output
kim
anonymous
3
90 0
```

`?.` 会短路。如果左边是 `none`，后面的方法永远不会被调用。

## 后缀 `?`：遇到 none 时提前返回

当函数需要依次解开多个 `T | none` 值时，使用后缀 `?`。`e?` 取出 `e` 的值，如果它是 `none`，就立即从外围函数返回 `none`。因此函数的返回类型必须包含 `none`。

```nabi
struct User { name: str; nick: str?; }

fn nick_len(u: User?) -> usize? {
    let user = u?;
    let nick = user.nick?;
    nick.len()
}

println(f"{nick_len(User { name: "a", nick: "bob" }):?}");
println(f"{nick_len(User { name: "a", nick: none }):?}");
println(f"{nick_len(none):?}");
```

```output
3
none
none
```

`?` 只作用于 `T | none`。它不是传播错误的方式；错误用 `throws` / `try` 处理（[错误控制流](../spec/refs-lang/003-expressions/314-error-control-flow.html)）。

> [!NOTE]
> 进阶：如果集合本身存放 `none`（`List<i32?>`），`get` 的结果就是两层的可选类型 `(i32 | none)?`，用来区分“没有元素”和“存放的是 none”。`??`、`?.` 和 `?` 只作用于外层。完整规则见[嵌套可选类型](../spec/refs-lang/002-types/206-sum-types.html#嵌套可选类型)。

> [!NOTE]
> 进阶：`T?` 值在比较和排序时 `none` 排在最前，所以对 `[3, none, 1]` 排序得到 `[none, 1, 3]`。由两个值类型组成的和类型（例如 `i32 | str`）没有顺序：对它使用 `<`、`min` / `max` 和 `sort` 都是编译错误。这类值要用 `sort_by_key` 或 `sort_by` 按键排序（[规范顺序](../spec/refs-lang/002-types/206-sum-types.html#规范顺序)）。

## 小结

- `A | B` 是封闭的和类型。变体的顺序和重复无关紧要，`match` 必须处理每个变体。
- 值可能缺失时，使用 `T | none`，简写为 `T?`。直接返回值，不需要包装。
- `x is T`、`x != none` 以及提前的 `if x == none { return .. }` 会收窄 `x`。用 `if let x = ...`、`let x = ... else { return .. };` 以及 `match` 中的 `n: T` / `none` 模式取出值。
- `??` 提供默认值，`?.` 遇到 none 时跳过访问，后缀 `?` 遇到 none 时从函数返回。

## 延伸阅读

- [2.6. 和类型](../spec/refs-lang/002-types/206-sum-types.html)，[2.1.5a. 可选类型 `T?`](../spec/refs-lang/002-types/201-type-expressions.html#215a-可选类型t)
- [2.2.6. 类型收窄](../spec/refs-lang/002-types/202-any-slot.html#226-类型收窄)，[3.10.5. `if let`](../spec/refs-lang/003-expressions/310-if-expression.html#3105-if-let与模式匹配结合)，[3.16. `let-else`](../spec/refs-lang/003-expressions/316-let-else.html)，[3.11. 模式](../spec/refs-lang/003-expressions/311-patterns.html)
- [3.8. 可选链与空值合并](../spec/refs-lang/003-expressions/308-optional-chaining-and-coalescing.html)，[3.14.10. `?` 运算符](../spec/refs-lang/003-expressions/314-error-control-flow.html#31410--运算符提前返回-none)
- [2.7. niche 优化](../spec/refs-lang/002-types/207-niche-optimization.html)
- 下一章：[6. 结构体、枚举与 trait](06-structs-enums-traits.html)
