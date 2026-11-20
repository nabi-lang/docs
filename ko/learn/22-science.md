# 22. Nabi로 하는 과학 계산

Nabi의 수치 계산 핵심은 표준 라이브러리에 있습니다. `Array<T, N>`, 점 연산자, 집계, `Complex`, 난수가 그렇습니다([21장](21-arrays-and-numerics.html)). 그 위에 과학자가 쌓는 선형대수, 통계, 신호, 데이터 프레임, 그래프는 패키지에 있습니다. 분야마다 패키지가 하나씩 있고 버전도 따로 매기며, 모두 C나 Fortran 없이 Nabi로 작성되어 WebAssembly를 포함한 Nabi의 모든 타깃에서 빌드됩니다.

이 장은 패키지들의 지도를 보이고, 입력 데이터에 대해 모든 패키지가 공유하는 규칙 하나를 설명하고, CSV 파일에서 그래프까지 작은 문제 하나를 풀어 본 뒤, NumPy, SciPy, pandas에서 오는 독자를 위한 대응표로 끝납니다.

## 패키지

| 패키지 | 하는 일 | Python의 대응 |
|--------|---------|---------------|
| `linalg` | 밀집 행렬: 곱, LU, Cholesky, QR, SVD, 고윳값, 연립방정식, `expm` | `numpy.linalg`, `scipy.linalg` |
| `sparse` | CSR / CSC / COO 행렬, CG, GMRES, BiCGSTAB, 희소 Cholesky와 LU | `scipy.sparse`, `scipy.sparse.linalg` |
| `spatial` | 게임과 그래픽스용 고정 크기 벡터, 쿼터니언, 행렬, 변환, 도형; `#[repr(C)]`, SIMD `Mat4` | `pyglm`, `scipy.spatial.transform` |
| `autodiff` | 직접 쓴 함수의 도함수: 이중수, 역방향 테이프, 헤세 행렬, 테일러 급수 | `jax.grad`, `autograd` |
| `stats` | 기술 통계, 19가지 분포, 검정, 회귀 | `scipy.stats`, `statsmodels`(기본) |
| `signal` | 임의 길이 FFT, 창 함수, FIR / IIR 필터, 스펙트럼, 피크 | `numpy.fft`, `scipy.signal` |
| `interpolate` | 스플라인, 다항식, 격자, 흩어진 점, 수치 적분 | `scipy.interpolate`, `scipy.integrate.quad` |
| `optimize` | 최소화, 최소제곱, 근 찾기, 선형 계획법 | `scipy.optimize` |
| `ode` | 초깃값 문제와 경곗값 문제, 강성 솔버, 심플렉틱 적분기 | `scipy.integrate.solve_ivp` |
| `special` | 감마, 베셀, erf, 제타, 타원 함수, 복소 초등 함수 | `scipy.special` |
| `table` | 열 기반 데이터 프레임, CSV, JSONL, Arrow와 Parquet 파일, 지연 쿼리 | `pandas`, `polars`, `pyarrow` |
| `units` | 단위가 붙은 양, 컴파일할 때 검사 | `pint` |
| `plot` | SVG, PNG, 터미널 그래프, 노트북 표시 | `matplotlib` |

쓰는 패키지는 `nabi add`로 `nabi.nabion`에 추가합니다(파일 하나짜리 스크립트라면 `nabi add <패키지> file.nabi`).

```sh
nabi add linalg
nabi add table
```

