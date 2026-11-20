# 22. Science with Nabi

Nabi's numeric core is in the standard library: `Array<T, N>`, the dot operators, reductions, `Complex` and random numbers ([chapter 21](21-arrays-and-numerics.html)). Everything a scientist builds on top of that, linear algebra, statistics, signals, data frames, plots, lives in packages: one package per field, each versioned on its own, all written in Nabi with no C or Fortran underneath, so they build for every target Nabi has, WebAssembly included.

This chapter maps the packages, shows the one rule they share for input data, works one small problem from a CSV file to a plot, and ends with a table for readers coming from NumPy, SciPy and pandas.

## The packages

| Package | What it does | Python counterpart |
|---------|--------------|--------------------|
| `linalg` | dense matrices: products, LU, Cholesky, QR, SVD, eigen, solvers, `expm` | `numpy.linalg`, `scipy.linalg` |
| `sparse` | CSR / CSC / COO matrices, CG, GMRES, BiCGSTAB, sparse Cholesky and LU | `scipy.sparse`, `scipy.sparse.linalg` |
| `spatial` | small fixed-size vectors, quaternions, matrices, transforms and shapes for games and graphics; `#[repr(C)]`, SIMD `Mat4` | `pyglm`, `scipy.spatial.transform` |
| `autodiff` | derivatives of your own functions: dual numbers, a reverse-mode tape, Hessians, Taylor series | `jax.grad`, `autograd` |
| `stats` | descriptive statistics, 19 distributions, tests, regression | `scipy.stats`, `statsmodels` (basics) |
| `signal` | FFT of any length, windows, FIR / IIR filters, spectra, peaks | `numpy.fft`, `scipy.signal` |
| `interpolate` | splines, polynomials, grids, scattered data, quadrature | `scipy.interpolate`, `scipy.integrate.quad` |
| `optimize` | minimization, least squares, root finding, linear programming | `scipy.optimize` |
| `ode` | initial and boundary value problems, stiff solvers, symplectic integrators | `scipy.integrate.solve_ivp` |
| `special` | gamma, Bessel, erf, zeta, elliptic functions; complex elementary functions | `scipy.special` |
| `table` | columnar data frames, CSV, JSONL, Arrow and Parquet files, lazy queries | `pandas`, `polars`, `pyarrow` |
| `units` | quantities with units, checked when you compile | `pint` |
| `plot` | SVG, PNG and terminal plots, notebook display | `matplotlib` |

Add the ones you use to `nabi.nabion` with `nabi add` (or, for a single-file script, `nabi add <package> file.nabi`):

```sh
nabi add linalg
nabi add table
```

