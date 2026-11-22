# 12. 并发

同时做几件事的正确方式取决于目标。线程和 `par` 适合用到多个 CPU 的计算，`async` 任务适合等待很多的 I/O。本章通过简短的示例逐一介绍这些工具，也介绍用于安全交换数据的通道、`Mutex` 和 `Atomic`。

> [!TIP]
> 并发运行的代码，输出顺序每次运行都可能不同。本章的示例用 `join` 收集结果，或者先排序再打印，所以输出总是一样的。

哪些需要 `use`？本章用到的 `core` 中的内容都在 prelude（[第 11 章](11-modules-packages.html)）里：`task`、`par`、`channel`、`Mutex`、`Atomic`、`Duration`。只有与操作系统打交道的 `std` 模块需要手动导入，所以使用线程要先写 `use std::thread;`，就像使用文件要先写 `use std::fs;`。

## 线程

`std::thread::spawn` 在一个新的操作系统线程上运行闭包，并返回 `JoinHandle`。`join()` 等待该线程结束，取出闭包的返回值。如果线程以 panic 结束，它会抛出 `JoinError`，所以示例加了 `try`。

```nabi
use std::thread;

let handles = [];
for i in 1..=4 {
    handles.push(thread::spawn(|| i * i));
}

let results = [];
for h in handles {
    results.push(try h.join());
}
println(f"{results:?}");

let named = try thread::Builder::new().name("worker").spawn(|| {
    thread::current().name() ?? "?"
});
println(try named.join());
```

```output
[1, 4, 9, 16]
worker
```

示例在创建完所有句柄之后才 join，所以四个线程并发运行。结果按句柄顺序收集，因此输出顺序是固定的。

## 通道

在线程之间传递值，最简单的方式是通道。`channel()` 返回发送端 `Sender` 和接收端 `Receiver`。`Sender` 可以用 `clone()` 复制，所有 `Sender` 都 `close()` 之后，接收端的 `for` 循环就会结束。

```nabi
use std::thread;

let (tx, rx) = channel();
let workers: List<thread::JoinHandle<()>> = [];
for id in 0..3 {
    let tx = tx.clone();
    workers.push(thread::spawn(|| {
        for j in 0..2 {
            try! tx.send(f"w{id}-{j}");
        }
        tx.close();
    }));
}
tx.close();

let got = [];
for msg in rx {
    got.push(msg);
}
for w in workers { try w.join(); }

got.sort();
println(f"{got:?}");
```

```output
["w0-0", "w0-1", "w1-0", "w1-1", "w2-0", "w2-1"]
```

`Sender` 只有一个所有者：`let tx2 = tx` 会移动它（之后再用 `tx` 是错误），`clone()` 则创建另一个拥有自己所有者的发送端。编译器在发送端的最后一个所有者离开作用域时释放它，任务结束时也会释放其 `async` 块捕获的发送端；所有发送端都关闭或释放后，接收端的循环就会结束。如果接收端需要更早看到结束，就自己调用 `close()`：这里 `main` 在创建 `tx` 的同一个作用域中读取通道，而线程不会释放其闭包捕获的值，所以每个工作线程在完成时关闭自己的发送端。要限制通道的容量，使用 `channel_bounded(n)`。

## `Mutex` 与 `Atomic`

多个线程修改同一个值时，需要同步。单个整数计数器用 `Atomic`；其他值用 `Mutex` 包起来。`Mutex::lock()` 返回的守卫必须用 `using` 绑定，这样块结束时锁就会释放（规则与[第 10 章](10-errors.html)中的 `using` 相同）。只做一个简短的步骤时，`m.with(|v| ..)` 获取锁，用该值调用闭包，并在闭包返回时释放锁，下面的示例就是这样做的；`RwLock` 有 `read_with` 和 `write_with`。

```nabi
use std::thread;

let counter = Atomic::new(0);
let log = Mutex::new([]);

let hs: List<thread::JoinHandle<()>> = [];
for i in 1..=5 {
    hs.push(thread::spawn(|| {
        counter.fetch_add(i);
        log.with(|v| v.push(i * 10));
    }));
}
for h in hs { try h.join(); }

let items = log.with(|v| v.clone());   // a copy: the list itself stays behind the lock
items.sort();
println(f"sum = {counter.load()}, items = {items:?}");
```

```output
sum = 15, items = [10, 20, 30, 40, 50]
```

列表是共享的，所以 `v.push(..)` 修改的是锁后面的那个列表。结构体是值：闭包和守卫的 `get()` 交出的是一份副本，在副本上设置的字段会丢失，这是错误（[E0057](../errors/E0057.html)、[E0140](../errors/E0140.html)）。要通过守卫用 `update` 或 `set` 保存修改：`using g = m.lock() { g.update(|v| { v.total += 1; v }); }`。

