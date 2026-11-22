# 集合

使用列表、字典和集合的任务。`List`、`Dict` 和 `Set` 都是共享的句柄：赋值不会复制它们，`clone()` 才会。

## 去除重复项并保持顺序

`Set` 记住已经见过的值；`insert` 返回该值是否是新的。

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

## 按键排序

`sorted` 返回一个新的有序列表；`sort` 原地排序。`sorted_by_key` 按函数的返回值排序，并保持相等元素的原有顺序（排序是稳定的），所以先按第二个键排序、再按第一个键排序，就得到按两个键的排序。

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

## 分组与计数

`counts` 构建频率表（相当于 Python 的 `Counter`）；`group_by` 把元素放到函数给出的键下面。

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

## 集合运算

`&`、`|`、`-` 和 `^` 分别是交集、并集、差集和对称差。

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

## zip、enumerate 与 flatten

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

## 按某个度量找出最大的元素

`max_by_key` 和 `min_by_key` 返回元素本身，而不是度量值；列表为空时返回 `none`。它们的键必须是 `Ord`；浮点数不是（NaN 没有顺序），所以浮点数键要在 `reduce` 中比较。

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

## 栈与队列

`List` 就是栈（`push`、`pop`）；`Deque` 是两端都很快的队列。

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

## 反转字典

交换键和值。多个键对应同一个值时，把它们收集到一个列表中。

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
