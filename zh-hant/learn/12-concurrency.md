# 12. 並行

同時做好幾件事的正確方式取決於目標。執行緒和 `par` 適合使用多顆 CPU 的計算，而 `async` 任務適合需要大量等待的 I/O。本章透過簡短的範例逐一介紹這些工具，也會說明用來安全交換資料的通道、`Mutex` 和 `Atomic`。

> [!TIP]
> 並行執行的程式碼，輸出順序每次執行都可能不同。本章的範例會先用 `join` 收集結果或先排序再印出，所以輸出永遠相同。

這些當中哪些需要 `use`？本章從 `core` 用到的東西都在預載模組（prelude，[第 11 章](11-modules-packages.html)）中：`task`、`par`、`channel`、`Mutex`、`Atomic`、`Duration`。只有與作業系統溝通的 `std` 模組需要手動匯入，所以使用執行緒要先寫 `use std::thread;`，就像使用檔案要先寫 `use std::fs;` 一樣。

## 執行緒

`std::thread::spawn` 在新的作業系統執行緒上執行閉包，並回傳 `JoinHandle`。`join()` 會等待該執行緒結束，並取出閉包的回傳值。如果執行緒以 panic 結束，它會擲回 `JoinError`，所以範例加上了 `try`。

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

因為範例在建立完所有控制代碼之後才 join，所以四個執行緒會並行執行。結果依控制代碼的順序收集，所以輸出順序是固定的。

## 通道

在執行緒之間傳遞值，最簡單的方式是通道（channel）。`channel()` 回傳負責傳送的 `Sender` 和負責接收的 `Receiver`。`Sender` 可以用 `clone()` 複製，當每個 `Sender` 都已 `close()` 之後，接收端的 `for` 迴圈就會結束。

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

`Sender` 只有一個擁有者：`let tx2 = tx` 會移動它（之後再使用 `tx` 是錯誤），而 `clone()` 會產生另一個擁有自己擁有者的傳送端。編譯器會在傳送端的最後一個擁有者離開作用域的地方釋放它，任務則會在結束時釋放其 `async` 區塊所擷取的傳送端；一旦每個傳送端都已關閉或釋放，接收端的迴圈就會結束。當接收端必須更早看到結尾時，請自己呼叫 `close()`：這裡的 `main` 在建立 `tx` 的同一個作用域中讀取通道，而執行緒不會釋放其閉包擷取的東西，所以每個工作執行緒在完成時都會關閉自己的傳送端。要限制通道的容量，請使用 `channel_bounded(n)`。

## `Mutex` 與 `Atomic`

當多個執行緒修改同一個值時，就需要同步。單一整數計數器請用 `Atomic`；其他值則用 `Mutex` 包起來。`Mutex::lock()` 回傳的守衛（guard）必須用 `using` 繫結，讓鎖在區塊結束時釋放（與[第 10 章](10-errors.html)中 `using` 的規則相同）。只做一個簡短步驟時，`m.with(|v| ..)` 會取得鎖、以該值呼叫閉包，並在閉包回傳時釋放鎖，如下面的範例所示；`RwLock` 則有 `read_with` 和 `write_with`。

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

串列是共享的，所以 `v.push(..)` 會改變鎖後面的那個串列。結構則是值：閉包和守衛的 `get()` 交出的是複本，在複本上設定的欄位會遺失，這是錯誤（[E0057](../errors/E0057.html)、[E0140](../errors/E0140.html)）。請透過守衛用 `update` 或 `set` 儲存變更：`using g = m.lock() { g.update(|v| { v.total += 1; v }); }`。

只有 `using` 會釋放守衛，所以編譯器會拒絕沒有任何東西釋放的守衛：用一般的 `let`（或 `if let`）繫結它，或是直接丟棄 `lock()` 的結果。`RwLock` 的 `read()` / `write()` 守衛也是如此。

```nabi norun
let g = m.lock();
```

```text check=checks/12-guard-let.nabi
error[E0065]: `g` holds a `MutexGuard` that is never released: only `using` releases a lock guard, so bind it with `using g = .. { .. }`
```

### 唯讀資料：`freeze`

