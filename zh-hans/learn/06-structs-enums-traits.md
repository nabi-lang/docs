# 6. 结构体、枚举与 trait

现在来定义你自己的类型。本章介绍组织数据的 `struct`、列出具名情况的 `enum`，以及定义多个类型共有行为的 `trait`。

## 结构体与字段默认值

`struct` 是一组具名字段。字段写作 `name: Type;`，可以用 `= value` 给出默认值。有默认值的字段在构造时可以省略。

```nabi
struct Config {
    name: str;
    retry: i32 = 3;
    verbose: bool = false;
}

let a = Config { name: "api" };
let b = Config { name: "db", retry: 5 };
println(f"{a.name} retry={a.retry} verbose={a.verbose}");
println(f"{b.name} retry={b.retry}");
```

```output
api retry=3 verbose=false
db retry=5
```

默认值只能是字面量或 `const` 表达式。需要在运行时计算的初始值，放到 `fn new() -> Config` 这样的构造函数中。此外还有按位置区分字段的元组结构体（`struct Rgb(u8, u8, u8);`），以及没有字段的单元结构体（`struct Marker;`）。单元结构体可以是泛型的（`struct Tag<T>;`）：它的值是 `Tag::<i64>`，在类型已知的地方也可以只写 `Tag`（`let t: Tag<i64> = Tag;`）。它们的构造器是值，所以同一模块中同名的函数、`const` 或 `static` 属于重复定义。

## 方法：`impl` 与 `self`

`impl Type { ... }` 块中的函数会成为该类型的方法。第一个参数是 `self` 的是实例方法（`p.dist2(q)`），否则是静态方法（`Point::origin()`）。修改字段的方法接收 `ref self`。

```nabi
struct Point { x: f64; y: f64; }

impl Point {
    fn origin() -> Point { Point { x: 0.0, y: 0.0 } }

    fn dist2(self, o: Point) -> f64 {
        let dx = self.x - o.x;
        let dy = self.y - o.y;
        dx * dx + dy * dy
    }

    fn shift(ref self, dx: f64) { self.x += dx; }
}

let p = Point { x: 3.0, y: 4.0 };
println(f"{p.dist2(Point::origin())}");
p.shift(1.5);
println(f"{p.x}");
```

```output
25
4.5
```

`ref self` 方法所做的修改会反映到调用者的变量上。调用处不需要写 `ref`。至于为什么需要 `ref`，下一节会讲。

## 结构体是值

结构体和 `i32` 一样是**值**。赋值、传给函数、从列表（`ps[i]`）或字典中读出，都会产生一个副本；写入副本不会影响原值。

```nabi
struct Point { x: i32; y: i32; }

let a = Point { x: 1, y: 2 };
let b = a;              // a copy
b.x = 100;
println(f"{a.x} {b.x}");

let ps = [Point { x: 1, y: 0 }, Point { x: 2, y: 0 }];
let first = ps[0];      // also a copy
first.x = 99;
println(f"{ps[0].x} {first.x}");
ps[0].x = 99;           // writes into the list
println(f"{ps[0].x}");
```

```output
1 100
1 99
99
```

`List`、`Dict` 和 `Set` 正好相反：变量是指向同一个共享对象的句柄，所以句柄的每个副本都能看到修改。

|                       | `struct`、`enum`、元组、数字 | `List`、`Dict`、`Set`、`Mutex` 等 |
|-----------------------|---------------------------------|-------------------------------------|
| `let b = a;`          | 独立的副本                | 同一个对象                     |
| 写入字段或元素 | 只改变副本       | 所有句柄都能看到                     |
| 另做一份副本 | 赋值本身即可          | `.clone()`                          |

这种复制是浅复制。保存 `List` 的结构体字段复制的是句柄，所以结构体的两个副本共享那个列表。`#[derive(Clone)]`（见下文）提供的 `clone()` 会把列表也复制一份。

```nabi
#[derive(Clone)]
struct Team { name: str; members: List<str>; }

let a = Team { name: "core", members: ["ada"] };
let b = a;
b.name = "docs";            // only b
b.members.push("grace");    // the list both share
println(f"{a.name} {a.members:?}");

let c = a.clone();
c.members.push("linus");
println(f"{a.members:?} {c.members:?}");
```

```output
core ["ada", "grace"]
["ada", "grace"] ["ada", "grace", "linus"]
```

### 修改调用者的值：`ref`

