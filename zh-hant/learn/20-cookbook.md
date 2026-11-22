# 20. 怎麼做……

日常工作的簡短解答。每一則都是完整的程式，可以貼進檔案，用 `nabi run file.nabi` 執行。每則食譜都附有說明它的章節連結。

## 讀取檔案

`fs::read_to_str` 以文字形式讀取整個檔案；`fs::write` 寫入一個檔案。兩者都會擲回 `IoError`，所以呼叫時要加上 `try`（[第 13 章](13-io.html#讀寫檔案)）。

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

## 逐行讀取標準輸入

`io::stdin().lines()` 逐一產生不含換行字元的每一行；讀取錯誤會以擲回的方式結束迴圈，`try for` 則把錯誤往外傳（[第 13 章](13-io.html#標準-io-與環境)）。下面的輸出是 `0 lines`，因為本頁建置時沒有輸入；請在本機執行，並把檔案導入（`nabi run count.nabi < file`）。

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

以 `printf 'a\nb\n' | nabi run lines.nabi` 執行，就能看到它為每一行編號。

## 取得命令列引數

`env::args()` 是一個 `List<str>`；第 0 個元素是程式本身。引數寫在 `--` 之後：`nabi run greet.nabi -- ada grace`（[第 13 章](13-io.html#標準-io-與環境)、[範例](../examples/cli-args.html)）。下面的輸出是 `0 arguments`，因為本頁建置時沒有引數；請在本機帶上一些引數執行。

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

## 以結束碼結束程式

`process::exit(code)` 會立即結束程式。`finally` 區塊與 `Drop` 都不會執行。指令碼最上層未被攔截的錯誤，本身就會以結束碼 1 結束（[第 10 章](10-errors.html#指令碼頂層的錯誤)）。

```nabi build
use std::{env, process};

let args = env::args();
if args.len() < 2 {
    eprintln("usage: greet NAME");
    process::exit(2);
}
println(f"hello, {args[1]}");
```

## 暫停，以及量測某件事花了多久

`Instant::now()` 讀取單調時鐘（monotonic clock），`elapsed()` 給出一個 `Duration`。`thread::sleep` 會阻塞目前的執行緒；在非同步程式碼中，請改用 `await task::sleep(..)`（[第 12 章](12-concurrency.html#async-與任務)、[`Duration` API](../../std/core/time.html#struct.Duration)）。

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

## 日期、時間與時區

日曆功能在 `std::time` 裡。`Date`、`Time` 與 `DateTime` 是日曆和掛鐘上顯示的內容，不帶時區；`Zoned` 則是某個 `TimeZone` 中的一個時間點。`Display` 寫出 ISO 8601 / RFC 3339 格式，`parse` 讀取這種格式，`format` / `parse_with` 則接受 `%d %B %Y` 之類的 strftime 樣式（[時區與樣式](../spec/refs-standard-libraries/300-std/303-time.html#337-offsets-time-zones-and-zoned)、[`time` API](../../std/std/time.html)）。

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

1 月 31 日的一個月後是 2 月的最後一天：加上月份時，若該月有同一天就保留這一天，否則取該月的最後一天。`TimeZone::get` 讀取系統的 tz 資料庫（`/usr/share/zoneinfo` 或 `$TZDIR`）；`TimeZone::local()` 是使用者所在的時區。Windows 和瀏覽器沒有 tz 資料庫，所以在那裡只有 `UTC` 與 `+09:00` 之類的固定時差可用（另外還有 `TimeZone::from_tzif`，用於程式自帶的時區檔案）。要取得某個時區的目前時間，寫 `Zoned::now(TimeZone::local())`。日光節約時間切換時被跳過或出現兩次的本地時間，預設會往後解析；`in_zone_with(tz, Disambiguation::Reject)` 則會讓它成為錯誤。

## 並行執行任務並等待它們

`task::spawn` 把一個 future 當作任務啟動，`task::join` 則等待所有任務，並依啟動順序回傳結果。只要最上層出現任何 `await`，指令碼的 `main` 就會成為非同步函式，而 `task::spawn` 需要這一點（[第 12 章](12-concurrency.html#async-與任務)）。

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

## 在執行緒之間共用計數器或串列

數字用 `Atomic`，其他東西用 `Mutex`。`m.with(|v| ..)` 只在一次呼叫期間持有鎖；需要持有較長一段時間時，就用 `using` 繫結 guard，它會在區塊結束時釋放鎖。這裡用 `[ref x]` 擷取無法編譯，因為它在執行緒之間並不安全（[第 12 章](12-concurrency.html#mutex-與-atomic)）。

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

## 處理錯誤

在可能失敗的函式上寫 `throws`，每次呼叫時寫 `try`。`throw "msg"` 會擲回帶有該訊息的 `ValueError`。`catch` 在錯誤發生的地方處理它；`try? f() ?? default` 則把錯誤換成預設值（[第 10 章](10-errors.html)）。

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

## 剖析 JSON

JSON 位於 [packages.nabi-lang.org](https://packages.nabi-lang.org) 上的 `json` 套件，而不在標準函式庫中。在套件中用 `nabi add json` 加入它，指令碼則用 `nabi add json app.nabi`（[第 18 章](18-features-and-publishing.html#nabi-套件)）。

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

## 發出 HTTP 請求，或提供 HTTP 服務

HTTP 由 `http` 套件提供，加入方式與上面的 `json` 相同（`nabi add http`，指令碼則用 `nabi add http app.nabi`）。它的用戶端與伺服器都是非同步的，所以指令碼要使用 `await`（[第 12 章](12-concurrency.html#async-與任務)、[第 18 章](18-features-and-publishing.html#nabi-套件)）。

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "http": Dep::version("^0.1.0"),
} }
```

發出請求：

```nabi norun
let r = try await http::get("https://example.com/");
println(f"{r.status} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

伺服器：由一個 `async fn` 處理每個請求，每個請求各在自己的任務中執行。

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

## 延伸閱讀

- [範例](../examples/index.html)：較長的完整程式，其中包括一個[命令列工具](../examples/cli-args.html)、一個 [TCP echo 伺服器](../examples/tcp-echo.html)，以及一個從零寫起的 [JSON 剖析器](../examples/json-parser.html)。
- 標準函式庫 API：[`fs`](../../std/std/fs.html)、[`io`](../../std/core/io.html)、[`env`](../../std/std/env.html)、[`time`](../../std/std/time.html)、[`thread`](../../std/std/thread.html)、[`task`](../../std/core/task.html)、[`sync`](../../std/core/sync.html)
