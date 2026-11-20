# 텍스트

문자열로 하는 흔한 작업입니다. 각 블록은 완전한 스크립트이고, 아래의 출력이 그 스크립트가 출력하는 내용입니다.

## 문자열 뒤집기

`reversed`는 바이트가 아니라 문자를 뒤집으므로 악센트가 붙은 글자도 깨지지 않습니다. 눈에 보이는 한 글자가 코드 포인트 여러 개로 된 텍스트(국기, 가족 이모지)라면 대신 grapheme을 뒤집으세요.

```nabi
println("stressed".reversed());
println("café".reversed());

let s = "a🇰🇷b";
println(s.graphemes().rev().collect::<str>());
```

```output
desserts
éfac
b🇰🇷a
```

## 회문인가?

글자만 대소문자를 하나로 맞춰 비교합니다. "A man, a plan, a canal: Panama!"는 앞뒤로 읽어도 같습니다.

```nabi
fn is_palindrome(s: str) -> bool {
    let letters = s.chars().filter(|c| c.is_alphanumeric()).map(|c| c.to_lowercase()).to_list().join("");
    letters == letters.reversed()
}

for s in ["A man, a plan, a canal: Panama!", "racecar", "nabi"] {
    println(f"{s:?}: {is_palindrome(s)}");
}
```

```output
"A man, a plan, a canal: Panama!": true
"racecar": true
"nabi": false
```

## 텍스트의 단어 세기

`split_whitespace`는 연속된 공백, 탭, 줄바꿈 어디서든 나눕니다. `counts`는 단어를 세고 `most_common`은 순위를 매깁니다.

```nabi
let text = "the quick brown fox jumps over the lazy dog the end";
let words = text.split_whitespace().to_list();
println(f"{words.len()} words, {words.counts().len()} different");
for (w, n) in words.counts().most_common(2) {
    println(f"{w}: {n}");
}
```

```output
11 words, 9 different
the: 3
quick: 1
```

## 카이사르 암호

글자마다 `k`칸씩 밀고, `z` 다음은 `a`로 돌아갑니다. 글자가 아닌 것은 그대로 둡니다. `as u32`와 `char::from_u32`는 문자와 그 코드 사이를 변환합니다.

```nabi
fn shift(s: str, k: u32) -> str {
    let out = StrBuf::new();
    for c in s.chars() {
        let base = if c >= 'a' && c <= 'z' { 'a' as u32 } else if c >= 'A' && c <= 'Z' { 'A' as u32 } else { 0 };
        if base == 0 {
            out.push(c);
        } else {
            out.push(char::from_u32((c as u32 - base + k) % 26 + base) ?? c);
        }
    }
    out.to_str()
}

let secret = shift("Veni, vidi, vici.", 3);
println(secret);
println(shift(secret, 23));
```

```output
Yhql, ylgl, ylfl.
Veni, vidi, vici.
```

## 열에 맞춰 채우고 정렬하기

서식 지정자는 구멍마다 너비와 정렬을 정합니다. `<`는 왼쪽, `>`는 오른쪽, `^`는 가운데이고, 채울 문자를 고를 수도 있습니다.

```nabi
let rows = [("apples", 3, 0.5), ("figs", 12, 2.25), ("pears", 7, 1.0)];
println(f"{"item":<8}{"qty":>5}{"price":>8}");
println("-".repeat(21));
for (name, qty, price) in rows {
    println(f"{name:<8}{qty:>5}{price:>8.2}");
}
println(f"{" total ":=^21}");
```

```output
item      qty   price
---------------------
apples      3    0.50
figs       12    2.25
pears       7    1.00
======= total =======
```

## 나누고, 다듬고, 잇기

`;`로 구분한 `key = value` 쌍의 줄을 파싱한 다음, 다른 형식으로 다시 씁니다.

```nabi
let line = " host = example.com ; port= 8080;  user =ada ";
let pairs: List<(str, str)> = [];
for part in line.split(";") {
    if let (k, v) = part.split_once("=") {
        pairs.push((k.trim(), v.trim()));
    }
}
println(pairs);
println(pairs.map(|(k, v)| f"{k}:{v}").to_list().join(","));
```

```output
[("host", "example.com"), ("port", "8080"), ("user", "ada")]
host:example.com,port:8080,user:ada
```

## 대소문자 바꾸기

`to_uppercase`, `to_lowercase`, `to_titlecase`는 유니코드를 따릅니다. `casefold`는 대소문자를 무시하고 비교할 때 쓰며, `ß`를 `ss`로 접습니다.

```nabi
let s = "hello, nabi world";
println(s.to_uppercase());
println(s.to_titlecase());
println("STRASSE".casefold() == "Straße".casefold());

// snake_case to camelCase
let parts = "max_line_length".split("_").to_list();
let camel = parts[0] + parts[1:].map(|p| p.to_titlecase()).to_list().join("");
println(camel);
```

```output
HELLO, NABI WORLD
Hello, Nabi World
true
maxLineLength
```

## 패턴과 맞는 곳 모두 찾기

고정된 텍스트라면 `find`와 `count`로 충분합니다. 패턴이라면 공식 패키지 `regex`가 백트래킹 없이 선형 시간에 매칭합니다. 다른 공식 패키지처럼 처음 쓸 때 추가됩니다.

```nabi
let log = "error at 10:42, warning at 11:05, error at 13:17";
println(f"{log.count("error")} errors, first at byte {log.find("error") ?? 0}");

let re = try regex::Regex::new(r"(?<h>\d\d):(?<m>\d\d)");
for c in re.captures_iter(log) {
    println(f"{c["h"]}h {c["m"]}m");
}
println(re.replace_all(log, "${h}.${m}"));
```

```output
2 errors, first at byte 0
10h 42m
11h 05m
13h 17m
error at 10.42, warning at 11.05, error at 13.17
```
