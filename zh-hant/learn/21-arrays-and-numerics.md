# 21. 陣列與數值運算

Nabi 的數值運算分成三層。一般的數字有常見的運算子，再加上表示乘冪的 `**`。`List<T>` 和固定長度陣列可以使用[第 8 章](08-collections-iterators.html)介紹的逐元素點運算子（`.+`、`.*` 等）。對於有多個軸的資料，`core::array::Array<T, N>` 是一個 n 維陣列，支援 NumPy 風格的索引、切片和廣播（broadcasting）。它內建在 core 中，讓所有做線性代數、統計或繪圖的函式庫都能共用同一個型別。

## 乘冪

`a ** b` 計算 `a` 的 `b` 次方。它的結合比前置的 `-` 更緊，並且向右結合，和數學及 Python 一樣：`-x ** 2` 是 `-(x ** 2)`，而 `2 ** 3 ** 2` 是 `2 ** 9`。要對負數取乘冪，請加上括號。

```nabi
let x = 3.0;
println(f"{x ** 2} {-x ** 2} {(-x) ** 2}");
println(f"{2 ** 3 ** 2} {2 ** 10}");
println(f"{2.0 ** -1} {2.0 ** 0.5}");
```

```output
9 -9 9
512 1024
0.5 1.4142135623730951
```

整數底數接受 `u32` 指數，結果是和底數同型別的整數；溢位時會像 `*` 一樣觸發 panic。浮點數底數接受浮點數指數，或 `i32` 指數，後者以重複相乘計算。指數不必和底數同型別：`n` 是 `i64` 時寫 `n ** 2` 也沒問題，字面值會成為 `u32`。`2 ** -1` 是編譯錯誤，因為答案是分數；錯誤訊息會告訴你改寫成 `2.0 ** -1.0`。`**=` 就地取乘冪，`.**` 則是逐元素的形式。

```nabi
let n = 7;
let k: u32 = 3;
let y = 1.5;
y **= 2;
println(f"{n ** 2} {n ** k} {y} {[1, 2, 3] .** 2:?}");
```

```output
49 343 2.25 [1, 4, 9]
```

`^` 仍然是位元 XOR，和 C 與 Rust 相同。

## 建立陣列

`Array<T, N>` 沿著 `N` 個軸存放型別為 `T` 的元素；`N` 是階數（rank），屬於型別的一部分。`Vector<T>` 和 `Matrix<T>` 分別是階數 1 和 2 的名稱。在預期為 `Array` 的位置，巢狀的串列字面值就會建立一個陣列：巢狀的深度就是階數，各列決定形狀。

```nabi
use core::array::{Array, Matrix};

let m: Matrix<f64> = [[1, 2, 3],
                      [4, 5, 6]];
println(f"{m:?}");
println(f"shape {m.shape():?}, {m.len()} elements, rank {m.ndim()}");
```

```output
[[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]]
shape [2, 3], 6 elements, rank 2
```

整數字面值變成了 `f64`，因為所在位置的型別是 `f64`。長度不同的列是編譯錯誤（E0142）：陣列是一個矩形，所以不規則的資料請放在 `List<List<T>>` 中。

建構函式以長度組成的陣列來接受形狀：

```nabi
use core::array::Array;

let z: Array<i64, 2> = Array::zeros([2, 3]);
let o: Array<f64, 1> = Array::ones([3]);
let g = Array::from_fn([2, 3], |i: [usize; 2]| -> usize { 10 * i[0] + i[1] });
let r = Array::from_list([1, 2, 3, 4, 5, 6], [3, 2]);
println(f"{z:?} {o:?}");
println(f"{g:?} {r:?}");
```

```output
[[0, 0, 0], [0, 0, 0]] [1.0, 1.0, 1.0]
[[0, 1, 2], [10, 11, 12]] [[1, 2], [3, 4], [5, 6]]
```

`Array::full(shape, v)` 重複同一個值，`Array::from(xs)` 則把串列轉成階數 1 的陣列。

固定長度的 `[T; N]` 有自己的重複形式 `[v; N]`：`v` 只求值一次，每個元素都拿到自己的一份副本。像 `0.0` 這樣的一般值會直接複製；存放集合的元素則會被 clone，所以 `[List::<f64>::new(); 4]` 是四個各自獨立的串列，`[[0.0; 3]; 3]` 是三個各自獨立的列（[第 2 章](02-values-and-types.html)）。`array::from_fn(f)` 建立一個 `[T; N]`，其中第 `i` 個元素是 `f(i)`，依序呼叫；`N` 來自預期的型別，或寫成 `array::from_fn::<T, N>`。每個元素都是自己的值，所以它也能建立 `[v; N]` 建立不了的陣列：元素不是 `Clone` 的，或是只有一個擁有者的資源。

