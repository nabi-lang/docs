# 6. 結構、列舉與 trait

現在來定義你自己的型別。本章介紹三者：把資料組合起來的 `struct`、列出具名情況的 `enum`，以及定義多個型別共有行為的 `trait`。

## 結構與欄位預設值

`struct` 是一組具名欄位。欄位寫成 `name: Type;`，可以用 `= value` 給予預設值。有預設值的欄位在建構時可以省略。

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

預設值只能是字面值或 `const` 運算式。必須在執行時計算的初始值，請放進 `fn new() -> Config` 這類建構函式。另外還有以位置識別欄位的元組結構（`struct Rgb(u8, u8, u8);`），以及沒有欄位的單元結構（`struct Marker;`）。單元結構可以是泛型的（`struct Tag<T>;`）：它的值是 `Tag::<i64>`，在型別已知的地方也可以只寫 `Tag`（`let t: Tag<i64> = Tag;`）。它們的建構子本身是值，因此模組中同名的函式、`const` 或 `static` 會構成重複定義。

## 方法：`impl` 與 `self`

`impl Type { ... }` 區塊內的函式會成為該型別的方法。第一個參數是 `self` 時，它是實例方法（`p.dist2(q)`）；否則是靜態方法（`Point::origin()`）。會修改欄位的方法接受 `ref self`。

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

`ref self` 方法所做的變更會反映到呼叫端的變數上。呼叫處不需要寫 `ref`。至於為什麼需要 `ref`，就是下一節的主題。

## 結構是值

結構和 `i32` 一樣是**值**。指派、傳給函式，以及從串列（`ps[i]`）或字典讀出，都會產生一份複本，寫入複本不會影響原本的值。

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

`List`、`Dict` 或 `Set` 正好相反：變數是指向同一個共用物件的控制代碼，因此控制代碼的每份複本都看得到變更。

|                       | `struct`、`enum`、元組、數字 | `List`、`Dict`、`Set`、`Mutex`、... |
|-----------------------|---------------------------------|-------------------------------------|
| `let b = a;`          | 獨立的複本                | 同一個物件                     |
| 寫入欄位 / 元素 | 只改變複本       | 透過每個控制代碼都看得到           |
| 製作另一份複本 | 指派本身          | `.clone()`                          |

複製是淺層的。存放 `List` 的結構欄位複製的是控制代碼，所以結構的兩份複本會共用那個串列。`#[derive(Clone)]`（見下文）提供的 `clone()` 會連串列一起複製。

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

### 改變呼叫端的值：`ref`

函式拿到的是結構引數的複本。若要改變呼叫端的變數，把參數宣告為 `ref`，並在呼叫處也寫上 `ref`，讓讀者看得出哪些引數可能被改變。

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

方法也是一樣：`self` 是複本，`ref self` 是呼叫端的值（方法的呼叫處不需要 `ref`）。在之後沒有人讀取的複本上設定欄位，變更就會遺失，所以忘了寫 `ref self` 是錯誤：

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

照 help 所說，改成接受 `ref self`：

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

迴圈也是如此。`for p in ps` 繫結的是每個元素的複本，因此 `p.x *= 10` 只會改變複本：

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

`for ref p in ps` 把每個元素繫結為位置（place）：讀取 `p` 就是讀取元素，寫入 `p` 或其欄位就是寫入元素。在 `p`（或 `ps[i]`、`Dict` 的項目）上呼叫 `ref self` 方法，也會原地改變它。

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

使用元素的 `for ref` 迴圈不能改變它所走訪串列的大小：在迴圈主體中呼叫 `ps.push(..)` 是錯誤（[E0137](../errors/E0137.html)）。請在迴圈結束後再改變長度。

閉包的結構引數同樣是複本，因此 `ps.for_each(|p| { p.x = 0; })` 也是同樣的錯誤（[E0057](../errors/E0057.html)）。請用 `for ref` 走訪串列，或回傳改變後的值並保留結果：`ps.map(|p| { p.x *= 10; p }).to_list()`。

## enum：具名的情況

`enum` 定義一個屬於數個變體之一的值。變體可以不帶資料（`Empty`）、帶位置欄位（`Circle(f64)`），或帶具名欄位（`Rect { w: f64; h: f64; }`）。用 `Shape::Circle(1.0)` 這樣的寫法建構值，並在 `match` 中用相同形狀的模式解構。

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

`s is Shape::Circle` 只測試變體，不管它帶的資料。