只有 `using` 会释放守卫，所以编译器拒绝没有任何东西释放的守卫：用普通的 `let`（或 `if let`）绑定它，或者直接丢弃 `lock()` 的结果。`RwLock` 的 `read()` / `write()` 守卫也是如此。

```nabi norun
let g = m.lock();
```

```text check=checks/12-guard-let.nabi
error[E0065]: `g` holds a `MutexGuard` that is never released: only `using` releases a lock guard, so bind it with `using g = .. { .. }`
```

### 只读数据：`freeze`

创建一次之后只读的集合，例如查找表或名字列表，不需要锁。`freeze()` 把 `List`、`Dict` 或 `Set` 变成 `FrozenList`、`FrozenDict` 或 `FrozenSet`，它们不能被任何东西修改，所以所有线程可以同时读同一个值，无需加锁，也无需复制。线程的闭包、`par` 闭包或 `static` 都可以持有它。

```nabi
use std::thread;

let words = ["apple", "bee", "cat", "dog"].freeze();
let lens = {"apple": 5, "bee": 3}.freeze();

let hs: List<thread::JoinHandle<i32>> = [];
for i in 0..3 {
    hs.push(thread::spawn(|| words[i].len() as i32 + lens.get_or(words[i], 0)));
}
let total = 0;
for h in hs { total += try h.join(); }
println(f"{total} {words.len()}");
```

```output
19 4
```

`freeze` 和 `Mutex::new` 一样会交出集合：冻结后的值接管了元素，所以之后再用这个列表是错误（[E0105](../errors/E0105.html)）。请改为读取冻结后的值，或者如果仍需要一个可修改的列表，就冻结它的 `clone()`。冻结的集合没有 `push`、`insert` 或 `xs[i] = v`；`to_list()`（`to_dict()`、`to_set()`）返回一份可以修改的副本。

只有元素本身不能改变的集合才能冻结：数字、`str`、由它们组成的结构体，以及其他冻结的集合。`List<List<i32>>` 会让每个线程拿到同一批内部列表，而它们可以被修改，所以编译器会要求你先冻结内部列表（`List<FrozenList<i32>>`），或者使用 `Mutex`。

读得多、偶尔修改的数据，例如服务器的路由表，可以做成锁后面的一个冻结快照。读者取出当前快照，读取时不持有锁；写者制作一份修改后的副本，冻结它并放回原处。冻结的值可以离开锁，因为没有任何东西能修改它。

```nabi
static ROUTES: Mutex<FrozenDict<str, i32>> = Mutex::new(FrozenDict::new());

fn lookup(path: str) -> i32 | none {
    let routes = ROUTES.with(|r| r);     // the current snapshot
    routes.get(path)
}

fn add_route(path: str, port: i32) {
    using g = ROUTES.lock() {
        let next = g.get().to_dict();
        next[path] = port;
        g.set(next.freeze());
    }
}

add_route("/a", 8080);
add_route("/b", 9090);
println(f"{lookup("/a") ?? 0} {lookup("/c") ?? 0}");
```

```output
8080 0
```

初读时可以跳过本小节的其余部分。对于由几部分组成的值，例如带几个小列表的设置，还有 `Frozen<T>`。`Frozen::new` 保留自己的一份副本，`get()` 给每个读者一份各自的副本，所以读者修改自己的副本不会影响其他人看到的内容。

```nabi
use std::thread;

struct Settings {
    name: str;
    limits: List<i32>;
}

static SETTINGS: sync::Frozen<Settings> = sync::Frozen::new(Settings { name: "demo", limits: [10, 20, 30] });

let hs: List<thread::JoinHandle<i32>> = [];
for i in 0..3 {
    hs.push(thread::spawn(|| {
        let s = SETTINGS.get();      // this thread's own copy
        s.limits.push(i);            // changes the copy only
        s.limits[0] + s.limits.len() as i32
    }));
}
let total = 0;
for h in hs { total += try h.join(); }
let s = SETTINGS.get();
println(f"{s.name} {s.limits:?} {total}");
```

```output
demo [10, 20, 30] 42
```

每次 `get` 都会复制值中的集合，所以要把大的只读部分排除在副本之外：冻结它们，例如 `table: FrozenList<Entry>`。冻结的集合和其他 `Sync` 部分（例如 `Mutex`）是共享的，不会被复制。

### 每个线程各自的值：`thread::local`

初读时可以跳过本小节。

`thread::local(init)` 给每个线程一个各自的值，在该线程第一次使用时由 `init` 创建。`with` 把本线程的值借给一个闭包；闭包的返回值会被带出来，但值本身仍是借出的：返回它、把它存到闭包外面或者发送它都是错误，闭包也不能是 `async`。`set`、`replace` 和 `take` 整体移入或移出值，`get` 复制一个 `Copy` 值（例如数字）。

