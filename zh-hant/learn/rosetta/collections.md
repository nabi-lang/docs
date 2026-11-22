# 集合型別

使用串列、字典與集合的題目。`List`、`Dict` 與 `Set` 都是共用的控制代碼：指派不會複製它，`clone()` 才會。

## 移除重複項目並保留順序

`Set` 會記住它看過的值；`insert` 回傳該值是否為新值。

```nabi
let xs = [3, 1, 3, 2, 1, 5, 2];
let seen: Set<i32> = Set::new();
let unique = xs.filter(|x| seen.insert(x)).to_list();
println(unique);

// When order does not matter, a set alone is enough.
let s: Set<i32> = xs.collect();
println(s.len());
```

```output
[3, 1, 2, 5]
4
```

## 依鍵排序

`sorted` 回傳一個新的已排序串列；`sort` 則就地排序。`sorted_by_key` 依函式的回傳值排序，並讓相等的項目保持原本的順序（這種排序是穩定的），所以先依第二個鍵排序、再依第一個鍵排序，就能得到依兩個鍵的排序。

```nabi
let people = [("ada", 36), ("grace", 85), ("alan", 41), ("linus", 36)];
println(people.sorted_by_key(|(_, age)| age));
println(people.sorted_by_key(|(name, _)| name.len(), reverse: true));

let words = ["pear", "Fig", "apple", "banana"];
println(words.sorted());
println(words.sorted_by_key(|w| w.to_lowercase()));
```

```output
[("ada", 36), ("linus", 36), ("alan", 41), ("grace", 85)]
[("grace", 85), ("linus", 36), ("alan", 41), ("ada", 36)]
["Fig", "apple", "banana", "pear"]
["apple", "banana", "Fig", "pear"]
```

## 分組與計數

`counts` 建立次數表（相當於 Python 的 `Counter`）；`group_by` 把項目放在函式為它們給出的鍵底下。

```nabi
let sales = [("north", 120), ("south", 80), ("north", 45), ("east", 60), ("south", 30)];

let total: Dict<str, i32> = Dict::new();
for (region, amount) in sales { total[region, default: 0] += amount; }
println(total);

println(sales.map(|(r, _)| r).counts());
println(["apple", "avocado", "banana", "blueberry", "cherry"].group_by(|w| w[0:1]));
```

```output
{"north": 165, "south": 110, "east": 60}
{"north": 2, "south": 2, "east": 1}
{"a": ["apple", "avocado"], "b": ["banana", "blueberry"], "c": ["cherry"]}
```

## 集合運算

`&`、`|`、`-` 與 `^` 分別是交集、聯集、差集與對稱差集。

```nabi
let rust: Set<str> = {"ada", "linus", "grace"};
let nabi: Set<str> = {"grace", "ada", "alan"};
println(f"both: {rust & nabi:?}");
println(f"either: {(rust | nabi).len()}");
println(f"only rust: {rust - nabi:?}");
println(f"one but not both: {rust ^ nabi:?}");
println(f"{"ada" in nabi} {rust.is_subset(rust | nabi)}");
```

```output
both: {"ada", "grace"}
either: 4
only rust: {"linus"}
one but not both: {"linus", "alan"}
true true
```

## zip、enumerate 與 flatten

```nabi
let names = ["x", "y", "z"];
let values = [1.5, -2.0, 0.25];
for (i, (n, v)) in names.zip(values).enumerate() {
    println(f"{i}: {n} = {v}");
}

let nested = [[1, 2], [3], [], [4, 5, 6]];
println(nested.flatten().to_list());
println(nested.map(|row| row.len()).to_list());
```

```output
0: x = 1.5
1: y = -2
2: z = 0.25
[1, 2, 3, 4, 5, 6]
[2, 1, 0, 3]
```

## 依某個量度找出最大者

`max_by_key` 與 `min_by_key` 回傳的是項目本身，而不是量度；對空串列則回傳 `none`。它們的鍵必須是 `Ord`；浮點數不是（NaN 沒有順序），所以要在 `reduce` 中比較浮點數鍵。

```nabi
let cities = [("Seoul", 9.4), ("Busan", 3.3), ("Incheon", 3.0), ("Daegu", 2.4)];
let (big, pop) = cities.reduce(|a, b| if b.1 > a.1 { b } else { a }) ?? ("", 0.0);
println(f"largest: {big} ({pop} million)");
println(f"{cities.min_by_key(|(name, _)| name.len()):?}");

let empty: List<i32> = [];
println(f"{empty.max():?}");
```

```output
largest: Seoul (9.4 million)
("Seoul", 9.4)
none
```

## 堆疊與佇列

`List` 就是堆疊（`push`、`pop`）；`Deque` 是兩端都很快的佇列。

```nabi
let stack = [1, 2, 3];
stack.push(4);
println(f"{stack.pop():?} {stack:?}");

let q: Deque<str> = Deque::new();
for job in ["build", "test", "deploy"] { q.push_back(job); }
while let job: str = q.pop_front() {
    println(f"running {job}, {q.len()} left");
}
```

```output
4 [1, 2, 3]
running build, 2 left
running test, 1 left
running deploy, 0 left
```

## 反轉字典

交換鍵與值。若有多個鍵共用同一個值，就把它們收集到串列中。

```nabi
let color: Dict<str, str> = {"apple": "red", "banana": "yellow", "cherry": "red"};
let by_color: Dict<str, List<str>> = Dict::new();
for (fruit, c) in color.iter() {
    by_color[c, default: []].push(fruit);
}
println(by_color);
```

```output
{"red": ["apple", "cherry"], "yellow": ["banana"]}
```
