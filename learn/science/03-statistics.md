# 3. Statistics

`stats` covers what a first statistics course does and a good part of what comes after: summaries, probability distributions, random samples, hypothesis tests, regression and the bootstrap. Each function takes any sequence of numbers (a `Vector<f64>`, a `List`, a fixed array, a range), and the results are plain structs whose fields you read by name.

## Summaries

`mean`, `median`, `var`, `std`, `quantile` and the rest take the data and return a number. Descriptive statistics never throw: an empty input gives NaN, as in NumPy. `stats::nan::` has the same functions skipping NaN values.

```nabi
let x = [5.1, 4.9, 5.6, 5.8, 6.0, 5.4, 5.2, 5.7];
println(f"mean {stats::mean(x):.4}, median {stats::median(x):.2}");
println(f"var {stats::var(x, ddof: 1):.4}, std {stats::std(x, ddof: 1):.4}");
println(f"90th percentile {(try stats::quantile(x, 0.9)):.3}");
println(f"skew {stats::skew(x):.3}");

let gaps = [1.0, f64::NAN, 3.0];
println(f"{stats::mean(gaps)} {stats::nan::mean(gaps)}");
```

```output
mean 5.4625, median 5.50
var 0.1427, std 0.3777
90th percentile 5.860
skew -0.097
NaN 2
```

`quantile` interpolates as NumPy does by default; the `method:` argument picks any of the Hyndman and Fan definitions when a field expects another.

## Distributions

A distribution is a value: `Normal::new(100.0, 15.0)` is the normal with mean 100 and standard deviation 15. Continuous distributions have `pdf`, `cdf`, `sf` (one minus the cdf, accurate in the tail), `ppf` (the inverse cdf) and `interval`; discrete ones have `pmf` instead of `pdf`.

```nabi
use stats::{Binomial, Continuous, Discrete, Normal};

let iq = try Normal::new(100.0, 15.0);
println(f"P(IQ > 130) = {iq.sf(130.0):.4}");
println(f"middle 95%: {iq.interval(0.95)}");
println(f"97.5th percentile {iq.ppf(0.975):.2}");

let coin = try Binomial::new(10, 0.5);
println(f"P(7 heads of 10) = {coin.pmf(7):.4}, P(at most 7) = {coin.cdf(7):.4}");
```

```output
P(IQ > 130) = 0.0228
middle 95%: [70.6005402318992, 129.3994597681008]
97.5th percentile 129.40
P(7 heads of 10) = 0.1172, P(at most 7) = 0.9453
```

## Random samples

A `Sampler` is a seeded random number generator: the same seed gives the same draws on every machine, which makes a simulation repeatable. `d.samples(ref g, n)` draws `n` values from `d`.

```nabi
use stats::{Continuous, Distribution, Normal, Sampler};

let g = Sampler::seed(7);
let d = try Normal::new(10.0, 2.0);
let xs = d.samples(ref g, 10000);
println(f"{xs.len()} draws: mean {stats::mean(xs):.2}, std {stats::std(xs, ddof: 1):.2}");

let fitted = try Normal::fit(xs);
println(f"fitted mean {fitted.mean():.2}");
```

```output
10000 draws: mean 10.00, std 1.99
fitted mean 10.00
```

`fit` estimates a distribution's parameters by maximum likelihood; most distributions in the package have one.

## Hypothesis tests

A test returns its statistic and p-value. The t-tests also return the estimate and its standard error, so the confidence interval is one more call (`ci`). Here a drug trial: did the treated group's blood pressure fall more than the control group's?

```nabi
let control = [-2.1, 0.5, -1.0, 1.3, -0.4, 0.8, -1.7, 0.2, -0.6, 1.1];
let treated = [-6.3, -4.1, -7.5, -2.9, -5.2, -3.8, -6.9, -4.4, -5.8, -3.1];

let t = try stats::ttest_ind(treated, control);
println(f"difference {t.estimate:.2} mmHg, 95% interval {t.ci(0.95)}");
println(f"p = {t.pvalue:.2e}, significant at 1%: {t.result().rejects(0.01)}");

// Are the treated values plausibly normal? Shapiro-Wilk.
let sw = try stats::shapiro(treated);
println(f"Shapiro-Wilk p = {sw.pvalue:.3}");

// The same question without assuming normality.
let u = try stats::mannwhitneyu(treated, control);
println(f"Mann-Whitney p = {u.pvalue:.2e}");
```

