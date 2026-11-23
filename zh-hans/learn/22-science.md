# 22. 用 Nabi 做科学计算

Nabi 的数值核心位于标准库中：`Array<T, N>`、点运算符、归约、`Complex` 和随机数（[第 21 章](21-arrays-and-numerics.html)）。科学计算工作者在此之上构建的一切，包括线性代数、统计、信号、数据框和绘图，都放在包中：每个领域一个包，各自独立地管理版本，全部用 Nabi 编写，底层没有 C 或 Fortran，因此能为 Nabi 支持的每个目标构建，包括 WebAssembly。

本章梳理这些包，介绍它们在输入数据上共用的一条规则，从 CSV 文件到绘图完整地解决一个小问题，给出每个包的教程，最后为从 NumPy、SciPy 和 pandas 转过来的读者提供一张对照表。

## 这些包

| 包 | 用途 | Python 中的对应物 |
|---------|--------------|--------------------|
| [`linalg`](https://docs.nabi-lang.org/packages/linalg/latest) | 稠密矩阵：乘积、LU、Cholesky、QR、SVD、特征值、求解器、`expm` | `numpy.linalg`, `scipy.linalg` |
| [`sparse`](https://docs.nabi-lang.org/packages/sparse/latest) | CSR / CSC / COO 矩阵、CG、GMRES、BiCGSTAB、稀疏 Cholesky 和 LU | `scipy.sparse`, `scipy.sparse.linalg` |
| [`spatial`](https://docs.nabi-lang.org/packages/spatial/latest) | 用于游戏和图形的小型定长向量、四元数、矩阵、变换和形状；`#[repr(C)]`、SIMD `Mat4` | `pyglm`, `scipy.spatial.transform` |
| [`autodiff`](https://docs.nabi-lang.org/packages/autodiff/latest) | 自定义函数的导数：对偶数、反向模式磁带、Hessian 矩阵、泰勒级数 | `jax.grad`, `autograd` |
| [`stats`](https://docs.nabi-lang.org/packages/stats/latest) | 描述统计、19 种分布、检验、回归 | `scipy.stats`, `statsmodels`（基础部分） |
| [`signal`](https://docs.nabi-lang.org/packages/signal/latest) | 任意长度的 FFT、窗函数、FIR / IIR 滤波器、频谱、峰值 | `numpy.fft`, `scipy.signal` |
| [`interpolate`](https://docs.nabi-lang.org/packages/interpolate/latest) | 样条、多项式、网格、散点数据、数值积分 | `scipy.interpolate`, `scipy.integrate.quad` |
| [`optimize`](https://docs.nabi-lang.org/packages/optimize/latest) | 最小化、最小二乘、求根、线性规划 | `scipy.optimize` |
| [`ode`](https://docs.nabi-lang.org/packages/ode/latest) | 初值和边值问题、刚性求解器、辛积分器 | `scipy.integrate.solve_ivp` |
| [`special`](https://docs.nabi-lang.org/packages/special/latest) | gamma、Bessel、erf、zeta、椭圆函数；复初等函数 | `scipy.special` |
| [`table`](https://docs.nabi-lang.org/packages/table/latest) | 列式数据框，CSV、JSONL、Arrow 和 Parquet 文件，惰性查询 | `pandas`, `polars`, `pyarrow` |
| [`units`](https://docs.nabi-lang.org/packages/units/latest) | 带单位的量，在编译时检查 | `pint` |
| [`plot`](https://docs.nabi-lang.org/packages/plot/latest) | SVG、PNG 和终端绘图，笔记本显示 | `matplotlib` |

用 `nabi add` 把要用的包添加到 `nabi.nabion`（单文件脚本则用 `nabi add <package> file.nabi`）：

```sh
nabi add linalg
nabi add table
```

每个包的文档，包括教程和 API，都在[包文档](https://docs.nabi-lang.org/packages)中：按表中的包名查找即可。本页的代码需要这些包，因此与本指南的其他部分不同，构建网站时不会运行它们；每个代码块都已用这些包的当前版本检查过。

## 统一的入口：类数组输入

每个包都以同样的方式接受数字序列。需要 `f64` 向量的函数接受 `linalg::AsVector<f64>` 涵盖的任何东西：元素为 `f64`、`f32` 或整数的 `Vector`、`List`、定长数组，以及整数区间。`Vector<f64>` 或 `List<f64>` 直接原地读取；其他类型只转换一次。

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

`stats` 还能读取任意秩的数组（按行优先顺序），`plot` 接受同样的形式，另外还接受用于网格的嵌套列表。你自己的函数也可以接受同一个 trait：

```nabi norun
use linalg::AsVector;
use core::array::Vector;

fn rms<X: AsVector<f64>>(x: X) -> f64 {
    let v: Vector<f64> = x.as_vector();
    (v .* v).mean().sqrt()
}

println(rms([3, 4]));        // 3.5355339059327378
```

混合字面量请先写出元素类型：`[0.0, 0.5, 1]` 是浮点数列表，而 `[0, 0.5, 1]` 起初是整数列表，其中的 `0.5` 会报错。

## 完整示例

一杯茶从 90 度开始冷却。读数存放在一个 CSV 文件中；牛顿冷却定律指出，温度以与温差成正比的速率趋向室温，即 `T' = k (room - T)`。我们根据数据拟合 `room` 和 `k`，把该定律作为 ODE 求解，检查拟合结果并绘图。

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

每一步都是对一个包的一次调用，步骤之间传递的都是普通的 `Vector<f64>`：没有转换层，除了解析 CSV 之外没有任何复制。

## 教程

每个包的文档都从一篇教程开始：逐步解决一个小问题，其中的代码由包的测试运行。

| 教程 | 包 | 内容 |
|----------|---------|--------------|
| [第一次数据分析](https://docs.nabi-lang.org/packages/table/latest) | `table`, `stats`, `plot` | 读取 CSV 文件，清洗、分组、检验差异、绘图 |
| [线性代数](https://docs.nabi-lang.org/packages/linalg/latest) | `linalg` | 求解方程组、最小二乘、特征值 |
| [稀疏方程组](https://docs.nabi-lang.org/packages/sparse/latest) | `sparse` | 一个有 10 000 个未知数的稀疏方程组，迭代求解器和直接求解器 |
| [统计](https://docs.nabi-lang.org/packages/stats/latest) | `stats` | 分布、随机样本、检验、回归、自助法 |
| [信号与频谱](https://docs.nabi-lang.org/packages/signal/latest) | `signal` | FFT、噪声中的功率谱、滤波器、峰值、卷积 |
| [拟合与优化](https://docs.nabi-lang.org/packages/optimize/latest) | `optimize` | 求根、求极小值、曲线拟合、线性规划 |
| [微分方程](https://docs.nabi-lang.org/packages/ode/latest) | `ode` | 初值问题、事件、刚性系统、边值问题 |
| [导数](https://docs.nabi-lang.org/packages/autodiff/latest) | `autodiff` | 自定义函数的精确梯度、雅可比矩阵和泰勒级数 |

## 从 NumPy、SciPy 和 pandas 迁移

| Python | Nabi |
|--------|------|
| `np.array([[1, 2], [3, 4]])` | `let m: Matrix<f64> = [[1, 2], [3, 4]];` |
| `np.linspace(0, 1, 5)`, `np.arange` | `array::linspace(0.0, 1.0, 5)`, `array::arange(0.0, 1.0, 0.25)` |
| `a.sum(axis=0)`, `a.argmax()`, `np.allclose(a, b)` | `a.sum_along(0)`, `a.argmax()`, `a.all_close(b)` |
| `a @ b`, `np.linalg.solve(a, b)` | `a @ b`，`linalg::solve(a, b)` 或 `a \ b` |
| `np.linalg.eigh`, `svd`, `cholesky`, `lstsq` | `linalg::eigh`, `svd`, `cholesky`, `lstsq` |
| `scipy.sparse.csr_matrix`, `spsolve`, `cg` | `sparse::Csr`, `sparse::lu(a).solve(b)`, `sparse::cg` |
| `jax.grad(f)(x)` | `autodiff::grad(f, x)` 或 `∇(f)(x)` |
| `scipy.stats.norm(0, 1).cdf(x)` | `stats::Normal::new(0.0, 1.0).cdf(x)` |
| `np.mean`, `np.std(x, ddof=1)`, `np.quantile` | `stats::mean`, `stats::std(x, ddof: 1)`, `stats::quantile` |
| `scipy.stats.ttest_ind`, `linregress` | `stats::ttest_ind`, `stats::ols` |
| `np.fft.fft`, `rfft`, `scipy.signal.welch` | `signal::fft`, `signal::rfft`, `signal::welch` |
| `scipy.signal.butter(..., output="sos")`, `sosfiltfilt` | `signal::filter::butter`, `sosfiltfilt` |
| `scipy.interpolate.CubicSpline`, `make_interp_spline` | `interpolate::cubic_spline`, `make_interp_spline` |
| `scipy.integrate.quad`, `solve_ivp` | `interpolate::quad`（或 `∫(f, a..b)`），`ode::solve_ivp` |
| `scipy.optimize.minimize`, `curve_fit`, `brentq` | `optimize::minimize`, `curve_fit`, `brentq` |
| `scipy.special.gamma`, `jv`, `erf` | `special::gamma`, `jv`, `erf` |
| `pd.read_csv`, `df[df.x > 2]`, `df.groupby("k").agg(...)` | `table::read_csv`, `t.filter(col("x") .> 2)`, `t.group_by(["k"]).agg([...])` |
| `pd.read_parquet`, `pl.scan_parquet(...).filter(...).collect()` | `table::read_parquet`, `table::scan_parquet(..).filter(..).collect()` |
| `plt.plot(x, y); plt.savefig("a.svg")` | `plot::axes().line(x, y).save("a.svg")` |

## 速度

这些包是纯 Nabi 实现，输入较大时会用上所有核心：稠密矩阵乘积在单个 Apple M 系列核心上约达 30 GFLOP/s，在全部核心上超过 100 GFLOP/s；任意长度的 FFT 都是 `O(n log n)`；稀疏矩阵向量乘积每秒可读取数亿个非零元。每个包的 README 都有一个“Speed”小节，列出其性能数字以及与 NumPy、SciPy 和参考库的比较。保持代码快速有两个习惯：用 `Array::from_fn` 或点运算符构造数组，而不是在循环中向列表 push；在调用之间传递 `Vector<f64>`，而不是来回转换。

## 延伸阅读

- 下一章：[23. 插件](23-plugins.html)
