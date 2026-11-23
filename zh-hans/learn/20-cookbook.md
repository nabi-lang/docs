# 20. 怎么做……

日常任务的简短解答。每个都是完整的程序，可以粘贴到文件里，用 `nabi run file.nabi` 运行。每个示例都链接到讲解它的章节。

## 读取文件

`fs::read_to_str` 把整个文件读成文本；`fs::write` 写入一个文件。两者都会抛出 `IoError`，所以调用处要标上 `try`（[第 13 章](13-io.html#读写文件)）。

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

## 逐行读取标准输入

`io::stdin().lines()` 逐行产出，不带换行符；读取出错时会抛出错误并结束循环，`try for` 把错误继续传出去（[第 13 章](13-io.html#标准-io-与环境)）。下面的输出是 `0 lines`，因为构建本页时没有输入；在本地运行并用管道传入一个文件（`nabi run count.nabi < file`）。

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

用 `printf 'a\nb\n' | nabi run lines.nabi` 运行，就能看到它给各行编号。

## 获取命令行参数

`env::args()` 是一个 `List<str>`，第 0 个元素是程序本身。参数写在 `--` 之后：`nabi run greet.nabi -- ada grace`（[第 13 章](13-io.html#标准-io-与环境)、[示例](../examples/cli-args.html)）。下面的输出是 `0 arguments`，因为构建本页时没有传参数；在本地带上参数运行。

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

## 以指定退出码退出

`process::exit(code)` 立即结束程序，`finally` 块和 `Drop` 都不会运行。脚本顶层未捕获的错误会自动以退出码 1 退出（[第 10 章](10-errors.html#脚本顶层的错误)）。

```nabi build
use std::{env, process};

let args = env::args();
if args.len() < 2 {
    eprintln("usage: greet NAME");
    process::exit(2);
}
println(f"hello, {args[1]}");
```

## 休眠，以及测量耗时

`Instant::now()` 读取单调时钟，`elapsed()` 给出一个 `Duration`。`thread::sleep` 阻塞当前线程；在异步代码中改用 `await task::sleep(..)`（[第 12 章](12-concurrency.html#async-与任务)、[`Duration` API](../../std/core/time.html#struct.Duration)）。

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

## 日期、时间与时区

日历功能在 `std::time` 中。`Date`、`Time` 和 `DateTime` 是日历和挂钟上显示的内容，不带时区；`Zoned` 是某个 `TimeZone` 中的一个时刻。`Display` 输出 ISO 8601 / RFC 3339，`parse` 读取这种格式，`format` / `parse_with` 接受 `%d %B %Y` 这样的 strftime 模式（[时区与模式](../spec/refs-standard-libraries/300-std/303-time.html#337-偏移时区与-zoned)、[`time` API](../../std/std/time.html)）。

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

1 月 31 日的一个月后是 2 月的最后一天：加月份时，如果目标月份有这一天就保持日期不变，否则取该月的最后一天。`TimeZone::get` 读取系统的 tz 数据库（`/usr/share/zoneinfo` 或 `$TZDIR`）；`TimeZone::local()` 是用户所在的时区。Windows 和浏览器没有 tz 数据库，所以在那里只能用 `UTC` 和 `+09:00` 这样的固定偏移（以及用 `TimeZone::from_tzif` 读取程序自带的时区文件）。要获取某个时区的当前时间，写 `Zoned::now(TimeZone::local())`。夏令时切换时被跳过或出现两次的本地时间，默认向后解析；`in_zone_with(tz, Disambiguation::Reject)` 则把它当作错误。

## 并发运行任务并等待完成

`task::spawn` 把一个 future 作为任务启动，`task::join` 等待所有任务完成，并按启动顺序返回结果。只要有顶层的 `await`，脚本的 `main` 就是异步的，而 `task::spawn` 需要这一点（[第 12 章](12-concurrency.html#async-与任务)）。

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

## 在线程间共享计数器或列表

数字用 `Atomic`，其他一切用 `Mutex`。`m.with(|v| ..)` 只在一次调用期间持有锁；需要持有更久时，用 `using` 绑定守卫，它会在块结束时释放锁。这里用 `[ref x]` 捕获无法通过编译，因为它在线程间不安全（[第 12 章](12-concurrency.html#mutex-与-atomic)）。

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

## 处理错误

可能失败的函数写上 `throws`，每次调用写上 `try`。`throw "msg"` 抛出带该消息的 `ValueError`。`catch` 在出错的地方处理错误；`try? f() ?? default` 用默认值代替错误（[第 10 章](10-errors.html)）。

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

## 解析 JSON

JSON 在 [packages.nabi-lang.org](https://packages.nabi-lang.org) 上的 `json` 包中，不在标准库里。在包中用 `nabi add json` 添加，脚本则用 `nabi add json app.nabi`（[第 18 章](18-features-and-publishing.html#nabi-官方包)）。

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

## 发送 HTTP 请求，或提供 HTTP 服务

HTTP 由 `http` 包提供，添加方式与上面的 `json` 相同（`nabi add http`，脚本则用 `nabi add http app.nabi`）。它的客户端和服务器都是异步的，所以脚本要用 `await`（[第 12 章](12-concurrency.html#async-与任务)、[第 18 章](18-features-and-publishing.html#nabi-官方包)）。

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "http": Dep::version("^0.1.0"),
} }
```

一个请求：

```nabi norun
let r = try await http::get("https://example.com/");
println(f"{r.status} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

一个服务器：一个 `async fn` 处理所有请求，每个请求在各自的任务中运行。

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

## 延伸阅读

- [示例](../examples/index.html)：更长的完整程序，其中有一个[命令行工具](../examples/cli-args.html)、一个 [TCP 回显服务器](../examples/tcp-echo.html)，以及一个从零写起的 [JSON 解析器](../examples/json-parser.html)。
- 标准库 API：[`fs`](../../std/std/fs.html)、[`io`](../../std/core/io.html)、[`env`](../../std/std/env.html)、[`time`](../../std/std/time.html)、[`thread`](../../std/std/thread.html)、[`task`](../../std/core/task.html)、[`sync`](../../std/core/sync.html)
- 下一章：[21. 数组与数值计算](21-arrays-and-numerics.html)
