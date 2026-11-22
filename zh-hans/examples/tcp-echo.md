# TCP 回显服务器

一个程序在空闲的 `127.0.0.1` 端口上启动回显服务器，让客户端发送若干行并读回回显，最后干净地关闭所有连接。

## 代码

```nabi
// A line-based TCP echo server and its clients in one program.
use std::net::{TcpListener, TcpStream, SocketAddr};
use core::io::{IoError, BufReader};
use std::thread;

// Echo every line of one connection back until the client closes its side.
fn serve(conn: TcpStream) throws (IoError) -> i32 {
    let lines = 0;
    let (rh, w) = conn.split();
    let r = BufReader::new(rh);
    try for line in r.lines() {
        try w.write_all(f"echo: {line}\n");
        lines += 1;
    }
    // Both halves closed: the socket closes and the client sees EOF.
    try r.into_inner().close();
    try w.close();
    lines
}

// Send each message, wait for its echo, then close the write side.
fn client(id: i32, addr: SocketAddr, msgs: List<str>) throws (IoError) {
    let (rh, w) = (try TcpStream::connect(addr)).split();
    using r = BufReader::new(rh) {
        for m in msgs {
            try w.write_all(f"{m}\n");
            let reply = StrBuf::new();
            try r.read_line(reply);
            println(f"client {id}: sent {m:?}, got {reply.to_str().trim_end():?}");
        }
        // Closing the write half ends the stream for the server.
        try w.close();
        // A read of 0 bytes means the server closed the connection.
        if (try r.read_line(StrBuf::new())) == 0 { println(f"client {id}: closed"); }
    }
}

fn main() throws (IoError) {
    // Port 0: the OS picks a free port; local_addr tells us which.
    let listener = try TcpListener::bind("127.0.0.1:0");
    let addr = try listener.local_addr();
    let clients = 3;

    // The server thread accepts `clients` connections, one handler thread each.
    let server = thread::spawn(|| {
        let handlers: List<thread::JoinHandle<i32>> = [];
        for _ in 0..clients {
            let (conn, _peer) = try! listener.accept();
            handlers.push(thread::spawn(|| try serve(conn) catch e { println(f"server: {e}"); 0 }));
        }
        let total = 0;
        for h in handlers { total += try! h.join(); }
        try! listener.close();
        total
    });

    try client(1, addr, ["hello", "nabi"]);
    try client(2, addr, ["one line"]);
    try client(3, addr, ["a", "b", "c"]);

    let served = try! server.join();
    println(f"server echoed {served} lines and stopped");
}
```

```output
client 1: sent "hello", got "echo: hello"
client 1: sent "nabi", got "echo: nabi"
client 1: closed
client 2: sent "one line", got "echo: one line"
client 2: closed
client 3: sent "a", got "echo: a"
client 3: sent "b", got "echo: b"
client 3: sent "c", got "echo: c"
client 3: closed
server echoed 6 lines and stopped
```

## 讲解

- 把 `TcpListener` 绑定到端口 0，由操作系统选择一个空闲端口，`local_addr()` 报告实际的地址，所以不会与其他程序冲突。`bind` 和 `connect` 接受文本、`SocketAddr` 或其他任何实现了 `ToSocketAddrs` 的东西（[3.2 net](../spec/refs-standard-libraries/300-std/302-net.html)，[`TcpListener`](../../std/std/net.html#struct.TcpListener)）。
- 服务器线程为每个连接启动一个处理线程，然后 `join` 所有线程，并累加处理过的行数。闭包捕获 `listener` 和 `conn`，并把它们带到其他线程（[并发](../learn/12-concurrency.html)，[`thread::spawn`](../../std/std/thread.html#fn.spawn)）。
- 为了在写入的同时逐行读取，套接字被 `split()` 成读半部（包装在 `BufReader` 中）和写半部：每一半拥有自己的方向（[拆分的两半](../spec/refs-standard-libraries/300-std/302-net.html#拆分为两半)，[`BufReader`](../../std/core/io.html#struct.BufReader)）。
- 每个套接字都由它的所有者关闭：服务器的处理程序关闭两半（`r.into_inner().close()`、`w.close()`），客户端关闭写半部并离开 `using` 块，监听器在所有客户端处理完后关闭。关闭可能失败，所以留下一个没有关闭就是编译错误。
- 关闭的顺序很重要。发送完所有内容后，客户端关闭写半部，表示“没有更多了”（即 `shutdown(Write)`，读半部保持打开），服务器的 `lines()` 收到这个信号后结束循环。服务器关闭两半之后，套接字关闭，客户端的 `read_line` 读到 0 字节。
- 同样的服务器也可以用 `accept_async` / `read_async` 和任务来写，而不使用阻塞 API（[3.2.0 通用规则](../spec/refs-standard-libraries/300-std/302-net.html#320-通用规则)）。
