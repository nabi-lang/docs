# 7. 泛型

想让同一段逻辑适用于多种类型时，使用泛型。Nabi 的泛型会针对调用时用到的每个具体类型分别编译（单态化），所以运行速度和为每种类型手写的函数一样快。

## 泛型函数

在函数名后的 `<T>` 中声明类型参数，并在参数类型和返回类型中使用它们。调用处通常不必写出类型，编译器会根据实参推断。

```nabi
fn first_or<T>(xs: List<T>, fallback: T) -> T {
    xs.first() ?? fallback
}

println(f"{first_or([10, 20], 0)}");
println(first_or(["a", "b"], "-"));
let empty: List<f64> = [];
println(f"{first_or(empty, 1.5)}");
```

```output
10
a
1.5
```

在一次调用中，`T` 是单一的类型。像 `first_or([1, 2], "x")` 这样传入不同的类型是编译错误。

## trait 约束

对没有约束的 `T`，你只能把值搬来搬去。要比较或相加，就用 `T: Ord` 这样的**约束**要求所需的 trait。多个约束用 `+` 连接。编译器只依据约束来检查泛型函数体，所以无论用哪个类型调用，函数体都不会出错。

```nabi
fn largest<T: Ord>(xs: List<T>) -> T? {
    let best = xs.first()?;
    for x in xs {
        if x > best { best = x; }
    }
    best
}

println(f"{largest([3, 9, 2]):?}");
println(f"{largest(["pear", "apple"]):?}");
let none_here: List<i32> = [];
println(f"{largest(none_here):?}");
```

```output
9
"pear"
none
```

有了 `T: num::Zero`，还可以在类型参数上调用静态方法，比如 `T::zero()`。

```nabi
fn sum<T: num::Zero + ops::Add<Output = T>>(xs: List<T>) -> T {
    let acc = T::zero();
    for x in xs { acc = acc + x; }
    acc
}

println(f"{sum([1, 2, 3])} {sum([0.5, 0.25])}");
```

```output
6 0.75
```

### 函数体依据约束检查

泛型函数体会在查看任何调用之前，单独做一次类型检查。在函数体内，`T` 只支持约束所承诺的操作，以及所有类型都支持的操作：移动、存入 `any`、构成 `T | none`，以及 `is`。忘记写约束就是函数体中的错误，即使每次调用传入的类型其实都可以。

```nabi norun
fn double<T>(x: T) -> T { x + x }
```

```text check=checks/07-missing-bound.nabi
error[E0054]: operator `+` on `T` needs the bound `T: Add`
  --> main.nabi:1:27
   |
 1 | fn double<T>(x: T) -> T { x + x }
   |                           ^^^^^
```

加上约束后，同样的函数体对每个具有 `Add` 的类型都能编译。不需要约束的操作没有约束也能用。

```nabi
fn double<T: ops::Add<T, Output = T>>(x: T) -> T { x + x }
fn keep<T>(x: T, yes: bool) -> T | none { if yes { x } else { none } }
fn boxed<T>(x: T) -> any { x }

println(f"{double(21)} {double(1.5)}");
println(f"{keep("a", true):?} {keep(1, false):?}");
println(f"{boxed(3) is i64}");
```

```output
42 3
"a" none
true
```

由于这条规则，一个库的泛型函数在你这里能编译，在所有调用者那里也都能编译；错误永远不会在遥远的调用处才冒出来。trait 的默认方法也以同样方式检查，此时 `Self` 只能通过 trait 了解。参数不写类型的函数（下文的隐式泛型）是例外。

一个约束可以用不同的参数两次指定同一个 trait。函数体中的每次使用会选择参数与实参类型匹配的那个约束，就像运算符在一个类型的多个实现中做选择一样。

