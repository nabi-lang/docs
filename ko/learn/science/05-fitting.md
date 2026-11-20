# 5. 피팅과 최적화

데이터에 모델을 맞추는 일, 비용을 가장 작게 만드는 입력을 찾는 일, 함수가 0이 되는 곳을 찾는 일은 한 갈래의 문제이며, `optimize`는 셋을 모두 풉니다. 목적 함수는 평범한 클로저입니다. `f64` 대신 `optimize::Dual` 위에서 쓴 클로저는 자동 미분으로 정확한 기울기를 얻으므로, 도함수를 손으로 쓸 일이 없습니다.

## 근 찾기

`cos x = x`는 어디일까요? `brentq`는 함수의 부호가 바뀌는 구간이 있어야 하며, 그러면 언제나 수렴합니다. `newton`은 좋은 출발점에서 더 빠르지만 도함수가 필요합니다. `newton_ad`는 `Dual` 위의 클로저를 받아 도함수를 스스로 구합니다.

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

## 함수의 최솟값 찾기

Rosenbrock 함수는 길게 휜 골짜기의 바닥 (1, 1)에서 최솟값을 가지므로 표준 시험 문제로 쓰입니다. `bfgs`는 기울기를 따라가며 곡률의 모습을 차츰 쌓아 갑니다. 결과는 어떻게 멈췄는지, 어디서, 몇 걸음 만에 멈췄는지를 알려 줍니다.

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

변수에 범위가 있으면 `lbfgsb`를, 일반적인 제약이 있으면 `minimize`와 그 방법들을, 기울기가 전혀 없으면(시뮬레이션, 측정) `nelder_mead`를 쓰세요. `f64` 위의 평범한 클로저도 어디서나 되며, 기울기는 유한 차분으로 구합니다.

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

## 곡선 맞추기

`curve_fit`은 모델 `y = f(x; p)`의 모수를 최소제곱의 의미로 데이터에 맞을 때까지 조정하고, 모수마다 표준 오차를 알려 줍니다. 모델은 `x` 하나와 모수 벡터를 받습니다. 여기서는 방사성 시료의 붕괴입니다. 시간에 따른 분당 계수에 일정한 배경값이 더해져 있습니다.

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

시작 추정값은 조금 신경 써서 고르세요. 비선형 피팅은 시작한 곳에서 가장 가까운 최솟값을 찾습니다. 보통은 데이터를 먼저 그려 보는 것으로 충분합니다. 모수에 대해 선형인 모델(다항식, 알려진 함수들의 합)이라면 [최소제곱](02-linear-algebra.html#최소제곱으로-직선-맞추기)이나 `stats::polyfit`에는 추정값이 아예 필요 없습니다.

## 선형 계획법

선형 계획법은 선형 제약 아래에서 선형 비용을 최대화하거나 최소화합니다. 공장이 두 제품을 만드는데, 제품마다 기계 시간과 노동이 들고 둘 다 쓸 수 있는 양이 정해져 있다고 합시다. `Lp`는 문제를 한 행씩 만들고(`<=` 제약에는 `le`), `linprog`는 최소화하므로 최대화할 이익은 부호를 바꿔 넣습니다.

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

`quadprog`는 비용이 2차식인 문제를 풉니다. 정수 변수는 지원하지 않습니다.

## 다음으로

- [미분방정식](06-ode.html): 변화율에 대한 방정식인 모델.
- [도함수](07-autodiff.html): `Dual` 뒤의 자동 미분을 직접 쓴 함수에.
- [`optimize` API](https://packages.nabi-lang.org/packages/optimize): 모든 방법, 옵션, 전역 최적화.
