# 9. 字串與 f-string

幾乎每個程式都會用到字串。本章介紹 `str` 和 `StrBuf`、常用的方法、各種字面值，以及你一路以來都在用的 f-string 的格式規範（format specifier）。

## `str`：不可變的 UTF-8

`str` 是一串 UTF-8 位元組，建立之後就不會再改變。因此 `len()` 回傳的是**位元組**數，而不是字元數。要計算字元（碼位）數，請用 `char_count()`。

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

這裡刻意使用韓文：每個韓文音節在 UTF-8 中佔 3 個位元組，所以 `"hello, 세계"` 有 13 個位元組，卻只有 9 個字元。

切片 `s[a..b]` 也是在位元組位置上切割。由於一個韓文字元佔 3 個位元組，除了字元邊界之外不能在其他地方切割；在字元內部切割會觸發 panic，並指出是哪個位元組：

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

要逐字元處理，請用 `chars()` 或 `char_indices()`。當位移可能落在字元內部時，`get(a..b)` 會回傳 `none` 而不觸發 panic，`floor_char_boundary(i)` 會把位移往回移到所在字元的起點，`char_slice(a, b)` 則以字元而非位元組計數：

```nabi
let s = "hello, 세계";
println(f"{s.get(0..8) ?? "none"} {s[0:s.floor_char_boundary(8)]} {s.char_slice(7, 8)}");
```

```output
none hello,  세
```

單一字元是 `char`，用單引號表示。它存放一個 Unicode 純量值，並提供分類與轉換的方法。

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

大小寫轉換可能把一個字元變成好幾個，所以 `to_uppercase`、`to_lowercase` 和 `to_titlecase` 回傳 `str`（`'ß'` 會變成 `"SS"`）。需要剛好拿回一個 `char` 時，請用 `to_simple_uppercase` 和其他 `to_simple_*` 形式，沒有單一字元對應的字元會保持原樣。對於不是該基數數字的字元，`to_digit` 回傳 `none`。

## 常用方法

`str` 的方法不會修改原本的字串，而是回傳新的值。可能找不到結果的操作會回傳 `T | none`，因此很適合搭配第 5 章的 `??` 和 `if let`。

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

