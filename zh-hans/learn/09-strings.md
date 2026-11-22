# 9. 字符串与 f-string

几乎每个程序都会用到字符串。本章介绍 `str` 和 `StrBuf`、常用方法、各种字面量，以及你一直在用的 f-string 的格式说明符。

## `str`：不可变的 UTF-8

`str` 是 UTF-8 字节序列，创建后就不会改变。因此 `len()` 返回的是**字节**数，而不是字符数。要统计字符（码位）数，使用 `char_count()`。

```nabi
let s = "hello, 세계";
println(f"{s.len()} bytes, {s.char_count()} chars");
println(s[0..5]);

for c in "가나".chars() {
    println(f"{c} = U+{c as u32:X}");
}
```

```output
13 bytes, 9 chars
hello
가 = U+AC00
나 = U+B098
```

这里特意用了韩文文本：每个韩文音节在 UTF-8 中占 3 个字节，所以 `"hello, 세계"` 有 13 个字节，却只有 9 个字符。

切片 `s[a..b]` 同样按字节位置切分。一个韩文字符占 3 个字节，所以只能在字符边界处切分；在字符内部切分会触发 panic，并指出出问题的字节：

```nabi build
let s = "hello, 세계";
println(s[0..8]);
```

```text
thread 'main' panicked at main.nabi:2:9:
string slice is not on a char boundary: byte 8 is inside `세` (bytes 7..10)
string offsets count bytes; to work by character, use `chars()`, `char_indices()` or `char_slice(a, b)`
to cut at a boundary, use `floor_char_boundary(i)`; `get(a..b)` gives `none` instead of a panic
```

要逐个字符处理，使用 `chars()` 或 `char_indices()`。当偏移量可能落在字符内部时，`get(a..b)` 返回 `none` 而不是触发 panic，`floor_char_boundary(i)` 把偏移量移回所在字符的开头，`char_slice(a, b)` 按字符而不是字节计数：

```nabi
let s = "hello, 세계";
println(f"{s.get(0..8) ?? "none"} {s[0:s.floor_char_boundary(8)]} {s.char_slice(7, 8)}");
```

```output
none hello,  세
```

单个字符是 `char`，用单引号书写。它保存一个 Unicode 标量值，并提供用于分类和转换的方法。

```nabi
let c = 'A';
println(f"{c} {c.is_ascii_uppercase()} {c.to_ascii_lowercase()}");
println(f"{'7'.is_ascii_digit()} {'세'.len_utf8()} {'\u{1F600}'}");
println(f"{'ß'.to_uppercase()} {'ß'.to_simple_uppercase()} {'7'.to_digit(10) ?? 0} {'x'.to_digit(10) is none}");
```

```output
A true a
true 3 😀
SS ß 7 true
```

大小写映射可能把一个字符变成多个，所以 `to_uppercase`、`to_lowercase` 和 `to_titlecase` 返回 `str`（`'ß'` 变成 `"SS"`）。需要恰好返回一个 `char` 时，使用 `to_simple_uppercase` 及其他 `to_simple_*` 形式，没有单字符映射的字符保持原样。对于不是该进制数字的字符，`to_digit` 返回 `none`。

## 常用方法

`str` 上的方法不修改原字符串，而是返回新值。可能什么也找不到的操作返回 `T | none`，所以和第 5 章的 `??` 及 `if let` 很搭配。

```nabi
let line = "  name = nabi  ";
let t = line.trim();
println(f"[{t}]");
println(f"{t.starts_with("name")} {t.contains("=")} {t.find("nabi") ?? 99}");

if let (k, v) = t.split_once(" = ") {
    println(f"key={k} value={v}");
}

let parts = "a,b,,c".split(",").to_list();
println(f"{parts:?}");
println(f"{"ab".repeat(3)} {t.to_uppercase()} {t.replace("nabi", "NABI")}");
println(f"{"stressed".reversed()} {"banana".count("an")} {"a-b-c".replacen("-", "+", 1)} {"hello nabi".to_titlecase()}");
println(42.to_str() + "px");

let n: i32 = try! "42".parse();
println(f"{n + 1}");
```

```output
[name = nabi]
true true 7
key=name value=nabi
["a", "b", "", "c"]
ababab NAME = NABI name = NABI
desserts 2 a+b-c Hello Nabi
42px
43
```

