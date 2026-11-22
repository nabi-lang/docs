# 写给 Rust 用户的 Nabi

这是一份给 Rust 程序员的对照图：哪些写法可以照搬，哪些习惯要改掉，以及 Cargo 的各个工具在 Nabi 中对应什么。Nabi 的表层语法很多地方看起来像 Rust，但内存模型不同：内存由垃圾回收管理，所以没有生命周期，也没有借用检查器。下面每个代码块都是完整的程序，用 `nabi run file.nabi` 运行即可。本指南的各章会完整讲解每一部分。

## 可以照搬的写法

- `fn`、`struct`、`enum`、`trait`、`impl Trait for Type`，带约束和 `where` 子句的泛型，关联类型和常量泛型。
- 带守卫的 `match`、`if let`、`while let`、`let else`，以及解构元组、结构体和枚举变体的模式。
- `#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]`、`#[repr(C)]`、`#[cfg(..)]`、`#[test]` 和 `#[inline]`。
- `mod`、`use a::b::{c, d}`、`pub`、`pub use` 重新导出，以及代替 `crate::` 的包内相对路径（`pkg::`）。
- 整数类型 `i8` .. `i64`、`u8` .. `u64`、`isize`、`usize`、`f32`、`f64`；字面量后缀（`7u8`、`2.5f32`）、`_` 分隔符、`as` 强制转换、`checked_add`、`wrapping_add`、`saturating_add`。
- 运算符优先级，区间 `a..b` 和 `a..=b`，向零截断的 `/` 和 `%`，以及溢出时触发 panic。
- 迭代器链（`map`、`filter`、`fold`、`zip`、`enumerate`、`windows`）、闭包 `|x| x + 1`、`async fn` 和 `await`、`unsafe` 块和 `extern "C"`。
- 一个能构建、运行、测试、格式化、lint 和生成文档的包管理器：`nabi` 扮演 Cargo 的角色。

## 首先要改掉的习惯

- 没有生命周期，没有 `&` 或 `&mut`，也没有借用检查器。GC 释放内存；只要还有东西引用某个值，它就一直存活。
- 没有 `mut`。每个变量都可以重新赋值，对已有的名字再写一次 `let` 会声明一个新变量，就像 Rust 中的遮蔽（shadowing）。
- 结构体、枚举和元组都是值，赋值时会复制，不需要额外声明。`List`、`Dict` 和 `Set` 是指向同一个共享对象的句柄。`ref` 让函数可以修改调用者的结构体。
- 错误不是返回的值。可能失败的函数要写 `throws`，对它的每次调用都写 `try`。没有用于错误的 `?`。
- 没有 `Option`。可能缺失的值的类型是 `T | none`，`T?` 是它的简写。
- 没有后缀的整数字面量从它在函数中的用法获得类型，与 Rust 相同；没有任何东西确定它的类型时，它是 `i64`。数字从不隐式转换，连 `i32` 到 `i64` 也不例外。
- 只有带 `Drop` 的类型（文件、套接字、通道的发送端）才有唯一的所有者并会被移动。关闭文件可能失败，所以文件要显式关闭，用 `using` 或 `try f.close()`。
- 语句以 `;` 结束，与 Rust 相同，但结构体字段也是如此：`struct P { x: i32; y: i32; }`。

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

如果仍然写了 `mut`，构建会停下，帮助信息会建议去掉它：

```text check=checks/rust-let-mut.nabi
error[E0012]: Nabi has no `let mut`: variables are mutable by default
  --> main.nabi:1:5
   |
 1 | let mut total = 0;
   |     ^^^
   |
   = help: drop `mut`: `let total = ..;`
```

其他 Rust 写法也会以同样的方式停止构建，每条帮助信息都会写出对应的 Nabi 形式：`&x` 和 `&mut x`、`&T` 和 `&mut self`、生命周期、`impl Trait`、`Box<dyn Trait>`、`Fn(i32) -> i32`、`move ||`、`println!`、`vec![..]`、`format!`、`macro_rules!`、`pub(crate)`、`crate::` 和 `super::`、`Vec`、`String` 和 `HashMap`、`throws` 函数中的 `Ok(x)` 和 `Err(e)`，以及对会抛出错误的调用使用 `.unwrap()`。一个错误只报告一次：`Rc::new(RefCell::new(x))` 只报告一次，并给出完整的 Nabi 形式（`Cell::new(x)`）。

## 内存：GC、值与句柄

结构体是值：赋值、传参或从列表中读出时都会复制，原来的值仍然可用。只有带 `Drop` 的类型才会移动（见下一节），所以重用之前不需要 clone。需要修改调用者结构体的函数接收 `ref`，调用处也要写 `ref`，这样读代码的人能看出什么可能被修改。修改接收者的方法接收 `ref self`，对应 `&mut self`；调用处不需要标记。单独的 `self` 是一个副本，就像 Rust 中 `Copy` 类型上的 `self`。

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