`reversed` 反轉字元順序，`count` 計算不重疊的符合次數，`replacen` 只取代前 `n` 個。`to_str()` 適用於所有 `Display` 值，就像 Python 的 `str(x)`。`parse` 是失敗時會擲回錯誤的函式，所以範例加上了 `try!`。錯誤會在[下一章](10-errors.html)說明。其他方法還有 `lines()`、`split_whitespace()`、`strip_prefix()` 和 `trim_start()`（[`str` API](../../std/core/str.html#impl.str)）。

## Unicode 文字：正規化與字素

同一段文字可以用不同的碼位表示：`"é"` 可以是一個碼位，也可以是 `e` 後面接一個組合重音符號。`==` 比較的是位元組，所以要比較或儲存的文字，請先用 `normalize(Nfc)` 正規化。要忽略大小寫比較時，`casefold()`（或 `eq_ignore_case`）會把 `"ß"` 摺疊成 `"ss"`，這是 `to_lowercase` 做不到的。讀者眼中的一個字元可能由好幾個碼位組成；`graphemes()` 產生這些字素叢集（grapheme cluster），`words()` 則產生單字。

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

`grapheme_slice(0, n)` 保留前 `n` 個字素，所以截短的標題絕不會把重音符號或國旗切成兩半。正規化形式、大小寫摺疊與分段規則請見 [`str` 參考](../spec/refs-standard-libraries/200-core/215-string.html#21510-unicode-文字正規化大小寫摺疊分段)。

## `StrBuf`：一段一段地建構

因為 `str` 是不可變的，在迴圈中用 `+` 串接每次都會建立新的字串。要一段一段地建構字串時，請寫入 `StrBuf` 緩衝區，完成後再用 `to_str()` 取出結果。

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

只串接一次兩個字串時，用 `+` 就夠了；它也能接上一個 `char`（`s + '!'`），`s += '!'` 也一樣。不過，和 C 或 Python 不同，相鄰的字串字面值（`"abc" "def"`）不會自動串接。這樣寫會產生編譯錯誤；想串接時，請寫 `+`（[1.7.4](../spec/refs-lang/001-lexical/107-string-literals.html#174-相鄰字串的自動串接)）。

```nabi
let greeting = "hello, " + "world" + '!';
println(greeting);
```

```output
hello, world!
```

## 字面值：跳脫、原始字串、續行

一般字串會處理 `\n`、`\t`、`\"`、`\\` 和 `\u{..}` 等跳脫序列。對於充滿反斜線的路徑或正規表示式，原始字串（raw string）`r"..."` 比較方便。如果內容需要 `"`，就在字串兩側各加上相同數量的 `#`。

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

如最後一個範例所示，行尾的 `\` 會同時移除換行和下一行開頭的空白。這條規則在原始字串中同樣不適用（[1.7.2](../spec/refs-lang/001-lexical/107-string-literals.html#172-原始字串)）。

## 位元組字串 `b"..."`

通訊協定和檔案格式使用的是位元組，而不是文字。位元組字串 `b"..."` 是由其位元組組成的固定陣列 `[u8; N]`。ASCII 直接寫，其他內容寫成 `\xHH`（任何 `\x00`-`\xFF` 的值）。在預期 `Slice<u8>` 的地方，同一個字面值就是該陣列的檢視，所以可以直接傳給接受位元組的函式。

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

在位元組字串中直接寫入非 ASCII 字元，或使用大於 `\xFF` 的跳脫序列，都是編譯錯誤；位元組字串也不支援續行。

單一位元組寫成 `b'0'`：它是遵循相同規則的 `u8`（`b'\n'`、`b'\xFF'`），在剖析器比較位元組時很方便。

```nabi
let digits = "a1b22".as_bytes().filter(|c| b'0' <= c <= b'9').count();
println(f"{b'0'} {digits}");
```

```output
48 3
```

## 嵌入檔案：`include_str!` / `include_bytes!`

要把樣板、測試資料或圖示放進執行檔裡一起發行，可以在編譯時嵌入檔案。`include_str!("path")` 是以 `str` 表示的檔案內容，`include_bytes!("path")` 則是以 `[u8; N]` 表示的檔案位元組，就像位元組字串一樣。路徑相對於包含該呼叫的原始檔，而結果是常數，所以可以用來初始化 `const` 或 `static`。

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

檔案不存在是編譯錯誤，對 `include_str!` 而言，不是有效 UTF-8 的檔案也是編譯錯誤。修改被嵌入的檔案時，使用它的程式碼會重新建置。

## f-string 的插值欄位

`f"..."` 的 `{ }` 裡可以放任何運算式。規則和 Python 3.12（PEP 701）相同：插值欄位內可以再次使用外層的引號字元，f-string 也可以巢狀使用。要寫出大括號本身，請用 `{{` 和 `}}`。

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

插值欄位內不使用 `\"` 這類跳脫序列；直接寫出引號即可。文字或插值欄位跨越多行時，請用 `f"""..."""`。

## 格式規範

在插值欄位的運算式後面加上 `:`，就能指定格式。語法是 `[fill][align][sign][#][0][width][grouping][.precision][type]`。

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

和 Python 一樣，`{x=}` 會印出運算式的文字，接著是 `=`，然後是它的值（`{x = }` 會保留空格）。在寬度後面加上 `,` 或 `_`，會把整數的位數每三位分組（`{1234567:,}` 是 `1,234,567`），型別 `%` 則會乘以 100 並加上百分比符號（`{0.256:.1%}` 是 `25.6%`）。

```nabi
let n = 1234567;
let rate = 0.256;
println(f"{n=} {n:,} {n:_} {rate:.1%} {n + 1 = }");
```

```output
n=1234567 1,234,567 1_234_567 25.6% n + 1 = 1234568
```

沒有明確指定對齊方式時，數字靠右對齊，其他一律靠左對齊。`?` 選擇 `Debug` 格式，它會連同引號和跳脫序列一起顯示字串。格式規範會在編譯時依插值欄位的型別檢查，所以像 `{s:x}`（其中 `s: str`）這樣的錯誤會立即報錯。

`g` 型別是 C 和 Python 的通用格式：精確度計算的是有效位數（預設為 6），很大或很小的數字會改用指數表示。`{1234.5678:.3g}` 是 `1.23e+03`，`{1e20:g}` 是 `1e+20`，`{0.5:g}` 是 `0.5`。

要在執行時決定寬度或精確度，就在格式規範裡再放一個插值欄位。`{..}` 和 `${..}` 兩種寫法都可以（[動態格式規範](../spec/refs-lang/001-lexical/107-string-literals.html#動態格式規範)）。

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

## 印出自己的型別：`Display` 與 `Debug`

`f"{x}"` 呼叫 `Display` trait，`f"{x:?}"` 呼叫 `Debug`。`Debug` 可以用 `#[derive(Debug)]` 自動實作，而給使用者看的 `Display` 則要自己實作。`fmt` 會把文字寫入它接收到的 `Formatter`。

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

當輸出只是一個由欄位組成的 f-string 時，在型別上寫 `#[display("({self.x}, {self.y})")]` 就會產生同樣的 `impl Display`；在列舉上，每個變體各自寫一個，並以名稱（`{w}`）或位置（`{_0}`）引用欄位。寬度和對齊（`{p:>10}`）套用在 `fmt` 寫出的整個輸出上；精確度則會傳到 `fmt` 本身，所以用 `Display::fmt(self.v, ref f)` 寫出數字的型別，會印出四捨五入後的數字（`{q:.2}` 是 `1500.00 m`）。實作 `Display` 之後，就能把該型別直接放在 f-string 的任何位置；有了 `Debug`，裝著該型別的 `List` 或 `Dict` 也能用 `{:?}` 印出。`{3.0}` 會印出 `3`，因為浮點數的預設輸出是「讀回時仍是同一個值的最短表示」；`Debug` 會在整數值後面加上 `.0`，所以 `h: 3.0` 看起來仍然是浮點數。`{p:#?}` 會把結構或集合每行印一個欄位或元素，並加上縮排。

## 小結

- `str` 是不可變的 UTF-8。`len()` 和切片以位元組為單位；處理字元請用 `chars()`。
- 比較文字前先 `normalize(Nfc)`，用 `eq_ignore_case` / `casefold()` 忽略大小寫，用 `graphemes()` 計算讀者看到的字元。
- 用 `StrBuf` 一段一段地建構字串，再用 `to_str()` 取出。字面值不會自動串接；請用 `+` 串接。
- 原始字串 `r"..."` 和 `r#"..."#` 不處理跳脫序列。
- `b"..."` 是 `[u8; N]`（在預期 `Slice<u8>` 的地方則是 `Slice<u8>`）。`include_str!` / `include_bytes!` 在編譯時嵌入檔案。
- f-string 的插值欄位可以放任何運算式，包括直接使用相同的引號字元。`{{` 和 `}}` 是大括號本身。
- `:.2`、`:>8`、`:x` 和 `:?` 等格式規範決定格式，`${..}` 讓它們成為動態的。
- 為自己的型別實作 `Display`（或寫 `#[display("..")]`），並衍生 `Debug`。`{x=}` 顯示名稱和值，`{n:,}` 為位數分組，`to_str()` 把任何 `Display` 值轉成 `str`。

## 延伸閱讀

- 規格：[字串字面值與 f-string](../spec/refs-lang/001-lexical/107-string-literals.html)、[字元字面值](../spec/refs-lang/001-lexical/108-character-literals.html)、[位元組字串](../spec/refs-lang/001-lexical/107-string-literals.html#175-位元組字串)、[檔案嵌入](../spec/refs-lang/006-meta-system/611-include.html)
- 標準函式庫規格：[fmt](../spec/refs-standard-libraries/200-core/210-fmt.html)、[字串](../spec/refs-standard-libraries/200-core/215-string.html)
- API：[`str` 方法](../../std/core/str.html#impl.str)、[`StrBuf`](../../std/core/str.html#struct.StrBuf)、[`Display`](../../std/core/fmt.html#trait.Display)、[`Formatter`](../../std/core/fmt.html#struct.Formatter)
- 下一章：[10. 錯誤處理](10-errors.html)
