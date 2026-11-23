# 21. Arrays and Numerics

Numeric work in Nabi has three layers. Plain numbers have the usual operators plus `**` for powers. `List<T>` and fixed arrays take the element-wise dot operators (`.+`, `.*`, ...) from [chapter 8](08-collections-iterators.html). For data with more than one axis, `core::array::Array<T, N>` is an n-dimensional array with NumPy-style indexing, slicing and broadcasting, built into core so that every library that does linear algebra, statistics or plotting can share one type.

## Powers

`a ** b` raises `a` to `b`. It binds tighter than a prefix `-` and groups to the right, as in mathematics and Python: `-x ** 2` is `-(x ** 2)`, and `2 ** 3 ** 2` is `2 ** 9`. Parenthesize to raise a negative number.

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

An integer base takes a `u32` exponent and gives an integer of its own type; overflow panics like `*` does. A float base takes a float exponent, or an `i32` one, which uses repeated multiplication. The exponent does not have to be the base's type: `n ** 2` with an `i64` `n` is fine, and the literal becomes a `u32`. `2 ** -1` is a compile error, since the answer is a fraction; the error says to write `2.0 ** -1.0`. `**=` raises in place, and `.**` is the element-wise form.

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

`^` stays the bitwise XOR it is in C and Rust.

## Making arrays

`Array<T, N>` holds elements of type `T` along `N` axes; `N` is the rank, part of the type. `Vector<T>` and `Matrix<T>` name ranks 1 and 2. Where an `Array` is expected, a nested list literal builds one: the nesting depth is the rank and the rows give the shape.

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

The integer literals became `f64`s because the slot says `f64`. Rows of different lengths are a compile error (E0142): an array is a rectangle, so keep ragged data in a `List<List<T>>`.

The constructors take the shape as an array of lengths:

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

`Array::full(shape, v)` repeats one value, and `Array::from(xs)` turns a list into a rank-1 array.

The fixed-size `[T; N]` has its own repeat form, `[v; N]`: `v` is evaluated once and each element gets its own copy. A plain value such as `0.0` is copied as it is; an element that holds a collection is cloned, so `[List::<f64>::new(); 4]` is four separate lists and `[[0.0; 3]; 3]` three separate rows ([chapter 2](02-values-and-types.html)). `array::from_fn(f)` builds a `[T; N]` whose element `i` is `f(i)`, called in order; `N` comes from the expected type or `array::from_fn::<T, N>`. Each element is its own value, so it also builds arrays `[v; N]` cannot: elements that are not `Clone`, or resources with one owner.

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

## Indexing and slicing

Inside the brackets, each comma-separated part is one axis. A position picks one index along its axis and removes that axis; a slice `a:b:k` keeps the axis with the elements it selects, by the same rules as a list slice (negative bounds from the end, clamping, a negative step walks backwards). As many positions as the rank give one element.

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

Two more parts help with shapes. `..` stands for all the axes the other parts leave, so `a[.., 0]` is the first column of an array of any rank. `none` inserts an axis of length 1, which is how a vector becomes a row or a column.

```nabi
use core::array::Array;

let a: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
let v: Array<i64, 1> = [7, 8, 9];
println(f"{a[.., 0]:?} {v[none, :].shape():?} {v[:, none].shape():?}");
```

```output
[1, 4] [1, 3] [3, 1]
```

Positions can be any integer type; a `usize` loop counter works as well as a literal.

A slice, a position that leaves axes, `..`, `none`, `t()` and `reshape` all return *views*: new shapes and strides over the same elements, with no copy. Writing through a view writes the array.

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

Use `copy()` when you want an independent array. A boolean array of the same shape is a *mask*: `a[mask]` collects the elements where it is `true`, in order, as a new rank-1 array.

```nabi
use core::array::Array;

let a: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
println(f"{a[a .% 2 .== 0]:?}");
```

```output
[2, 4, 6]
```

## Assigning to a selection

Anything you can index, you can assign to. A single value fills the selection; an array copies in, stretched to the selection's shape (see broadcasting below); a mask assignment changes only the selected elements.

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

A `List` takes slice assignment too, for a run of the same length: `xs[1:3] = [8, 9];`. A list slice with a step cannot be assigned (the compiler says so and suggests storing elements one at a time).

## Element-wise operations and broadcasting

The dot operators work on arrays element by element: arithmetic gives an array of the element type, comparisons an array of `bool`. A scalar on either side applies to every element.

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

Two arrays of the same rank must agree axis by axis, except that an axis of length 1 stretches to the other side's length. That is NumPy's rule, and it is the only one: it lives in core (`broadcast_shape`) so every array type agrees. Anything else panics and names both shapes.

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

An array of lower rank lines up with the last axes of the other, as if it had leading axes of length 1, so a vector adds to every row of a matrix. The result has the higher rank.

```nabi
use core::array::Array;

let m: Array<f64, 2> = [[1, 2], [3, 4]];
let v: Array<f64, 1> = [10, 20];
println(f"{m .+ v:?}");
```

```output
[[11.0, 22.0], [13.0, 24.0]]
```

