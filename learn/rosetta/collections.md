# Collections

Tasks with lists, dictionaries and sets. `List`, `Dict` and `Set` are shared handles: assigning one does not copy it, `clone()` does.

## Remove duplicates, keeping the order

A `Set` remembers what it has seen; `insert` returns whether the value was new.

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

## Sort by a key

`sorted` returns a new sorted list; `sort` sorts in place. `sorted_by_key` sorts by what a function returns, and keeps equal items in their order (the sort is stable), so sorting by a second key and then a first gives a sort by both.

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

## Group and count

`counts` builds a frequency table (Python's `Counter`); `group_by` puts items under the key a function gives them.

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

## Set operations

`&`, `|`, `-` and `^` are intersection, union, difference and symmetric difference.

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

## Zip, enumerate and flatten

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

## Find the largest by some measure

`max_by_key` and `min_by_key` return the item, not the measure; on an empty list they return `none`. Their key must be `Ord`; floats are not (NaN has no order), so compare float keys in `reduce`.

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

## A stack and a queue

A `List` is a stack (`push`, `pop`); a `Deque` is a queue that is fast at both ends.

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

## Invert a dictionary

Swap keys and values. When several keys share a value, collect them in a list.

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
