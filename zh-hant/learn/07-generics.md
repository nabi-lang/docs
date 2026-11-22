# 7. 泛型

想讓同一套邏輯適用於多種型別時，就使用泛型（generics）。Nabi 的泛型會針對每個呼叫時用到的具體型別分別編譯（單態化，monomorphization），因此執行速度和為每個型別手寫的函式一樣快。

## 泛型函式

在函式名稱後的 `<T>` 中宣告型別參數，並在參數型別與回傳型別中使用它們。呼叫時通常不必寫出型別；編譯器會從引數推論。

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

在一次呼叫中，`T` 是單一型別。像 `first_or([1, 2], "x")` 這樣傳入不同型別，是編譯錯誤。

## trait 約束

對沒有約束的 `T`，你能做的只有搬移值。若要比較或相加，就用 `T: Ord` 這樣的**約束**（bound）要求所需的 trait。多個約束用 `+` 連接。編譯器只依據約束來檢查泛型主體，因此不論用哪個型別呼叫，主體都不會出錯。

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

有了 `T: num::Zero`，你也可以在型別參數上呼叫靜態方法，例如 `T::zero()`。

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

### 主體依約束檢查

泛型主體會在檢視任何呼叫之前，單獨進行一次型別檢查。在主體內，`T` 只支援它的約束所承諾的操作，再加上每個型別都支援的操作：搬移、存入 `any`、組成 `T | none`，以及 `is`。忘了加約束會在主體中產生錯誤，即使每個呼叫傳入的型別其實都行得通。

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

加上約束後，同一個主體對每個具有 `Add` 的型別都能編譯。不需要約束的操作，沒有約束也能使用。

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

由於這條規則，函式庫中的泛型函式只要在你這裡能編譯，對所有呼叫它的人也都能編譯；錯誤絕不會出現在遠處的呼叫點。trait 的預設方法也以同樣方式檢查，`Self` 只能透過 trait 得知。參數沒有寫型別的函式（隱含泛型，見下文）是例外。

一個約束可以用不同的參數兩次指名同一個 trait。主體中的每次使用，會挑選參數與引數型別相符的約束，就像運算子在某個型別的多個實作之間挑選一樣。

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

約束變長時，把它們移到簽章結尾的 `where` 子句，會比較容易閱讀。意義完全相同。

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

## 泛型結構與 impl

結構和列舉也可以接受型別參數。新增方法時，也要在 `impl` 上宣告參數，例如 `impl<A, B> Pair<A, B>`。

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

### 預設值

型別參數可以宣告預設值：`struct Vec3<T = f32>`。寫成型別時，`Vec3` 就是 `Vec3<f32>`。在運算式中，參數和其他參數一樣經由推論決定，順序如下：明確寫出的引數（`Vec3::<f64>::new`），接著是預期型別與後續的使用，然後是參數的預設值，最後是字面值的預設型別 `i64` / `f64`。所以填入 `T` 的無後綴字面值，除非另有指定，否則是 `f32`。過程中不做任何轉換：有型別的引數本身就會決定 `T`。

```nabi
struct Vec3<T = f32> { x: T; y: T; z: T; }

impl<T> Vec3<T> {
    fn new(x: T, y: T, z: T) -> Vec3<T> { Vec3 { x: x, y: y, z: z } }
}

impl Vec3<f32> { const X: Vec3<f32> = Vec3 { x: 1.0, y: 0.0, z: 0.0 }; }
impl Vec3<f64> { const X: Vec3<f64> = Vec3 { x: 1.0, y: 0.0, z: 0.0 }; }

let a = Vec3::new(1.0, 2.0, 3.0);           // 預設值：Vec3<f32>
let b: Vec3<f64> = Vec3::new(1.0, 2.0, 3.0); // 預期型別優先
let x: f64 = 0.5;
let c = Vec3::new(x, 0.0, 0.0);             // x 決定 T：Vec3<f64>
let d: Vec3<f64> = Vec3::X;                 // Vec3<f64> 的 X
println(f"{a.type_name_of_val()} {b.type_name_of_val()} {c.type_name_of_val()} {d.x}");
```

```output
Vec3 Vec3<f64> Vec3<f64> 1
```

`Vec3::X` 也以同樣方式決定實例：依預期型別；沒有的話，依唯一定義了 `X` 的那個實作；再沒有的話，就用預設值。函式的預設值（`fn zero<T: Num = f32>() -> T`）同樣是後備：`let r = zero();` 是 `f32`，除非之後對 `r` 的使用需要其他型別。

