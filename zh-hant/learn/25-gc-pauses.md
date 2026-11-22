# 25. 垃圾回收器的暫停

Nabi 以垃圾回收器釋放記憶體，大多數程式從來不必想到它。但有時間預算的程式，例如每秒繪製 60 個影格的遊戲、音訊回呼、要在幾毫秒內回應的伺服器，會想知道回收器何時讓它停下，以及如何讓這些停頓避開不能等待的部分。本章說明暫停是怎麼產生的、如何觀察它們，以及 `bare::gc` 中讓暫停保持短暫或避開關鍵時段的控制項。

## 暫停從何而來

回收器會以短暫的暫停讓所有執行緒停下。回收有兩種：

- **minor** 回收只檢查上次回收之後配置的物件。它的成本大約取決於其中存活下來的部分，而不是堆積的大小，所以當大多數新物件都很早死亡（這是常見情況）時，它很短。每當年輕代預算（依堆積大小為 32 到 512 MiB）配置完畢，就會執行一次。
- **full** 回收檢查所有物件。在 32 MiB 以上的堆積上，它會在程式執行的同時進行標記，只短暫讓程式停下兩次：一次在開始，一次在結束。之後的清除也與程式同時進行，而配置速度比標記器標記還快的執行緒，會協助標記（mark assist），而不是讓所有執行緒停下。

需要讓整個程式停下（stop-the-world）的工作會分散到多顆 CPU 上：回收器最多會用七個輔助執行緒進行標記。

## 觀察暫停

`NABIR_GC_LOG=1` 會在每次回收時於 stderr 印出一行，`NABIR_GC_STATS=1` 則在程式結束時印出摘要，列出每種暫停的 p50、p90、p99 與最長值（[工具](16-tools.html#garbage-collector-pauses)）。程式可以用 `bare::gc::stats()` 讀取同樣的計數器：

```nabi
use bare::gc;

let before = gc::stats();
for i in 0..100000 { let _s = f"temporary {i}"; }
gc::collect_young();
let s = gc::stats();
println(f"minor collections ran: {s.minor > before.minor}");
println(f"within budget: {gc::collect_young_within(4.0)}");
```

```output
minor collections ran: true
within budget: true
```

`stats()` 會短暫取得回收器的鎖：每個影格或每次匯出指標時讀一次即可，不要在熱迴圈中讀取。

## 在影格之間回收

`gc::collect_young()` 會立即執行一次 minor 回收。兩個影格之間，或兩個請求之間，是代價最低的時機：那時該影格的垃圾都已經死亡，存活的很少。`gc::collect_young_within(ms)` 只在最近幾次 minor 回收顯示它能在 `ms` 毫秒（影格剩下的時間）內完成時才執行，並回傳是否有執行。在 full 回收正在標記期間，這兩者都不做任何事：標記會自行完成，不會造成長時間的暫停。

`gc::collect()` 則是另一回事：它是一次完整回收，在整個過程中讓所有執行緒停下，並移動物件以壓縮堆積。這是在影格中最慢的呼叫。

## 暫停目標

`gc::set_pause_goal(ms)` 要求回收器讓每次暫停維持在 `ms` 毫秒左右。它會依據最近幾次 minor 回收的成本來決定年輕代的大小，讓 minor 回收符合目標，並以並行方式標記每次 full 回收，而且絕不會自行壓縮堆積：

```nabi
use bare::gc;

gc::set_pause_goal(2.0);
let kept: List<str> = [];
for i in 0..200000 {
    let s = f"item {i}";
    if i % 100 == 0 { kept.push(s); }
}
println(f"{kept.len()} kept, young budget at most 128 MiB: {gc::stats().young_budget <= 128 << 20}");
gc::set_pause_goal(0.0);
```

```output
2000 kept, young budget at most 128 MiB: true
```

這是目標，不是保證：當存活的堆積快速成長時，暫停可能會更長。`set_pause_goal(0.0)` 會回到預設的大小設定。

## 不能暫停的影格

在 `gc::no_collect(f)` 內，直到 `f` 回傳為止，任何執行緒上的任何配置都不會啟動回收。當年輕代預算在其中用完時，堆積會改為成長，回收則在該區段之後進行：

```nabi
use bare::gc;

let before = gc::stats().collections;
let (n, during) = gc::no_collect(|| {
    let lines: List<str> = [];
    for i in 0..10000 { lines.push(f"line {i}"); }
    (lines.len(), gc::stats().collections - before)
});
println(f"{n} lines drawn, {during} collections meanwhile");
```

```output
10000 lines drawn, 0 collections meanwhile
```

請讓區段保持簡短，例如一次繪製呼叫或一個音訊緩衝區：堆積會因區段內配置的所有東西而成長（`GcStats::grown_bytes` 會計算這個量）。在區段內，堆積上限仍然有效。

## 單一影格的值

`Arena<T>` 把一個影格的值放在同一個緩衝區中。`alloc` 會就地附加一個值，所以回收器既不必為每個值配置物件，也不必追蹤它們；`reset` 會立即清空 arena，並保留它的緩衝區給下一個影格使用：

```nabi
use bare::gc::{Arena, ArenaRef};

struct Spark { x: f64; y: f64; }

let sparks: Arena<Spark> = Arena::with_capacity(1024);
let first: ArenaRef<Spark> = sparks.alloc(Spark { x: 1.0, y: 2.0 });
for i in 0..99 { _ = sparks.alloc(Spark { x: i as f64, y: 0.0 }); }
println(f"{sparks.len()} sparks, first at x {sparks.get(first).x}");
sparks.reset();
println(f"after reset: {sparks.len()} sparks, first still there: {sparks.holds(first)}");
println(f"try_get: {sparks.try_get(first) == none}");
```

```output
100 sparks, first at x 1
after reset: 0 sparks, first still there: false
try_get: true
```

`ArenaRef` 是單純的資料，所以可以保存在任何地方。每次 `reset` 都會開始一個新的世代，在那之前取得的參照在 `get` 中會觸發 panic（`try_get` 則回傳 `none`）：過時的參照永遠讀不到下一個影格的值。存活時間超過一個影格的值，應該放在 `List` 或 `GcPtr` 中。

## 記憶體上限

`gc::set_heap_limit(bytes)` 讓堆積維持在某個大小附近：越接近上限，回收就越早發生，達到上限後則進行 full 回收。當光是存活的資料就超過上限時，程式不會失敗；堆積會一點一點地超過上限成長。

```nabi
use bare::gc;

gc::set_heap_limit(64 << 20);
for round in 0..50 {
    let block: List<i64> = List::filled(262144, round);
    if block[0] != round { println("lost"); }
}
let s = gc::stats();
println(f"heap within the limit: {s.live_bytes <= 96 << 20}");
gc::set_heap_limit(0);
```

```output
heap within the limit: true
```

## 遊戲迴圈

[遊戲迴圈範例](../examples/game-loop.html)把這些組合在一起：暫停目標、放在影格 arena 中的粒子、在 `no_collect` 內進行的繪製，以及在每個影格剩餘時間內執行的 minor 回收。

這一切背後的規則記載於標準函式庫規格（[bare 1.6.10 到 1.6.15](../spec/refs-standard-libraries/100-bare/106-gc-pin.html)）與語言參考（[7.5](../spec/refs-lang/007-runtime/705-concurrent-collection.html)）中。
