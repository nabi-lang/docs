# Files and Data

Reading and writing files and common data formats. Each script works in a fresh directory under the system's temporary directory and removes it at the end. For one-line answers (read a file, read stdin, arguments, dates) see [How Do I...](../20-cookbook.html).

## Count lines, words and bytes

`wc` in a few lines: read the file once, then count.

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

## Write and read a CSV file

For a few columns, joining and splitting by hand is enough. Quote any field that may contain a comma; for real data with quotes, nulls and types, use the `table` package ([A first data analysis](https://packages.nabi-lang.org/docs/table/latest)).

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

## A configuration file

NABION is the standard library's data format: Nabi's own literal syntax. `#[derive(Nabion)]` reads it into a struct, with defaults for the fields left out and an error that names the line for anything wrong.

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

## Read and write JSON

JSON is the official `json` package, added on first use. `parse` gives a `Value` tree to walk; `#[derive(json::Json)]` reads into a struct and writes one back.

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

## Walk a directory tree

`fs::walk` visits every entry under a directory, depth-first in name order. Here it adds up the size of every `.txt` file.

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

## Read an environment variable with a default

`env::var` throws when the variable is not set, so the default is a `catch`.

```nabi
use std::env;

let editor = try env::var("NABI_ROSETTA_EDITOR") catch _ { "vi" };
let jobs = try? (try env::var("NABI_ROSETTA_JOBS")).parse::<u32>() ?? 4;
println(f"editor {editor}, {jobs} jobs");
```

```output
editor vi, 4 jobs
```

## Run another program

`Command` runs a program and collects what it prints. `output` waits for it to finish.

```nabi build
use std::process::Command;

let out = try Command::new("git").args(["--version"]).output();
if out.status.success() {
    print(try str::from_utf8(out.stdout));
} else {
    println(f"git failed: {out.status}");
}
```

This one only builds when the site is checked, since the machine may have no `git`; the [process example](../../examples/process.html) shows more.