## 關聯型別

若要讓 trait 的每個實作各自選定一個型別，就宣告**關聯型別**（associated type）`type Item;`。在 trait 內部直接寫 `Item`；在外部的泛型程式碼中寫 `C::Item`。標準的 `Iterator` 就是這樣運作的（下一章）。

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

這裡用關聯型別而不是型別參數，是因為對某個實作者而言，`Item` 恰好固定為一個型別。使用者不必寫出 `Shelf` 的元素型別。trait 宣告的每個沒有預設值的關聯型別，每個實作都必須提供：省略 `type Item = ..;` 是錯誤（[E0072](../errors/E0072.html)），其 help 會寫出這一行；如果實作的方法提到了該型別，就從方法中取用。

## 進階：常數泛型

第一次閱讀時可以跳過本節。參數也可以是編譯期的**值**，而不是型別。當某個值是型別的一部分時就會用到，例如固定長度陣列 `[T; N]` 的長度。

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

`const` 參數只能是整數、`bool` 或 `char`。

對 const 參數的檢查寫在 `const { .. }` 區塊中。它在編譯期執行，對函式被呼叫時的每個長度各執行一次，因此 `halves([1, 2, 3])` 會在那個呼叫處成為編譯錯誤，而不是在執行時觸發 panic。函式內的 `const` 也可以讀取 `N`，並對每個長度各有一個值：

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

## 隱藏回傳型別：`some`

迭代器配接器和閉包的型別很長，你不會想把它們寫進簽章。`-> some Trait` 回傳一個具體型別，而呼叫端只把它看作 `Trait`：主體決定這個型別，呼叫仍然是靜態的，而且你可以修改主體而不必改動簽章。它也能用在 `pub` 函式上。

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

呼叫端只能使用約束所提供的功能：`evens(10).contains(4)` 是錯誤，因為 `Iterator` 沒有 `contains`。主體只回傳一個型別；若要從不同的路徑回傳不同的型別，請寫 `-> any Trait`。

## 接受任何具有某 trait 的型別：`some` 參數

寫成 `x: some Trait` 的參數接受任何具有 `Trait` 的型別，而不必為型別參數命名：`fn show(x: some Display)` 就是 `fn show<T: Display>(x: T)`。每次呼叫都有一個靜態型別，也就是引數的型別，而每個 `some` 參數各自是獨立的型別。

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

這個型別沒有名稱，所以呼叫時無法用 `::<..>` 指定它；需要指定時，請宣告 `<T: Trait>`。`some` 是 `any Trait` 的靜態對應：`some` 是一個型別，在每次呼叫時或由主體決定；而 `any` 可以存放任何符合該 trait 的型別的值，並在執行時分派。

## 為隱藏型別命名：`type X = some Trait`

`-> some Trait` 型別沒有名稱，所以你無法把它存在結構欄位中，也無法寫進其他簽章。`type X = some Trait;` 為它取一個名稱。X 所在模組中回傳 `X` 的函式負責定義它：和 `-> some` 一樣，由這些函式的主體決定唯一的具體型別。其他所有模組都可以把 `X` 存進欄位、`let x: X` 和 `List<X>`，傳遞它、回傳它，而且只看得到約束。

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

在 `nums` 內部，`Evens` 就是具體的 filter 型別，因此其中的函式可以使用它的全部功能。在外部則只有 `Iterator`：`b.items.len()` 是錯誤，對 `some Display` 型別寫 `let n: i64 = ..` 也是錯誤（[E0146](../errors/E0146.html)）。模組至少需要一個回傳 `Evens` 的函式，而且這些函式必須給出相同的型別（[E0145](../errors/E0145.html)）。呼叫和欄位仍然是靜態的：`Batch` 直接存放 filter 本身，不經過任何封裝。`type X = any Trait` 則是另一回事，它只是動態槽位的普通別名，其中每個值都可能是不同的型別。

`some` 只能出現在這三個地方。寫成 `some Trait` 的欄位或型別引數是錯誤；請像 `Evens` 那樣用 `type` 為型別命名，或宣告 `<T: Trait>`：

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

## 為單一型別提供快速路徑