```nabi
struct Meters { v: f64; }
impl ops::Add<i32> for Meters { type Output = Meters; fn add(self, rhs: i32) -> Meters { Meters { v: self.v + rhs as f64 } } }
impl ops::Add<f64> for Meters { type Output = Meters; fn add(self, rhs: f64) -> Meters { Meters { v: self.v + rhs } } }

fn nudge<T: ops::Add<i32, Output = T> + ops::Add<f64, Output = T>>(x: T) -> T {
    (x + 1) + 0.5          // Add<i32>, then Add<f64>
}

println(f"{nudge(Meters { v: 2.0 }).v}");
```

```output
3.5
```

## `where` 子句

约束变长时，把它们移到签名末尾的 `where` 子句中更易读。含义不变。

```nabi
fn join_all<T>(xs: List<T>, sep: str) -> str
where
    T: Display,
{
    let out = StrBuf::new();
    for i in 0..xs.len() {
        if i > 0 { out.push_str(sep); }
        out.push_str(f"{xs[i]}");
    }
    out.to_str()
}

println(join_all([1, 2, 3], ", "));
println(join_all([true, false], " / "));
```

```output
1, 2, 3
true / false
```

## 泛型结构体与 impl

结构体和枚举也可以带类型参数。添加方法时，像 `impl<A, B> Pair<A, B>` 这样在 `impl` 上也声明这些参数。

```nabi
struct Pair<A, B> { first: A; second: B; }

impl<A, B> Pair<A, B> {
    fn swap(self) -> Pair<B, A> {
        Pair { first: self.second, second: self.first }
    }
}

let p = Pair { first: 1, second: "one" };
let q = p.swap();
println(f"{q.first} {q.second}");
```

```output
one 1
```

### 默认值

类型参数可以声明默认值：`struct Vec3<T = f32>`。作为类型书写时，`Vec3` 就是 `Vec3<f32>`。在表达式中，这个参数和其他参数一样被推断，顺序是：先看写出的实参（`Vec3::<f64>::new`），再看期望类型和后续的使用，然后是参数的默认值，最后是字面量的默认类型 `i64` / `f64`。所以填入 `T` 的无后缀字面量，除非另有说明，都是 `f32`。不会发生任何转换：带类型的实参本身就决定了 `T`。

```nabi
struct Vec3<T = f32> { x: T; y: T; z: T; }

impl<T> Vec3<T> {
    fn new(x: T, y: T, z: T) -> Vec3<T> { Vec3 { x: x, y: y, z: z } }
}

impl Vec3<f32> { const X: Vec3<f32> = Vec3 { x: 1.0, y: 0.0, z: 0.0 }; }
impl Vec3<f64> { const X: Vec3<f64> = Vec3 { x: 1.0, y: 0.0, z: 0.0 }; }

let a = Vec3::new(1.0, 2.0, 3.0);           // 默认值：Vec3<f32>
let b: Vec3<f64> = Vec3::new(1.0, 2.0, 3.0); // 期望类型优先
let x: f64 = 0.5;
let c = Vec3::new(x, 0.0, 0.0);             // x 决定 T：Vec3<f64>
let d: Vec3<f64> = Vec3::X;                 // Vec3<f64> 的 X
println(f"{a.type_name_of_val()} {b.type_name_of_val()} {c.type_name_of_val()} {d.x}");
```

```output
Vec3 Vec3<f64> Vec3<f64> 1
```

`Vec3::X` 以同样的方式确定实例：先看期望类型，否则看唯一定义了 `X` 的那个实现，再否则用默认值。函数的默认值（`fn zero<T: Num = f32>() -> T`）也是一种后备：`let r = zero();` 是 `f32`，除非后面对 `r` 的使用需要别的类型。

## 关联类型

要让 trait 的每个实现各自选定一个类型，声明**关联类型** `type Item;`。在 trait 内部直接写 `Item`；在外部的泛型代码中写 `C::Item`。标准的 `Iterator` 就是这样工作的（下一章）。

