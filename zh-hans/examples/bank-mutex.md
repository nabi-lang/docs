# 银行转账

多个线程在由 `Mutex` 保护的账户之间转账，用 `Atomic` 计数器收集统计数据，最后检查总额从未改变。

## 代码

```nabi
// Threads move money between accounts; locks keep the total constant.
use std::random::Rng;
use std::{random, thread};

const ACCOUNTS: usize = 5;
const THREADS: i32 = 4;
const TRANSFERS: i32 = 20000;
const START: i64 = 1000;

struct Bank {
    // A fixed array, not a List: a List is a shared mutable handle and is
    // not `Sync`, so threads may not share one. The array's elements are
    // `Mutex` handles, which are.
    accounts: [Mutex<i64>; 5];
    moved: Atomic<i64>;
    rejected: Atomic<i64>;
}

impl Bank {
    fn new() -> Bank {
        let accounts: [Mutex<i64>; 5] = [
            Mutex::new(START), Mutex::new(START), Mutex::new(START), Mutex::new(START),
            Mutex::new(START),
        ];
        Bank { accounts: accounts, moved: Atomic::new(0), rejected: Atomic::new(0) }
    }

    // Lock both accounts, always the lower index first, so two opposite
    // transfers can never wait for each other forever.
    fn transfer(self, from: usize, to: usize, amount: i64) {
        if from == to { return; }
        let first = if from < to { from } else { to };
        let second = if from < to { to } else { from };
        using a = self.accounts[first].lock() {
            using b = self.accounts[second].lock() {
                let (src, dst) = if from == first { (a, b) } else { (b, a) };
                if src.get() < amount {
                    self.rejected.fetch_add(1);
                } else {
                    src.set(src.get() - amount);
                    dst.set(dst.get() + amount);
                    self.moved.fetch_add(amount);
                }
            }
        }
    }

    // Sum of all balances (called while no transfers are running).
    fn total(self) -> i64 {
        let sum = 0;
        for m in self.accounts {
            sum += m.with(|v| v);
        }
        sum
    }
}

fn main() {
    let bank = Bank::new();
    let before = bank.total();

    let handles: List<thread::JoinHandle<i32>> = [];
    for t in 0..THREADS {
        handles.push(thread::spawn(|| {
            let rng = random::Xoshiro256PlusPlus::seed(t as u64 + 1);
            for _ in 0..TRANSFERS {
                let from = rng.below(ACCOUNTS as u64) as usize;
                let to = rng.below(ACCOUNTS as u64) as usize;
                bank.transfer(from, to, rng.range(1..301));
            }
            TRANSFERS
        }));
    }
    let attempted = 0;
    for h in handles { attempted += try! h.join(); }

    let after = bank.total();
    println(f"{THREADS} threads attempted {attempted} transfers");
    println(f"money moved: {bank.moved.load() > 0}, some rejected for lack of funds: {bank.rejected.load() > 0}");
    for i in 0..ACCOUNTS {
        using g = bank.accounts[i].lock() {
            let ok = if g.get() >= 0 { "ok" } else { "NEGATIVE" };
            println(f"account {i}: balance {ok}");
        }
    }
    println(f"total before {before}, after {after}, invariant holds: {before == after}");
}
```

```output
4 threads attempted 80000 transfers
money moved: true, some rejected for lack of funds: true
account 0: balance ok
account 1: balance ok
account 2: balance ok
account 3: balance ok
account 4: balance ok
total before 5000, after 5000, invariant holds: true
```

## 讲解

- 每个账户是一个独立的 `Mutex<i64>`，存放在定长数组中。这里用 `List` 是无法编译的：列表是共享的可变句柄，不是 `Sync`，所以线程闭包不能捕获它（[4.6.9](../spec/refs-lang/004-functions/406-capture-semantics.html)）。锁用 `using g = m.lock() { ... }` 获取，在块结束时释放；受保护的值用 `get()` / `set()` 读写。只需读取时，`m.with(|v| v)` 在一次调用期间加锁，`total` 就是这样做的（[并发](../learn/12-concurrency.html)，[3.17 `using`](../spec/refs-lang/003-expressions/317-using-expression.html)，[`Mutex`](../../std/core/sync.html#struct.Mutex)）。
- 一次转账必须锁住两个账户。如果两个线程同时执行 A→B 和 B→A，并以相反的顺序加锁，它们可能会永远互相等待，所以总是先锁编号较小的账户。
- 互相独立的计数器，例如转出的金额和被拒绝的次数，用 `Atomic<i64>` 的 `fetch_add` 无锁计数（[`Atomic`](../../std/core/sync.html#struct.Atomic)，[2.16 sync](../spec/refs-standard-libraries/200-core/216-sync.html)）。
- 每个线程从自己的带种子的生成器 `random::Xoshiro256PlusPlus::seed(..)` 取数：`below(n)` 给出 `0..n` 中的值，`range(1..301)` 给出一个金额；对同一个种子，每次运行的序列都相同（[`random`](../../std/std/random.html)）。
- 每个账户的余额取决于线程的交错方式，但只要转账是原子的，总额就总是相同的。输出中只包含不依赖这种顺序的事实。
