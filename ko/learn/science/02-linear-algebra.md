# 2. 밀집 행렬과 희소 행렬의 선형대수

수치 계산의 대부분은 결국 행렬로 귀결됩니다. 연립방정식, 최소제곱 피팅, 구조물의 진동 모드가 그렇습니다. `linalg`는 밀집 행렬을, `sparse`는 대부분이 0인 큰 행렬을 다룹니다. 둘 다 core의 `Matrix<f64>`와 `Vector<f64>`([21장](../21-arrays-and-numerics.html)) 위에서 동작하므로, 결과를 다른 모든 곳에 그대로 넘길 수 있습니다.

## 행렬과 벡터

`Matrix<f64>`는 2차원 배열입니다. 타입을 밝히고 중첩 리스트로 씁니다. linalg의 `ops`(기본으로 켜짐)가 있으면 `@`는 행렬 곱, `\`는 풀기, `†`는 전치(복소 행렬이면 켤레 전치)입니다. 점 연산자(`.+`, `.*`)는 21장에서처럼 원소별로 동작합니다.

```nabi
use core::array::{Matrix, Vector};
use linalg::ops::*;

let a: Matrix<f64> = [[2, 1], [1, 3]];
let v: Vector<f64> = [1, 2];
println(a @ v);
println(a† @ a);
println(a .* 10.0);
println(linalg::eye::<f64>(2));
```

```output
[4.0, 7.0]
[[5.0, 5.0], [5.0, 10.0]]
[[20.0, 10.0], [10.0, 30.0]]
[[1.0, 0.0], [0.0, 1.0]]
```

`a @ v`는 모양이 맞지 않으면 `.+`처럼 panic합니다. `linalg::matmul(a, b)`는 같은 곱을 호출로 하되, panic 대신 `LinalgError`를 던집니다.

## 연립방정식 풀기

미지수 셋, 방정식 셋: `a x = b`. `linalg::solve`는 `a`를 분해(부분 피벗팅 LU)해서 풉니다. `a \ b`는 같은 일을 연산자로 하며, `get`으로 꺼내는 결과를 돌려줍니다. 잔차 `a x - b`를 보면 답이 얼마나 정확한지 알 수 있습니다.

```nabi
use core::array::{Matrix, Vector};
use linalg::ops::*;

let a: Matrix<f64> = [[4, 1, 0], [1, 3, 1], [0, 1, 2]];
let b: Vector<f64> = [1, 2, 3];

let x = try linalg::solve(a, b);
println(f"x = {x:.4}");
println(f"residual {linalg::norm(a @ x .- b) < 1e-12}");

let same = try (a \ b).get();
println(same.all_close(x));
```

```output
x = [0.2222, 0.1111, 1.4444]
residual true
true
```

같은 행렬로 여러 번 풀려면 한 번만 분해하세요. `linalg::lu(a)`(아무 정방 행렬)나 `linalg::cholesky(a)`(대칭 양의 정부호, 두 배 빠름)는 `solve`, `det`, `inv`를 갖춘 분해를 돌려줍니다.

```nabi
use core::array::{Matrix, Vector};

let a: Matrix<f64> = [[4, 1, 0], [1, 3, 1], [0, 1, 2]];
let c = try linalg::cholesky(a);
let rhs: List<Vector<f64>> = [[1, 0, 0], [0, 1, 0]];
for b in rhs {
    println(f"{try c.solve(b):.4}");
}
println(f"det {(try linalg::lu(a)).det():.1}");
```

```output
[0.2778, -0.1111, 0.0556]
[-0.1111, 0.4444, -0.2222]
det 18.0
```

## 최소제곱으로 직선 맞추기

방정식이 미지수보다 많으면 정확한 해가 없으므로, 최소제곱은 잔차 제곱의 합이 가장 작은 해를 찾습니다. 직선 `y = c0 + c1 t`라면 데이터 점마다 행 `[1, t]`가 하나씩입니다. `linalg::lstsq`는 QR 분해로 풀기 때문에, 정규 방정식 `aᵀa x = aᵀy`가 자릿수를 잃는 곳에서도 정확도를 지킵니다.

```nabi
use core::array::{Matrix, Vector};

let t: Vector<f64> = [0, 1, 2, 3, 4];
let y: Vector<f64> = [1.1, 2.9, 5.2, 7.1, 8.8];
let a: Matrix<f64> = Matrix::from_fn([5, 2], |i: [usize; 2]| -> f64 { if i[1] == 0 { 1.0 } else { t[i[0]] } });

let c = try linalg::lstsq(a, y);
println(f"y = {c[0]:.3} + {c[1]:.3} t");
```

```output
y = 1.100 + 1.960 t
```

`stats::ols`는 같은 피팅을 하면서 계수마다 표준 오차와 p-값을 더해 줍니다([통계](03-statistics.html#회귀)).

## 고윳값과 SVD

`linalg::eigh`는 대칭 행렬을 받아 고윳값을 오름차순으로, 정규 직교 고유벡터를 열로 돌려줍니다. 여기서는 용수철로 이은 질량 셋의 고유 진동수를 구합니다. 강성 행렬의 고윳값이 진동수의 제곱입니다.

```nabi
use core::array::Matrix;