只建立一次、之後只會讀取的集合，例如查找表或名稱串列，不需要鎖。`freeze()` 會把 `List`、`Dict` 或 `Set` 轉成任何東西都無法改變的 `FrozenList`、`FrozenDict` 或 `FrozenSet`，所以每個執行緒都能同時讀取同一份，不用鎖也不用複製。執行緒的閉包、`par` 閉包或 `static` 都可以持有它。

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

`freeze` 會像 `Mutex::new` 一樣把集合交出去：凍結後的值接收了元素，所以之後再使用原本的串列是錯誤（[E0105](../errors/E0105.html)）。請改為讀取凍結後的值；如果你仍需要一個可以改變的串列，就凍結一份 `clone()`。凍結的集合沒有 `push`、`insert` 或 `xs[i] = v`；`to_list()`（`to_dict()`、`to_set()`）會產生一份可以改變的複本。

只有由本身無法改變的值所組成的集合才能凍結：數字、`str`、由這些組成的結構，以及其他凍結的集合。`List<List<i32>>` 會讓每個執行緒拿到同樣的內層串列，而內層串列是可以改變的，所以編譯器會要求你先凍結內層串列（`List<FrozenList<i32>>`），或改用 `Mutex`。

經常讀取、偶爾才改變的資料，例如伺服器的路由表，可以做成放在鎖後面的凍結快照。讀取者取得目前的快照，在不持有鎖的情況下讀取；寫入者製作一份修改過的複本，凍結後放回原位。凍結的值可以離開鎖，因為沒有任何東西能改變它。

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

第一次閱讀時可以跳過本小節的其餘部分。對於由好幾個部分組成的值，例如帶有幾個小串列的設定，還有 `Frozen<T>` 可用。`Frozen::new` 會保有自己的一份複本，`get()` 則給每個讀取者各自的複本，所以讀取者修改自己的複本，並不會改變其他人看到的內容。

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

每次 `get` 都會複製值內部的集合，所以請讓大型的唯讀部分不要落入那份複本：把它們凍結，例如 `table: FrozenList<Entry>`。凍結的集合和其他 `Sync` 部分（例如 `Mutex`）是共享的，不會被複製。

### 每個執行緒各自的值：`thread::local`

第一次閱讀時可以跳過本小節。

`thread::local(init)` 給每個執行緒各自的值，在該執行緒第一次使用時由 `init` 建立。`with` 把這個執行緒的值借給閉包；閉包回傳的東西會被帶出來，但值本身仍是借出的狀態：回傳它、把它存到閉包之外，或傳送它都是錯誤，而且閉包不能是 `async`。`set`、`replace` 和 `take` 會把整個值移入和移出，`get` 則複製數字之類的 `Copy` 值。

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

## 把值交給其他執行緒

本節說明值在執行緒之間移動時，編譯器所強制的規則。第一次閱讀時可以略讀，等編譯器把你指引到這裡時再回來看。

`Mutex::new(xs)`、`RwLock::new(xs)` 和 `tx.send(xs)` 會**交出 `xs`**：從那之後它屬於鎖或接收端。在 Nabi 中，只有 `Sender` 或 `File` 這類 `Drop` 資源會移動（[第 10 章](10-errors.html)）；對於其他值，編譯器改為追蹤交接：之後再使用 `xs` 是編譯錯誤，因為那會與它現在的擁有者發生競爭。這項檢查會透過複本（`let ys = xs`）、結構欄位、持有它的物件、集合的元素、閉包擷取以及函式呼叫一路追蹤這個值。

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

請透過鎖使用這個值；如果你仍需要自己的一份，就交出 `clone()`。

元素也算在內。從集合讀出的東西與集合本身是同一個值，所以先傳送 `rows`、再對 `first = rows[0]` 做 push，同樣是競爭：

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

要一個一個交出元素，請先把每個元素取出來。`pop` 和 `remove` 會把元素從集合中取出，所以集合仍可繼續使用。這是常見的工作佇列：

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

### 把值移入執行緒：`[move x]`

傳給 `thread::spawn` 或 `task::spawn_blocking` 的閉包，會與建立它的程式碼共享擷取的東西，所以這些東西必須是 `Sync`：串列不是。當只有新的執行緒需要該串列時，用 `[move rows]` 把它移入閉包。閉包會取得自己的一份，而產生執行緒的程式碼就不能再使用 `rows`：

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

