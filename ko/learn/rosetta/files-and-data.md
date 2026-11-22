# 파일과 데이터

파일과 흔한 데이터 형식을 읽고 씁니다. 각 스크립트는 시스템 임시 디렉터리 아래의 새 디렉터리에서 일하고, 끝나면 지웁니다. 파일 읽기, 표준 입력, 인자, 날짜 같은 한 줄 답은 [어떻게 하나요](../20-cookbook.html)를 보세요.

## 줄, 단어, 바이트 세기

몇 줄로 쓰는 `wc`입니다. 파일을 한 번 읽은 다음 셉니다.

```nabi
use std::{env, fs, process};

let path = env::temp_dir() / f"wc_{process::id()}.txt";
try fs::write(path, "one two three\nfour five\n\nsix\n");

let text = try fs::read_to_str(path);
let lines = text.lines().count();
let words = text.split_whitespace().count();
println(f"{lines} lines, {words} words, {text.len()} bytes");
try fs::remove_file(path);
```

```output
4 lines, 6 words, 29 bytes
```

## CSV 파일 쓰고 읽기

열이 몇 개뿐이라면 손으로 잇고 나누는 것으로 충분합니다. 쉼표가 들어갈 수 있는 필드는 따옴표로 감싸세요. 따옴표, null, 타입이 있는 실제 데이터에는 `table` 패키지를 쓰세요([처음 해 보는 데이터 분석](https://packages.nabi-lang.org/docs/table/latest)).

```nabi
use std::{env, fs, process};

let rows = [("ada", 36, 1.75), ("grace", 85, 1.6), ("alan", 41, 1.8)];
let out = StrBuf::new();
out.push_str("name,age,height\n");
for (name, age, h) in rows { out.push_str(f"{name},{age},{h}\n"); }

let path = env::temp_dir() / f"people_{process::id()}.csv";
try fs::write(path, out.to_str());

let total = 0;
for line in (try fs::read_to_str(path)).lines().skip(1) {
    let fields = line.split(",").to_list();
    total += try fields[1].parse::<i32>();
}
println(f"mean age {total as f64 / rows.len() as f64:.1}");
try fs::remove_file(path);
```

```output
mean age 54.0
```

## 설정 파일

NABION은 표준 라이브러리의 데이터 형식으로, Nabi 자체의 리터럴 문법입니다. `#[derive(Nabion)]`은 이것을 구조체로 읽어 들이며, 빠진 필드에는 기본값을 쓰고 잘못된 곳은 그 줄을 짚는 에러를 냅니다.

```nabi
use std::nabion;

#[derive(Nabion)]
struct Config {
    name: str;
    port: u16 = 8080;
    debug: bool = false;
    hosts: List<str>;
}

let text = r#"Config { name: "demo", hosts: ["a.example", "b.example"], debug: true }"#;
let cfg: Config = try nabion::from_str(text);
println(f"{cfg.name} on port {cfg.port}, debug {cfg.debug}, {cfg.hosts.len()} hosts");

let bad: Config = try nabion::from_str(r#"Config { name: "x", hosts: [], port: 70000 }"#) catch e {
    println(f"error: {e}");
    return;
};
```

```output
demo on port 8080, debug true, 2 hosts
error: 1:38: integer 70000 is out of range for u16
```

## JSON 읽고 쓰기

JSON은 공식 패키지 `json`이 맡고, 처음 쓸 때 추가됩니다. `parse`는 따라 내려갈 수 있는 `Value` 트리를 주고, `#[derive(json::Json)]`은 구조체로 읽어 들이고 다시 씁니다.

```nabi
#[derive(json::Json)]
struct Point { x: f64; y: f64; label: str = ""; }

let v = try json::parse(r#"{"name": "route", "points": [{"x": 1, "y": 2}, {"x": 3.5, "y": -1, "label": "end"}]}"#);
println(v.get("name")?.as_str() ?? "?");
println(v.pointer("/points/1/label")?.as_str() ?? "?");

let p: Point = try json::from_str(r#"{"x": 1.5, "y": 2}"#);
println(f"{p.x} {p.y} {p.label:?}");
println(json::to_str(Point { x: 0.0, y: 1.0, label: "origin" }));
```

```output
route
end
1.5 2 ""
{"x":0.0,"y":1.0,"label":"origin"}
```

## 디렉터리 트리 순회하기

`fs::walk`는 디렉터리 아래의 모든 항목을 이름 순서대로 깊이 우선으로 방문합니다. 여기서는 `.txt` 파일의 크기를 모두 더합니다.

```nabi
use std::{env, fs, process};

let root = env::temp_dir() / f"walk_{process::id()}";
try fs::create_dir_all(root / "a" / "b");
try fs::write(root / "top.txt", "12345");
try fs::write(root / "a" / "mid.txt", "123");
try fs::write(root / "a" / "b" / "deep.txt", "1");
try fs::write(root / "a" / "skip.log", "ignored");

let (n, bytes) = (0, 0u64);
try for e in fs::walk(root) {
    if e.is_file() && e.path().as_str().ends_with(".txt") {
        n += 1;
        bytes += e.metadata().len();
    }
}
println(f"{n} text files, {bytes} bytes");
try fs::remove_dir_all(root);
```

```output
3 text files, 9 bytes
```

## 기본값과 함께 환경 변수 읽기

`env::var`는 변수가 설정되지 않았으면 에러를 던지므로, 기본값은 `catch`로 줍니다.

```nabi
use std::env;

let editor = try env::var("NABI_ROSETTA_EDITOR") catch _ { "vi" };
let jobs = try? (try env::var("NABI_ROSETTA_JOBS")).parse::<u32>() ?? 4;
println(f"editor {editor}, {jobs} jobs");
```

```output
editor vi, 4 jobs
```

## 다른 프로그램 실행하기

`Command`는 프로그램을 실행하고 출력을 모읍니다. `output`은 끝날 때까지 기다립니다.

```nabi build
use std::process::Command;

let out = try Command::new("git").args(["--version"]).output();
if out.status.success() {
    print(try str::from_utf8(out.stdout));
} else {
    println(f"git failed: {out.status}");
}
```

컴퓨터에 `git`이 없을 수도 있으므로, 이 블록은 사이트를 검사할 때 빌드만 합니다. 더 많은 내용은 [프로세스 예제](../../examples/process.html)에 있습니다.