```nabi
use std::thread;

static NOTES: thread::ThreadLocal<List<i32>> = thread::local(|| -> List<i32> { [] });

fn note(x: i32) { NOTES.with(|l| l.push(x)); }

note(1);
note(2);
let h = thread::spawn(|| { note(9); NOTES.with(|l| l.len()) });
println(f"worker {try h.join()}, main {NOTES.with(|l| l.clone()):?}");
let mine = NOTES.take();
println(f"took {mine:?}, left {NOTES.with(|l| l.len())}");
```

```output
worker 1, main [1, 2]
took [1, 2], left 0
```

## 把值交给其他线程

本节讲的是值在线程之间移动时编译器强制执行的规则。初读时可以略读，等编译器把你指到这里时再回来看。

`Mutex::new(xs)`、`RwLock::new(xs)` 和 `tx.send(xs)` 会*交出 `xs`*：从此它属于锁或接收端。在 Nabi 中只有 `Sender` 或 `File` 这样的 `Drop` 资源会移动（[第 10 章](10-errors.html)）；对其他值，编译器改为跟踪交接：之后再用 `xs` 是编译错误，因为这会与它现在的所有者产生竞争。这项检查会沿着副本（`let ys = xs`）、结构体字段、持有它的对象、集合的元素、闭包捕获和函数调用一路跟踪这个值。

```nabi norun
let xs = [1, 2, 3];
let ys = xs;                  // the same list
let m = Mutex::new(xs);
ys.push(4);
```

```text check=checks/12-handoff-copy.nabi
error[E0105]: `ys` shares `xs`, which was moved into the `Mutex` at line 3
  --> main.nabi:4:1
   |
 3 | let m = Mutex::new(xs);
   |         -------------- `xs` moved into the `Mutex` here
 4 | ys.push(4);
   | ^^
   |
   = help: it belongs to the `Mutex` now; use it through the lock: `using g = m.lock() { g.get().push(4) }`, or hand over `xs.clone()` to keep a separate copy
```

请通过锁使用这个值，或者如果仍需要自己的一份，就交出它的 `clone()`。

元素也算在内。从集合中读出的东西和集合本身是同一个值，所以发送 `rows` 之后再向 `first = rows[0]` 推入元素，是同样的竞争：

```text check=checks/12-handoff-element.nabi
error[E0105]: `first` shares `rows`, which was sent through a channel at line 4
  --> main.nabi:5:1
   |
 4 | try! tx.send(rows);
   |      ------------- `rows` sent here
 5 | first.push(9);
   | ^^^^^
   |
   = help: the receiver owns it now; hand over `rows.clone()` to keep a separate copy, or give `first` a new value before using it again
```

要逐个交出元素，先把每个元素取出来。`pop` 和 `remove` 把元素从集合中取出，所以集合仍然可用。这就是常见的工作队列：

```nabi
use std::thread;

let jobs = [[3, 1, 2], [9, 7], [5, 4, 6]];
let (tx, rx) = channel();
let worker = thread::spawn(|| {
    let total = 0;
    for job in rx {
        job.sort();
        total += job[0];
    }
    total
});
while let job = jobs.pop() {
    try! tx.send(job);         // `pop` took it out of `jobs`, so `jobs` stays usable
}
tx.close();
println(f"smallest items add up to {try worker.join()}, jobs left: {jobs.len()}");
```

```output
smallest items add up to 12, jobs left: 0
```

### 把值移入线程：`[move x]`

传给 `thread::spawn` 或 `task::spawn_blocking` 的闭包与创建它的代码共享捕获，所以这些捕获必须是 `Sync`：列表不是。只有新线程需要这个列表时，用 `[move rows]` 把它移入闭包。闭包得到自己的一份副本，启动线程的代码就不能再用 `rows` 了：

```nabi
use std::thread;

let rows = [3, 1, 2];
let h = thread::spawn([move rows] || {
    rows.sort();
    rows[0]
});
println(f"smallest: {try! h.join()}");
```

```output
smallest: 1
```

在 spawn 之后使用 `rows` 是错误，和发送之后一样；不加 `[move x]` 就捕获列表时，帮助信息也会提到它。对于 `task::spawn_blocking`，也可以把值作为实参传入：`task::spawn_blocking_with_arg(rows, |rows: List<i32>| rows.len())`。

`fn` 值也以同样的方式捕获。参数 `f: fn() + Send` 是 `Send` 但不是 `Sync`，所以调用它的 spawn 闭包必须把它移入，即 `[move f]`。要在多个 spawn 闭包中调用它，改为接受 `fn() + Send + Sync`。

拥有被移入值的闭包只能交出一次，不论经由什么途径：另一个变量、字段、列表、`if` 分支，或者返回它的函数。把同一个闭包两次传给同一个调用，`par::join(f, f)`，也是第二次交出，同样是错误（[E0133](../errors/E0133.html)）；请创建两个闭包，各自移入自己的 `.clone()`。`async [move ..]` 闭包和它的调用产生的 future 共用一个所有者：你可以在这里依次 await 它们，但把其中一个交给 `task::spawn` 会把其他的一起带走。