let k: Matrix<f64> = [[2, -1, 0], [-1, 2, -1], [0, -1, 2]];
let e = try linalg::eigh(k);
println(f"squared frequencies {e.values():.4}");
println(f"frequencies {e.values().map(|v: f64| -> f64 { v.sqrt() }):.4}");
```

```output
squared frequencies [0.5858, 2.0000, 3.4142]
frequencies [0.7654, 1.4142, 1.8478]
```

특잇값 분해는 어떤 행렬에도 됩니다. 특잇값을 보면 랭크와 행렬의 조건이 얼마나 좋은지 알 수 있습니다. 가장 큰 값과 가장 작은 값의 비가 조건수이며, `linalg::cond`가 이를 계산합니다.

```nabi
use core::array::Matrix;

let m: Matrix<f64> = [[1, 2], [2, 4.001]];
let s = try linalg::svd(m);
println(f"singular values {s.s():.5}");
println(f"condition number {(try linalg::cond(m)):.0}");
```

```output
singular values [5.00080, 0.00020]
condition number 25008
```

조건수가 2.5 x 10<sup>4</sup>쯤이면 풀이에서 `f64`가 지닌 열여섯 자리 가운데 네 자리 넘게 잃을 수 있다는 뜻입니다. 두 행이 거의 같은 방정식이기 때문입니다.

## 희소 행렬

행이 백만 개인 행렬은 밀집 배열로는 메모리에 들어가지 않지만, 그런 행렬은 대부분 거의 다 0입니다. 미지수 하나가 이웃 몇 개와만 이어져 있기 때문입니다. `sparse`는 0이 아닌 값만 저장합니다. `Coo`로 (행, 열, 값) 세 쌍에서 행렬을 만든 다음, 연산과 풀이를 위해 `Csr`로 바꿉니다.

```nabi
let c = sparse::Coo::new([4, 4]);
for i in 0..4 {
    try c.push(i, i, 4.0);
    if i + 1 < 4 {
        try c.push(i, i + 1, -1.0);
        try c.push(i + 1, i, -1.0);
    }
}
let a = c.to_csr();
println(f"{a.nnz()} nonzeros in a {a.shape():?} matrix");
println(try a.matvec([1, 1, 1, 1]));
```

```output
10 nonzeros in a [4, 4] matrix
[3.0, 2.0, 2.0, 3.0]
```

## 큰 희소 연립방정식 풀기

정사각형 판의 열 방정식을 100 x 100 격자로 이산화하면 미지수가 10 000개이고, 행마다 0이 아닌 값이 다섯 개인 행렬이 나옵니다. `sparse::diags`는 대각선들로 1차원 차분 행렬을 만들고, `kron`이 둘을 합쳐 2차원 행렬을 만듭니다. `sparse::cg`는 켤레 기울기법으로 반복해서 풀며, `ic0`(불완전 촐레스키 분해)가 반복 횟수를 줄입니다.

```nabi
use core::array::Vector;

let n = 100;
let t = try sparse::diags([[2.0], [-1.0], [-1.0]], [0, 1, -1], [n, n]);
let i = sparse::eye(n);
let a = sparse::kron(i, t) + sparse::kron(t, i);
let b = Vector::<f64>::ones([n * n]);

let plain = try sparse::cg(a, b);
let pre = try sparse::cg(a, b, pre: try sparse::ic0(a));
println(f"{a.nnz()} nonzeros; cg {plain.iterations} iterations, with ic0 {pre.iterations}");

let x = try (try sparse::cholesky(a)).solve(b);
println(f"direct and iterative agree: {linalg::norm(x .- pre.x) / linalg::norm(x) < 1e-6}");
println(f"hottest point {x.max() ?? 0.0:.2}");
```

```output
49600 nonzeros; cg 187 iterations, with ic0 79
direct and iterative agree: true
hottest point 751.34
```

반복 풀이기는 허용 오차(`rtol:`), 반복 한도, 그리고 어떤 `Operator`든 받습니다. 행렬을 저장하지 않고 곱하기만 하는 함수(`sparse::operator`)도 됩니다. `sparse::cholesky`와 `sparse::lu`는 분해 결과가 희소하게 남도록 행렬의 순서를 바꾸는 직접 풀이기입니다.

## 다음으로

- [통계](03-statistics.html): 분포, 검정, 회귀.
- [`linalg`](https://packages.nabi-lang.org/packages/linalg)와 [`sparse`](https://packages.nabi-lang.org/packages/sparse) API: QR, Schur, `expm`, 행렬 함수; GMRES, BiCGSTAB, 전처리기.
