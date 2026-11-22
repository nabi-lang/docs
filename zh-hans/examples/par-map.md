# 并行 map / reduce

用 `core::par` 的 `map`、`fold`、`filter` 和 `join` 把素数计数分摊到多个核心上，并与顺序计数的结果核对。

## 代码

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

## 讲解

- `par::map` 把元素分成若干块，在线程池上同时处理，但结果列表保持输入的顺序，所以 `counts[i]` 就是从 `starts[i]` 开始的那一块的答案（[2.19.1 core API](../spec/refs-standard-libraries/200-core/219-par.html#2191-core-api)，[`par::map`](../../std/core/par.html#fn.map)）。
- `par::fold` 先对每一块折叠，再折叠各块的结果。函数必须满足结合律，这样重新分组才能得到同样的答案：加法满足，减法不满足（[`par::fold`](../../std/core/par.html#fn.fold)）。`par::reduce(counts, |a, b| a + b)` 与之相同，只是不需要 `0`，对空列表给出 `none`（[`par::reduce`](../../std/core/par.html#fn.reduce)）。
- `par::filter` 同样保持顺序，所以一百万附近的素数按升序输出。
- `par::join` 同时运行两个互不相关的计算，并以元组返回两个结果（[并发](../learn/12-concurrency.html)）。
- 比较并行和顺序的答案是检查并行化代码最简单的方法。耗时因机器而异，所以不在输出中。

## 运行

要看到加速效果，请对优化构建计时。

```bash
nabic build --release par.nabi -o par
time ./par
```
