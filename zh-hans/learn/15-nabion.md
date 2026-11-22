# 15. NABION

本章介绍如何用 Nabi 的数据格式 NABION（Nabi Object Notation）读写配置文件。这里没有新语法要学：NABION 文件使用的就是你在 Nabi 源代码中写值时用的字面量语法。包清单 `nabi.nabion` 也使用这种格式。

## 格式一览

一个文件存放一个值，通常是一个结构体。

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

| 形式 | 示例 |
|------|----|
| 基本值 | `none`、`true`、`42`、`0xff`、`3.14`、`'a'`、`"text"` |
| 列表 / 元组 | `[1, 2, 3]`、`("x", 1)` |
| dict | `{ "k": 1, "j": 2 }`（键是值） |
| struct | `Point { x: 1, y: 2 }`、`Pair(1, 2)` |
| 枚举变体 | `Mode::Fast`、`Shape::Circle(1.0)`、`Shape::Rect { w: 2, h: 3 }` |

允许 `//` 注释和末尾逗号。变量、运算符、函数调用、f-string 这类“代码”则不允许。NABION 是纯数据，永远不会被执行。

## `#[derive(Nabion)]` 与 `from_str` / `to_str`

在结构体或枚举上加 `#[derive(Nabion)]`，就可以用 `std::nabion` 的 `from_str` 读取、用 `to_str` 写出。读取哪种类型由变量的类型标注决定。

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

嵌套的结构体、列表和枚举都以同样的方式读取。像第二个 server 那样缺少字段时，按以下规则处理：

- 字段有 `= default` 时，取该值（`port: u16 = 80`）。
- 类型是 `T | none` 时，取 `none`（`tls`）。
- 否则报错。

`to_str` 写出**规范形式**：缩进 4 个空格，能在 100 列内放下就写成一行，否则每行一个元素并带末尾逗号。把规范形式读回来，得到的是同一个值。`to_str_flat` 写出相同的文本，但不论多长都写成一行。对于没有 NABION 形式的值（例如 NaN 浮点数），两者都会抛出 `NabionError`。

## 枚举、元组结构体与 Dict

枚举变体总是写出完整路径，如 `Enum::Variant`。元组结构体写作 `Name(a, b)`。标准集合类型和元组无需 derive 即可直接读取。

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
> `Dict` 和 `Set` 保持插入顺序，所以 `to_str` 按添加顺序写出其中的条目，每次运行结果都相同。

## 错误输入与错误

出错时 `from_str` 抛出 `NabionError`。消息的形式是 `line:column: description`，便于人在文件中立即找到出错位置。

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

类型中没有的字段（`user`）会被跳过，因此为较新版本的程序写的文件仍然可以读取。若要把这类字段视为错误（例如用来发现拼错的键），在类型上加 `#[nabion(deny_unknown_fields)]`。

## 不指定类型读取：`Value`

事先不知道结构的数据，可以用 `nabion::parse` 读成 `Value` 枚举。`Value` 有 `None`、`Int`、`Str`、`List`、`Dict`、`Struct`、`Variant` 等变体，`kind_name()` 返回种类的名称。以后的版本可能会增加变体，所以对 `Value` 的 `match` 需要一个 `_` 分支。

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

## 读写配置文件

与第 13 章的 `fs` 结合，就能处理配置文件。构建本网站的程序正是这样做的：它把网站仓库中的 `site.nabion` 读成一个 `#[derive(Nabion)] struct Site`，带默认值的字段可以在文件中省略。

下面的示例把一份配置写入临时文件，读取、修改后再写回。`load` 可能抛出 `IoError` 和 `NabionError` 两种错误，调用方分别捕获。

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

用 `to_str` 重写会丢掉注释。人工编辑的文件最好不要让程序覆盖。`nabi fmt` 对 `.nabion` 文件只修正缩进，不改动注释和布局。

## 小结

- NABION 是直接使用 Nabi 字面量语法的数据格式，文件扩展名为 `.nabion`。
- 加上 `#[derive(Nabion)]`，用 `nabion::from_str` 读取，用 `nabion::to_str` 写出（写成一行用 `to_str_flat`）。
- 缺少的字段取其 `= default`，类型为 `T | none` 时取 `none`，否则报错。未知字段会被跳过，除非类型标了 `#[nabion(deny_unknown_fields)]`。
- 错误消息的形式是 `line:column: description`。
- 不知道结构时，用 `nabion::parse` 得到一个 `Value`。
- `Dict` 保持插入顺序，所以映射会按读入的顺序写回。

## 延伸阅读

- 规范：[3.8 NABION](../spec/refs-standard-libraries/300-std/308-nabion.html)、[字段默认值](../spec/refs-lang/002-types/203-struct.html#233-字段默认值)、[nabi.nabion 清单](../spec/refs-lang/009-tools/910-nabi-frontend.html)
- API：[std::nabion](../../std/std/nabion.html)
- 下一章：[16. 工具](16-tools.html)
