# 命令行工具

`mgrep` 是一个类似 `grep` 的小工具，它手动把命令行参数解析到一个结构体中，并用 `throws` 处理 `--help` 和错误的参数。

## 代码

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

不带参数运行时，它会依次尝试几个内置的示例调用。

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

## 讲解

- `env::args()` 是一个 `[0]` 处为程序名的 `List<str>`，所以用 `skip(1).to_list()` 只保留其余部分（[`env::args`](../../std/std/env.html#fn.args)）。
- 选项存放在带字段默认值的 `Options` 结构体中。有默认值的字段可以省略，例如 `Options { pattern: "" }`（[结构体、枚举与 trait](../learn/06-structs-enums-traits.html)，[2.3.3 字段默认值](../spec/refs-lang/002-types/203-struct.html#233-field-default-values)）。
- 用 `|` 连接的字符串字面量模式，例如 `match a { "-i" | "--ignore-case" => ... }`，可以同时接受短形式和长形式。赋值是语句而不是表达式，所以这些分支的主体包在 `{ ...; }` 块中（[控制流](../learn/04-control-flow.html)）。
- 错误的参数以 `ArgError` 枚举抛出。`run` 声明了 `throws (ArgError | IoError)`，`main` 为每种类型使用一个 `catch` 来选择退出码（2 或 1）（[错误处理](../learn/10-errors.html)，[3.14.4 `catch`](../spec/refs-lang/003-expressions/314-error-control-flow.html#3144-catch-postfix-handling-expression)）。
- `-m` 之后的数字用 `parse::<usize>()` 读取；失败时，`catch _ { throw ...; }` 把它变成更友好的错误。`opts.max` 是 `usize | none`，所以比较时用 `?? usize::MAX` 提供默认值。

## 运行

要搜索真实的文件，构建可执行文件并传入参数。

```bash
nabic build mgrep.nabi -o mgrep
./mgrep -n -i nabi notes.txt todo.txt
./mgrep --count error app.log
```
