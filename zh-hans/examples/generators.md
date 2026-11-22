# 生成器

用 `yield fn` 把斐波那契数列、素数流、手写的 `take_while` 适配器和递归的树遍历写成惰性迭代器。

## 代码

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

## 讲解

- 调用 `yield fn` 不会立即运行它的函数体，而是返回一个迭代器 `GenIter<T>`。`-> i64` 不是返回值，而是产出的元素的类型（[4.11.3 具名生成器函数](../spec/refs-lang/004-functions/411-generators.html#4113-named-generator-functions)）。
- `fibonacci()` 和 `primes()` 是无限生成器。它们的局部变量（`a`、`b`、`found`）在两次 `yield` 之间保持不变，使用方用 `take`、`nth` 等只取出需要的元素个数（[4.11.9 无限生成器](../spec/refs-lang/004-functions/411-generators.html#4119-infinite-generators)）。
- 生成器实现了 [`Iterator`](../../std/core/iter.html#trait.Iterator)，所以可以与 `filter`、`sum`、`to_list` 等适配器组合。手写的 `take_while`（标准迭代器也有一个）是单个泛型 `yield fn`，用 `return;` 结束迭代（[集合与迭代器](../learn/08-collections-iterators.html)，[泛型](../learn/07-generics.html)）。
- `yield { ... }` 是内联生成器块，在需要表达式的地方直接构造一个迭代器（[4.11.4](../spec/refs-lang/004-functions/411-generators.html#4114-inline-yield-blocks-values)）。
- `leaves` 调用自身，并重新产出每棵子树的值。递归生成器是展平嵌套结构的常用方法（[和类型](../learn/05-sum-types.html)）。

> [!NOTE]
> 直接包含自身的枚举，例如 `Node(Tree, i64, Tree)`，大小是无限的，所以编译器会拒绝它（``recursive type `Tree` has infinite size``）。这个示例把子节点放在引用类型 `List<Tree>` 中，以表达同样的结构。
