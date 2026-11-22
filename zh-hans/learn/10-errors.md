# 10. 错误处理

文件不存在、输入有误，这类可能失败的情况随处可见。Nabi 只有一种错误模型。可能抛出错误的函数在签名中写 `throws`，调用方在这次调用上标 `try`。这样，代码中每个可能传出错误的位置都一目了然。

## `throws`、`throw` 与 `try`

`throws` 写在参数列表之后、`->` 之前。在函数体中，`throw` 抛出一个错误值。大多数代码只需要一条消息：`throw "msg"`（任意 `str`，f-string 也可以）抛出标准库的 `ValueError::new("msg")`（除非 `throws` 列表或 `catch e: str` 原样接收 `str`），所以你不必定义错误类型。

每次调用会抛出错误的函数都必须加 `try`。调用失败时，错误交给最近的 `catch`；没有 `catch` 时，错误从外层函数传播出去，这个函数也必须是 `throws`。

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

`catch e { .. }` 当场处理错误，并以其块的值代替原来的值。`try` 覆盖它右侧的整个表达式：`try a(b())` 同时标记两次调用，`try { .. }` 覆盖块内的所有调用。

单独的 `throws` 表示函数可以抛出任意错误类型（是 `throws (any Error)` 的简写）；`s.parse()` 抛出 `ParseError`，`parse_age` 原样把它传出去。如何列出确切的错误类型，见[下文](#自定义错误类型)。

### 脚本顶层的错误

程序的顶层语句构成一个隐式的、带 `throws` 的 `main`。所以在脚本中只写 `try` 就够了：没有被捕获的错误会结束程序，在 stderr 上输出消息，退出码为 1。

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

显式写出、没有返回类型的 `fn main()` 也以同样的方式抛出错误，不必写 `throws`：

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

顶层语句中单独的 `return;` 以退出码 0 结束脚本。配合 `let .. else`，可以在缺少输入时停止脚本：

```nabi
let first = ["a", "b"].first() else {
    return;
};
println(first);
```

```output
a
```

在那里写 `return v;` 是错误（E0018）：要用其他退出码，请用 `process::exit(v)`。

## `try?` 与 `??`：用默认值代替错误

很多时候你只想要一个备用值。`try? e` 丢弃错误并产生 `none`，`??` 用默认值替换 `none`。`??` 会结束 `try?`（以及 `try!`）的操作数，所以 `try? f() ?? d` 就是 `(try? f()) ?? d`：出错时得到 `d`。`try` 仍然覆盖其右侧的全部内容，包括 `??`。

```nabi
fn parse_port(s: str) -> i32 {
    try? s.parse::<i32>() ?? 8080
}

println(f"{parse_port("3000")} {parse_port("abc")}");
```

```output
3000 8080
```

如果 `f()` 本身返回 `T | none`，`try? f()` 就是两层的 `(T | none)?`：外层的 `none` 表示抛出了错误，内层的是 `f` 自己的 `none`，而 `??` 只去掉外层。希望两者含义相同时，写 `try? f() ?? none`。

要有意忽略错误，写 `let _ = try? f();`。`try! e` 把错误变成 panic；只在那里失败就意味着 bug 时使用。

| 形式     | 出错时                                |
|----------|---------------------------------------|
| `try e`  | 交给最近的 `catch`，或向外传播        |
| `try? e` | 丢弃错误，产生 `none`                 |
| `try! e` | 触发 panic（视为程序 bug）            |

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

## `catch`：按类型选择

`catch` 的模式从上到下依次检查，执行第一个匹配的块。`e: T` 按类型选择，在该分支内 `e` 就是这个类型，可以读取它的字段。最后的 `catch e` 接住其余所有错误。

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

## 块形式与 `finally`

要一次包住多个步骤，使用 `try { .. } catch .. { .. }` 形式。无论控制流如何离开（正常结束、出错、`return` 等），`finally` 块都会最后执行。

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

从 `finally` 块本身跳出的 `return`、`break` 或 `continue` 是错误：它会丢掉正在向外传出的错误。请把跳转放在 `try` 表达式之后。

## `try for`：每一步都可能失败的循环

有些迭代器的每一步都可能失败：会抛出错误的生成器，或从流中读取的字节。这样的迭代器是 `TryIterator`，对它的 `for` 写作 `try for`。这里的 `try` 覆盖被迭代对象和每一步，但不覆盖循环体：循环体中会抛出错误的调用需要自己的 `try`。

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

存放在变量中的 `TryIterator` 会原地前进：用 `break` 提前离开的循环，会让变量停在循环结束的位置，再对它循环一次就从那里继续。`try! for` 在某一步失败时触发 panic，`try? for` 则在那里结束循环。要越过错误继续，对 `it.results()` 循环，它把每一步作为 `Result` 给出。

## 自定义错误类型

调用方需要区分不同的失败时，定义一个错误类型：一个带有 `Debug` 和 `Display`、并实现 `Error` trait 的结构体或枚举。`#[derive(Debug, Error)]` 提供其中两个，`#[display("..")]` 用基于字段的 f-string 写出 `Display`；手写的 `impl Display` 形式见[第 6 章](06-structs-enums-traits.html#运算符-trait)。这样签名就可以准确列出它抛出的错误，`throws (E)`，或者用和类型列出多种，`throws (A | B)`。编译器会检查列出的每个类型都被处理或传出。

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

`lookup` 的错误种类封闭为 `NotFound | Invalid`，所以覆盖了这两种类型的 `describe` 成为一个不再抛出错误的函数。这由编译器检查。错误种类难以封闭时（例如公开 API），使用单独的 `throws`；接收方用 `catch e` 捕获全部，或者像 `catch e: ValueError` 那样先收窄（[4.14.3](../spec/refs-lang/004-functions/414-throws.html#4143-无类型的-throws)）。

`catch` 后面可以跟任意模式，所以用结构体模式可以直接取出字段。

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

## `rethrows`：只在实参抛出时才抛出

以函数为实参的高阶函数可以声明为 `rethrows`。传入的函数不抛出错误时，调用不需要 `try`；只有传入会抛出错误的函数时才加 `try`。

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

## `using`：清理资源

GC 会回收内存，但文件、套接字、锁这类资源必须在确定的时刻关闭。`using x = acquire_expr { .. }` 无论以何种方式离开块（正常结束、`return` 或错误传播），都会调用 `x` 的 `Drop::drop`。多个资源用逗号分隔，按相反顺序清理。

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

`using` 是表达式，所以也能产生值，如 `let text = using f = try fs::open(path) { try f.read_to_end() };`。

资源应当一直存活到外层块结束时，语句形式 `using x = e;` 可以省去多出来的块。它相当于把其后的语句包进块形式，所以释放发生在块结束处，按相反顺序，对每一种离开方式都成立：

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

类型实现了 `Drop` 的值只有一个所有者。`let b = a` 会移动它，之后再使用 `a` 是错误；`a.clone()` 生成拥有自己所有者的第二个值。不使用 `using` 时，编译器在所有者离开作用域的地方释放该值，对每一种离开方式都成立：

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

关闭可能失败的类型（例如 `File`）不会被自动关闭，因为它的错误会丢失：用 `try f.close()` 关闭它，或者用 `using` 绑定。在它仍然打开时离开作用域是编译错误。出于同样的原因，`try?` 不能丢弃可能持有这类值的错误，例如 `tx.send(conn)` 失败时的 `SendError<TcpStream>`，它会把流还给你：用 `try .. catch e { .. }` 捕获它，取出值并关闭。

## 错误与 panic

`throws` 的错误是“可能发生的失败”。与之相对，索引越界、除以零、`try!` 失败或调用 `panic(..)` 是 **panic**，被视为程序 bug。panic 不能用 `catch` 捕获；它向 stderr 写一条消息，并以退出码 101 退出。`finally` 和 `using` 中的清理也不会执行。

```nabi build
let xs = [1, 2, 3];
let i = 7;
println(f"{xs[i]}");
```

```text
thread 'main' panicked at main.nabi:3:12:
index 7 out of bounds for length 3
```

要查看程序是怎样走到那里的，用 `NABI_BACKTRACE=1` 运行：panic 还会列出调用栈上的函数，最内层的在前（`NABI_BACKTRACE=full` 还会加上地址）。不需要 `-g`，但被优化器内联的函数没有单独的一行。

```text
stack backtrace:
   0: core::index::index_oob
   1: core::collections::list::<impl>::index
   2: main
note: some details are omitted, run with `NABI_BACKTRACE=full` for a verbose backtrace.
```

只有需要把错误作为值保存（收集多个结果或稍后处理）时，才使用 `Result<T, E>`。规则是：“要抛出，用 `throws`；要保存，用 `Result`”。

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

## 小结

- 可能抛出错误的函数声明 `throws`（在 `->` 之前），用 `throw` 抛出；`throw "msg"` 抛出 `ValueError`，能满足大多数需要。`fn main()` 不用声明也能抛出。
- 调用用 `try` 标记。未捕获的错误传播到外层函数；在脚本顶层则以退出码 1 结束程序。
- `try? e ?? default` 给出备用值；`let _ = try? f();` 忽略错误；`try!` 把错误变成 panic。单独的 `return;` 结束脚本。
- `catch` 是按模式选择的表达式，`finally` 总会执行。
- 调用方必须区分失败时，定义自己的错误类型（`Display` + `Error`），并用 `throws (A | B)` 列出。
- `rethrows` 只在收到的函数抛出时才抛出。
- 用 `using` 绑定的资源无论如何离开块都会被清理；`using x = e;` 覆盖所在块的剩余部分。其他 `Drop` 值在所有者离开作用域时释放；`File` 必须关闭或用 `using` 绑定。
- `try for` 遍历每一步都可能失败的迭代器；循环体自己标记 `try`。
- panic 是无法捕获的 bug 信号。只有需要把错误作为值保存时才用 `Result`。

## 延伸阅读

- 规范：[错误控制流](../spec/refs-lang/003-expressions/314-error-control-flow.html)、[`throws` 签名](../spec/refs-lang/004-functions/414-throws.html)、[`using` 表达式](../spec/refs-lang/003-expressions/317-using-expression.html)、[panic 与致命错误](../spec/refs-lang/003-expressions/314-error-control-flow.html#3148-panic-与致命错误)
- 标准库规范：[error](../spec/refs-standard-libraries/200-core/214-error.html)
- API：[`Error`](../../std/core/error.html#trait.Error)、[`ValueError`](../../std/core/error.html#struct.ValueError)、[`Result`](../../std/core/error.html#enum.Result)、[`Drop`](../../std/core/marker.html#trait.Drop)
- 下一章：[11. 模块与包](11-modules-packages.html)
