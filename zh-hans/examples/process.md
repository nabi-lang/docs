# 子进程

用 `process::run` 和 `std::process::Command` 运行 `echo`、shell 管道和 `tr` 等外部程序，并处理它们的输出、退出码、环境变量和标准输入。

## 代码

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

## 讲解

- `process::run(program, args)` 把程序运行到结束，并以文本返回它的标准输出；退出状态表示失败时，会抛出携带标准错误输出的 `ExitStatusError`（[`process::run`](../../std/std/process.html#fn.run)）。`main` 没有声明 `throws`：它放过的错误会被输出，程序以状态 1 退出。
- [`Command`](../../std/std/process.html#struct.Command) 是一个构建器。`args`、`env`、`current_dir`、`stdin` 等方法返回更新后的命令，所以可以在一行中链式调用，最后以 `output()`、`status()` 或 `spawn()` 之一结束（[3.6.1 `Command` 构建器](../spec/refs-standard-libraries/300-std/306-process.html#361-command-构建器)）。
- `output()` 等待子进程结束，并把标准输出和标准错误输出都收集到 [`Output`](../../std/std/process.html#struct.Output) 中。它同时读取两个管道，所以子进程不会因为管道写满而停住。`str::from_utf8` 把字节变成字符串。
- 管道（`|`）由 `sh -c` 处理，而不是由 nabi 处理。需要 shell 语法时，这是最简单的方法。
- `env` 只为子进程添加变量，`env_clear` 清空继承的环境。父进程的环境不会改变。
- `spawn()` 不等待，直接返回 [`Child`](../../std/std/process.html#struct.Child)。`Child` 持有用 `Stdio::Piped` 打开的管道，直到你取走它们：`child.take_stdin()` 交出标准输入的管道并留下 `none`，所以管道只有一个所有者，也就是你。从未取走的管道在 `Child` 释放时关闭。向 `stdin` 写入，用 `close()` 发送 EOF，然后读取 `stdout`，同样关闭它，再 `wait()`。管道的关闭可能失败，所以编译器要求你关闭它，而不是替你关闭。
- `Command` 的副本相互独立：构建器方法返回修改后的副本，所以 `base.arg("hi")` 不会改变 `base`，一个基础命令可以启动多个变体。丢弃了结果的构建器调用会产生警告，因为它什么也没有改变。
- `signal()` 来自 `std::os::unix::process` 中的 `ExitStatusExt`。在 Linux 和 macOS 上它在预导入中，所以不需要 `use`；在 Windows 上它不存在（[3.10 `os::`](../spec/refs-standard-libraries/300-std/310-os.html)）。
- 找不到可执行文件时会抛出 `IoError`。`try ... catch e { ... }` 就地捕获它并换成默认值（[3.14 错误控制流](../spec/refs-lang/003-expressions/314-error-control-flow.html)，[错误处理](../learn/10-errors.html)）。
