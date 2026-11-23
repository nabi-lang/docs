# 25. 가비지 컬렉터 일시 정지

Nabi는 가비지 컬렉터로 메모리를 해제하고, 대부분의 프로그램은 이를 신경 쓸 일이 없습니다. 하지만 시간 예산이 있는 프로그램, 곧 초당 60 프레임을 그리는 게임, 오디오 콜백, 몇 밀리초 안에 답해야 하는 서버는 컬렉터가 언제 자신을 멈추는지, 그 멈춤을 기다리면 안 되는 부분 밖으로 어떻게 내보내는지 알고 싶어 합니다. 이 장은 일시 정지가 어디서 생기는지, 어떻게 지켜보는지, 그리고 일시 정지를 짧게 하거나 비켜 가게 하는 `bare::gc`의 도구를 보여 줍니다.

## 일시 정지가 생기는 곳

컬렉터는 짧은 일시 정지 동안 모든 스레드를 멈춥니다. 수집에는 두 가지가 있습니다.

- **minor** 수집은 지난 수집 뒤에 할당한 객체만 봅니다. 힙 크기가 아니라 그중 살아남은 것만큼 들므로, 새 객체 대부분이 일찍 죽는 흔한 경우에는 짧습니다. young 예산(힙에 따라 32~512 MiB)만큼 할당할 때마다 한 번 돕니다.
- **full** 수집은 모든 객체를 봅니다. 32 MiB 이상의 힙에서는 프로그램이 도는 동안 mark하고, 시작할 때와 끝낼 때 두 번만 잠깐 멈춥니다. 그 뒤의 sweep도 프로그램 옆에서 돌고, marker보다 빨리 할당하는 스레드는 모두를 멈추는 대신 mark를 돕습니다(mark assist).

world를 멈추는 동안의 일은 여러 CPU에 나뉩니다. 컬렉터는 도우미 스레드 최대 일곱 개와 함께 mark합니다.

## 일시 정지 지켜보기

`NABIR_GC_LOG=1`은 수집마다 stderr에 한 줄을, `NABIR_GC_STATS=1`은 프로그램이 끝날 때 종류별 p50, p90, p99와 가장 긴 일시 정지를 담은 요약을 씁니다([도구](16-tools.html#가비지-컬렉터-일시-정지)). 프로그램은 같은 카운터를 `bare::gc::stats()`로 읽습니다.

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

`stats()`는 컬렉터의 lock을 잠깐 잡습니다. 자주 도는 루프가 아니라 프레임마다, 지표를 내보낼 때마다 한 번 읽습니다.

## 프레임 사이에 수집하기

`gc::collect_young()`은 지금 minor 수집을 합니다. 두 프레임 사이, 두 요청 사이가 싼 때입니다. 그때는 프레임의 쓰레기가 이미 죽어 있어 살아남는 것이 적습니다. `gc::collect_young_within(ms)`는 최근 minor 수집으로 보아 `ms` 밀리초, 곧 프레임에 남은 시간 안에 끝날 때만 수집하고, 했는지를 돌려줍니다. full 수집이 mark하는 동안에는 둘 다 아무것도 하지 않습니다. mark는 긴 일시 정지 없이 스스로 끝납니다.

`gc::collect()`는 다른 것입니다. 모든 스레드를 처음부터 끝까지 멈추고 힙을 압축하려고 객체를 옮기는 full 수집이라, 프레임 안에서 부르기에 가장 느린 호출입니다.

## 일시 정지 목표

`gc::set_pause_goal(ms)`는 일시 정지를 `ms` 밀리초 가까이에 두라고 컬렉터에 요청합니다. 최근 minor 수집의 비용으로 young 세대 크기를 정해 minor 수집이 목표 안에 들게 하고, full 수집은 모두 동시에 mark하며, 힙을 스스로 압축하지 않습니다.

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

약속이 아니라 목표입니다. 살아 있는 힙이 빠르게 자라는 동안에는 일시 정지가 더 길 수 있습니다. `set_pause_goal(0.0)`은 기본 크기 정하기로 돌아갑니다.

## 멈추면 안 되는 프레임

`gc::no_collect(f)` 안에서는 `f`가 돌아올 때까지 어느 스레드의 할당도 수집을 시작하지 않습니다. 그 안에서 young 예산이 다 떨어지면 대신 힙을 늘리고, 수집은 구간 뒤에 합니다.

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

구간은 그리기 한 번, 오디오 버퍼 하나처럼 짧게 둡니다. 안에서 할당한 만큼 힙이 자랍니다(`GcStats::grown_bytes`가 셉니다). 힙 상한은 구간 안에서도 지킵니다.

## 한 프레임의 값

`Arena<T>`는 한 프레임의 값을 버퍼 하나에 담습니다. `alloc`은 값을 제자리에 덧붙이므로 컬렉터는 값마다 객체를 할당하거나 추적하지 않고, `reset`은 버퍼를 다음 프레임을 위해 남긴 채 arena를 한 번에 비웁니다.

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

`ArenaRef`는 평범한 데이터라 어디에든 둘 수 있습니다. `reset`마다 새 세대가 시작되고, 그 전의 참조는 `get`에서 panic합니다(`try_get`은 `none`). 지난 참조가 다음 프레임의 값을 읽는 일은 없습니다. 한 프레임보다 오래 사는 값은 `List`나 `GcPtr`에 둡니다.

## 메모리 상한

`gc::set_heap_limit(bytes)`는 힙을 그 크기 근처에 둡니다. 상한에 가까워질수록 수집이 일찍 오고, 상한에 이르면 full 수집을 합니다. 살아 있는 데이터만으로 상한보다 크면 프로그램이 실패하지 않고, 힙이 상한을 조금씩 넘어 자랍니다.

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

## 게임 루프

[게임 루프 예제](../examples/game-loop.html)가 이것들을 모읍니다. 일시 정지 목표, 프레임 arena의 particle, `no_collect` 안의 그리기, 프레임마다 남은 시간에 하는 minor 수집입니다.

이 모든 규칙은 표준 라이브러리 명세([bare 1.6.10~1.6.15](../spec/refs-standard-libraries/100-bare/106-gc-pin.html))와 언어 레퍼런스([7.5](../spec/refs-lang/007-runtime/705-concurrent-collection.html))에 있습니다.

## 더 읽을거리

- 다음 장: [26. 핫 리로드](26-hot-reload.html)
