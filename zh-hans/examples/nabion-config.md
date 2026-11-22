# NABION 配置

把 NABION 配置读入 `#[derive(Nabion)]` 结构体并填入默认值，对它进行校验、修改，再以 NABION 写回。

## 代码

```nabi
// Load a server configuration from NABION, validate it, change it and write it back.
use std::nabion;
use std::nabion::NabionError;

#[derive(Nabion, Debug, PartialEq)]
enum Level { Debug, Info, Warn, Error }

#[derive(Nabion, Debug)]
struct Listen {
    host: str = "127.0.0.1";
    port: u16 = 8080;
}

#[derive(Nabion, Debug)]
struct Route {
    path: str;
    target: str;
    timeout_ms: u32 = 5000;
}

#[derive(Nabion, Debug)]
struct Tls {
    cert: str;
    key: str;
}

#[derive(Nabion, Debug)]
#[nabion(deny_unknown_fields)]
struct Config {
    name: str;
    listen: Listen;
    workers: i32 = 4;
    log: Level = Level::Info;
    routes: List<Route> = [];
    limits: Dict<str, i64> = Dict::new();
    tls: Tls | none;
}

const SOURCE: str = "// gateway.nabion
Config {
    name: \"gateway\",
    listen: { port: 9000 },
    log: Level::Debug,
    routes: [
        Route { path: \"/api\", target: \"http://10.0.0.5:8000\" },
        Route { path: \"/static\", target: \"file:///srv/www\", timeout_ms: 500 },
    ],
    limits: { \"body_kb\": 512, \"conns\": 1000 },
}";

fn validate(c: Config) -> List<str> {
    let problems: List<str> = [];
    if c.listen.port == 0 { problems.push("listen.port must not be 0"); }
    if c.workers < 1 || c.workers > 64 { problems.push(f"workers must be 1..=64, got {c.workers}"); }
    for r in c.routes {
        if !r.path.starts_with("/") { problems.push(f"route '{r.path}' must start with '/'"); }
    }
    if c.log == Level::Debug && c.tls is Tls { problems.push("debug logging with TLS leaks secrets"); }
    problems
}

fn load(text: str) throws (NabionError) -> Config {
    try nabion::from_str(text)
}

fn report(label: str, text: str) {
    let c = try load(text) catch e {
        println(f"{label}: parse error {e}");
        return;
    };
    let problems = validate(c);
    if problems.is_empty() { println(f"{label}: ok"); }
    for p in problems { println(f"{label}: {p}"); }
}

fn main() throws (NabionError) {
    let c = try load(SOURCE);
    println(f"{c.name} on {c.listen.host}:{c.listen.port}, {c.workers} workers, log {c.log:?}");
    for r in c.routes {
        println(f"  {r.path:<8} -> {r.target} ({r.timeout_ms} ms)");
    }
    println(f"  tls off: {c.tls is none}  body limit: {c.limits["body_kb"]} KB");

    report("good", SOURCE);
    report("bad", "Config { name: \"x\", listen: { port: 0 }, workers: 100, routes: [Route { path: \"api\", target: \"t\" }] }");
    report("typo", "Config { name: \"x\", listen: {}, wokers: 2 }");
    report("type", "Config {\n    name: \"x\",\n    listen: { port: \"80\" },\n}");

    // Modify and serialize back; defaults are written out explicitly.
    c.log = Level::Warn;
    c.workers = 8;
    c.tls = Tls { cert: "/etc/gw/cert.pem", key: "/etc/gw/key.pem" };
    c.routes.push(Route { path: "/health", target: "internal:health", timeout_ms: 100 });
    let out = try nabion::to_str(c);
    println(out);

    let back = try load(out);
    println(f"round trip: {(try nabion::to_str(back)) == out}");
}
```

```output
gateway on 127.0.0.1:9000, 4 workers, log Debug
  /api     -> http://10.0.0.5:8000 (5000 ms)
  /static  -> file:///srv/www (500 ms)
  tls off: true  body limit: 512 KB
good: ok
bad: listen.port must not be 0
bad: workers must be 1..=64, got 100
bad: route 'api' must start with '/'
typo: parse error 1:33: unknown field `wokers` in Config
type: parse error 3:21: expected u16, found string
Config {
    name: "gateway",
    listen: Listen { host: "127.0.0.1", port: 9000 },
    workers: 8,
    log: Level::Warn,
    routes: [
        Route { path: "/api", target: "http://10.0.0.5:8000", timeout_ms: 5000 },
        Route { path: "/static", target: "file:///srv/www", timeout_ms: 500 },
        Route { path: "/health", target: "internal:health", timeout_ms: 100 },
    ],
    limits: { "body_kb": 512, "conns": 1000 },
    tls: Tls { cert: "/etc/gw/cert.pem", key: "/etc/gw/key.pem" },
}
round trip: true
```

## 讲解

- 只需一个 `#[derive(Nabion)]`，结构体和枚举就能与 NABION 互相转换。嵌套的结构体（`Listen`）、枚举（`Level`）、`List`、`Dict` 和 `Tls | none` 都能直接对应（[NABION](../learn/15-nabion.html)，[3.8.2 与 Nabi 类型的对应](../spec/refs-standard-libraries/300-std/308-nabion.html#382-与-nabi-类型的对应)）。
- 文档中缺少的字段采用结构体的默认值（`workers: i32 = 4`）；没有默认值的 `T | none` 字段为 `none`。不写类型名的嵌套值，例如 `listen: { port: 9000 }`，也可以接受。
- 遇到类型不对的值时，`nabion::from_str` 抛出 `NabionError`，消息中带有 `line:col` 位置。类型中没有的字段默认会被跳过；在 `Config` 上加 `#[nabion(deny_unknown_fields)]` 会把它变成错误，这样 `wokers` 这样的拼写错误就不会被悄悄忽略（[`nabion::from_str`](../../std/std/nabion.html#fn.from_str)）。
- 格式之外的规则（端口不能为 0，工作线程数在范围内）由 `validate` 收集，并一起报告。`T | none` 值是否为空用 `c.tls is none` 检查。
- `nabion::to_str` 写出规范形式，遇到没有 NABION 形式的值（NaN 浮点数）时抛出错误。从默认值填入的字段也会写出，所以读取输出会得到同样的值。`Dict` 保持键的顺序，所以限制项按读入时的顺序输出（[`nabion::to_str`](../../std/std/nabion.html#fn.to_str)）。
