# 3. Keeping an API Stable

Users of a published package build against it with a version requirement such as `^0.3`, so a release in that range must not break their code. This page shows how to retire an API and how to leave room to grow.

## Retiring an API

When a function, method, field or type of your package should go away, mark it `#[deprecated]` instead of removing it. Code that uses it keeps building and gets a warning that says what to write instead; remove it in the next breaking release.

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

## Leaving room to grow

Two attributes keep room to grow. `#[non_exhaustive]` on an enum makes a `match` in other packages end with `_ =>`, so you can add variants later. On a struct, it marks a record you may add fields to: other packages still build it with a literal, leaving out fields that have defaults, but a pattern on it ends in `..`. Give every field you add later a default, so those literals keep compiling.

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

`#[sealed]` on a trait means only your package implements it. Other packages still use it in bounds and call its methods, but an `impl` of it elsewhere is an error, so you may rely on knowing every implementation.

## Further reading

- Spec: [6.2.7 `#[deprecated]`](../../spec/refs-lang/006-meta-system/602-built-in-attributes.html#627-deprecated-discouraged-use-marker), [6.2.10 `#[non_exhaustive]`](../../spec/refs-lang/006-meta-system/602-built-in-attributes.html#6210-non_exhaustive-opting-out-of-external-exhaustiveness), [6.2.15 `#[sealed]`](../../spec/refs-lang/006-meta-system/602-built-in-attributes.html#6215-sealed-impls-only-in-the-defining-package)
- Back to [Publishing Packages](index.html)
