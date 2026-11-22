# 3. 函数

本章介绍函数的声明与调用、默认值和具名参数，以及可以作为值传递的闭包。在 Nabi 中，函数体也是一个块表达式，所以它的最后一个表达式就是返回值。

## 声明与返回值

用 `fn name(param: Type, ...) -> ReturnType { body }` 声明函数。函数体的最后一个表达式（不带分号的那个）就是返回值；只有提前退出时才用 `return`。

```nabi
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn sign(n: i32) -> str {
    if n < 0 {
        return "negative";
    }
    if n == 0 { "zero" } else { "positive" }
}

fn divmod(a: i32, b: i32) -> (i32, i32) {
    (a / b, a % b)
}

println(f"{add(2, 3)} {sign(-4)} {sign(0)}");
let (q, r) = divmod(17, 5);
println(f"17 = 5 * {q} + {r}");
```

```output
5 negative zero
17 = 5 * 3 + 2
```

要返回多个值，把它们放进元组，在调用处解构。

块中的语句以 `;` 结尾，但 `}` 前的最后一条可以省略，含义不变。所以 `while i < n { i += 1 }` 与 `{ i += 1; }` 相同，最后的 `let`、`return`、`break` 或 `throw` 也一样。只有最后一个表达式才会成为块的值，就像上面的 `a + b`。

返回类型和参数类型可以省略。省略的返回类型从最后一个表达式推断（以 `;` 结尾时为 `()`）。不写类型的参数会让函数成为泛型函数，针对调用时的每种实参类型分别编译。

```nabi
fn double(x: i32) { x * 2 }   // return type inferred: i32
fn id(x) { x }                // generic: fn id<T>(x: T) -> T
println(f"{double(4)} {id("hi")}");
```

```output
8 hi
```

`pub` 或 `pub(pkg)` 函数则不同：其他模块会调用它，所以它要写出完整签名：每个参数都有类型，没有 `->` 时返回 `()`（`main` 和生成器 `yield fn` 除外）。函数体给出了值时，编译器会告诉你该写哪个 `-> T`（[E0138](../errors/E0138.html)）。

```nabi
pub fn double(x: i32) -> i32 { x * 2 }   // pub: types written out
pub fn greet(name: str) { println(f"hi {name}"); }   // no `->`: returns ()
greet("nabi");
println(f"{double(4)}");
```

```output
hi nabi
8
```

块 `{ ... }` 在函数体之外也是表达式。用它可以把临时变量留在块内，只取出结果。

```nabi
let area = {
    let w = 3;
    let h = 4;
    w * h
};
println(f"{area}");
```

```output
12
```

## 默认值与具名参数

参数可以用 `= value` 设置默认值，调用时可以用 `name: value` 按名字传递实参。位置实参在前，具名实参在后。

```nabi
fn greet(name: str, greeting: str = "Hello", punct: str = "!") -> str {
    f"{greeting}, {name}{punct}"
}

println(greet("nabi"));
println(greet("nabi", punct: "?"));
println(greet(name: "Ada", greeting: "Hi"));
```

```output
Hello, nabi!
Hello, nabi?
Hi, Ada!
```

默认值必须在编译时已知，例如字面量或 `const`。可变数量的参数用 `...T` 接收；在函数体中它是只读切片。

```nabi
fn sum(xs: ...i32) -> i32 {
    let total = 0;
    for x in xs {
        total += x;
    }
    total
}

println(f"{sum()} {sum(1, 2, 3)}");
```

```output
0 6
```

## 闭包

闭包是匿名的函数值，写作 `|params| expr` 或 `|params| { block }`。参数类型从上下文推断；没有上下文时要写出来。闭包或函数的参数可以是总能匹配的模式，与 `let` 相同：`|(a, b)| a + b` 接受一个元组，`|a, b|` 接受两个值，`fn dist((x, y): (f64, f64))` 则解构其实参。

```nabi
let double = |x: i32| x * 2;
let describe = |x: i32| -> str {
    let half = x / 2;
    f"{x} is {half} * 2"
};
println(f"{double(21)}");
println(describe(10));

let sum = |(a, b): (i32, i32)| a + b;
println(sum((3, 4)));
```