锁守卫的值也是一样。`g.get()` 返回的是 `Mutex` 所保护对象的一个视图，所以它不能离开 `using` 块：块结束后，锁就不再保护它。返回它、把它放进错误中抛出或者发送它都是错误；把它作为 `break` 值、`if` 或 `match` 分支、`catch` 或闭包的结果带出块，同样是错误。

```text check=checks/12-lock-value.nabi
error[E0106]: the value of `g`'s `using` block is what the `Mutex` protects, and the block releases the lock before it is used
  --> main.nabi:2:10
   |
 2 | let xs = using g = m.lock() { g.get() };
   |          ^^^^^
   |                               ------- taken out of the lock here
   |
   = help: use it inside the `using` block, or make the block's value `g.get_clone()`
```

只要函数仅仅使用它，把它传给函数是可以的。会保留它的函数（存进 `Cell`、字段、闭包、线程局部变量）会让它比锁活得更久，所以那样的调用也是错误。请改为保留一份深拷贝：`g.get_clone()` 复制该值及其中的所有内容，副本归你所有。

```nabi norun
fn remember(c: Cell<List<i32>>, xs: List<i32>) { c.set(xs); }

let m = Mutex::new([1, 2, 3]);
let kept: Cell<List<i32>> = Cell::new([]);
using g = m.lock() { remember(kept, g.get()); }
```

```text check=checks/12-lock-view-kept.nabi
error[E0106]: `g.get()` is a view of what the `Mutex` behind `g` protects, and `remember` keeps it past the call
  --> main.nabi:5:37
   |
 5 | using g = m.lock() { remember(kept, g.get()); }
   |                                     ^^^^^^^
   |
   = help: `g`'s `using` block releases the lock while `remember` still holds it; pass a deep copy, `g.get_clone()`
```

### 接收端得到什么

交接在运行时也会复制：新的所有者得到该值能到达的每个集合的各自副本（列表、字典、双端队列、字符串缓冲区以及基于它们构建的集合，任意深度）。所以即使是编译器看不到的途径，也绝不会让两个线程共用一个列表。这一点你什么都不用写，在编译器接受的程序中你也不会察觉，只有两点例外：

- **耗时与复制的量成正比**。数字、`str` 和不含集合的结构体没有额外开销；纯数字的列表是一次内存复制，所以一百万个数字的列表远不到一毫秒。大数据要一次发送，不要在热循环中一点一点地发。
- **副本保留自己的共享关系**。发送的值中指向同一个列表的两个句柄，在另一边仍然是同一个列表，切片也仍然是它所来自的那个列表的视图：

```nabi
let data = [1, 2, 3, 4];
let (tx, rx) = channel();
try! tx.send((data, data[0:2]));
tx.close();

if let (list, head) = rx.recv() {
    list[0] = 10;           // the view still sees the list it came from
    println(f"{list:?} {head:?}");
}
```

```output
[10, 2, 3, 4] [10, 2]
```

在编译期构建的值，例如字段的默认值（`log: Mutex<List<str>> = Mutex::new([])`），不会被复制：此时还没有其他东西能到达它们。

有两类值完全不能跨线程，因为复制也无法让它们变得安全：

- **`any Tr` 需要 `Sync` 约束**。`any` 的副本共享它的盒子，所以只有当 trait 同时以 `Send` 和 `Sync` 作为父 trait 时，`any Tr` 才是 `Send`。错误信息会说明要改什么：`an any crosses threads only when its bound has both Send and Sync, since copies share its box: give the trait Sync as a supertrait, or send the concrete type`。

```nabi
use std::thread;

trait Shape: Send + Sync { fn area(self) -> f64; }
struct Sq { s: f64; }
impl Shape for Sq { fn area(self) -> f64 { self.s * self.s } }

let (tx, rx) = channel::<any Shape>();
let h = thread::spawn(|| {
    if let s = rx.recv() { s.area() } else { 0.0 }
});
try! tx.send(Sq { s: 3.0 });
tx.close();
println(f"{try h.join()}");
```

```output
9
```

- **`Weak` 留在自己的线程上**。交接无法复制 `Weak`（`bare::gc`）所指向的东西，所以 `Weak<T>` 不是 `Send`；请改为发送 `GcPtr`。

在 `unsafe` 中编译器不再检查交接，所以发送之后再使用值由你自己负责；运行时的复制仍然会发生。`Mutex::new_unshared` 是*不*复制就交出值的 `unsafe` 方式，用于刚创建、没有其他东西能到达的值。在 `using` 块之后仍从守卫保留的句柄，以及共享同一个列表的 `par` 项，都不会自动复制：在这些地方编译器的检查是唯一的保护，所以不要在那里用 `unsafe` 关掉它。

## `async` 与任务

