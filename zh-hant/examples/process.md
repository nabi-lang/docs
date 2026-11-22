# 子行程

用 `process::run` 和 `std::process::Command` 執行 `echo`、shell 管線和 `tr` 等外部程式，並處理它們的輸出、結束碼、環境變數與標準輸入。

## 程式碼

```nabi
use std::process;
use std::process::{Command, Stdio, Output};

// Captured bytes as trimmed text.
fn text(bytes: List<u8>) -> str {
    (try str::from_utf8(bytes) catch _ { "<not utf-8>" }).trim()
}

fn show(label: str, out: Output) {
    println(f"{label}: [{text(out.stdout)}] ({out.status})");
}

fn main() {
    // 1. Run a program and take what it prints as text.
    let said = try process::run("echo", ["hello", "from", "a child"]);
    println(f"echo: [{said.trim()}]");

    // 2. A shell pipeline: the shell does the piping for us.
    let sorted = try Command::new("sh")
        .args(["-c", "printf 'pear\\napple\\nfig\\napple\\n' | sort | uniq -c | sort -rn"])
        .output();
    println("word counts:");
    for line in text(sorted.stdout).lines() { println(f"  {line.trim()}"); }

    // 3. Exit codes and stderr.
    let failed = try Command::new("sh").args(["-c", "echo oops >&2; exit 3"]).output();
    println(f"success={failed.status.success()} code={failed.status.code() ?? -1} stderr=[{text(failed.stderr)}]");

    // 4. Environment variables for the child only.
    let greet = try Command::new("sh")
        .args(["-c", "echo \"$GREETING, $NAME\""])
        .env("GREETING", "annyeong")
        .env("NAME", "nabi")
        .output();
    show("env", greet);

    let clean = try Command::new("/usr/bin/env").env_clear().env("ONLY", "1").output();
    show("env_clear", clean);

    // 5. Feed the child's stdin ourselves and read its stdout.
    let child = try Command::new("tr").args(["a-z", "A-Z"])
        .stdin(Stdio::Piped)
        .stdout(Stdio::Piped)
        .spawn();
    let input = child.take_stdin() ?? panic("stdin not piped");
    try input.write_all("shout this\n");
    try input.close();   // EOF lets `tr` finish
    let output = child.take_stdout() ?? panic("stdout not piped");
    let shouted = try output.read_to_str();
    try output.close();
    println(f"tr: [{shouted.trim()}] ({try child.wait()})");

    // 6. A working directory for the child.
    show("pwd", try Command::new("pwd").current_dir("/").output());

    // 7. One base command, several variants: each builder call returns a
    //    changed copy and leaves `base` as it was.
    let base = Command::new("sh").args(["-c", "echo $0 $1", "greet"]);
    show("variant", try base.arg("hi").output());
    show("base", try base.output());

    // 8. A child ended by a signal has no exit code; `signal()` names it.
    let killed = try Command::new("sh").args(["-c", "kill -9 $$"]).status();
    println(f"killed: code={killed.code() ?? -1} signal={killed.signal():?}");

    // 9. A missing program is an IoError, not a crash.
    let ok = try Command::new("no-such-program").status().success() catch e {
        println(f"spawn failed: {e.kind():?}");
        false
    };
    println(f"ok={ok}");
}
```

```output
echo: [hello from a child]
word counts:
  2 apple
  1 pear
  1 fig
success=false code=3 stderr=[oops]
env: [annyeong, nabi] (exit status: 0)
env_clear: [ONLY=1] (exit status: 0)
tr: [SHOUT THIS] (exit status: 0)
pwd: [/] (exit status: 0)
variant: [greet hi] (exit status: 0)
base: [greet] (exit status: 0)
killed: code=-1 signal=Kill
spawn failed: NotFound
ok=false
```

## 說明

- `process::run(program, args)` 把程式執行到結束，並以文字回傳它的 stdout；結束狀態表示失敗時，會擲回帶有 stderr 的 `ExitStatusError`（[`process::run`](../../std/std/process.html#fn.run)）。`main` 沒有宣告 `throws`：它放行的錯誤會被印出，程式以狀態 1 結束。
- [`Command`](../../std/std/process.html#struct.Command) 是建構器。`args`、`env`、`current_dir` 和 `stdin` 等方法回傳更新後的指令，所以呼叫可以在同一行串接，最後以 `output()`、`status()` 或 `spawn()` 之一結束（[3.6.1 `Command` 建構器](../spec/refs-standard-libraries/300-std/306-process.html#361-command-建構器)）。
- `output()` 等待子行程結束，並把 stdout 和 stderr 都收集到 [`Output`](../../std/std/process.html#struct.Output) 中。它同時清空兩條管道，所以子行程永遠不會因為管道滿了而卡住。`str::from_utf8` 把位元組轉成字串。
- 管線（`|`）由 `sh -c` 處理，而不是由 nabi 處理。需要 shell 語法時，這是最簡單的做法。
- `env` 只為子行程加入變數，`env_clear` 則清空繼承來的環境。父行程的環境不會改變。
- `spawn()` 不等待，直接回傳 [`Child`](../../std/std/process.html#struct.Child)。`Child` 持有以 `Stdio::Piped` 開啟的管道，直到你取走它們：`child.take_stdin()` 移交 stdin 管道並留下 `none`，所以管道只有一個擁有者，就是你。從未取走的管道會在 `Child` 釋放時關閉。寫入 `stdin`，以 `close()` 送出 EOF，再讀取 `stdout`，同樣把它關閉，然後 `wait()`。管道的關閉可能失敗，所以編譯器要求你自己關閉，而不是替你關閉。
- `Command` 的各個複本互相獨立：建構器方法回傳修改後的複本，所以 `base.arg("hi")` 不會改變 `base`，一個基礎指令可以啟動多個變形。結果被丟棄的建構器呼叫會產生警告，因為它什麼也沒改變。
- `signal()` 來自 `std::os::unix::process` 中的 `ExitStatusExt`。在 Linux 和 macOS 上它位於 prelude 中，所以不需要 `use`；在 Windows 上則不存在（[3.10 `os::`](../spec/refs-standard-libraries/300-std/310-os.html)）。
- 找不到執行檔時會擲回 `IoError`。`try ... catch e { ... }` 當場攔截它並以預設值代替（[3.14 錯誤控制流程](../spec/refs-lang/003-expressions/314-error-control-flow.html)、[錯誤](../learn/10-errors.html)）。
