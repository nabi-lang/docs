# 3. 保持 API 稳定

已发布的包的用户以 `^0.3` 这样的版本要求依赖它，所以这个范围内的版本不能破坏他们的代码。本页介绍如何淘汰一个 API，以及如何为以后的扩展留出余地。

## 淘汰一个 API

当包中的某个函数、方法、字段或类型应当去掉时，给它加上 `#[deprecated]`，而不是直接删除。使用它的代码照常构建，并得到一条告诉你改写成什么的警告；在下一个破坏性版本中再删除它。

```nabi norun
#[deprecated(since = "0.3.0", note = "use `wrap_width`")]
pub fn wrap(s: str) -> str { wrap_width(s, 80) }
```

```text
warning[E0131]: use of deprecated function `wrap`
  --> src/main.nabi:6:9
   |
 6 | println(textwrap::wrap("hi"));
   |         ^^^^^^^^^^^^^^^^^^^^
   |
   = note: deprecated since 0.3.0
   = note: use `wrap_width`
   = note: `#[warn(deprecated)]` on by default
```

## 为扩展留出余地

有两个属性可以为以后的扩展留出余地。枚举上的 `#[non_exhaustive]` 使其他包中的 `match` 必须以 `_ =>` 结尾，这样你以后可以添加变体。用在结构体上时，它表示一个以后可能添加字段的记录：其他包仍然可以用字面量构造它，省略有默认值的字段，但对它的模式必须以 `..` 结尾。以后添加的每个字段都要给默认值，这样那些字面量才能继续编译。

```nabi norun
#[non_exhaustive]
pub struct WrapOptions {
    pub width: i32 = 80;
    pub break_words: bool = false;
}

// in another package
let o = textwrap::WrapOptions { width: 60 };
let textwrap::WrapOptions { width, .. } = o;
```

trait 上的 `#[sealed]` 表示只有你的包实现它。其他包仍然可以在约束中使用它、调用它的方法，但在别处为它写 `impl` 是错误，因此你可以确信自己掌握了它的全部实现。

## 延伸阅读

- 规范：[6.2.7 `#[deprecated]`](../../spec/refs-lang/006-meta-system/602-built-in-attributes.html#627-deprecated-不推荐使用的标记)、[6.2.10 `#[non_exhaustive]`](../../spec/refs-lang/006-meta-system/602-built-in-attributes.html#6210-non_exhaustive-退出外部穷尽性)、[6.2.14 `#[sealed]`](../../spec/refs-lang/006-meta-system/602-built-in-attributes.html#6214-sealed-只在定义它的包中实现)
- 返回[发布包](index.html)
