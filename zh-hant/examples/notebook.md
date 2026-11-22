# 在 Jupyter 筆記本中探索資料

在 Jupyter 中安裝 Nabi 核心，逐個儲存格探索氣溫資料，並附上一個完成相同工作、可以直接執行的程式。

在筆記本中，你一步一步地工作：一個儲存格載入資料，另一個定義函式，再一個檢查結果。下面的程式就是把這些儲存格由上而下依序接起來。它從四週的每日最高氣溫算出摘要統計、每週平均值和文字長條圖。

## 程式碼

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

## 說明

- 筆記本儲存格中頂層的 `let` 和 `fn` 會在整個工作階段中保留。和原始檔一樣，儲存格可以再次 `let` 同一個名稱，所以你可以用不同的資料重新執行儲存格（[5.1.4 同一作用域中的重新宣告](../spec/refs-lang/005-modules/501-variable-declaration.html#514-同一作用域中的重新宣告)）。
- `f64` 沒有 `Ord`（NaN 無法排序），所以中位數依 IEEE 全序排序，即 `sorted_by(f64::total_cmp)`（[集合型別與迭代器](../learn/08-collections-iterators.html)）。
- `temps.min()` / `temps.max()` 可用於浮點數（和 NumPy 一樣，NaN 會勝出）；`?? f64::NAN` 處理串列為空的情況（[`Iterator`](../../std/core/iter.html#trait.Iterator)）。
- `temps[week * 7:(week + 1) * 7]` 把串列的一段範圍取為切片，`to_list()` 再把它複製成 `mean` 接受的 `List`。`sum()` 加總整個串列，`(x - m) ** 2` 計算平方（[3.6 索引與切片](../spec/refs-lang/003-expressions/306-indexing-and-slicing.html)）。
- 長條圖的長條來自 `"#".repeat(c)`，區間標籤以 `{from:4.1}` 這類格式規範對齊。用 `{}` 印出的 `f64` 會使用能讀回相同值的最短位數，所以 `26.0` 印成 `26`；想要固定位數時，請寫上精度，例如 `{hi:.1}`（[字串](../learn/09-strings.html)）。

## 執行

首先向 Jupyter 註冊 Nabi 核心。`nabi jupyter install` 會把 kernelspec（`kernels/nabi/kernel.json`）寫入 Jupyter 的資料目錄（macOS 上是 `~/Library/Jupyter`，Windows 上是 `%APPDATA%\jupyter`，其他系統是 `~/.local/share/jupyter`）（[9.8.3 安裝 kernelspec](../spec/refs-lang/009-tools/908-jupyter-integration.html#983-安裝-kernelspec)）。

```bash
nabi jupyter install
jupyter lab            # pick the "nabi" kernel in a new notebook
```

想為每個環境保留各自的核心，請用 `nabi jupyter install --env <name>` 安裝。這會另外建立一個顯示為 `nabi (<name>)` 的核心。

在筆記本中，上面的程式會拆成幾個儲存格執行。儲存格的最後一個運算式就是它的結果，以 `Display` 顯示；在結尾加上 `?` 可以看到 `Debug` 形式。

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

`let temps` 和 `fn mean` 在之後的儲存格中仍然有定義，所以儲存格 `[3]` 和 `[4]` 使用先前建立的值與函式。如果你修正了某個函式並重新執行它的儲存格，後面的儲存格就會使用新的定義（[9.7 JIT 儲存格執行](../spec/refs-lang/009-tools/907-jit-cells.html)）。想在終端機中使用同樣的儲存格工作階段，請執行 `nabi repl`（[工具](../learn/16-tools.html)）。
