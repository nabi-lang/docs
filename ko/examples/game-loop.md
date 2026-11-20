# 게임 루프

가비지 컬렉터를 프레임 밖에 두는 60 fps 루프입니다. 일시 정지 목표, 프레임 arena에 담은 particle, 수집하지 않는 구간 안의 그리기, 프레임마다 남은 시간에 하는 minor 수집을 씁니다.

## 코드

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

## 살펴보기

- `gc::set_pause_goal(2.0)`은 최근 minor 수집의 비용을 보고 minor 수집 한 번이 2 ms쯤 걸리도록 young 세대의 크기를 정합니다([가비지 컬렉터 일시 정지](../learn/25-gc-pauses.html#일시-정지-목표)).
- 한 프레임의 particle은 `Arena<Particle>`에 삽니다. `alloc`은 particle을 제자리에 덧붙이고, `parts.reset()`은 한 번에 모두 버립니다. 지난 프레임의 `ArenaRef`를 쓰면 panic하므로, 지난 참조가 이번 프레임의 particle을 읽는 일은 없습니다([`Arena`](../learn/25-gc-pauses.html#한-프레임의-값)).
- entity는 `for ref e in ents`로 제자리에서 바꿉니다. 각 원소를 복사본이 아니라 자리로 묶습니다([컬렉션과 반복자](../learn/08-collections-iterators.html)).
- 디버그 줄은 평범한 문자열입니다. 다음 프레임이면 쓰레기가 되고, minor 수집이 해제합니다.
- 그리기는 `gc::no_collect` 안에서 합니다. 그 안의 할당은 어느 스레드에서도 수집을 시작하지 않습니다. young 예산이 다 떨어지면 힙을 늘리고, 수집은 구간이 끝날 때까지 기다립니다.
- 프레임 끝에서 `gc::collect_young_within`은 최근 minor 수집으로 보아 프레임에 남은 시간(여유 1 ms를 뺀 것) 안에 끝날 때만 minor 수집을 합니다. 그다음 프레임은 다음 16.7 ms 눈금까지 잠듭니다.
- 프레임 시간을 보려면 가장 긴 `ms`를 기억해 출력하거나 `NABIR_GC_STATS=1`로 실행합니다. 요약에 일시 정지 종류마다 p99와 가장 긴 값이 나옵니다.
