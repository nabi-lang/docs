# TCP echo 伺服器

一個程式在空閒的 `127.0.0.1` 連接埠上啟動 echo 伺服器，讓用戶端送出幾行文字並讀回回音，最後乾淨地關閉一切。

## 程式碼

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

## 說明

- 把 `TcpListener` 繫結到連接埠 0，讓作業系統挑選空閒的連接埠，`local_addr()` 回報實際的位址，所以不會和其他程式衝突。`bind` 和 `connect` 接受文字、`SocketAddr`，或其他任何實作 `ToSocketAddrs` 的值（[3.2 net](../spec/refs-standard-libraries/300-std/302-net.html)、[`TcpListener`](../../std/std/net.html#struct.TcpListener)）。
- 伺服器執行緒為每個連線啟動一個處理執行緒，然後 `join` 全部的執行緒，並加總處理過的行數。閉包擷取 `listener` 和 `conn`，把它們帶到其他執行緒（[並行](../learn/12-concurrency.html)、[`thread::spawn`](../../std/std/thread.html#fn.spawn)）。
- 為了在寫入的同時逐行讀取，socket 經由 `split()` 分成包在 `BufReader` 中的讀取半部，以及寫入半部：每一半各自擁有自己的方向（[拆成兩半](../spec/refs-standard-libraries/300-std/302-net.html#拆成兩半)、[`BufReader`](../../std/core/io.html#struct.BufReader)）。
- 每個 socket 都由它的擁有者關閉：伺服器的處理執行緒關閉兩個半部（`r.into_inner().close()`、`w.close()`），用戶端關閉寫入半部並離開 `using` 區塊，listener 則在服務完所有用戶端後關閉。留下任何一個沒關會是編譯錯誤，因為關閉可能失敗。
- 關閉的順序很重要。用戶端送完所有資料後，關閉寫入半部表示「沒有更多了」（即 `shutdown(Write)`，讀取半部仍保持開啟），伺服器的 `lines()` 收到這個訊號就結束迴圈。伺服器關閉兩個半部後，socket 隨之關閉，用戶端的 `read_line` 讀到 0 位元組。
- 同一個伺服器也可以用 `accept_async` / `read_async` 和任務來寫，而不使用阻塞式 API（[3.2.0 共通規則](../spec/refs-standard-libraries/300-std/302-net.html#320-共通規則)）。
