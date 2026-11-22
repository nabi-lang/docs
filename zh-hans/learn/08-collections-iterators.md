# 8. 集合类型与迭代器

大多数程序都要收集、遍历和变换大量的值。本章介绍标准集合类型（`List`、`Dict`、`Set` 及其相关类型）、切片，以及把它们串联在一起的 `Iterator`。

## List

`[1, 2, 3]` 是动态数组 `List<T>`。索引从 0 开始，负索引从末尾倒数。越界的 `xs[i]` 会触发 panic，所以读取可能不存在的位置时用 `get`，它返回 `T | none`。

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

`List`、`Dict` 和 `Set` 是**引用句柄**。`let ys = xs;` 以及作为函数实参传递，都共享同一个集合；需要独立的副本时调用 `clone()`。

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

`xs[start:end:step]` 的含义与 Python 相同。边界会被截断到长度范围内，所以切片永远不会因为越界而 panic。对 `List` 切片得到的是 `Slice<T>`，即原缓冲区上的视图；需要独立的列表时用 `to_list()` 复制。

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

`Slice` 是只读的。填充或重排列表一部分的函数改为接收 `SliceMut<T>`，它是一个可写视图，类似 Rust 的 `&mut [T]`。在期望 `SliceMut` 的地方，`xs` 传入整个列表，`xs[a:b]` 传入那个区间；不会复制任何内容，写入直接作用于列表。

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

视图在其列表改变大小（`push`、`insert`、`clear` 等）后就失效：之后再使用它，编译器能看出时是错误，看不出时会触发 panic。改变大小之后重新创建视图。

## Dict 与 Set

`{"key": value}` 是哈希映射 `Dict<K, V>`，`{1, 2}` 是哈希集合 `Set<T>`。空的 `{}` 是一个块，所以用 `Dict::new()` / `Set::new()` 创建空集合。键不存在时 `d[k]` 会触发 panic；`d.get(k)` 返回 `V | none`。`k in d` 检查键是否存在，`x in s` 检查集合成员（相反的检查用 `!in`）。

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

对于不存在的键，`d[k, default: v]` 读出 `v`；作为 `=` 或 `+=` 的目标，或作为方法的接收者时，它会先存入 `v`。集合上的 `&`、`|`、`-` 和 `^` 分别是交集、并集、差集和对称差，`+` 把两个列表连接成一个新列表。

在 `ages.remove("lee")` 之后，如果第四行是 `println(f"{ages["lee"]}");`，就会触发 panic，消息会告诉你应该怎么写：

```text
thread 'main' panicked at main.nabi:4:12:
key "lee" not found; use `d.get(k)` to get `i32 | none`
```

> [!NOTE]
> `Dict` 和 `Set` 按插入顺序迭代。再次插入的键保留原来的位置，删除后再插入的键排到最后。哈希值本身每次运行都会变化，所以不要依赖它们。

## 插入顺序与 Deque

由于 `Dict` 和 `Set` 保留插入顺序，所以没有单独的有序类型：`iter()` 按插入的顺序给出 `(key, value)` 对。`Deque` 是一个两端都可以推入和弹出的队列。

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

## 有序映射与有序集合

当你需要的顺序是键本身的顺序，而不是插入顺序时，使用 `core::collections` 中的 `SortedDict` 和 `SortedSet`。它们让键保持有序（B 树），所以查找第一个和最后一个键、不大于某个值的最近键（`floor`）或大于它的最近键（`ceil`），以及某个区间内的所有键，开销都很小。

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

查找、插入和删除是 O(log n)，而 `Dict` 平均是 O(1)；需要顺序时才使用有序映射。`range` 接受任何区间：`a..b`、`a..=b`、`a..`，以及写在括号中的 `(..b)`。

## 迭代器与适配器

`for x in collection` 驱动的是一个 `Iterator`。在集合的 `iter()` 返回的迭代器上，可以链式调用 `map`、`filter`、`enumerate`、`zip`、`take`、`skip`、`rev`、`step_by` 和 `chain` 这样的**适配器**。适配器是惰性的：只有在调用真正消费这些值的方法，比如 `collect`、`sum`、`fold` 或 `count` 时，它们才会运行。

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

