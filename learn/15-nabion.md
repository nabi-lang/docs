# 15. NABION

This chapter shows how to read and write configuration files in NABION (Nabi Object Notation), Nabi's data format. There is no new syntax to learn: a NABION file uses the same literal syntax you write values with in Nabi source. The package manifest `nabi.nabion` uses this format too.

## The format at a glance

A file holds one value, usually a single struct.

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

| Form | Example |
|------|----|
| Basic values | `none`, `true`, `42`, `0xff`, `3.14`, `'a'`, `"text"` |
| List / tuple | `[1, 2, 3]`, `("x", 1)` |
| dict | `{ "k": 1, "j": 2 }` (keys are values) |
| struct | `Point { x: 1, y: 2 }`, `Pair(1, 2)` |
| enum variant | `Mode::Fast`, `Shape::Circle(1.0)`, `Shape::Rect { w: 2, h: 3 }` |

`//` comments and trailing commas are allowed. "Code" such as variables, operators, function calls, and f-strings is not. NABION is pure data that never executes.

## `#[derive(Nabion)]` with `from_str` / `to_str`

Put `#[derive(Nabion)]` on a struct or enum and you can read it with `from_str` and write it with `to_str` from `std::nabion`. The variable's type annotation decides which type is read.

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

Nested structs, lists, and enums are all read the same way. When a field is missing, as in the second server, these rules apply:

- If the field has a `= default`, that value (`port: u16 = 80`).
- If the type is `T | none`, `none` (`tls`).
- Otherwise, an error.

`to_str` writes the **canonical form**: 4-space indentation, one line if it fits in 100 columns, otherwise one element per line with a trailing comma. Reading the canonical form back yields the same value. `to_str_flat` writes the same text on one line, whatever its width. Both throw a `NabionError` for a value with no NABION form, such as a NaN float.

## Enums, tuple structs, Dict

Enum variants are always written with their full path, like `Enum::Variant`. Tuple structs are written `Name(a, b)`. Standard collections and tuples are read directly, without a derive.

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
> `Dict` and `Set` keep insertion order, so `to_str` writes their entries in the order they were added, the same on every run.

## Bad input and errors

`from_str` throws `NabionError` when something is wrong. Messages have the form `line:column: description`, so a person can find the spot in the file right away.

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

A field the type does not have (`user`) is skipped, so a file written for a newer version of the program still reads. To make such a field an error, for example to catch a misspelled key, put `#[nabion(deny_unknown_fields)]` on the type.

## Reading without a type: `Value`

Data whose structure you don't know in advance can be read into the `Value` enum with `nabion::parse`. `Value` has variants such as `None`, `Int`, `Str`, `List`, `Dict`, `Struct`, and `Variant`, and `kind_name()` returns the name of the kind. Later releases may add variants, so a `match` on a `Value` needs a `_` arm.

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

## Reading and writing configuration files

Combined with `fs` from chapter 13, this handles configuration files. The program that builds this site does exactly that: it reads `site.nabion` from the site repository into a `#[derive(Nabion)] struct Site`, and fields with defaults can be omitted from the file.

The example below writes a configuration to a temporary file, reads it, modifies it, and writes it back. `load` can throw both `IoError` and `NabionError`, and the caller catches each kind separately.

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

Rewriting with `to_str` drops comments. Files that people edit are better left for programs not to overwrite. `nabi fmt` only fixes indentation in `.nabion` files and leaves comments and layout alone.

## Summary

- NABION is a data format that uses Nabi literal syntax as is; the file extension is `.nabion`.
- Add `#[derive(Nabion)]`, read with `nabion::from_str`, write with `nabion::to_str` (or `to_str_flat` for one line).
- A missing field takes its `= default`, or `none` if its type is `T | none`; otherwise it is an error. Unknown fields are skipped unless the type says `#[nabion(deny_unknown_fields)]`.
- Error messages have the form `line:column: description`.
- If you don't know the structure, use `nabion::parse` to get a `Value`.
- `Dict` keeps insertion order, so a map is written back in the order it was read.

## Further reading

- Spec: [3.8 NABION](../spec/refs-standard-libraries/300-std/308-nabion.html), [Field defaults](../spec/refs-lang/002-types/203-struct.html#233-field-default-values), [The nabi.nabion manifest](../spec/refs-lang/009-tools/910-nabi-frontend.html)
- API: [std::nabion](../std/std/nabion.html)
- Next chapter: [16. Tools](16-tools.html)
