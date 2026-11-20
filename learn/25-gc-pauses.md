# 25. Garbage collector pauses

Nabi frees memory with a garbage collector, and most programs never think about it. A program with a time budget, a game drawing 60 frames a second, an audio callback, a server answering within a few milliseconds, wants to know when the collector stops it and how to keep those stops out of the part that must not wait. This chapter shows how the pauses come about, how to watch them, and the controls in `bare::gc` that keep them short or out of the way.

## Where pauses come from

The collector stops every thread for short pauses. There are two kinds of collection:

- A **minor** collection looks only at the objects allocated since the last collection. It costs about what survived of them, not the size of the heap, so it is short when most new objects die young, the common case. One runs each time the young budget (32 to 512 MiB, depending on the heap) has been allocated.
- A **full** collection looks at every object. On a heap of 32 MiB or more it marks while the program runs and stops it only twice, briefly: once to start and once to finish. The sweep afterwards runs beside the program too, and a thread that allocates faster than the marker marks helps it (a mark assist) instead of stopping everyone.

Stop-the-world work is spread over the CPUs: the collector marks with up to seven helper threads.

## Watching pauses

`NABIR_GC_LOG=1` prints one line per collection on stderr, and `NABIR_GC_STATS=1` a summary when the program exits, with p50, p90, p99 and the longest pause of each kind ([Tools](16-tools.html#garbage-collector-pauses)). A program reads the same counters with `bare::gc::stats()`:

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

`stats()` takes the collector's lock for a moment: read it once per frame or per metrics export, not in a hot loop.

## Collecting between frames

`gc::collect_young()` runs a minor collection now. Between two frames, or two requests, is the cheap moment: the frame's garbage is dead by then, so little survives. `gc::collect_young_within(ms)` does it only when the last minor collections say it fits in `ms` milliseconds, the time left of the frame, and says whether it ran. While a full collection is marking, both do nothing: the mark finishes on its own without a long pause.

`gc::collect()` is a different thing: a full collection that stops every thread for all of it, and moves objects to compact the heap. It is the slowest call to make in a frame.

## A pause goal

`gc::set_pause_goal(ms)` asks the collector to keep each pause near `ms` milliseconds. It sizes the young generation from what recent minor collections cost, so a minor collection fits the goal, marks every full collection concurrently, and never compacts the heap on its own:

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

It is a goal, not a promise: while the live heap grows fast, a pause can be longer. `set_pause_goal(0.0)` goes back to the default sizing.

## Frames that must not pause

Inside `gc::no_collect(f)` no allocation starts a collection, on any thread, until `f` returns. When the young budget runs out inside, the heap grows instead, and the collection comes after the region:

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

Keep regions short, a draw call or an audio buffer: the heap grows by everything allocated inside (`GcStats::grown_bytes` counts it). A heap limit still holds inside a region.

## Values for one frame

An `Arena<T>` holds the values of one frame in one buffer. `alloc` appends a value in place, so the collector neither allocates nor traces an object per value, and `reset` empties the arena at once, keeping its buffer for the next frame:

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

An `ArenaRef` is plain data, so it can be kept anywhere. Every `reset` starts a new generation, and a reference from before it panics in `get` (`try_get` gives `none`): a stale reference can never read the next frame's value. Values that outlive a frame belong in a `List` or a `GcPtr`.

## A memory ceiling

`gc::set_heap_limit(bytes)` keeps the heap near a size: collections come sooner as it nears the limit, and full ones once it is reached. When the live data alone is larger, the program does not fail; the heap grows past the limit a little at a time.

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

## A game loop

The [game loop example](../examples/game-loop.html) puts these together: a pause goal, particles in a frame arena, drawing inside `no_collect`, and a minor collection in the time left of each frame.

The rules behind all of this are in the standard library specification ([bare 1.6.10 to 1.6.15](../spec/refs-standard-libraries/100-bare/106-gc-pin.html)) and the language reference ([7.5](../spec/refs-lang/007-runtime/705-concurrent-collection.html)).
