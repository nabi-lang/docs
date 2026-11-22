# 并发下载器

一组工作任务分担模拟的资源（用 `task::sleep` 代替网络来模拟延迟），结果通过通道返回，最后输出一份排好序的报告。

## 代码

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

## 讲解

- 通道可以有多个发送者和多个接收者（MPMC）。`Receiver` 的副本共享同一个队列，所以每个工作任务都持有那一个任务接收者的副本，每个 URL 交给最先取走它的空闲工作任务（[2.17.4 多生产者与多消费者](../spec/refs-standard-libraries/200-core/217-channel.html#2174-multiple-producers-and-consumers)，[`channel`](../../std/core/channel.html#fn.channel)）。
- 每个工作任务通过 `clone()` 得到自己的发送者，完成后 `close()` 它。`for await r in done_rx` 只有在所有发送者都关闭后才结束，所以 `main` 提前关闭了它原来的 `done_tx`（[并发](../learn/12-concurrency.html)）。
- `await jobs.recv_async()` 挂起的是任务而不是线程，直到有值到达。`main` 关闭 `job_tx` 并且队列为空后，它产出 `none`，`let-else` 中的 `break` 就会离开工作循环。
- 如果任务没有按时完成，`task::timeout` 会抛出 `TimeoutError` 并取消该任务，所以慢的下载会在下一个 `await` 处停止（`task::race` 也以同样的方式取消落败的任务；[`task::timeout`](../../std/core/task.html#fn.timeout)，[2.18.1 core API](../spec/refs-standard-libraries/200-core/218-task.html#2181-core-api)）。
- 每次运行时结果到达的顺序都可能不同，所以输出前按 URL 排序，报告总是相同的。
