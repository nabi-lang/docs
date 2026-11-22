# 呼叫 C 函式庫

在 `extern "C"` 區塊中宣告 libc 的 `strlen`、`getpid`、`getenv`、`snprintf` 和 `qsort`，並用 `bare` 的指標與配置函式直接管理 C 的記憶體。

## 程式碼

```nabi
use bare::alloc::{Layout, LayoutError, alloc, dealloc};
use bare::ptr::{CPtr, ConstPtr, ManyPtr, Ptr};

extern "C" {
    fn strlen(s: str) -> usize;
    fn getpid() -> i32;
    fn setenv(name: str, value: str, overwrite: i32) -> i32;
    fn getenv(name: str) -> CPtr<c_char>;   // NULL when unset
    fn snprintf(buf: Ptr<u8>, size: usize, fmt: str, ...) -> i32;
    fn qsort(base: Ptr<u8>, n: usize, size: usize, cmp: extern "C" fn(ConstPtr<i32>, ConstPtr<i32>) -> i32);
}

// A C string (NUL-terminated chars) copied into a nabi `str`.
unsafe fn from_c(p: ConstPtr<c_char>) -> str {
    let bytes: List<u8> = [];
    let i = 0;
    loop {
        let b = p.add(i).read();
        if b == c'\0' { break; }
        bytes.push(b as u8);
        i += 1;
    }
    try str::from_utf8(bytes) catch _ { "<invalid utf-8>" }
}

// getenv with the NULL case turned into `none`.
fn c_getenv(name: str) -> str | none {
    let p = unsafe { getenv(name) }.as_const()?;
    unsafe { from_c(p) }
}

// The comparator qsort calls back: a nabi function with the C ABI.
extern "C" fn descending(a: ConstPtr<i32>, b: ConstPtr<i32>) -> i32 {
    let (x, y) = unsafe { (a.read(), b.read()) };
    if x < y { 1 } else if x > y { -1 } else { 0 }
}

fn main() throws (LayoutError) {
    // str arguments arrive in C as NUL-terminated char*.
    println(f"strlen(\"hello, nabi\") = {unsafe { strlen("hello, nabi") }}");
    println(f"strlen(\"naïve\") = {unsafe { strlen("naïve") }} bytes");

    let pid = unsafe { getpid() };
    println(f"getpid() > 0: {pid > 0}, matches process::id(): {pid as u32 == std::process::id()}");

    unsafe { setenv("NABI_FFI_DEMO", "set from C", 1) };
    println(f"NABI_FFI_DEMO = {c_getenv("NABI_FFI_DEMO") ?? "(unset)"}");
    println(f"NABI_FFI_MISSING = {c_getenv("NABI_FFI_MISSING") ?? "(unset)"}");

    // A variadic call writing into a buffer we allocate.
    let text_layout = try Layout::array::<u8>(64);
    let buf: Ptr<u8> = unsafe { alloc(text_layout) }.as_ptr() ?? panic("out of memory");
    let n = unsafe { snprintf(buf, 64, "%-6s|%5d|%08.3f", "pi", 42, 3.14159) };
    println(f"snprintf wrote {n} bytes: {unsafe { from_c(buf.cast::<c_char>().as_const()) }}");
    unsafe { dealloc(buf, text_layout) };

    // qsort over a C array of i32.
    let values: List<i32> = [42, 7, 19, 88, 3, 56, 21];
    let layout = try Layout::array::<i32>(values.len());
    let base: Ptr<u8> = unsafe { alloc(layout) }.as_ptr() ?? panic("out of memory");
    let arr: ManyPtr<i32> = base.cast::<i32>().as_many();
    for (i, v) in values.enumerate() { unsafe { arr.write(i, v) }; }
    unsafe { qsort(base, values.len(), 4, descending) };
    let sorted: List<i32> = [];
    for i in 0..values.len() { sorted.push(unsafe { arr.read(i) }); }
    println(f"qsort descending: {sorted:?}");
    unsafe { dealloc(base, layout) };
}
```

```output
strlen("hello, nabi") = 11
strlen("naïve") = 6 bytes
getpid() > 0: true, matches process::id(): true
NABI_FFI_DEMO = set from C
NABI_FFI_MISSING = (unset)
snprintf wrote 21 bytes: pi    |   42|0003.142
qsort descending: [88, 56, 42, 21, 19, 7, 3]
```

## 說明

- `extern "C" { ... }` 區塊只包含沒有主體的簽章。在其中宣告的函式隱含為 `unsafe`，所以每次呼叫都要寫在 `unsafe { ... }` 中（[5.9 `extern` 區塊](../spec/refs-lang/005-modules/509-extern-block.html)、[4.15 unsafe](../spec/refs-lang/004-functions/415-unsafe.html)）。
- `str` 參數以 NUL 結尾的 `char*` 傳給 C。所以 `strlen("naïve")` 回傳的是 UTF-8 位元組數 6，而不是 5 個字元（[5.9.4 FFI 相容型別](../spec/refs-lang/005-modules/509-extern-block.html#594-ffi-相容型別)）。
- 可能為 NULL 的 C 指標以 [`CPtr`](../../std/bare/ptr.html#struct.CPtr) 接收。`as_const()` 回傳非 null 的 [`ConstPtr`](../../std/bare/ptr.html#struct.ConstPtr) 或 `none`，所以只要一個 `?`，就能把 `getenv` 包成回傳 `str | none` 的安全函式（[和型別](../learn/05-sum-types.html)）。
- 記憶體以 [`Layout`](../../std/bare/alloc.html#struct.Layout) 和 `alloc` / `dealloc` 手動配置，並透過 [`ManyPtr`](../../std/bare/ptr.html#struct.ManyPtr) 的 `read(i)` / `write(i, v)` 像 C 陣列一樣存取。GC 不管理這塊記憶體，所以要自己釋放。
- `extern "C" fn descending` 是以 C ABI 編譯的 Nabi 函式。把它當作函式指標傳給 `qsort`，C 就能在排序時回呼 Nabi 程式碼（[5.9.10 定義 `extern fn`](../spec/refs-lang/005-modules/509-extern-block.html#5910-extern-fn-定義對外公開)）。
- `snprintf` 這類可變參數函式，在簽章結尾以 `...` 宣告。詳情請見 [FFI 章節](../learn/14-ffi.html)。
