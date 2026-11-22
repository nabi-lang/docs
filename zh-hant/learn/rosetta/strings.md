# 文字

與字串有關的常見題目。每個區塊都是完整的指令碼；下方的輸出就是它印出的內容。

## 反轉字串

`reversed` 反轉的是字元而不是位元組，所以帶重音的字母能完整保留。若文字中一個可見字元由多個碼位組成（國旗、家庭表情符號），請改為反轉字素。

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

## 是不是迴文？

只比較字母，並統一成同一種大小寫：「A man, a plan, a canal: Panama!」正讀反讀都一樣。

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

## 計算文字中的單字

`split_whitespace` 以任何連續的空格、定位字元與換行來分割；`counts` 統計單字，`most_common` 則為它們排名。

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

## 凱撒密碼

把每個字母移動 `k` 個位置，從 `z` 繞回 `a`；其他字元保持不變。`as u32` 與 `char::from_u32` 在字元與其碼位之間轉換。

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

## 填補並對齊欄位

格式規範設定每個欄位的寬度與對齊方式：`<` 靠左、`>` 靠右、`^` 置中，還可以指定填補字元。

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

## 分割、修剪與合併

剖析一行以 `;` 分隔的 `key = value` 配對，再用另一種格式寫回去。

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

## 轉換大小寫

`to_uppercase`、`to_lowercase` 與 `to_titlecase` 遵循 Unicode；`casefold` 用於不分大小寫的比較，並把 `ß` 轉為 `ss`。

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

## 找出模式的所有符合項目

固定文字用 `find` 與 `count` 就夠了。若是模式，官方的 `regex` 套件能以線性時間比對，不需要回溯；與所有官方套件一樣，它會在第一次使用時自動加入。

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
