# 文件与数据

读写文件和常见的数据格式。每个脚本都在系统临时目录下的一个新目录中工作，并在结束时删除它。一行就能解决的问题（读文件、读 stdin、命令行参数、日期）见[怎么做……](../20-cookbook.html)。

## 统计行数、单词数和字节数

几行代码写出 `wc`：把文件读一次，然后计数。

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

## 写入和读取 CSV 文件

只有几列时，手动连接和拆分就够了。可能含有逗号的字段要加引号；对于带引号、空值和类型的真实数据，请使用 `table` 包（[第一次数据分析](https://docs.nabi-lang.org/packages/table/latest)）。

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

## 配置文件

NABION 是标准库的数据格式，也就是 Nabi 自己的字面量语法。`#[derive(Nabion)]` 把它读进结构体，省略的字段使用默认值，出错时错误信息会指出所在的行。

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

## 读写 JSON

JSON 由官方包 `json` 提供，第一次使用时自动添加。`parse` 给出一棵可以遍历的 `Value` 树；`#[derive(json::Json)]` 把 JSON 读进结构体，也能把结构体写回 JSON。

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

## 遍历目录树

`fs::walk` 访问目录下的每个条目，按名称顺序深度优先。这里用它累加每个 `.txt` 文件的大小。

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

## 读取环境变量并提供默认值

变量未设置时 `env::var` 会抛出错误，所以默认值写在 `catch` 中。

```nabi
use std::env;

let editor = try env::var("NABI_ROSETTA_EDITOR") catch _ { "vi" };
let jobs = try? (try env::var("NABI_ROSETTA_JOBS")).parse::<u32>() ?? 4;
println(f"editor {editor}, {jobs} jobs");
```

```output
editor vi, 4 jobs
```

## 运行另一个程序

`Command` 运行一个程序并收集它的输出。`output` 等待它结束。

```nabi build
use std::process::Command;

let out = try Command::new("git").args(["--version"]).output();
if out.status.success() {
    print(try str::from_utf8(out.stdout));
} else {
    println(f"git failed: {out.status}");
}
```

由于机器上可能没有 `git`，这个示例只在检查网站时构建；[进程示例](../../examples/process.html)中有更多内容。
