# 21. 数组与数值计算

Nabi 的数值计算分三层。普通数字有常见的运算符，另加用于求幂的 `**`。`List<T>` 和定长数组支持[第 8 章](08-collections-iterators.html)中的逐元素点运算符（`.+`、`.*`……）。对于不止一个轴的数据，`core::array::Array<T, N>` 是一个 n 维数组，支持 NumPy 风格的索引、切片和广播。它内置在 core 中，因此做线性代数、统计或绘图的库都能共用同一个类型。

## 求幂

`a ** b` 求 `a` 的 `b` 次幂。它比前缀 `-` 结合得更紧，并且右结合，与数学和 Python 一致：`-x ** 2` 是 `-(x ** 2)`，`2 ** 3 ** 2` 是 `2 ** 9`。要对负数求幂，请加括号。

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

整数底数接受 `u32` 指数，结果是与底数相同类型的整数；溢出时与 `*` 一样触发 panic。浮点底数接受浮点指数或 `i32` 指数，后者用重复乘法计算。指数不必与底数同类型：`n` 为 `i64` 时写 `n ** 2` 没有问题，字面量会成为 `u32`。`2 ** -1` 是编译错误，因为结果是分数；错误信息会提示改写为 `2.0 ** -1.0`。`**=` 原地求幂，`.**` 是逐元素形式。

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

`^` 仍然是 C 和 Rust 中的按位异或。

## 创建数组

`Array<T, N>` 沿 `N` 个轴存放 `T` 类型的元素；`N` 是秩（rank），属于类型的一部分。`Vector<T>` 和 `Matrix<T>` 分别是秩为 1 和 2 的数组。在需要 `Array` 的位置，嵌套列表字面量会构造一个数组：嵌套深度就是秩，各行给出形状。

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

整数字面量变成了 `f64`，因为该位置要求 `f64`。各行长度不同是编译错误（E0142）：数组是矩形，参差不齐的数据请放在 `List<List<T>>` 中。

构造函数以长度数组的形式接受形状：

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

`Array::full(shape, v)` 重复同一个值，`Array::from(xs)` 把列表变成秩为 1 的数组。

定长的 `[T; N]` 有自己的重复形式 `[v; N]`：`v` 只求值一次，每个元素得到各自的副本。`0.0` 这样的普通值按原样复制；存放集合的元素会被克隆，因此 `[List::<f64>::new(); 4]` 是四个独立的列表，`[[0.0; 3]; 3]` 是三个独立的行（[第 2 章](02-values-and-types.html)）。`array::from_fn(f)` 构造一个 `[T; N]`，其第 `i` 个元素是 `f(i)`，按顺序调用；`N` 来自期望类型或 `array::from_fn::<T, N>`。每个元素都是独立的值，所以它还能构造 `[v; N]` 无法构造的数组：元素不是 `Clone` 的，或是只有一个所有者的资源。

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

## 索引与切片

方括号中以逗号分隔的每一部分对应一个轴。位置在其轴上选取一个索引并去掉该轴；切片 `a:b:k` 保留该轴及其选中的元素，规则与列表切片相同（负数边界从末尾算起，越界时截断，负步长反向遍历）。位置的个数等于秩时，得到单个元素。

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

还有两种部分用于调整形状。`..` 代表其他部分剩下的所有轴，因此 `a[.., 0]` 是任意秩数组的第一列。`none` 插入一个长度为 1 的轴，向量就是这样变成行或列的。

```nabi
use core::array::Array;

let a: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
let v: Array<i64, 1> = [7, 8, 9];
println(f"{a[.., 0]:?} {v[none, :].shape():?} {v[:, none].shape():?}");
```

```output
[1, 4] [1, 3] [3, 1]
```

位置可以是任意整数类型；`usize` 循环计数器和字面量一样可用。

切片、保留轴的位置、`..`、`none`、`t()` 和 `reshape` 都返回*视图*：在同一组元素上的新形状和步长，不做复制。通过视图写入就是写入原数组。

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

需要独立的数组时，使用 `copy()`。形状相同的布尔数组是*掩码*：`a[mask]` 按顺序收集其为 `true` 处的元素，组成新的秩为 1 的数组。

```nabi
use core::array::Array;

let a: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
println(f"{a[a .% 2 .== 0]:?}");
```

```output
[2, 4, 6]
```

