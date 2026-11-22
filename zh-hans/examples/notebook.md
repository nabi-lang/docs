# 在 Jupyter Notebook 中探索数据

在 Jupyter 中安装 Nabi 内核，逐个单元格地探索气温数据；另有一个完成同样工作的可运行程序。

在 notebook 中，你一步一步地工作：一个单元格加载数据，另一个定义函数，再一个检查结果。下面的程序就是把这些单元格从上到下依次连起来。它根据四周的每日最高气温计算汇总统计、每周平均值和一个文本直方图。

## 代码

```nabi
// Daily high temperatures (°C) for four weeks.
let temps = [
    12.5, 14.0, 13.2, 15.8, 17.1, 16.4, 18.9,
    19.5, 21.0, 20.2, 22.7, 24.1, 23.3, 22.0,
    20.8, 19.4, 21.6, 23.9, 25.2, 26.0, 24.4,
    22.1, 20.5, 18.7, 17.9, 19.2, 21.3, 22.8,
];

fn mean(xs: List<f64>) -> f64 { xs.sum() / xs.len() as f64 }

fn std_dev(xs: List<f64>) -> f64 {
    let m = mean(xs);
    (xs.map(|x| (x - m) ** 2).sum() / xs.len() as f64).sqrt()
}

fn median(xs: List<f64>) -> f64 {
    let sorted = xs.sorted_by(f64::total_cmp);
    let n = sorted.len();
    if n % 2 == 1 { sorted[n / 2] } else { (sorted[n / 2 - 1] + sorted[n / 2]) / 2.0 }
}

// Summary statistics.
let lo = temps.min() ?? f64::NAN;
let hi = temps.max() ?? f64::NAN;
println(f"n      = {temps.len()}");
println(f"mean   = {mean(temps):.2}");
println(f"median = {median(temps):.2}");
println(f"stddev = {std_dev(temps):.2}");
println(f"range  = {lo:.1} .. {hi:.1}");

// Weekly averages.
for week in 0..4 {
    let days = temps[week * 7:(week + 1) * 7].to_list();
    println(f"week {week + 1}: {mean(days):5.2}");
}

// An ASCII histogram with 2-degree buckets.
let start = 12.0;
let width = 2.0;
let counts = List::filled(8, 0);
for t in temps {
    counts[((t - start) / width) as usize] += 1;
}
println("histogram:");
for (i, c) in counts.enumerate() {
    let from = start + width * i as f64;
    println(f"  {from:4.1}-{from + width:4.1} | {"#".repeat(c)} {c}");
}
```

```output
n      = 28
mean   = 20.16
median = 20.65
stddev = 3.49
range  = 12.5 .. 26.0
week 1: 15.41
week 2: 21.83
week 3: 23.04
week 4: 20.36
histogram:
  12.0-14.0 | ## 2
  14.0-16.0 | ## 2
  16.0-18.0 | ### 3
  18.0-20.0 | ##### 5
  20.0-22.0 | ###### 6
  22.0-24.0 | ###### 6
  24.0-26.0 | ### 3
  26.0-28.0 | # 1
```

## 讲解

- notebook 单元格中顶层的 `let` 和 `fn` 在整个会话中都保持有效。单元格可以再次 `let` 同一个名称，就像源文件一样，所以可以用不同的数据重新运行单元格（[5.1.4 同一作用域中的重新声明](../spec/refs-lang/005-modules/501-variable-declaration.html#514-redeclaration-in-the-same-scope)）。
- `f64` 没有 `Ord`（NaN 无法排序），所以中位数按 IEEE 全序排序：`sorted_by(f64::total_cmp)`（[集合与迭代器](../learn/08-collections-iterators.html)）。
- `temps.min()` / `temps.max()` 可用于浮点数（与 NumPy 一样，NaN 会胜出）；`?? f64::NAN` 处理空列表的情况（[`Iterator`](../../std/core/iter.html#trait.Iterator)）。
- `temps[week * 7:(week + 1) * 7]` 以切片取出列表的一个区间，`to_list()` 把它复制成 `mean` 接受的 `List`。`sum()` 对整个列表求和，`(x - m) ** 2` 求平方（[3.6 索引与切片](../spec/refs-lang/003-expressions/306-indexing-and-slicing.html)）。
- 直方图的条来自 `"#".repeat(c)`，区间标签用 `{from:4.1}` 这样的格式说明符对齐。用 `{}` 输出的 `f64` 使用能读回同一个值的最短位数，所以 `26.0` 输出为 `26`；需要固定位数时，写出精度，例如 `{hi:.1}`（[字符串](../learn/09-strings.html)）。

## 运行

先在 Jupyter 中注册 Nabi 内核。`nabi jupyter install` 把 kernelspec（`kernels/nabi/kernel.json`）写入 Jupyter 的数据目录（macOS 上是 `~/Library/Jupyter`，Windows 上是 `%APPDATA%\jupyter`，其他系统上是 `~/.local/share/jupyter`；见 [9.8.3 安装 kernelspec](../spec/refs-lang/009-tools/908-jupyter-integration.html#983-kernelspec-installation)）。

```bash
nabi jupyter install
jupyter lab            # pick the "nabi" kernel in a new notebook
```

要为每个环境保留单独的内核，用 `nabi jupyter install --env <name>` 安装。这会再创建一个显示为 `nabi (<name>)` 的内核。

在 notebook 中，上面的程序被拆成几个单元格运行。单元格的最后一个表达式就是它的结果，用 `Display` 显示；以 `?` 结尾可以看到 `Debug` 形式。

```text
In [1]:  let temps = [12.5, 14.0, 13.2, 15.8, 17.1, 16.4, 18.9,
                      19.5, 21.0, 20.2, 22.7, 24.1, 23.3, 22.0,
                      20.8, 19.4, 21.6, 23.9, 25.2, 26.0, 24.4,
                      22.1, 20.5, 18.7, 17.9, 19.2, 21.3, 22.8];
         temps.len()
Out[1]:  28

In [2]:  fn mean(xs: List<f64>) -> f64 { xs.sum() / xs.len() as f64 }
         mean(temps)
Out[2]:  20.16071428571428

In [3]:  temps[0:7]?
Out[3]:  [12.5, 14.0, 13.2, 15.8, 17.1, 16.4, 18.9]

In [4]:  mean(temps[0:7].to_list())
Out[4]:  15.414285714285715
```

`let temps` 和 `fn mean` 在之后的单元格中仍然有定义，所以单元格 `[3]` 和 `[4]` 使用的是之前创建的值和函数。如果修正了一个函数并重新运行它的单元格，之后的单元格就会使用新的定义（[9.7 JIT 单元格执行](../spec/refs-lang/009-tools/907-jit-cells.html)）。要在终端中使用同样的单元格会话，运行 `nabi repl`（[工具](../learn/16-tools.html)）。
