# 8. 集合型別與迭代器

大多數程式都要收集、走訪和轉換大量的值。本章介紹標準集合型別（collection）——`List`、`Dict`、`Set` 及其相關型別——以及切片，還有把它們串在一起的 `Iterator`。

## List

`[1, 2, 3]` 是動態陣列 `List<T>`。索引從 0 開始，負索引從尾端往回數。超出範圍的 `xs[i]` 會觸發 panic，所以讀取可能不存在的位置時要用 `get`，它回傳 `T | none`。

```nabi
let xs = [5, 3, 8, 1];
xs.push(7);
println(f"{xs:?} len={xs.len()}");
println(f"{xs[0]} {xs[-1]} {xs.get(10):?}");

xs.sort();
let top = xs.pop();
println(f"{xs:?} popped={top:?}");

xs.insert(0, 0);
xs.remove(1);
println(f"{xs:?} first={xs.first():?}");
```

```output
[5, 3, 8, 1, 7] len=5
5 7 none
[1, 3, 5, 7] popped=8
[0, 3, 5, 7] first=0
```

`List`、`Dict` 和 `Set` 是**參考控制代碼**（reference handle）。`let ys = xs;` 和當作函式引數傳遞時，共用的都是同一個集合；需要獨立的副本時，呼叫 `clone()`。

```nabi
let xs = [1, 2];
let shared = xs;
let copy = xs.clone();
shared.push(3);
copy.push(99);
println(f"{xs:?} {copy:?}");
```

```output
[1, 2, 3] [1, 2, 99]
```

## 切片

`xs[start:end:step]` 的意思和 Python 相同。邊界會被限制在長度之內，所以切片不會因為範圍而觸發 panic。對 `List` 切片會得到 `Slice<T>`，它是指向原本緩衝區的檢視；需要獨立的串列時，用 `to_list()` 複製。

```nabi
let xs = [10, 20, 30, 40, 50];
println(f"{xs[1:3]:?} {xs[:2]:?} {xs[-2:]:?}");
println(f"{xs[::2]:?} {xs[::-1]:?} {xs[3:100]:?}");

let part = xs[1:4];
println(f"{part.len()} {part[0]} {part.to_list():?}");
```

```output
[20, 30] [10, 20] [40, 50]
[10, 30, 50] [50, 40, 30, 20, 10] [40, 50]
3 20 [20, 30, 40]
```

`Slice` 是唯讀的。要填入或重新排列串列一部分的函式，改為接受 `SliceMut<T>`，它是可寫入的檢視，就像 Rust 的 `&mut [T]`。在預期 `SliceMut` 的地方，`xs` 傳入整個串列，`xs[a:b]` 傳入該範圍；不會複製任何東西，寫入會直接作用在串列上。

```nabi
fn zero(buf: SliceMut<i32>) { buf.fill(0); }

let ys = [5, 4, 3, 2, 1];
let mid: SliceMut<i32> = ys[1:4];
mid.sort();
mid[0] = 20;
zero(ys[4:]);
println(f"{ys:?}");
```

```output
[5, 20, 3, 4, 0]
```

串列改變大小（`push`、`insert`、`clear` 等）之後，指向它的檢視就失效了：在那之後使用檢視，編譯器看得出來時是錯誤，看不出來時會觸發 panic。改變大小之後，請重新建立檢視。

## Dict 與 Set

`{"key": value}` 是雜湊對應 `Dict<K, V>`，`{1, 2}` 是雜湊集合 `Set<T>`。空的 `{}` 是區塊，所以空集合型別要用 `Dict::new()` / `Set::new()` 建立。鍵不存在時 `d[k]` 會觸發 panic；`d.get(k)` 回傳 `V | none`。`k in d` 檢查鍵是否存在，`x in s` 檢查是否為集合成員（相反的檢查用 `!in`）。

```nabi
let ages: Dict<str, i32> = {"kim": 30, "lee": 25};
ages["park"] = 41;
ages.remove("lee");
println(f"{ages["kim"]} {ages.get("choi") ?? -1} {"park" in ages} {ages.len()}");

let counts: Dict<str, i32> = Dict::new();
for w in ["a", "b", "a", "a"] {
    counts[w, default: 0] += 1;
}
println(f"a={counts["a"]} b={counts["b"]}");

let s: Set<i32> = {1, 2, 3};
let t: Set<i32> = {2, 3, 4};
println(f"{2 in s} {s.intersection(t).len()} {s.union(t).len()}");
println(f"{s & t:?} {s | t:?} {s - t:?} {s ^ t:?}");
println(f"{counts["z", default: 0]} {[1, 2] + [3]:?}");
```