Each package has its API documentation on [packages.nabi-lang.org](https://packages.nabi-lang.org). The code on this page needs those packages, so unlike the rest of this guide it is not run when the site is built; every block was checked with the packages' current versions.

## One way in: array-like input

Every package takes a sequence of numbers the same way. A function that wants a vector of `f64` takes anything `linalg::AsVector<f64>` covers: a `Vector`, a `List`, a fixed array, of `f64`, `f32` or integers, and an integer range. A `Vector<f64>` or a `List<f64>` is read in place; anything else is converted once.

```nabi norun
use core::array::Vector;

let xs: List<f64> = [0.0, 1.0, 2.0, 3.0];
let ys: List<i64> = [0, 1, 8, 27];
let s = try interpolate::cubic_spline(xs, ys);       // lists of different types
let same = try interpolate::cubic_spline(0..4, [0f32, 1f32, 8f32, 27f32]);
assert(s(1.5) == same(1.5));

let r = try ode::solve_ivp(|t: f64, y: Vector<f64>| -> Vector<f64> { -y }, 0.0..1.0, [1, 2]);
println(stats::mean(1..5));                            // 2.5
```

`stats` also reads arrays of any rank (in row-major order), and `plot` the same forms plus nested lists for grids. Your own functions can take the same trait:

```nabi norun
use linalg::AsVector;
use core::array::Vector;

fn rms<X: AsVector<f64>>(x: X) -> f64 {
    let v: Vector<f64> = x.as_vector();
    (v .* v).mean().sqrt()
}

println(rms([3, 4]));        // 3.5355339059327378
```

Write the element type of a mixed literal first: `[0.0, 0.5, 1]` is a list of floats, while `[0, 0.5, 1]` starts as a list of integers and the `0.5` is an error.

## A worked example

A cup of tea cools from 90 degrees. The readings are in a CSV file; Newton's law of cooling says the temperature falls toward the room's at a rate proportional to the difference, `T' = k (room - T)`. We fit `room` and `k` to the data, solve the law as an ODE, check the fit and plot it.

```nabi norun
use core::array::{Vector, linspace};

let csv = "minute,temp\n0,90.0\n5,71.2\n10,57.9\n15,48.6\n20,41.9\n30,33.6\n40,29.1\n";
let t = try table::read_csv_str(csv);                 // or table::read_csv("tea.csv")
let minute = try t.column("minute").to_f64();
let temp = try t.column("temp").to_f64();

// The solution of the law: T(t) = room + (T0 - room) e^(-k t).
let model = |x: f64, p: Vector<f64>| -> f64 { p[0] + (p[1] - p[0]) * (-p[2] * x).exp() };
let fit = try optimize::curve_fit(model, minute, temp, [20.0, 90.0, 0.1]);
let (room, k) = (fit.params[0], fit.params[2]);
println(f"room {room:.1} C, k {k:.3} per minute");    // room 24.9 C, k 0.067 per minute

// The law itself, integrated from the first reading.
let law = |_t: f64, y: Vector<f64>| -> Vector<f64> { (room .- y) .* k };
let r = try ode::solve_ivp(law, 0.0..40.0, [90.0], rtol: 1e-8);
println(f"after 40 minutes: {r.last()[0]:.1} C");      // after 40 minutes: 29.3 C

// The residuals: three parameters were fitted, so ddof is 3.
let p = fit.params;
let resid = temp .- minute.map(|x: f64| -> f64 { model(x, p) });
println(f"residual std {stats::std(resid, ddof: 3):.2} C");  // residual std 0.15 C

// Readings as points, the ODE's dense solution as a line.
let ts: Vector<f64> = linspace(0.0, 40.0, 81);
let ax = plot::axes()
    .scatter(minute, temp).label("measured")
    .line(ts, (r.sol)(ts)[0, :]).label("Newton's law")
    .xlabel("minute").ylabel("temperature [C]");
try ax.save("tea.svg");
```

Each step is one call into one package, and the values between them are plain `Vector<f64>`s: no conversion layer, no copies beyond the CSV parse.

## Linear algebra, dense and sparse

`linalg` works on `Matrix<T>` for `f64`, `f32` and their complex forms. Factorizations are types you build once and solve with many times. With the `ops` feature (on by default) `@` is the matrix product, `\` solves and `†` is the adjoint:

```nabi norun
use core::array::{Matrix, Vector};

let a: Matrix<f64> = [[4, 1, 0], [1, 3, 1], [0, 1, 2]];
let b: Vector<f64> = [1, 2, 3];
let x = try linalg::solve(a, b);
let residual = a @ x .- b |> linalg::norm;           // about 1e-16
let e = try linalg::eigh(a);                         // eigenvalues rising
let c = try linalg::cholesky(a);
let y = try c.solve(b);
```

Large systems with few nonzeros go to `sparse`. Build a matrix from triplets or diagonals, then solve directly or iteratively; the iterative solvers take a preconditioner and any `Operator`, including a matrix-free one:

```nabi norun
use core::array::Vector;

// The 2-d Laplacian on a 100 x 100 grid: 10 000 unknowns.
let t = try sparse::diags([[2.0], [-1.0], [-1.0]], [0, 1, -1], [100, 100]);
let i = sparse::eye(100);
let a = sparse::kron(i, t) + sparse::kron(t, i);
let b = Vector::<f64>::ones([10000]);

let s = try sparse::cg(a, b, pre: try sparse::ic0(a));     // conjugate gradients
let f = try sparse::cholesky(a);                           // or a direct solve
let x = try f.solve(b);
println(f"{s.iterations} iterations; difference {linalg::norm(s.x .- x):e}");
```

## Derivatives

`autodiff` differentiates functions you write over its number types. Write the function once, generic over `Smooth`, and pick the mode by the type you call it with: `Dual` (forward mode, one pass per input) for few inputs, `Var` (reverse mode, one backward sweep for the whole gradient) for many, `HyperDual` for exact second derivatives, `Jet` for Taylor coefficients of any order.

```nabi norun
use autodiff::{Dual, HyperDual, Smooth, Var};
use core::array::Vector;

fn rosen<T: Smooth>(x: Vector<T>) -> T {
    let a = x[1].clone() - x[0].clone() * x[0].clone();
    let b = T::of(1.0) - x[0].clone();
    a.clone() * a * 100.0 + b.clone() * b
}

let g = autodiff::grad(rosen::<Var>, [-1.2, 1.0]);       // reverse mode
let h = autodiff::hessian(rosen::<HyperDual>, [-1.2, 1.0]);
let d = autodiff::derivative(|x: Dual| -> Dual { x.sin() * x }, 1.0);
let best = try optimize::lbfgs(rosen::<Var>, [-1.2, 1.0]);  // optimize takes AD functions too
```

The notation `∂` and `∇` (from `autodiff::ops`, also re-exported by `interpolate::ops` and `optimize::ops`) follows one rule across the packages: the derivative comes back in the form of what you gave, never as a number. `∂(f)` of a function is a function, so its value at a point is `∂(f)(x)`; `∂(s)` of a spline is a spline; `∇(y)` of sampled values is sampled values.

```nabi norun
use autodiff::{Dual, Var};
use autodiff::ops::{∂, ∇};
use core::array::Vector;

let cos = ∂(|x: Dual| -> Dual { x.sin() });
println(cos(0.0));                                           // 1
let g = ∇(|x: Vector<Var>| -> Var { x[0] * x[1] });
println(g([2.0, 5.0]));                                      // [5.0, 2.0]
let s = try interpolate::cubic_spline(0..4, [0, 1, 8, 27]);
println(f"{∂(s)(2.0):.6}");                                // 12.000000, the spline's slope
let y: Vector<f64> = [0, 1, 4, 9];
println(∇(y));                                               // [1.0, 2.0, 4.0, 5.0]
```

## Signals, curves and equations

The rest follows the same shape: plain functions on arrays, options as named arguments, errors that say what to change.

```nabi norun
use core::array::{Array, Vector};
use signal::filter::{Band, butter, sosfiltfilt};

// A 50 Hz tone with a 300 Hz one on top, sampled at 1 kHz.
let fs = 1khz;
let x = Array::from_fn([2000], |i: [usize; 1]| -> f64 {
    let t = (i[0] as f64) / fs;
    (2.0 * f64::PI * 50hz * t).sin() + 0.3 * (2.0 * f64::PI * 300hz * t).sin()
});
let spectrum = try signal::welch(x, fs: fs, nperseg: 500);
println(spectrum.peak());                                    // 50
let clean = try sosfiltfilt(try butter(6, Band::Lowpass(100hz), fs: fs), x);

// Quadrature and root finding.
let area = try interpolate::quad(f64::sin, 0.0..f64::PI);    // area.value is 2
let root = try optimize::brentq(|x: f64| -> f64 { x * x - 2.0 }, 0.0, 2.0);
```

`special` adds the functions of mathematical physics, and the elementary functions of a complex argument that core's `Complex` leaves out, with the branch cuts of C99 and Python's `cmath`:

```nabi norun
use special::Elementary;

let z = 1.0 + 2.0i;
println(f"{special::gamma(4.5)} {special::jv(0.5, 2.0)} {special::erf(0.5)}");
println(f"{z.sin()} {z.atanh()} {z.cbrt()}");
```

`units` checks dimensions when you compile: adding a length to a time does not build.

```nabi norun
use units::kinds::{Length, Time, Velocity};
use units::symbols::{h, km};

fn speed(d: Length, t: Time) -> Velocity { d / t }

println(speed(42.195km, 2h) → km / h);                        // 21.0975 km/h
```

## Data files and queries

`table` reads and writes CSV and JSON Lines, and the columnar formats the rest of the data world uses: Arrow IPC (files and streams) and Parquet (Snappy compressed by default; zstd pages are read too). A `Table`'s methods run at once; `lazy()` on a table, or `scan_parquet`, `scan_arrow` and `scan_csv` on a file, builds a `Query` instead, which `collect` runs after pushing filters down into the scan and reading only the columns the query uses. A Parquet row group whose statistics rule a filter out is never read.

```nabi norun
use table::{col, desc, mean};
use std::env;

let t = table::table!{
    "station": ["north", "south", "north", "east", "south", "east"],
    "temp":    [12.5, 18.0, 13.5, 15.0, 19.5, 14.0],
    "rain":    [3.0, 0.0, 1.5, 0.5, 0.0, 2.0],
};
let path = f"{env::temp_dir()}/weather.parquet";
try table::write_parquet(t, path);

let q = table::scan_parquet(path)
    .filter(col("rain") .< 2.5)
    .group_by(["station"]).agg([mean("temp")])
    .sort([desc("temp_mean")]);
println(try q.explain());             // the plan: the filter runs inside the scan
println(try q.collect());
```

`table::read_arrow` and `write_arrow` give other Arrow tools (pyarrow, Polars, DuckDB) the same columns with the same types, and Parquet files written here store their Arrow schema, so categorical columns come back as categorical.

## From NumPy, SciPy and pandas

| Python | Nabi |
|--------|------|
| `np.array([[1, 2], [3, 4]])` | `let m: Matrix<f64> = [[1, 2], [3, 4]];` |
| `np.linspace(0, 1, 5)`, `np.arange` | `array::linspace(0.0, 1.0, 5)`, `array::arange(0.0, 1.0, 0.25)` |
| `a.sum(axis=0)`, `a.argmax()`, `np.allclose(a, b)` | `a.sum_along(0)`, `a.argmax()`, `a.all_close(b)` |
| `a @ b`, `np.linalg.solve(a, b)` | `a @ b`, `linalg::solve(a, b)` or `a \ b` |
| `np.linalg.eigh`, `svd`, `cholesky`, `lstsq` | `linalg::eigh`, `svd`, `cholesky`, `lstsq` |
| `scipy.sparse.csr_matrix`, `spsolve`, `cg` | `sparse::Csr`, `sparse::lu(a).solve(b)`, `sparse::cg` |
| `jax.grad(f)(x)` | `autodiff::grad(f, x)` or `∇(f)(x)` |
| `scipy.stats.norm(0, 1).cdf(x)` | `stats::Normal::new(0.0, 1.0).cdf(x)` |
| `np.mean`, `np.std(x, ddof=1)`, `np.quantile` | `stats::mean`, `stats::std(x, ddof: 1)`, `stats::quantile` |
| `scipy.stats.ttest_ind`, `linregress` | `stats::ttest_ind`, `stats::ols` |
| `np.fft.fft`, `rfft`, `scipy.signal.welch` | `signal::fft`, `signal::rfft`, `signal::welch` |
| `scipy.signal.butter(..., output="sos")`, `sosfiltfilt` | `signal::filter::butter`, `sosfiltfilt` |
| `scipy.interpolate.CubicSpline`, `make_interp_spline` | `interpolate::cubic_spline`, `make_interp_spline` |
| `scipy.integrate.quad`, `solve_ivp` | `interpolate::quad` (or `∫(f, a..b)`), `ode::solve_ivp` |
| `scipy.optimize.minimize`, `curve_fit`, `brentq` | `optimize::minimize`, `curve_fit`, `brentq` |
| `scipy.special.gamma`, `jv`, `erf` | `special::gamma`, `jv`, `erf` |
| `pd.read_csv`, `df[df.x > 2]`, `df.groupby("k").agg(...)` | `table::read_csv`, `t.filter(col("x") .> 2)`, `t.group_by(["k"]).agg([...])` |
| `pd.read_parquet`, `pl.scan_parquet(...).filter(...).collect()` | `table::read_parquet`, `table::scan_parquet(..).filter(..).collect()` |
| `plt.plot(x, y); plt.savefig("a.svg")` | `plot::axes().line(x, y).save("a.svg")` |

## Speed

The packages are pure Nabi and use every core when the input is large: dense matrix products run at about 30 GFLOP/s on one Apple M-series core and above 100 GFLOP/s on all of them, FFTs of any length are `O(n log n)`, and a sparse matrix-vector product reads several hundred million nonzeros a second. Each package's README has a "Speed" section with its numbers and how they compare with NumPy, SciPy and the reference libraries. Two habits keep code fast: build arrays with `Array::from_fn` or the dot operators instead of pushing to a list in a loop, and pass `Vector<f64>`s between calls rather than converting back and forth.
