# Game loop

A 60 fps loop that keeps the garbage collector out of its frames: a pause goal, particles in a frame arena, drawing in a region that never collects, and a minor collection in the time left of each frame.

## Code

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

## Walkthrough

- `gc::set_pause_goal(2.0)` sizes the young generation so a minor collection takes about 2 ms, from what the last ones cost ([Garbage collector pauses](../learn/25-gc-pauses.html#a-pause-goal)).
- The particles of a frame live in an `Arena<Particle>`: `alloc` appends each one in place and `parts.reset()` drops them all at once. An `ArenaRef` from an earlier frame panics when used, so a stale reference cannot read this frame's particle ([`Arena`](../learn/25-gc-pauses.html#values-for-one-frame)).
- The entities change in place with `for ref e in ents`, which binds each element as a place instead of a copy ([Collections and iterators](../learn/08-collections-iterators.html)).
- The debug lines are ordinary strings: garbage by the next frame, freed by the minor collection.
- Drawing runs inside `gc::no_collect`: no allocation there starts a collection, on any thread; if the young budget runs out, the heap grows and the collection waits for the end of the region.
- At the end of the frame `gc::collect_young_within` runs a minor collection only if the last ones say it fits in what is left of the frame, minus a millisecond of margin. The frame then sleeps to the next 16.7 ms tick.
- To see the frame times, keep the longest `ms` and print it, or run with `NABIR_GC_STATS=1`: the summary gives each kind of pause's p99 and longest.
