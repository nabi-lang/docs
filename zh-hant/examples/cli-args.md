# 命令列工具

`mgrep` 是一個類似 `grep` 的小工具，它手動把命令列引數剖析成結構，並以 `throws` 處理 `--help` 和錯誤的引數。

## 程式碼

```nabi
// mgrep: print the lines of FILEs that contain PATTERN.
use std::env;
use std::fs;
use core::io::IoError;

const USAGE: str = "usage: mgrep [-i] [-n] [-c] [-m NUM] PATTERN [FILE]...
  -i, --ignore-case   match case-insensitively
  -n, --line-number   prefix each line with its number
  -c, --count         print only the number of matching lines
  -m, --max NUM       stop after NUM matches
  -h, --help          show this help";

// Used when no FILE is given (the demo never reads stdin).
const SAMPLE: str = "Roses are red,
violets are blue.
nabi is fast,
and so are you.";

enum ArgError {
    Unknown(str),
    MissingValue(str),
    BadNumber(str),
    NoPattern,
}

impl Display for ArgError {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        try f.write_str(match self {
            ArgError::Unknown(a) => f"unknown option '{a}'",
            ArgError::MissingValue(a) => f"option '{a}' needs a value",
            ArgError::BadNumber(v) => f"'{v}' is not a number",
            ArgError::NoPattern => "missing PATTERN",
        })
    }
}

struct Options {
    pattern: str;
    ignore_case: bool = false;
    line_numbers: bool = false;
    count_only: bool = false;
    max: usize | none = none;
    files: List<str> = [];
    help: bool = false;
}

fn parse_args(args: List<str>) throws (ArgError) -> Options {
    let opts = Options { pattern: "" };
    let have_pattern = false;
    let i = 0;
    while i < args.len() {
        let a = args[i];
        match a {
            "-h" | "--help" => { opts.help = true; },
            "-i" | "--ignore-case" => { opts.ignore_case = true; },
            "-n" | "--line-number" => { opts.line_numbers = true; },
            "-c" | "--count" => { opts.count_only = true; },
            "-m" | "--max" => {
                i += 1;
                if i >= args.len() { throw ArgError::MissingValue(a); }
                opts.max = try args[i].parse::<usize>() catch _ { throw ArgError::BadNumber(args[i]); };
            },
            _ => {
                if a.starts_with("-") && a.len() > 1 { throw ArgError::Unknown(a); }
                if have_pattern { opts.files.push(a); } else { opts.pattern = a; have_pattern = true; }
            },
        }
        i += 1;
    }
    if !have_pattern && !opts.help { throw ArgError::NoPattern; }
    opts
}

fn search(opts: Options, name: str, text: str) {
    let needle = if opts.ignore_case { opts.pattern.to_lowercase() } else { opts.pattern };
    let hits = 0;
    for (n, line) in text.lines().enumerate(start: 1) {
        let hay = if opts.ignore_case { line.to_lowercase() } else { line };
        if needle !in hay { continue; }
        hits += 1;
        if !opts.count_only {
            let prefix = if opts.files.len() > 1 { f"{name}:" } else { "" };
            if opts.line_numbers { println(f"{prefix}{n}:{line}"); } else { println(f"{prefix}{line}"); }
        }
        if hits == (opts.max ?? usize::MAX) { break; }
    }
    if opts.count_only { println(f"{hits}"); }
}

fn run(args: List<str>) throws (ArgError | IoError) {
    let opts = try parse_args(args);
    if opts.help { println(USAGE); return; }
    if opts.files.is_empty() { search(opts, "-", SAMPLE); return; }
    for path in opts.files {
        search(opts, path, try fs::read_to_str(path));
    }
}

fn main() -> i32 {
    let args = env::args().skip(1).to_list();
    // No arguments: demonstrate a few invocations on the built-in sample.
    let runs = if args.is_empty() {
        [["--help"], ["-n", "are"], ["-i", "-c", "NABI"], ["-m", "1", "are"], ["-x", "are"], ["--max"]]
    } else {
        [args]
    };
    let status = 0;
    for r in runs {
        if args.is_empty() { println(f"$ mgrep {r.join(" ")}"); }
        try run(r)
        catch e: ArgError { println(f"mgrep: {e}\n(try --help)"); status = 2; }
        catch e: IoError { println(f"mgrep: {e}"); status = 1; };
    }
    if args.is_empty() { 0 } else { status }
}
```

不帶引數執行時，它會依序嘗試幾組內建的範例呼叫。

```output
$ mgrep --help
usage: mgrep [-i] [-n] [-c] [-m NUM] PATTERN [FILE]...
  -i, --ignore-case   match case-insensitively
  -n, --line-number   prefix each line with its number
  -c, --count         print only the number of matching lines
  -m, --max NUM       stop after NUM matches
  -h, --help          show this help
$ mgrep -n are
1:Roses are red,
2:violets are blue.
4:and so are you.
$ mgrep -i -c NABI
1
$ mgrep -m 1 are
Roses are red,
$ mgrep -x are
mgrep: unknown option '-x'
(try --help)
$ mgrep --max
mgrep: option '--max' needs a value
(try --help)
```

## 說明

- `env::args()` 是一個 `List<str>`，`[0]` 是程式名稱，所以 `skip(1).to_list()` 保留其餘部分（[`env::args`](../../std/std/env.html#fn.args)）。
- 選項存放在帶有欄位預設值的 `Options` 結構中。有預設值的欄位可以省略，例如 `Options { pattern: "" }`（[結構、列舉與 trait](../learn/06-structs-enums-traits.html)、[2.3.3 欄位預設值](../spec/refs-lang/002-types/203-struct.html#233-欄位預設值)）。
- 以 `|` 連接的字串字面值模式，例如 `match a { "-i" | "--ignore-case" => ... }`，可同時接受短形式與長形式。指派是陳述式而不是運算式，所以這些分支的主體包在 `{ ...; }` 區塊中（[控制流程](../learn/04-control-flow.html)）。
- 錯誤的引數以 `ArgError` 列舉擲回。`run` 宣告了 `throws (ArgError | IoError)`，`main` 則對每個型別各用一個 `catch` 來選擇結束碼（2 或 1）（[錯誤](../learn/10-errors.html)、[3.14.4 `catch`](../spec/refs-lang/003-expressions/314-error-control-flow.html#3144-catch後綴處理運算式)）。
- `-m` 後面的數字以 `parse::<usize>()` 讀取；失敗時，`catch _ { throw ...; }` 把它轉成更友善的錯誤。`opts.max` 是 `usize | none`，所以比較時以 `?? usize::MAX` 提供預設值。

## 執行

要搜尋真正的檔案，請建置執行檔並傳入引數。

```bash
nabic build mgrep.nabi -o mgrep
./mgrep -n -i nabi notes.txt todo.txt
./mgrep --count error app.log
```
