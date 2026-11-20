# 2. Linear Algebra, Dense and Sparse

Most numerical work comes down to matrices: a system of equations, a least-squares fit, the modes of a structure. `linalg` handles dense matrices, `sparse` the large ones that are mostly zeros. Both work on core's `Matrix<f64>` and `Vector<f64>` ([chapter 21](../21-arrays-and-numerics.html)), so their results feed straight into everything else.

## Matrices and vectors

A `Matrix<f64>` is a two-dimensional array; write one as nested lists with the type given. With linalg's `ops` (on by default), `@` is the matrix product, `\` solves and `†` is the transpose (the conjugate transpose for complex matrices). The dot operators (`.+`, `.*`) work element by element, as in chapter 21.

```nabi
use core::array::{Matrix, Vector};
use linalg::ops::*;

let a: Matrix<f64> = [[2, 1], [1, 3]];
let v: Vector<f64> = [1, 2];
println(a @ v);
println(a† @ a);
println(a .* 10.0);
println(linalg::eye::<f64>(2));
```

```output
[4.0, 7.0]
[[5.0, 5.0], [5.0, 10.0]]
[[20.0, 10.0], [10.0, 30.0]]
[[1.0, 0.0], [0.0, 1.0]]
```

`a @ v` panics when the shapes do not fit, the way `.+` does; `linalg::matmul(a, b)` is the same product as a call that throws a `LinalgError` instead.

## Solve a system

Three unknowns, three equations: `a x = b`. `linalg::solve` factors `a` (LU with partial pivoting) and solves; `a \ b` is the same as an operator, returning a result to unwrap with `get`. The residual `a x - b` shows how close the answer is.

```nabi
use core::array::{Matrix, Vector};
use linalg::ops::*;

let a: Matrix<f64> = [[4, 1, 0], [1, 3, 1], [0, 1, 2]];
let b: Vector<f64> = [1, 2, 3];

let x = try linalg::solve(a, b);
println(f"x = {x:.4}");
println(f"residual {linalg::norm(a @ x .- b) < 1e-12}");

let same = try (a \ b).get();
println(same.all_close(x));
```

```output
x = [0.2222, 0.1111, 1.4444]
residual true
true
```

To solve with the same matrix many times, factor it once: `linalg::lu(a)` (any square matrix) or `linalg::cholesky(a)` (symmetric positive definite, twice as fast) return a factorization with `solve`, `det` and `inv`.

```nabi
use core::array::{Matrix, Vector};

let a: Matrix<f64> = [[4, 1, 0], [1, 3, 1], [0, 1, 2]];
let c = try linalg::cholesky(a);
let rhs: List<Vector<f64>> = [[1, 0, 0], [0, 1, 0]];
for b in rhs {
    println(f"{try c.solve(b):.4}");
}
println(f"det {(try linalg::lu(a)).det():.1}");
```

```output
[0.2778, -0.1111, 0.0556]
[-0.1111, 0.4444, -0.2222]
det 18.0
```

## Fit a line by least squares

More equations than unknowns: there is no exact solution, so least squares finds the one that makes the squared residuals smallest. For a line `y = c0 + c1 t`, each data point is a row `[1, t]`. `linalg::lstsq` solves it through a QR factorization, which keeps its accuracy where the normal equations `aᵀa x = aᵀy` lose digits.

```nabi
use core::array::{Matrix, Vector};

let t: Vector<f64> = [0, 1, 2, 3, 4];
let y: Vector<f64> = [1.1, 2.9, 5.2, 7.1, 8.8];
let a: Matrix<f64> = Matrix::from_fn([5, 2], |i: [usize; 2]| -> f64 { if i[1] == 0 { 1.0 } else { t[i[0]] } });

let c = try linalg::lstsq(a, y);
println(f"y = {c[0]:.3} + {c[1]:.3} t");
```

```output
y = 1.100 + 1.960 t
```