A whole chain of dot operators runs as one loop with no intermediate arrays, on an `Array` as on a `List`: `a .* x .+ b .* 2.0` reads each element of `a`, `x` and `b` once and writes the result once.

## Updating in place

`y .= expr` writes the elements of `expr` into `y`, and `y .+= x`, `.-=`, `.*=`, `./=`, `.%=`, `.**=` combine `y` with `x` element by element. Nothing is allocated: the loop writes straight into `y`. The target can be a view, so a slice or a column of an array is updated where it sits.

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

The target keeps its shape: the right side stretches to it, never the other way. When the right side reads the target's own elements at other positions (`v .= v[::-1]`), it reads a copy taken first, so the result is what you would get by computing every element before storing any. Other operators are written with `.=`: `xs .= xs .%% 3`.

## Missing values

An element type `T | none` marks missing values. `a .?? b` fills each `none` with the matching element of `b`, or with `b` itself when it is a single value. NaN is not `none`: it is a float value and passes through. To treat NaN as missing, select it with `f64::is_nan.(a)` and assign.

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

## Reductions and loops

`sum()` adds every element, `map(f)` builds a new array of the same shape, `to_list()` gives the elements in row-major order (the last axis fastest), and `for x in a` visits them in that order.

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

Every reduction also has an `_along(axis)` form that reduces each lane along one axis and drops it, like NumPy's `axis=` argument: `m.sum_along(0)` sums the columns of a matrix, `m.sum_along(1)` its rows. There are `sum`, `prod`, `mean`, `var` and `std` (with `ddof`), `min`, `max`, `argmin`, `argmax`, `cumsum` and `cumprod`, and `any`, `all` and `count` for boolean arrays such as `a .> 0.0`. A NaN makes `min`, `max`, `sum` and `mean` NaN; the `nan` forms (`nanmax`, `nanmean`, ...) leave it out. Sums are pairwise, so their rounding error stays small for long arrays.

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

`concat(parts, axis)` joins arrays along an axis, `stack(parts, axis)` puts arrays of one shape on a new axis, and `where(mask, a, b)` picks `a`'s element where the mask is true and `b`'s elsewhere. `sorted_along(axis)` sorts each lane (NaN last), and `a.all_close(b)` compares two float arrays with a relative tolerance, as `x.is_close(y)` does for two numbers: `0.1 + 0.2 == 0.3` is `false`, but `(0.1 + 0.2).is_close(0.3)` is `true`.

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

## Floats, complex numbers and random numbers

`num::FloatingPoint` is the bound for code over both `f32` and `f64`: their constants (`T::NAN`, `T::PI`) and math (`sqrt`, `exp`, `sin`, ...) as one trait. `num::Num` is the wider bound every integer and float type meets: `+ - * /`, `0`, `1` and comparison. Float lists and iterators have `min`, `max` and `sum`; a NaN among the items makes `min` and `max` NaN, and `sum` adds pairwise in a fixed order, so the result does not depend on the platform. Floats are only partly ordered, so sort them with `total_cmp`. `num::linspace(a, b, n)` gives `n` evenly spaced values from `a` to `b`, and `num::arange(a, b, step)` steps from `a` up to, not including, `b`.

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

`total_cmp` is the IEEE 754 total order: a NaN with the sign bit set comes first, one with it clear last. `f64::NAN` and `f32::NAN` are the quiet NaN with the sign bit clear on every target, so they sort after `inf`; a NaN that arithmetic makes (`0.0 / 0.0`) can have either sign, and x86 sets it.

```nabi
let ys = [1.0, f64::NAN, f64::NEG_INFINITY, -0.0, f64::INFINITY];
println(f"{ys.sorted_by(|a, b| a.total_cmp(b)):?}");
println(f"{f64::NAN.is_sign_negative()} {f64::NAN.to_bits():x}");
```

```output
[-inf, -0.0, 1.0, inf, NaN]
false 7ff8000000000000
```

Float arithmetic gives the same bits on every target. Each operation rounds to nearest, ties to even; `a * b + c` rounds twice (only `a.mul_add(b, c)` fuses), subnormal numbers are kept, and `min` / `max` skip a NaN and put `-0.0` below `0.0`. Which NaN an operation makes is the one thing left to the CPU, as the sign above shows; `sin`, `exp` and the other library functions may also differ in the last bit.

### 16-bit floats: `f16` and `bf16`

