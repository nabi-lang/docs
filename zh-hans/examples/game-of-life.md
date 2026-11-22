# 生命游戏

在边缘相互连通的 10×8 环面网格上运行 Conway 的生命游戏，并以文本输出几代。

## 代码

```nabi
const W: usize = 10;
const H: usize = 8;

// A grid of cells on a torus: the edges wrap around.
struct Grid {
    cells: [[bool; W]; H];
}

impl Grid {
    fn parse(rows: List<str>) -> Grid {
        let cells = [[false; W]; H];
        for (y, row) in rows.enumerate() {
            for (x, c) in row.chars().enumerate() {
                cells[y][x] = c == '#';
            }
        }
        Grid { cells: cells }
    }

    fn alive(self, x: usize, y: usize) -> bool { self.cells[y][x] }

    fn neighbors(self, x: usize, y: usize) -> usize {
        let n = 0;
        for dy in [H - 1, 0, 1] {
            for dx in [W - 1, 0, 1] {
                if dx == 0 && dy == 0 { continue; }
                // Adding W - 1 and taking % W steps one cell left with wraparound.
                if self.alive((x + dx) % W, (y + dy) % H) { n += 1; }
            }
        }
        n
    }

    fn step(self) -> Grid {
        let next = [[false; W]; H];
        for y in 0..H {
            for x in 0..W {
                next[y][x] = match (self.alive(x, y), self.neighbors(x, y)) {
                    (true, 2) | (true, 3) => true,
                    (false, 3) => true,
                    _ => false,
                };
            }
        }
        Grid { cells: next }
    }

    fn population(self) -> usize {
        let n = 0;
        for row in self.cells {
            for c in row { if c { n += 1; } }
        }
        n
    }
}

impl Display for Grid {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        for row in self.cells {
            for c in row { try f.write_char(if c { '#' } else { '.' }); }
            try f.write_char('\n');
        }
    }
}

// A glider in the corner and a blinker on the right.
let grid = Grid::parse([
    ".#........",
    "..#.......",
    "###.......",
    "..........",
    "......###.",
    "..........",
    "..........",
    "..........",
]);

for step in 0..4 {
    println(f"generation {step} (population {grid.population()})");
    println(f"{grid}");
    grid = grid.step();
}

// A lone glider returns to its shape every 4 steps, shifted by (1, 1);
// on a 10x8 torus it needs lcm(10, 8) = 40 such moves to come home.
let glider = Grid::parse([".#", "..#", "###"]);
let g = glider.step();
let period = 1;
while g.cells != glider.cells {
    g = g.step();
    period += 1;
}
println(f"a lone glider comes back after {period} generations");
```

```output
generation 0 (population 8)
.#........
..#.......
###.......
..........
......###.
..........
..........
..........

generation 1 (population 8)
..........
#.#.......
.##.......
.#.....#..
.......#..
.......#..
..........
..........

generation 2 (population 8)
..........
..#.......
#.#.......
.##.......
......###.
..........
..........
..........

generation 3 (population 8)
..........
.#........
..##......
.##....#..
.......#..
.......#..
..........
..........

a lone glider comes back after 160 generations
```

## 讲解

- 网格是定长二维数组 `[[bool; W]; H]`（10×8）。数组的大小写在类型中，不需要单独分配；它用重复形式 `[[false; W]; H]` 构造，并且可以用 `==` / `!=` 整体比较（[值与类型](../learn/02-values-and-types.html)）。
- 统计邻居时用 `(x + W - 1) % W` 代替 `x - 1`。这样既避免了 `usize` 减法变为负数，又让边缘连到对面。
- 存活规则是对 `(alive, neighbors)` 元组的单个 `match`。模式可以用 `|` 组合，例如 `(true, 2) | (true, 3)`（[3.12 match](../spec/refs-lang/003-expressions/312-match-expression.html)，[控制流](../learn/04-control-flow.html)）。
- `impl Display for Grid` 用 `Formatter::write_char` 逐个字符地写出，所以 `f"{grid}"` 会画出网格（[`Formatter`](../../std/core/fmt.html#struct.Formatter)，[结构体、枚举与 trait](../learn/06-structs-enums-traits.html)）。
- 滑翔机每 4 代沿对角线移动一格，并保持形状。在 10×8 的环面上，它需要 lcm(10, 8) = 40 次移动才能回到起点，也就是 160 代。