```nabi
let xs = [3, 1, 2];
let tens: [i64; 3] = array::from_fn(|i| xs[i] * 10);
let lists = array::from_fn::<List<i64>, 2>(|_| []);
lists[0].push(1);
println(f"{tens:?} {lists:?}");
```

```output
[30, 10, 20] [[1], []]
```

## 索引與切片

在中括號內，以逗號分隔的每個部分對應一個軸。位置會在它的軸上選出一個索引並移除該軸；切片 `a:b:k` 則保留該軸以及它選出的元素，規則和串列切片相同（負數邊界從尾端算起、超出範圍時夾住、負數步長往回走）。位置的數量等於階數時，得到的是單一元素。

```nabi
use core::array::Array;

let a: Array<i64, 2> = [[0, 1, 2, 3],
                        [10, 11, 12, 13],
                        [20, 21, 22, 23]];
println(f"{a[1, 2]} {a[-1, -1]}");
println(f"{a[1]:?} {a[:, 0]:?}");
println(f"{a[1:, ::2]:?}");
println(f"{a[::-1, 3]:?}");
```

```output
12 23
[10, 11, 12, 13] [0, 10, 20]
[[10, 12], [20, 22]]
[23, 13, 3]
```

還有兩種部分可以幫忙處理形狀。`..` 代表其他部分沒有用到的所有軸，所以不論陣列的階數是多少，`a[.., 0]` 都是它的第一行。`none` 插入一個長度為 1 的軸，向量就是這樣變成一列或一行的。

```nabi
use core::array::Array;

let a: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
let v: Array<i64, 1> = [7, 8, 9];
println(f"{a[.., 0]:?} {v[none, :].shape():?} {v[:, none].shape():?}");
```

```output
[1, 4] [1, 3] [3, 1]
```

位置可以是任何整數型別；`usize` 的迴圈計數器和字面值一樣可以用。

切片、留下軸的位置、`..`、`none`、`t()` 和 `reshape` 都回傳*檢視*（view）：在相同元素上套用新的形狀和步幅，不做任何複製。透過檢視寫入，就是寫入原本的陣列。

```nabi
use core::array::Array;

let a: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
let col = a[:, 1];
col[0] = 20;
let t = a.t();
println(f"{a:?} {t[2, 1]} {t.shape():?}");
println(f"{a.reshape([3, 2]):?}");
```

```output
[[1, 20, 3], [4, 5, 6]] 6 [3, 2]
[[1, 20], [3, 4], [5, 6]]
```

需要獨立的陣列時，使用 `copy()`。形狀相同的布林陣列是一個*遮罩*（mask）：`a[mask]` 會依序收集遮罩為 `true` 之處的元素，組成一個新的階數 1 陣列。

```nabi
use core::array::Array;

let a: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
println(f"{a[a .% 2 .== 0]:?}");
```

```output
[2, 4, 6]
```

## 指派給選取範圍

凡是可以索引的，都可以指派。單一值會填滿整個選取範圍；陣列會被複製進去，並延展成選取範圍的形狀（請見下文的廣播）；遮罩指派只會改變被選取的元素。

```nabi
use core::array::Array;

let a: Array<f64, 2> = Array::zeros([3, 4]);
a[0, :] = 1.0;
a[1:, 0] = 2.0;
let row: Array<f64, 1> = [5, 6, 7, 8];
a[2] = row;
a[a .> 6.0] = -1.0;
println(f"{a:?}");
```

```output
[[1.0, 1.0, 1.0, 1.0], [2.0, 0.0, 0.0, 0.0], [5.0, 6.0, -1.0, -1.0]]
```

`List` 也接受切片指派，用於長度相同的一段連續元素：`xs[1:3] = [8, 9];`。帶有步長的串列切片不能被指派（編譯器會指出這點，並建議一次存入一個元素）。

## 逐元素運算與廣播

點運算子會對陣列逐元素運算：算術運算得到元素型別的陣列，比較運算得到 `bool` 陣列。任一側的純量會套用到每個元素。

