# 矩陣運算

一個實作運算子與索引 trait 的小型矩陣型別，讓你可以寫 `a * b`、`a .* b` 和 `m[i, j]`。

## 程式碼

```nabi
// A dense row-major matrix of f64.
struct Matrix {
    rows: usize;
    cols: usize;
    data: List<f64>;
}

impl Matrix {
    fn zeros(rows: usize, cols: usize) -> Matrix {
        Matrix { rows: rows, cols: cols, data: List::filled(rows * cols, 0.0) }
    }

    fn from_rows(rows: List<List<f64>>) -> Matrix {
        let m = Matrix::zeros(rows.len(), rows[0].len());
        for i in 0..m.rows {
            for j in 0..m.cols { m[i, j] = rows[i][j]; }
        }
        m
    }

    fn identity(n: usize) -> Matrix {
        let m = Matrix::zeros(n, n);
        for i in 0..n { m[i, i] = 1.0; }
        m
    }

    fn transpose(self) -> Matrix {
        let t = Matrix::zeros(self.cols, self.rows);
        for i in 0..self.rows {
            for j in 0..self.cols { t[j, i] = self[i, j]; }
        }
        t
    }
}

// m[i, j] reads an element ...
impl index::Index<(usize, usize)> for Matrix {
    type Output = f64;
    fn index(self, (i, j): (usize, usize)) -> f64 {
        self.data[i * self.cols + j]
    }
}

// ... and m[i, j] = v writes one.
impl index::IndexSet<(usize, usize), f64> for Matrix {
    fn index_set(ref self, (i, j): (usize, usize), value: f64) {
        self.data[i * self.cols + j] = value;
    }
}

impl ops::Add for Matrix {
    type Output = Matrix;
    fn add(self, rhs: Matrix) -> Matrix {
        Matrix { rows: self.rows, cols: self.cols, data: self.data .+ rhs.data }
    }
}

impl ops::Mul for Matrix {
    type Output = Matrix;
    fn mul(self, rhs: Matrix) -> Matrix {
        let out = Matrix::zeros(self.rows, rhs.cols);
        for i in 0..self.rows {
            for j in 0..rhs.cols {
                let sum = 0.0;
                for k in 0..self.cols { sum += self[i, k] * rhs[k, j]; }
                out[i, j] = sum;
            }
        }
        out
    }
}

// Scalar scaling: Matrix * f64.
impl ops::Mul<f64> for Matrix {
    type Output = Matrix;
    fn mul(self, k: f64) -> Matrix {
        Matrix { rows: self.rows, cols: self.cols, data: self.data .* k }
    }
}

// Element-wise (Hadamard) product: a .* b.
impl ops::DotMul for Matrix {
    type Output = Matrix;
    fn dot_mul(self, rhs: Matrix) -> Matrix {
        Matrix { rows: self.rows, cols: self.cols, data: self.data .* rhs.data }
    }
}

impl Display for Matrix {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        for i in 0..self.rows {
            try f.write_str("[");
            for j in 0..self.cols {
                try f.write_str(f"{self[i, j]:6.1}");
            }
            try f.write_str(" ]\n");
        }
    }
}

let a = Matrix::from_rows([[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]]);
let b = Matrix::from_rows([[7.0, 8.0], [9.0, 10.0], [11.0, 12.0]]);

println(f"A =\n{a}");
println(f"A^T =\n{a.transpose()}");
println(f"A * B =\n{a * b}");
println(f"A + A =\n{a + a}");
println(f"A .* A =\n{a .* a}");
println(f"A * 0.5 =\n{a * 0.5}");

let i3 = Matrix::identity(3);
let same = a * i3;
println(f"A * I == A: {same.data == a.data}");
```

```output
A =
[   1.0   2.0   3.0 ]
[   4.0   5.0   6.0 ]

A^T =
[   1.0   4.0 ]
[   2.0   5.0 ]
[   3.0   6.0 ]

A * B =
[  58.0  64.0 ]
[ 139.0 154.0 ]

A + A =
[   2.0   4.0   6.0 ]
[   8.0  10.0  12.0 ]

A .* A =
[   1.0   4.0   9.0 ]
[  16.0  25.0  36.0 ]

A * 0.5 =
[   0.5   1.0   1.5 ]
[   2.0   2.5   3.0 ]

A * I == A: true
```

## 說明

- Nabi 中的每個運算子都是 trait 方法呼叫。有了 `impl ops::Add for Matrix`，`a + a` 會轉譯為 `ops::Add::add(a, a)`（[3.1 運算子分派](../spec/refs-lang/003-expressions/301-operator-dispatch.html)、[`Add`](../../std/core/ops.html#trait.Add)、[`Mul`](../../std/core/ops.html#trait.Mul)）。
- 同一個運算子可以針對不同的右側型別實作多次。`impl ops::Mul for Matrix` 是矩陣乘積，`impl ops::Mul<f64> for Matrix` 是純量乘法；編譯器依運算元的型別選擇 impl。
- `.*` 這類以點開頭的運算子，會轉譯為另一組用於逐元素運算的 trait（[`DotMul`](../../std/core/ops.html#trait.DotMul)），所以矩陣乘積 `*` 和 Hadamard 乘積 `.*` 可以並存。在內部，`self.data .+ rhs.data` 和 `self.data .* k` 是作用在 `List<f64>` 本身的相同運算子，各自只有一個迴圈，不會產生中間串列（[3.3 廣播運算子](../spec/refs-lang/003-expressions/303-broadcast-operators.html)）。
- 以逗號分隔的索引會組成元組，所以讀取 `m[i, j]` 經由 [`Index<(usize, usize)>`](../../std/core/index/index.html#trait.Index)，寫入 `m[i, j] = v` 則經由 [`IndexSet<(usize, usize), f64>`](../../std/core/index/index.html#trait.IndexSet)，它的第二個引數是寫入值的型別。寫入方法接受 `ref self`，以就地修改矩陣（[3.6 索引與切片](../spec/refs-lang/003-expressions/306-indexing-and-slicing.html)）。
- 有了 `impl Display for Matrix`，f-string 中的 `{a}` 會印出整個矩陣。每個元素以 `{x:6.1}` 指定寬度與精度（[結構、列舉與 trait](../learn/06-structs-enums-traits.html)）。
- 這個範例示範你自己的型別如何加入運算子。至於數值運算，核心函式庫已經有 `Array<T, N>`，這是支援切片、廣播與融合點運算式的 n 維陣列（[陣列與數值運算](../learn/21-arrays-and-numerics.html)）。
