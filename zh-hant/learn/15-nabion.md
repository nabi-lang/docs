# 15. NABION

本章說明如何讀寫 NABION（Nabi Object Notation）格式的設定檔，NABION 是 Nabi 的資料格式。你不必學新語法：NABION 檔案使用的字面值語法，就和你在 Nabi 原始碼中寫值的語法相同。套件清單 `nabi.nabion` 也使用這種格式。

## 格式一覽

一個檔案存放一個值，通常是一個結構。

```nabion
// app.nabion
Config {
    name: "demo",
    retries: 3,
    ratio: 0.75,
    tags: ["a", "b"],
    limits: { "cpu": 2, "mem": 512 },
    mode: Mode::Fast,
    proxy: none,
}
```

| 形式 | 範例 |
|------|----|
| 基本值 | `none`、`true`、`42`、`0xff`、`3.14`、`'a'`、`"text"` |
| List / 元組 | `[1, 2, 3]`、`("x", 1)` |
| dict | `{ "k": 1, "j": 2 }`（鍵是值） |
| struct | `Point { x: 1, y: 2 }`、`Pair(1, 2)` |
| 列舉變體 | `Mode::Fast`、`Shape::Circle(1.0)`、`Shape::Rect { w: 2, h: 3 }` |

允許 `//` 註解和結尾逗號。變數、運算子、函式呼叫、f-string 之類的「程式碼」則不允許。NABION 是純資料，永遠不會執行。

## `#[derive(Nabion)]` 與 `from_str` / `to_str`

在結構或列舉上加上 `#[derive(Nabion)]`，就可以用 `std::nabion` 的 `from_str` 讀取它，用 `to_str` 寫出它。讀取哪個型別，由變數的型別標註決定。

```nabi
use std::nabion;
use std::nabion::NabionError;

#[derive(Nabion)]
enum Mode { Fast, Slow }

#[derive(Nabion)]
struct Server { host: str; port: u16 = 80; tls: bool | none; }

#[derive(Nabion)]
struct Config {
    name: str;
    mode: Mode;
    servers: List<Server>;
    limits: Dict<str, i32>;
}

fn main() throws (NabionError) {
    let text = r#"
Config {
    name: "demo",
    mode: Mode::Fast,
    servers: [
        Server { host: "a.example", port: 8080, tls: true },
        Server { host: "b.example" },
    ],
    limits: { "cpu": 2, "mem": 512 },
}
"#;
    let cfg: Config = try nabion::from_str(text);
    println(f"{cfg.name}: {cfg.servers.len()} servers");
    for s in cfg.servers {
        println(f"  {s.host}:{s.port} tls={s.tls ?? false}");
    }
    println(f"cpu limit: {cfg.limits["cpu"]}");
    println(try nabion::to_str(cfg));
}
```

```output
demo: 2 servers
  a.example:8080 tls=true
  b.example:80 tls=false
cpu limit: 2
Config {
    name: "demo",
    mode: Mode::Fast,
    servers: [
        Server { host: "a.example", port: 8080, tls: true },
        Server { host: "b.example", port: 80, tls: none },
    ],
    limits: { "cpu": 2, "mem": 512 },
}
```

巢狀的結構、串列和列舉都以相同方式讀取。當欄位缺少時，例如第二個伺服器，會套用以下規則：

- 如果欄位有 `= default`，就使用該值（`port: u16 = 80`）。
- 如果型別是 `T | none`，就使用 `none`（`tls`）。
- 否則是錯誤。

`to_str` 寫出的是**標準形式**（canonical form）：縮排 4 個空格，100 欄內放得下就寫成一行，否則每個元素一行並加上結尾逗號。把標準形式讀回來，會得到相同的值。`to_str_flat` 則不論寬度，都把同樣的內容寫成一行。對於沒有 NABION 形式的值，例如 NaN 浮點數，兩者都會擲回 `NabionError`。

## 列舉、元組結構、Dict

列舉變體一律以完整路徑書寫，例如 `Enum::Variant`。元組結構寫成 `Name(a, b)`。標準集合型別和元組可以直接讀取，不需要 derive。

```nabi
use std::nabion;

#[derive(Nabion)]
enum Shape {
    Dot,
    Circle(f64),
    Rect { w: i32; h: i32; },
}

#[derive(Nabion)]
struct Point(i32, i32);

let shapes: List<Shape> = try! nabion::from_str("[Shape::Dot, Shape::Circle(1.5), Shape::Rect { w: 3, h: 2 }]");
for s in shapes {
    let d = match s {
        Shape::Dot => "dot",
        Shape::Circle(r) => f"circle r={r}",
        Shape::Rect { w, h } => f"rect {w}x{h}",
    };
    println(d);
}

println(try! nabion::to_str(Point(3, -4)));
let p: Point = try! nabion::from_str("Point(1, 2)");
println(f"{p.0} {p.1}");

let ages: Dict<str, i32> = try! nabion::from_str(r#"{ "kim": 31, "lee": 27 }"#);
println(f"lee is {ages["lee"]}");
let pair: (str, List<i32>) = try! nabion::from_str(r#"("primes", [2, 3, 5])"#);
println(f"{pair.0}: {pair.1:?}");
```

```output
dot
circle r=1.5
rect 3x2
Point(3, -4)
1 2
lee is 27
primes: [2, 3, 5]
```

> [!TIP]
> `Dict` 和 `Set` 會保留插入順序，所以 `to_str` 會依加入的順序寫出它們的項目，每次執行都一樣。