集合本身也能响应迭代器方法：`xs.sum()` 就是 `xs.iter().sum()`，同样是惰性的（`List` 自己的方法，比如 `contains` 和 `windows`，优先）。`to_list()` 把各项收集到一个新的 `List` 中；`collect` 构建接收方的类型（上面的 `Set<i32>`），没有任何说明时构建 `List`。`sorted`、`sorted_by_key`、`counts`（Python 的 `Counter`，用 `most_common` 排序）、`group_by` 和 `enumerate(start:)` 返回新的集合，或者从你指定的位置开始计数；`combinations(k)`、`permutations(k)` 和 `pairwise()` 是惰性的，和 Python 的 `itertools` 一样。`min` 和 `max` 只需要 `PartialOrd`，所以对浮点数也能用；浮点数的 `sum` 按固定顺序两两相加。

## 编写自己的迭代器：`Iterator` 与 `yield`

`Iterator` trait 只要求关联类型 `Item` 和一个方法 `next(ref self) -> Item | none`。所有适配器都作为默认方法随之而来。

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

手动管理状态变得繁琐时，使用**生成器**。`yield fn` 是这样一种函数：函数体用 `yield value;` 一次产出一个值，调用它会返回一个迭代器（`GenIter<T>`）。即使是无限生成器也没问题：用 `take` 只取需要的个数。在表达式位置，还可以用 `yield { ... }` 块直接创建一个。

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

生成器可以在产出值的同时持有资源。当遍历一个由循环自己持有的生成器的 `for` 循环（如 `for r in rows()` 这样的调用）因 `break`、`return` 或错误而提前退出时，生成器会当场结束：它的 `using` 块（[第 10 章](10-errors.html#using清理资源)）和 `finally` 块会在循环的下一条语句之前运行。保存在变量中的生成器保持打开，之后的循环可以接着使用它。

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

## 进阶：广播运算符

第一次阅读可以跳过本节。`.+`、`.*`、`.>` 这样带点的运算符是逐元素运算。它们会脱糖为 `DotAdd`、`DotMul`、`DotGt` 这样的 trait，所以为你的类型实现这些 trait，就能立即使用这些运算符。

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

规范为 `List`、定长数组和 `Array` 标准地定义了这些运算符，还有 `f.(xs)`：在函数后面加一个点，就把它应用到每个元素上。一连串这样的运算作为一个循环执行，`ys .= expr`、`ys .+= xs` 以及其他 `.op=` 形式直接写入现有列表，不会创建新列表（[第 21 章](21-arrays-and-numerics.html)）。

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

## 小结

- `List` 提供 `push` / `pop` / `insert` / `remove` / `sort` / `get`，`Dict` 提供 `d[k]` / `get` / `d[k, default: v]`，`Set` 提供集合运算（`&`、`|`、`-`、`^`）。
- 集合是引用句柄。需要副本时调用 `clone()`。
- `xs[a:b:s]` 切片与 Python 一样，越界的边界永远不会触发 panic（步长为 0 时会）。结果是 `Slice<T>` 视图；`SliceMut<T>` 可以写入列表。
- `Dict` 和 `Set` 保留插入顺序；双端队列使用 `Deque`。
- 在集合上或 `iter()` 之后调用适配器，再用 `to_list` / `collect` / `sum` / `fold` 消费。只要实现 `next`，你自己的类型就成了迭代器。
- `yield fn` 和 `yield { }` 让迭代器写起来很容易。

## 延伸阅读

- [标准库 2.9. 集合类型](../spec/refs-standard-libraries/200-core/209-collections.html)、[引用句柄](../spec/refs-standard-libraries/200-core/209-collections.html#引用句柄)
- [3.6. 索引与切片](../spec/refs-lang/003-expressions/306-indexing-and-slicing.html)
- [标准库 2.7. iter](../spec/refs-standard-libraries/200-core/207-iter.html)、[4.11. 生成器](../spec/refs-lang/004-functions/411-generators.html)
- [3.3. 广播运算符](../spec/refs-lang/003-expressions/303-broadcast-operators.html)、[标准库 2.6. broadcast](../spec/refs-standard-libraries/200-core/206-broadcast.html)
- API：[`List`](../../std/core/collections/list.html#struct.List)、[`Dict`](../../std/core/collections/dict.html#struct.Dict)、[`Set`](../../std/core/collections/set.html#struct.Set)、[`Deque`](../../std/core/collections/deque.html#struct.Deque)、[`Slice`](../../std/core/collections/slice.html#struct.Slice)、[`Iterator`](../../std/core/iter.html#trait.Iterator)、[`GenIter`](../../std/core/iter.html#struct.GenIter)
- 下一章：[9. 字符串与 f-string](09-strings.html)
