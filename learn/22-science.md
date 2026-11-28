# 22. Science with Nabi

Nabi's numeric core is in the standard library: `Array<T, N>`, the dot operators, reductions, `Complex` and random numbers ([chapter 21](21-arrays-and-numerics.html)). Everything a scientist builds on top of that, linear algebra, statistics, signals, data frames, plots, lives in packages: one package per field, each versioned on its own, all written in Nabi with no C or Fortran underneath, so they build for every target Nabi has, WebAssembly included.

This chapter maps the packages, shows the one rule they share for input data, works one small problem from a CSV file to a plot, points to each package's tutorial, and ends with a table for readers coming from NumPy, SciPy and pandas.

## The packages

| Package | What it does | Python counterpart |
|---------|--------------|--------------------|
| [`linalg`](https://docs.nabi-lang.org/packages/linalg/latest) | dense matrices: products, LU, Cholesky, QR, SVD, eigen, solvers, `expm` | `numpy.linalg`, `scipy.linalg` |
| [`sparse`](https://docs.nabi-lang.org/packages/sparse/latest) | CSR / CSC / COO matrices, CG, GMRES, BiCGSTAB, sparse Cholesky and LU | `scipy.sparse`, `scipy.sparse.linalg` |
| [`spatial`](https://docs.nabi-lang.org/packages/spatial/latest) | small fixed-size vectors, quaternions, matrices, transforms and shapes for games and graphics; `#[repr(C)]`, SIMD `Mat4` | `pyglm`, `scipy.spatial.transform` |
| [`autodiff`](https://docs.nabi-lang.org/packages/autodiff/latest) | derivatives of your own functions: dual numbers, a reverse-mode tape, Hessians, Taylor series | `jax.grad`, `autograd` |
| [`stats`](https://docs.nabi-lang.org/packages/stats/latest) | descriptive statistics, 19 distributions, tests, regression | `scipy.stats`, `statsmodels` (basics) |
| [`signal`](https://docs.nabi-lang.org/packages/signal/latest) | FFT of any length, windows, FIR / IIR filters, spectra, peaks | `numpy.fft`, `scipy.signal` |
| [`interpolate`](https://docs.nabi-lang.org/packages/interpolate/latest) | splines, polynomials, grids, scattered data, quadrature | `scipy.interpolate`, `scipy.integrate.quad` |
| [`optimize`](https://docs.nabi-lang.org/packages/optimize/latest) | minimization, least squares, root finding, linear programming | `scipy.optimize` |
| [`ode`](https://docs.nabi-lang.org/packages/ode/latest) | initial and boundary value problems, stiff solvers, symplectic integrators | `scipy.integrate.solve_ivp` |
| [`special`](https://docs.nabi-lang.org/packages/special/latest) | gamma, Bessel, erf, zeta, elliptic functions; complex elementary functions | `scipy.special` |
| [`table`](https://docs.nabi-lang.org/packages/table/latest) | columnar data frames, CSV, JSONL, Arrow and Parquet files, lazy queries | `pandas`, `polars`, `pyarrow` |
| [`units`](https://docs.nabi-lang.org/packages/units/latest) | quantities with units, checked when you compile | `pint` |
| [`plot`](https://docs.nabi-lang.org/packages/plot/latest) | SVG, PNG and terminal plots, notebook display | `matplotlib` |

Add the ones you use to `nabi.nabion` with `nabi add` (or, for a single-file script, `nabi add <package> file.nabi`):

```sh
nabi add linalg table
```

Each package's docs, a tutorial and its API, are under [Package docs](https://docs.nabi-lang.org/packages): follow the package's name in the table. The code on this page needs those packages, so unlike the rest of this guide it is not run when the site is built; every block was checked with the packages' current versions.

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

## The tutorials

Each package's docs start with a tutorial: a small problem worked step by step, with code that its tests run.

| Tutorial | Package | What it does |
|----------|---------|--------------|
| [A first data analysis](https://docs.nabi-lang.org/packages/table/latest) | `table`, `stats`, `plot` | read a CSV file, clean, group, test a difference, plot |
| [Linear algebra](https://docs.nabi-lang.org/packages/linalg/latest) | `linalg` | solve systems, least squares, eigenvalues |
| [Sparse systems](https://docs.nabi-lang.org/packages/sparse/latest) | `sparse` | a 10 000-unknown sparse system, iterative and direct solvers |
| [Statistics](https://docs.nabi-lang.org/packages/stats/latest) | `stats` | distributions, random samples, tests, regression, the bootstrap |
| [Signals and spectra](https://docs.nabi-lang.org/packages/signal/latest) | `signal` | FFT, power spectra in noise, filters, peaks, convolution |
| [Fitting and optimization](https://docs.nabi-lang.org/packages/optimize/latest) | `optimize` | roots, minima, curve fitting, linear programming |
| [Differential equations](https://docs.nabi-lang.org/packages/ode/latest) | `ode` | initial value problems, events, stiff systems, boundary value problems |
| [Derivatives](https://docs.nabi-lang.org/packages/autodiff/latest) | `autodiff` | exact gradients, Jacobians and Taylor series of your own functions |

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

## Further reading

- Next chapter: [23. Plugins](23-plugins.html)