有時某個元素型別值得擁有自己的程式碼，例如 `f32` 核心函式，而其他所有型別則沿用泛型版本。這時直接用 `if T is f32` 測試型別參數本身。在這個分支內，`T` *就是* `f32`：`xs` 是 `List<f32>`，而 `f32` 的結果就是 `T`。每個實例只保留其型別會走的分支，因此這項測試在執行時沒有任何成本。

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

這個分支仍然只在定義處檢查一次，而且只有它知道 `T` 是什麼：`else` 分支和 `if` 之後的程式碼都是一般的泛型程式碼。型別用 `is` 比較，而不是 `==`。

要在這種寫法與 trait 之間選擇，看型別的集合是否開放：

- **使用 trait**：當每個型別都需要自己的行為，而且之後還會有新型別加入時。每個型別一個實作正好表達了這一點，而沒有實作的型別會是編譯錯誤。
- **使用 `if T is`**：用於保留泛型後備的快速路徑。像 `if T is f32 { .. } else if T is f64 { .. } else { panic(..) }` 這樣的鏈，其實是換了個樣子的封閉型別集合；`nabi lint` 會標示它（`closed_type_switch`）並建議改用 trait。

## 泛型與 `any`

泛型是靜態的。如果你需要的是一個值的型別可能在執行時改變的槽位，就**明確地**寫 `any`。使用 `any` 值之前，先用 `is` 收窄。動態分派的成本只會出現在你寫了 `any` 的地方。

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

規格把省略參數型別的函式 `fn show_sum(x, y)` 定義為隱含泛型，而不是 `any`。只有不是 `pub` 或 `pub(pkg)` 的函式可以省略參數型別；`pub` 函式必須寫出每個參數的型別（[E0138](../errors/E0138.html)）。約束不會被推論：函式在每個呼叫點各自單態化，主體的每份複本都以該次呼叫的引數型別檢查，因此錯誤會同時指向主體內部與造成錯誤的呼叫。

```nabi
fn show_sum(x, y) { println(f"{x + y}"); }   // implicit generic: fn show_sum<T, U>(x: T, y: U)

show_sum(1, 2);       // 3
show_sum(1.5, 2.0);   // 3.5
```

```output
3
3.5
```

## 小結

- 用 `fn f<T>(..)`、`struct S<A, B>` 和 `impl<A, B> S<A, B>` 宣告泛型。型別引數通常由推論決定。
- `T: Ord + Clone` 這樣的約束決定主體可以使用哪些操作。較長的約束可以移到 `where` 子句。
- 型別參數的預設值（`struct Vec3<T = f32>`）在沒有其他因素決定該參數時套用；填入該參數的無後綴字面值會採用它。
- 關聯型別 `type Item;` 對每個實作固定為一個型別，除非 trait 提供了預設值，否則每個實作都必須給出；從外部寫成 `C::Item`。
- `const N: usize` 以值作為型別參數。
- `-> some Trait` 回傳一個隱藏的具體型別，呼叫端只能透過 `Trait` 使用它；`x: some Trait` 接受任何具有 `Trait` 的型別，每次呼叫一個；`type X = some Trait;` 為隱藏型別命名，由其模組的函式決定，可用於欄位與簽章。
- `if T is f32 { .. }` 在泛型主體中為某個型別提供快速路徑；開放的型別集合則用 trait。
- 泛型是靜態的（單態化）；`any` 是只存在於你寫下它之處的動態槽位。

## 延伸閱讀

- [2.10. 泛型](../spec/refs-lang/002-types/210-generics.html)：[2.10.3. trait 約束](../spec/refs-lang/002-types/210-generics.html#2103-trait-約束)、[2.10.4. where 子句](../spec/refs-lang/002-types/210-generics.html#2104-where-子句)、[2.10.9. 主體檢查](../spec/refs-lang/002-types/210-generics.html#2109-主體檢查)、[2.10.7. 與漸進型別的互動](../spec/refs-lang/002-types/210-generics.html#2107-與漸進式型別結合)、[2.10.13. 對型別參數的型別測試](../spec/refs-lang/002-types/210-generics.html#21013-對型別參數的型別測試)
- [2.8.3. 關聯型別](../spec/refs-lang/002-types/208-trait.html#283-關聯型別)
- [2.11. `const` 泛型](../spec/refs-lang/002-types/211-const-generics.html)
- [2.2. `any` 槽位](../spec/refs-lang/002-types/202-any-slot.html)
- 下一章：[8. 集合型別與迭代器](08-collections-iterators.html)
