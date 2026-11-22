# 數獨求解器

在固定長度陣列的盤面上用回溯法解數獨，在每個空格依序嘗試每個候選數字。

## 程式碼

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

## 說明

- 盤面是固定長度陣列 `[[u8; 9]; 9]`。大小是型別的一部分，所以 `cells[r][c]` 直接讀寫，不需要另外配置（[值與型別](../learn/02-values-and-types.html)）。
- `solve` 透過 `ref self` 就地填寫盤面，並透過 `ref steps` 累加呼叫端的計數器。呼叫端也寫上 `ref`，即 `board.solve(ref steps)`，所以一看就知道值會改變（[4.7 `ref` 關鍵字](../spec/refs-lang/004-functions/407-ref-keyword.html)、[函式](../learn/03-functions.html)）。
- 回溯法的核心只有三行：放入一個數字（`self.cells[r][c] = d`）、遞迴嘗試解出其餘部分、失敗時再把格子清空。如果第一個空格試遍 1-9 都失敗，就回傳 `false`，由上一層嘗試別的數字。
- 像 `0..9` 這樣的範圍字面值，型別取決於迴圈變數的用法：`r` 和 `c` 用來索引盤面，並以 `usize` 引數傳遞，所以不加後綴也是 `usize`（[1.6 數字字面值](../spec/refs-lang/001-lexical/106-numeric-literals.html)）。
- 讀取題目字串時只取數字和 `.`，略過空白與換行，所以盤面可以排得容易閱讀（[字串](../learn/09-strings.html)）。
