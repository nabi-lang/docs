# 6. 미분방정식

많은 모델은 어떤 양이 무엇인지보다 어떻게 변하는지를 말합니다. 개체 수는 그 크기에 비례해 늘고, 진자는 바닥 쪽으로 되돌아가며 가속됩니다. 이런 것이 상미분방정식 `y' = f(t, y)`이며, `ode`가 이를 풉니다. `f`는 클로저로 쓰고, 풀이기는 허용 오차를 지키도록 걸음 크기를 고른 다음, 걸음마다의 해와 함께 아무 시각에서나 부를 수 있는 함수로서의 해를 돌려줍니다.

## 첫 방정식

지수 감쇠 `y' = -0.5 y`, `y(0) = 10`의 정확한 해는 `10 e^(-t/2)`입니다. `solve_ivp`는 우변, 범위로 쓴 시간 구간, 초기 상태를 받습니다. 상태는 성분이 하나여도 `Vector<f64>`입니다.

```nabi
use core::array::Vector;

let f = |t: f64, y: Vector<f64>| -> Vector<f64> { y .* -0.5 };
let r = try ode::solve_ivp(f, 0.0..4.0, [10.0]);
println(f"{r.status:?} after {r.steps} steps, {r.nfev} evaluations");
println(f"y(4) = {r.last()[0]:.6}, exact {10.0 * (-2.0f64).exp():.6}");
```

```output
Finished after 4 steps, 26 evaluations
y(4) = 1.354160, exact 1.353353
```

기본 방법은 `Rk45`(Dormand-Prince)이고 `rtol: 1e-3`이라 그래프에는 충분합니다. 더 정확하게 하려면 `rtol:`과 `atol:`을 주세요. 풀이기는 국소 오차를 `atol + rtol * |y|` 아래로 유지합니다.

## 연립 방정식과 함수로서의 해

진자: 각도 `θ`와 각속도 `ω`가 있고 `θ' = ω`, `ω' = -(g / L) sin θ`입니다. 2계 방정식은 1계 방정식 두 개의 연립이 됩니다. `r.sol`은 밀집 출력, 곧 시간의 함수이므로 다시 풀지 않고도 구간 어디에서나 해를 읽을 수 있습니다.

```nabi
use core::array::{Vector, linspace};

let g_over_l = 9.81 / 1.0;
let pendulum = |t: f64, y: Vector<f64>| -> Vector<f64> { [y[1], -g_over_l * y[0].sin()] };
let r = try ode::solve_ivp(pendulum, 0.0..10.0, [1.0, 0.0], rtol: 1e-9, atol: 1e-12);

let sol = r.sol;
for t in [0.5, 1.0, 1.5, 2.0] {
    println(f"t = {t}: angle {sol(t)[0]:>8.5} rad");
}
let ts: Vector<f64> = linspace(0.0, 10.0, 201);
let angles = sol(ts)[0, :];
println(f"{angles.len()} points for a plot, smallest {angles.min() ?? 0.0:.5}");
```

```output
t = 0.5: angle  0.10426 rad
t = 1: angle -0.98007 rad
t = 1.5: angle -0.30787 rad
t = 2: angle  0.92079 rad
201 points for a plot, smallest -0.99998
```

시각 하나에서 `sol(t)`는 상태를 벡터로 줍니다. 시각들의 벡터에서는 성분마다 행이 하나인 행렬을 주므로, `[0, :]`은 모든 시각에서의 각도입니다.

## 이벤트

이벤트는 상태의 함수로, 풀이기가 그 함수가 0이 되는 곳을 정확히 찾습니다. 여기서는 공을 위로 던지고, 이벤트 `ground`가 공이 땅에 닿을 때 실행을 멈춥니다. `t_events`에는 각 이벤트가 일어난 시각이 들어 있습니다.