在 spawn 之後使用 `rows` 是錯誤，就像傳送之後一樣；當你擷取串列卻沒有寫 `[move x]` 時，說明訊息也會提到它。對於 `task::spawn_blocking`，你也可以改把值當作引數傳入：`task::spawn_blocking_with_arg(rows, |rows: List<i32>| rows.len())`。

`fn` 值也以同樣的方式擷取。參數 `f: fn() + Send` 是 `Send` 但不是 `Sync`，所以呼叫它的 spawn 閉包必須把它移入：`[move f]`。若要從好幾個 spawn 閉包呼叫它，請改為接收 `fn() + Send + Sync`。

擁有被移入之值的閉包只能交出一次，不論經由什麼途徑：透過另一個變數、欄位、串列、`if` 分支，或回傳它的函式。把同一個閉包兩次傳給同一個呼叫，例如 `par::join(f, f)`，也是第二次交出，同樣是錯誤（[E0133](../errors/E0133.html)）；請建立兩個閉包，各自移入自己的 `.clone()`。`async [move ..]` 閉包與它的呼叫所產生的 future 共用同一個擁有者：你可以在這裡依序 await 它們，但把其中一個交給 `task::spawn` 會把其他的也一起帶走。

鎖守衛的值也是同樣的道理。`g.get()` 回傳的是 `Mutex` 所保護之物件的檢視，所以它不能離開 `using` 區塊：區塊一結束，鎖就不再保護它。回傳它、把它放進錯誤中擲回，或傳送它都是錯誤；把它當作 `break` 值、`if` 或 `match` 分支、`catch` 或閉包的結果帶出區塊，也同樣是錯誤。

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

只要函式只是使用它，把它傳給函式是可以的。會保留它的函式（存進 `Cell`、欄位、閉包或執行緒區域變數）會讓它的壽命超過鎖，所以那樣的呼叫也是錯誤。請改為保留深層複本：`g.get_clone()` 會複製該值及其中的一切，複本就歸你所有。

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

### 接收端拿到什麼

交接在執行時也會複製：新的擁有者會拿到該值觸及的每個集合（串列、字典、雙端佇列、字串緩衝區，以及建立在它們之上的集合，不論深度）各自的一份複本。所以即使是編譯器看不到的途徑，也絕不會讓兩個執行緒共用同一個串列。你不需要為此寫任何東西，而且在編譯器接受的程式中，你永遠不會察覺它，只有兩點例外：

- **它花費的時間與複製的量成正比。** 數字、`str` 和不含集合的結構不會有額外成本；純數字的串列只是一次記憶體複製，所以一百萬個數字的串列花不到一毫秒。大型資料請一次傳送，不要在熱迴圈中一塊一塊地傳。
- **複本保有自己的共享關係。** 傳送的值中指向同一個串列的兩個控制代碼，在另一端仍是同一個串列，而切片也會繼續檢視它原本來自的串列：

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

在編譯時建立的值，例如欄位的預設值（`log: Mutex<List<str>> = Mutex::new([])`），不會被複製：此時還沒有其他東西能觸及它們。

有兩種值完全不能跨越執行緒，因為複製也無法讓它們變得安全：

- **`any Tr` 需要 `Sync` 約束。** `any` 的複本共用它的 box，所以只有當 trait 同時以 `Send` 和 `Sync` 作為 supertrait 時，`any Tr` 才是 `Send`。錯誤訊息會說明要改什麼：`an any crosses threads only when its bound has both Send and Sync, since copies share its box: give the trait Sync as a supertrait, or send the concrete type`。

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

- **`Weak` 會留在自己的執行緒上。** 交接無法複製 `Weak`（`bare::gc`）所指的東西，所以 `Weak<T>` 不是 `Send`；請改為傳送 `GcPtr`。

在 `unsafe` 內，編譯器會停止檢查交接，所以傳送之後再使用值是你自己的責任；執行時的複製仍會發生。`Mutex::new_unshared` 是以 `unsafe` 方式**不複製**就交出值的做法，適用於剛建立、沒有其他東西能觸及的值。從守衛取得並保留到 `using` 區塊之後的控制代碼，以及共用同一個串列的 `par` 項目，永遠不會自動為你複製：在這些地方，編譯器的檢查是唯一的防護，所以不要在那裡用 `unsafe` 把它關掉。

## `async` 與任務

