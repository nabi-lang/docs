# 5. Fitting and Optimization

Fitting a model to data, finding the input that makes a cost smallest and finding where a function is zero are one family of problems, and `optimize` solves all three. Objectives are ordinary closures. Written over `optimize::Dual` instead of `f64`, a closure gets its exact gradient by automatic differentiation, so you never write a derivative by hand.

## Find a root

Where is `cos x = x`? `brentq` needs an interval where the function changes sign, and from there it always converges. `newton` is faster from a good start but needs the derivative; `newton_ad` takes a closure over `Dual` and gets the derivative by itself.

```nabi
use optimize::Dual;

let f = |x: f64| -> f64 { x.cos() - x };
let r = try optimize::brentq(f, 0.0, 1.0);
println(f"cos x = x at x = {r.x:.12} ({r.iterations} iterations)");

let g = |x: Dual| -> Dual { x.cos() - x };
let n = try optimize::newton_ad(g, 0.5);
println(f"newton: {n.x:.12} ({n.iterations} iterations)");
```

```output
cos x = x at x = 0.739085133215 (6 iterations)
newton: 0.739085133215 (4 iterations)
```

## Minimize a function

The Rosenbrock function has its minimum at (1, 1) at the bottom of a long curved valley, which makes it a standard test. `bfgs` follows the gradient and builds a picture of the curvature as it goes. The result says how it stopped, where and after how many steps.

```nabi
use optimize::Dual;
use core::array::Vector;

let rosen = |x: Vector<Dual>| -> Dual { 100.0 * (x[1] - x[0] ** 2) ** 2 + (1.0 - x[0]) ** 2 };
let m = try optimize::bfgs(rosen, [-1.2, 1.0]);
println(f"{m.status:?} at x = {m.x:.6}, f = {m.fun:.1e}, {m.iterations} iterations");
```

```output
Converged at x = [1.000000, 1.000000], f = 3.8e-19, 37 iterations
```

With bounds on the variables, use `lbfgsb`; with general constraints, `minimize` and its methods; without any gradient at all (a simulation, a measurement), `nelder_mead`. A plain closure over `f64` works everywhere too, with the gradient by finite differences.

```nabi
use optimize::{Bounds, Dual};
use core::array::Vector;

let rosen = |x: Vector<Dual>| -> Dual { 100.0 * (x[1] - x[0] ** 2) ** 2 + (1.0 - x[0]) ** 2 };
let box = try Bounds::new([-2.0, -2.0], [0.5, 2.0]);
let m = try optimize::lbfgsb(rosen, [-1.2, 1.0], box);
println(f"inside the box: x = {m.x:.4}");

let nm = try optimize::nelder_mead(|x: Vector<f64>| -> f64 { (x[0] - 3.0) ** 2 + (x[1] + 1.0) ** 2 }, [0.0, 0.0]);
println(f"no gradient: x = {nm.x:.4}");
```

```output
inside the box: x = [0.5000, 0.2500]
no gradient: x = [3.0000, -1.0000]
```

## Fit a curve

`curve_fit` adjusts the parameters of a model `y = f(x; p)` until it matches the data in the least-squares sense, and reports a standard error for each parameter. The model takes one `x` and the parameter vector. Here the decay of a radioactive sample: counts per minute over time, with a constant background.

```nabi
use optimize::Dual;
use core::array::Vector;

let t: Vector<f64> = [0, 5, 10, 15, 20, 30, 40, 50, 60];
let counts: Vector<f64> = [1011, 760, 590, 455, 362, 238, 169, 128, 110];

// counts = a e^(-t / tau) + c
let model = |t: f64, p: Vector<Dual>| -> Dual { p[0] * (-t / p[1]).exp() + p[2] };
let fit = try optimize::curve_fit(model, t, counts, [1000.0, 20.0, 50.0]);
let (a, tau, c) = (fit.params[0], fit.params[1], fit.params[2]);
println(f"a = {a:.0}, tau = {tau:.2} min, background {c:.0}");
println(f"half-life {tau * 2.0f64.ln():.2} min +- {fit.stderr[1] * 2.0f64.ln():.2}");
```

```output
a = 921, tau = 16.45 min, background 87
half-life 11.40 min +- 0.13
```

Give the starting guess some thought: a nonlinear fit finds the minimum nearest to where it starts. Plotting the data first is usually enough. For a model that is linear in its parameters (a polynomial, a sum of known functions), [least squares](02-linear-algebra.html#fit-a-line-by-least-squares) or `stats::polyfit` need no guess at all.

## Linear programming

A linear program maximizes or minimizes a linear cost under linear constraints: a factory makes two products, each needing machine time and labour, and has only so much of each. `Lp` builds the problem row by row (`le` for `<=` constraints); `linprog` minimizes, so a profit to maximize goes in with its sign flipped.

```nabi
use optimize::Lp;

// Profit 30 per chair and 50 per table. A chair takes 1 h of machine time
// and 2 h of labour, a table 2 h and 3 h; 40 machine hours and 70 labour
// hours are available.
let lp = Lp::new([-30.0, -50.0]).le([[1, 2], [2, 3]], [40.0, 70.0]);
let r = try optimize::linprog(lp);
println(f"chairs {r.x[0]:.0}, tables {r.x[1]:.0}, profit {-r.fun:.0}");
```

```output
chairs 20, tables 10, profit 1100
```

`quadprog` solves problems with a quadratic cost. Integer variables are not supported.

## What next

- [Differential equations](06-ode.html): models that are equations for a rate of change.
- [Derivatives](07-autodiff.html): the automatic differentiation behind `Dual`, on your own functions.
- The [`optimize` API](https://packages.nabi-lang.org/packages/optimize): every method, the options and global optimizers.