```nabi
trait Container {
    type Item;
    fn get(self, i: usize) -> Item | none;
    fn first(self) -> Item | none { self.get(0) }
}

struct Shelf { books: List<str>; }

impl Container for Shelf {
    type Item = str;
    fn get(self, i: usize) -> str? { self.books.get(i) }
}

fn first_of<C: Container>(c: C) -> C::Item | none { c.first() }

let s = Shelf { books: ["Dune", "Emma"] };
println(f"{first_of(s):?} {s.get(5):?}");
```

```output
"Dune" none
```

之所以用关联类型而不是类型参数，是因为对给定的实现者，`Item` 恰好固定为一个类型。使用者不需要写出 `Shelf` 的元素类型。对于 trait 声明的、没有默认值的每个关联类型，每个实现都要给出：漏写 `type Item = ..;` 是一个错误（[E0072](../errors/E0072.html)），它的帮助信息会写出这一行，若实现的方法中提到了该类型，就从中取得。

## 进阶：常量泛型

第一次阅读可以跳过本节。参数也可以是编译期的**值**而不是类型。当一个值是类型的一部分时使用它，比如定长数组 `[T; N]` 的长度。

```nabi
fn total<const N: usize>(a: [i32; N]) -> i32 {
    let s = 0;
    for i in 0..N { s += a[i]; }
    s
}

let a: [i32; 4] = [1, 2, 3, 4];
let z: [f64; 3] = [0.5; 3];
println(f"{total(a)} {a.len()} {z:?}");
```

```output
10 4 [0.5, 0.5, 0.5]
```

`const` 参数只能是整数、`bool` 或 `char`。

对常量参数的检查写在 `const { .. }` 块中。它在编译期运行，对函数被调用时用到的每个长度各运行一次，所以 `halves([1, 2, 3])` 是该调用处的编译错误，而不是运行时的 panic。函数内的 `const` 也可以读取 `N`，并对每个长度各有一个值：

```nabi
fn halves<const N: usize>(a: [i32; N]) -> ([i32; N / 2], [i32; N / 2]) {
    const { assert(N % 2 == 0, "halves needs an even length") };
    const HALF: usize = N / 2;
    let l: [i32; N / 2] = [0; N / 2];
    let r: [i32; N / 2] = [0; N / 2];
    for i in 0..HALF {
        l[i] = a[i];
        r[i] = a[HALF + i];
    }
    (l, r)
}

let (l, r) = halves([1, 2, 3, 4]);
println(f"{l:?} {r:?}");
```

```output
[1, 2] [3, 4]
```

## 隐藏返回类型：`some`

迭代器适配器和闭包的类型很长，你不会想把它们写进签名。`-> some Trait` 返回一个具体类型，调用者只把它看作 `Trait`：由函数体选定类型，调用仍是静态的，修改函数体也不必改签名。它同样适用于 `pub` 函数。

```nabi
pub fn evens(n: i32) -> some Iterator<Item = i32> {
    (0..n).filter(|x| x % 2 == 0)
}

fn adder(k: i32) -> some fn(i32) -> i32 {
    |x| x + k
}

println(evens(10).map(|x| x * x).sum());
println(adder(3)(4));
```

```output
120
7
```

调用者只能使用约束提供的功能：`evens(10).contains(4)` 是错误，因为 `Iterator` 没有 `contains`。函数体只返回一个类型；要在不同的路径上返回不同的类型，写 `-> any Trait`。

## 接受任何实现了 trait 的类型：`some` 参数

写作 `x: some Trait` 的参数接受任何具有 `Trait` 的类型，而不必为类型参数起名：`fn show(x: some Display)` 就是 `fn show<T: Display>(x: T)`。每次调用都有一个静态类型，即实参的类型，并且每个 `some` 参数各自是一个独立的类型。

```nabi
fn show(x: some Display) -> str { f"<{x}>" }
fn both(a: some Display, b: some Display) -> str { f"{a} {b}" }
fn twice(f: some fn(i32) -> i32, x: i32) -> i32 { f(f(x)) }

println(show(5));
println(both("hi", 2.5));
println(twice(|x| x * 3, 2));
```

