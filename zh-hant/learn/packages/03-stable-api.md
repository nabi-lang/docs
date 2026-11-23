# 3. 保持 API 穩定

已發布套件的使用者會以 `^0.3` 這類版本需求依賴它，所以這個範圍內的版本不能破壞他們的程式碼。本頁說明如何淘汰一個 API，以及如何為日後的擴充保留空間。

## 淘汰 API

當套件中的某個函式、方法、欄位或型別應該移除時，請為它加上 `#[deprecated]`，而不是直接刪掉。使用它的程式碼仍能照常建置，並會收到一則說明該改寫成什麼的警告；等到下一個破壞性版本再移除它。

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

## 為擴充保留空間

有兩個屬性可以為日後的擴充保留空間。列舉上的 `#[non_exhaustive]` 會讓其他套件中的 `match` 必須以 `_ =>` 結尾，這樣你之後就能加入變體。用在結構上時，它表示這是一筆之後可能加入欄位的紀錄：其他套件仍可用字面值建構它，並省略有預設值的欄位，但對它的模式必須以 `..` 結尾。之後加入的每個欄位都要給預設值，那些字面值才能繼續編譯。

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

trait 上的 `#[sealed]` 表示只有你的套件會實作它。其他套件仍可在約束中使用它、呼叫它的方法，但在別處為它寫 `impl` 是錯誤，所以你可以放心依賴自己知道它的所有實作。

## 延伸閱讀

- 規格：[6.2.7 `#[deprecated]`](../../spec/refs-lang/006-meta-system/602-built-in-attributes.html#627-deprecated-不建議使用的標記)、[6.2.10 `#[non_exhaustive]`](../../spec/refs-lang/006-meta-system/602-built-in-attributes.html#6210-non_exhaustive-退出外部的窮盡性)、[6.2.15 `#[sealed]`](../../spec/refs-lang/006-meta-system/602-built-in-attributes.html#6215-sealed-只在定義套件中實作)
- 回到[發布套件](index.html)