```nabi
use core::array::Array;

let a: Array<f64, 2> = [[1, 2], [3, 4]];
println(f"{a .* a .+ 1.0:?}");
println(f"{2.0 .* a:?} {a .** 2.0:?}");
println(f"{a .>= 2.0:?}");
```

```output
[[2.0, 5.0], [10.0, 17.0]]
[[2.0, 4.0], [6.0, 8.0]] [[1.0, 4.0], [9.0, 16.0]]
[[false, true], [true, true]]
```

兩個階數相同的陣列必須逐軸一致，唯一的例外是長度為 1 的軸會延展成另一側的長度。這就是 NumPy 的規則，而且是唯一的規則：它放在 core 中（`broadcast_shape`），讓所有陣列型別的行為一致。其他情況都會觸發 panic，並指出兩邊的形狀。

```nabi
use core::array::Array;

let m: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
let row: Array<i64, 2> = [[10, 20, 30]];
let col: Array<i64, 2> = [[100], [200]];
println(f"{m .+ row:?}");
println(f"{m .+ col:?}");
```

```output
[[11, 22, 33], [14, 25, 36]]
[[101, 102, 103], [204, 205, 206]]
```

階數較低的陣列會對齊另一個陣列的最後幾個軸，就像它前面多了幾個長度為 1 的軸，所以向量可以加到矩陣的每一列上。結果的階數是較高的那一個。

```nabi
use core::array::Array;

let m: Array<f64, 2> = [[1, 2], [3, 4]];
let v: Array<f64, 1> = [10, 20];
println(f"{m .+ v:?}");
```

```output
[[11.0, 22.0], [13.0, 24.0]]
```

一整串點運算子會以單一迴圈執行，不產生中間陣列，對 `Array` 和 `List` 都一樣：`a .* x .+ b .* 2.0` 只讀取 `a`、`x` 和 `b` 的每個元素一次，並只寫入結果一次。

## 就地更新

`y .= expr` 把 `expr` 的元素寫入 `y`，而 `y .+= x`、`.-=`、`.*=`、`./=`、`.%=`、`.**=` 則把 `y` 和 `x` 逐元素結合。過程中不會配置任何記憶體：迴圈直接寫入 `y`。目標可以是檢視，所以陣列的切片或某一行會在原位更新。

```nabi
use core::array::Array;

let y: Array<f64, 1> = Array::zeros([4]);
let x: Array<f64, 1> = [1, 2, 3, 4];
y .= 0.5 .* x .+ 1.0;
y .*= 2.0;
let m: Array<f64, 2> = [[1, 2], [3, 4]];
m[:, 0] .= 0.0;
let xs = [1, 2, 3];
xs .+= [10, 10, 10];
println(f"{y:?} {m:?} {xs:?}");
```

```output
[3.0, 4.0, 5.0, 6.0] [[0.0, 2.0], [0.0, 4.0]] [11, 12, 13]
```

目標會保持自己的形狀：是右側延展成目標的形狀，絕不會反過來。當右側讀取目標本身其他位置的元素時（`v .= v[::-1]`），它讀的是事先取得的副本，所以結果和先算出每個元素、再全部存入時相同。其他運算子用 `.=` 來寫：`xs .= xs .%% 3`。

## 缺失值

元素型別 `T | none` 用來標示缺失值。`a .?? b` 會用 `b` 中對應的元素填補每個 `none`，若 `b` 是單一值，就直接用 `b` 填補。NaN 不是 `none`：它是一個浮點數值，會原樣保留。若要把 NaN 當作缺失值，用 `f64::is_nan.(a)` 選出它們再指派。

```nabi
use core::array::Array;

let raw: Array<f64 | none, 1> = Array::from_list([1.0, none, f64::NAN], [3]);
let filled = raw .?? 0.0;
println(f"{filled:?}");
filled[f64::is_nan.(filled)] = -1.0;
println(f"{filled:?}");
```

```output
[1.0, 0.0, NaN]
[1.0, 0.0, -1.0]
```

## 歸約與迴圈

`sum()` 把所有元素相加，`map(f)` 建立一個形狀相同的新陣列，`to_list()` 以列優先順序（最後一個軸變化最快）給出元素，`for x in a` 也依這個順序走訪它們。

```nabi
use core::array::Array;

let a: Array<i64, 2> = [[1, 2], [3, 4]];
let total = a.sum();
let sq = a.map(|x: i64| -> i64 { x * x });
let parts = [];
for x in a.t() {
    parts.push(f"{x}");
}
println(f"{total} {sq:?} {parts.join(" ")}");
```