```output
30 -1 true 2
a=3 b=1
true 2 4
{2, 3} {1, 2, 3, 4} {1} {1, 4}
0 [1, 2, 3]
```

`d[k, default: v]` 在鍵不存在時讀到 `v`；當它是 `=` 或 `+=` 的目標，或是方法的接收者時，會先把 `v` 存進去。集合上的 `&`、`|`、`-` 和 `^` 分別是交集、聯集、差集和對稱差集，`+` 則把兩個串列接成一個新串列。

在 `ages.remove("lee")` 之後，如果第四行是 `println(f"{ages["lee"]}");`，就會觸發 panic，訊息會告訴你應該改寫成什麼：

```text
thread 'main' panicked at main.nabi:4:12:
key "lee" not found; use `d.get(k)` to get `i32 | none`
```

> [!NOTE]
> `Dict` 和 `Set` 依插入順序走訪。再次插入的鍵保留原本的位置，移除後再插入的鍵則排到最後。雜湊值本身每次執行都會改變，所以絕對不要仰賴它。

## 插入順序與 Deque

因為 `Dict` 和 `Set` 會保留插入順序，所以沒有另外的有序型別：`iter()` 依插入的順序給出 `(key, value)` 配對。`Deque` 是兩端都能推入和取出的佇列。

```nabi
let d: Dict<str, i32> = Dict::new();
d.insert("zebra", 1);
d.insert("apple", 2);
for (k, v) in d.iter() { println(f"{k}={v}"); }

let tags = Set::from(["b", "a", "b", "c"]);
println(f"{tags:?}");

let q: Deque<i32> = Deque::new();
q.push_back(1);
q.push_back(2);
q.push_front(0);
println(f"{q:?}");
let front = q.pop_front();
let back = q.pop_back();
println(f"{front:?} {back:?} {q:?}");
```

```output
zebra=1
apple=2
{"b", "a", "c"}
[0, 1, 2]
0 2 [1]
```

## 有序對應與有序集合

當你需要的順序是鍵本身的順序，而不是插入的順序時，請用 `core::collections` 裡的 `SortedDict` 和 `SortedSet`。它們讓鍵保持排序（B-tree），所以第一個和最後一個鍵、小於等於某個值的最近鍵（`floor`）或大於它的最近鍵（`ceil`），以及某個範圍內的所有鍵，都能以很低的成本找到。

```nabi
use core::collections::{SortedDict, SortedSet};

let prices: SortedDict<str, f64> = SortedDict::new();
prices["pear"] = 1.25;
prices["apple"] = 0.5;
prices["fig"] = 2.0;
println(f"{prices:?}");
println(f"{prices.first():?} {prices.last():?}");

let log: SortedDict<i64, str> = SortedDict::from([(1650, "power"), (1700, "boot"), (1720, "login")]);
for (t, what) in log.range(1600..1710) {
    println(f"{t} {what}");
}
println(f"{log.floor(1710):?} {log.ceil(1710):?} {log.range_rev(1700..).map(|(t, _)| t).collect::<List<i64>>():?}");

let seen: SortedSet<i64> = SortedSet::from([42, 7, 19, 7]);
println(f"{seen:?} {seen.range(10..50).collect::<List<i64>>():?} {seen.lower(19):?}");
```

```output
{"apple": 0.5, "fig": 2.0, "pear": 1.25}
("apple", 0.5) ("pear", 1.25)
1650 power
1700 boot
(1700, "boot") (1720, "login") [1720, 1700]
{7, 19, 42} [19, 42] 7
```

查詢、插入和移除都是 O(log n)，而 `Dict` 平均是 O(1)；需要順序時才用有序對應。`range` 接受任何範圍：`a..b`、`a..=b`、`a..`，以及放在括號裡的 `(..b)`。

## 迭代器與配接器

`for x in collection` 驅動的是 `Iterator`。在集合的 `iter()` 回傳的迭代器上，可以串接 `map`、`filter`、`enumerate`、`zip`、`take`、`skip`、`rev`、`step_by` 和 `chain` 等**配接器**（adapter）。配接器是惰性的：只有在呼叫 `collect`、`sum`、`fold` 或 `count` 這類真正消耗值的方法時才會執行。

