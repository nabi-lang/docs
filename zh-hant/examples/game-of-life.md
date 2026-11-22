# 生命遊戲

在邊緣首尾相接的 10×8 環面格子上執行 Conway 的生命遊戲，並以文字印出幾個世代。

## 程式碼

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

## 說明

- 格子是固定長度的二維陣列 `[[bool; W]; H]`（10×8）。陣列的大小寫在型別中，不需要另外配置；它用重複形式 `[[false; W]; H]` 建立，並用 `==` / `!=` 整個比較（[值與型別](../learn/02-values-and-types.html)）。
- 計算鄰居時用 `(x + W - 1) % W` 而不是 `x - 1`。這樣可以避免 `usize` 減法小於零，並讓邊緣接到對面。
- 存活規則是對 `(alive, neighbors)` 元組做的單一個 `match`。模式可以用 `|` 組合，例如 `(true, 2) | (true, 3)`（[3.12 match](../spec/refs-lang/003-expressions/312-match-expression.html)、[控制流程](../learn/04-control-flow.html)）。
- `impl Display for Grid` 用 `Formatter::write_char` 一次寫一個字元，所以 `f"{grid}"` 會畫出格子（[`Formatter`](../../std/core/fmt.html#struct.Formatter)、[結構、列舉與 trait](../learn/06-structs-enums-traits.html)）。
- 滑翔機每 4 個世代沿對角線移動一格，形狀保持不變。在 10×8 的環面上，它需要移動 lcm(10, 8) = 40 次才會回到起點，也就是 160 個世代。
