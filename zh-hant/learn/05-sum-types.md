# 5. 和型別與 none

當一個值是「這個或那個」時，Nabi 用 `|` 把型別連起來，寫成 `A | B`。本章介紹和型別（sum type），以及處理其中最常見的 `T | none` 的工具；`T | none` 表示「可能沒有值」。

## 和型別 `A | B`

和型別的值恰好屬於列出的變體之一。值本身帶有標籤，記錄它是哪一個，因此 `match` 可以用型別測試模式 `name: Type` 把它取出來。

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

和型別是**封閉的**。編譯器知道所有可能的變體，所以若從上面的 `match` 移除 `bool` 分支，就會得到編譯錯誤，指出並非所有情況都已處理。新增變體時，編譯器會指出每個需要更新的地方。

和型別的行為像集合。`i32 | str` 和 `str | i32` 是同一個型別，`i32 | i32` 就是 `i32`。較長的和型別可以用 `type` 命名。

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

## `none` 與 `T?`

`none` 是表示「沒有值」的字面值。其他語言使用 `Option<T>` 或 `null` 的地方，Nabi 使用和型別 `T | none`，簡寫為 `T?`。沒有 `Some(..)` 包裝，值直接原樣回傳即可。

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

標準函式庫遵循相同的慣例。可能失敗的查詢，例如 `List::get`、`Dict::get` 和 `List::pop`，都回傳 `T | none`。

> [!NOTE]
> 進階：`T | none` 幾乎沒有成本。對於有未使用位元模式的型別，例如 `bool`、指標和沒有欄位的 `enum`，`none` 存放在其中一個模式裡，不需要額外的標籤（[niche 最佳化](../spec/refs-lang/002-types/207-niche-optimization.html)）。這就是 `bool?` 只占 1 個位元組的原因。

## 用 `is` 收窄

`x is T` 是一個 `bool` 運算式，測試值目前是否為 `T` 變體。當 `if` 條件測試的是區域變數時，該變數在這個分支內會被**收窄**為 `T`，因此可以直接使用 `T` 的操作。之後以 `&&` 連接的條件也是如此。

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

如果值的型別既不是和型別也不是 `any`，`x is T` 在編譯期就會決定。此時沒有後綴的字面值採用其預設型別：在 `let n = 3;` 之後，`n is i64` 為 `true`。

在收窄後的分支內仍然可以對變數指派。指派收窄後型別的值，收窄會繼續保持；指派宣告型別中其他成員的值，收窄就會結束，從那裡到分支結尾，變數又回到宣告的型別。

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

在 `v = "six"` 之後寫 `v + 1` 會是型別錯誤，因為 `i32 | str` 上沒有定義 `+`。

## `if let` 與 `match`

若要同時測試值並把它繫結到新名稱，使用 `if let name = expr`：對 `T | none` 的值而言，`name` 是去掉 `none` 之後的 `T`。只有在要從更寬的和型別中挑出一個成員時，才寫 `if let name: T = expr`。在 `match` 中，`none` 寫成字面值模式。

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

與 `none` 比較同樣會收窄。在 `x != none` 之後、在 `if x == none` 的 `else` 中，以及在 `if x == none { return ..; }` 之下，`x` 都是不含 `none` 的值，直到你指派一個可能是 `none` 的值給它為止。

```nabi
fn shout(name: str?) -> str {
    if name == none { return "nobody"; }
    name.to_uppercase()       // 這裡 name: str
}
println(shout("kim"));
println(shout(none));
```

```output
KIM
nobody
```

## `let else`：提早離開

`let pattern = expr else { ... };` 在模式比對成功時繫結名稱，否則執行 `else` 區塊；這個區塊必須用 `return`、`break`、`continue` 或 `throw` 離開作用域。這樣就能寫出「缺少時提早離開」而不必多一層縮排，而且在它之後，名稱具有收窄後的型別。

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

## 用 `??` 提供預設值，用 `?.` 串接

每次都寫 `match` 很繁瑣。`a ?? b` 在 `a` 為 `none` 時回傳 `b`，否則回傳 `a` 的值。`x?.field` 和 `x?.method()` 在 `x` 為 `none` 時求值為 `none`，只有在有值時才會進行存取。兩者結合，「有就取用，沒有就用預設值」一行就能寫完。

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

`?.` 會短路求值。如果左邊是 `none`，後面的方法根本不會被呼叫。

## 後置 `?`：遇到 none 就提前回傳

當函式必須依序拆開好幾個 `T | none` 值時，使用後置 `?`。`e?` 取出 `e` 的值，如果它是 `none`，就立即從外層函式回傳 `none`。因此函式的回傳型別必須包含 `none`。

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

`?` 只作用於 `T | none`。它不是傳遞錯誤的方式；錯誤用 `throws` / `try` 處理（[錯誤控制流程](../spec/refs-lang/003-expressions/314-error-control-flow.html)）。

> [!NOTE]
> 進階：如果集合本身存放 `none`（`List<i32?>`），`get` 的結果是兩層的可選型別 `(i32 | none)?`，可以區分「沒有元素」與「存放的是 none」。`??`、`?.` 和 `?` 只作用於外層。完整規則請見[巢狀可選型別](../spec/refs-lang/002-types/206-sum-types.html#nested-optional)。

> [!NOTE]
> 進階：`T?` 值在比較和排序時 `none` 排在最前面，所以排序 `[3, none, 1]` 會得到 `[none, 1, 3]`。由兩個值型別組成的和型別，例如 `i32 | str`，沒有順序：對它使用 `<`、`min` / `max` 和 `sort` 都是編譯錯誤。這類值請用 `sort_by_key` 或 `sort_by` 依鍵排序（[標準順序](../spec/refs-lang/002-types/206-sum-types.html#canonical-order)）。

## 小結

- `A | B` 是封閉的和型別。變體的順序和重複都不影響，`match` 必須處理每個變體。
- 值可能不存在時，使用 `T | none`，簡寫為 `T?`。值直接原樣回傳，不需要包裝。
- `x is T`、`x != none` 以及提早的 `if x == none { return .. }` 會收窄 `x`。用 `if let x = ...`、`let x = ... else { return .. };`，以及 `match` 中的 `n: T` / `none` 模式取出值。
- `??` 提供預設值，`?.` 在 none 時略過存取，後置 `?` 在 none 時從函式回傳。

## 延伸閱讀

- [2.6. 和型別](../spec/refs-lang/002-types/206-sum-types.html)、[2.1.5a. 可選型別 `T?`](../spec/refs-lang/002-types/201-type-expressions.html#215a-optional-types-t)
- [2.2.6. 收窄](../spec/refs-lang/002-types/202-any-slot.html#226-narrowing)、[3.10.5. `if let`](../spec/refs-lang/003-expressions/310-if-expression.html#3105-if-let-combining-with-pattern-matching)、[3.16. `let-else`](../spec/refs-lang/003-expressions/316-let-else.html)、[3.11. 模式](../spec/refs-lang/003-expressions/311-patterns.html)
- [3.8. 可選鏈結與空值合併](../spec/refs-lang/003-expressions/308-optional-chaining-and-coalescing.html)、[3.14.10. `?` 運算子](../spec/refs-lang/003-expressions/314-error-control-flow.html#31410-the--operator---early-return-of-none)
- [2.7. Niche 最佳化](../spec/refs-lang/002-types/207-niche-optimization.html)
- 下一章：[6. 結構、列舉與 trait](06-structs-enums-traits.html)