각 패키지의 API 문서는 [packages.nabi-lang.org](https://packages.nabi-lang.org)에 있습니다. 이 페이지의 코드는 이 패키지들이 필요하므로, 이 가이드의 다른 코드와 달리 사이트를 빌드할 때 실행하지 않습니다. 모든 블록은 패키지의 현재 버전으로 확인했습니다.

## 입력은 한 가지 방식으로: 배열 같은 값

모든 패키지는 수의 나열을 같은 방식으로 받습니다. `f64` 벡터를 원하는 함수는 `linalg::AsVector<f64>`가 다루는 것을 모두 받습니다. `f64`, `f32`, 정수의 `Vector`, `List`, 고정 크기 배열, 그리고 정수 범위입니다. `Vector<f64>`와 `List<f64>`는 그 자리에서 읽고, 나머지는 한 번 변환합니다.

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

`stats`는 차원 수와 관계없이 배열을 행 우선 순서로 읽고, `plot`은 같은 형태에 더해 격자용 중첩 리스트도 받습니다. 직접 쓰는 함수도 같은 트레이트를 받을 수 있습니다.

```nabi norun
use linalg::AsVector;
use core::array::Vector;

fn rms<X: AsVector<f64>>(x: X) -> f64 {
    let v: Vector<f64> = x.as_vector();
    (v .* v).mean().sqrt()
}

println(rms([3, 4]));        // 3.5355339059327378
```

섞인 리터럴은 원소 타입을 앞에 씁니다. `[0.0, 0.5, 1]`은 부동소수점 리스트지만, `[0, 0.5, 1]`은 정수 리스트로 시작하므로 `0.5`에서 에러가 납니다.

## 예제 하나를 끝까지

차 한 잔이 90도에서 식어 갑니다. 측정값은 CSV 파일에 있습니다. 뉴턴의 냉각 법칙은 온도가 방 온도와의 차이에 비례하는 속도로 방 온도를 향해 내려간다고 말합니다. 식으로는 `T' = k (room - T)`입니다. 데이터에 `room`과 `k`를 맞추고, 법칙을 ODE로 풀고, 맞춤을 확인하고, 그래프를 그립니다.

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

각 단계는 패키지 하나를 한 번 부르는 호출이고, 그 사이를 오가는 값은 평범한 `Vector<f64>`입니다. 변환 계층도 없고, CSV를 파싱할 때 말고는 복사도 없습니다.

## 밀집 선형대수와 희소 선형대수

`linalg`는 `f64`, `f32`와 그 복소 형태의 `Matrix<T>`를 다룹니다. 분해는 한 번 만들어 여러 번 푸는 타입입니다. `ops` 기능(기본으로 켜짐)을 쓰면 `@`가 행렬 곱, `\`가 풀이, `†`가 수반 행렬입니다.

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

0이 아닌 원소가 적은 큰 연립방정식은 `sparse`가 맡습니다. 삼중항이나 대각선으로 행렬을 만들고 직접법이나 반복법으로 풉니다. 반복 솔버는 전처리기와 어떤 `Operator`든 받으며, 행렬 없이 정의한 연산자도 됩니다.

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

## 도함수

`autodiff`는 이 패키지의 수 타입으로 쓴 함수를 미분합니다. 함수를 `Smooth`에 대해 제네릭하게 한 번 쓰고, 부를 때의 타입으로 방식을 고릅니다. 입력이 적으면 `Dual`(순방향, 입력 하나에 한 번씩), 많으면 `Var`(역방향, 기울기 전체를 역방향 한 번으로), 정확한 2계 도함수는 `HyperDual`, 임의 차수의 테일러 계수는 `Jet`입니다.

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

`∂`와 `∇` 표기(`autodiff::ops`에 있고 `interpolate::ops`와 `optimize::ops`도 다시 내보냄)는 모든 패키지에서 한 가지 규칙을 따릅니다. 도함수는 준 것과 같은 형태로 돌아오고, 수로 돌아오지 않습니다. 함수의 `∂(f)`는 함수이므로 한 점에서의 값은 `∂(f)(x)`입니다. 스플라인의 `∂(s)`는 스플라인이고, 표본 값의 `∇(y)`는 표본 값입니다.

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

## 신호, 곡선, 방정식

나머지도 같은 모양입니다. 배열을 받는 평범한 함수, 이름 붙은 인자로 주는 옵션, 무엇을 바꿔야 하는지 말해 주는 에러입니다.

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

`special`은 수리 물리의 함수들과, core의 `Complex`에 없는 복소 인자의 초등 함수를 더합니다. 분지 절단은 C99와 Python `cmath`를 따릅니다.

```nabi norun
use special::Elementary;

let z = 1.0 + 2.0i;
println(f"{special::gamma(4.5)} {special::jv(0.5, 2.0)} {special::erf(0.5)}");
println(f"{z.sin()} {z.atanh()} {z.cbrt()}");
```

`units`는 컴파일할 때 차원을 검사합니다. 길이에 시간을 더하면 빌드되지 않습니다.

```nabi norun
use units::kinds::{Length, Time, Velocity};
use units::symbols::{h, km};

fn speed(d: Length, t: Time) -> Velocity { d / t }

println(speed(42.195km, 2h) → km / h);                        // 21.0975 km/h
```

## 데이터 파일과 쿼리

`table`은 CSV와 JSON Lines를 읽고 쓰며, 데이터 세계의 다른 도구들이 쓰는 열 기반 형식인 Arrow IPC(파일과 스트림)와 Parquet(기본은 Snappy 압축, zstd 페이지도 읽음)도 읽고 씁니다. `Table`의 메서드는 바로 실행됩니다. 테이블에 `lazy()`를 부르거나 파일에 `scan_parquet`, `scan_arrow`, `scan_csv`를 부르면 대신 `Query`를 만듭니다. `collect`는 필터를 스캔 안으로 밀어 넣고 쿼리가 쓰는 열만 읽은 뒤 실행합니다. 통계로 보아 필터에 걸릴 행이 없는 Parquet 행 그룹은 읽지 않습니다.

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

`table::read_arrow`와 `write_arrow`는 다른 Arrow 도구(pyarrow, Polars, DuckDB)에 같은 타입의 같은 열을 줍니다. 여기서 쓴 Parquet 파일은 Arrow 스키마를 함께 저장하므로 범주형 열은 범주형으로 돌아옵니다.

## NumPy, SciPy, pandas에서 오는 경우

| Python | Nabi |
|--------|------|
| `np.array([[1, 2], [3, 4]])` | `let m: Matrix<f64> = [[1, 2], [3, 4]];` |
| `np.linspace(0, 1, 5)`, `np.arange` | `array::linspace(0.0, 1.0, 5)`, `array::arange(0.0, 1.0, 0.25)` |
| `a.sum(axis=0)`, `a.argmax()`, `np.allclose(a, b)` | `a.sum_along(0)`, `a.argmax()`, `a.all_close(b)` |
| `a @ b`, `np.linalg.solve(a, b)` | `a @ b`, `linalg::solve(a, b)` 또는 `a \ b` |
| `np.linalg.eigh`, `svd`, `cholesky`, `lstsq` | `linalg::eigh`, `svd`, `cholesky`, `lstsq` |
| `scipy.sparse.csr_matrix`, `spsolve`, `cg` | `sparse::Csr`, `sparse::lu(a).solve(b)`, `sparse::cg` |
| `jax.grad(f)(x)` | `autodiff::grad(f, x)` 또는 `∇(f)(x)` |
| `scipy.stats.norm(0, 1).cdf(x)` | `stats::Normal::new(0.0, 1.0).cdf(x)` |
| `np.mean`, `np.std(x, ddof=1)`, `np.quantile` | `stats::mean`, `stats::std(x, ddof: 1)`, `stats::quantile` |
| `scipy.stats.ttest_ind`, `linregress` | `stats::ttest_ind`, `stats::ols` |
| `np.fft.fft`, `rfft`, `scipy.signal.welch` | `signal::fft`, `signal::rfft`, `signal::welch` |
| `scipy.signal.butter(..., output="sos")`, `sosfiltfilt` | `signal::filter::butter`, `sosfiltfilt` |
| `scipy.interpolate.CubicSpline`, `make_interp_spline` | `interpolate::cubic_spline`, `make_interp_spline` |
| `scipy.integrate.quad`, `solve_ivp` | `interpolate::quad`(또는 `∫(f, a..b)`), `ode::solve_ivp` |
| `scipy.optimize.minimize`, `curve_fit`, `brentq` | `optimize::minimize`, `curve_fit`, `brentq` |
| `scipy.special.gamma`, `jv`, `erf` | `special::gamma`, `jv`, `erf` |
| `pd.read_csv`, `df[df.x > 2]`, `df.groupby("k").agg(...)` | `table::read_csv`, `t.filter(col("x") .> 2)`, `t.group_by(["k"]).agg([...])` |
| `pd.read_parquet`, `pl.scan_parquet(...).filter(...).collect()` | `table::read_parquet`, `table::scan_parquet(..).filter(..).collect()` |
| `plt.plot(x, y); plt.savefig("a.svg")` | `plot::axes().line(x, y).save("a.svg")` |

## 속도

패키지는 Nabi로만 작성되었고 입력이 크면 모든 코어를 씁니다. 밀집 행렬 곱은 Apple M 계열 코어 하나에서 약 30 GFLOP/s, 모든 코어에서 100 GFLOP/s를 넘고, FFT는 길이와 관계없이 `O(n log n)`이며, 희소 행렬-벡터 곱은 초당 수억 개의 0 아닌 원소를 읽습니다. 각 패키지의 README에 있는 "Speed" 절에 그 패키지의 수치와 NumPy, SciPy, 참조 라이브러리와의 비교가 있습니다. 빠르게 유지하는 습관은 두 가지입니다. 리스트에 루프로 `push`하지 말고 `Array::from_fn`이나 점 연산자로 배열을 만들고, 호출 사이에는 변환을 오가지 말고 `Vector<f64>`를 그대로 넘기세요.