## 錯誤的輸入與錯誤

有問題時，`from_str` 會擲回 `NabionError`。訊息的形式是 `line:column: description`，讓人能立刻在檔案中找到出問題的位置。

```nabi
use std::nabion;

#[derive(Nabion)]
struct Server { host: str; port: u16; }

fn check(text: str) {
    let s: Server = try nabion::from_str(text) catch e {
        println(f"error: {e}");
        return;
    };
    println(f"ok: {s.host}:{s.port}");
}

check(r#"Server { host: "h", port: 22 }"#);
check(r#"Server { host: "h" }"#);
check(r#"Server { host: "h", port: 70000 }"#);
check(r#"Server { host: h, port: 22 }"#);
check(r#"Server { host: "h", port: 22, user: "me" }"#);
check(r#"Client { host: "h", port: 22 }"#);
```

```output
ok: h:22
error: 1:1: missing field `port` in Server
error: 1:27: integer 70000 is out of range for u16
error: 1:16: bare identifier `h` is not allowed in NABION (strings are quoted, variants are written `Enum::h`)
ok: h:22
error: 1:1: expected Server, found `Client { .. }`
```

型別沒有的欄位（`user`）會被略過，所以為較新版本程式寫的檔案仍然讀得進來。若要讓這種欄位成為錯誤，例如為了抓出拼錯的鍵，請在型別上加上 `#[nabion(deny_unknown_fields)]`。

## 不指定型別讀取：`Value`

事先不知道結構的資料，可以用 `nabion::parse` 讀進 `Value` 列舉。`Value` 有 `None`、`Int`、`Str`、`List`、`Dict`、`Struct`、`Variant` 等變體，`kind_name()` 會回傳種類的名稱。之後的版本可能會新增變體，所以對 `Value` 的 `match` 需要一個 `_` 分支。

```nabi
use std::nabion;
use std::nabion::Value;

let v = try! nabion::parse(r#"Site { title: "nabi", pages: 12, draft: false }"#);
match v {
    Value::Struct { name, fields } => {
        println(f"struct {name ?? "(anonymous)"}");
        for (k, x) in fields {
            println(f"  {k}: {x.kind_name()} = {x}");
        }
    },
    _ => println(f"not a struct: {v.kind_name()}"),
}
```

```output
struct Site
  title: string = "nabi"
  pages: integer = 12
  draft: bool = false
```

## 讀寫設定檔

搭配第 13 章的 `fs`，就能處理設定檔。建置本網站的程式正是這麼做的：它從網站儲存庫讀取 `site.nabion`，存進 `#[derive(Nabion)] struct Site`，有預設值的欄位可以從檔案中省略。

下面的範例把設定寫到暫存檔，讀取它、修改它，再寫回去。`load` 可能擲回 `IoError` 和 `NabionError` 兩種錯誤，呼叫端分別攔截每一種。

```nabi
use std::fs;
use std::env;
use std::process;
use std::nabion;
use std::nabion::NabionError;
use core::io::IoError;

#[derive(Nabion)]
struct Site {
    title: str;
    tagline: str;
    spec: str = "../spec/ko";
    out: str = "public";
}

fn load(path: str) throws (IoError | NabionError) -> Site {
    try nabion::from_str(try fs::read_to_str(path))
}

let path = (env::temp_dir() / f"nabi_learn15_{process::id()}.nabion").as_str();
try! fs::write(path, r#"// Site configuration
Site {
    title: "nabi",
    tagline: "statically typed, rich library",
}
"#);
let site = try! load(path);
println(f"{site.title} -> {site.out}");

site.out = "dist";
try! fs::write(path, nabion::to_str(site) + "\n");
print(try! fs::read_to_str(path));

try! fs::write(path, "Site { title: 1 }");
try load(path) catch e: NabionError { println(f"bad config: {e}"); };
try! fs::remove_file(path);
try load(path) catch e: IoError { println(f"missing: {e.kind():?}"); };
```

```output
nabi -> public
Site { title: "nabi", tagline: "statically typed, rich library", spec: "../spec/ko", out: "dist" }
bad config: 1:15: expected str, found integer
missing: NotFound
```

用 `to_str` 改寫會丟掉註解。由人編輯的檔案，最好不要讓程式覆寫。`nabi fmt` 對 `.nabion` 檔案只會修正縮排，不會動註解和排版。

## 小結

- NABION 是直接使用 Nabi 字面值語法的資料格式；副檔名是 `.nabion`。
- 加上 `#[derive(Nabion)]`，用 `nabion::from_str` 讀取，用 `nabion::to_str` 寫出（或用 `to_str_flat` 寫成一行）。
- 缺少的欄位採用它的 `= default`，若型別是 `T | none` 則為 `none`；否則是錯誤。未知的欄位會被略過，除非型別標有 `#[nabion(deny_unknown_fields)]`。
- 錯誤訊息的形式是 `line:column: description`。
- 不知道結構時，用 `nabion::parse` 取得 `Value`。
- `Dict` 會保留插入順序，所以對應會依讀入的順序寫回。

## 延伸閱讀

- 規格：[3.8 NABION](../spec/refs-standard-libraries/300-std/308-nabion.html)、[欄位預設值](../spec/refs-lang/002-types/203-struct.html#233-field-default-values)、[nabi.nabion 清單](../spec/refs-lang/009-tools/910-nabi-frontend.html)
- API：[std::nabion](../std/std/nabion.html)
- 下一章：[16. 工具](16-tools.html)