```output
10 [[1, 4], [9, 16]] 1 3 2 4
```

每個歸約都另有一個 `_along(axis)` 形式，會沿著某個軸對每一條一維切片做歸約並移除該軸，就像 NumPy 的 `axis=` 引數：`m.sum_along(0)` 加總矩陣的各行，`m.sum_along(1)` 加總各列。可用的有 `sum`、`prod`、`mean`、`var` 和 `std`（可指定 `ddof`）、`min`、`max`、`argmin`、`argmax`、`cumsum` 和 `cumprod`，以及用於 `a .> 0.0` 這類布林陣列的 `any`、`all` 和 `count`。只要有一個 NaN，`min`、`max`、`sum` 和 `mean` 的結果就是 NaN；`nan` 形式（`nanmax`、`nanmean` 等）則會略過它。加總採用成對相加，所以即使陣列很長，捨入誤差也維持很小。

```nabi
use core::array::{Matrix, Vector};

let m: Matrix<f64> = [[1, 2, 3], [4, 5, 6]];
println(f"{m.sum()} {m.mean()} {m.max() ?? 0.0} {m.argmax() ?? 0}");
println(f"{m.sum_along(0):?} {m.mean_along(1):?} {m.max_along(0):?} {m.argmin_along(1):?}");
println(f"{m.std(ddof: 1)} {m.cumsum_along(1):?}");

let v: Vector<f64> = [3.0, f64::NAN, 1.0];
println(f"{v.max() ?? 0.0} {v.nanmax() ?? 0.0} {v.nanmean()} {(v .> 2.0).count()}");
```

```output
21 3.5 6 5
[5.0, 7.0, 9.0] [2.0, 5.0] [4.0, 5.0, 6.0] [0, 0]
1.8708286933869707 [[1.0, 3.0, 6.0], [4.0, 9.0, 15.0]]
NaN 3 2 1
```

`concat(parts, axis)` 沿著某個軸串接陣列，`stack(parts, axis)` 把形狀相同的陣列放到一個新的軸上，`where(mask, a, b)` 在遮罩為真之處取 `a` 的元素，其他地方取 `b` 的元素。`sorted_along(axis)` 對每一條一維切片排序（NaN 排在最後），`a.all_close(b)` 以相對容差比較兩個浮點數陣列，就像 `x.is_close(y)` 比較兩個數字一樣：`0.1 + 0.2 == 0.3` 是 `false`，但 `(0.1 + 0.2).is_close(0.3)` 是 `true`。

```nabi
use core::array::{Matrix, concat, stack, where};

let m: Matrix<f64> = [[3, 1], [2, 4]];
println(f"{concat([m, m]).shape():?} {stack([m, m]).shape():?}");
println(f"{where(m .> 2.0, m, m .* -1.0):?} {m.sorted_along(0):?}");
println(f"{((m .* 0.1) .* 10.0).all_close(m)} {(0.1 + 0.2).is_close(0.3)}");
```

```output
[4, 2] [2, 2, 2]
[[3.0, -1.0], [-2.0, 4.0]] [[2.0, 1.0], [3.0, 4.0]]
true true
```

## 浮點數、複數與亂數

`num::FloatingPoint` 是同時適用 `f32` 和 `f64` 的程式碼所用的約束：它把兩者的常數（`T::NAN`、`T::PI`）和數學函式（`sqrt`、`exp`、`sin` 等）集中成一個 trait。`num::Num` 是範圍更廣的約束，所有整數和浮點數型別都滿足它：`+ - * /`、`0`、`1` 以及比較。浮點數串列和迭代器有 `min`、`max` 和 `sum`；只要其中有一個 NaN，`min` 和 `max` 就是 NaN，而 `sum` 以固定順序成對相加，所以結果不受平台影響。浮點數只有部分排序，所以請用 `total_cmp` 排序。`num::linspace(a, b, n)` 給出從 `a` 到 `b` 等距分布的 `n` 個值，`num::arange(a, b, step)` 則從 `a` 開始逐步遞增到 `b` 為止，不包含 `b`。

