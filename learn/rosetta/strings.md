# Text

Common tasks with strings. Each block is a complete script; the output under it is what it prints.

## Reverse a string

`reversed` reverses the characters, not the bytes, so accented letters survive. For text where one visible character is several code points (a flag, a family emoji), reverse the graphemes instead.

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

## Is it a palindrome?

Compare the letters only, folded to one case: "A man, a plan, a canal: Panama!" reads the same both ways.

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

## Count the words in a text

`split_whitespace` splits on any run of spaces, tabs and newlines; `counts` tallies the words and `most_common` ranks them.

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

## Caesar cipher

Shift each letter by `k` places, wrapping from `z` to `a`; leave everything else alone. `as u32` and `char::from_u32` convert between a character and its code.

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

## Pad and align in columns

Format specifiers set the width and alignment of each hole: `<` left, `>` right, `^` centre, with an optional fill character.

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

## Split, trim and join

Parse a line of `key = value` pairs separated by `;`, then write it back in another format.

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

## Change case

`to_uppercase`, `to_lowercase` and `to_titlecase` follow Unicode; `casefold` is for comparing without case, and folds `ß` to `ss`.

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

## Find all matches of a pattern

For fixed text, `find` and `count` are enough. For a pattern, the official `regex` package matches in linear time, with no backtracking; like every official package, it is added on first use.

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
