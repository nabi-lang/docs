# 22. 用 Nabi 做科學計算

Nabi 的數值核心在標準函式庫中：`Array<T, N>`、點運算子、歸約、`Complex` 與亂數（[第 21 章](21-arrays-and-numerics.html)）。科學工作者在這之上建構的一切，包括線性代數、統計、訊號、資料框（data frame）、繪圖，都放在套件裡：每個領域一個套件，各自獨立管理版本，全部以 Nabi 撰寫，底下沒有 C 或 Fortran，因此能為 Nabi 支援的每個目標建置，WebAssembly 也包括在內。

本章介紹這些套件的全貌，說明它們對輸入資料共同遵守的一條規則，從 CSV 檔案到繪圖完整解一個小問題，指引各套件的教學，最後附上一張給 NumPy、SciPy 與 pandas 使用者的對照表。

## 套件

| 套件 | 功能 | 對應的 Python 套件 |
|---------|--------------|--------------------|
| [`linalg`](https://docs.nabi-lang.org/packages/linalg/latest) | 稠密矩陣：乘積、LU、Cholesky、QR、SVD、特徵值、求解器、`expm` | `numpy.linalg`、`scipy.linalg` |
| [`sparse`](https://docs.nabi-lang.org/packages/sparse/latest) | CSR / CSC / COO 矩陣、CG、GMRES、BiCGSTAB、稀疏 Cholesky 與 LU | `scipy.sparse`、`scipy.sparse.linalg` |
| [`spatial`](https://docs.nabi-lang.org/packages/spatial/latest) | 供遊戲與圖學使用的小型固定長度向量、四元數、矩陣、變換與幾何形狀；`#[repr(C)]`、SIMD `Mat4` | `pyglm`、`scipy.spatial.transform` |
| [`autodiff`](https://docs.nabi-lang.org/packages/autodiff/latest) | 你自己的函式的導數：對偶數、反向模式記錄帶（tape）、黑塞矩陣、泰勒級數 | `jax.grad`、`autograd` |
| [`stats`](https://docs.nabi-lang.org/packages/stats/latest) | 敘述統計、19 種機率分布、檢定、迴歸 | `scipy.stats`、`statsmodels`（基礎部分） |
| [`signal`](https://docs.nabi-lang.org/packages/signal/latest) | 任意長度的 FFT、窗函數、FIR / IIR 濾波器、頻譜、峰值 | `numpy.fft`、`scipy.signal` |
| [`interpolate`](https://docs.nabi-lang.org/packages/interpolate/latest) | 樣條、多項式、網格、散佈資料、數值積分 | `scipy.interpolate`、`scipy.integrate.quad` |
| [`optimize`](https://docs.nabi-lang.org/packages/optimize/latest) | 最小化、最小平方法、求根、線性規劃 | `scipy.optimize` |
| [`ode`](https://docs.nabi-lang.org/packages/ode/latest) | 初值與邊界值問題、剛性求解器、辛積分器 | `scipy.integrate.solve_ivp` |
| [`special`](https://docs.nabi-lang.org/packages/special/latest) | gamma、Bessel、erf、zeta、橢圓函數；複數初等函數 | `scipy.special` |
| [`table`](https://docs.nabi-lang.org/packages/table/latest) | 欄式資料框、CSV、JSONL、Arrow 與 Parquet 檔案、惰性查詢 | `pandas`、`polars`、`pyarrow` |
| [`units`](https://docs.nabi-lang.org/packages/units/latest) | 帶單位的物理量，在編譯時檢查 | `pint` |
| [`plot`](https://docs.nabi-lang.org/packages/plot/latest) | SVG、PNG 與終端機繪圖，筆記本顯示 | `matplotlib` |

用 `nabi add` 把要用的套件加入 `nabi.nabion`（單檔指令碼則用 `nabi add <package> file.nabi`）：

```sh
nabi add linalg
nabi add table
```

各套件的文件，包括一份教學和 API，都在[套件文件](https://docs.nabi-lang.org/packages)底下：依表中的套件名稱點進去即可。本頁的程式碼需要這些套件，所以和本指南的其他部分不同，網站建置時不會執行它們；每個區塊都已用這些套件目前的版本檢查過。

## 統一的入口：類陣列輸入

每個套件都以相同的方式接收一串數字。需要 `f64` 向量的函式，接受 `linalg::AsVector<f64>` 涵蓋的任何值：`f64`、`f32` 或整數的 `Vector`、`List`、固定長度陣列，以及整數範圍。`Vector<f64>` 或 `List<f64>` 會就地讀取；其他值則轉換一次。

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

`stats` 也能讀取任意維度的陣列（依列優先順序），`plot` 則接受同樣的形式，另外還接受代表網格的巢狀串列。你自己的函式也可以接收同一個 trait：

```nabi norun
use linalg::AsVector;
use core::array::Vector;

fn rms<X: AsVector<f64>>(x: X) -> f64 {
    let v: Vector<f64> = x.as_vector();
    (v .* v).mean().sqrt()
}

println(rms([3, 4]));        // 3.5355339059327378
```

混合型字面值的第一個元素要寫成想要的元素型別：`[0.0, 0.5, 1]` 是浮點數串列，而 `[0, 0.5, 1]` 一開始就是整數串列，其中的 `0.5` 會是錯誤。

## 完整範例

一杯茶從 90 度開始冷卻。讀數存在一個 CSV 檔案中；牛頓冷卻定律指出，溫度會朝室溫下降，下降速率與兩者的差成正比，即 `T' = k (room - T)`。我們以資料擬合 `room` 與 `k`，把這條定律當作 ODE 求解，檢查擬合結果並繪圖。

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

每個步驟都只是對一個套件的一次呼叫，步驟之間傳遞的值都是單純的 `Vector<f64>`：沒有轉換層，除了剖析 CSV 之外也沒有任何複製。

## 教學

每個套件的文件都以一份教學開頭：一步步解一個小問題，其中的程式碼由該套件的測試執行。

| 教學 | 套件 | 內容 |
|----------|---------|--------------|
| [第一次資料分析](https://docs.nabi-lang.org/packages/table/latest) | `table`、`stats`、`plot` | 讀取 CSV 檔案、清理、分組、檢定差異、繪圖 |
| [線性代數](https://docs.nabi-lang.org/packages/linalg/latest) | `linalg` | 解線性方程組、最小平方法、特徵值 |
| [稀疏系統](https://docs.nabi-lang.org/packages/sparse/latest) | `sparse` | 含 10 000 個未知數的稀疏系統、迭代與直接求解器 |
| [統計](https://docs.nabi-lang.org/packages/stats/latest) | `stats` | 機率分布、隨機樣本、檢定、迴歸、自助法（bootstrap） |
| [訊號與頻譜](https://docs.nabi-lang.org/packages/signal/latest) | `signal` | FFT、雜訊中的功率頻譜、濾波器、峰值、摺積 |
| [擬合與最佳化](https://docs.nabi-lang.org/packages/optimize/latest) | `optimize` | 求根、求極小值、曲線擬合、線性規劃 |
| [微分方程](https://docs.nabi-lang.org/packages/ode/latest) | `ode` | 初值問題、事件、剛性系統、邊界值問題 |
| [導數](https://docs.nabi-lang.org/packages/autodiff/latest) | `autodiff` | 你自己的函式的精確梯度、Jacobian 矩陣與泰勒級數 |

## 從 NumPy、SciPy 與 pandas 轉過來

| Python | Nabi |
|--------|------|
| `np.array([[1, 2], [3, 4]])` | `let m: Matrix<f64> = [[1, 2], [3, 4]];` |
| `np.linspace(0, 1, 5)`、`np.arange` | `array::linspace(0.0, 1.0, 5)`、`array::arange(0.0, 1.0, 0.25)` |
| `a.sum(axis=0)`、`a.argmax()`、`np.allclose(a, b)` | `a.sum_along(0)`、`a.argmax()`、`a.all_close(b)` |
| `a @ b`、`np.linalg.solve(a, b)` | `a @ b`、`linalg::solve(a, b)` 或 `a \ b` |
| `np.linalg.eigh`、`svd`、`cholesky`、`lstsq` | `linalg::eigh`、`svd`、`cholesky`、`lstsq` |
| `scipy.sparse.csr_matrix`、`spsolve`、`cg` | `sparse::Csr`、`sparse::lu(a).solve(b)`、`sparse::cg` |
| `jax.grad(f)(x)` | `autodiff::grad(f, x)` 或 `∇(f)(x)` |
| `scipy.stats.norm(0, 1).cdf(x)` | `stats::Normal::new(0.0, 1.0).cdf(x)` |
| `np.mean`、`np.std(x, ddof=1)`、`np.quantile` | `stats::mean`、`stats::std(x, ddof: 1)`、`stats::quantile` |
| `scipy.stats.ttest_ind`、`linregress` | `stats::ttest_ind`、`stats::ols` |
| `np.fft.fft`、`rfft`、`scipy.signal.welch` | `signal::fft`、`signal::rfft`、`signal::welch` |
| `scipy.signal.butter(..., output="sos")`、`sosfiltfilt` | `signal::filter::butter`、`sosfiltfilt` |
| `scipy.interpolate.CubicSpline`、`make_interp_spline` | `interpolate::cubic_spline`、`make_interp_spline` |
| `scipy.integrate.quad`、`solve_ivp` | `interpolate::quad`（或 `∫(f, a..b)`）、`ode::solve_ivp` |
| `scipy.optimize.minimize`、`curve_fit`、`brentq` | `optimize::minimize`、`curve_fit`、`brentq` |
| `scipy.special.gamma`、`jv`、`erf` | `special::gamma`、`jv`、`erf` |
| `pd.read_csv`、`df[df.x > 2]`、`df.groupby("k").agg(...)` | `table::read_csv`、`t.filter(col("x") .> 2)`、`t.group_by(["k"]).agg([...])` |
| `pd.read_parquet`、`pl.scan_parquet(...).filter(...).collect()` | `table::read_parquet`、`table::scan_parquet(..).filter(..).collect()` |
| `plt.plot(x, y); plt.savefig("a.svg")` | `plot::axes().line(x, y).save("a.svg")` |

## 速度

這些套件是純 Nabi 實作，輸入很大時會用上每一個核心：稠密矩陣乘積在單一 Apple M 系列核心上約為 30 GFLOP/s，使用全部核心時超過 100 GFLOP/s；任意長度的 FFT 都是 `O(n log n)`；稀疏矩陣與向量的乘積每秒可讀取數億個非零元素。每個套件的 README 都有一節「Speed」，列出它的效能數字，以及與 NumPy、SciPy 和參考函式庫的比較。有兩個習慣能讓程式碼保持快速：用 `Array::from_fn` 或點運算子建立陣列，而不是在迴圈中逐一 push 到串列；在呼叫之間傳遞 `Vector<f64>`，而不是來回轉換。
