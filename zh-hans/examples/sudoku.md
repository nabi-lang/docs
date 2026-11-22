# 数独求解器

在定长数组表示的棋盘上用回溯法求解数独，在每个空格中逐个尝试候选数字。

## 代码

```nabi
// A 9x9 board; 0 marks an empty cell.
struct Board {
    cells: [[u8; 9]; 9];
}

impl Board {
    fn parse(text: str) -> Board {
        let cells = [[0u8; 9]; 9];
        let i = 0;
        for c in text.chars() {
            if c == '.' || c.is_ascii_digit() {
                cells[i / 9][i % 9] = if c == '.' { 0 } else { (c as u32 - '0' as u32) as u8 };
                i += 1;
            }
        }
        Board { cells: cells }
    }

    // Can digit `d` go at (r, c) without clashing in its row, column or box?
    fn allowed(self, r: usize, c: usize, d: u8) -> bool {
        for k in 0..9 {
            if self.cells[r][k] == d || self.cells[k][c] == d { return false; }
        }
        let (br, bc) = (r / 3 * 3, c / 3 * 3);
        for i in br..br + 3 {
            for j in bc..bc + 3 {
                if self.cells[i][j] == d { return false; }
            }
        }
        true
    }

    // Depth-first search; returns true once every cell is filled.
    fn solve(ref self, ref steps: u64) -> bool {
        for r in 0..9 {
            for c in 0..9 {
                if self.cells[r][c] != 0 { continue; }
                for d in 1..=9 {
                    if self.allowed(r, c, d) {
                        self.cells[r][c] = d;
                        steps += 1;
                        if self.solve(ref steps) { return true; }
                        self.cells[r][c] = 0;   // backtrack
                    }
                }
                return false;
            }
        }
        true
    }
}

impl Display for Board {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        for r in 0..9 {
            if r > 0 && r % 3 == 0 { try f.write_str("------+-------+------\n"); }
            for c in 0..9 {
                if c > 0 && c % 3 == 0 { try f.write_str("| "); }
                let v = self.cells[r][c];
                try f.write_str(if v == 0 { ". " } else { f"{v} " });
            }
            try f.write_str("\n");
        }
    }
}

let board = Board::parse("
    53..7.... 6..195... .98....6.
    8...6...3 4..8.3..1 7...2...6
    .6....28. ...419..5 ....8..79
");
println(f"puzzle:\n{board}");

let steps = 0;
if board.solve(ref steps) {
    println(f"solved after {steps} placements:\n{board}");
} else {
    println("no solution");
}
```

```output
puzzle:
5 3 . | . 7 . | . . . 
6 . . | 1 9 5 | . . . 
. 9 8 | . . . | . 6 . 
------+-------+------
8 . . | . 6 . | . . 3 
4 . . | 8 . 3 | . . 1 
7 . . | . 2 . | . . 6 
------+-------+------
. 6 . | . . . | 2 8 . 
. . . | 4 1 9 | . . 5 
. . . | . 8 . | . 7 9 

solved after 4208 placements:
5 3 4 | 6 7 8 | 9 1 2 
6 7 2 | 1 9 5 | 3 4 8 
1 9 8 | 3 4 2 | 5 6 7 
------+-------+------
8 5 9 | 7 6 1 | 4 2 3 
4 2 6 | 8 5 3 | 7 9 1 
7 1 3 | 9 2 4 | 8 5 6 
------+-------+------
9 6 1 | 5 3 7 | 2 8 4 
2 8 7 | 4 1 9 | 6 3 5 
3 4 5 | 2 8 6 | 1 7 9
```

## 讲解

- 棋盘是定长数组 `[[u8; 9]; 9]`。大小是类型的一部分，所以 `cells[r][c]` 直接读写，不需要单独分配（[值与类型](../learn/02-values-and-types.html)）。
- `solve` 通过 `ref self` 就地修改棋盘，并通过 `ref steps` 增加调用方的计数器。调用方也要写 `ref`，即 `board.solve(ref steps)`，所以可以看出这个值会被修改（[4.7 `ref` 关键字](../spec/refs-lang/004-functions/407-ref-keyword.html)，[函数](../learn/03-functions.html)）。
- 回溯的核心是三行：放下一个数字（`self.cells[r][c] = d`），递归地尝试求解其余部分，失败时再把这一格清空。如果第一个空格中 1 到 9 都失败了，就返回 `false`，由上一层尝试另一个数字。
- `0..9` 这样的字面量区间从循环变量的用法得到类型：`r` 和 `c` 用来索引棋盘，并作为 `usize` 实参传入，所以不需要后缀，它们就是 `usize`（[1.6 数字字面量](../spec/refs-lang/001-lexical/106-numeric-literals.html)）。
- 读取谜题字符串时只取数字和 `.`，跳过空格和换行，所以棋盘可以排得便于阅读（[字符串](../learn/09-strings.html)）。
