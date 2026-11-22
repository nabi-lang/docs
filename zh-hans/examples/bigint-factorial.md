# 大整数与阶乘

用任意精度的 `int` 计算 100!、第 300 个斐波那契数和超过 128 位的幂，并找出 `i64` 在哪里溢出。

## 代码

```nabi
// n! as an arbitrary-precision `int`.
fn factorial(n: i64) -> int {
    let acc: int = 1;
    for k in 2..=n { acc = acc * int::from(k); }
    acc
}

// n! in i64: none once it overflows.
fn factorial_i64(n: i64) -> i64 | none {
    let acc = 1;
    for k in 2..=n {
        acc = acc.checked_mul(k)?;
    }
    acc
}

fn fib(n: i64) -> int {
    let a: int = 0;
    let b: int = 1;
    for _ in 0..n { (a, b) = (b, a + b); }
    a
}

fn digit_sum(x: int) -> i64 {
    let sum = 0;
    for c in f"{x}".chars() { sum += (c as i64) - ('0' as i64); }
    sum
}

let f100 = factorial(100);
let text = f"{f100}";
println(f"100! = {text[:20]}...{text[-10:]}");
println(f"digits: {text.len()}, digit sum: {digit_sum(f100)}");
println(f"bits: {f100.bit_length()}");

// Where does i64 give up?
for n in 19..=22 {
    match factorial_i64(n) {
        v: i64 => println(f"{n}! fits in i64: {v}"),
        none => println(f"{n}! overflows i64, int says {factorial(n)}"),
    }
}

println(f"fib(300) = {fib(300)}");
println(f"fib(300) % 1_000_000_007 = {fib(300) % int::from(1_000_000_007)}");

// Literals larger than i64 work directly in an int slot.
let big: int = 123_456_789_012_345_678_901_234_567_890;
println(f"big * big = {big * big}");
println(f"2^128 = {int::from(2).pow(128)}");
println(f"gcd(100!, 2^200) = 2^{int::from(2).pow(200).gcd(f100).bit_length() - 1}");
println(f"100! / 98! = {f100 / factorial(98)}");
```

```output
100! = 93326215443944152681...0000000000
digits: 158, digit sum: 648
bits: 525
19! fits in i64: 121645100408832000
20! fits in i64: 2432902008176640000
21! overflows i64, int says 51090942171709440000
22! overflows i64, int says 1124000727777607680000
fib(300) = 222232244629420445529739893461909967206666939096499764990979600
fib(300) % 1_000_000_007 = 644264086
big * big = 15241578753238836750495351562536198787501905199875019052100
2^128 = 340282366920938463463374607431768211456
gcd(100!, 2^200) = 2^97
100! / 98! = 9900
```

## 讲解

- `int` 是没有大小限制的整数。小的值存放在单个 `i64` 中，只在溢出时才换成数字数组，所以对小数字来说，它的速度和定宽整数差不多（[2.1.2 `int`](../spec/refs-standard-libraries/200-core/201-numbers.html#212-int任意精度整数)，[`Int`](../../std/core/num/bigint.html#struct.Int)）。
- `int` 和 `i64` 不能在算术中混用。要像 `int::from(k)` 这样显式转换，所以从代码中一眼就能看出哪些运算可能溢出（[2.1.6 算术结果类型](../spec/refs-standard-libraries/200-core/201-numbers.html#216-算术结果类型)）。
- 在 `i64` 这一边，`checked_mul` 溢出时返回 `none`。后缀 `?` 把这个 `none` 直接从函数中返回，所以 `factorial_i64` 以 `i64 | none` 报告结果（[和类型](../learn/05-sum-types.html)）。
- 超出 `i64` 范围的整数字面量可以直接写在 `int` 位置上（`let big: int = 123_456_...`）。
- `int` 实现了 `Display`，所以 `f"{x}"` 给出它的十进制字符串，字符串切片 `text[:20]` 和 `text[-10:]` 只显示开头和结尾（[字符串](../learn/09-strings.html)）。
