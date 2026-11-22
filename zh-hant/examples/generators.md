# 產生器

用 `yield fn` 把費氏數列、質數串流、自己寫的 `take_while` 配接器以及遞迴的樹走訪寫成惰性迭代器。

## 程式碼

```nabi
// An endless Fibonacci stream.
yield fn fibonacci() -> i64 {
    let (a, b) = (0, 1);
    loop {
        yield a;
        (a, b) = (b, a + b);
    }
}

// Primes by trial division against the primes found so far.
yield fn primes() -> i64 {
    let found: List<i64> = [];
    let n = 2;
    loop {
        let is_prime = true;
        for p in found {
            if p * p > n { break; }
            if n % p == 0 { is_prime = false; break; }
        }
        if is_prime {
            found.push(n);
            yield n;
        }
        n += 1;
    }
}

// A generic adapter: pass items through until `pred` fails.
yield fn take_while<I: Iterator>(it: I, pred: fn(I::Item) -> bool) -> I::Item {
    for x in it {
        if !pred(x) { return; }
        yield x;
    }
}

// A nested list, walked depth-first by a recursive generator.
enum Tree {
    Leaf(str),
    Branch(List<Tree>),
}

yield fn leaves(t: Tree) -> str {
    match t {
        Tree::Leaf(name) => yield name,
        Tree::Branch(kids) => for k in kids {
            for name in leaves(k) { yield name; }
        },
    }
}

let first = fibonacci().take(12).to_list();
println(f"fib: {first:?}");

let small = take_while(primes(), |p| p < 50).to_list();
println(f"primes < 50: {small:?}");

let even_fib_sum = take_while(fibonacci(), |x| x < 4_000_000)
    .filter(|x| x % 2 == 0)
    .sum();
println(f"sum of even fib < 4M: {even_fib_sum}");

// The 1000th prime: nothing past it is ever computed.
println(f"1000th prime: {primes().nth(999) ?? -1}");

// An inline generator block.
let squares = yield {
    for i in 1..=5 { yield i * i; }
};
println(f"squares: {squares.to_list():?}");

let tree = Tree::Branch([
    Tree::Leaf("a"),
    Tree::Branch([Tree::Leaf("b"), Tree::Branch([Tree::Leaf("c")]), Tree::Leaf("d")]),
    Tree::Leaf("e"),
]);
let names = leaves(tree).to_list();
println(f"leaves: {names:?}");
```

```output
fib: [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]
primes < 50: [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]
sum of even fib < 4M: 4613732
1000th prime: 7919
squares: [1, 4, 9, 16, 25]
leaves: ["a", "b", "c", "d", "e"]
```

## 說明

- 呼叫 `yield fn` 不會立刻執行它的主體，而是回傳一個迭代器 `GenIter<T>`。`-> i64` 不是回傳值，而是產出項目的型別（[4.11.3 具名產生器函式](../spec/refs-lang/004-functions/411-generators.html#4113-具名產生器函式)）。
- `fibonacci()` 和 `primes()` 是無窮產生器。它們的區域變數（`a`、`b`、`found`）在兩次 `yield` 之間會保留下來，使用端只用 `take`、`nth` 之類的方法取出需要的項目數（[4.11.9 無窮產生器](../spec/refs-lang/004-functions/411-generators.html#4119-無窮產生器)）。
- 產生器實作了 [`Iterator`](../../std/core/iter.html#trait.Iterator)，所以能和 `filter`、`sum`、`to_list` 等配接器組合。自己動手寫的 `take_while`（標準迭代器也有一個）只是一個泛型的 `yield fn`，用 `return;` 結束迭代（[集合型別與迭代器](../learn/08-collections-iterators.html)、[泛型](../learn/07-generics.html)）。
- `yield { ... }` 是行內產生器區塊，在需要運算式的地方直接建立迭代器（[4.11.4](../spec/refs-lang/004-functions/411-generators.html#4114-行內-yield-區塊值)）。
- `leaves` 呼叫自己，並把每棵子樹的值再次產出。遞迴產生器是攤平巢狀結構的常見做法（[和型別](../learn/05-sum-types.html)）。

> [!NOTE]
> 像 `Node(Tree, i64, Tree)` 這樣直接包含自己的 enum 大小是無限的，所以編譯器會拒絕它（``recursive type `Tree` has infinite size``）。這個範例把子節點放在參考型別 `List<Tree>` 中，以表達相同的結構。
