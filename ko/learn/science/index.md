# 과학 계산과 데이터 분석

Nabi의 과학 계산 패키지를 분야별 튜토리얼 하나씩으로 따라가는 길잡이입니다. 튜토리얼마다 작은 문제에서 시작해 차근차근 쌓아 가며, 이 페이지들의 모든 코드 블록은 사이트를 빌드할 때 실제 패키지로 컴파일하고 실행해 그 아래에 출력을 보여 줍니다.

`Vector<f64>`, `Matrix<f64>`, 점 연산자(`.+`, `.*`)가 처음이라면 먼저 [배열과 수치 계산](../21-arrays-and-numerics.html)을 읽으세요. 아래의 모든 패키지가 이 타입들을 받고 돌려줍니다. 전체 개관은 [Nabi로 하는 과학 계산](../22-science.html)에 있습니다. 패키지마다 무엇을 하는지, 입력 데이터에 대해 모든 패키지가 공유하는 규칙, NumPy·SciPy·pandas 대응표를 다룹니다.

## 준비할 것 없음

과학 계산 패키지는 공식 패키지라서 툴체인과 함께 설치됩니다. `table::read_csv(..)`나 `use stats::Normal;`을 쓰는 스크립트는 처음 실행할 때 패키지가 추가되며, `nabi add`도 내려받기도 필요 없습니다.

```text
$ nabi run tea.nabi
       Added table ^0.1 (official)
       Added stats ^0.1 (official)
```

패키지 안에서도 똑같이 `nabi.nabion`에 추가되고, `nabi.lock`이 버전을 고정합니다. 노트북(`nabi jupyter`)에서는 패키지를 쓰는 셀이 그 패키지를 노트북의 환경에 추가합니다.

## 튜토리얼

| 튜토리얼 | 패키지 | 하는 일 |
|----------|--------|---------|
| [첫 데이터 분석](01-data-analysis.html) | `table`, `stats`, `plot` | CSV 파일 읽기, 정리, 그룹 묶기, 차이 검정, 그래프 |
| [밀집 행렬과 희소 행렬의 선형대수](02-linear-algebra.html) | `linalg`, `sparse` | 연립방정식, 최소제곱, 고윳값, 미지수 10 000개짜리 희소 연립방정식 |
| [통계](03-statistics.html) | `stats` | 분포, 무작위 표본, 검정, 회귀, 부트스트랩 |
| [신호와 스펙트럼](04-signals.html) | `signal` | FFT, 잡음 속 파워 스펙트럼, 필터, 피크, 합성곱 |
| [피팅과 최적화](05-fitting.html) | `optimize` | 근, 최솟값, 곡선 피팅, 선형 계획법 |
| [미분방정식](06-ode.html) | `ode` | 초깃값 문제, 이벤트, 강성 시스템, 경곗값 문제 |
| [도함수](07-autodiff.html) | `autodiff` | 직접 쓴 함수의 정확한 기울기, 야코비 행렬, 테일러 급수 |

여기서 다루지 않는 패키지는 [packages.nabi-lang.org](https://packages.nabi-lang.org)에 저마다 안내가 있습니다. `interpolate`(스플라인과 수치 적분), `special`(특수 함수), `units`(컴파일할 때 검사하는 물리량)입니다.

과학 계산 밖의 짧은 풀이는 [Rosetta](../rosetta/index.html) 페이지를 보세요. 흔한 작업을 Nabi 몇 줄로 풀어 둡니다.
