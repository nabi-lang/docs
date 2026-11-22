# 3. 函式

本章介紹如何宣告和呼叫函式、預設值與具名引數，以及可以當作值傳遞的閉包。在 Nabi 中，函式主體也是一個區塊運算式，所以它的最後一個運算式就是回傳值。

## 宣告與回傳值

用 `fn name(param: Type, ...) -> ReturnType { body }` 宣告函式。主體的最後一個運算式（不帶分號的那一個）就是回傳值；`return` 只用於提前離開。

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

要回傳多個值時，把它們組成元組，再在呼叫處解構。

區塊內的陳述式以 `;` 結尾，但 `}` 之前的最後一個陳述式可以省略它，意思相同。所以 `while i < n { i += 1 }` 和 `{ i += 1; }` 相同，最後的 `let`、`return`、`break` 或 `throw` 也一樣。只有最後一個運算式會成為區塊的值，就像上面的 `a + b`。

回傳型別和參數型別都可以省略。省略的回傳型別會從最後一個運算式推論（如果以 `;` 結尾則為 `()`）。沒有型別的參數會讓函式成為泛型函式，依呼叫時的每種引數型別分別編譯。

```nabi
fn double(x: i32) { x * 2 }   // return type inferred: i32
fn id(x) { x }                // generic: fn id<T>(x: T) -> T
println(f"{double(4)} {id("hi")}");
```

```output
8 hi
```

`pub` 或 `pub(pkg)` 函式則不同：其他模組會呼叫它，所以它要寫出完整的簽章：每個參數都有型別，沒有 `->` 時回傳 `()`（`main` 和產生器 `yield fn` 除外）。主體產生值時，編譯器會告訴你該寫哪一個 `-> T`（[E0138](../errors/E0138.html)）。

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

區塊 `{ ... }` 在函式主體之外也是運算式。用它可以把暫時的值留在區塊內，只取出結果。

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

## 預設值與具名引數

參數可以用 `= value` 設定預設值，呼叫時可以用 `name: value` 以名稱傳入引數。位置引數在前，具名引數在後。

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

預設值必須在編譯時期就能確定，例如字面值或 `const`。不定數量的引數用 `...T` 接收；在主體中它是一個唯讀的切片。

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

## 閉包

閉包（closure）是匿名的函式值，寫作 `|params| expr` 或 `|params| { block }`。參數型別會從上下文推論；沒有上下文時，請寫出型別。閉包或函式的參數可以是一定會比對成功的模式，和 `let` 相同：`|(a, b)| a + b` 接收一個元組，`|a, b|` 接收兩個值，而 `fn dist((x, y): (f64, f64))` 會解構它的引數。

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

閉包會**複製值**來擷取周圍的變數。如果外部變數在閉包建立之後改變，閉包仍然記得建立當時的值。若要修改外部變數本身，請用 `[ref name]` 擷取清單要求以參考方式擷取。

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

如果你嘗試指派給以值擷取的變數，或以值擷取的結構的欄位，會得到錯誤，並告訴你加上 `[ref count]`。`List` 這類堆積上的物件則不同：變數是指向物件的參考，所以即使以值擷取，也能修改物件本身，例如用 `push`。

`[ref name]` 的反面是 `[move name]`：它把值交給閉包。閉包會取得內部串列的自有複本，建立閉包的程式碼就不能再使用這個名稱。串列就是這樣傳到另一個執行緒的（[第 12 章](12-concurrency.html)）：

```nabi
use std::thread;

let rows = ["a", "b", "c"];
let h = thread::spawn([move rows] || rows.len());
println(f"rows: {try! h.join()}");
```

```output
rows: 3
```

## 函式型別

函式和閉包都是型別為 `fn(ParamTypes) -> ReturnType` 的值。你可以把它們存在變數中、當作引數傳遞、回傳，或放進串列。

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

`make_adder` 回傳的閉包擷取了 `n`，並持續存在。

## 尾隨閉包

如果最後一個引數是閉包，可以把它移到括號外：有參數時寫 `f(args) |x| { ... }`，沒有參數時寫 `f(args) { ... }`。這樣迭代和資源管理之類的模式，讀起來就像內建語法。

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

