# 檔案與資料

讀寫檔案與常見的資料格式。每個指令碼都在系統暫存目錄下的一個新目錄中工作，並在結束時刪除它。一行就能解決的問題（讀取檔案、讀取 stdin、引數、日期），請見[怎麼做……](../20-cookbook.html)。

## 計算行數、單字數與位元組數

幾行程式碼寫成的 `wc`：讀取檔案一次，然後計數。

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

## 寫入與讀取 CSV 檔案

只有幾個欄位時，手動合併與分割就夠了。可能含有逗號的欄位要加上引號；對於含有引號、null 與型別的真實資料，請使用 `table` 套件（[第一次資料分析](https://docs.nabi-lang.org/packages/table/latest)）。

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

## 設定檔

NABION 是標準函式庫的資料格式，也就是 Nabi 本身的字面值語法。`#[derive(Nabion)]` 會把它讀進結構，省略的欄位使用預設值，任何錯誤都會產生指出行號的錯誤訊息。

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

## 讀寫 JSON

JSON 由官方的 `json` 套件處理，第一次使用時會自動加入。`parse` 會給出一棵可以走訪的 `Value` 樹；`#[derive(json::Json)]` 能把 JSON 讀進結構，也能把結構寫回 JSON。

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

## 走訪目錄樹

`fs::walk` 以深度優先、依名稱順序走訪目錄下的每個項目。這裡用它加總每個 `.txt` 檔案的大小。

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

## 讀取環境變數並提供預設值

變數未設定時，`env::var` 會擲回錯誤，所以預設值寫在 `catch` 中。

```nabi
use std::env;

let editor = try env::var("NABI_ROSETTA_EDITOR") catch _ { "vi" };
let jobs = try? (try env::var("NABI_ROSETTA_JOBS")).parse::<u32>() ?? 4;
println(f"editor {editor}, {jobs} jobs");
```

```output
editor vi, 4 jobs
```

## 執行其他程式

`Command` 執行一個程式並收集它印出的內容。`output` 會等待程式結束。

```nabi build
use std::process::Command;

let out = try Command::new("git").args(["--version"]).output();
if out.status.success() {
    print(try str::from_utf8(out.stdout));
} else {
    println(f"git failed: {out.status}");
}
```

檢查網站時，這個範例只會建置而不執行，因為該機器上可能沒有 `git`；[行程範例](../../examples/process.html)有更多內容。
