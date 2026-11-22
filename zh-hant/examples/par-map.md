# 平行 map / reduce

用 `core::par` 的 `map`、`fold`、`filter` 和 `join` 把質數計數分攤到多個核心，並與循序計數的結果核對。

## 程式碼

```nabi
// Count primes in parallel with core::par and check against a sequential count.

fn is_prime(n: i64) -> bool {
    if n < 2 { return false; }
    if n % 2 == 0 { return n == 2; }
    let d = 3;
    while d * d <= n {
        if n % d == 0 { return false; }
        d += 2;
    }
    true
}

fn count_primes(lo: i64, hi: i64) -> i64 {
    let c = 0;
    for n in lo..hi {
        if is_prime(n) { c += 1; }
    }
    c
}

fn main() {
    let limit = 2000000;
    let block = 250000;

    // 1. map: one block per item; results come back in input order.
    let starts: List<i64> = [];
    let s = 0;
    while s < limit { starts.push(s); s += block; }
    let counts = par::map(starts, |lo| count_primes(lo, lo + block));
    for (i, c) in counts.enumerate() {
        let lo = starts[i];
        println(f"[{lo:>7}, {lo + block:>7})  {c:>6} primes");
    }

    // 2. fold: sum the per-block counts (the function must be associative).
    let total = par::fold(counts, 0, |a, b| a + b);
    println(f"total below {limit}: {total}");

    // 3. filter: keep the primes of a smaller range, in order.
    let near = par::filter((999900..1000100).to_list(), is_prime);
    println(f"primes around one million: {near:?}");

    // 4. join: two independent computations at once.
    let (twins, max_gap) = par::join(|| twin_primes(100000), || largest_gap(100000));
    println(f"twin prime pairs below 100000: {twins}, largest gap: {max_gap}");

    // The sequential answer must match the parallel one.
    let seq = count_primes(0, limit);
    println(f"sequential: {seq}, parallel: {total}, match: {seq == total}");
}

fn twin_primes(limit: i64) -> i64 {
    let c = 0;
    for n in 3..limit - 2 {
        if is_prime(n) && is_prime(n + 2) { c += 1; }
    }
    c
}

fn largest_gap(limit: i64) -> i64 {
    let prev = 2;
    let best = 0;
    for n in 3..limit {
        if is_prime(n) {
            if n - prev > best { best = n - prev; }
            prev = n;
        }
    }
    best
}
```

```output
[      0,  250000)   22044 primes
[ 250000,  500000)   19494 primes
[ 500000,  750000)   18700 primes
[ 750000, 1000000)   18260 primes
[1000000, 1250000)   17971 primes
[1250000, 1500000)   17686 primes
[1500000, 1750000)   17453 primes
[1750000, 2000000)   17325 primes
total below 2000000: 148933
primes around one million: [999907, 999917, 999931, 999953, 999959, 999961, 999979, 999983, 1000003, 1000033, 1000037, 1000039, 1000081, 1000099]
twin prime pairs below 100000: 1224, largest gap: 72
sequential: 148933, parallel: 148933, match: true
```

## 說明

- `par::map` 把項目切成多個區塊，在執行緒池上同時處理，但結果串列保留輸入順序，所以 `counts[i]` 就是從 `starts[i]` 開始那一段的答案（[2.19.1 核心 API](../spec/refs-standard-libraries/200-core/219-par.html#2191-核心-api)、[`par::map`](../../std/core/par.html#fn.map)）。
- `par::fold` 先摺疊每個區塊，再摺疊各區塊的結果。函式必須滿足結合律，重新分組才會得到相同的答案：加法可以，減法不行（[`par::fold`](../../std/core/par.html#fn.fold)）。`par::reduce(counts, |a, b| a + b)` 是不帶 `0` 的相同運算，串列為空時得到 `none`（[`par::reduce`](../../std/core/par.html#fn.reduce)）。
- `par::filter` 也保留順序，所以一百萬附近的質數會依遞增順序輸出。
- `par::join` 同時執行兩個互不相關的計算，並以元組回傳兩個結果（[並行](../learn/12-concurrency.html)）。
- 比對平行與循序的答案，是驗證平行化程式碼最簡單的方法。執行時間因機器而異，所以不列入輸出。

## 執行

想看到加速效果，請對最佳化建置計時。

```bash
nabic build --release par.nabi -o par
time ./par
```