对于等待时间长的工作，例如 I/O，`async` 更好，因为它不必为每项工作占用一个线程。调用 `async fn` 会立即返回一个 Future，`await` 等待它的结果。用 `task::spawn` 把 Future 作为单独的任务启动，可以让多个任务同时推进，`task::join` **按启动的顺序**收集所有结果。

```nabi
async fn fetch(id: i32) -> str {
    await task::sleep(Duration::ms((40 - id * 10) as i64));
    f"item-{id}"
}

let tasks = [task::spawn(fetch(1)), task::spawn(fetch(2)), task::spawn(fetch(3))];
let items = await task::join(tasks);
println(f"{items:?}");

let f = fetch(3);
println("not started waiting yet");
println(await f);
```

```output
["item-1", "item-2", "item-3"]
not started waiting yet
item-3
```

`await` 是前缀运算符，结合得和 `-` 或 `!` 一样紧：它作用于紧跟其后的调用，表达式的其余部分作用于结果。`await answer() == 42` 比较的是 await 得到的值；它不会 await 一个比较。

```nabi
async fn answer() -> i32 { 42 }

println(f"{await answer() == 42} {await answer() + 1}");
let r = try await task::timeout(Duration::ms(50), task::spawn(answer()));
println(f"{r}");
```

```output
true 43
42
```

如最后一行所示，`try await f()` 从左往右读：先 await 这个 future，再传播它的错误。

如果顶层语句中含有 `await`，整个程序就作为 `async fn main` 运行，所以在脚本中可以直接使用 `await`。在普通 `fn` 的函数体中不能使用 `await`。

用 `task::timeout` 设置时间限制。时间用完时，它抛出 `TimeoutError` 并取消任务。`task::race` 给出最先完成的那个任务的结果，并以同样的方式取消其他任务。

```nabi
async fn slow() -> i32 {
    await task::sleep(Duration::ms(200));
    1
}

let r = try await task::timeout(Duration::ms(10), task::spawn(slow()))
    catch e: task::TimeoutError { -1 };
println(f"{r}");
```

```output
-1
```

`async` 方法和同步方法一样可以接受 `ref self`：它的修改在 future 完成时到达接收者。这样的调用要在调用处 await，即 `await r.next_byte()`；保留 future 以后再 await 是错误，因为接收者可能在此期间改变。异步 io 方法（`read_async`、`write_all_async` 等，[第 13 章](13-io.html)）也接受 `ref self`。

```nabi
struct Cursor { pos: usize; data: List<u8>; }

impl Cursor {
    async fn next_byte(ref self) -> u8 | none {
        if self.pos >= self.data.len() { return none; }
        self.pos += 1;
        self.data[self.pos - 1]
    }
}

let c = Cursor { pos: 0, data: [7u8, 8, 9] };
while await c.next_byte() is u8 {}
println(f"{c.pos}");
```

```output
3
```

通道在异步代码中也能用。`for await` 等待值而不阻塞任务。下面的任务在结束时释放 `tx`，通道随之关闭。

```nabi
let (tx, rx) = channel();
let producer = task::spawn(async {
    for i in 1..=3 {
        try! tx.send(i);
        await task::sleep(Duration::ms(5));
    }
    "producer done"
});

let sum = 0;
for await v in rx {
    println(f"got {v}");
    sum += v;
}
println(f"{await producer}, sum = {sum}");
```

```output
got 1
got 2
got 3
producer done, sum = 6
```

`task::spawn` 交出 future 的方式与通道发送交出值相同：任务在 future 所捕获的列表和其他集合的各自副本上工作。之后在启动任务的代码中使用其中任何一个都是错误，因为任务所做的修改不会在那里体现：

```nabi norun
async fn total(xs: List<i32>) -> i32 {
    let t = 0;
    for x in xs { t += x; }
    t
}

let data = [1, 2, 3];
let t = task::spawn(total(data));
data.push(4);
```

```text check=checks/12-task-handoff.nabi
error[E0105]: `data` was spawned as a task at line 8
  --> main.nabi:9:1
   |
 8 | let t = task::spawn(total(data));
   |         ------------------------ `data` spawned here
 9 | data.push(4);
   | ^^^^
   |
   = help: the task works on its own copy, so changes it makes are not seen here: return them from the task (`await`), hand over `data.clone()` to keep a separate copy, or share a collection it only reads with no copy: `.freeze()` it
```

请返回任务修改过的内容，再用 `await` 取回：

```nabi
async fn grow(xs: List<i32>) -> List<i32> {
    xs.push(4);
    xs
}

let data = [1, 2, 3];
let t = task::spawn(grow(data));
data = await t;
println(f"{data:?}");
```

```output
[1, 2, 3, 4]
```

## `par`：结构化并行

对许多份数据应用同一个计算时，不需要自己管理线程。`core::par` 把工作分配到一个工作线程池上，函数返回时所有工作都已完成。`map` 和 `filter` 保持输入顺序。