```output
42
10 is 5 * 2
7
```

闭包**通过复制值**捕获周围的变量。闭包创建之后，即使外部变量发生变化，闭包记住的仍是创建时的值。要修改外部变量本身，用捕获列表 `[ref name]` 请求按引用捕获。

```nabi
let count = 0;
let snapshot = || count + 1;
count = 10;
println(f"snapshot: {snapshot()}");

let inc = [ref count] || { count += 1; };
inc();
inc();
println(f"count: {count}");
```

```output
snapshot: 1
count: 12
```

给按值捕获的变量赋值，或者给按值捕获的结构体的字段赋值，都是错误，错误信息会让你加上 `[ref count]`。`List` 等堆对象则不同：变量是指向对象的引用，所以即使按值捕获，也能修改对象本身，例如调用 `push`。

与 `[ref name]` 相反的是 `[move name]`：它把值交给闭包。闭包得到其中列表的独立副本，创建闭包的代码则不能再使用这个名字。列表就是这样交给另一个线程的（[第 12 章](12-concurrency.html)）：

```nabi
use std::thread;

let rows = ["a", "b", "c"];
let h = thread::spawn([move rows] || rows.len());
println(f"rows: {try! h.join()}");
```

```output
rows: 3
```

## 函数类型

函数和闭包都是 `fn(ParamTypes) -> ReturnType` 类型的值。可以把它们存进变量、作为实参传递、作为返回值返回，或者放进列表。

```nabi
fn apply(f: fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}

fn make_adder(n: i32) -> fn(i32) -> i32 {
    |x| x + n
}

let add5 = make_adder(5);
println(f"{apply(add5, 10)}");

let ops: List<fn(i32, i32) -> i32> = [|a, b| a + b, |a, b| a * b];
for op in ops {
    println(f"{op(6, 7)}");
}
```

```output
15
13
42
```

`make_adder` 返回的闭包捕获了 `n`，并一直保持有效。

## 尾随闭包

如果最后一个实参是闭包，可以把它移到括号外：有参数时写 `f(args) |x| { ... }`，没有参数时写 `f(args) { ... }`。这样，迭代、资源管理等模式读起来就像内置语法。

```nabi
fn repeat(times: i32, body: fn(i32)) {
    for i in 0..times {
        body(i);
    }
}

fn measure(label: str, body: fn() -> i32) -> i32 {
    let r = body();
    println(f"{label}: {r}");
    r
}

repeat(3) |i| {
    println(f"tick {i}");
}
let total = measure("sum") { 1 + 2 + 3 };
let evens = [1, 2, 3, 4].filter |x| { x % 2 == 0 }.to_list();
println(f"{total} {evens:?}");
```

```output
tick 0
tick 1
tick 2
sum: 6
6 [2, 4]
```

以尾随闭包调用开头的语句，例如上面的 `repeat(3) |i| { .. }`，在其 `}` 处结束，就像后面跟了 `;`：不需要写 `;`，下一行若以运算符（`-1`）开头，则是独立的一条语句。它和其他表达式语句一样，所以非单元类型的结果会被丢弃，`unused_result` lint 会对此发出警告（要有意丢弃，写 `_ = f(..) { .. };`）。在更大的表达式中，例如 `let total = measure("sum") { 6 } - 1;`，运算符会接续这个调用。

这种闭包中单独的 `return` 只会结束闭包的这一次调用，而不是外围的函数，因此它是错误。要结束这次调用，写出接受该闭包的函数名：`return@each`（带值时写 `return@find x`）。要离开外围函数，改用 `for` 循环。

```nabi
fn each(xs: List<i32>, body: fn(i32)) {
    for x in xs { body(x); }
}

each([1, 2, 3]) |x| {
    if x == 2 { return@each; }   // ends this call; 3 still prints
    println(f"{x}");
}
```

```output
1
3
```

## 管道：`|>`