```nabi
fn hypot<T: num::FloatingPoint>(a: T, b: T) -> T { (a * a + b * b).sqrt() }
println(f"{hypot(3.0, 4.0)} {hypot(5.0f32, 12.0f32)}");

let xs = [2.5, -1.0, 7.0, 0.5];
println(f"{xs.max() ?? 0.0} {xs.sum()} {xs.sorted_by(|a, b| a.total_cmp(b)):?}");
println(f"{num::linspace(0.0, 1.0, 5):?}");
```

```output
5 13
7 9 [-1.0, 0.5, 2.5, 7.0]
[0.0, 0.25, 0.5, 0.75, 1.0]
```

`total_cmp` 是 IEEE 754 的全序：符號位元為 1 的 NaN 排在最前面，符號位元為 0 的排在最後面。`f64::NAN` 和 `f32::NAN` 在每個目標上都是符號位元為 0 的 quiet NaN，所以排在 `inf` 之後；算術運算產生的 NaN（`0.0 / 0.0`）可能是任一種符號，x86 會把符號位元設為 1。

```nabi
let ys = [1.0, f64::NAN, f64::NEG_INFINITY, -0.0, f64::INFINITY];
println(f"{ys.sorted_by(|a, b| a.total_cmp(b)):?}");
println(f"{f64::NAN.is_sign_negative()} {f64::NAN.to_bits():x}");
```

```output
[-inf, -0.0, 1.0, inf, NaN]
false 7ff8000000000000
```

浮點數算術在每個目標上都得到相同的位元。每個運算都以「最接近值、同距時取偶數」的方式捨入；`a * b + c` 會捨入兩次（只有 `a.mul_add(b, c)` 會融合成一次），次正規數會保留，`min` / `max` 會略過 NaN，並把 `-0.0` 視為小於 `0.0`。唯一交給 CPU 決定的是運算會產生哪一個 NaN，如上面的符號所示；`sin`、`exp` 和其他函式庫函式在最後一個位元上也可能有差異。

### 16 位元浮點數：`f16` 與 `bf16`

`f16`（IEEE 半精度，最大到 65504）和 `bf16`（bfloat16：具有 `f32` 的範圍，精度為 8 個位元）是儲存用的型別：存放權重或像素的 `Array<f16, 2>` 只佔用 `f32` 一半的記憶體。放在它們所在位置的字面值，或帶有後綴 `f16` / `bf16` 的字面值，會從十進位數只捨入一次；`as` 可以和所有數字型別互相轉換；`+ - * /` 以 `f32` 計算再捨入回來，這正是 IEEE 的精確結果。輸出時會印出能讀回相同值的最短位數。

```nabi
use core::array::Array;

let w: Array<f16, 2> = [[0.1, 0.25], [1.5, -2.0]];
let h = w[0, 0];
println(f"{h} {h.to_bits():x} {h as f64} {h * 3.0}");
println(f"{70000.0f32 as f16} {3.14159 as bf16} {f16::MAX}");
```

```output
0.1 2e66 0.0999755859375 0.2998
inf 3.14 65500
```

運算子以外的數學運算，請先轉換：`h.to_f32().sin()`。字面值模式無法比對 16 位元浮點數；請在守衛（guard）中比較（`x if x == 1.0 =>`）。

`num::Complex<T>` 是以 `f32` 或 `f64` 為基礎的複數，後綴 `i` 用來寫虛部：`3.0 + 4.0i`。它有算術運算子，任一側都可以是實數，另外還有 `abs`、`arg`、`conj`、`exp`、`ln`、`sqrt` 和 `pow`。

```nabi
let z = 3.0 + 4.0i;
println(f"{z} {z.abs()} {z * 2i} {z.conj()}");
```

```output
3+4i 5 -8+6i 3-4i
```

`std::random` 提供只需一次呼叫的形式（`random::range(1..7)`、`random::f64()`、`random::normal()`、`random::choice(xs)`、`random::shuffle(xs)`），它們都取自同一個共用的產生器，這個產生器只從作業系統取得一次種子，所以在迴圈中呼叫也很快。若需要可以重現的執行結果，用 `seed` 建立一個 `Xoshiro256PlusPlus`：相同的種子在每個目標和每個版本上都會產生相同的數字。它的方法來自 `Rng` trait：可用於任何整數型別的 `range`（`0..n`、`1..=6`、`0u8..=255`）、用於浮點數的 `uniform`、分布 `normal`、`exponential`、`gamma`、`beta`、`poisson` 和 `binomial`、依權重選出索引的 `weighted`、`choice`、`sample` 和 `shuffle`，以及一次填滿整個 `Array` 的 `normal_array(shape)` / `random_array(shape)`。`split()` 會為另一個執行緒建立一個獨立的產生器。這些都不適合用於機密資料：金鑰和權杖要取自 `random::bytes` 或 `OsRng`，它們會讀取作業系統的安全來源。