```nabi
use ode::{Direction, Event};
use core::array::Vector;

let fall = |t: f64, y: Vector<f64>| -> Vector<f64> { [y[1], -9.81] };
let ground = Event { f: |t: f64, y: Vector<f64>| -> f64 { y[0] }, direction: Direction::Falling, terminal: true };
let r = try ode::solve_ivp(fall, 0.0..10.0, [0.0, 15.0], events: [ground], rtol: 1e-10);
let landed = r.t_events[0][0];
println(f"lands after {landed:.4} s, exact {2.0 * 15.0 / 9.81:.4} s");
```

```output
lands after 3.0581 s, exact 3.0581 s
```

## 강성 방정식

어떤 시스템은 아주 빠른 변화와 아주 느린 변화가 섞여 있습니다. 반응 속도가 여러 자릿수씩 차이 나는 화학 반응이 그렇습니다. `Rk45` 같은 명시적 방법은 해가 거의 움직이지 않는 곳에서도 안정을 유지하려면 아주 작은 걸음을 걸어야 합니다. 암시적 방법(`Radau`, `Bdf`, `Rodas`)은 큰 걸음을 걷고, `Auto`는 문제가 바뀌는 데 따라 두 종류를 오갑니다. Robertson 반응이 고전적인 시험 문제입니다.

```nabi
use ode::Method;
use core::array::Vector;

let robertson = |t: f64, y: Vector<f64>| -> Vector<f64> {
    [-0.04 * y[0] + 1e4 * y[1] * y[2],
     0.04 * y[0] - 1e4 * y[1] * y[2] - 3e7 * y[1] * y[1],
     3e7 * y[1] * y[1]]
};
let y0: Vector<f64> = [1.0, 0.0, 0.0];
for m in [Method::Bdf, Method::Radau, Method::Auto] {
    let r = try ode::solve_ivp(robertson, 0.0..1e5, y0, method: m, rtol: 1e-6, atol: 1e-10);
    let y = r.last();
    println(f"{m:?}: {r.steps} steps, y = [{y[0]:.4}, {y[1]:.3e}, {y[2]:.4}]");
}
```

```output
Bdf: 327 steps, y = [0.0179, 7.275e-8, 0.9821]
Radau: 188 steps, y = [0.0179, 7.275e-8, 0.9821]
Auto: 302 steps, y = [0.0179, 7.275e-8, 0.9821]
```

## 경곗값 문제

모든 문제가 알려진 상태에서 시작하지는 않습니다. 보는 양 끝이 고정되어 있고, 벽의 온도는 양쪽에서 유지됩니다. `solve_bvp`는 방정식, 조건이 성립할 때 0이 되는 양 끝의 함수, 격자, 첫 추정값을 받습니다. 여기서는 두 기둥 사이에 걸린 케이블 `y'' = sqrt(1 + y'^2) / a`이며, 양 끝에서 `y = 0`입니다.

```nabi
use core::array::{Array, Matrix, Vector, linspace};

let a = 2.0;
let cable = |x: f64, y: Vector<f64>| -> Vector<f64> { [y[1], (1.0 + y[1] * y[1]).sqrt() / a] };
let ends = |ya: Vector<f64>, yb: Vector<f64>| -> Vector<f64> { [ya[0], yb[0]] };
let x: Vector<f64> = linspace(-1.0, 1.0, 11);
let guess: Matrix<f64> = Array::zeros([2, 11]);
let b = try ode::solve_bvp(cable, ends, x, guess, tol: 1e-8);

// The exact shape is a catenary: a cosh(x / a) - a cosh(1 / a).
let sag = b.sol.at(0.0)[0];
println(f"{b.success()}: sag at the middle {sag:.6}, exact {a - a * (1.0 / a).cosh():.6}");
```

```output
true: sag at the middle -0.255252, exact -0.255252
```

## 다음으로

- [도함수](07-autodiff.html): 암시적 풀이기에 필요한 야코비 행렬을 대신 계산합니다.
- [`ode` API](https://packages.nabi-lang.org/packages/ode): 고정 걸음과 심플렉틱 적분기, 야코비 옵션, 모든 상태값.