```nabi
fn is_prime(n: i32) -> bool {
    if n < 2 { return false; }
    let d = 2;
    while d * d <= n {
        if n % d == 0 { return false; }
        d += 1;
    }
    true
}

let squares = par::map(1..=8, |x| x * x);
println(f"{squares:?}");

let count = Atomic::new(0);
par::for_each(1..=10000, |n| {
    if is_prime(n) { count.fetch_add(1); }
});
println(f"primes below 10000: {count.load()}");

let evens = par::filter([1, 2, 3, 4, 5, 6], |x| x % 2 == 0);
let total = par::fold([1, 2, 3, 4], 0, |a, b| a + b);
println(f"{evens:?} {total}");
```

```output
[1, 4, 9, 16, 25, 36, 49, 64]
primes below 10000: 1229
[2, 4, 6] 10
```

`fold` 分别折叠每个分块，再合并结果，所以必须传入满足结合律的函数（`+`、`max` 等），结果才是确定的。`par::reduce` 做同样的事，但没有初始值，没有项时给出 `none`：`par::reduce(xs, |a, b| a.max(b))`。

正好两项工作时，用 `par::join`；数量在运行时才确定时，用 `par::scope`。在作用域内 `spawn` 的每项工作都会在作用域结束前完成。

```nabi
let (a, b) = par::join(|| (1..=100).sum(), || "done");
println(f"{a} {b}");

par::scope(|s| {
    let left = s.spawn(|| (1..=50).sum());
    let right = s.spawn(|| (51..=100).sum());
    println(f"{left.join() + right.join()}");
});
```

```output
5050 done
5050
```

`par` 把每一项交给一个工作线程，所以两项不能共享一个列表或其他集合：两个工作线程会同时修改它。开始之前，`par` 会检查各项到达的是不同的集合。如果其中两项共享一个集合，这些项就在调用线程上依次运行，结果正确但不再并行，并且 `par` 会对这次调用发出一次警告：

```nabi norun
let shared = [1, 2, 3];
par::for_each([shared, shared], |xs| { xs.sort(); });
```

```text
warning: par::for_each at main.nabi:2:1: two items reach one collection or future, so they run one at a time on this thread; give each item its own copy, freeze what they only read (`xs.freeze()`), or split a list with `chunks_mut`
```

覆盖同一个列表不同部分的切片不算共享，所以把一个大列表拆成分块仍然是并行的。这项检查会访问各项到达的每个集合，每次调用对每个集合有少量开销。

### 并行处理大列表

由多个工作线程运行的闭包不能捕获列表，因为两个工作线程可能同时修改它。请改为把列表的一部分作为项交给每个工作线程：读取用 `chunks(n)`，写入用 `chunks_mut(n)`。各部分互不重叠，所以每个工作线程就地修改列表中属于自己的部分，无需复制，也无需加锁。冻结的列表可以被捕获，因为没有人修改它。

```nabi
let xs: List<f64> = List::from_fn(1000, |i| i as f64);

// Read: one sum per part, then add the parts up.
let parts = par::map(xs.chunks(100), |part| {
    let s = 0.0;
    for x in part { s += x; }
    s
});
let total: f64 = parts.iter().sum();
println(f"sum {total}");

// Write: each worker doubles its own part.
par::for_each(xs.chunks_mut(100), |part| {
    for i in 0..part.len() { part[i] = part[i] * 2.0; }
});
println(f"last {xs[999]}");

// Several lists of one length, as in a game's entity data: zip the parts.
let pos: List<f64> = List::filled(1000, 0.0);
let vel: List<f64> = List::filled(1000, 1.5);
let dt = 0.5;
par::for_each(pos.chunks_mut(100).zip(vel.chunks(100)), |(p, v)| {
    for i in 0..p.len() { p[i] = p[i] + v[i] * dt; }
});
println(f"pos {pos[0]} {pos[999]}");
```

```output
sum 499500
last 1998
pos 0.75 0.75
```

每一部分是一个 `SliceMut`，即写入列表的视图（[第 8 章](08-collections-iterators.html)）。如果两项的部分重叠，或者某一项持有整个列表，`par` 就带着上面的警告逐项运行，所以出错只会损失速度，绝不会损失正确性。

## 流

`async yield fn` 创建一个流，异步地逐个产生值。可以把它看作生成器的异步版本；用 `for await` 消费它。

```nabi
async yield fn ticks(n: i32) -> i32 {
    for i in 1..=n {
        await task::sleep(Duration::ms(5));
        yield i * 10;
    }
}

for await t in ticks(3) {
    println(f"tick {t}");
}
```

```output
tick 10
tick 20
tick 30
```

保存在变量（或字段）中的流是就地轮询的，而不是通过副本：用 `break` 提前停止的循环会让流停在停下的位置，之后对它的下一个 `for await` 从那里继续。拥有资源（文件、套接字）的流只有一个所有者，所以 `let t = s` 会移动它，之后就不能再遍历 `s`。