函数得到的是结构体实参的副本。要修改调用者的变量，把参数声明为 `ref`，并在调用处也写上 `ref`，这样读代码的人能看出哪些实参可能被修改。

```nabi
struct Counter { n: i32; }

fn bumped(c: Counter) -> i32 { c.n += 1; c.n }    // changes its own copy
fn bump(ref c: Counter) { c.n += 1; }             // changes the caller's variable

let c = Counter { n: 0 };
println(f"{bumped(c)} {c.n}");
bump(ref c);
println(f"{c.n}");
```

```output
1 0
1
```

方法也一样：`self` 是副本，`ref self` 是调用者的值（方法的调用处不需要 `ref`）。在副本上设置字段而之后没有任何地方读取，这次修改就会丢失，所以忘记写 `ref self` 是一个错误：

```nabi norun
struct Counter { n: i32; }

impl Counter {
    fn inc(self) { self.n += 1; }
}
```

```text check=checks/06-write-to-copy.nabi
error[E0057]: `self.n` is set on a copy of the receiver, so the change is lost when `inc` returns
  --> main.nabi:4:20
   |
 4 |     fn inc(self) { self.n += 1; }
   |                    ^^^^^^
   |
   = help: take `ref self` to change the caller's value: `fn inc(ref self)`; to change a local copy, write `let c = self;` and change `c`
```

按提示改为接收 `ref self`：

```nabi
struct Counter { n: i32; }

impl Counter {
    fn inc(ref self) { self.n += 1; }
}

let c = Counter { n: 0 };
c.inc();
c.inc();
println(f"{c.n}");
```

```output
2
```

循环也是如此。`for p in ps` 绑定的是每个元素的副本，所以 `p.x *= 10` 只会修改副本：

```nabi norun
for p in ps { p.x *= 10; }
```

```text check=checks/06-loop-copy.nabi
error[E0057]: `p.x` is set on a copy of the element, so `ps` does not change
  --> main.nabi:4:15
   |
 4 | for p in ps { p.x *= 10; }
   |               ^^^
   |
   = help: bind each element as a place with `for ref`: `for ref p in ps { p.x *= 10; }`
```

`for ref p in ps` 把每个元素作为位置绑定：读 `p` 就是读元素，写 `p` 或它的字段就是写元素。在 `p`（或 `ps[i]`、`Dict` 的条目）上调用 `ref self` 方法，同样会就地修改它。

```nabi
struct Point { x: i32; y: i32; }

let ps = [Point { x: 1, y: 0 }, Point { x: 2, y: 0 }];
for ref p in ps {
    p.x *= 10;
}
println(f"{ps[0].x} {ps[1].x}");
```

```output
10 20
```

使用元素的 `for ref` 循环不能改变所遍历列表的大小：在循环体中调用 `ps.push(..)` 是一个错误（[E0137](../errors/E0137.html)）。请在循环之后再改变长度。

闭包的结构体实参同样是副本，所以 `ps.for_each(|p| { p.x = 0; })` 也是同样的错误（[E0057](../errors/E0057.html)）。用 `for ref` 遍历列表，或者返回修改后的值并保留结果：`ps.map(|p| { p.x *= 10; p }).to_list()`。

## enum：具名的情况

`enum` 定义的值是若干变体之一。变体可以不带数据（`Empty`）、带位置字段（`Circle(f64)`）或带具名字段（`Rect { w: f64; h: f64; }`）。用 `Shape::Circle(1.0)` 这样的写法构造值，在 `match` 中用形状相同的模式解构。

```nabi
enum Shape {
    Circle(f64),
    Rect { w: f64; h: f64; },
    Empty,
}

fn area(s: Shape) -> f64 {
    match s {
        Shape::Circle(r) => 3.0 * r * r,
        Shape::Rect { w, h } => w * h,
        Shape::Empty => 0.0,
    }
}

let shapes = [Shape::Circle(1.0), Shape::Rect { w: 2.0, h: 3.0 }, Shape::Empty];
for s in shapes {
    println(f"{area(s)}");
}
println(f"{shapes[0] is Shape::Circle}");
```

```output
3
6
0
true
```

`s is Shape::Circle` 只检查变体，不管携带的数据。

在期望类型为该枚举的地方，变体可以直接写名字：对 `Shape` 做 `match` 的各个分支、传给 `Shape` 参数的实参、类型为 `Shape` 的 `let`、`return` 的值、结构体字面量中的字段，以及 `s == ..` 的右侧。这对所有枚举都有效，标准库的枚举也一样（`let o: Ordering = Less;`）。