`f16` (IEEE half precision, up to 65504) and `bf16` (bfloat16: `f32`'s range with 8 bits of precision) are storage types: an `Array<f16, 2>` of weights or pixels takes half the memory of `f32`. A literal in their slot, or with the suffix `f16` / `bf16`, is rounded once from the decimal; `as` converts to and from every number type; `+ - * /` compute in `f32` and round back, which is the exact IEEE result. They print the shortest digits that read back as the same value.

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

For math beyond the operators, convert: `h.to_f32().sin()`. Literal patterns do not match a 16-bit float; compare in a guard (`x if x == 1.0 =>`).

`num::Complex<T>` is a complex number over `f32` or `f64`, and the `i` suffix writes the imaginary part: `3.0 + 4.0i`. It has the arithmetic operators, with a real number on either side, and `abs`, `arg`, `conj`, `exp`, `ln`, `sqrt` and `pow`.

```nabi
let z = 3.0 + 4.0i;
println(f"{z} {z.abs()} {z * 2i} {z.conj()}");
```

```output
3+4i 5 -8+6i 3-4i
```

`std::random` has one-call forms (`random::range(1..7)`, `random::f64()`, `random::normal()`, `random::choice(xs)`, `random::shuffle(xs)`) drawn from one shared generator seeded once from the operating system, so calling them in a loop is fast. For a run you can repeat, make a `Xoshiro256PlusPlus` with `seed`: the same seed gives the same numbers on every target and release. Its methods come from the `Rng` trait: `range` over any integer type (`0..n`, `1..=6`, `0u8..=255`), `uniform` for floats, the distributions `normal`, `exponential`, `gamma`, `beta`, `poisson` and `binomial`, `weighted` to pick an index by weight, `choice`, `sample` and `shuffle`, and `normal_array(shape)` / `random_array(shape)` to fill a whole `Array` at once. `split()` makes an independent generator for another thread. None of these are for secrets: keys and tokens come from `random::bytes` or `OsRng`, which read the operating system's secure source.

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

## Memory and speed

An array's elements sit in one packed buffer. When the element type holds no references (numbers, `bool`, `char`, `#[derive(Pod)]` structs), the garbage collector never looks inside it, so even a huge array costs a collection nothing. A buffer of 8 KiB or more is aligned to 64 bytes and never moves while the array is reachable, which is what vector code and C libraries such as BLAS want: `unsafe { a.as_ptr() }` gives the first element, and `strides()` says how to walk the rest. A smaller one stays put while you hold a pin: `bare::gc::pin(a)` gives a guard whose `as_data_ptr()` is the first element. `is_contiguous()` tells whether the elements are adjacent in row-major order; `copy()` makes them so.

`Array::from_list` and `to_list()` copy. To share one buffer instead, `Array::view_list(xs, shape)` views a list's elements as an array, and `a.as_list()` gives a list over a contiguous array's buffer (`none` when the array is a strided or offset view: use `to_list()` there). A write through either side shows in the other until the list grows and moves to a buffer of its own.

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

## SIMD vectors

Loops over lists may already run in vector instructions: the compiler vectorizes simple element-wise loops on its own. When you want to say where the vectors are, `core::simd` has `Simd<T, N>`, `N` lanes of a number type, with the aliases `f32x4`, `f64x2`, `i32x8` and so on. Its operators work on every lane at once, a dot comparison gives a `Mask`, and `select` picks lanes by one.

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

The arithmetic, comparisons and `select` are vector instructions on `x86_64`, `aarch64`, `wasm32` and the other targets with a vector unit, at every optimization level, and a loop over the lanes elsewhere, with the same result. Integer lanes wrap on overflow instead of panicking, since a vector instruction checks nothing. The full list is in the [library spec, 2.24](../spec/refs-standard-libraries/200-core/224-simd.html).

## Your own array type

The indexing syntax is not reserved for core's `Array`. A bracket with several parts calls `Index` (reads) or `IndexSet` (writes) with a tuple, and a slice part arrives as a `core::index::SliceArg`. If your type implements the marker trait `AxisIndexed`, every integer part arrives as an `isize`, `none` as `NewAxis` and `..` as `Rest`, so one impl covers every way of writing the index.

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

A package that defines a numeric type can also give it scalar-left operators. The orphan rule ([chapter 6](06-structs-enums-traits.html)) lets you implement a trait from core for a type from core when one of the trait's type arguments is yours, so `2.0 * v` works:

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

The rule still refuses `impl<T> Mul<V2> for T`, where a bare type parameter comes before your type: another package could write the same impl for its own type.

To make dot expressions over your type fuse like `Array`'s, implement `Broadcastable`, which shows its elements as an `Array`, and `Materialize`, which builds your type around a result's elements. A type that keeps its data in an `Array` returns it:

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

When your type's own dot operators change the type, as a quantity's do (a length over a time is a speed), the expression is first typed through those operators, and the type they give builds the result: the loop stays one loop and the types stay checked. A type that wraps one number implements `BroadcastScalar` (its plain value) so it can stand in such an expression as a scalar: `d ./ t .+ v0` with arrays of lengths and times and a speed `v0`.

## Literal suffixes

A package can give numbers a suffix of its own with `#[literal(number "suffix")]` on a handler function. When the handler is `pub`, every package that depends on yours gets the suffix too, without a `use`:

```nabi norun
// in the `units` package
pub struct Meters { pub v: f64; }

#[literal(number "m")]
pub fn meters(n: f64) -> Meters { Meters { v: n } }
```

```nabi norun
// in a package with `"units": Dep::path("../units")`
let d = 9.81m;          // units::meters(9.81)
println(f"{d.v}");
```

A handler that is not `pub` works only inside its own package. Two dependencies that register the same suffix make its uses ambiguous; give one a `priority`.

## Further reading

- Next chapter: [22. Science with Nabi](22-science.html)
