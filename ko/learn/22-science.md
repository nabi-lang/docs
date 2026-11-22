# 22. Nabi로 하는 과학 계산

Nabi의 수치 계산 핵심은 표준 라이브러리에 있습니다. `Array<T, N>`, 점 연산자, 집계, `Complex`, 난수가 그렇습니다([21장](21-arrays-and-numerics.html)). 그 위에 과학자가 쌓는 선형대수, 통계, 신호, 데이터 프레임, 그래프는 패키지에 있습니다. 분야마다 패키지가 하나씩 있고 버전도 따로 매기며, 모두 C나 Fortran 없이 Nabi로 작성되어 WebAssembly를 포함한 Nabi의 모든 타깃에서 빌드됩니다.

이 장은 패키지들의 지도를 보이고, 입력 데이터에 대해 모든 패키지가 공유하는 규칙 하나를 설명하고, CSV 파일에서 그래프까지 작은 문제 하나를 풀어 보고, 패키지마다 튜토리얼을 안내한 뒤, NumPy, SciPy, pandas에서 오는 독자를 위한 대응표로 끝납니다.

## 패키지

| 패키지 | 하는 일 | Python의 대응 |
|--------|---------|---------------|
| [`linalg`](https://docs.nabi-lang.org/packages/linalg/latest) | 밀집 행렬: 곱, LU, Cholesky, QR, SVD, 고윳값, 연립방정식, `expm` | `numpy.linalg`, `scipy.linalg` |
| [`sparse`](https://docs.nabi-lang.org/packages/sparse/latest) | CSR / CSC / COO 행렬, CG, GMRES, BiCGSTAB, 희소 Cholesky와 LU | `scipy.sparse`, `scipy.sparse.linalg` |
| [`spatial`](https://docs.nabi-lang.org/packages/spatial/latest) | 게임과 그래픽스용 고정 크기 벡터, 쿼터니언, 행렬, 변환, 도형; `#[repr(C)]`, SIMD `Mat4` | `pyglm`, `scipy.spatial.transform` |
| [`autodiff`](https://docs.nabi-lang.org/packages/autodiff/latest) | 직접 쓴 함수의 도함수: 이중수, 역방향 테이프, 헤세 행렬, 테일러 급수 | `jax.grad`, `autograd` |
| [`stats`](https://docs.nabi-lang.org/packages/stats/latest) | 기술 통계, 19가지 분포, 검정, 회귀 | `scipy.stats`, `statsmodels`(기본) |
| [`signal`](https://docs.nabi-lang.org/packages/signal/latest) | 임의 길이 FFT, 창 함수, FIR / IIR 필터, 스펙트럼, 피크 | `numpy.fft`, `scipy.signal` |
| [`interpolate`](https://docs.nabi-lang.org/packages/interpolate/latest) | 스플라인, 다항식, 격자, 흩어진 점, 수치 적분 | `scipy.interpolate`, `scipy.integrate.quad` |
| [`optimize`](https://docs.nabi-lang.org/packages/optimize/latest) | 최소화, 최소제곱, 근 찾기, 선형 계획법 | `scipy.optimize` |
| [`ode`](https://docs.nabi-lang.org/packages/ode/latest) | 초깃값 문제와 경곗값 문제, 강성 솔버, 심플렉틱 적분기 | `scipy.integrate.solve_ivp` |
| [`special`](https://docs.nabi-lang.org/packages/special/latest) | 감마, 베셀, erf, 제타, 타원 함수, 복소 초등 함수 | `scipy.special` |
| [`table`](https://docs.nabi-lang.org/packages/table/latest) | 열 기반 데이터 프레임, CSV, JSONL, Arrow와 Parquet 파일, 지연 쿼리 | `pandas`, `polars`, `pyarrow` |
| [`units`](https://docs.nabi-lang.org/packages/units/latest) | 단위가 붙은 양, 컴파일할 때 검사 | `pint` |
| [`plot`](https://docs.nabi-lang.org/packages/plot/latest) | SVG, PNG, 터미널 그래프, 노트북 표시 | `matplotlib` |

쓰는 패키지는 `nabi add`로 `nabi.nabion`에 추가합니다(파일 하나짜리 스크립트라면 `nabi add <패키지> file.nabi`).

```sh
nabi add linalg
nabi add table
```

튜토리얼과 API가 담긴 각 패키지의 문서는 [패키지 문서](https://docs.nabi-lang.org/packages)에 있습니다. 표에서 패키지 이름을 따라가세요. 이 페이지의 코드는 이 패키지들이 필요하므로, 이 가이드의 다른 코드와 달리 사이트를 빌드할 때 실행하지 않습니다. 모든 블록은 패키지의 현재 버전으로 확인했습니다.

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

## 튜토리얼

각 패키지의 문서는 튜토리얼로 시작합니다. 작은 문제 하나를 단계별로 풀고, 그 코드는 패키지의 테스트가 실행합니다. 튜토리얼은 영어로 되어 있습니다.

| 튜토리얼 | 패키지 | 하는 일 |
|----------|--------|---------|
| [처음 해 보는 데이터 분석](https://docs.nabi-lang.org/packages/table/latest) | `table`, `stats`, `plot` | CSV 파일 읽기, 정리, 그룹 묶기, 차이 검정, 그래프 |
| [선형대수](https://docs.nabi-lang.org/packages/linalg/latest) | `linalg` | 연립방정식, 최소제곱, 고윳값 |
| [희소 시스템](https://docs.nabi-lang.org/packages/sparse/latest) | `sparse` | 미지수 10 000개짜리 희소 시스템, 반복법과 직접법 |
| [통계](https://docs.nabi-lang.org/packages/stats/latest) | `stats` | 분포, 난수 표본, 검정, 회귀, 부트스트랩 |
| [신호와 스펙트럼](https://docs.nabi-lang.org/packages/signal/latest) | `signal` | FFT, 잡음 속 파워 스펙트럼, 필터, 피크, 합성곱 |
| [피팅과 최적화](https://docs.nabi-lang.org/packages/optimize/latest) | `optimize` | 근, 최솟값, 곡선 피팅, 선형 계획법 |
| [미분방정식](https://docs.nabi-lang.org/packages/ode/latest) | `ode` | 초깃값 문제, 이벤트, 강성 시스템, 경곗값 문제 |
| [도함수](https://docs.nabi-lang.org/packages/autodiff/latest) | `autodiff` | 직접 만든 함수의 정확한 기울기, 야코비안, 테일러 급수 |

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
