# 4. 控制流程

本章介紹分支（`if`、`match`）、迴圈（`while`、`for`、`loop`）和解構。重點是：Nabi 的 `if`、`match` 和 `loop` 都是會產生值的運算式。

## `if` 是運算式

`if` 的條件不需要括號，主體一律寫在大括號中。每個分支的最後一個運算式會成為整個 `if` 的值，所以沒有另外的三元運算子。

```nabi
let temp = 23;
let feel = if temp >= 28 {
    "hot"
} else if temp >= 15 {
    "mild"
} else {
    "cold"
};
println(f"{temp} degrees: {feel}");
```

```output
23 degrees: mild
```

當作值使用時，所有分支都必須是相同的型別，或是某個分支的和型別的成員（`x: i32?` 時，`if c { x } else { none }` 是 `i32?`）。若要混用不同的型別，請用 `i32 | str` 這樣的和型別（sum type）標註接收的變數。

## `match` 與模式

`match` 會由上而下把值和模式比對，執行第一個比對成功的分支。你可以使用字面值、以 `|` 連接的多個模式、範圍（`4..=9`）、綁定（`k @ pattern`）、守衛條件（`if condition`），以及萬用字元 `_`。

```nabi
fn describe(n: i32) -> str {
    match n {
        0 => "zero",
        1 | 2 | 3 => "small",
        k @ 4..=9 => f"digit {k}",
        k if k < 0 => "negative",
        _ => "large",
    }
}

for n in [0, 2, 7, -5, 42] {
    println(f"{n}: {describe(n)}");
}
```

```output
0: zero
2: small
7: digit 7
-5: negative
42: large
```

編譯器會檢查是否處理了每一種情況。`i32` 的值多到無法一一列出，所以最後必須有 `_`。元組也可以用模式解構和比對。

```nabi
fn locate(p: (i32, i32)) -> str {
    match p {
        (0, 0) => "origin",
        (x, 0) => f"on the x-axis at {x}",
        (0, y) => f"on the y-axis at {y}",
        (a, b) if a == b => "on the diagonal",
        _ => "somewhere else",
    }
}

println(locate((0, 0)));
println(locate((5, 0)));
println(locate((2, 2)));
println(locate((1, 9)));
```

```output
origin
on the x-axis at 5
on the diagonal
somewhere else
```

模式中單純的名稱（`x`、`a`）會綁定該位置的任何值。所以模式不能靠寫出常數名稱來和常數比較：`LIMIT => ..` 會比對所有的值，因此是錯誤。請改寫成守衛條件 `n if n == LIMIT => ..`。

串列、切片和固定陣列可以用 `[..]` 模式依元素比對。`[]` 是空的，`[a, b]` 是剛好兩個元素，`..` 代表中間任意數量的元素；`..rest` 會把它們綁定為一個切片。

```nabi
fn sum(xs: Slice<i32>) -> i32 {
    match xs {
        [] => 0,
        [first, ..rest] => first + sum(rest),
    }
}

fn ends(xs: List<str>) -> str {
    match xs {
        [] => "nothing",
        [only] => f"just {only}",
        [first, .., last] => f"{first} to {last}",
    }
}

let xs: List<i32> = [1, 2, 3, 4];
println(sum(xs[:]));
println(ends(["a"]));
println(ends(["a", "b", "c"]));
let arr: [i32; 3] = [10, 20, 30];
let [x, y, z] = arr;
println(x + y + z);
```

```output
10
just a
a to c
60
```

編譯器也會檢查長度：如果沒有 `[first, .., last]` 分支，`ends` 就會漏掉有兩個以上元素的串列，錯誤訊息會用 `[_, _, ..]` 模式指出這一點。一般的 `let` 可以拆開固定陣列，因為它的長度已知；串列的長度未知，所以請寫 `let [first, ..rest] = xs else { .. };` 或使用 `match`。

當值可能是幾種型別之一（`i32 | str | none`）時，`match` 可以用 `n: i32` 這類模式依型別分流。這些模式，以及 `is`、`if let` 和 `let else`，會在[下一章](05-sum-types.html)與和型別一起介紹。

## `while` 與 `for`

`while` 會在條件為真時持續重複。`for` 會從範圍、串列或迭代器中一次取出一個元素。`a..b` 不含 `b`；`a..=b` 則包含。範圍也能用於 `in`：`if i in 0..n { .. }` 不需要括號。`continue` 進入下一次迭代，`break` 離開迴圈。

```nabi
let i = 0;
while i < 3 {
    print(f"{i} ");
    i += 1;
}
println("");

for k in 0..10 {
    if k % 2 == 1 { continue; }
    if k > 6 { break; }
    print(f"{k} ");
}
println("");

for (idx, name) in ["ada", "grace"].enumerate() {
    println(f"{idx}: {name}");
}
```