```nabi
let xs = [1, 2, 3, 4, 5, 6];

let evens = xs.filter(|x| x % 2 == 0).map(|x| x * 10).to_list();
println(evens);
println(xs.sum(), xs.count(), xs.any(|x| x > 5));
println(xs.fold(0, |acc, x| acc + x * x));
println(f"{xs.iter().find(|x| x > 3):?} {xs.position(|x| x == 3):?} {xs.max():?}");

for (i, x) in xs.enumerate().skip(4) { println(f"{i}: {x}"); }

let names = ["a", "b", "c"];
let pairs = names.zip(xs.iter()).map(|(n, x)| f"{n}{x}").to_list();
println(pairs);

let back = xs.rev().take(2).to_list();
let uniq: Set<i32> = [1, 1, 2].collect();
println(back, uniq.len());
```

```output
[20, 40, 60]
21 6 true
91
4 2 6
4: 5
5: 6
["a1", "b2", "c3"]
[6, 5] 2
```

```nabi
let words = ["kiwi", "fig", "apple", "fig"];
println(words.sorted(), words.sorted_by_key(|w| w.len(), reverse: true));
println(words.counts().most_common(1), words.group_by(|w| w.len()));
println(words.enumerate(start: 1).map(|(i, w)| f"{i}.{w}").to_list().join(" "));
println([1, 2, 3].combinations(2).to_list(), [1, 2, 3].pairwise().to_list());
```

```output
["apple", "fig", "fig", "kiwi"] ["apple", "kiwi", "fig", "fig"]
[("fig", 2)] {4: ["kiwi"], 3: ["fig", "fig"], 5: ["apple"]}
1.kiwi 2.fig 3.apple 4.fig
[[1, 2], [1, 3], [2, 3]] [(1, 2), (2, 3)]
```

集合本身就能回應迭代器方法：`xs.sum()` 就是 `xs.iter().sum()`，一樣是惰性的（`List` 自己的方法，例如 `contains` 和 `windows`，優先）。`to_list()` 把項目收集成新的 `List`；`collect` 建立接收端的型別（上例中是 `Set<i32>`），沒有指定時則建立 `List`。`sorted`、`sorted_by_key`、`counts`（相當於 Python 的 `Counter`，用 `most_common` 排名）、`group_by` 和 `enumerate(start:)` 會回傳新的集合，或從你指定的數字開始計數；`combinations(k)`、`permutations(k)` 和 `pairwise()` 是惰性的，就像 Python 的 `itertools`。`min` 和 `max` 只需要 `PartialOrd`，所以也能用在浮點數上，而浮點數的 `sum` 會以固定順序成對相加。

## 撰寫自己的迭代器：`Iterator` 與 `yield`

`Iterator` trait 只要求關聯型別 `Item` 和一個方法 `next(ref self) -> Item | none`。所有配接器都會以預設方法的形式一併提供。

```nabi
struct Countdown { n: i32; }

impl Iterator for Countdown {
    type Item = i32;
    fn next(ref self) -> i32 | none {
        if self.n == 0 { return none; }
        self.n -= 1;
        self.n + 1
    }
}

let c: List<i32> = Countdown { n: 3 }.collect();
println(f"{c:?}");
for v in (Countdown { n: 5 }).filter(|v| v % 2 == 1) { println(f"{v}"); }
```

```output
[3, 2, 1]
5
3
1
```

手動管理狀態變得繁瑣時，請用**產生器**（generator）。`yield fn` 是一種函式，它的主體用 `yield value;` 一次送出一個值，呼叫它會回傳一個迭代器（`GenIter<T>`）。即使是無限的產生器也沒問題：用 `take` 只取需要的數量就好。在運算式的位置，也可以用 `yield { ... }` 區塊直接建立一個產生器。

```nabi
yield fn fib() -> i64 {
    let a = 0;
    let b = 1;
    loop {
        yield a;
        (a, b) = (b, a + b);
    }
}

let first = fib().take(10).to_list();
println(f"{first:?}");

let squares = yield { for i in 1..5 { yield i * i; } };
for v in squares { print(f"{v} "); }
println("");
```

```output
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
1 4 9 16
```

