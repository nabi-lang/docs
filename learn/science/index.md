# Science and Data Analysis

A path through Nabi's science packages, one tutorial per field. Each one starts from a small problem and builds on it, and every code block on these pages is compiled and run against the real packages when the site is built, with the output shown under it.

Read [Arrays and Numerics](../21-arrays-and-numerics.html) first if `Vector<f64>`, `Matrix<f64>` and the dot operators (`.+`, `.*`) are new to you: every package below takes and returns them. [Science with Nabi](../22-science.html) is the overview: what each package does, the one rule they share for input data, and a table from NumPy, SciPy and pandas.

## No setup

The science packages are official packages: they come with the toolchain. A script that writes `table::read_csv(..)` or `use stats::Normal;` gets the package added on its first run, with no `nabi add` and no download:

```text
$ nabi run tea.nabi
       Added table ^0.1 (official)
       Added stats ^0.1 (official)
```

In a package the same happens to `nabi.nabion`, and `nabi.lock` pins the version. In a notebook (`nabi jupyter`), a cell that uses a package adds it to the notebook's environment.

## The tutorials

| Tutorial | Packages | What it does |
|----------|----------|--------------|
| [A first data analysis](01-data-analysis.html) | `table`, `stats`, `plot` | read a CSV file, clean, group, test a difference, plot |
| [Linear algebra, dense and sparse](02-linear-algebra.html) | `linalg`, `sparse` | solve systems, least squares, eigenvalues, a 10 000-unknown sparse system |
| [Statistics](03-statistics.html) | `stats` | distributions, random samples, tests, regression, the bootstrap |
| [Signals and spectra](04-signals.html) | `signal` | FFT, power spectra in noise, filters, peaks, convolution |
| [Fitting and optimization](05-fitting.html) | `optimize` | roots, minima, curve fitting, linear programming |
| [Differential equations](06-ode.html) | `ode` | initial value problems, events, stiff systems, boundary value problems |
| [Derivatives](07-autodiff.html) | `autodiff` | exact gradients, Jacobians and Taylor series of your own functions |

The packages not covered here have their own guides on [packages.nabi-lang.org](https://packages.nabi-lang.org): `interpolate` (splines and quadrature), `special` (special functions) and `units` (quantities checked when you compile).

For short recipes outside science, see the [Rosetta](../rosetta/index.html) pages: common tasks, each solved in a few lines of Nabi.