`stats::ols` does the same fit and adds standard errors and p-values for each coefficient ([Statistics](03-statistics.html#regression)).

## Eigenvalues and the SVD

`linalg::eigh` takes a symmetric matrix and returns its eigenvalues in rising order with orthonormal eigenvectors as columns. Here it finds the natural frequencies of three masses on springs: the eigenvalues of the stiffness matrix are the squared frequencies.

```nabi
use core::array::Matrix;

let k: Matrix<f64> = [[2, -1, 0], [-1, 2, -1], [0, -1, 2]];
let e = try linalg::eigh(k);
println(f"squared frequencies {e.values():.4}");
println(f"frequencies {e.values().map(|v: f64| -> f64 { v.sqrt() }):.4}");
```

```output
squared frequencies [0.5858, 2.0000, 3.4142]
frequencies [0.7654, 1.4142, 1.8478]
```

The singular value decomposition works on any matrix. Its singular values show the rank and how well conditioned a matrix is: the ratio of the largest to the smallest is the condition number, and `linalg::cond` computes it.

```nabi
use core::array::Matrix;

let m: Matrix<f64> = [[1, 2], [2, 4.001]];
let s = try linalg::svd(m);
println(f"singular values {s.s():.5}");
println(f"condition number {(try linalg::cond(m)):.0}");
```

```output
singular values [5.00080, 0.00020]
condition number 25008
```

A condition number of about 2.5 x 10<sup>4</sup> means a solve can lose more than four of the sixteen digits a `f64` carries: the two rows are almost the same equation.

## Sparse matrices

A matrix with a million rows does not fit in memory as a dense array, but most such matrices are almost all zeros: each unknown couples to a few neighbours. `sparse` stores only the nonzeros. Build a matrix from (row, column, value) triplets with `Coo`, then convert it to `Csr` for arithmetic and solving.

```nabi
let c = sparse::Coo::new([4, 4]);
for i in 0..4 {
    try c.push(i, i, 4.0);
    if i + 1 < 4 {
        try c.push(i, i + 1, -1.0);
        try c.push(i + 1, i, -1.0);
    }
}
let a = c.to_csr();
println(f"{a.nnz()} nonzeros in a {a.shape():?} matrix");
println(try a.matvec([1, 1, 1, 1]));
```

```output
10 nonzeros in a [4, 4] matrix
[3.0, 2.0, 2.0, 3.0]
```

## Solve a large sparse system

The heat equation on a square plate, discretized on a 100 x 100 grid, gives 10 000 unknowns and a matrix with five nonzeros per row. `sparse::diags` builds the one-dimensional difference matrix from its diagonals, and `kron` combines two of them into the two-dimensional one. `sparse::cg` solves it iteratively with conjugate gradients; `ic0` (an incomplete Cholesky factorization) cuts the number of iterations.

```nabi
use core::array::Vector;

let n = 100;
let t = try sparse::diags([[2.0], [-1.0], [-1.0]], [0, 1, -1], [n, n]);
let i = sparse::eye(n);
let a = sparse::kron(i, t) + sparse::kron(t, i);
let b = Vector::<f64>::ones([n * n]);

let plain = try sparse::cg(a, b);
let pre = try sparse::cg(a, b, pre: try sparse::ic0(a));
println(f"{a.nnz()} nonzeros; cg {plain.iterations} iterations, with ic0 {pre.iterations}");

let x = try (try sparse::cholesky(a)).solve(b);
println(f"direct and iterative agree: {linalg::norm(x .- pre.x) / linalg::norm(x) < 1e-6}");
println(f"hottest point {x.max() ?? 0.0:.2}");
```

```output
49600 nonzeros; cg 187 iterations, with ic0 79
direct and iterative agree: true
hottest point 751.34
```

The iterative solvers take a tolerance (`rtol:`), an iteration limit and any `Operator`, including a function that applies the matrix without storing it (`sparse::operator`). `sparse::cholesky` and `sparse::lu` are direct solvers that reorder the matrix to keep the factors sparse.

## What next

- [Statistics](03-statistics.html): distributions, tests and regression.
- The [`linalg`](https://packages.nabi-lang.org/packages/linalg) and [`sparse`](https://packages.nabi-lang.org/packages/sparse) API: QR, Schur, `expm`, matrix functions; GMRES, BiCGSTAB and preconditioners.