`x |> f` 调用 `f(x)`，`x |> f(a)` 调用 `f(x, a)`：左边的值成为第一个实参。这样，一串自由函数读起来就是它们执行的顺序。`|>` 的结合比其他所有运算符都松，所以左边的整个表达式都会被传入（`1 + 2 |> double` 是 `double(3)`），以 `|>` 开头的一行会接续上一行。右边是一个函数、一个调用或括号中的闭包；方法仍然用 `.`（`s.trim()`）。

```nabi
fn double(x: i32) -> i32 { x * 2 }
fn add(x: i32, y: i32) -> i32 { x + y }

let n = 1 + 2
    |> double
    |> add(10)
    |> (|v| v - 1);
println(f"{n}");
```

```output
15
```

## 方法预览

`impl` 块中的函数是方法。如果第一个参数是 `self`，就用 `value.method()` 调用；没有 `self` 时，它是静态方法，用 `Type::function()` 调用。要在方法中修改字段，接受 `ref self`。结构体和方法会在后面的章节中详细介绍。

```nabi
struct Counter { n: i32 }

impl Counter {
    fn new() -> Counter { Counter { n: 0 } }
    fn bump(ref self) { self.n += 1; }
    fn get(self) -> i32 { self.n }
}

let c = Counter::new();
c.bump();
c.bump();
println(f"{c.get()}");
```

```output
2
```

## 嵌套函数与入口点

可以在函数内部声明函数。嵌套函数只在其所在的块中可见，并且与闭包不同，它**不**捕获外部变量。用它来隐藏不需要外部状态的辅助函数。

```nabi
fn hypot_sq(a: i32, b: i32) -> i32 {
    fn sq(x: i32) -> i32 { x * x }
    sq(a) + sq(b)
}
println(f"{hypot_sq(3, 4)}");
```

```output
25
```

程序从 `src/main.nabi`（或独立文件）的顶层语句开始，或者从 `fn main()` 开始。顶层语句会成为编译器生成的隐式 `main` 的函数体，所以顶层的 `let` 变量是**局部变量**，在其他函数体中不可见。如果函数需要这样的值，就把它作为实参传入，或者在闭包中捕获它。

```nabi
let base = 100;
let shifted = |x: i32| x + base;

fn scale(x: i32, factor: i32) -> i32 {
    x * factor
}

println(f"{shifted(1)} {scale(base, 3)}");
```

```output
101 300
```

## 小结

- 函数体的最后一个表达式就是返回值。用 `return` 提前退出。
- 参数可以有默认值，调用时可以用 `name: value` 写出实参名。`...T` 接收可变参数。
- 闭包 `|x| expr` 默认通过复制值来捕获；`[ref x]` 请求按引用捕获，`[move x]` 把 `x` 交给闭包。
- 函数值的类型是 `fn(A) -> R`。位于最后一个实参位置的闭包可以写成尾随形式；在其中，`return@name` 结束这次调用。
- `x |> f(a)` 就是 `f(x, a)`：管道把左边的值传给函数，它是结合最松的运算符。
- 顶层的 `let` 是隐式 `main` 的局部变量，所以其他函数看不到它。

## 延伸阅读

- [函数声明](../spec/refs-lang/004-functions/401-function-declaration.html)，[参数](../spec/refs-lang/004-functions/402-parameters.html)，[返回](../spec/refs-lang/004-functions/403-return.html)
- [函数类型](../spec/refs-lang/004-functions/404-function-type.html)，[闭包](../spec/refs-lang/004-functions/405-closure.html)，[捕获语义](../spec/refs-lang/004-functions/406-capture-semantics.html)
- [尾随闭包](../spec/refs-lang/004-functions/408-trailing-closure.html)，[方法](../spec/refs-lang/004-functions/409-method.html)，[嵌套函数](../spec/refs-lang/004-functions/410-nested-functions.html)
- [入口点](../spec/refs-lang/004-functions/416-entry-point.html)，[块表达式](../spec/refs-lang/003-expressions/309-block-expressions.html)，[管道表达式](../spec/refs-lang/003-expressions/318-pipe-expression.html)
- 下一章：[4. 控制流](04-control-flow.html)
