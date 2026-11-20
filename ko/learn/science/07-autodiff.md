# 7. 도함수

최적화기, 암시적 ODE 풀이기, 민감도 분석, 기계 학습에는 모두 직접 쓴 함수의 도함수가 필요합니다. 유한 차분은 쉽지만 자릿수의 3분의 1 이상을 잃고 입력마다 평가 한 번씩이 듭니다. `autodiff`는 도함수를 함께 지니고 다니는 수 위에서 함수를 돌려, `f64`의 마지막 비트까지 정확하게 도함수를 계산합니다.

## 변수 하나

`Dual` 수는 값 하나와 도함수 하나입니다. 함수를 `Dual` 위에서 쓰면 `autodiff::derivative`가 한 점에서의 도함수를 돌려줍니다.

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

## 함수는 한 번 쓰고, 미분은 여러 방법으로

`Smooth`에 대해 제네릭인 함수는 `f64`와 이 패키지의 모든 수 타입에서 돌아갑니다. 어떤 타입으로 부르느냐가 방법을 정합니다. `Dual`은 전진 모드로 입력마다 한 번씩 돌고, `Var`는 역방향 모드로 입력이 몇 개든 앞으로 한 번, 뒤로 한 번 돌아 기울기 전체를 얻습니다. `HyperDual`은 정확한 2계 도함수를 줍니다.

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

역방향 모드는 모델 가중치에 대한 손실처럼 입력이 많은 스칼라 함수에 알맞습니다. 기울기의 비용이 평가 한 번의 작은 배수로 머뭅니다. 입력이 적으면 전진 모드가 낫습니다.

## 야코비 행렬

수 `n`개에서 수 `m`개로 가는 함수에 대해 `jacobian`은 편도함수의 `m x n` 행렬을 돌려줍니다. 연립 방정식용 뉴턴 방법과 암시적 ODE 풀이기에 필요한 것이 이것입니다. 여기서는 극좌표에서 직교 좌표로의 변환이며, 그 야코비 행렬식은 `r`입니다.

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

## 테일러 계수

`Jet`은 잘린 테일러 급수 전체를 지니고 다닙니다. `derivatives`는 한 점에서의 값과 처음 `n`개의 도함수를 한 번에 정확하게 돌려줍니다.

```nabi
use autodiff::Jet;

let d = autodiff::derivatives(|x: Jet| -> Jet { x.exp() * x.sin() }, 0.0, 5);
println(f"f, f', .., f^(5) at 0: {d}");
```

```output
f, f', .., f^(5) at 0: [0.0, 1.0, 2.0, 2.0, 0.0, -4.0]
```

## 최적화기를 위한 기울기

`optimize`는 `Dual`이나 `Var` 위의 함수를 바로 받아 기울기를 스스로 구합니다. 여기서는 작은 로지스틱 회귀를, 음의 로그 가능도를 L-BFGS로 최소화해 맞춥니다. 기울기는 역방향 모드에서 나옵니다.

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

`stats::logistic`은 같은 모델을 바로 맞추고 표준 오차도 더해 줍니다. 직접 써 보면 어떤 가능도든 같은 방법으로 맞출 수 있다는 것이 보입니다.

## 표기법

`ops` 기능(기본으로 켜짐)이 있으면 `∂(f)`는 함수로서의 `f`의 도함수이고, `∇(f)`는 그 기울기입니다. 둘 다 받은 것과 같은 형태로 도함수를 돌려줍니다. 함수에는 함수, 스플라인에는 스플라인, 표본에는 표본입니다.

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

## 다음으로

- [과학 계산 길잡이](index.html): 모든 튜토리얼과 각 패키지의 자리.
- [`autodiff` API](https://packages.nabi-lang.org/packages/autodiff): 여러 전진 레인을 한 번에 쓰는 `Duals<N>`, `jvp`와 `vjp`, `Tape`를 직접 쓰기.