`reversed` 反转字符，`count` 统计不重叠的匹配次数，`replacen` 只替换前 `n` 个。`to_str()` 适用于所有 `Display` 值，就像 Python 的 `str(x)`。`parse` 是一个失败时会抛出错误的函数，所以示例加了 `try!`。错误在[下一章](10-errors.html)介绍。其他方法还有 `lines()`、`split_whitespace()`、`strip_prefix()` 和 `trim_start()`（[`str` API](../../std/core/str.html#impl.str)）。

## Unicode 文本：规范化与字素

同样的文本可以用不同的码位拼写：`"é"` 可以是一个码位，也可以是 `e` 后面跟一个组合重音符。`==` 比较的是字节，所以要比较或保存的文本先用 `normalize(Nfc)` 规范化。忽略大小写比较时，`casefold()`（或 `eq_ignore_case`）会把 `"ß"` 折叠为 `"ss"`，`to_lowercase` 则不会。读者看到的一个字符可能由多个码位组成；`graphemes()` 产出这些字素簇，`words()` 产出单词。

```nabi
let a = "caf\u{e9}";
let b = "cafe\u{301}";
println(f"{a == b} {a == b.normalize(Nfc)} {"Straße".eq_ignore_case("STRASSE")}");

let s = "e\u{301}🇰🇷👩\u{200D}💻";
println(f"{s.char_count()} chars, {s.grapheme_count()} graphemes");
println(f"{"can't stop, 3.14!".words().to_list():?}");
```

```output
false true true
7 chars, 3 graphemes
["can't", "stop", "3.14"]
```

`grapheme_slice(0, n)` 保留前 `n` 个字素，所以截短的标题永远不会把重音符或旗帜切成两半。规范化形式、折叠和分割规则见 [`str` 参考](../spec/refs-standard-libraries/200-core/215-string.html#21510-unicode-文本规范化大小写折叠分段)。

## `StrBuf`：逐段构建

由于 `str` 不可变，在循环中用 `+` 拼接每次都会创建一个新字符串。逐段构建字符串时，写入 `StrBuf` 缓冲区，完成后用 `to_str()` 取出结果。

```nabi
let b = StrBuf::new();
for i in 1..=5 {
    if !b.is_empty() { b.push_str(", "); }
    b.push_str(f"{i * i}");
}
b.push('!');
let s = b.to_str();
println(f"{s} ({s.len()} bytes)");
```

```output
1, 4, 9, 16, 25! (16 bytes)
```

只拼接一次两个字符串，用 `+` 就够了；它也可以追加一个 `char`（`s + '!'`），`s += '!'` 也一样。不过和 C 或 Python 不同，相邻的字符串字面量（`"abc" "def"`）不会自动连接。这样写是编译错误；要连接就写 `+`（[1.7.4](../spec/refs-lang/001-lexical/107-string-literals.html#174-相邻字符串的自动连接)）。

```nabi
let greeting = "hello, " + "world" + '!';
println(greeting);
```

```output
hello, world!
```

## 字面量：转义、原始字符串、续行

普通字符串会处理 `\n`、`\t`、`\"`、`\\` 和 `\u{..}` 这样的转义。对于满是反斜杠的路径或正则表达式，原始字符串 `r"..."` 更方便。如果其中需要 `"`，就在字符串两侧各加上相同数量的 `#`。

```nabi
println("tab:\t| quote:\" | snow: \u{2603}");
println(r"C:\path\to\file\n");
println(r#"she said "hi""#);
println("one long \
         line");
```

```output
tab:	| quote:" | snow: ☃
C:\path\to\file\n
she said "hi"
one long line
```

如最后一个示例所示，行尾的 `\` 会同时去掉换行符和下一行开头的空白。这条规则同样不适用于原始字符串（[1.7.2](../spec/refs-lang/001-lexical/107-string-literals.html#172-原始字符串)）。

## 字节串 `b"..."`

协议和文件格式使用的是字节，而不是文本。字节串 `b"..."` 是由其字节组成的定长数组 `[u8; N]`。ASCII 直接书写，其他内容写成 `\xHH`（任意 `\x00`-`\xFF` 的值）。在期望 `Slice<u8>` 的地方，同一个字面量是该数组的视图，所以可以直接传给接收字节的函数。

```nabi
let magic = b"\x7FELF";                  // [u8; 4]
println(f"{magic.len()} {magic:?}");

fn starts_with(data: Slice<u8>, prefix: Slice<u8>) -> bool {
    data.len() >= prefix.len() && data[:prefix.len()] == prefix
}
println(f"{starts_with(b"\x7FELF\x02\x01", b"\x7FELF")}");

let req: Slice<u8> = b"GET / HTTP/1.1\r\n";
println(f"{req[:3]:?} {try! str::from_utf8(req[:3])}");
```

```output
4 [127, 69, 76, 70]
true
[71, 69, 84] GET
```

在字节串中直接书写非 ASCII 字符，或使用超过 `\xFF` 的转义，都是编译错误；字节串不支持续行。

单个字节写作 `b'0'`：它是一个 `u8`，规则相同（`b'\n'`、`b'\xFF'`），在解析器比较字节时很方便。

```nabi
let digits = "a1b22".as_bytes().filter(|c| b'0' <= c <= b'9').count();
println(f"{b'0'} {digits}");
```

```output
48 3
```

## 嵌入文件：`include_str!` / `include_bytes!`

要把模板、测试数据或图标放进可执行文件，就在编译期嵌入该文件。`include_str!("path")` 是文件内容构成的 `str`，`include_bytes!("path")` 是文件字节构成的 `[u8; N]`，和字节串一样。路径相对于包含该调用的源文件，结果是一个常量，所以可以用来初始化 `const` 或 `static`。

```text
app/
└── src/
    ├── main.nabi
    └── page.html        // <h1>{title}</h1>
```

```nabi norun
// app/src/main.nabi
const PAGE: str = include_str!("page.html");
static LOGO: [u8; 1406] = include_bytes!("logo.png");

print(PAGE.replace("{title}", "Hello"));
```

文件不存在是编译错误；对 `include_str!` 来说，文件不是有效的 UTF-8 也是编译错误。修改被嵌入的文件会重新构建用到它的代码。

## f-string 的插值

`f"..."` 的 `{ }` 中可以放任意表达式。规则与 Python 3.12（PEP 701）相同：插值内部可以复用外层的引号字符，f-string 也可以嵌套。要写出字面的花括号，使用 `{{` 和 `}}`。

```nabi
let d = {"lang": "nabi"};
let words = ["a", "b"];
println(f"{d["lang"]} has {words.len()} words: {words[0] + words[1]}");
println(f"nested: {f"[{1 + 1}]"}");
println(f"{{literal braces}}");
```

```output
nabi has 2 words: ab
nested: [2]
{literal braces}
```

插值内部不使用 `\"` 这样的转义，直接写引号即可。跨多行的文本或插值，使用 `f"""..."""`。

## 格式说明符

在插值的表达式后面加上 `:` 来指定格式。语法是 `[fill][align][sign][#][0][width][grouping][.precision][type]`。

```nabi
let pi = 3.14159;
println(f"[{pi:.2}] [{pi:8.3}] [{pi:e}]");
println(f"[{42:>6}] [{42:<6}] [{42:^6}] [{42:*^6}]");
println(f"[{"ab":5}] [{"ab":>5}] [{7:03}] [{5:+}]");
println(f"{255:x} {255:#X} {5:b} {-1i8:x}");
println(f"{"hi":?} {[1, 2]:?}");
```

```output
[3.14] [   3.142] [3.14159e0]
[    42] [42    ] [  42  ] [**42**]
[ab   ] [   ab] [007] [+5]
ff 0xFF 101 ff
"hi" [1, 2]
```

和 Python 一样，`{x=}` 先打印表达式的文本，然后是 `=`，再是它的值（`{x = }` 会保留空格）。宽度后面的 `,` 或 `_` 把整数的数字每三位分组（`{1234567:,}` 是 `1,234,567`），类型 `%` 乘以 100 并加上百分号（`{0.256:.1%}` 是 `25.6%`）。

```nabi
let n = 1234567;
let rate = 0.256;
println(f"{n=} {n:,} {n:_} {rate:.1%} {n + 1 = }");
```

```output
n=1234567 1,234,567 1_234_567 25.6% n + 1 = 1234568
```

没有显式指定对齐时，数字右对齐，其他内容左对齐。`?` 选择 `Debug` 格式，它显示字符串时带上引号和转义。说明符在编译期按插值的类型检查，所以 `{s:x}`（`s: str`）这样的错误会立即报出。

`g` 类型是 C 和 Python 的通用格式：精度统计的是有效数字（默认 6 位），很大或很小的数会改用指数形式。`{1234.5678:.3g}` 是 `1.23e+03`，`{1e20:g}` 是 `1e+20`，`{0.5:g}` 是 `0.5`。

要在运行时选择宽度或精度，在说明符中再放一个插值。`{..}` 和 `${..}` 都可以（[动态格式说明符](../spec/refs-lang/001-lexical/107-string-literals.html#动态格式说明符)）。

```nabi
let x = 2.0 / 3.0;
for prec in 1..=3 {
    let width = prec + 4;
    println(f"[{x:>${width}.${prec}f}] [{x:.{prec}f}]");
}
```

```output
[  0.7] [0.7]
[  0.67] [0.67]
[  0.667] [0.667]
```

## 打印你自己的类型：`Display` 与 `Debug`

`f"{x}"` 调用 `Display` trait，`f"{x:?}"` 调用 `Debug`。`Debug` 可以用 `#[derive(Debug)]` 自动实现，而面向用户的 `Display` 要自己实现。`fmt` 把文本写入它收到的 `Formatter`。

```nabi
#[derive(Debug)]
struct Point { x: i32; y: i32; }

impl Display for Point {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        try f.write_str(f"({self.x}, {self.y})");
    }
}

#[derive(Debug)]
enum Shape { Circle(f64), Rect { w: f64; h: f64; } }

let p = Point { x: 1, y: -2 };
println(f"{p} | {p:?} | [{p:>10}]");
println(f"{[p, Point { x: 3, y: 4 }]:?}");
println(f"{Shape::Circle(1.5):?} {Shape::Rect { w: 2.5, h: 3.0 }:?}");
```

```output
(1, -2) | Point { x: 1, y: -2 } | [   (1, -2)]
[Point { x: 1, y: -2 }, Point { x: 3, y: 4 }]
Circle(1.5) Rect { w: 2.5, h: 3.0 }
```

当输出是基于字段的单个 f-string 时，在类型上写 `#[display("({self.x}, {self.y})")]` 会生成同样的 `impl Display`；在枚举上，每个变体各写一个，按名字（`{w}`）或位置（`{_0}`）引用字段。宽度和对齐（`{p:>10}`）作用于 `fmt` 写出的整个输出；精度会传给 `fmt` 本身，所以用 `Display::fmt(self.v, ref f)` 写出数字的类型会打印舍入后的结果（`{q:.2}` 是 `1500.00 m`）。实现 `Display` 后，就可以把该类型直接放进 f-string 的任何位置；有了 `Debug`，保存该类型的 `List` 或 `Dict` 也能用 `{:?}` 打印。`{3.0}` 打印 `3`，因为浮点数的默认输出是“能读回同一个值的最短表示”；`Debug` 会给整数值加上 `.0`，所以 `h: 3.0` 仍然看得出是浮点数。`{p:#?}` 把结构体或集合按每行一个字段或元素缩进打印。

## 小结

- `str` 是不可变的 UTF-8。`len()` 和切片按字节工作；用 `chars()` 处理字符。
- 比较文本前先 `normalize(Nfc)`，用 `eq_ignore_case` / `casefold()` 忽略大小写，用 `graphemes()` 统计读者看到的字符。
- 在 `StrBuf` 中逐段构建字符串，用 `to_str()` 取出。字面量不会自动连接；用 `+` 连接。
- 原始字符串 `r"..."` 和 `r#"..."#` 不处理转义。
- `b"..."` 是 `[u8; N]`（在期望 `Slice<u8>` 的地方则是 `Slice<u8>`）。`include_str!` / `include_bytes!` 在编译期嵌入文件。
- f-string 的插值可以放任意表达式，包括原样书写的同一种引号。`{{` 和 `}}` 是字面的花括号。
- `:.2`、`:>8`、`:x`、`:?` 这样的说明符设定格式，`${..}` 让它们成为动态的。
- 为你自己的类型实现 `Display`（或写 `#[display("..")]`），并 derive `Debug`。`{x=}` 显示名字和值，`{n:,}` 给数字分组，`to_str()` 把任何 `Display` 值转换为 `str`。

## 延伸阅读

- 规范：[字符串字面量与 f-string](../spec/refs-lang/001-lexical/107-string-literals.html)、[字符字面量](../spec/refs-lang/001-lexical/108-character-literals.html)、[字节串](../spec/refs-lang/001-lexical/107-string-literals.html#175-字节串)、[嵌入文件](../spec/refs-lang/006-meta-system/611-include.html)
- 标准库规范：[fmt](../spec/refs-standard-libraries/200-core/210-fmt.html)、[字符串](../spec/refs-standard-libraries/200-core/215-string.html)
- API：[`str` 的方法](../../std/core/str.html#impl.str)、[`StrBuf`](../../std/core/str.html#struct.StrBuf)、[`Display`](../../std/core/fmt.html#trait.Display)、[`Formatter`](../../std/core/fmt.html#struct.Formatter)
- 下一章：[10. 错误处理](10-errors.html)