對於需要長時間等待的工作，例如 I/O，`async` 比較合適，因為它不會為每項工作佔用一個執行緒。呼叫 `async fn` 會立即回傳 Future，`await` 則等待它的結果。用 `task::spawn` 把 Future 啟動成獨立的任務，可以讓好幾個任務同時推進，而 `task::join` 會**依照產生的順序**收集所有結果。

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

`await` 是前置運算子，結合的緊密程度與 `-` 或 `!` 相同：它套用在緊接其後的呼叫上，運算式的其餘部分則作用在結果上。`await answer() == 42` 比較的是 await 後的值；它不是去 await 一個比較運算。

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

如最後一行所示，`try await f()` 由左往右讀：先 await future，再傳遞它的錯誤。

如果頂層陳述式中含有 `await`，整個程式會以 `async fn main` 的形式執行，所以你可以在指令碼中直接使用 `await`。一般 `fn` 的主體中不能使用 `await`。

用 `task::timeout` 設定時間限制。時間到了，它會擲回 `TimeoutError` 並取消任務。`task::race` 給出最先完成之任務的結果，並以同樣的方式取消其他任務。

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

`async` 方法可以像同步方法一樣接收 `ref self`：它的變更會在 future 完成時套用到接收者上。這樣的呼叫要在呼叫的地方就 await，例如 `await r.next_byte()`；把 future 保留下來之後再 await 是錯誤，因為接收者可能在這段期間改變。非同步 io 方法（`read_async`、`write_all_async` 等，[第 13 章](13-io.html)）也接收 `ref self`。

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

通道在非同步程式碼中也能使用。`for await` 會等待值，而不會阻塞任務。下面的任務在結束時釋放 `tx`，這就會關閉通道。

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

`task::spawn` 把它的 future 交給任務，方式與通道傳送交出值相同：任務在 future 所擷取的串列及其他集合的自有複本上運作。之後在產生任務的程式碼中使用其中任何一個都是錯誤，因為任務所做的變更不會反映在那裡：

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

把任務改變的東西回傳，再用 `await` 取回：

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

## `par`：結構化平行

對許多份資料套用相同的計算時，你不需要自己管理執行緒。`core::par` 會把工作分配到工作執行緒池上，而在函式回傳時，所有工作都已完成。`map` 和 `filter` 會保留輸入的順序。

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

`fold` 會分別摺疊每個區塊，再合併結果，所以你必須傳入具有結合律的函式（`+`、`max` 等），結果才會是確定的。`par::reduce` 做同樣的事，但沒有起始值，沒有項目時會給出 `none`：`par::reduce(xs, |a, b| a.max(b))`。

剛好兩項工作時，請用 `par::join`；數量要在執行時才決定時，請用 `par::scope`。在 scope 內 `spawn` 的每項工作，都會在 scope 結束前完成。

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

`par` 把每個項目交給一個工作執行緒，所以兩個項目不能共用一個串列或其他集合：兩個工作執行緒會同時改變它。在開始之前，`par` 會檢查各項目觸及的是不同的集合。如果其中兩個共用一個集合，這些項目會在呼叫端的執行緒上依序執行；這樣是正確的，但不再平行，而且 `par` 會針對該次呼叫警告一次：

```nabi norun
let shared = [1, 2, 3];
par::for_each([shared, shared], |xs| { xs.sort(); });
```

```text
warning: par::for_each at main.nabi:2:1: two items reach one collection or future, so they run one at a time on this thread; give each item its own copy, freeze what they only read (`xs.freeze()`), or split a list with `chunks_mut`
```

涵蓋同一個串列不同部分的切片不算共用，所以把大串列切成區塊仍然是平行的。這項檢查會走訪項目觸及的每個集合，每次呼叫時每個集合都會花費一點成本。

### 平行處理大型串列

由多個工作執行緒執行的閉包不能擷取串列，因為兩個工作執行緒可能同時改變它。請改為把串列的一部分當作項目交給每個工作執行緒：讀取用 `chunks(n)`，寫入用 `chunks_mut(n)`。各部分互不重疊，所以每個工作執行緒都能就地改變自己那部分的串列，不用複製也不用鎖。凍結的串列可以被擷取，因為沒有人會改變它。

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

每個部分都是 `SliceMut`，一種可以寫入串列的檢視（[第 8 章](08-collections-iterators.html)）。如果兩個項目的部分重疊，或某個項目持有整個串列，`par` 就會依序執行這些項目並且發出上述警告，所以錯誤只會讓速度變慢，絕不會影響正確性。