```output
0 1 2 
0 2 4 6 
0: ada
1: grace
```

`while let` 會在模式比對成功時持續重複。`pop()` 回傳 `T | none`，所以要把串列一路取到空為止就很簡單。

```nabi
let stack = [1, 2, 3];
while let top = stack.pop() {
    print(f"{top} ");
}
println("");
```

```output
3 2 1 
```

## `loop` 與帶值的 `break`

`loop` 會一直重複，直到遇到 `break`。如果用 `break value` 離開，該值就是 `loop` 運算式的結果。這很適合結果要在結束那一刻才能確定的計算。

```nabi
let n = 27;
let steps = 0;
let total = loop {
    if n == 1 { break steps; }
    n = if n % 2 == 0 { n / 2 } else { 3 * n + 1 };
    steps += 1;
};
println(f"Collatz(27) reaches 1 in {total} steps");
```

```output
Collatz(27) reaches 1 in 111 steps
```

## 標籤

若要從巢狀迴圈中直接結束外層迴圈，請加上 `'name:` 標籤，並使用 `break 'name` 或 `continue 'name`。`loop` 和區塊上的標籤也可以把值帶出來。

```nabi
let pair = 'search: loop {
    for a in 1..10 {
        for b in a..10 {
            if a * b == 42 { break 'search (a, b); }
        }
    }
    break (0, 0);
};
println(f"{pair:?}");

let limit = 50;
let checked = 'check: {
    if limit > 100 { break 'check -1; }
    limit * 2
};
println(f"{checked}");
```

```output
(6, 7)
100
```

## 解構

`let` 和 `for` 的左邊可以是任何一定會比對成功的模式（不可反駁模式，irrefutable pattern）。可以拆開巢狀元組，用 `..` 收集其餘部分，或用 `_` 丟棄。

```nabi
let ((a, b), c) = ((1, 2), 3);
let (first, ..rest) = (10, 20, 30, 40);
let (_, second) = ("skip", "keep");
println(f"{a} {b} {c} {first} {rest:?} {second}");

let points = [(1, 2), (3, 4)];
for (x, y) in points {
    println(f"({x}, {y}) -> {x + y}");
}
```

```output
1 2 3 10 (20, 30, 40) keep
(1, 2) -> 3
(3, 4) -> 7
```

```nabi
let a = 0;
let b = 1;
for _ in 0..10 {
    (a, b) = (b, a + b);
}
println(f"{a} {b}");

let xs = [1, 2, 3];
(xs[0], xs[2]) = (xs[2], xs[0]);
println(f"{xs:?}");
```

```output
55 89
[3, 2, 1]
```

不帶 `let` 時，`=` 左邊由既有位置組成的元組會逐一指派給每個位置：變數、欄位、元素、巢狀元組和 `_`。右邊會先完整求值，所以 `(a, b) = (b, a + b)` 不需要暫存變數，而 `(xs[i], xs[j]) = (xs[j], xs[i])` 會交換兩個元素。閉包和函式的參數也接受相同的模式：`|(k, v)| ..`、`fn f((x, y): (i32, i32))`（[第 3 章](03-functions.html)）。

## 小結

- `if`、`match`、`loop` 和區塊都是會產生值的運算式。各分支必須是相同的型別。
- `match` 支援字面值、`|`、範圍、`@` 綁定、守衛條件和切片模式（`[first, ..rest]`），並檢查窮盡性。
- `while let` 會在模式比對成功時持續重複。
- `for x in a..b`／`a..=b` 迭代範圍；`xs.enumerate()` 連同索引一起迭代；`x in a..b` 檢查是否在範圍內。
- `(a, b) = (b, a + b)` 會先求出右邊的值，再一次指派給多個位置。
- `break value` 和 `'label:` 讓你離開巢狀迴圈並回傳結果。

## 延伸閱讀

- [`if` 運算式](../spec/refs-lang/003-expressions/310-if-expression.html)、[`match` 運算式](../spec/refs-lang/003-expressions/312-match-expression.html)、[模式](../spec/refs-lang/003-expressions/311-patterns.html)
- [迴圈運算式](../spec/refs-lang/003-expressions/313-loop-expressions.html)、[範圍運算式](../spec/refs-lang/003-expressions/307-range-expressions.html)、[帶標籤的區塊](../spec/refs-lang/003-expressions/309-block-expressions.html)
- [解構指派](../spec/refs-lang/003-expressions/315-destructuring.html)、[`let-else`](../spec/refs-lang/003-expressions/316-let-else.html)
- [`is` 與槽位上的型別檢查](../spec/refs-lang/002-types/202-any-slot.html)、[和型別](../spec/refs-lang/002-types/206-sum-types.html)
- 下一章：[5. 和型別與 none](05-sum-types.html)