在預期型別就是該列舉的地方，變體可以直接寫名稱：對 `Shape` 做 `match` 的各分支、`Shape` 參數的引數、型別為 `Shape` 的 `let`、`return` 的值、結構字面值中的欄位，以及 `s == ..` 的右邊。這適用於所有列舉，包括標準函式庫的列舉（`let o: Ordering = Less;`）。

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

沒有預期型別時，例如 `let s = Empty;`，請寫出完整路徑 `Shape::Empty`；錯誤訊息也會這樣告訴你。你自己定義的名稱永遠優先於變體。

這和和型別有什麼不同？和型別把既有的型別無名地組合在一起，而 `enum` 給每個情況一個有意義的名稱。即使兩者都是 `f64`，如果你需要區分「半徑」和「邊長」，`enum` 才是正確的選擇。

## trait 與預設方法

`trait` 是型別必須提供的一組方法。只寫簽章、沒有主體的方法必須由實作者提供；有主體的方法則作為預設實作。用 `impl Trait for Type` 實作 trait。

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

只有在 trait 位於作用域內時，才找得到它的方法：定義在同一個模組、用 `use` 匯入，或是標準函式庫的 trait。來自其他模組的 trait 方法需要 `use` 該 trait；沒有的話，編譯器會告訴你要加哪一個：

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

如果 trait 的實作承諾了編譯器無法檢查的事，就把它宣告為 `unsafe trait`，而它的每個實作都寫成 `unsafe impl`。標準函式庫的 `Pod` 就是一例；[第 14 章](14-ffi.html)會示範兩者。

## 用 derive 自動實作

