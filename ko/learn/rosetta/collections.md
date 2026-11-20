# 컬렉션

리스트, 딕셔너리, 집합으로 하는 작업입니다. `List`, `Dict`, `Set`은 공유되는 핸들입니다. 대입하면 복사되지 않고, `clone()`을 해야 복사됩니다.

## 순서를 지키며 중복 없애기

`Set`은 본 값을 기억합니다. `insert`는 그 값이 새것이었는지를 돌려줍니다.

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

## 키로 정렬하기

`sorted`는 정렬한 새 리스트를 돌려주고, `sort`는 제자리에서 정렬합니다. `sorted_by_key`는 함수가 돌려주는 값으로 정렬하며, 같은 항목은 원래 순서를 지킵니다(안정 정렬). 그래서 두 번째 키로 정렬한 다음 첫 번째 키로 정렬하면 두 키로 정렬한 결과가 됩니다.

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

## 묶고 세기

`counts`는 빈도표를 만듭니다(Python의 `Counter`). `group_by`는 함수가 정해 준 키 아래에 항목을 모읍니다.

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

## 집합 연산

`&`, `|`, `-`, `^`는 교집합, 합집합, 차집합, 대칭 차집합입니다.

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

## zip, enumerate, 펼치기

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

## 어떤 기준으로 가장 큰 것 찾기

`max_by_key`와 `min_by_key`는 기준 값이 아니라 항목을 돌려줍니다. 빈 리스트에서는 `none`을 돌려줍니다. 기준은 `Ord`여야 하는데 부동소수점 수는 그렇지 않으므로(NaN에는 순서가 없습니다) 부동소수점 기준은 `reduce`에서 비교합니다.

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

## 스택과 큐

`List`는 스택(`push`, `pop`)이고, `Deque`는 양쪽 끝이 모두 빠른 큐입니다.

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

## 딕셔너리 뒤집기

키와 값을 맞바꿉니다. 여러 키가 같은 값을 가지면 리스트에 모읍니다.

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
