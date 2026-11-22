# 遊戲迴圈

一個 60 fps 的迴圈，讓垃圾回收器不打擾每個影格：設定暫停目標、把粒子放進影格 arena、在不會回收的區域中繪製，並利用每個影格剩下的時間做一次 minor 回收。

## 程式碼

```nabi
use bare::gc;
use bare::gc::{Arena, ArenaRef};
use core::time::{Duration, Instant};
use std::thread;

const FRAMES: i64 = 60;
const FRAME_MS: f64 = 16.7;

struct Particle { x: f64; y: f64; vx: f64; vy: f64; }

struct Entity { name: str; x: f64; y: f64; }

// One frame's work: spawn particles into the frame's arena, move them,
// and build the debug text, which is garbage by the next frame.
fn simulate(frame: i64, ents: List<Entity>, parts: Arena<Particle>) -> List<str> {
    let spawned: List<ArenaRef<Particle>> = [];
    for i in 0..200 {
        let a = (frame * 200 + i) as f64;
        spawned.push(parts.alloc(Particle { x: 0.0, y: 0.0, vx: a % 7.0 - 3.0, vy: a % 5.0 - 2.0 }));
    }
    for r in spawned {
        let p = parts.get(r);
        parts.set(r, Particle { x: p.x + p.vx, y: p.y + p.vy, vx: p.vx, vy: p.vy - 0.1 });
    }
    for ref e in ents {
        e.x += 1.0;
    }
    let lines: List<str> = [];
    for i in 0..20 {
        lines.push(f"frame {frame}: {ents[i].name} at ({ents[i].x}, {ents[i].y})");
    }
    lines
}

fn main() {
    // Keep each pause near 2 ms.
    gc::set_pause_goal(2.0);
    let ents: List<Entity> = [];
    for i in 0..5000 {
        ents.push(Entity { name: f"ent{i}", x: 0.0, y: i as f64 });
    }
    let parts: Arena<Particle> = Arena::with_capacity(256);
    let before = gc::stats();
    let drawn: i64 = 0;
    for frame in 0..FRAMES {
        let start = Instant::now();
        let lines = simulate(frame, ents, parts);
        // Drawing must not pause: no collection starts in here.
        drawn += gc::no_collect(|| {
            let n: i64 = 0;
            for l in lines { n += l.len() as i64; }
            n + parts.len() as i64
        });
        parts.reset();
        // Spend some of what is left of the frame on a minor collection.
        let used = start.elapsed().as_secs_f64() * 1000.0;
        if used < FRAME_MS - 1.0 {
            _ = gc::collect_young_within(FRAME_MS - 1.0 - used);
        }
        let ms = start.elapsed().as_secs_f64() * 1000.0;
        if ms < FRAME_MS {
            thread::sleep(Duration::us(((FRAME_MS - ms) * 1000.0) as i64));
        }
    }
    let s = gc::stats();
    println(f"{FRAMES} frames, {drawn} bytes and particles drawn");
    println(f"entities: {ents.len()}, first at x {ents[0].x}, arena empty: {parts.is_empty()}");
    println(f"collected between frames: {s.minor > before.minor}");
}
```

```output
60 frames, 42820 bytes and particles drawn
entities: 5000, first at x 60, arena empty: true
collected between frames: true
```

## 說明

- `gc::set_pause_goal(2.0)` 依據最近幾次 minor 回收的成本調整年輕世代的大小，讓一次 minor 回收大約花 2 ms（[垃圾回收器暫停](../learn/25-gc-pauses.html#暫停目標)）。
- 一個影格的粒子存放在 `Arena<Particle>` 中：`alloc` 就地附加每個粒子，`parts.reset()` 一次全部丟棄。使用先前影格的 `ArenaRef` 會觸發 panic，所以過期的參考不會讀到這個影格的粒子（[`Arena`](../learn/25-gc-pauses.html#一個影格的值)）。
- 實體透過 `for ref e in ents` 就地修改，它把每個元素繫結為位置而不是複本（[集合型別與迭代器](../learn/08-collections-iterators.html)）。
- 偵錯用的文字行是一般字串：到下一個影格就成了垃圾，由 minor 回收釋放。
- 繪製在 `gc::no_collect` 中執行：在這裡，任何執行緒上的配置都不會啟動回收；如果年輕世代的額度用完，堆積會成長，回收則等到區域結束後才進行。
- 在影格結尾，`gc::collect_young_within` 只在最近幾次回收顯示它放得進影格剩餘時間（扣掉一毫秒的餘裕）時，才執行 minor 回收。接著影格會休眠到下一個 16.7 ms 的時間點。
- 想看影格時間，可以記下最長的 `ms` 並印出來，或者以 `NABIR_GC_STATS=1` 執行：摘要會列出每一種暫停的 p99 與最長值。