位於陳述式開頭的尾隨閉包呼叫，例如上面的 `repeat(3) |i| { .. }`，會在它的 `}` 結束，就像後面接了一個 `;`：不需要寫 `;`，而以運算子開頭的下一行（`-1`）是獨立的陳述式。它和其他運算式陳述式一樣，所以非 unit 的結果會被丟棄，`unused_result` lint 會對此發出警告（若是刻意丟棄，請寫 `_ = f(..) { .. };`）。在較大的運算式中，例如 `let total = measure("sum") { 6 } - 1;`，運算子會延續這個呼叫。

在這種閉包中單獨寫 `return`，只會結束閉包的這一次呼叫，而不是外層的函式，所以這是錯誤。若要結束這次呼叫，請寫出接收閉包的函式名稱：`return@each`（帶值時寫 `return@find x`）。若要離開外層函式，請改用 `for` 迴圈。

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

## 管線：`|>`

`x |> f` 會呼叫 `f(x)`，`x |> f(a)` 會呼叫 `f(x, a)`：左邊的值會成為第一個引數。這樣一連串的自由函式，讀起來就是它們執行的順序。`|>` 的結合力比其他所有運算子都弱，所以會傳入它左邊的整個運算式（`1 + 2 |> double` 就是 `double(3)`），而以 `|>` 開頭的一行會延續上一行。右邊是函式、呼叫，或放在括號中的閉包；方法仍然使用 `.`（`s.trim()`）。

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

## 方法預覽

`impl` 區塊中的函式就是方法。如果第一個參數是 `self`，就用 `value.method()` 呼叫；沒有 `self` 時，它是靜態方法，用 `Type::function()` 呼叫。要在方法中修改欄位，請接收 `ref self`。結構和方法會在後面的章節詳細介紹。

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

## 巢狀函式與進入點

你可以在函式內宣告函式。巢狀函式只在它的區塊內看得到，而且和閉包不同，它**不會**擷取外部變數。用它來隱藏不需要外部狀態的輔助函式。

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

程式從 `src/main.nabi`（或單獨檔案）的頂層陳述式，或從 `fn main()` 開始執行。頂層陳述式會成為編譯器產生的隱含 `main` 的主體，所以頂層的 `let` 變數是**區域變數**，在其他函式的主體中看不到。如果函式需要這樣的值，請把它當作引數傳入，或用閉包擷取。

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

## 小結

- 函式主體的最後一個運算式就是它的回傳值。用 `return` 提前離開。
- 參數可以有預設值，呼叫時可以用 `name: value` 指名引數。`...T` 接收不定數量的引數。
- 閉包 `|x| expr` 預設以複製值的方式擷取；`[ref x]` 要求以參考方式擷取，`[move x]` 則把 `x` 交給閉包。
- 函式值的型別是 `fn(A) -> R`。位於最後一個引數位置的閉包可以寫成尾隨形式；在其中，`return@name` 會結束那一次呼叫。
- `x |> f(a)` 就是 `f(x, a)`：管線把左邊的值傳給函式，是結合力最弱的運算子。
- 頂層的 `let` 是隱含 `main` 的區域變數，所以其他函式看不到它。

## 延伸閱讀

- [函式宣告](../spec/refs-lang/004-functions/401-function-declaration.html)、[參數](../spec/refs-lang/004-functions/402-parameters.html)、[return](../spec/refs-lang/004-functions/403-return.html)
- [函式型別](../spec/refs-lang/004-functions/404-function-type.html)、[閉包](../spec/refs-lang/004-functions/405-closure.html)、[擷取語意](../spec/refs-lang/004-functions/406-capture-semantics.html)
- [尾隨閉包](../spec/refs-lang/004-functions/408-trailing-closure.html)、[方法](../spec/refs-lang/004-functions/409-method.html)、[巢狀函式](../spec/refs-lang/004-functions/410-nested-functions.html)
- [進入點](../spec/refs-lang/004-functions/416-entry-point.html)、[區塊運算式](../spec/refs-lang/003-expressions/309-block-expressions.html)、[管線運算式](../spec/refs-lang/003-expressions/318-pipe-expression.html)
- 下一章：[4. 控制流程](04-control-flow.html)