產生器可以在 yield 的同時持有資源。當 `for` 迴圈走訪的是它自己持有的產生器（例如 `for r in rows()` 這樣的呼叫），並且因為 `break`、`return` 或錯誤而提前離開時，產生器會當場結束：它的 `using` 區塊（[第 10 章](10-errors.html#using清理資源)）和 `finally` 區塊會在迴圈的下一個陳述式之前執行。存在變數裡的產生器則保持開啟，之後的迴圈可以接著使用它。

```nabi
struct Conn { name: str; }

impl Drop for Conn {
    fn drop(ref self) { println(f"close {self.name}"); }
}

yield fn rows() -> i32 {
    using c = Conn { name: "db" };
    for i in 1..=5 { yield i; }
}

for r in rows() {
    if r == 3 { break; }
    println(f"row {r}");
}
println("done");
```

```output
row 1
row 2
close db
done
```

## 進階：廣播運算子

第一次閱讀時可以跳過本節。`.+`、`.*` 和 `.>` 這類帶點的運算子是逐元素運算。它們會去語法糖（desugar）成 `DotAdd`、`DotMul` 和 `DotGt` 等 trait，所以只要為你的型別實作這些 trait，就能立刻使用這些運算子。

```nabi
struct Vector { xs: List<f64>; }

impl ops::DotAdd for Vector {
    type Output = Vector;
    fn dot_add(self, o: Vector) -> Vector {
        Vector { xs: self.xs.zip(o.xs.iter()).map(|(a, b)| a + b).to_list() }
    }
}

impl ops::DotMul<f64> for Vector {
    type Output = Vector;
    fn dot_mul(self, k: f64) -> Vector {
        Vector { xs: self.xs.map(|x| x * k).to_list() }
    }
}

let a = Vector { xs: [1.0, 2.0, 3.0] };
let b = Vector { xs: [10.0, 20.0, 30.0] };
let c = a .+ b .* 2.0;
println(f"{c.xs:?}");
```

```output
[21.0, 42.0, 63.0]
```

規格把這些運算子定義為 `List`、固定長度陣列和 `Array` 的標準運算，另外還有 `f.(xs)`：在函式後面加一個點，就會把函式套用到每個元素上。一串這樣的運算會以單一迴圈執行，而 `ys .= expr`、`ys .+= xs` 和其他 `.op=` 形式會直接寫入既有的串列，不會產生新串列（[第 21 章](21-arrays-and-numerics.html)）。

```nabi
let a = [1.0, 2.0, 3.0];
let b = [10.0, 20.0, 30.0];
let c = a .+ b;              // [11.0, 22.0, 33.0]
let d = a .* 2.0;            // [2.0, 4.0, 6.0]
let r = f64::sqrt.(a);       // [1.0, 1.414..., 1.732...]
println(f"{c:?} {d:?} {r:?}");
```

```output
[11.0, 22.0, 33.0] [2.0, 4.0, 6.0] [1.0, 1.4142135623730951, 1.7320508075688772]
```

## 小結

- `List` 提供 `push` / `pop` / `insert` / `remove` / `sort` / `get`，`Dict` 提供 `d[k]` / `get` / `d[k, default: v]`，`Set` 提供集合運算（`&`、`|`、`-`、`^`）。
- 集合型別是參考控制代碼。需要副本時呼叫 `clone()`。
- `xs[a:b:s]` 切片的行為和 Python 一樣，邊界超出範圍時不會觸發 panic（步長為 0 才會）。結果是 `Slice<T>` 檢視；`SliceMut<T>` 則會寫入串列。
- `Dict` 和 `Set` 保留插入順序；需要雙端佇列時，用 `Deque`。
- 在集合上或 `iter()` 之後呼叫配接器，再用 `to_list` / `collect` / `sum` / `fold` 消耗。只要實作 `next`，你自己的型別就成為迭代器。
- `yield fn` 和 `yield { }` 讓迭代器容易撰寫。

## 延伸閱讀

- [stdlib 2.9. Collections](../spec/refs-standard-libraries/200-core/209-collections.html)、[參照控制代碼](../spec/refs-standard-libraries/200-core/209-collections.html#參照控制代碼)
- [3.6. 索引與切片](../spec/refs-lang/003-expressions/306-indexing-and-slicing.html)
- [stdlib 2.7. iter](../spec/refs-standard-libraries/200-core/207-iter.html)、[4.11. 產生器](../spec/refs-lang/004-functions/411-generators.html)
- [3.3. 廣播運算子](../spec/refs-lang/003-expressions/303-broadcast-operators.html)、[stdlib 2.6. broadcast](../spec/refs-standard-libraries/200-core/206-broadcast.html)
- API：[`List`](../../std/core/collections/list.html#struct.List)、[`Dict`](../../std/core/collections/dict.html#struct.Dict)、[`Set`](../../std/core/collections/set.html#struct.Set)、[`Deque`](../../std/core/collections/deque.html#struct.Deque)、[`Slice`](../../std/core/collections/slice.html#struct.Slice)、[`Iterator`](../../std/core/iter.html#trait.Iterator)、[`GenIter`](../../std/core/iter.html#struct.GenIter)
- 下一章：[9. 字串與 f-string](09-strings.html)
