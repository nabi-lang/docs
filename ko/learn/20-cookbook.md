# 20. 어떻게 하나요

자주 하는 작업의 짧은 답입니다. 각 답은 파일에 붙여 넣고 `nabi run file.nabi`로 실행할 수 있는 완전한 프로그램입니다. 자세히 설명하는 장으로 가는 링크도 붙어 있습니다.

## 파일 읽기

`fs::read_to_str`은 파일 전체를 텍스트로 읽고, `fs::write`는 파일을 씁니다. 둘 다 `IoError`를 던지므로 호출에 `try`를 붙입니다([13장](13-io.html#파일-읽고-쓰기)).

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

## 표준 입력을 한 줄씩 읽기

`io::stdin().lines()`는 줄바꿈을 뺀 한 줄씩을 내놓습니다. 읽기 에러는 던져서 루프를 끝내고, `try for`가 그 에러를 전파합니다([13장](13-io.html#표준-io와-환경)). 이 페이지는 입력 없이 빌드되므로 아래 출력은 `0 lines`입니다. 직접 실행하면서 파일을 파이프로 넣어 보세요(`nabi run count.nabi < file`).

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

`printf 'a\nb\n' | nabi run lines.nabi`로 실행하면 줄마다 번호가 붙어 나옵니다.

## 명령줄 인자 얻기

`env::args()`는 `List<str>`이고, 원소 0은 프로그램 자신입니다. 인자는 `nabi run greet.nabi -- ada grace`처럼 `--` 뒤에 넘깁니다([13장](13-io.html#표준-io와-환경), [예제](../examples/cli-args.html)). 이 페이지는 인자 없이 빌드되므로 아래 출력은 `0 arguments`입니다. 직접 인자를 주어 실행해 보세요.

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

## 종료 코드와 함께 끝내기

`process::exit(code)`는 프로그램을 즉시 끝냅니다. `finally` 블록과 `Drop`은 실행되지 않습니다. 스크립트의 최상위에서 아무도 잡지 않은 에러는 저절로 코드 1로 프로그램을 끝냅니다([10장](10-errors.html#스크립트-최상위의-에러)).

```nabi build
use std::{env, process};

let args = env::args();
if args.len() < 2 {
    eprintln("usage: greet NAME");
    process::exit(2);
}
println(f"hello, {args[1]}");
```

## 잠들기, 그리고 걸린 시간 재기

`Instant::now()`는 단조 시계(monotonic clock)를 읽고, `elapsed()`는 `Duration`을 줍니다. `thread::sleep`은 현재 스레드를 막습니다. 비동기 코드에서는 대신 `await task::sleep(..)`을 씁니다([12장](12-concurrency.html#async와-task), [`Duration` API](../../std/core/time.html#struct.Duration)).

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

## 날짜, 시각, 시간대

달력은 `std::time`에 있습니다. `Date`, `Time`, `DateTime`은 시간대 없이 달력과 벽시계가 보여 주는 값이고, `Zoned`는 `TimeZone` 안의 한 순간입니다. `Display`는 ISO 8601 / RFC 3339로 쓰고 `parse`는 그것을 읽으며, `format` / `parse_with`는 `%d %B %Y` 같은 strftime 패턴을 받습니다([시간대와 패턴](../spec/refs-standard-libraries/300-std/303-time.html#337-오프셋-시간대-zoned), [`time` API](../../std/std/time.html)).

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

1월 31일의 한 달 뒤는 2월의 마지막 날입니다. 월을 더하면 그 달에 같은 날이 있으면 그날을, 없으면 그 달의 마지막 날을 씁니다. `TimeZone::get`은 시스템의 tz 데이터베이스(`/usr/share/zoneinfo`, 또는 `$TZDIR`)를 읽고, `TimeZone::local()`은 사용자의 시간대입니다. Windows와 브라우저에는 tz 데이터베이스가 없으므로 거기서는 `UTC`와 `+09:00` 같은 고정 오프셋만 동작합니다(프로그램이 가져온 시간대 파일은 `TimeZone::from_tzif`로 읽습니다). 어느 시간대의 현재 시각은 `Zoned::now(TimeZone::local())`로 씁니다. 일광 절약 시간 변경이 건너뛰거나 두 번 보여 주는 지역 시각은 기본으로 앞쪽으로 정해집니다. `in_zone_with(tz, Disambiguation::Reject)`를 쓰면 대신 에러가 됩니다.

## task를 동시에 실행하고 기다리기

`task::spawn`은 future를 task로 시작하고, `task::join`은 task를 모두 기다려 결과를 띄운 순서대로 돌려줍니다. 최상위에 `await`가 하나라도 있으면 스크립트의 `main`이 비동기 함수가 됩니다. `task::spawn`은 main이 비동기일 때만 쓸 수 있습니다([12장](12-concurrency.html#async와-task)).

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

## 스레드 사이에서 카운터나 리스트 공유하기

숫자에는 `Atomic`을, 그 밖의 것에는 `Mutex`를 씁니다. `m.with(|v| ..)`는 호출 한 번 동안 잠금을 잡습니다. 더 길게 잡으려면 가드를 `using`으로 바인딩하고, 블록 끝에서 잠금이 풀립니다. 여기서 `[ref x]` 캡처를 쓰면 스레드 사이에서 안전하지 않으므로 컴파일되지 않습니다([12장](12-concurrency.html#mutex와-atomic)).

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

## 에러 처리하기

실패할 수 있는 함수에는 `throws`를, 각 호출에는 `try`를 씁니다. `throw "msg"`는 그 메시지의 `ValueError`를 던집니다. `catch`는 에러가 난 자리에서 처리하고, `try? f() ?? default`는 에러를 기본값으로 바꿉니다([10장](10-errors.html)).

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

## JSON 파싱하기

JSON은 표준 라이브러리가 아니라 [packages.nabi-lang.org](https://packages.nabi-lang.org)의 `json` 패키지에 있습니다. 패키지에서는 `nabi add json`으로, 스크립트에서는 `nabi add json app.nabi`로 추가합니다([18장](18-features-and-publishing.html#nabi-프로젝트의-패키지)).

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

## HTTP 요청 보내기, HTTP 서비스하기

HTTP는 `http` 패키지에 있고, 위의 `json`과 같은 방법으로 추가합니다(`nabi add http`, 스크립트는 `nabi add http app.nabi`). 클라이언트와 서버가 비동기이므로 스크립트에서 `await`를 씁니다([12장](12-concurrency.html#async와-task), [18장](18-features-and-publishing.html#nabi-프로젝트의-패키지)).

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "http": Dep::version("^0.1.0"),
} }
```

요청을 보내는 쪽입니다.

```nabi norun
let r = try await http::get("https://example.com/");
println(f"{r.status} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

서버 쪽입니다. `async fn` 하나가 모든 요청을 처리하고, 요청마다 자기 task에서 실행됩니다.

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

## 더 읽을거리

- [예제](../examples/index.html): 더 긴 완전한 프로그램들입니다. [명령줄 도구](../examples/cli-args.html), [TCP echo 서버](../examples/tcp-echo.html), 처음부터 작성한 [JSON 파서](../examples/json-parser.html) 등이 있습니다.
- 표준 라이브러리 API: [`fs`](../../std/std/fs.html), [`io`](../../std/core/io.html), [`env`](../../std/std/env.html), [`time`](../../std/std/time.html), [`thread`](../../std/std/thread.html), [`task`](../../std/core/task.html), [`sync`](../../std/core/sync.html)
- 다음 장: [21. 배열과 수치 계산](21-arrays-and-numerics.html)
