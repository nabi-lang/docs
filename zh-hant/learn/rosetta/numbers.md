# 數字

算術題目，從 FizzBuzz 到大整數。整數型別有固定的大小（`i32`、`i64`、`u8` 等），溢位是錯誤，而不是無聲的環繞；`int` 則沒有大小限制。

## FizzBuzz

對兩個條件組成的元組做 `match`，讀起來就像規則本身。

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

## 費氏數列

一個無止盡產生數列的產生器；`take` 讓它停下。`yield fn` 回傳一個迭代器，走訪其主體所 yield 的值。

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

## 階乘與大整數

`20!` 還能放進 `i64`；`30!` 就放不下了。`int` 會視需要增長，而且在值很小時與 `i64` 一樣快。

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

## 質數：埃拉托斯特尼篩法

劃掉每個質數的倍數，剩下的就是質數。`List::filled(n, x)` 建立一個由 `n` 個 `x` 組成的串列。

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

## 最大公因數與最小公倍數

以遞迴寫出的歐幾里得演算法。

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

## 羅馬數字

貪婪法：一再取出能放得下的最大值。同一份配對串列驅動兩個方向的轉換。

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

## 安全地剖析數字

`parse` 遇到不正確的輸入時會擲回錯誤，所以錯誤的值絕不會無聲地變成零。`try?` 搭配 `??` 則可以改用預設值。

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

## 浮點數的捨入與格式化

`round`、`floor`、`ceil` 與 `trunc` 會捨入為整數值；格式規範只捨入印出的內容。比較浮點數時要使用容許誤差，算術運算之後絕不使用 `==`。

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

## 亂數

`Xoshiro256PlusPlus::seed` 提供可重現的產生器：相同的種子在每台機器上都會產生相同的數字。`random::range` 與其他單次呼叫的形式，使用以系統提供的種子初始化的產生器。

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
