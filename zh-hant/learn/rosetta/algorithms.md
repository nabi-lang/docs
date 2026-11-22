# 演算法

經典的小型演算法，以直白的方式寫出。標準函式庫已經提供其中大部分（`sort`、`binary_search`、`permutations`）；把它們寫出來，可以看出 Nabi 讀起來的樣子。

## 二分搜尋

每次將範圍減半，在已排序的串列中搜尋。結果是一個位置或 `none`，型別也如此表明。

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

## 快速排序

選一個基準值（pivot），把其餘元素分成較小與較大兩部分，再分別排序。這個版本會建立新的串列；它寫起來短，但並不快。實際上應該使用 `xs.sort()`。

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

## 河內塔

先把 `n - 1` 個圓盤移開，移動最大的那一個，再把 `n - 1` 個圓盤移回它上面。

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

## 排列與組合

`permutations(k)` 與 `combinations(k)` 是惰性迭代器，與 Python 的 `itertools` 相同。

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

## 八皇后

用回溯法在西洋棋盤上放置八個皇后，使它們互不攻擊：每一列放一個皇后，每個皇后都放在先前的皇后看不到的行上。

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

## 網格中的最短路徑

廣度優先搜尋會繞過牆（`#`），找出從 `S` 到 `E` 的最少步數。

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

## 矩陣乘法

對一般的串列使用三層巢狀迴圈。真正的數值計算請用 core 的 `Matrix<f64>` 與 `linalg` 套件的 `@`，它們快得多（[線性代數](https://docs.nabi-lang.org/packages/linalg/latest)）。

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