在 Rust 要求你在 `&` 和 `&mut` 之间做选择的地方，Nabi 把错误用法直接变成错误：给一个之后没有人读取的副本设置字段（`fn inc(self) { self.n += 1; }`，或 `for p in ps { p.x = 0; }`）无法通过编译，帮助信息会指出应写 `ref self` 或 `for ref`（[第 6 章](06-structs-enums-traits.html#修改调用者的值ref)）。

与锁或其他线程共享的句柄也会被检查。`Mutex` 守卫的 `g.get()` 返回的是受保护对象的视图，所以它不能离开持有锁的 `using` 块（[E0106](../errors/E0106.html)）；应改用 `g.get_clone()`。`r.lines()` 这样的流视图通过 `r` 读取，只能留在 `r` 的作用域内（[E0139](../errors/E0139.html)）；需要返回时用拥有数据的形式 `r.into_lines()`。只有在这几处，Nabi 才会追踪类似借用的东西，而且每个错误都会说明应该怎么写。

## 资源：用 `using` 代替作用域结束时的 `Drop`

GC 会释放内存，但从不关闭文件。实现了 `Drop` 的类型只有一个所有者，就像每个 Rust 值一样：`let b = a` 会移动它，之后 `a` 就不存在了。`using x = e { .. }` 无论以何种方式离开块都会关闭 `x`，语句形式 `using x = e;` 则覆盖外层块的剩余部分，按相反顺序关闭，就像 Rust 丢弃局部变量那样。没有用 `using` 绑定的 `Drop` 值，在其所有者离开作用域时释放。

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

关闭文件可能失败（缓冲到关闭时才写出的数据可能没有写到磁盘上），而 Rust 的 `Drop` 会吞掉这个错误。Nabi 不会替你关闭这样的类型：写 `try f.close()`，或用 `using` 绑定，它会抛出关闭时的错误。离开作用域时 `File` 仍未关闭，是编译错误（[E0134](../errors/E0134.html)）。

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

## 错误：用 `throws` 和 `try` 代替 `Result` 和 `?`

`fn f() -> Result<T, E>` 写作 `fn f() throws -> T`，`f()?` 写作 `try f()`。`try` 标出每个可能失败的调用，读代码的人能看到函数可能提前退出的每一处。`throw "message"` 抛出一个 `ValueError`；你自己的错误类型实现 `Display` 和 `Error`，并列在 `throws (A | B)` 中。

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

`catch e: IoError { .. }` 按类型选择，`finally` 总会运行。`Result<T, E>` 仍然存在，用于把错误作为值保存，以收集多个结果：`Result::catching(|| try f())`。`panic(..)`、越界的索引和失败的 `try!` 都是 panic，`catch` 捕获不到，与 Rust 相同。

## 用 `T | none` 和和类型代替 `Option`

`Option<T>` 写作 `T | none`，简写为 `T?`。`Some(x)` 就是 `x`，`None` 就是 `none`。`??` 相当于 `unwrap_or`，`?.` 相当于对字段或方法做 `map`，后缀 `?` 在返回 `T?` 的函数中返回 `none`，就像 Rust 的 `?` 作用于 `Option` 那样。

`A | B` 是任意类型的和，不需要声明枚举：用 `match` 分支或 `is` 选出类型。`enum` 为每种情况命名，并像 Rust 那样携带数据。

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

`match` 所针对的枚举，其变体直接写名字（写 `Circle(r)`，不写 `Shape::Circle(r)`）。Rust 写 `if let Some(x) = e` 的地方，`if let x = e` 从 `T?` 中取出值；`if let x: T = e` 从更宽的和类型中选出一个成员。

## trait、泛型与 `impl`

trait、`impl Trait for Type`、泛型约束、`where` 子句、默认方法和关联类型的用法都与 Rust 相同，孤儿规则也一样：`impl` 要求 trait 或类型来自你的包；trait 实参中的本地类型也算，只要它出现在任何裸类型参数之前（`impl Mul<Matrix> for f64`）（[E0076](../errors/E0076.html)）。区别如下：

- trait 不是类型。`dyn Trait` 写作 `any Trait`，不需要 `Box`：`List<any Shape>` 可以保存任何实现了 `Shape` 的类型的值。`impl Trait` 写作 `some Trait`，可用于参数和返回位置：`fn area(s: some Shape)` 是一个没有名字的泛型参数，`fn evens() -> some Iterator<Item = i32>` 隐藏了它返回的类型。Rust 的 `type X = impl Trait;`（类型别名 `impl Trait`）写作 `type X = some Trait;`：X 所在模块中返回 `X` 的函数确定它的类型，其他模块只能看到约束。
- 没有写类型的参数是泛型的：`fn loud(x) -> str { x.to_uppercase() }` 接受任何具有函数体所调用方法的类型。
- `Fn`、`FnMut` 和 `FnOnce` 是同一个类型 `fn(i32) -> i32`，可以保存函数或闭包。`fn() + Send` 表示可以移动到其他线程的函数。
- 闭包通过复制值来捕获。`[ref x]` 捕获变量本身，所以闭包可以修改它；`[move x]` 把值交出去：创建闭包的代码之后就不能再使用 `x`。
- `#[derive(..)]` 像 Rust 那样写出各个 `impl`。`Copy` 只是一个标记，因为每个结构体在 `let y = x;` 时本来就会复制。

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

## 数字

没有后缀的整数字面量从它的去处（`let b: u8 = 200;`、一个实参、另一种类型的操作数）或之后的用法获得类型，没有任何东西给它类型时是 `i64`。浮点字面量同理是 `f64`。长度和索引是 `usize`，所以当 `total` 是 `i32` 时，`total / xs.len()` 需要 `as`。`int` 是任意大小的整数。

没有任何隐式转换：`i32 + i64` 是一个错误，帮助信息会写出 `as`。`as` 会截断，`.into()` 只允许用在不丢失信息的地方，`.try_into::<T>()` 在值放不下时抛出错误，所以 `try_from` 需要 `try`。

```nabi
let a = 7;                    // i64：没有别的东西给它类型
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

`%%` 是向下取整的余数，除数为正时相当于 Rust 的 `rem_euclid`。`x.pow(n)` 是方法，与 Rust 相同，`x ** n` 是运算符形式（整数底数要求 `u32` 指数）。

## 运算符

算术、位运算和比较运算符及其优先级都与 Rust 相同，运算符 trait 是 `Add`、`Mul`、`Neg`、`Index` 等。Nabi 另外加了几个：

- 链式比较：`0 <= i < n`。
- 用于成员判断的 `in` 和 `!in`：`x in xs`、`k !in dict`。
- 逐元素的“点”运算符，如 `.+`、`.*`、`.>` 等，把运算符应用到列表或 `Array` 的每个元素上，以及它们的原地形式 `.=` 和 `.+=`（[第 8 章](08-collections-iterators.html#进阶广播运算符)、[第 21 章](21-arrays-and-numerics.html)）。
- 求幂的 `**`：`2 ** 10`、`x ** 0.5`。
- 给出默认值的 `??`，以及穿过 `none` 链式访问的 `?.`。
- `Path` 上的 `/` 拼接路径：`dir / "file.txt"`。

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

## 模块与可见性

`mod name;` 加载与当前文件同目录的 `name.nabi`，`name/` 保存它的子模块，与 Rust 2018 不用 `mod.rs` 的做法相同。从包根开始的路径以 `pkg::` 开头，对应 Rust 的 `crate::`，`sup::` 对应 `super::`。项默认对其模块私有；`pub(sub)` 向子模块公开，`pub(pkg)` 向整个包公开（`pub(crate)`），`pub` 向所有人公开。标准库中有少数 `pub` 项是 `#[internal]` 的：只有工具链自己的包可以使用它们。

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

`println`、`List`、`Dict`、`Mutex`、`channel`、`task` 和 `par` 都在 prelude 中；`std` 中与操作系统打交道的模块（`fs`、`env`、`process`、`thread`、`net`）用 `use std::fs;` 导入。

## 属性与宏

属性的写法与 Rust 相同。原本会用到 `macro_rules!` 或过程宏的程序，在 Nabi 中只有一种机制：元定义，即带 `@` 记号的 Nabi 代码模板。`#[impl_derive(T)]` 定义一个 derive，`#[impl_attribute(name)]` 定义一个属性，`#[impl_macro(name)]` 定义一个调用 `name!(..)`（[6.8](../spec/refs-lang/006-meta-system/608-meta-blocks.html)）。没有在编译时对词法单元流运行的代码。`println`、`assert` 和 `format` 风格的字符串都是普通函数和 f-string，所以不带 `!`。

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

## 并发

线程与 Rust 相同：`thread::spawn` 接受一个闭包，`join` 返回它的值（如果线程触发了 panic 则抛出错误）。`Send` 和 `Sync` 都存在，集合类型不是 `Sync`：要把列表交给另一个线程的闭包，用 `[move rows]` 捕获它。`Mutex`、`RwLock`、`Atomic` 和 `channel` 都在 prelude 中；交给锁或通道的值之后不能再使用，这项交接检查取代了 Rust 中把值移动进 `Mutex` 的做法。

`async fn` 和 `await` 自带内置的运行时，所以不需要 Tokio：`task::spawn` 启动一个任务，`task::join` 等待多个任务，带顶层 `await` 的脚本以 `async fn main` 运行。`await` 是前缀运算符：写 `await f()`，不写 `f().await`。`par::map`、`par::filter` 和 `par::for_each` 把工作分散到线程池中，与 Rayon 类似。

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

## `unsafe` 与 FFI

`extern "C" { .. }` 声明 C 函数，对它们的每次调用都要放在 `unsafe { .. }` 中。`#[repr(C)]`、裸指针（`bare::ptr` 中的 `Ptr<T>`、`ConstPtr<T>`）、`#[link("m")]` 和 `extern "C" fn` 回调的用法都与 Rust 相同。`Pod`（`bare::mem::Pod`，plain old data）是一个 `unsafe trait`，类似 `bytemuck::Pod`：写 `#[derive(Pod)]` 或 `unsafe impl Pod for T {}`，编译器仍会检查布局。`List`、`Dict` 和闭包不能传入 C，因为回收器可能移动它们；请传指针和长度。

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

## Cargo、crate 与 `nabi.nabion`

`nabi` 是包管理器，清单是 `nabi.nabion`，用 Nabi 的数据格式 NABION 而不是 TOML 编写。包发布到 [packages.nabi-lang.org](https://packages.nabi-lang.org)，就像 crate 发布到 crates.io。`nabi.lock` 像 `Cargo.lock` 一样固定版本。`nabi publish` 不上传源代码压缩包：它在索引中添加一行，指向你的 git 仓库中的一个提交。单个文件也可以是程序：`nabi file.nabi` 不需要包和 `main` 就能运行它，`nabi add json file.nabi` 把依赖记录在文件本身中。

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
| `[features]`, `--features` | `nabi.nabion` 中的 `features:`，`--features`（[第 18 章](18-features-and-publishing.html)） |

`nabi fmt` 修正缩进、行尾空白和空行，并把模块顶部的 `mod` / `use` 行按 `pub mod`、`mod`、`pub use`、`use` 分组排序；行内的词法单元和空格则保持你写的样子。测试就是包中任意位置的 `#[test]` 函数，外面不需要包一层 `#[cfg(test)] mod tests`：普通构建会排除它，`nabi test` 在各自独立的进程中运行每个测试，所以一个 panic 不会让其他测试停下。

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

## Nabi 中的 Rust 惯用写法

| Rust | Nabi |
|---|---|
| `let mut x = 0;` | `let x = 0;` |
| `let x = x.trim();`（遮蔽） | `let x = x.trim();` |
| `&T`、`&mut T` 参数 | `T`（结构体的副本，`List` 则是同一个对象）、`ref p: T` |
| `&mut self` | `ref self` |
| `for x in xs.iter_mut()` | `for ref x in xs` |
| 重用之前的 `x.clone()` | 不需要：结构体会复制；需要独立的 `List` 时用 `.clone()` |
| `Box<dyn Trait>`, `Vec<Box<dyn Trait>>` | `any Trait`, `List<any Trait>` |
| `fn f(x: impl Trait)` | `fn f(x: some Trait)` |
| `fn f() -> impl Trait` | `fn f() -> some Trait` |
| `type X = impl Trait;` | `type X = some Trait;` |
| `impl Fn(i32) -> i32`, `Box<dyn FnMut()>` | `some fn(i32) -> i32`, `fn(i32) -> i32`, `fn()` |
| `move \|\| ..` | `[move x] \|\| ..` |
| 修改所捕获局部变量的闭包 | `[ref x] \|\| { x += 1; }` |
| `Option<T>`, `Some(x)`, `None` | `T?`, `x`, `none` |
| `opt.unwrap_or(d)`, `opt.map(\|v\| v.f())` | `opt ?? d`, `opt?.f()` |
| `if let Some(x) = opt` | `if let x = opt` |
| `fn f() -> Result<T, E>` | `fn f() throws -> T` |
| `f()?`, `f().unwrap()`, `f().ok()` | `try f()`, `try! f()`, `try? f()` |
| `Err("msg".into())`, `anyhow!("msg")` | `throw "msg"` |
| `match r { Ok(v) => v, Err(e) => .. }` | `try f() catch e { .. }` |
| `impl Drop` + 作用域结束 | `using x = e;`，文件则用 `try f.close()` |
| `Vec<T>`, `HashMap<K, V>`, `HashSet<T>` | `List<T>`, `Dict<K, V>`, `Set<T>` |
| `vec![1, 2]`, `String`, `&str` | `[1, 2]`、`str`（只有一种字符串类型） |
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