## 串流

`async yield fn` 會建立一個以非同步方式逐一產生值的串流（stream）。可以把它想成產生器的非同步版本；用 `for await` 來取用它。

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

存放在變數（或欄位）中的串流會在原地輪詢，而不是透過複本：以 `break` 提前停止的迴圈，會讓串流停在它停下的地方，之後對它的下一個 `for await` 會從那裡繼續。擁有資源（檔案、socket）的串流只有一個擁有者，所以 `let t = s` 會移動它，之後就不能再走訪 `s`。

`Stream` trait 有 `map`、`filter`、`take` 和 `throttle` 等組合子（[`Stream` API](../../std/core/stream.html#trait.Stream)）。

## 常見陷阱

**`[ref x]` 擷取不能跨越執行緒。** `[ref x]` 儲存格（[第 3 章](03-functions.html)）沒有同步機制，所以在交給 `thread::spawn`、`task::spawn` 或 `par::*` 的閉包中，編譯器會拒絕它，因為在那裡兩個執行緒可能同時改變它。

```text check=checks/12-ref-thread.nabi
error[E0085]: `[ref count]` can't be used in a closure that may run on another thread: both threads could change `count` at once
   = help: write `let count = Atomic::new(count);` before the closure, capture `count` without `ref`, and use `count.load()` / `count.store(..)`; or return the value from the closure instead
```

請改為共享 `Atomic`（用於數字）或 `Mutex`（用於其他任何東西），或者從執行緒回傳值，在 `join` 之後合併結果。

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

**由多個執行緒執行的閉包不能擷取集合。** 一般的擷取會與建立閉包的程式碼共享，所以被 `par`、`thread::spawn` 或 `Scope::spawn` 閉包擷取的串列，可能會被兩個執行緒同時改變。編譯器會告訴你應該改寫成什麼：

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

**`static` 必須是 `Sync`。** `static` 對每個執行緒都可見，所以它的型別必須能安全地共享。`static XS: List<i32> = [];` 是錯誤：只會讀取的串列就凍結它（`static XS: FrozenList<i32> = [1, 2, 3].freeze();`），會改變的串列則用 `Mutex` 包起來。

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

**沒有 join 的執行緒會在 `main` 回傳時被終止。** 程式在 `main` 結束時就結束，仍在執行的執行緒也會跟著半途停止。請保留 `JoinHandle` 並 `join` 它。`nabi check` 和 `nabi lint` 會對從未 join 的控制代碼發出警告：

```text check=checks/12-unjoined.nabi
warning[E0131]: this thread is never joined, so the program may end before it finishes
  --> main.nabi:3:1
   |
 3 | thread::spawn(|| println("hi"));
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = help: join it with `try thread::spawn(|| println("hi")).join()`, or write `thread::spawn(|| println("hi")).detach()` to let it run on its own
```

如果執行緒確實應該在你不再關心它之後繼續執行，就對控制代碼呼叫 `detach()`。

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

**Future 是惰性的。** 呼叫 `async fn` 只會建立 future；它的主體要等到你 `await` 它（或把它交給 `task::spawn`）時才會執行。從未被 await 的 future 永遠不會執行。這不是錯誤，但 `nabi check` 和 `nabi lint` 會警告 `this future is never awaited, so its body never runs`，並建議寫成 `await f`。丟棄 `task::spawn` 產生的 `Task` 時不會有警告：產生的任務會自行執行（fire-and-forget）。

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

**`task::spawn` 需要非同步的 `main`。** 任務在執行器（executor）上執行，而執行器由 `async fn main` 啟動。指令碼只要頂層含有 `await` 就會變成 `async fn main`；沒有的話，`task::spawn` 會 panic。

```text
thread 'main' panicked at .../library/core/src/task.nabi:564:18:
no task executor is running: await inside `async fn main` or `task::block_on`
```

所以請 `await` 你產生的任務，如上面的範例所示。要從一般的 `fn` 呼叫非同步程式碼，請使用 `task::block_on`，它會把 future 執行到完成並回傳其值。

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

## 小結

- `thread::spawn` 啟動一個作業系統執行緒；用 `try h.join()` 取得它的結果。
- 用 `channel()` 建立通道。`Sender` 會在其最後一個擁有者離開作用域的地方被釋放；在執行緒中，或接收端在同一個作用域中讀取時，請自己 `close()` 它。
- 預載模組包含 `task`、`par`、`channel`、`Mutex` 和 `Atomic`；`std::thread` 需要 `use`。
- 用 `Atomic` 或 `Mutex` 保護共享的值。鎖守衛必須用 `using` 繫結，其他寫法都是編譯錯誤。`[ref x]` 擷取不能跨越執行緒，`static` 必須是 `Sync`。
- 交給 `Mutex::new` 或 `send` 的值屬於它的新擁有者：之後再使用它或它的任何元素，都是編譯錯誤（`E0105`）。請透過鎖使用它、交出 `clone()`，或在傳送前用 `pop` / `remove` 把元素取出。`g.get()` 回傳的東西要留在 `using` 區塊內；要保留就用 `g.get_clone()`。
- 接收端會拿到值中各集合的自有複本；複本保有它的共享關係，花費的時間與其大小成正比。`any Tr` 只有在 trait 帶有 `Send + Sync` 時才能跨越執行緒，`Weak` 則永遠不能。
- `thread::local` 給每個執行緒各自的值：在 `with(|v| ..)` 內使用，或用 `set` / `replace` / `take` 把它移入和移出。
- 只會讀取的集合用 `freeze()` 共享：`FrozenList` / `FrozenDict` / `FrozenSet` 可由每個執行緒讀取，不用鎖也不用複製。對於以讀取為主的狀態，把凍結的快照放在 `Mutex` 中，並整個替換。`Frozen<T>` 給每個讀取者一份值的自有複本。
- `await` 是作用在其後呼叫上的前置運算子（`await f() == x` 比較的是結果）。用 `async fn` / `await` 和 `task::spawn` / `task::join` 並行執行大量等待的工作。
- `task::spawn` 像傳送一樣交出它的 future：任務在 future 所擷取之物的自有複本上運作，所以請用 `await` 取回它的變更。閉包用 `[move xs]` 把串列交給執行緒，而這樣的閉包只能交出一次：`par::join(f, f)` 是錯誤。
- 請 join 你的執行緒、await 你的 future：沒有 join 的執行緒會隨 `main` 結束，而 future 只有在被 await 時才會執行；lint 會對這兩者發出警告。被丟棄的 `task::spawn` 任務仍會執行。`task::spawn` 需要非同步的 `main`（或 `task::block_on`）。
- 資料平行的計算，用 `par::map`、`par::for_each`、`par::join` 和 `par::scope` 就能保持簡單。處理大型串列時，把一部分交給每個工作執行緒：讀取用 `par::map(xs.chunks(n), ..)`，就地寫入用 `par::for_each(xs.chunks_mut(n), ..)`。共用同一個集合的項目會依序執行，並且發出警告。
- 用 `async yield fn` 和 `for await` 建立與取用串流。

## 延伸閱讀

- 規格：[非同步（`async` / `await`）](../spec/refs-lang/004-functions/412-async.html)、[運算子優先順序](../spec/refs-lang/003-expressions/302-precedence-and-associativity.html)、[串流](../spec/refs-lang/004-functions/413-stream.html)、[`using`](../spec/refs-lang/003-expressions/317-using-expression.html)、[跨執行緒的擷取](../spec/refs-lang/004-functions/406-capture-semantics.html)、[`static`](../spec/refs-lang/005-modules/503-static.html)
- 標準函式庫規格：[sync](../spec/refs-standard-libraries/200-core/216-sync.html)、[channel](../spec/refs-standard-libraries/200-core/217-channel.html)、[task](../spec/refs-standard-libraries/200-core/218-task.html)、[par](../spec/refs-standard-libraries/200-core/219-par.html)
- API：[`thread::spawn`](../../std/std/thread.html#fn.spawn)、[`task::spawn`](../../std/core/task.html#fn.spawn)、[`task::join`](../../std/core/task.html#fn.join)、[`channel`](../../std/core/channel.html#fn.channel)、[`Mutex`](../../std/core/sync.html#struct.Mutex)、[`Atomic`](../../std/core/sync.html#struct.Atomic)、[`par::map`](../../std/core/par.html#fn.map)、[`par::scope`](../../std/core/par.html#fn.scope)
- 下一章：[13. 檔案與網路 IO](13-io.html)
