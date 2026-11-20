# 수

FizzBuzz부터 큰 정수까지, 산술 문제입니다. 정수 타입은 크기가 정해져 있고(`i32`, `i64`, `u8` ..), 오버플로는 조용히 감기지 않고 에러가 됩니다. `int`에는 크기 제한이 없습니다.

## FizzBuzz

두 조건의 튜플에 `match`를 쓰면 규칙 그대로 읽힙니다.

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

## 피보나치 수

수열을 끝없이 내놓는 제너레이터이고, `take`가 멈춥니다. `yield fn`은 본문이 내놓는 값들의 반복자를 돌려줍니다.

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

## 큰 정수로 팩토리얼 구하기

`20!`은 아직 `i64`에 들어가지만 `30!`은 들어가지 않습니다. `int`는 필요한 만큼 커지고, 값이 작은 동안에는 `i64`만큼 빠릅니다.

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

## 소수: 에라토스테네스의 체

소수마다 그 배수를 지워 나가면 남는 것이 소수입니다. `List::filled(n, x)`는 `x`를 `n`개 담은 리스트를 만듭니다.

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

## 최대공약수와 최소공배수

유클리드 호제법을 재귀로 씁니다.

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

## 로마 숫자

탐욕법입니다. 들어가는 가장 큰 값을 거듭 뺍니다. 쌍의 리스트 하나로 양방향 변환을 모두 합니다.

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

## 수를 안전하게 파싱하기

`parse`는 잘못된 입력에 에러를 던지므로, 잘못된 값이 조용히 0이 되는 일은 없습니다. `try?`와 `??`를 쓰면 대신 기본값을 줍니다.

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

## 부동소수점 수 반올림과 서식

`round`, `floor`, `ceil`, `trunc`는 정수 값으로 반올림합니다. 서식 지정자는 출력하는 값만 반올림합니다. 부동소수점 수는 계산한 뒤 `==`로 비교하지 말고 허용 오차로 비교합니다.

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

## 난수

`Xoshiro256PlusPlus::seed`는 재현 가능한 생성기를 줍니다. 같은 시드는 어느 컴퓨터에서나 같은 수를 냅니다. `random::range`를 비롯한 한 번 호출 형태는 시스템에서 시드를 받은 생성기를 씁니다.

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