```output
<5>
hi 2.5
18
```

这个类型没有名字，所以调用时无法用 `::<..>` 指定它；需要指定时就声明 `<T: Trait>`。`some` 是 `any Trait` 的静态对应物：`some` 是一个类型，在每次调用时或由函数体固定；而 `any` 可以保存任何这类类型的值，在运行时分派。

## 为隐藏类型命名：`type X = some Trait`

`-> some Trait` 类型没有名字，所以无法保存在结构体字段中，也无法写进其他签名。`type X = some Trait;` 给它一个名字。X 所在模块中返回 `X` 的函数定义了它：它们的函数体选定那一个具体类型，就像 `-> some` 一样。其他所有模块都可以把 `X` 存进字段、`let x: X` 和 `List<X>`，传递它、返回它，但只能看到约束。

```nabi
mod nums {
    pub type Evens = some Iterator<Item = i64>;

    pub fn evens(n: i64) -> Evens {
        (0..n).filter(|x| x % 2 == 0)
    }
}

struct Batch {
    name: str;
    items: nums::Evens;
}

fn total(b: Batch) -> i64 { b.items.sum() }

let batches = [
    Batch { name: "small", items: nums::evens(5) },
    Batch { name: "large", items: nums::evens(10) },
];
for b in batches {
    println(f"{b.name}: {total(b)}");
}
```

```output
small: 6
large: 20
```

在 `nums` 内部，`Evens` 就是具体的 filter 类型，所以模块内的函数可以使用它的全部功能。在外部只有 `Iterator`：`b.items.len()` 是错误，对 `some Display` 类型写 `let n: i64 = ..` 也是错误（[E0146](../errors/E0146.html)）。模块中至少需要一个返回 `Evens` 的函数，而且所有这些函数都必须给出同一个类型（[E0145](../errors/E0145.html)）。调用和字段保持静态：`Batch` 直接保存 filter 本身，没有装箱。`type X = any Trait` 是另一回事，它只是动态槽的普通别名，其中每个值的类型都可以不同。

`some` 只能出现在这三个位置。写作 `some Trait` 的字段或类型实参是错误；像 `Evens` 那样用 `type` 为类型命名，或者声明 `<T: Trait>`：

```nabi norun
struct Label {
    text: some Display;
}
```

```text check=checks/07-some-field.nabi
error[E0010]: `some Trait` is only a function's parameter or return type, or the whole right side of a `type` alias
  --> main.nabi:2:11
   |
 2 |     text: some Display;
   |           ^^^^
   |
   = help: for a variable, leave the type out (`let x = ..`); for a field or a type argument, declare `<T: Trait>` and write `T`, or name the type with `type Name = some Trait;`; for a value of several types, write `any Trait`
```

## 针对单一类型的快速路径

有时某一个元素类型值得拥有专门的代码，比如一个 `f32` 内核，而其他所有类型仍用泛型代码。用 `if T is f32` 直接检查类型参数。在这个分支内 `T` *就是* `f32`：`xs` 是 `List<f32>`，`f32` 结果就是 `T`。每个实例只保留其类型所走的那个分支，所以这个检查在运行时没有开销。

```nabi
fn sum_f32(xs: List<f32>) -> f32 { xs.sum() }

fn total<T: num::Num + Clone>(xs: List<T>) -> T {
    if T is f32 {
        return sum_f32(xs);      // here xs is a List<f32>, and an f32 is a T
    }
    let acc = T::zero();         // every other type: the generic path
    for x in xs { acc = acc + x; }
    acc
}

let a: List<f32> = [1.5, 2.5];
let b: List<i64> = [1, 2, 3];
println(f"{total(a)} {total(b)}");
```

```output
4 6
```

这个分支仍然只在定义处检查一次，并且只有它知道 `T` 是什么：`else` 分支和 `if` 之后的代码都是普通的泛型代码。类型用 `is` 比较，而不是 `==`。

