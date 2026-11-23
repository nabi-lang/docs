# 20. How Do I...

Short answers to everyday tasks, each a complete program you can paste into a file and run with `nabi run file.nabi`. Each recipe links to the chapter that explains it.

## Read a file

`fs::read_to_str` reads a whole file as text; `fs::write` writes one. Both throw an `IoError`, so the calls are marked with `try` ([chapter 13](13-io.html#reading-and-writing-files)).

```nabi
use std::{env, fs, process};

let path = env::temp_dir() / f"nabi_cookbook_{process::id()}.txt";
try fs::write(path, "one\ntwo\nthree\n");

let text = try fs::read_to_str(path);
for line in text.lines() {
    println(line);
}
try fs::remove_file(path);
```

```output
one
two
three
```

## Read stdin line by line

`io::stdin().lines()` yields each line without its newline; a read error ends the loop by throwing, and `try for` passes it on ([chapter 13](13-io.html#standard-io-and-the-environment)). The output below is `0 lines` because this page builds with no input; run it locally and pipe a file in (`nabi run count.nabi < file`).

```nabi
let count = 0;
try for line in io::stdin().lines() {
    count += 1;
    println(f"{count}: {line}");
}
println(f"{count} lines");
```

```output
0 lines
```

Run it as `printf 'a\nb\n' | nabi run lines.nabi` to see it number the lines.

## Get command-line arguments

`env::args()` is a `List<str>`; element 0 is the program itself. Pass arguments after `--`: `nabi run greet.nabi -- ada grace` ([chapter 13](13-io.html#standard-io-and-the-environment), [example](../examples/cli-args.html)). The output below is `0 arguments` because this page builds with none; run it locally with some.

```nabi
use std::env;

let args = env::args();
for name in args.skip(1) {
    println(f"hello, {name}");
}
println(f"{args.len() - 1} arguments");
```

```output
0 arguments
```

## Exit with a code

`process::exit(code)` ends the program at once. `finally` blocks and `Drop` do not run. An uncaught error at the top of a script exits with code 1 by itself ([chapter 10](10-errors.html#errors-at-the-top-of-a-script)).

```nabi build
use std::{env, process};

let args = env::args();
if args.len() < 2 {
    eprintln("usage: greet NAME");
    process::exit(2);
}
println(f"hello, {args[1]}");
```

## Sleep, and measure how long something takes

`Instant::now()` reads a monotonic clock, and `elapsed()` gives a `Duration`. `thread::sleep` blocks the current thread; in async code use `await task::sleep(..)` instead ([chapter 12](12-concurrency.html#async-and-tasks), [`Duration` API](../std/core/time.html#struct.Duration)).

```nabi
use std::thread;

let start = Instant::now();
thread::sleep(Duration::ms(20));
let took = start.elapsed();
println(f"slept at least 20 ms: {took.as_ms() >= 20}");
println(f"{took.as_secs_f64() < 5.0}");
```

```output
slept at least 20 ms: true
true
```

## Dates, times and time zones

`std::time` has the calendar. `Date`, `Time` and `DateTime` are what a calendar and a wall clock show, with no zone; `Zoned` is an instant in a `TimeZone`. `Display` writes ISO 8601 / RFC 3339, `parse` reads it, and `format` / `parse_with` take strftime patterns such as `%d %B %Y` ([time zones and patterns](../spec/refs-standard-libraries/300-std/303-time.html#337-offsets-time-zones-and-zoned), [`time` API](../std/std/time.html)).

```nabi
use std::time::{Date, DateTime, TimeZone, Zoned};

let d = try Date::new(2024, 1, 31);
println(f"{d} is a {d.weekday()}; a month later: {d.add_months(1)}");
println(d.format("%A %-d %B %Y"));

let paris = try TimeZone::get("Europe/Paris");
let meeting = (try DateTime::new(2024, 3, 31, 9, 30, 0)).in_zone(paris);
println(f"{meeting} = {meeting.to_utc()}");
println(f"{meeting.add_days(1).format("%F %H:%M %Z")}");

let logged = try "2024-03-31T07:30:00Z".parse::<Zoned>();
println(f"{logged == meeting} {try Date::parse_with("31/03/2024", "%d/%m/%Y")}");
```

```output
2024-01-31 is a Wednesday; a month later: 2024-02-29
Wednesday 31 January 2024
2024-03-31T09:30:00+02:00 = 2024-03-31T07:30:00Z
2024-04-01 09:30 CEST
true 2024-03-31
```

A month later than January 31 is the last day of February: adding months keeps the day where the month has it and takes the month's last day otherwise. `TimeZone::get` reads the system's tz database (`/usr/share/zoneinfo`, or `$TZDIR`); `TimeZone::local()` is the user's zone. Windows and the browser have no tz database, so there only `UTC` and fixed offsets such as `+09:00` work (and `TimeZone::from_tzif` for a zone file the program brings). For the current time in a zone, write `Zoned::now(TimeZone::local())`. A local time that a daylight saving change skips or shows twice resolves forward by default; `in_zone_with(tz, Disambiguation::Reject)` makes it an error instead.

## Run tasks concurrently and wait for them

`task::spawn` starts a future as a task, and `task::join` waits for all of them and returns their results in spawn order. Any top-level `await` makes the script's `main` async, which `task::spawn` needs ([chapter 12](12-concurrency.html#async-and-tasks)).

```nabi
async fn fetch(id: i32) -> str {
    await task::sleep(Duration::ms(10));
    f"item-{id}"
}

let tasks = [task::spawn(fetch(1)), task::spawn(fetch(2)), task::spawn(fetch(3))];
let items = await task::join(tasks);
println(f"{items:?}");
```

```output
["item-1", "item-2", "item-3"]
```

## Share a counter or a list across threads

Use `Atomic` for a number and `Mutex` for anything else. `m.with(|v| ..)` holds the lock for one call; for a longer stretch, bind the guard with `using`, which releases it at the end of the block. A `[ref x]` capture won't compile here, because it isn't safe across threads ([chapter 12](12-concurrency.html#mutex-and-atomic)).

```nabi
use std::thread;

let hits = Atomic::new(0);
let seen = Mutex::new([]);

let hs = [];
for i in 1..=4 {
    hs.push(thread::spawn(|| {
        hits.fetch_add(1);
        seen.with(|v| v.push(i));
    }));
}
for h in hs { try h.join(); }

let all = seen.with(|v| v.clone());
all.sort();
println(f"{hits.load()} {all:?}");
```

```output
4 [1, 2, 3, 4]
```

## Handle errors

Write `throws` on a function that can fail and `try` on each call. `throw "msg"` throws a `ValueError` with that message. `catch` handles an error where it happens; `try? f() ?? default` swaps it for a default ([chapter 10](10-errors.html)).

```nabi
fn parse_port(s: str) throws -> u16 {
    let n: u16 = try s.parse();
    if n < 1024 { throw f"port {n} is reserved"; }
    n
}

let a = try parse_port("8080");
let b = try parse_port("80") catch e { println(f"error: {e}"); 8000 };
let c = try? parse_port("abc") ?? 3000;
println(f"{a} {b} {c}");
```

```output
error: port 80 is reserved
8080 8000 3000
```

## Parse JSON

JSON lives in the `json` package on [packages.nabi-lang.org](https://packages.nabi-lang.org), not in the standard library. Add it with `nabi add json` in a package, or `nabi add json app.nabi` for a script ([chapter 18](18-features-and-publishing.html#the-nabi-packages)).

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "json": Dep::version("^0.1.0"),
} }
```

```nabi norun
#[derive(json::Json)]
struct User { name: str; age: i32; tags: List<str> = []; }

// Untyped: walk a Value tree.
let v = try json::parse("{\"name\": \"ada\", \"langs\": [\"nabi\", \"c\"]}");
println(v.get("name")?.as_str() ?? "?");          // ada
println(v.pointer("/langs/0")?.as_str() ?? "?");  // nabi

// Typed: decode into your own struct, and back.
let u: User = try json::from_str("{\"name\": \"grace\", \"age\": 85}");
println(f"{u.name} {u.age}");                      // grace 85
println(json::to_str(u));                       // {"name":"grace","age":85,"tags":[]}
```

## Make an HTTP request, or serve HTTP

HTTP is the `http` package, added the same way as `json` above (`nabi add http`, or `nabi add http app.nabi` for a script). Its client and server are async, so the script uses `await` ([chapter 12](12-concurrency.html#async-and-tasks), [chapter 18](18-features-and-publishing.html#the-nabi-packages)).

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "http": Dep::version("^0.1.0"),
} }
```

A request:

```nabi norun
let r = try await http::get("https://example.com/");
println(f"{r.status} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

A server: one `async fn` handles every request, each in its own task.

```nabi norun
use http::{Request, Response};

async fn handle(req: Request) -> Response {
    Response::text(f"hello from {req.uri.path}\n")
}

println("listening on http://127.0.0.1:8080");
try await http::serve("127.0.0.1:8080", handle);
```

```text
$ curl http://127.0.0.1:8080/hi
hello from /hi
```

## Further reading

- [Examples](../examples/index.html): longer complete programs, among them a [command-line tool](../examples/cli-args.html), a [TCP echo server](../examples/tcp-echo.html), and a [JSON parser](../examples/json-parser.html) written from scratch.
- Standard library API: [`fs`](../std/std/fs.html), [`io`](../std/core/io.html), [`env`](../std/std/env.html), [`time`](../std/std/time.html), [`thread`](../std/std/thread.html), [`task`](../std/core/task.html), [`sync`](../std/core/sync.html)
- Next chapter: [21. Arrays and Numerics](21-arrays-and-numerics.html)