```nabi
enum Shape {
    Circle(f64),
    Rect { w: f64; h: f64; },
    Empty,
}

fn sides(s: Shape) -> i32 {
    match s {
        Circle(_) => 0,
        Rect { .. } => 4,
        Empty => 0,
    }
}

println(f"{sides(Rect { w: 1.0, h: 2.0 })} {sides(Empty)}");
```

```output
4 0
```

没有期望类型时，比如 `let s = Empty;`，要写完整路径 `Shape::Empty`，错误信息会这样提示。你自己定义的名字总是优先于变体。

这和和类型有什么不同？和类型把已有的类型不加名字地组合在一起，而 `enum` 给每种情况一个有意义的名字。如果两者都是 `f64`，却需要区分“半径”和“边长”，就该用 `enum`。

## trait 与默认方法

`trait` 是一个类型必须提供的方法列表。只写签名、没有函数体的方法必须由实现者提供；带函数体的方法作为默认实现。用 `impl Trait for Type` 实现 trait。

```nabi
trait Animal {
    fn name(self) -> str;
    fn sound(self) -> str;
    fn speak(self) -> str { f"{self.name()}: {self.sound()}!" }
}

struct Dog;
struct Cat { lives: i32; }

impl Animal for Dog {
    fn name(self) -> str { "dog" }
    fn sound(self) -> str { "woof" }
}

impl Animal for Cat {
    fn name(self) -> str { "cat" }
    fn sound(self) -> str { "meow" }
    fn speak(self) -> str { f"cat ({self.lives} lives): meow" }
}

println(Dog.speak());
println(Cat { lives: 9 }.speak());
```

```output
dog: woof!
cat (9 lives): meow
```

只有在 trait 处于作用域中时才能找到它的方法：定义在同一模块中、用 `use` 导入，或者是标准库的 trait。来自其他模块的 trait 的方法需要 `use` 该 trait；缺少时编译器会告诉你要添加哪一个：

```nabi norun
use pkg::zoo::Animal;         // without this, `zoo::Dog.name()` is an error

mod zoo {
    pub trait Animal { fn name(self) -> str; }
    pub struct Dog;
    impl Animal for Dog { fn name(self) -> str { "dog" } }
}

println(zoo::Dog.name());
```

```text
error[E0135]: method `name` comes from trait `zoo::Animal`, which is not in scope
   = help: add `use pkg::zoo::Animal;`
```

如果一个 trait 的实现承诺了编译器无法检查的事情，就把它声明为 `unsafe trait`，它的每个实现都写作 `unsafe impl`。标准库的 `Pod` 就是一例；[第 14 章](14-ffi.html)展示了两者。

## 用 derive 自动实现