```nabi
use std::random;
use std::random::{Rng, Xoshiro256PlusPlus};

let a = Xoshiro256PlusPlus::seed(7);
let b = Xoshiro256PlusPlus::seed(7);
println(a.range(1..=6) == b.range(1..=6));
println(0.0 <= a.uniform(0.0, 1.0) < 1.0);
println(f"{a.weighted([0.1, 0.0, 0.9]) != 1} {a.normal_array([2, 3], std: 0.1).shape():?}");
println(random::range(1..7) in 1..7);
```

```output
true
true
true [2, 3]
true
```

## 記憶體與速度

陣列的元素緊密存放在同一個緩衝區中。當元素型別不含任何參考（數字、`bool`、`char`、`#[derive(Pod)]` 結構）時，垃圾回收器永遠不會檢查它的內部，所以即使是很大的陣列，也不會增加任何回收成本。8 KiB 以上的緩衝區會對齊到 64 位元組，並且在陣列仍可觸及時永遠不會移動，這正是向量程式碼和 BLAS 這類 C 函式庫所需要的：`unsafe { a.as_ptr() }` 給出第一個元素，`strides()` 則說明如何走訪其餘元素。較小的緩衝區則在你持有 pin 期間保持不動：`bare::gc::pin(a)` 會給出一個守衛物件，它的 `as_data_ptr()` 就是第一個元素。`is_contiguous()` 判斷元素是否以列優先順序相鄰排列；`copy()` 會讓它們變成相鄰排列。

`Array::from_list` 和 `to_list()` 會複製資料。若要改為共用同一個緩衝區，`Array::view_list(xs, shape)` 會把串列的元素當作陣列來檢視，而 `a.as_list()` 會在連續陣列的緩衝區上給出一個串列（當陣列是帶步幅或偏移的檢視時回傳 `none`：這時請用 `to_list()`）。透過任一邊寫入，另一邊都看得到，直到串列成長並搬到自己的緩衝區為止。

```nabi
use core::array::Array;

let xs = [1.0, 2.0, 3.0, 4.0];
let m: Array<f64, 2> = Array::view_list(xs, [2, 2]);
m[1, 0] = 30.0;
println(f"{xs:?}");
```

```output
[1.0, 2.0, 30.0, 4.0]
```

## SIMD 向量

走訪串列的迴圈可能已經以向量指令執行：編譯器會自動把簡單的逐元素迴圈向量化。當你想明確指出向量在哪裡時，`core::simd` 提供 `Simd<T, N>`，也就是由 `N` 個數字型別通道（lane）組成的向量，並有 `f32x4`、`f64x2`、`i32x8` 等別名。它的運算子一次作用於所有通道，點比較運算會得到一個 `Mask`，而 `select` 依照遮罩選取通道。

```nabi
use core::simd::{f32x4, Simd};

fn dot(a: List<f32>, b: List<f32>) -> f32 {
    let acc = f32x4::splat(0.0);
    let i = 0;
    while i + 4 <= a.len() {
        acc += f32x4::load(a, i) * f32x4::load(b, i);
        i += 4;
    }
    let s = acc.reduce_sum();
    while i < a.len() {
        s += a[i] * b[i];
        i += 1;
    }
    s
}

let a: List<f32> = [1.0, 2.0, 3.0, 4.0, 5.0];
let b: List<f32> = [2.0, 2.0, 2.0, 2.0, 2.0];
println(dot(a, b));

let x = f32x4::from_array([1.0, -2.0, 3.0, -4.0]);
let relu = (x .> f32x4::splat(0.0)).select(x, f32x4::splat(0.0));
println(f"{relu:?} {x.abs().reduce_max()}");
```

```output
30
[1.0, 0.0, 3.0, 0.0] 4
```

在 `x86_64`、`aarch64`、`wasm32` 以及其他具有向量單元的目標上，不論最佳化等級為何，算術、比較和 `select` 都是向量指令；在其他目標上則是走訪各通道的迴圈，結果相同。整數通道在溢位時會環繞，而不是觸發 panic，因為向量指令不做任何檢查。完整的清單在[函式庫規格 2.24](../spec/refs-standard-libraries/200-core/224-simd.html) 中。