## 给选区赋值

凡是能索引的，都能赋值。单个值填满选区；数组会被复制进去，并拉伸到选区的形状（见下文的广播）；掩码赋值只改变选中的元素。

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

`List` 也支持切片赋值，用于等长的一段：`xs[1:3] = [8, 9];`。带步长的列表切片不能赋值（编译器会指出这一点，并建议逐个存储元素）。

## 逐元素运算与广播

点运算符对数组逐元素运算：算术运算得到元素类型的数组，比较运算得到 `bool` 数组。任一侧为标量时，它作用于每个元素。

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

两个秩相同的数组必须逐轴一致，只有长度为 1 的轴会拉伸到另一侧的长度。这是 NumPy 的规则，也是唯一的规则：它位于 core 中（`broadcast_shape`），所以所有数组类型的行为一致。其他情况会触发 panic，并给出两个形状。

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

秩较低的数组与另一个数组的最后几个轴对齐，就像它前面有长度为 1 的轴，因此向量可以加到矩阵的每一行上。结果取较高的秩。

```nabi
use core::array::Array;

let m: Array<f64, 2> = [[1, 2], [3, 4]];
let v: Array<f64, 1> = [10, 20];
println(f"{m .+ v:?}");
```

```output
[[11.0, 22.0], [13.0, 24.0]]
```

一整串点运算符作为一个循环执行，不产生中间数组，`Array` 和 `List` 都是如此：`a .* x .+ b .* 2.0` 对 `a`、`x`、`b` 的每个元素只读一次，结果只写一次。

## 原地更新

`y .= expr` 把 `expr` 的元素写入 `y`，`y .+= x`、`.-=`、`.*=`、`./=`、`.%=`、`.**=` 则把 `y` 与 `x` 逐元素组合。整个过程不分配内存：循环直接写入 `y`。目标可以是视图，因此数组的切片或某一列可以就地更新。

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

目标保持自己的形状：右侧拉伸到目标的形状，反之则不行。当右侧读取目标自身其他位置的元素时（`v .= v[::-1]`），它读的是事先取得的副本，所以结果与先算出所有元素再统一存储相同。其他运算符用 `.=` 写：`xs .= xs .%% 3`。

## 缺失值

元素类型 `T | none` 表示缺失值。`a .?? b` 用 `b` 中对应的元素填充每个 `none`，`b` 是单个值时就用 `b` 本身填充。NaN 不是 `none`：它是一个浮点值，会原样传递。要把 NaN 当作缺失值，用 `f64::is_nan.(a)` 选出它们再赋值。

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

## 归约与循环

`sum()` 把所有元素相加，`map(f)` 构造一个形状相同的新数组，`to_list()` 按行优先顺序（最后一个轴变化最快）给出元素，`for x in a` 也按这个顺序遍历。

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

每个归约都有一个 `_along(axis)` 形式，沿一个轴归约每条线并去掉该轴，类似 NumPy 的 `axis=` 参数：`m.sum_along(0)` 对矩阵的各列求和，`m.sum_along(1)` 对各行求和。可用的有 `sum`、`prod`、`mean`、`var` 和 `std`（带 `ddof`）、`min`、`max`、`argmin`、`argmax`、`cumsum` 和 `cumprod`，以及用于 `a .> 0.0` 这类布尔数组的 `any`、`all` 和 `count`。出现 NaN 时，`min`、`max`、`sum` 和 `mean` 的结果为 NaN；`nan` 形式（`nanmax`、`nanmean`……）会跳过它。求和采用两两相加，因此长数组的舍入误差也很小。

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

`concat(parts, axis)` 沿某个轴连接数组，`stack(parts, axis)` 把形状相同的数组放到一个新轴上，`where(mask, a, b)` 在掩码为 true 处取 `a` 的元素，其余位置取 `b` 的元素。`sorted_along(axis)` 对每条线排序（NaN 排在最后），`a.all_close(b)` 以相对容差比较两个浮点数组，就像 `x.is_close(y)` 比较两个数：`0.1 + 0.2 == 0.3` 为 `false`，而 `(0.1 + 0.2).is_close(0.3)` 为 `true`。

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

## 浮点数、复数与随机数