比较、哈希、调试输出这类机械的实现，用 `#[derive(...)]` 生成。内置的 derive 有 `Clone`、`Copy`、`Debug`、`Default`、`PartialEq`、`Eq`、`PartialOrd`、`Ord` 和 `Hash`；标准库另外提供用于错误类型的 `Error`（[第 10 章](10-errors.html#自定义错误类型)）和用于 NABION 序列化的 `Nabion`（[第 15 章](15-nabion.html)）。`Default` 对声明了默认值（`= value`）的字段使用该默认值，其余字段使用字段类型的 `default()`。

```nabi
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Hash, Default)]
struct Version { major: i32; minor: i32; }

let vs = [Version { major: 1, minor: 2 }, Version { major: 0, minor: 9 }];
vs.sort();
println(f"{vs:?}");

let seen = Set::new();
seen.insert(vs[0].clone());
println(f"{Version { major: 0, minor: 9 } in seen}");

let zero: Version = Default::default();
println(f"{zero == vs[0]}");
```

```output
[Version { major: 0, minor: 9 }, Version { major: 1, minor: 2 }]
true
false
```

`Ord` 按声明顺序比较字段，所以 `sort` 按版本排序。`Display` 不能 derive，因为人类可读的格式由类型的作者决定：在类型上写 `#[display("..")]`，即基于 `self` 的 f-string（`#[display("v{self.major}.{self.minor}")]`），或者手动实现（见下文的运算符一节）。

手写了相等比较时，`Hash` 也必须手写，而且必须把相等比较所看的内容原样交给哈希器，这样相等的值才有相同的哈希。`Hash` 实现并不计算一个数，而是把自己的各个部分写入传给它的 `Hasher`。`Hasher` 不在 prelude 中，所以实现要从 `core::hash` 导入它：

```nabi
use core::hash::Hasher;

struct Tag { name: str; }

impl PartialEq for Tag {
    fn eq(self, o: Tag) -> bool { self.name.to_lowercase() == o.name.to_lowercase() }
    fn ne(self, o: Tag) -> bool { !self.eq(o) }
}
impl Eq for Tag {}
impl Hash for Tag {
    fn hash<H: Hasher>(self, ref h: H) { h.write_str(self.name.to_lowercase()); }
}

let tags: Set<Tag> = Set::new();
tags.insert(Tag { name: "Nabi" });
println(f"{tags.contains(Tag { name: "NABI" })}");
```

```output
true
```

`Dict` 和 `Set` 使用的哈希器按进程设定密钥，所以每次运行的哈希值都不同；不要保存哈希值。

标准库自己的类型遵循同样的规则。每个公开类型都实现了 `Debug`，所以 `{x:?}` 可以用在区间（`0..3`）、`File`（`File { path: "a.txt" }`）或迭代器上；持有密钥的类型打印 `{ .. }` 而不是秘密内容。`Path`、`Ipv4Addr`、`Ordering` 这样的值类型实现了 `Eq` 和 `Hash`，所以可以作为 `Dict` 的键和 `Set` 的元素。以后可能增加情况的标准库枚举，比如 `IoErrorKind` 或 `HashAlgorithm`，标记为 `#[non_exhaustive]`：对它们做 `match` 需要一个 `_` 分支。

你自己的 derive 是一个 Nabi 代码模板：`#[impl_derive(Trait)] @derive { .. }` 用 `@for_fields` 按字段重复其中的部分，用 `@field.name` / `@field.type` 引用字段，并可以用 `@concat` 声明一个具名的配套项。下面 `Point` 上的 `#[derive(Columns)]` 还声明了 `PointColumns`，每个字段一个列表（[6.9](../spec/refs-lang/006-meta-system/609-template-tokens.html)）：

```nabi
trait Columns { fn width() -> usize; }

#[impl_derive(Columns)] @derive {
    pub struct @concat(@Self.name, Columns) {
        @for_fields { pub @field.name: List<@field.type>; }
    }
    impl @concat(@Self.name, Columns) {
        fn push(ref self, v: @Self) {
            @for_fields { self.@field.name.push(v.@field); }
        }
    }
    impl Columns for @Self {
        fn width() -> usize { @Self.field_count }
    }
}

#[derive(Columns)]
struct Point { x: f32; y: f32; }

let cols = PointColumns { x: [], y: [] };
cols.push(Point { x: 1.0, y: 2.0 });
cols.push(Point { x: 3.0, y: 4.0 });
println(f"{cols.x:?} {cols.y:?} {Point::width()}");
```

```output
[1.0, 3.0] [2.0, 4.0] 2
```

## trait 不是类型

在 Nabi 中，trait 名不能直接用在类型位置。`fn f(a: Animal)` 是编译错误。这样可以避免隐藏的动态分派开销。

```text check=checks/06-trait-as-type.nabi
error[E0071]: trait `Animal` cannot be used as a type directly
  --> main.nabi:2:9
   |
 2 | fn f(a: Animal) {}
   |         ^^^^^^
   |
   = help: use `any Animal` for a value of any type implementing it, or a generic parameter `<T: Animal>`
```

改用泛型或和类型。

```nabi
trait Animal { fn sound(self) -> str; }
struct Dog;
struct Cat;
impl Animal for Dog { fn sound(self) -> str { "woof" } }
impl Animal for Cat { fn sound(self) -> str { "meow" } }

// 1. Open set of types: a generic with a bound (static dispatch).
fn twice<T: Animal>(a: T) -> str { f"{a.sound()} {a.sound()}" }
fn thrice(a: some Animal) -> str { f"{twice(a)} {a.sound()}" }

// 2. Closed set of types: a sum type.
fn loud(a: Dog | Cat) -> str {
    match a {
        d: Dog => d.sound().to_uppercase(),
        c: Cat => c.sound().to_uppercase(),
    }
}

println(twice(Dog));
println(thrice(Cat));
println(loud(Cat));
```

```output
woof woof
meow meow meow
MEOW
```

`a: some Animal` 是同样的泛型，只是不必给 `T` 起名。候选类型固定为少数几个时用和类型；集合是开放的就用泛型（下一章）。

> [!NOTE]
> 进阶：当一个集合必须混合在运行时选定的多种实现时，写 `any Trait`，如 `let zoo: List<any Animal> = [Dog, Cat];`。通过它的调用使用动态分派，类型中的 `any` 让这一开销显而易见。自带类型参数的方法，比如 `fn write<B: AsBytes>(..)`，也可以通过它调用（[`any Trait`](../spec/refs-lang/002-types/202-any-slot.html#2210-受约束的-anyany-trait)）。

## 运算符 trait

`+`、`==`、`<` 这些运算符都是 trait 方法调用。`+` 是 `Add`，字符串插值 `{x}` 是 `Display`。所以实现这些 trait，你自己的类型就有了运算符和打印功能。运算符 trait 位于模块 `ops` 中，所以实现时写 `ops::Add`；运算符本身无需导入，在任何地方都能用。

```nabi
struct V2 { x: i32; y: i32; }

impl ops::Add for V2 {
    type Output = V2;
    fn add(self, o: V2) -> V2 { V2 { x: self.x + o.x, y: self.y + o.y } }
}

impl Display for V2 {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        try f.write_str(f"({self.x}, {self.y})");
    }
}

let v = V2 { x: 1, y: 2 } + V2 { x: 10, y: 20 };
println(f"v = {v}");
```

```output
v = (11, 22)
```

`type Output = V2;` 设定 trait 的**关联类型**。下一章还会见到它们。

`Display` 实现有固定的形式，可以照抄：`ref f` 让 `fmt` 写入调用者的 `Formatter`；写入可能失败，所以 `fmt` 是 `throws (FmtError)`，调用处用 `try` 标记。错误是[第 10 章](10-errors.html)的主题。像这里这样输出只是一个 f-string 时，在结构体上写 `#[display("({self.x}, {self.y})")]` 会生成同样的实现。每个 `Display` 类型还都有 `to_str()`。

## 小结

- `struct` 字段可以用 `= value` 设默认值，方法写在 `impl` 块中。
- 结构体是值：赋值、实参和 `list[i]` 都会复制它（浅复制；`List` 字段仍然共享）。要修改调用者的值，接收 `ref` 参数（`f(ref x)`）或 `ref self`。
- `enum` 的变体有单元、元组、具名三种形状，用 `match` 解构。
- `trait` 有必需方法和默认方法，用 `impl Trait for Type` 实现。
- `#[derive(...)]` 自动实现 `Debug`、`Clone`、`PartialEq`、`Eq`、`PartialOrd`、`Ord`、`Hash` 和 `Default`（以及标准库提供的 `Error` 和 `Nabion`）；`#[display("..")]` 生成 `Display`。
- trait 不是类型。使用泛型（`<T: Tr>` 或 `x: some Tr`）或和类型（或者进阶的 `any Trait`）。
- 运算符和 `{x}` 打印同样来自 trait 的实现（`Add`、`Display` 等）。

## 延伸阅读

- [2.3. `struct`](../spec/refs-lang/002-types/203-struct.html)、[2.3.8. 值语义](../spec/refs-lang/002-types/203-struct.html#238-值语义)、[4.7. `ref`](../spec/refs-lang/004-functions/407-ref-keyword.html)、[2.4. `enum`](../spec/refs-lang/002-types/204-enum.html)、[2.5. `union`](../spec/refs-lang/002-types/205-union.html)
- [2.8. `trait`](../spec/refs-lang/002-types/208-trait.html)、[2.9. `impl`](../spec/refs-lang/002-types/209-impl.html)、[4.9. 方法](../spec/refs-lang/004-functions/409-method.html)
- [6.2.1. `#[derive(...)]`](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#621-derive-自动实现-trait)
- [2.12. trait 不能作类型](../spec/refs-lang/002-types/212-trait-as-type-banned.html)、[2.2.10. `any Trait`](../spec/refs-lang/002-types/202-any-slot.html#2210-受约束的-anyany-trait)
- [3.1. 运算符分派](../spec/refs-lang/003-expressions/301-operator-dispatch.html)，API：[`Add`](../../std/core/ops.html#trait.Add)、[`Display`](../../std/core/fmt.html#trait.Display)、[NABION](../../std/std/nabion.html)
- 下一章：[7. 泛型](07-generics.html)
