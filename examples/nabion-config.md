# NABION configuration

Read a NABION configuration into `#[derive(Nabion)]` structs with defaults filled in, validate it, change it and write it back as NABION.

## Code

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

## Walkthrough

- A single `#[derive(Nabion)]` makes structs and enums convert to and from NABION. A nested struct (`Listen`), an enum (`Level`), `List`, `Dict` and `Tls | none` all map directly ([NABION](../learn/15-nabion.html), [3.8.2 mapping to Nabi types](../spec/refs-standard-libraries/300-std/308-nabion.html#382-correspondence-with-nabi-types)).
- Fields missing from the document take the struct's default (`workers: i32 = 4`); a `T | none` field without a default becomes `none`. Nested values written without a type name, like `listen: { port: 9000 }`, are accepted too.
- `nabion::from_str` throws a `NabionError` for a value of the wrong type, with a `line:col` position in the message. A field the type does not have is skipped by default; `#[nabion(deny_unknown_fields)]` on `Config` makes it an error, so a typo such as `wokers` is not silently ignored ([`nabion::from_str`](../std/std/nabion.html#fn.from_str)).
- Rules beyond the format (port must not be 0, worker count in range) are collected by `validate` and reported together. Whether a `T | none` value is empty is checked with `c.tls is none`.
- `nabion::to_str` writes the canonical form, and throws for a value with no NABION form (a NaN float). Fields filled from defaults are written out too, so reading the output gives the same value back. A `Dict` keeps its key order, so the limits come out in the order they were read ([`nabion::to_str`](../std/std/nabion.html#fn.to_str)).