`num::FloatingPoint` 是同时适用于 `f32` 和 `f64` 的代码所用的约束：它把两者的常量（`T::NAN`、`T::PI`）和数学函数（`sqrt`、`exp`、`sin`……）合为一个 trait。`num::Num` 是更宽的约束，所有整数和浮点类型都满足：`+ - * /`、`0`、`1` 和比较。浮点列表和迭代器有 `min`、`max` 和 `sum`；元素中有 NaN 时，`min` 和 `max` 为 NaN，`sum` 以固定顺序两两相加，因此结果与平台无关。浮点数只是部分有序，所以用 `total_cmp` 排序。`num::linspace(a, b, n)` 给出从 `a` 到 `b` 的 `n` 个等间距值，`num::arange(a, b, step)` 从 `a` 开始按步长递增到 `b`，不包括 `b`。

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

`total_cmp` 是 IEEE 754 全序：符号位为 1 的 NaN 排在最前，符号位为 0 的排在最后。`f64::NAN` 和 `f32::NAN` 在每个目标上都是符号位为 0 的 quiet NaN，所以排在 `inf` 之后；算术运算产生的 NaN（`0.0 / 0.0`）符号位可能为任一值，x86 会把它置 1。

```nabi
let ys = [1.0, f64::NAN, f64::NEG_INFINITY, -0.0, f64::INFINITY];
println(f"{ys.sorted_by(|a, b| a.total_cmp(b)):?}");
println(f"{f64::NAN.is_sign_negative()} {f64::NAN.to_bits():x}");
```

```output
[-inf, -0.0, 1.0, inf, NaN]
false 7ff8000000000000
```

浮点运算在每个目标上都得到相同的位。每次运算都舍入到最近值，平局时取偶数；`a * b + c` 舍入两次（只有 `a.mul_add(b, c)` 会融合），保留次正规数，`min` / `max` 跳过 NaN 并把 `-0.0` 视为小于 `0.0`。运算产生哪个 NaN 是唯一留给 CPU 决定的事，如上面的符号所示；`sin`、`exp` 等库函数的最后一位也可能不同。

### 16 位浮点数：`f16` 与 `bf16`

`f16`（IEEE 半精度，最大 65504）和 `bf16`（bfloat16：`f32` 的范围加 8 位精度）是存储类型：存放权重或像素的 `Array<f16, 2>` 占用的内存只有 `f32` 的一半。在它们的位置上的字面量，或带后缀 `f16` / `bf16` 的字面量，从十进制只舍入一次；`as` 可以与所有数字类型相互转换；`+ - * /` 在 `f32` 中计算后再舍入回来，结果正是精确的 IEEE 结果。打印时输出能读回同一个值的最短位数。

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

运算符以外的数学运算需要先转换：`h.to_f32().sin()`。字面量模式不能匹配 16 位浮点数；请在守卫中比较（`x if x == 1.0 =>`）。

`num::Complex<T>` 是基于 `f32` 或 `f64` 的复数，后缀 `i` 用来写虚部：`3.0 + 4.0i`。它支持算术运算符，任一侧都可以是实数，还有 `abs`、`arg`、`conj`、`exp`、`ln`、`sqrt` 和 `pow`。

```nabi
let z = 3.0 + 4.0i;
println(f"{z} {z.abs()} {z * 2i} {z.conj()}");
```

```output
3+4i 5 -8+6i 3-4i
```

`std::random` 提供一次调用的形式（`random::range(1..7)`、`random::f64()`、`random::normal()`、`random::choice(xs)`、`random::shuffle(xs)`），它们取自同一个共享生成器，该生成器只从操作系统取一次种子，所以在循环中调用也很快。需要可重复的运行时，用 `seed` 创建一个 `Xoshiro256PlusPlus`：相同的种子在每个目标和每个版本上都给出相同的数。它的方法来自 `Rng` trait：适用于任意整数类型的 `range`（`0..n`、`1..=6`、`0u8..=255`），用于浮点数的 `uniform`，分布 `normal`、`exponential`、`gamma`、`beta`、`poisson` 和 `binomial`，按权重选取索引的 `weighted`，`choice`、`sample` 和 `shuffle`，以及一次填满整个 `Array` 的 `normal_array(shape)` / `random_array(shape)`。`split()` 为另一个线程创建独立的生成器。这些都不能用于机密：密钥和令牌应来自 `random::bytes` 或 `OsRng`，它们读取操作系统的安全随机源。

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

## 内存与速度