在这种写法和 trait 之间，按类型集合是否开放来选择：

- **使用 trait**：当每个类型都需要自己的行为，并且以后还会加入新类型时。每个类型一个实现正好表达了这一点，没有实现的类型会导致编译错误。
- **使用 `if T is`**：用于保留泛型后备的快速路径。像 `if T is f32 { .. } else if T is f64 { .. } else { panic(..) }` 这样的链条其实是一个伪装的封闭类型集合；`nabi lint` 会标记它（`closed_type_switch`）并建议改用 trait。

## 泛型与 `any`

泛型是静态的。如果你需要的是一个值的类型可以在运行时改变的槽，就**显式**写出 `any`。`any` 值在使用前要用 `is` 收窄。动态分派的开销只出现在你写了 `any` 的地方。

```nabi
let v: any = 1;
println(f"{v is i64}");
v = "text";
if v is str {
    println(f"now a string, length {v.len()}");
}
```

```output
true
now a string, length 4
```

规范把省略参数类型的函数 `fn show_sum(x, y)` 定义为隐式泛型，而不是 `any`。只有非 `pub` 且非 `pub(pkg)` 的函数可以省略参数类型；`pub` 函数要写出每个参数的类型（[E0138](../errors/E0138.html)）。不会推断任何约束：函数在每个调用处单态化，函数体的每份副本都用该调用的实参类型检查，所以错误会指向函数体内部，以及引发它的那次调用。

```nabi
fn show_sum(x, y) { println(f"{x + y}"); }   // implicit generic: fn show_sum<T, U>(x: T, y: U)

show_sum(1, 2);       // 3
show_sum(1.5, 2.0);   // 3.5
```

```output
3
3.5
```

## 小结

- 用 `fn f<T>(..)`、`struct S<A, B>` 和 `impl<A, B> S<A, B>` 声明泛型。类型实参通常可以推断。
- `T: Ord + Clone` 这样的约束决定函数体可以使用哪些操作。约束太长时移到 `where` 子句。
- 类型参数的默认值（`struct Vec3<T = f32>`）在没有其他因素确定该参数时生效；填入它的无后缀字面量采用这个类型。
- 关联类型 `type Item;` 在每个实现中固定为一个类型，除非 trait 有默认值，否则每个实现都必须给出它；在外部写作 `C::Item`。
- `const N: usize` 把一个值作为类型参数。
- `-> some Trait` 返回一个隐藏的具体类型，调用者只能通过 `Trait` 使用；`x: some Trait` 接受任何具有 `Trait` 的类型，每次调用一个；`type X = some Trait;` 为隐藏类型命名，由所在模块的函数确定，可用于字段和签名。
- `if T is f32 { .. }` 在泛型函数体内为某一个类型提供快速路径；trait 适用于开放的类型集合。
- 泛型是静态的（单态化）；`any` 是动态槽，只存在于你写出它的地方。

## 延伸阅读

- [2.10. 泛型](../spec/refs-lang/002-types/210-generics.html)：[2.10.3. trait 约束](../spec/refs-lang/002-types/210-generics.html#2103-trait-bounds)、[2.10.4. where 子句](../spec/refs-lang/002-types/210-generics.html#2104-where-clauses)、[2.10.9. 函数体检查](../spec/refs-lang/002-types/210-generics.html#2109-body-checking)、[2.10.7. 与渐进类型的结合](../spec/refs-lang/002-types/210-generics.html#2107-combination-with-gradual-typing)、[2.10.13. 对类型参数的类型检查](../spec/refs-lang/002-types/210-generics.html#21013-type-tests-on-type-parameters)
- [2.8.3. 关联类型](../spec/refs-lang/002-types/208-trait.html#283-associated-types)
- [2.11. `const` 泛型](../spec/refs-lang/002-types/211-const-generics.html)
- [2.2. `any` 槽](../spec/refs-lang/002-types/202-any-slot.html)
- 下一章：[8. 集合类型与迭代器](08-collections-iterators.html)