```output
difference -4.81 mmHg, 95% interval [-6.131505142918329, -3.48849485708167]
p = 7.45e-7, significant at 1%: true
Shapiro-Wilk p = 0.696
Mann-Whitney p = 1.83e-4
```

The package also has paired and one-sample t-tests, chi-squared tests of fit and of contingency tables, Fisher's exact test, Kolmogorov-Smirnov, Wilcoxon, one-way ANOVA (`f_oneway`) and the correlation tests. Each takes `alternative:` for a one-sided test.

## Regression

`stats::ols` fits `y = b0 + b1 x1 + ...` by least squares and reports what a regression table does: coefficients, standard errors, t statistics, p-values, R². Each row of `x` is one observation, each column one predictor; the intercept is added for you.

```nabi
use core::array::{Matrix, Vector};

// Fuel use (l/100 km) against mass (tonnes) and engine size (litres).
let x: Matrix<f64> = [[1.1, 1.2], [1.3, 1.4], [1.4, 1.6], [1.5, 2.0], [1.7, 2.0], [1.8, 2.5], [2.0, 3.0], [2.2, 3.0]];
let y: Vector<f64> = [5.2, 5.9, 6.3, 7.1, 7.4, 8.3, 9.2, 9.6];

let fit = try stats::ols(x, y);
for (k, name) in ["intercept", "mass", "engine"].enumerate() {
    println(f"{name:>9}: {fit.coef[k]:>6.3} (se {fit.se[k]:.3}, p {fit.p[k]:.3})");
}
println(f"R^2 {fit.r2:.4}");

let next: Matrix<f64> = [[1.6, 1.8]];
println(f"predicted for a 1.6 t car with a 1.8 l engine: {(try fit.predict(next))[0]:.2}");
```

```output
intercept:  1.684 (se 0.155, p 0.000)
     mass:  1.807 (se 0.259, p 0.001)
   engine:  1.319 (se 0.138, p 0.000)
R^2 0.9991
predicted for a 1.6 t car with a 1.8 l engine: 6.95
```

`stats::logistic` fits a logistic regression for a yes/no outcome, and `stats::polyfit` a polynomial in one variable.

## The bootstrap

When no formula gives the uncertainty of a statistic, the bootstrap estimates it by recomputing the statistic on many resamples of the data. `stats::bootstrap` takes the data and the statistic as a function and returns the estimate, its standard error and a confidence interval (BCa by default).

```nabi
use core::array::Vector;

let incomes = [21.0, 23.5, 25.0, 26.0, 28.5, 30.0, 31.0, 34.0, 38.0, 45.0, 60.0, 120.0];
let b = try stats::bootstrap(incomes, |v: Vector<f64>| -> f64 { stats::median(v) }, resamples: 2000, seed: 1);
println(f"median {b.estimate}, se {b.se:.2}, 95% interval {b.ci}");
```

```output
median 30.5, se 4.09, 95% interval [25, 38]
```

## Notation

With the `ops` feature (on by default) some of this reads like a statistics text: `95pct` is the number 0.95, `m ± h` is an interval, and `g ~ d` draws from `d`.

```nabi
use stats::{Distribution, Normal, Sampler};

let g = Sampler::seed(42);
let xs = g ~ (try Normal::new(10.0, 2.0)).iid(1000);
let ci = stats::mean(xs) ± 1.96 * stats::sem(xs);
println(f"{ci} contains 10: {10.0 in ci}");
println(95pct);
```

```output
[9.886659540274959, 10.129771738653204] contains 10: true
0.95
```

## What next

- [Signals and spectra](04-signals.html): FFT and filters.
- The [`stats` API](https://packages.nabi-lang.org/packages/stats): every distribution, test and option.
