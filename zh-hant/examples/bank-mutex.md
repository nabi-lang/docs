# 銀行轉帳

多個執行緒在以 `Mutex` 保護的帳戶之間轉移金額，用 `Atomic` 計數器收集統計，最後確認總額從未改變。

## 程式碼

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

## 說明

- 每個帳戶都是獨立的 `Mutex<i64>`，存放在固定長度陣列中。這裡用 `List` 無法通過編譯：串列是共用的可變控制代碼，不是 `Sync`，所以執行緒閉包不能擷取它（[4.6.9](../spec/refs-lang/004-functions/406-capture-semantics.html)）。鎖以 `using g = m.lock() { ... }` 取得，在區塊結束時釋放；受保護的值用 `get()` / `set()` 讀寫。只需要讀取時，`m.with(|v| v)` 只在一次呼叫期間持有鎖，`total` 就是這樣做的（[並行](../learn/12-concurrency.html)、[3.17 `using`](../spec/refs-lang/003-expressions/317-using-expression.html)、[`Mutex`](../../std/core/sync.html#struct.Mutex)）。
- 一次轉帳必須鎖住兩個帳戶。如果兩個執行緒同時執行 A→B 和 B→A，並以相反的順序上鎖，就可能永遠互相等待，所以一律先鎖編號較小的帳戶。
- 互不相關的計數器，例如轉移的金額和拒絕的次數，用 `Atomic<i64>` 的 `fetch_add` 在不上鎖的情況下計數（[`Atomic`](../../std/core/sync.html#struct.Atomic)、[2.16 sync](../spec/refs-standard-libraries/200-core/216-sync.html)）。
- 每個執行緒從自己帶種子的產生器 `random::Xoshiro256PlusPlus::seed(..)` 取值：`below(n)` 給出 `0..n` 中的值，`range(1..301)` 給出金額；種子相同時，每次執行的序列都一樣（[`random`](../../std/std/random.html)）。
- 每個帳戶的餘額取決於執行緒交錯執行的方式，但只要轉帳是原子的，總額就永遠相同。輸出只包含不受這個順序影響的事實。
