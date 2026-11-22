# 並行下載器

一組工作任務分攤模擬的資源（以 `task::sleep` 假裝延遲，而不是真的連網），結果經由通道傳回，最後印出排序好的報告。

## 程式碼

```nabi
// A "concurrent downloader": a pool of worker tasks fetches simulated resources.

enum FetchError {
    NotFound,
    Timeout,
}

enum Status {
    Done(usize),
    Failed(FetchError),
}

struct Outcome {
    url: str;
    status: Status;
}

const WORKERS: i32 = 3;
const DEADLINE_MS: i64 = 250;

// Pretend to download `url`: latency and size are derived from the name.
async fn fetch(url: str) throws (FetchError) -> usize {
    let latency = if "slow" in url { 5000 } else { (url.len() * 7 % 40) as i64 };
    await task::sleep(Duration::ms(latency));
    if url.ends_with(".missing") { throw FetchError::NotFound; }
    url.len() * 1024
}

// Fetch with a deadline; a fetch that takes too long is cancelled.
async fn fetch_with_deadline(url: str) -> Status {
    let t = task::spawn(async {
        Status::Done(try await fetch(url) catch e { return Status::Failed(e); })
    });
    try await task::timeout(Duration::ms(DEADLINE_MS), t) catch _ {
        Status::Failed(FetchError::Timeout)
    }
}

async fn main() {
    let urls = [
        "https://example.org/index.html", "https://example.org/logo.png",
        "https://example.org/old.missing", "https://example.org/style.css",
        "https://example.org/slow/video.mp4", "https://example.org/app.js",
        "https://example.org/data.json", "https://example.org/about.html",
    ];

    // A channel is multi-producer, multi-consumer: every worker holds a copy
    // of one job receiver, and each URL goes to whichever worker takes it first.
    let (job_tx, job_rx) = channel();
    let (done_tx, done_rx) = channel();
    for _ in 0..WORKERS {
        let jobs = job_rx;
        let done = done_tx.clone();
        task::spawn(async {
            loop {
                let url = await jobs.recv_async() else { break; };
                try! done.send(Outcome { url: url, status: await fetch_with_deadline(url) });
            }
            done.close();
        });
    }
    done_tx.close();

    // Queue every URL, then close the channel so idle workers stop.
    for url in urls { try! job_tx.send(url); }
    job_tx.close();

    // Collect until every worker has closed its `done` sender.
    let results: List<Outcome> = [];
    for await r in done_rx { results.push(r); }
    results.sort_by_key(|r| r.url);

    let ok = 0;
    let bytes = 0;
    for r in results {
        let name = r.url.strip_prefix("https://example.org/") ?? r.url;
        match r.status {
            Status::Done(n) => { ok += 1; bytes += n; println(f"{name:<16} {n:>7} B"); },
            Status::Failed(FetchError::NotFound) => println(f"{name:<16}   404 not found"),
            Status::Failed(FetchError::Timeout) => println(f"{name:<16}   timed out"),
        }
    }
    println(f"{ok}/{results.len()} downloaded, {bytes} bytes total");
}
```

```output
about.html         30720 B
app.js             26624 B
data.json          29696 B
index.html         30720 B
logo.png           28672 B
old.missing        404 not found
slow/video.mp4     timed out
style.css          29696 B
6/8 downloaded, 176128 bytes total
```

## 說明

- 通道可以有多個傳送端和多個接收端（MPMC）。`Receiver` 的複本共用同一個佇列，所以每個工作者都拿一份工作接收端的複本，而每個 URL 會交給最先取走它的閒置工作者（[2.17.4 多個生產者與消費者](../spec/refs-standard-libraries/200-core/217-channel.html#2174-多個生產者與消費者)、[`channel`](../../std/core/channel.html#fn.channel)）。
- 每個工作者透過 `clone()` 取得自己的傳送端，完成時將它 `close()`。`for await r in done_rx` 要等所有傳送端都關閉才會結束，所以 `main` 會提早關閉原本的 `done_tx`（[並行](../learn/12-concurrency.html)）。
- `await jobs.recv_async()` 暫停的是任務而不是執行緒，直到有值送達。`main` 關閉 `job_tx` 且佇列清空後，它會產出 `none`，`let-else` 中的 `break` 便離開工作迴圈。
- 任務沒有及時完成時，`task::timeout` 會擲回 `TimeoutError` 並取消該任務，所以緩慢的抓取會在下一個 `await` 停下（`task::race` 也以同樣方式取消落後的任務）（[`task::timeout`](../../std/core/task.html#fn.timeout)、[2.18.1 核心 API](../spec/refs-standard-libraries/200-core/218-task.html#2181-核心-api)）。
- 每次執行時結果送達的順序都可能不同，所以印出前先依 URL 排序，報告才會永遠相同。
