# 7. Derivatives

Optimizers, implicit ODE solvers, sensitivity studies and machine learning all need derivatives of functions you wrote. Finite differences are easy but lose a third of the digits or more and cost one evaluation per input. `autodiff` computes them exactly, to the last bit of `f64`, by running your function on numbers that carry their derivatives along.

## One variable

A `Dual` number is a value and a derivative. Write the function over `Dual` and `autodiff::derivative` returns its derivative at a point.

```nabi
use autodiff::Dual;

let f = |x: Dual| -> Dual { x.sin() * x.exp() };
let x = 1.0;
let exact = x.cos() * x.exp() + x.sin() * x.exp();
println(f"f'(1) = {autodiff::derivative(f, x)}");
println(f"exact = {exact}");

// The same with central differences: about ten digits right.
let h = 1e-5;
let fd = ((x + h).sin() * (x + h).exp() - (x - h).sin() * (x - h).exp()) / (2.0 * h);
println(f"finite difference error {(fd - exact).abs():.1e}");
```

```output
f'(1) = 3.7560492270947274
exact = 3.7560492270947274
finite difference error 2.9e-11
```

## Write a function once, differentiate it many ways

A function generic over `Smooth` runs on `f64` and on every number type of the package. The type you call it with picks the method: `Dual` is forward mode, one pass per input; `Var` is reverse mode, one pass forward and one backward for the whole gradient, whatever the number of inputs; `HyperDual` gives exact second derivatives.

```nabi
use autodiff::{Dual, HyperDual, Smooth, Var};
use core::array::Vector;

// The energy of a spring network: three points on a line.
fn energy<T: Smooth>(x: Vector<T>) -> T {
    let s1 = x[1].clone() - x[0].clone() - T::of(1.0);
    let s2 = x[2].clone() - x[1].clone() - T::of(1.0);
    s1.clone() * s1 + s2.clone() * s2 * 2.0
}

let x: Vector<f64> = [0.0, 1.5, 2.0];
println(f"energy {energy(x)}");
println(f"gradient (reverse) {autodiff::grad(energy::<Var>, x)}");
println(f"gradient (forward) {autodiff::grad(energy::<Dual>, x)}");
println(f"hessian {autodiff::hessian(energy::<HyperDual>, x)}");
```

```output
energy 0.75
gradient (reverse) [-1.0, 3.0, -2.0]
gradient (forward) [-1.0, 3.0, -2.0]
hessian [[2.0, -2.0, 0.0], [-2.0, 6.0, -4.0], [0.0, -4.0, 4.0]]
```

Reverse mode is the choice for a scalar function of many inputs, such as a loss over model weights: the cost of the gradient stays a small multiple of one evaluation. Forward mode wins when there are few inputs.

## Jacobians

For a function from `n` numbers to `m` numbers, `jacobian` returns the `m x n` matrix of partial derivatives. It is what Newton's method for systems and the implicit ODE solvers need. Here polar to Cartesian coordinates, whose Jacobian determinant is `r`.

```nabi
use autodiff::Var;
use core::array::{Array, Vector};

let polar = |p: Vector<Var>| -> Vector<Var> { Array::from([p[0] * p[1].cos(), p[0] * p[1].sin()]) };
let j = autodiff::jacobian(polar, [2.0, 0.5]);
println(j);
println(f"det = {(try linalg::det(j)):.12}");
```

```output
[[0.8775825618903728, -0.958851077208406], [0.479425538604203, 1.7551651237807455]]
det = 2.000000000000
```

## Taylor coefficients

A `Jet` carries a whole truncated Taylor series. `derivatives` returns the value and the first `n` derivatives at a point in one pass, exactly.

```nabi
use autodiff::Jet;

let d = autodiff::derivatives(|x: Jet| -> Jet { x.exp() * x.sin() }, 0.0, 5);
println(f"f, f', .., f^(5) at 0: {d}");
```

```output
f, f', .., f^(5) at 0: [0.0, 1.0, 2.0, 2.0, 0.0, -4.0]
```

## Gradients for an optimizer

`optimize` takes functions over `Dual` or `Var` directly and asks for their gradients itself. Here a small logistic regression, fitted by minimizing its negative log-likelihood with L-BFGS; the gradient comes from reverse mode.

```nabi
use autodiff::Var;
use core::array::Vector;

// Hours studied, and whether the exam was passed.
let hours: List<f64> = [0.5, 1.0, 1.5, 2.0, 2.5, 3.0, 3.5, 4.0, 4.5, 5.0];
let passed: List<f64> = [0.0, 0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0, 1.0];

let nll = |w: Vector<Var>| -> Var {
    let total = Var::constant(0.0);
    for k in 0..hours.len() {
        let z = w[0].clone() + w[1].clone() * hours[k];
        // log(1 + e^z) - y z, the logistic loss
        total = total + (z.exp() + 1.0).ln() - z * passed[k];
    }
    total
};
let m = try optimize::lbfgs(nll, [0.0, 0.0]);
println(f"intercept {m.x[0]:.3}, slope {m.x[1]:.3}");
println(f"pass chance after 3 hours: {1.0 / (1.0 + (-(m.x[0] + 3.0 * m.x[1])).exp()):.3}");
```

```output
intercept -2.990, slope 1.087
pass chance after 3 hours: 0.568
```

`stats::logistic` fits the same model directly and adds standard errors; writing it out shows how any likelihood can be fitted the same way.

## Notation

With the `ops` feature (on by default), `∂(f)` is the derivative of `f` as a function, and `∇(f)` its gradient. Both return the derivative in the form of what they are given: a function for a function, a spline for a spline, samples for samples.

```nabi
use autodiff::{Dual, Var};
use autodiff::ops::{∂, ∇};
use core::array::Vector;

let dsin = ∂(|x: Dual| -> Dual { x.sin() });
println(f"{dsin(0.0)} {dsin(f64::PI)}");
let g = ∇(|x: Vector<Var>| -> Var { x[0] * x[1] * x[1] });
println(g([3.0, 2.0]));
```

```output
1 -1
[4.0, 12.0]
```

## What next

- [The science path](index.html): every tutorial, and where each package fits.
- The [`autodiff` API](https://packages.nabi-lang.org/packages/autodiff): `Duals<N>` for many forward lanes at once, `jvp` and `vjp`, using a `Tape` by hand.
