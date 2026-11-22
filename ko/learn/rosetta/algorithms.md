# 알고리즘

고전적인 작은 알고리즘을 꾸밈없이 씁니다. 대부분은 표준 라이브러리에 이미 있지만(`sort`, `binary_search`, `permutations`), 직접 써 보면 Nabi가 어떻게 읽히는지 보입니다.

## 이진 탐색

범위를 반씩 줄이며 정렬된 리스트를 찾습니다. 결과는 위치이거나 `none`이고, 타입이 그렇다고 말해 줍니다.

```nabi
fn search(xs: List<i32>, want: i32) -> usize | none {
    let (lo, hi) = (0usize, xs.len());
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        if xs[mid] == want { return mid; }
        if xs[mid] < want { lo = mid + 1; } else { hi = mid; }
    }
    none
}

let xs = [2, 3, 5, 7, 11, 13, 17, 19];
println(f"{search(xs, 13):?} {search(xs, 4):?}");
println(f"{xs.binary_search(13):?}");
```

```output
5 none
5
```

## 퀵정렬

피벗을 고르고, 나머지를 작은 것과 큰 것으로 나눈 뒤 둘 다 정렬합니다. 이 버전은 새 리스트를 만들기 때문에 짧을 뿐 빠르지는 않습니다. 실제로는 `xs.sort()`를 쓰세요.

```nabi
fn quicksort(xs: List<i32>) -> List<i32> {
    if xs.len() <= 1 { return xs.clone(); }
    let pivot = xs[0];
    let rest = xs[1:];
    quicksort(rest.filter(|x| x < pivot).to_list()) + [pivot] + quicksort(rest.filter(|x| x >= pivot).to_list())
}

println(quicksort([38, 27, 43, 3, 9, 82, 10, 3]));
```

```output
[3, 3, 9, 10, 27, 38, 43, 82]
```

## 하노이의 탑

원판 `n - 1`개를 비켜 놓고, 가장 큰 원판을 옮긴 다음, `n - 1`개를 그 위로 다시 옮깁니다.

```nabi
fn hanoi(n: u32, from: str, to: str, via: str, moves: List<str>) {
    if n == 0 { return; }
    hanoi(n - 1, from, via, to, moves);
    moves.push(f"{from}->{to}");
    hanoi(n - 1, via, to, from, moves);
}

let moves: List<str> = [];
hanoi(3, "A", "C", "B", moves);
println(f"{moves.len()} moves: {moves.join(" ")}");
```

```output
7 moves: A->C A->B C->B A->C B->A B->C A->C
```

## 순열과 조합

`permutations(k)`와 `combinations(k)`는 Python의 `itertools`처럼 지연 반복자입니다.

```nabi
println([1, 2, 3].permutations(3).to_list());
println(["a", "b", "c", "d"].combinations(2).map(|p| p.join("")).to_list());
println(f"{(1..=10).to_list().combinations(3).count()} ways to pick 3 of 10");
```

```output
[[1, 2, 3], [1, 3, 2], [2, 1, 3], [2, 3, 1], [3, 1, 2], [3, 2, 1]]
["ab", "ac", "ad", "bc", "bd", "cd"]
120 ways to pick 3 of 10
```

## 여덟 퀸

서로 공격하지 않도록 체스판에 퀸 여덟 개를 백트래킹으로 놓습니다. 줄마다 퀸 하나를, 앞서 놓은 퀸이 보지 못하는 열에 놓습니다.

```nabi
fn solve(n: usize, row: usize, cols: List<usize>, out: List<List<usize>>) {
    if row == n {
        out.push(cols.clone());
        return;
    }
    for c in 0..n {
        let safe = cols.enumerate().all(|(r, q)| q != c && (q as i64 - c as i64).abs() != (row - r) as i64);
        if safe {
            cols.push(c);
            solve(n, row + 1, cols, out);
            cols.pop();
        }
    }
}

let all: List<List<usize>> = [];
solve(8, 0, [], all);
println(f"{all.len()} solutions; the first:");
for q in all[0] {
    println(".".repeat(q) + "Q" + ".".repeat(7 - q));
}
```

```output
92 solutions; the first:
Q.......
....Q...
.......Q
.....Q..
..Q.....
......Q.
.Q......
...Q....
```

## 격자에서 최단 경로

너비 우선 탐색이 벽(`#`)을 돌아 `S`에서 `E`까지 가장 적은 걸음 수를 찾습니다.

```nabi
let maze = [
    "S..#....",
    ".#.#.##.",
    ".#...#..",
    ".####.#.",
    "......#E",
];
let (h, w) = (maze.len() as i64, maze[0].len() as i64);
let at = |r: i64, c: i64| -> str { maze[r as usize][c as usize:c as usize + 1] };

let dist: Dict<(i64, i64), i64> = {(0, 0): 0};
let q: Deque<(i64, i64)> = Deque::new();
q.push_back((0, 0));
while let (r, c): (i64, i64) = q.pop_front() {
    for (dr, dc) in [(1, 0), (-1, 0), (0, 1), (0, -1)] {
        let (nr, nc) = (r + dr, c + dc);
        if nr < 0 || nc < 0 || nr >= h || nc >= w || at(nr, nc) == "#" || (nr, nc) in dist { continue; }
        dist[(nr, nc)] = dist[(r, c)] + 1;
        q.push_back((nr, nc));
    }
}
println(f"shortest path: {dist.get((h - 1, w - 1)) ?? -1} steps");
```

```output
shortest path: 15 steps
```

## 행렬 곱셈

평범한 리스트 위의 세 겹 반복문입니다. 실제 수치 계산에는 core의 `Matrix<f64>`와 `linalg` 패키지의 `@`가 훨씬 빠릅니다([선형대수](https://docs.nabi-lang.org/packages/linalg/latest)).

```nabi
fn matmul(a: List<List<f64>>, b: List<List<f64>>) -> List<List<f64>> {
    let (n, m, p) = (a.len(), b.len(), b[0].len());
    let c: List<List<f64>> = [];
    for i in 0..n {
        let row = List::filled(p, 0.0);
        for k in 0..m {
            for j in 0..p { row[j] += a[i][k] * b[k][j]; }
        }
        c.push(row);
    }
    c
}

println(matmul([[1.0, 2.0], [3.0, 4.0]], [[5.0, 6.0], [7.0, 8.0]]));
```

```output
[[19.0, 22.0], [43.0, 50.0]]
```
