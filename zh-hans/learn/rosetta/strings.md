# 文本

字符串的常见任务。每个代码块都是完整的脚本，其下方的输出就是它打印的内容。

## 反转字符串

`reversed` 反转的是字符而不是字节，所以带重音符号的字母不会被破坏。对于一个可见字符由多个码点组成的文本（旗帜、家庭 emoji），应改为反转字素。

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

## 是不是回文？

只比较字母，并统一大小写：“A man, a plan, a canal: Panama!” 正读反读都一样。

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

## 统计文本中的单词

`split_whitespace` 按任意连续的空格、制表符和换行符拆分；`counts` 统计单词，`most_common` 给它们排名。

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

## 凯撒密码

把每个字母移动 `k` 位，从 `z` 回绕到 `a`；其他字符保持不变。`as u32` 和 `char::from_u32` 在字符和它的编码之间转换。

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

## 按列填充与对齐

格式说明符设置每个占位的宽度和对齐方式：`<` 左对齐，`>` 右对齐，`^` 居中，还可以指定填充字符。

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

## 拆分、修剪与连接

解析一行以 `;` 分隔的 `key = value` 对，再用另一种格式写回。

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

## 改变大小写

`to_uppercase`、`to_lowercase` 和 `to_titlecase` 遵循 Unicode；`casefold` 用于不区分大小写的比较，它把 `ß` 折叠为 `ss`。

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

## 查找模式的所有匹配

对于固定文本，`find` 和 `count` 就够了。对于模式，官方包 `regex` 以线性时间匹配，不做回溯；和所有官方包一样，它在第一次使用时自动添加。

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
