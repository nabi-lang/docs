# 25. 垃圾回收器停顿

Nabi 用垃圾回收器释放内存，大多数程序从不需要考虑它。但有时间预算的程序，例如每秒绘制 60 帧的游戏、音频回调、要在几毫秒内响应的服务器，需要知道回收器何时让程序停下，以及如何让这些停顿远离不能等待的部分。本章介绍停顿是怎样产生的，如何观察它们，以及 `bare::gc` 中用来缩短停顿或避开停顿的控制手段。

## 停顿从何而来

回收器会让所有线程短暂停顿。回收分为两种：

- **minor** 回收只查看上次回收以来分配的对象。它的开销大约等于其中存活下来的部分，而不是堆的大小，所以在大多数新对象很快死亡的常见情况下，它很短。每分配完一次年轻代预算（32 到 512 MiB，取决于堆的大小）就运行一次。
- **full** 回收查看所有对象。在 32 MiB 及以上的堆上，它在程序运行的同时进行标记，只让程序短暂停下两次：一次在开始，一次在结束。之后的清扫也与程序并行运行，分配速度快于标记器标记速度的线程会帮助标记（mark assist），而不是让所有线程停下。

全局停顿（stop-the-world）期间的工作分摊到各个 CPU 上：回收器最多用七个辅助线程进行标记。

## 观察停顿

`NABIR_GC_LOG=1` 在 stderr 上为每次回收打印一行，`NABIR_GC_STATS=1` 在程序退出时打印摘要，包括每种回收的 p50、p90、p99 和最长停顿（[工具](16-tools.html#垃圾回收器停顿)）。程序可以用 `bare::gc::stats()` 读取同样的计数器：

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

`stats()` 会短暂地获取回收器的锁：每帧或每次导出指标时读一次即可，不要在热循环中调用。

## 在帧之间回收

`gc::collect_young()` 立即运行一次 minor 回收。两帧之间或两个请求之间是开销最低的时机：此时这一帧的垃圾已经死亡，存活下来的很少。`gc::collect_young_within(ms)` 只在最近几次 minor 回收表明它能在 `ms` 毫秒（这一帧剩余的时间）内完成时才运行，并返回是否运行了。full 回收正在标记时，两者都不做任何事：标记会自行完成，不会造成长停顿。

`gc::collect()` 则完全不同：它是一次全程让所有线程停下的 full 回收，还会移动对象以压缩堆。在一帧中调用它是最慢的选择。

## 停顿目标

`gc::set_pause_goal(ms)` 要求回收器让每次停顿保持在 `ms` 毫秒左右。它根据最近几次 minor 回收的开销确定年轻代的大小，使 minor 回收符合目标；每次 full 回收都并发标记；并且从不自行压缩堆：

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

这是目标，不是保证：存活堆快速增长时，停顿可能更长。`set_pause_goal(0.0)` 恢复默认的大小设定。

## 不能停顿的帧

在 `gc::no_collect(f)` 内，直到 `f` 返回之前，任何线程上的分配都不会启动回收。如果年轻代预算在其中用完，堆会改为增长，回收推迟到该区域之后：

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

让区域保持简短，例如一次绘制调用或一个音频缓冲区：堆会按区域内分配的全部内容增长（由 `GcStats::grown_bytes` 统计）。在区域内，堆上限依然有效。

## 一帧内的值

`Arena<T>` 把一帧的值存放在一个缓冲区中。`alloc` 就地追加一个值，因此回收器既不需要为每个值分配对象，也不需要追踪它们；`reset` 一次清空 arena，并保留缓冲区供下一帧使用：

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

`ArenaRef` 是普通数据，可以存放在任何地方。每次 `reset` 都开始一个新的代，在此之前取得的引用在 `get` 中会触发 panic（`try_get` 返回 `none`）：过期的引用永远读不到下一帧的值。活得比一帧更久的值应放在 `List` 或 `GcPtr` 中。

## 内存上限

`gc::set_heap_limit(bytes)` 让堆保持在某个大小附近：接近上限时回收更早进行，达到上限后进行 full 回收。仅存活数据就超过上限时，程序也不会失败；堆会一点一点地超出上限增长。

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

## 游戏循环

[游戏循环示例](../examples/game-loop.html)把这些组合在一起：停顿目标、放在帧 arena 中的粒子、在 `no_collect` 内绘制，以及在每帧剩余时间内进行的 minor 回收。

这一切背后的规则见标准库规范（[bare 1.6.10 至 1.6.15](../spec/refs-standard-libraries/100-bare/106-gc-pin.html)）和语言参考（[7.5](../spec/refs-lang/007-runtime/705-concurrent-collection.html)）。

## 延伸阅读

- 下一章：[26. 热重载](26-hot-reload.html)
