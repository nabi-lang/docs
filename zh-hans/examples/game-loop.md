# 游戏循环

一个 60 fps 的循环，让垃圾回收器不打扰它的帧：设定停顿目标，把粒子放在帧区域分配器中，在从不回收的区域中绘制，并利用每帧剩余的时间进行一次 minor 回收。

## 代码

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

## 讲解

- `gc::set_pause_goal(2.0)` 根据最近几次 minor 回收的开销来设定年轻代的大小，使一次 minor 回收大约耗时 2 ms（[垃圾回收器的停顿](../learn/25-gc-pauses.html#停顿目标)）。
- 一帧的粒子存放在 `Arena<Particle>` 中：`alloc` 把每个粒子就地追加进去，`parts.reset()` 一次性把它们全部丢弃。使用之前帧的 `ArenaRef` 会触发 panic，所以过时的引用不会读到本帧的粒子（[`Arena`](../learn/25-gc-pauses.html#一帧内的值)）。
- 实体通过 `for ref e in ents` 就地修改，它把每个元素绑定为位置，而不是副本（[集合与迭代器](../learn/08-collections-iterators.html)）。
- 调试文本是普通的字符串：到下一帧就成了垃圾，由 minor 回收释放。
- 绘制在 `gc::no_collect` 中运行：在其中的分配不会在任何线程上启动回收；如果年轻代的预算用完了，堆会增长，回收会等到这个区域结束后再进行。
- 在帧的末尾，只有当最近几次回收表明一次回收能放进本帧剩余的时间（减去 1 毫秒的余量）时，`gc::collect_young_within` 才会运行 minor 回收。然后这一帧休眠到下一个 16.7 ms 的时刻。
- 要查看帧时间，记录最长的 `ms` 并输出，或者用 `NABIR_GC_STATS=1` 运行：汇总会给出每种停顿的 p99 和最长值。