## 自訂陣列型別

索引語法並非 core 的 `Array` 專用。含有多個部分的中括號會以元組呼叫 `Index`（讀取）或 `IndexSet`（寫入），而切片部分會以 `core::index::SliceArg` 傳入。如果你的型別實作了標記 trait `AxisIndexed`，每個整數部分都會以 `isize` 傳入，`none` 以 `NewAxis` 傳入，`..` 以 `Rest` 傳入，所以一個 impl 就能涵蓋所有寫法的索引。

```nabi
use core::index::{AxisIndexed, SliceArg};

struct Grid { rows: usize; cols: usize; }

impl AxisIndexed for Grid {}

impl index::Index<(SliceArg, isize)> for Grid {
    type Output = str;
    fn index(self, at: (SliceArg, isize)) -> str {
        let (first, count, step) = at.0.indices(self.rows);
        f"{count} rows from {first} by {step}, column {at.1}"
    }
}

let g = Grid { rows: 10, cols: 4 };
let c: usize = 2;
println(g[2:, c]);
println(g[::-3, -1]);
```

```output
8 rows from 2 by 1, column 2
4 rows from 9 by -3, column -1
```

定義數值型別的套件也可以為它提供純量在左側的運算子。孤兒規則（[第 6 章](06-structs-enums-traits.html)）允許你為 core 中的型別實作 core 中的 trait，只要該 trait 的某個型別引數是你自己的型別，所以 `2.0 * v` 可以運作：

```nabi
#[derive(Clone, Copy)]
struct V2 { x: f64; y: f64; }

impl ops::Mul<V2> for f64 {
    type Output = V2;
    fn mul(self, v: V2) -> V2 { V2 { x: self * v.x, y: self * v.y } }
}

let v = 2.0 * V2 { x: 1.5, y: -1.0 };
println(f"{v.x} {v.y}");
```

```output
3 -2
```

這條規則仍然拒絕 `impl<T> Mul<V2> for T`，因為其中有一個單純的型別參數出現在你的型別之前：別的套件也可能為它自己的型別寫出同樣的 impl。

若要讓你的型別上的點運算式像 `Array` 一樣融合，請實作 `Broadcastable`，把它的元素呈現為一個 `Array`，以及 `Materialize`，用結果的元素建構出你的型別。把資料存放在 `Array` 中的型別，直接回傳它即可：

```nabi
use core::array::Array;
use core::ops::{Broadcastable, Materialize};

struct Signal { data: Array<f64, 1>; }

impl Broadcastable<1> for Signal {
    type Elem = f64;
    fn broadcast_view(self) -> Array<f64, 1> { self.data }
}

impl Materialize<f64, 1> for Signal {
    type Output = Signal;
    fn materialize(out: Array<f64, 1>) -> Signal { Signal { data: out } }
}

let s = Signal { data: [1, 2, 3] };
let t = s .* 2.0 .+ 0.5;
t .-= 0.5;
println(f"{t.data:?}");
```

```output
[2.0, 4.0, 6.0]
```

當你的型別自己的點運算子會改變型別時，就像物理量那樣（長度除以時間是速度），運算式會先依那些運算子推定型別，再由它們得出的型別建構結果：迴圈仍然只有一個，型別也仍然受到檢查。包裝單一數字的型別可以實作 `BroadcastScalar`（回傳它的一般值），讓它在這類運算式中充當純量：例如 `d ./ t .+ v0`，其中有長度和時間的陣列，以及一個速度 `v0`。

## 字面值後綴

套件可以在處理函式上加上 `#[literal(number "suffix")]`，為數字提供自己的後綴。當處理函式是 `pub` 時，所有相依於你的套件的套件都會取得這個後綴，不需要 `use`：

```nabi norun
// 在 `units` 套件中
pub struct Meters { pub v: f64; }

#[literal(number "m")]
pub fn meters(n: f64) -> Meters { Meters { v: n } }
```

```nabi norun
// 在含有 `"units": Dep::path("../units")` 的套件中
let d = 9.81m;          // units::meters(9.81)
println(f"{d.v}");
```

不是 `pub` 的處理函式只在它自己的套件內有效。兩個相依套件註冊相同的後綴時，它的用法就會有歧義；請為其中一個指定 `priority`。

## 延伸閱讀

- 下一章：[22. 用 Nabi 做科學計算](22-science.html)