`Stream` trait 有 `map`、`filter`、`take`、`throttle` 等组合器（[`Stream` API](../../std/core/stream.html#trait.Stream)）。

## 常见陷阱

**`[ref x]` 捕获不能跨线程**。`[ref x]` 单元（[第 3 章](03-functions.html)）没有同步，所以编译器在传给 `thread::spawn`、`task::spawn` 或 `par::*` 的闭包中拒绝它，因为那里两个线程可能同时修改它。

```text check=checks/12-ref-thread.nabi
error[E0085]: `[ref count]` can't be used in a closure that may run on another thread: both threads could change `count` at once
   = help: write `let count = Atomic::new(count);` before the closure, capture `count` without `ref`, and use `count.load()` / `count.store(..)`; or return the value from the closure instead
```

请改为共享 `Atomic`（用于数字）或 `Mutex`（用于其他任何东西），或者从线程返回值，在 `join` 之后合并结果。

```nabi
use std::thread;

let count = Atomic::new(0);
let hs: List<thread::JoinHandle<()>> = [];
for _ in 0..4 {
    hs.push(thread::spawn(|| {
        for _ in 0..10000 { count.fetch_add(1); }
    }));
}
for h in hs { try h.join(); }
println(f"{count.load()}");
```

```output
40000
```

**由多个线程运行的闭包不能捕获集合**。普通捕获与创建闭包的代码共享，所以被 `par`、`thread::spawn` 或 `Scope::spawn` 闭包捕获的列表可能被两个线程同时修改。编译器会告诉你应该怎么写：

```text check=checks/12-capture-list.nabi
error[E0040]: mismatched types: argument 2 of `map` expects `fn(i64) -> i64 + Send + Sync`, found `closure(i64) -> i64` - the closure's captures don't meet the `fn` type's bounds
  --> main.nabi:2:26
   |
 2 | let out = par::map(0..3, |i| table[i] * 2);
   |                          ^^^^^^^^^^^^^^^^
   |
   = note: the closure captures `table` (type `List<i64>`), which is not `Sync` (collections are not `Sync`: share one read-only with `.freeze()`, or through a `Mutex` to change it)
   = help: share `table` read-only, with no copy: `let table = table.freeze();` before the closure; to write parts of it in parallel, give each worker its own part as an item: `par::for_each(table.chunks_mut(n), |part| ..)`
```

**`static` 必须是 `Sync`**。`static` 对每个线程都可见，所以它的类型必须能安全共享。`static XS: List<i32> = [];` 是错误：只读的列表要冻结（`static XS: FrozenList<i32> = [1, 2, 3].freeze();`），会修改的列表要用 `Mutex` 包起来。

```nabi
static SEEN: Mutex<List<i32>> = Mutex::new([]);

fn record(x: i32) {
    using g = SEEN.lock() { g.get().push(x); }
}

record(1);
record(2);
println(f"{using g = SEEN.lock() { g.get_clone() }:?}");
```

```output
[1, 2]
```

**没有 join 的线程会在 `main` 返回时被终止**。程序在 `main` 结束时结束，仍在运行的线程也随之停止，停在半途。请保留 `JoinHandle` 并 `join` 它。`nabi check` 和 `nabi lint` 会对从未 join 的句柄发出警告：

```text check=checks/12-unjoined.nabi
warning[E0131]: this thread is never joined, so the program may end before it finishes
  --> main.nabi:3:1
   |
 3 | thread::spawn(|| println("hi"));
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = help: join it with `try thread::spawn(|| println("hi")).join()`, or write `thread::spawn(|| println("hi")).detach()` to let it run on its own
```

如果线程确实应该在你不再关心它之后继续运行，就对句柄调用 `detach()`。

```nabi
use std::thread;

let t = thread::spawn(|| {
    thread::sleep(Duration::ms(10));
    println("thread done");
});
println("main done");
try t.join();               // without this line, "thread done" may never print
```

```output
main done
thread done
```

**Future 是惰性的**。调用 `async fn` 只会创建一个 future；它的函数体在你 `await` 它（或者把它交给 `task::spawn`）时才运行。从未被 await 的 future 永远不会运行。这不是错误，但 `nabi check` 和 `nabi lint` 会警告：`this future is never awaited, so its body never runs`，并建议写成 `await f`。`task::spawn` 返回的 `Task` 被丢弃时不会有警告：启动的任务会自行运行（发后不管）。

```nabi
async fn work() -> i32 {
    println("work ran");
    1
}

let f = work();             // nothing runs yet
println("before await");
println(f"{await f}");
```

```output
before await
work ran
1
```

**`task::spawn` 需要异步的 `main`**。任务在执行器上运行，而执行器由 `async fn main` 启动。脚本的顶层一旦含有 `await` 就会成为这样的 `main`；否则 `task::spawn` 会触发 panic。

```text
thread 'main' panicked at .../library/core/src/task.nabi:564:18:
no task executor is running: await inside `async fn main` or `task::block_on`
```

所以，要像上面的示例那样 `await` 你启动的任务。要从普通 `fn` 调用异步代码，使用 `task::block_on`，它把 future 运行到完成并返回它的值。

```nabi
async fn work() -> i32 { 7 }

fn compute() -> i32 {
    task::block_on(work())
}

println(f"{compute()}");
```

```output
7
```

## 小结

- `thread::spawn` 启动一个操作系统线程；用 `try h.join()` 取得它的结果。
- 用 `channel()` 创建通道。`Sender` 在最后一个所有者离开作用域时被释放；在线程中，或者接收端在同一个作用域中读取时，要自己 `close()` 它。
- prelude 中有 `task`、`par`、`channel`、`Mutex` 和 `Atomic`；`std::thread` 需要 `use`。
- 用 `Atomic` 或 `Mutex` 保护共享的值。锁守卫必须用 `using` 绑定，否则就是编译错误。`[ref x]` 捕获不能跨线程，`static` 必须是 `Sync`。
- 交给 `Mutex::new` 或 `send` 的值属于它的新所有者：之后再使用它或它的某个元素，是编译错误（`E0105`）。请通过锁使用它，交出 `clone()`，或者在发送之前用 `pop` / `remove` 取出元素。`g.get()` 返回的东西留在 `using` 块内；要保留就用 `g.get_clone()`。
- 接收端得到值中集合的各自副本；副本保留自己的共享关系，耗时与其大小成正比。只有当 trait 带有 `Send + Sync` 时 `any Tr` 才能跨线程，`Weak` 则永远不能。
- `thread::local` 给每个线程一个各自的值：在 `with(|v| ..)` 中使用它，或者用 `set` / `replace` / `take` 移入移出。
- 只读的集合用 `freeze()` 共享：`FrozenList` / `FrozenDict` / `FrozenSet` 可被所有线程读取，无需加锁，也无需复制。以读为主的状态，在 `Mutex` 中保存一个冻结快照并整体替换。`Frozen<T>` 给每个读者一份值的各自副本。
- `await` 是作用于其后调用的前缀运算符（`await f() == x` 比较的是结果）。用 `async fn` / `await` 和 `task::spawn` / `task::join` 并发运行等待较多的工作。
- `task::spawn` 像发送一样交出它的 future：任务在 future 所捕获内容的各自副本上工作，所以要用 `await` 取回它的修改。闭包用 `[move xs]` 把列表交给线程，这样的闭包只能交出一次：`par::join(f, f)` 是错误。
- 要 join 你的线程、await 你的 future：没有 join 的线程随 `main` 一起终止，future 只有被 await 时才运行；lint 对两者都会警告。被丢弃的 `task::spawn` 任务仍会运行。`task::spawn` 需要异步的 `main`（或 `task::block_on`）。
- 数据并行计算用 `par::map`、`par::for_each`、`par::join` 和 `par::scope` 就能保持简单。处理大列表时把一部分交给每个工作线程：读取用 `par::map(xs.chunks(n), ..)`，就地写入用 `par::for_each(xs.chunks_mut(n), ..)`。共享集合的项会逐项运行，并发出警告。
- 用 `async yield fn` 和 `for await` 创建和消费流。

## 延伸阅读

- 规范：[异步（`async` / `await`）](../spec/refs-lang/004-functions/412-async.html)、[运算符优先级](../spec/refs-lang/003-expressions/302-precedence-and-associativity.html)、[流](../spec/refs-lang/004-functions/413-stream.html)、[`using`](../spec/refs-lang/003-expressions/317-using-expression.html)、[跨线程的捕获](../spec/refs-lang/004-functions/406-capture-semantics.html)、[`static`](../spec/refs-lang/005-modules/503-static.html)
- 标准库规范：[sync](../spec/refs-standard-libraries/200-core/216-sync.html)、[channel](../spec/refs-standard-libraries/200-core/217-channel.html)、[task](../spec/refs-standard-libraries/200-core/218-task.html)、[par](../spec/refs-standard-libraries/200-core/219-par.html)
- API：[`thread::spawn`](../../std/std/thread.html#fn.spawn)、[`task::spawn`](../../std/core/task.html#fn.spawn)、[`task::join`](../../std/core/task.html#fn.join)、[`channel`](../../std/core/channel.html#fn.channel)、[`Mutex`](../../std/core/sync.html#struct.Mutex)、[`Atomic`](../../std/core/sync.html#struct.Atomic)、[`par::map`](../../std/core/par.html#fn.map)、[`par::scope`](../../std/core/par.html#fn.scope)
- 下一章：[13. 文件与网络 IO](13-io.html)