比較、雜湊、除錯輸出這類機械式的實作，可以用 `#[derive(...)]` 產生。內建的 derive 有 `Clone`、`Copy`、`Debug`、`Default`、`PartialEq`、`Eq`、`PartialOrd`、`Ord` 和 `Hash`；標準函式庫另外加上用於錯誤型別的 `Error`（[第 10 章](10-errors.html#自訂錯誤型別)）以及用於 NABION 序列化的 `Nabion`（[第 15 章](15-nabion.html)）。`Default` 對有宣告預設值（`= value`）的欄位使用該預設值，其他欄位則使用欄位型別的 `default()`。

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

`Ord` 依宣告順序比較欄位，因此 `sort` 會依版本排序。`Display` 不能 derive，因為人類可讀的格式由型別作者決定：在型別上寫 `#[display("..")]`，內容是引用 `self` 的 f-string（`#[display("v{self.major}.{self.minor}")]`），或是手動實作（見下方的運算子一節）。

手寫相等比較時，`Hash` 也必須手寫，而且餵給 hasher 的內容必須正好是相等比較所看的部分，這樣相等的值才會有相同的雜湊。`Hash` 實作不是計算出一個數字，而是把各個部分寫進它收到的 `Hasher`。`Hasher` 不在預載模組中，所以實作要從 `core::hash` 匯入它：

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

`Dict` 和 `Set` 使用的 hasher 每個行程有各自的金鑰，因此每次執行的雜湊值都不同；絕對不要儲存雜湊值。

標準函式庫自己的型別也遵循相同的規則。每個公開型別都實作了 `Debug`，所以 `{x:?}` 可以用在範圍（`0..3`）、`File`（`File { path: "a.txt" }`）或迭代器上；存放金鑰的型別會印出 `{ .. }` 而不是秘密內容。`Path`、`Ipv4Addr`、`Ordering` 這類值型別實作了 `Eq` 和 `Hash`，因此可以當作 `Dict` 的鍵和 `Set` 的元素。之後可能新增情況的標準函式庫列舉，例如 `IoErrorKind` 或 `HashAlgorithm`，標有 `#[non_exhaustive]`：對它做 `match` 時需要 `_` 分支。

你自己的 derive 是一份 Nabi 程式碼樣板：`#[impl_derive(Trait)] @derive { .. }` 用 `@for_fields` 對每個欄位重複某些部分，用 `@field.name` / `@field.type` 指稱它們，並可以用 `@concat` 宣告一個具名的附屬項目。下面在 `Point` 上的 `#[derive(Columns)]` 還會宣告 `PointColumns`，每個欄位各一個串列（[6.9](../spec/refs-lang/006-meta-system/609-template-tokens.html)）：

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

## trait 不是型別

在 Nabi 中，trait 名稱不能直接用在型別的位置。`fn f(a: Animal)` 是編譯錯誤。這可以避免隱藏的動態分派成本。

```text check=checks/06-trait-as-type.nabi
error[E0071]: trait `Animal` cannot be used as a type directly
  --> main.nabi:2:9
   |
 2 | fn f(a: Animal) {}
   |         ^^^^^^
   |
   = help: use `any Animal` for a value of any type implementing it, or a generic parameter `<T: Animal>`
```

改用泛型或和型別。

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

`a: some Animal` 是同樣的泛型，只是不必為 `T` 命名。如果候選型別是固定的幾個，就用和型別；如果集合是開放的，就用泛型（下一章）。

> [!NOTE]
> 進階：當一個集合必須混合數個在執行時才選定的實作時，寫 `any Trait`，例如 `let zoo: List<any Animal> = [Dog, Cat];`。透過它的呼叫會使用動態分派，而型別中的 `any` 讓這項成本一目了然。帶有自己型別參數的方法，例如 `fn write<B: AsBytes>(..)`，也可以透過它呼叫（[`any Trait`](../spec/refs-lang/002-types/202-any-slot.html#2210-constrained-any-any-trait)）。

## 運算子 trait

`+`、`==`、`<` 等運算子都是 trait 方法的呼叫。`+` 是 `Add`，字串插值 `{x}` 是 `Display`。因此只要實作這些 trait，你自己的型別就能使用運算子和輸出。運算子 trait 位於 `ops` 模組，所以實作時要寫 `ops::Add`；運算子本身不需要匯入就能在任何地方使用。

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

`type Output = V2;` 設定 trait 的**關聯型別**。下一章還會再看到它。

`Display` 實作有固定的形式，可以直接照抄：`ref f` 讓 `fmt` 寫進呼叫端的 `Formatter`；而寫入可能失敗，所以 `fmt` 是 `throws (FmtError)`，呼叫處要標上 `try`。錯誤是[第 10 章](10-errors.html)的主題。像這裡一樣輸出只是一個 f-string 時，在結構上寫 `#[display("({self.x}, {self.y})")]` 就會產生相同的實作。每個 `Display` 型別也都有 `to_str()`。

## 小結

- `struct` 欄位可以用 `= value` 設定預設值，方法寫在 `impl` 區塊中。
- 結構是值：指派、引數和 `list[i]` 都會複製它（淺層複製；`List` 欄位仍然共用）。若要改變呼叫端的值，使用 `ref` 參數（`f(ref x)`）或 `ref self`。
- `enum` 變體有三種形狀：單元、元組與具名，並用 `match` 解構。
- `trait` 包含必要方法與預設方法，用 `impl Trait for Type` 實作。
- `#[derive(...)]` 自動實作 `Debug`、`Clone`、`PartialEq`、`Eq`、`PartialOrd`、`Ord`、`Hash` 和 `Default`（以及標準函式庫提供的 `Error` 和 `Nabion`）；`#[display("..")]` 產生 `Display`。
- trait 不是型別。請使用泛型（`<T: Tr>` 或 `x: some Tr`）或和型別（進階用法則是 `any Trait`）。
- 運算子和 `{x}` 輸出同樣來自實作 trait（`Add`、`Display` 等）。

## 延伸閱讀

- [2.3. `struct`](../spec/refs-lang/002-types/203-struct.html)、[2.3.8. 值語意](../spec/refs-lang/002-types/203-struct.html#238-value-semantics)、[4.7. `ref`](../spec/refs-lang/004-functions/407-ref-keyword.html)、[2.4. `enum`](../spec/refs-lang/002-types/204-enum.html)、[2.5. `union`](../spec/refs-lang/002-types/205-union.html)
- [2.8. `trait`](../spec/refs-lang/002-types/208-trait.html)、[2.9. `impl`](../spec/refs-lang/002-types/209-impl.html)、[4.9. 方法](../spec/refs-lang/004-functions/409-method.html)
- [6.2.1. `#[derive(...)]`](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#621-derive-automatic-trait-implementation)
- [2.12. 禁止將 trait 作為型別](../spec/refs-lang/002-types/212-trait-as-type-banned.html)、[2.2.10. `any Trait`](../spec/refs-lang/002-types/202-any-slot.html#2210-constrained-any-any-trait)
- [3.1. 運算子分派](../spec/refs-lang/003-expressions/301-operator-dispatch.html)、API：[`Add`](../../std/core/ops.html#trait.Add)、[`Display`](../../std/core/fmt.html#trait.Display)、[NABION](../../std/std/nabion.html)
- 下一章：[7. 泛型](07-generics.html)