数组的元素紧凑地存放在一个缓冲区中。当元素类型不含引用时（数字、`bool`、`char`、`#[derive(Pod)]` 结构体），垃圾回收器从不查看其内部，所以即使是巨大的数组，回收时也没有任何开销。8 KiB 及以上的缓冲区按 64 字节对齐，并且在数组可达期间从不移动，这正是向量代码和 BLAS 等 C 库所需要的：`unsafe { a.as_ptr() }` 给出第一个元素，`strides()` 说明如何遍历其余元素。较小的缓冲区在你持有 pin 期间保持不动：`bare::gc::pin(a)` 返回一个守卫，其 `as_data_ptr()` 是第一个元素。`is_contiguous()` 判断元素是否按行优先顺序相邻；`copy()` 会使它们相邻。

`Array::from_list` 和 `to_list()` 会复制。若要共享一个缓冲区，`Array::view_list(xs, shape)` 把列表的元素视为数组，`a.as_list()` 在连续数组的缓冲区上给出一个列表（数组是带步长或偏移的视图时为 `none`：这时请用 `to_list()`）。通过任一侧写入，另一侧都能看到，直到列表增长并移到自己的缓冲区为止。

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

列表上的循环可能已经在用向量指令运行：编译器会自动向量化简单的逐元素循环。想要明确指定向量的位置时，`core::simd` 提供 `Simd<T, N>`，即数字类型的 `N` 个通道，别名有 `f32x4`、`f64x2`、`i32x8` 等。它的运算符同时作用于所有通道，点比较给出 `Mask`，`select` 按掩码选取通道。

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

在 `x86_64`、`aarch64`、`wasm32` 以及其他有向量单元的目标上，算术、比较和 `select` 在任何优化级别都是向量指令，在其他目标上则是遍历各通道的循环，结果相同。整数通道溢出时回绕而不是触发 panic，因为向量指令不做任何检查。完整列表见[库规范 2.24](../spec/refs-standard-libraries/200-core/224-simd.html)。

## 自定义数组类型

索引语法并非 core 的 `Array` 专用。带多个部分的方括号会以元组调用 `Index`（读）或 `IndexSet`（写），切片部分以 `core::index::SliceArg` 传入。如果你的类型实现了标记 trait `AxisIndexed`，每个整数部分都以 `isize` 传入，`none` 以 `NewAxis` 传入，`..` 以 `Rest` 传入，于是一个 impl 就能覆盖所有索引写法。

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

定义数值类型的包还可以为它提供标量在左侧的运算符。孤儿规则（[第 6 章](06-structs-enums-traits.html)）允许你为 core 中的类型实现 core 中的 trait，只要该 trait 的某个类型实参是你的类型，因此 `2.0 * v` 可以工作：

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

该规则仍然拒绝 `impl<T> Mul<V2> for T`，因为裸类型参数出现在你的类型之前：其他包也可能为自己的类型写出同样的 impl。

要让你的类型上的点表达式像 `Array` 那样融合，请实现 `Broadcastable`（把它的元素表示为 `Array`）和 `Materialize`（用结果的元素构造你的类型）。把数据存放在 `Array` 中的类型直接返回它：

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

当你的类型自己的点运算符会改变类型时，就像物理量那样（长度除以时间是速度），表达式会先通过这些运算符确定类型，再由它们给出的类型构造结果：循环仍是一个循环，类型仍受检查。包装单个数字的类型实现 `BroadcastScalar`（给出其普通值），就能在这样的表达式中作为标量出现：长度数组和时间数组配上速度 `v0`，写成 `d ./ t .+ v0`。

## 字面量后缀

包可以在处理函数上用 `#[literal(number "suffix")]` 为数字定义自己的后缀。处理函数为 `pub` 时，所有依赖你的包的包也会获得这个后缀，无需 `use`：

```nabi norun
// 在 `units` 包中
pub struct Meters { pub v: f64; }

#[literal(number "m")]
pub fn meters(n: f64) -> Meters { Meters { v: n } }
```

```nabi norun
// 在带有 `"units": Dep::path("../units")` 的包中
let d = 9.81m;          // units::meters(9.81)
println(f"{d.v}");
```

不是 `pub` 的处理函数只在自己的包内有效。两个依赖注册了相同的后缀时，该后缀的使用会产生歧义；请给其中一个设置 `priority`。
