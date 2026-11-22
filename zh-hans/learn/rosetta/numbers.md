# 数字

算术任务，从 FizzBuzz 到大整数。整数类型的大小是固定的（`i32`、`i64`、`u8` 等），溢出是错误，而不是悄悄回绕；`int` 没有大小限制。

## FizzBuzz

对两个条件组成的元组做 `match`，读起来就和规则本身一样。

```nabi
for i in 1..=15 {
    let line = match (i % 3 == 0, i % 5 == 0) {
        (true, true) => "FizzBuzz",
        (true, false) => "Fizz",
        (false, true) => "Buzz",
        _ => i.to_str(),
    };
    print(line + " ");
}
println("");
```

```output
1 2 Fizz 4 Buzz Fizz 7 8 Fizz Buzz 11 Fizz 13 14 FizzBuzz 
```

## 斐波那契数列

一个永远产出数列的生成器；`take` 让它停下。`yield fn` 返回一个迭代器，遍历函数体产出的值。

```nabi
yield fn fib() -> u64 {
    let (a, b) = (0u64, 1u64);
    loop {
        yield a;
        (a, b) = (b, a + b);
    }
}

println(fib().take(15).to_list());
println(f"the first above a million: {fib().find(|x| x > 1_000_000) ?? 0}");
```

```output
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377]
the first above a million: 1346269
```

## 阶乘与大整数

`20!` 还能放进 `i64`；`30!` 就放不下了。`int` 按需增长，并且在值较小时和 `i64` 一样快。

```nabi
fn factorial(n: i64) -> int {
    let r: int = 1;
    for k in 2..=n { r = r * int::from(k); }
    r
}

println(factorial(20));
println(factorial(30));
println(f"100! has {factorial(100).to_str().len()} digits");
```

```output
2432902008176640000
265252859812191058636308480000000
100! has 158 digits
```

## 素数：埃拉托斯特尼筛法

划掉每个素数的倍数，剩下的就是素数。`List::filled(n, x)` 生成一个包含 `n` 个 `x` 的列表。

```nabi
fn primes(n: usize) -> List<usize> {
    let prime = List::filled(n + 1, true);
    prime[0] = false;
    prime[1] = false;
    let p = 2;
    while p * p <= n {
        if prime[p] {
            for m in (p * p..=n).step_by(p) { prime[m] = false; }
        }
        p += 1;
    }
    (0..=n).filter(|k| prime[k]).to_list()
}

println(primes(50));
println(f"{primes(1_000_000).len()} primes below a million");
```

```output
[2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]
78498 primes below a million
```

## 最大公约数与最小公倍数

递归实现的欧几里得算法。

```nabi
fn gcd(a: u64, b: u64) -> u64 { if b == 0 { a } else { gcd(b, a % b) } }
fn lcm(a: u64, b: u64) -> u64 { a / gcd(a, b) * b }

println(f"gcd(1071, 462) = {gcd(1071, 462)}");
println(f"lcm of 1..=20 = {(1u64..=20).fold(1u64, |acc, k| lcm(acc, k))}");
```

```output
gcd(1071, 462) = 21
lcm of 1..=20 = 232792560
```

## 罗马数字

贪心法：一次又一次地取能放下的最大值。同一个由数对组成的列表驱动两个方向的转换。

```nabi
const ROMAN: [(u32, str); 13] = [(1000, "M"), (900, "CM"), (500, "D"), (400, "CD"), (100, "C"), (90, "XC"), (50, "L"), (40, "XL"), (10, "X"), (9, "IX"), (5, "V"), (4, "IV"), (1, "I")];

fn to_roman(n: u32) -> str {
    let out = StrBuf::new();
    let left = n;
    for (v, s) in ROMAN {
        while left >= v {
            out.push_str(s);
            left -= v;
        }
    }
    out.to_str()
}

fn from_roman(s: str) -> u32 {
    let n = 0u32;
    let rest = s;
    for (v, sym) in ROMAN {
        while rest.starts_with(sym) {
            n += v;
            rest = rest[sym.len():];
        }
    }
    n
}

for n in [1994, 2026, 3999] {
    let r = to_roman(n);
    println(f"{n} = {r} = {from_roman(r)}");
}
```

```output
1994 = MCMXCIV = 1994
2026 = MMXXVI = 2026
3999 = MMMCMXCIX = 3999
```

## 安全地解析数字

`parse` 遇到错误的输入会抛出错误，所以错误的值永远不会悄悄变成零。`try?` 配合 `??` 则改为给出默认值。

```nabi
for s in ["42", "-7", "3.5", "12abc", "99999999999"] {
    let n = try s.parse::<i32>() catch e { println(f"{s:?}: {e}"); continue; };
    println(f"{s:?} -> {n}");
}
let port = try? "abc".parse::<u16>() ?? 8080;
println(port);
```

```output
"42" -> 42
"-7" -> -7
"3.5": cannot parse "3.5": invalid digit
"12abc": cannot parse "12abc": invalid digit
"99999999999": cannot parse "99999999999": number out of range
8080
```

## 浮点数的舍入与格式化

`round`、`floor`、`ceil` 和 `trunc` 舍入到整数值；格式说明符只对打印出来的内容做舍入。浮点数要在容差范围内比较，经过算术运算后永远不要用 `==` 比较。

```nabi
let x = 2.675;
println(f"{x.round()} {x.floor()} {x.ceil()} {(-x).trunc()}");
println(f"{x:.2} {1234567.891:,.1} {0.000123:e} {0.25:.0%}");
println(f"{0.1 + 0.2 == 0.3} {(0.1 + 0.2 - 0.3).abs() < 1e-12}");
println(f"{10.0 / 0.0} {(0.0 / 0.0).is_nan()} {f64::EPSILON:e}");
```

```output
3 2 3 -2
2.67 1,234,567.9 1.23e-4 25%
false true
inf true 2.220446049250313e-16
```

## 随机数

`Xoshiro256PlusPlus::seed` 给出可重现的生成器：相同的种子在每台机器上都给出相同的数。`random::range` 和其他一次调用的形式使用由系统提供种子的生成器。

```nabi
use std::random;
use std::random::{Rng, Xoshiro256PlusPlus};

let g = Xoshiro256PlusPlus::seed(2026);
let dice = (0..10).map([ref g] |_| g.range(1..=6)).to_list();
println(dice);
println(f"{g.uniform(-1.0, 1.0):.4} {g.normal():.4}");

let roll = random::range(1..=6);
println(roll >= 1 && roll <= 6);
```

```output
[4, 2, 2, 1, 1, 2, 5, 5, 3, 5]
0.8183 0.2643
true
```
