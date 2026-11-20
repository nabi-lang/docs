# 3. 통계

`stats`는 통계학 첫 강의에서 배우는 것과 그다음의 상당 부분을 다룹니다. 요약, 확률 분포, 무작위 표본, 가설 검정, 회귀, 부트스트랩입니다. 함수마다 어떤 수열이든 받고(`Vector<f64>`, `List`, 고정 배열, 범위), 결과는 이름으로 필드를 읽는 평범한 구조체입니다.

## 요약

`mean`, `median`, `var`, `std`, `quantile` 등은 데이터를 받아 수 하나를 돌려줍니다. 기술 통계는 던지지 않습니다. 입력이 비어 있으면 NumPy처럼 NaN을 줍니다. `stats::nan::`에는 NaN 값을 건너뛰는 같은 함수들이 있습니다.

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

`quantile`은 기본적으로 NumPy처럼 보간합니다. 분야에서 다른 정의를 쓴다면 `method:` 인자로 Hyndman과 Fan의 정의 가운데 아무것이나 고를 수 있습니다.

## 분포

분포는 값입니다. `Normal::new(100.0, 15.0)`은 평균 100, 표준편차 15인 정규 분포입니다. 연속 분포에는 `pdf`, `cdf`, `sf`(1에서 cdf를 뺀 값, 꼬리에서도 정확함), `ppf`(cdf의 역함수), `interval`이 있고, 이산 분포에는 `pdf` 대신 `pmf`가 있습니다.

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

## 무작위 표본

`Sampler`는 시드를 준 난수 생성기입니다. 시드가 같으면 어느 컴퓨터에서나 같은 값이 나오므로 시뮬레이션을 되풀이할 수 있습니다. `d.samples(ref g, n)`은 `d`에서 값 `n`개를 뽑습니다.

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

`fit`은 최대 가능도로 분포의 모수를 추정합니다. 이 패키지의 분포 대부분에 있습니다.

## 가설 검정

검정은 통계량과 p-값을 돌려줍니다. t-검정은 추정값과 그 표준 오차도 돌려주므로 신뢰 구간은 호출 한 번(`ci`)이면 됩니다. 여기서는 약물 시험입니다. 투약군의 혈압이 대조군보다 더 내려갔을까요?

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

이 패키지에는 대응 표본과 단일 표본 t-검정, 적합도와 분할표의 카이제곱 검정, Fisher 정확 검정, Kolmogorov-Smirnov, Wilcoxon, 일원 분산 분석(`f_oneway`), 상관 검정도 있습니다. 모두 단측 검정을 위한 `alternative:`를 받습니다.

## 회귀

`stats::ols`는 최소제곱으로 `y = b0 + b1 x1 + ...`를 맞추고 회귀 표에 나오는 것을 알려 줍니다. 계수, 표준 오차, t 통계량, p-값, R²입니다. `x`의 행 하나가 관측 하나, 열 하나가 예측 변수 하나이며, 절편은 알아서 더해집니다.

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

`stats::logistic`은 예/아니오 결과에 로지스틱 회귀를, `stats::polyfit`은 한 변수의 다항식을 맞춥니다.

## 부트스트랩

어떤 공식으로도 통계량의 불확실성을 구할 수 없을 때, 부트스트랩은 데이터를 다시 뽑은 많은 표본에서 통계량을 다시 계산해 그것을 추정합니다. `stats::bootstrap`은 데이터와 통계량 함수를 받아 추정값, 표준 오차, 신뢰 구간(기본은 BCa)을 돌려줍니다.

```nabi
use core::array::Vector;

let incomes = [21.0, 23.5, 25.0, 26.0, 28.5, 30.0, 31.0, 34.0, 38.0, 45.0, 60.0, 120.0];
let b = try stats::bootstrap(incomes, |v: Vector<f64>| -> f64 { stats::median(v) }, resamples: 2000, seed: 1);
println(f"median {b.estimate}, se {b.se:.2}, 95% interval {b.ci}");
```

```output
median 30.5, se 4.09, 95% interval [25, 38]
```

## 표기법

`ops` 기능(기본으로 켜짐)이 있으면 일부는 통계 교과서처럼 읽힙니다. `95pct`는 수 0.95, `m ± h`는 구간, `g ~ d`는 `d`에서 하나 뽑기입니다.

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

## 다음으로

- [신호와 스펙트럼](04-signals.html): FFT와 필터.
- [`stats` API](https://packages.nabi-lang.org/packages/stats): 모든 분포, 검정, 옵션.
