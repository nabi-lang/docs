# 14. FFI 基礎

本章說明如何從 Nabi 呼叫以 C 撰寫的函式、如何把 Nabi 函式交給 C，以及如何用 Nabi 自己的 C 編譯器編譯 C 程式碼。作業系統和無數函式庫都透過 C ABI 提供功能，所以懂得 FFI（外部函式介面，foreign function interface），就能使用標準函式庫沒有提供的功能。

所有範例都以 macOS / Linux 上的 libc 執行。Windows 上的差異集中在本章最後。

## `extern "C"` 區塊

要使用 C 函式，先在 `extern "C" { ... }` 區塊中宣告它的簽章。宣告沒有主體，以 `;` 結尾。名稱就是 C 的符號名稱，原樣照寫。

```nabi
extern "C" {
    fn strlen(s: str) -> c_size_t;
    fn abs(x: c_int) -> c_int;
    fn labs(x: c_long) -> c_long;
    fn getpid() -> c_int;
}

println(f"strlen: {unsafe { strlen("hello, C") }}");
println(f"abs: {unsafe { abs(-42) }}");
println(f"labs: {unsafe { labs(-7) }}");
println(f"pid positive: {unsafe { getpid() } > 0}");
```

```output
strlen: 8
abs: 42
labs: 7
pid positive: true
```

C 的型別用它們的 C 名稱來寫：`c_int`、`c_uint`、`c_long`、`c_ulong`、`c_size_t`、`c_double` 等都在預載模組中。每一個都是對目標而言大小正確的 Nabi 數值型別（`c_int` 是 `i32`，`c_size_t` 是 `usize`），所以值在兩者之間傳遞時不需要轉換。請使用它們，而不是固定大小的型別：C 的 `long` 在 Linux 和 macOS 上是 64 位元，在 Windows 上卻是 32 位元，所以 `fn labs(x: i64) -> i64` 在一邊正確、在另一邊錯誤，而 `fn labs(x: c_long) -> c_long` 在每個平台上都正確。`void*` 是 `CPtr<c_void>`。

`_Bool` 是 `bool`；`_Float16` 和 `__bf16` 是 `f16` 和 `bf16`，而 8 或 16 位元組的向量（`float32x4_t`、`__m128`）在 arm64 和 x86-64 上是 `Simd`。`str` 參數會以 NUL 結尾的 `char*` 傳遞；必要時 Nabi 會製作一份以 NUL 結尾的複本。

只有 C 能容納的型別才能跨越邊界，而編譯器會逐一檢查，連別名和結構欄位也不例外：元組、Unicode `char`、一般列舉或 `List` 都是錯誤，錯誤訊息會告訴你改寫成什麼（`struct`、`u32`、`#[repr(i32)]`、指標加上長度）。

`extern` 區塊只能出現在模組的最上層，而且必須寫出 ABI 字串（`"C"`）。

## 為什麼需要 `unsafe`

編譯器沒有辦法檢查 C 函式的行為是否和它的簽章相符。傳入錯誤的指標或寫錯宣告，都可能破壞記憶體。因此 `extern "C"` 區塊中的每個函式都隱含為 `unsafe`，只能在 `unsafe { ... }` 中呼叫。

```text check=checks/14-unsafe-call.nabi
error[E0061]: call to unsafe function `abs` outside an `unsafe` block or `unsafe fn`
  --> main.nabi:4:12
   |
 4 | println(f"{abs(-1)}");
   |            ^^^^^^^
   |
   = help: wrap the call in `unsafe { .. }`; every `extern` function, `extern "nabi"` included, is unsafe to call
```

在 `extern` 區塊中宣告的 C 全域變數（`static mut optind: c_int;`）同樣不受檢查，所以讀寫它也需要 `unsafe`：

```nabi
extern "C" {
    static mut optind: c_int;
}

let next = unsafe {
    optind = 1;
    optind + 1
};
println(f"next: {next}");
```

在 `unsafe` 之外使用它是錯誤，說明會給出修正方式（`unsafe { optind = 1; }`）。C 的 `errno` 不是這種全域變數（它是每個執行緒各自的巨集）；[來自 C 的錯誤](#errors-from-c-errno)說明如何讀取它。

`unsafe` 區塊的意思是「我保證這裡是安全的」。好的做法是讓這些區塊保持小巧，並把危險的呼叫包裝在安全的 Nabi 函式中，讓外部只看得到安全的 API。

## 字串

### C 字元與 `CStr`

C 的 `char` 是一個位元組；在 Nabi 中它是 `c_char` 型別，字面值寫作 `c'A'`。它和 Unicode `char`（`'A'`）是不同的型別，所以碰觸 FFI 邊界的程式碼會一眼可見。

```nabi
extern "C" {
    fn toupper(c: c_int) -> c_int;
}

let c: c_char = c'a';
let up = unsafe { toupper(c as c_int) };
println(f"toupper: {up} = {up as u8 as char}");
```

```output
toupper: 65 = A
```

如果 C 函式可能回傳 NULL，就把回傳型別宣告為 `ConstPtr<T> | none`：一個位址，以 `none` 表示 NULL。（`CPtr<T>` 則像 C 指標一樣，把 NULL 當作一個值來保存。）C 字串就是這樣回來的：一個 `char*`，其位元組屬於 C。`strerror` 的緩衝區在下一次呼叫時會改變，`getenv` 的則在 `setenv` 之後改變。`CStr::from_ptr` 複製到 NUL 為止的位元組，`to_str` 檢查它們是否為 UTF-8。危險的部分侷限在 `from_c` 中，對外只公開安全的 `env_var`。

```nabi
use bare::ffi::CStr;
use bare::ptr::ConstPtr;

extern "C" {
    fn getenv(name: str) -> ConstPtr<c_char> | none;
    fn setenv(name: str, value: str, overwrite: c_int) -> c_int;
    fn strerror(code: c_int) -> ConstPtr<c_char> | none;
}

// Copy a C string into a nabi str: `none` for NULL or bytes that are not UTF-8.
unsafe fn from_c(p: ConstPtr<c_char> | none) -> str | none {
    let c = CStr::from_ptr(p?)?;
    try? c.to_str()
}

fn env_var(name: str) -> str | none {
    unsafe { from_c(getenv(name)) }
}

unsafe { setenv("NABI_FFI_DEMO", "hello", 1) };
println(env_var("NABI_FFI_DEMO") ?? "(unset)");
println(env_var("NABI_FFI_MISSING") ?? "(unset)");
println(unsafe { from_c(strerror(2)) } ?? "?");
```

```output
hello
(unset)
No such file or directory
```

回傳型別寫成 `str` 是錯誤，說明會指向這裡：Nabi 無法得知 C 會保留這些位元組多久、呼叫端是否必須釋放它們，或它們是否為 UTF-8。要傳遞由位元組建立的字串，請建立 `CString`（`try CString::new(bytes)`，內含 NUL 時會拒絕），並在 `cs` 仍在作用域內時傳入 `cs.as_ptr()`。

### 寬字串：`WideCString`

C 的 `wchar_t` 函式，以及 Windows 的 `W` API，接受由 `c_wchar` 單位組成的字串。`WideCString::new(s)` 從 `str` 建立一個（包含 NUL）：在 Windows 上是 UTF-16，在其他平台上則是每個字元一個單位。`as_ptr()` 是要傳入的 `const wchar_t*`，`to_str()` 則把字串取回。`WideCStr::from_ptr(p)` 複製 C 回傳的寬字串，和 `CStr::from_ptr` 一樣。

```nabi
use bare::ffi::WideCString;
use bare::ptr::ConstPtr;

extern "C" {
    fn wcslen(s: ConstPtr<c_wchar>) -> c_size_t;
}

let w = try! WideCString::new("h\u{E9}llo");
println(f"{w.len()} {unsafe { wcslen(w.as_ptr()) }} {w.to_str()}");
```

```output
5 5 héllo
```

## 緩衝區

### 傳遞串列：`gc::pin`

填寫或讀取緩衝區的 C 函式接受一個指標和一個長度。`List` 位在 GC 堆積上，回收器可能會移動它。每次 C 呼叫，不論多短，都可能讓一次回收執行，所以在 C 使用串列的位址期間，必須把串列固定（pin）住：`gc::pin(xs)` 會在 `Pinned` 值仍在作用域內時讓串列留在原處，而 `as_data_ptr()` 是它第一個元素的位址。

```nabi
use bare::gc::pin;
use bare::ptr::Ptr;

extern "C" {
    fn memset(p: Ptr<u8>, c: c_int, n: c_size_t) -> Ptr<u8>;
}

let buf: List<u8> = [0, 0, 0, 0];
let p = pin(buf);
unsafe { memset(p.as_data_ptr(), 7, buf.len()) };
println(f"{buf:?}");
```

```output
[7, 7, 7, 7]
```

編譯器絕不會替你加上固定。簽章裡直接寫 `List` 是錯誤，說明會給出要寫的宣告和呼叫：`ptr: ConstPtr<T>, len: usize`，然後從 `let p = gc::pin(xs);` 取得 `p.as_data_ptr()` 和 `xs.len()`。`str` 引數不需要固定：Nabi 會在呼叫期間讓它留在原處。

串列的切片也以同樣方式固定（`pin(xs[1:3])`，其 `as_data_ptr()` 是切片的第一個元素），`StrBuf` 也一樣（`as_bytes_ptr()` 是它的第一個位元組）：

```nabi
use bare::gc::pin;
use bare::ptr::Ptr;

extern "C" {
    fn strlen(p: Ptr<u8>) -> c_size_t;
}

let xs: List<i64> = [1, 2, 3, 4];
let s = pin(xs[1:3]);
let b = StrBuf::with_capacity(16);
b.push_str("hello");
b.push('\u{0}');
let pb = pin(b);
println(f"{unsafe { s.as_data_ptr().read() }} {unsafe { strlen(pb.as_bytes_ptr()) }}");
```

```output
2 5
```

被固定的串列無法增長：增長會把元素移到新的緩衝區，而 C 可能還握著舊的位址。請在固定之前先預留空間，並在固定期間填入：

```nabi
use bare::gc::pin;

let ys: List<i64> = [1];
ys.reserve(7);
let py = pin(ys);
for i in 0..7 { ys.push(i); }
println(f"{py.obj().len()}");
```

```output
8
```

這裡如果再多一次 `ys.push(9)`，就需要更大的緩衝區，所以會觸發 panic 並說明該怎麼做：

```text
cannot grow a list while it is pinned (C may hold its address): call `xs.reserve(n)` before `gc::pin(xs)`
```

`StrBuf` 也遵循同樣的規則。

### C 擁有的記憶體：`bare::ptr` 與 `bare::alloc`

原始指標來自 `bare` 層：

| 型別 | 對應的 C | null |
|------|--------|------|
| `Ptr<T>`      | `T*`        | 否（null 以 `none` 表示） |
| `ConstPtr<T>` | `const T*`  | 否 |
| `CPtr<T>`     | `T*`        | 是 |

基本操作是透過指標進行的 `read()` / `write()`、`add(n)`（前進 n 個元素）和 `cast::<U>()`。來自 C 堆積的記憶體由 `bare::alloc` 取得，並由你自己歸還。GC 不管理這塊記憶體，所以忘了 `dealloc` 就會洩漏。

```nabi
use bare::mem::size_of;
use bare::ptr::{Ptr, ConstPtr};
use bare::alloc;

let xs: List<i32> = [10, 20, 30];
let p = unsafe { alloc::alloc_array::<i32>(xs.len()) }.as_ptr() ?? panic("out of memory");
for i in 0..xs.len() { unsafe { p.add(i).write(xs[i] * 2) }; }
let second = unsafe { p.add(1).read() };
println(f"second: {second}, i32 is {size_of::<i32>()} bytes");
unsafe { alloc::dealloc_array(p, xs.len()) };
```

```output
second: 40, i32 is 4 bytes
```

和 C 交換結構時，加上 `#[repr(C)]` 表明配置採用 C 的方式（預設配置本來就是如此）。`union` 也以同樣方式跨越邊界：C 依成員傳遞它，Nabi 也是。

```nabi
#[repr(C)]
struct DivT { quot: c_int; rem: c_int; }

extern "C" {
    fn div(num: c_int, den: c_int) -> DivT;
}

let r = unsafe { div(17, 5) };
println(f"17 / 5 = {r.quot} rem {r.rem}");
```

```output
17 / 5 = 3 rem 2
```

## 編譯器無法檢查的承諾：`unsafe trait`

有些 trait 對每個實作它的型別做出承諾，而編譯器沒有辦法檢查。這樣的 trait 宣告為 `unsafe trait`，它的每個 impl 都寫成 `unsafe impl`：這個 `unsafe` 是作者擔保承諾成立，就像 `unsafe { }` 區塊之於呼叫。使用這個 trait（作為約束、呼叫它）不需要 `unsafe`。

標準函式庫中的這種 trait 是 `bare::mem::Pod`，即「plain old data」：該型別的每一種位元樣式都是一個值。整數、浮點數和 `c_char` 本來就是 `Pod`。`reinterpret` 把一個 `Pod` 值的位元當成另一個同樣大小的值。由 `Pod` 欄位組成的 `#[repr(C)]` 結構，可以用 `#[derive(Pod)]` 選用，或手動寫 `unsafe impl Pod for T {}`；不論哪一種，編譯器仍會檢查配置、欄位，以及沒有 `Drop`。

```nabi
use bare::mem::{Pod, reinterpret};

/// Every all-zero bit pattern is a value.
unsafe trait Zeroable {
    fn zeroed() -> Self;
}

unsafe impl Zeroable for i32 {
    fn zeroed() -> i32 { 0 }
}

fn zero<T: Zeroable>() -> T { T::zeroed() }   // using the trait needs no `unsafe`

#[derive(Pod)]
#[repr(C)]
struct Rgba { r: u8; g: u8; b: u8; a: u8; }

#[repr(C)]
struct Pair { lo: u16; hi: u16; }

unsafe impl Pod for Pair {}                    // checked like the derive

let z: i32 = zero();
let bytes: [u8; 4] = reinterpret(Rgba { r: 1, g: 2, b: 3, a: 255 });
let one: u32 = reinterpret(1.0f32);
let p: Pair = reinterpret(0x0009_0007u32);
println(f"{z} {bytes[3]} {one:x} {p.lo + p.hi}");
```

```output
0 255 3f800000 16
```

對 `unsafe trait` 寫一般的 `impl` 是錯誤，錯誤會給出應寫的形式（[E0078](../errors/E0078.html)）：

```text check=checks/14-unsafe-impl.nabi
error[E0078]: `Zeroable` is an `unsafe trait`: the impl promises what the compiler cannot check; write `unsafe impl Zeroable for i32`
  --> main.nabi:5:1
   |
 5 | impl Zeroable for i32 {
   | ^^^^
```

## 回呼：`extern "C" fn`

對於接受函式指標的 C 函式，例如 `qsort`，把 Nabi 函式以 C ABI 編譯後傳入即可。在函式定義前加上 `extern "C"`。它的參數和回傳型別必須與 FFI 相容（整數、浮點數、指標、`#[repr(C)]` 結構）。

```nabi
use bare::gc::pin;
use bare::mem::size_of;
use bare::ptr::Ptr;

extern "C" {
    fn qsort(base: Ptr<i32>, n: c_size_t, size: c_size_t, cmp: extern "C" fn(Ptr<i32>, Ptr<i32>) -> c_int);
}

extern "C" fn by_value(a: Ptr<i32>, b: Ptr<i32>) -> c_int {
    let (x, y) = unsafe { (a.read(), b.read()) };
    if x < y { -1 } else if x > y { 1 } else { 0 }
}

let xs: List<i32> = [42, 7, 19, 3, 25];
let p = pin(xs);
unsafe { qsort(p.as_data_ptr(), xs.len(), size_of::<i32>(), by_value) };
println(f"{xs:?}");
```

```output
[3, 7, 19, 25, 42]
```

在這裡 `str` 不是 FFI 型別，所以它不能作為 `extern "C" fn` 的參數或回傳值。編譯器會告訴你改為接受 `ConstPtr<c_char>`，並用 `CStr::from_ptr` 複製。

C 程式碼也可能從它自己啟動的執行緒呼叫 `extern "C" fn`，執行緒池或音訊函式庫就是這樣。Nabi 會在這種執行緒第一次呼叫時將它接上，所以回呼可以配置記憶體、使用任何 Nabi 程式碼；C 那一端不需要做任何設定。

`extern "C" fn` 不是閉包：它無法擷取周圍的變數，而在 C 接受函式的地方傳入閉包是錯誤。錯誤的說明會指向下一節。

## 控制代碼：由 C 保管的 Nabi 值

會回呼的 C API 通常在函式指標旁邊接受一個 `void* user_data`，並在每次呼叫時把它交回。`Handle::new(value)`（`bare::ffi`）把 Nabi 值變成這樣的指標：`h.as_ptr()` 是要傳入的 `void*`，回呼則用 `Handle::<T>::from_ptr(p).get()` 取回值。`List` 或 `Dict` 取回時是同一個物件，所以回呼可以填入內容。控制代碼會讓值保持存活並留在原處，直到 `h.release()`。

閉包也是值。要從 C 呼叫閉包，就把它存放在控制代碼中（以 `fn` 型別儲存），並傳入一個只有一行的 `extern "C" fn`，由它讀回閉包並呼叫。

下面的 `each` 代表一個 C 函式庫，它會帶著拿到的指標進行回呼：

```nabi
use bare::ffi::Handle;
use bare::ptr::CPtr;

// void each(int n, void (*cb)(void *, int), void *user_data);
extern "C" fn each(n: c_int, cb: extern "C" fn(CPtr<c_void>, c_int), ud: CPtr<c_void>) {
    for i in 0..n {
        unsafe { cb(ud, i) };
    }
}

extern "C" fn on_row(ud: CPtr<c_void>, i: c_int) {
    let rows = unsafe { Handle::<List<str>>::from_ptr(ud) }.get();
    rows.push(f"row {i}");
}

extern "C" fn call(ud: CPtr<c_void>, i: c_int) {
    let f = unsafe { Handle::<fn(c_int)>::from_ptr(ud) }.get();
    f(i);
}

let rows: List<str> = [];
let h = Handle::new(rows);
unsafe { each(3, on_row, h.as_ptr()) };
h.release();
println(f"{rows.len()} {rows[2]}");

let total = Cell::new(0);
let f = Handle::<fn(c_int)>::new(|x: c_int| {
    let t = total.get();
    total.set(t + x);
});
unsafe { each(5, call, f.as_ptr()) };
f.release();
println(f"{total.get()}");
```

```output
3 row 2
10
```

這個指標是控制代碼編號，不是位址：C 只負責把它傳下去。`Handle` 的大小和指標相同，而且是 `Copy`，所以可以作為 `#[repr(C)]` 結構的欄位。C 用完指標後呼叫 `release`；如果 C 保留了第二份複本，`retain` 會多加一次 `release`。用錯時會觸發 panic，並告訴你該寫什麼：在最後一次 `release` 之後呼叫 `get`，會顯示 `handle already released ...`；以和建立控制代碼時不同的型別呼叫 `from_ptr`，會顯示 `` this handle holds `List<str>`, not `Dict<str, i64>`: write `Handle::<List<str>>::from_ptr(p)` ``。

### 關閉 C 開啟的資源：`Drop`

GC 會釋放 Nabi 的記憶體，但不執行任何終結器（finalizer），所以 C 交出來的資源（`FILE*`、以 `malloc` 配置的區塊、函式庫的控制代碼）要由實作了 `Drop` 的包裝型別來釋放。編譯器會在值的擁有者離開作用域時，或在繫結它的 `using` 區塊結束時呼叫 `drop`。

```nabi
use bare::ptr::CPtr;

extern "C" {
    fn malloc(size: c_size_t) -> CPtr<c_void>;
    fn free(p: CPtr<c_void>);
}

struct CBuffer { raw: CPtr<c_void>; }

impl Drop for CBuffer {
    fn drop(ref self) {
        unsafe { free(self.raw) };
        println("freed");
    }
}

fn buffer(n: usize) -> CBuffer {
    let raw = unsafe { malloc(n) };
    if raw.is_null() { panic("out of memory"); }
    CBuffer { raw }
}

using b = buffer(64) {
    println(f"null: {b.raw.is_null()}");
}
```

```output
null: false
freed
```

## 來自 C 的錯誤：`#[errno]`

許多 C 函式透過 `errno` 回報失敗。在 `extern` 區塊（或單一宣告）上加上 `#[errno]`：之後每次呼叫一回傳，就會在其他東西改動它之前立刻儲存 errno，而 `bare::ffi::errno()` 讀取儲存的值。`io::IoError::last_os_error()` 會把它轉成帶有作業系統錯誤種類和訊息的 `IoError`，讓包裝函式可以 `throw` 它。

```nabi
use bare::ffi::{errno, set_errno};

#[errno]
extern "C" {
    fn open(path: str, flags: c_int, ...) -> c_int;
}

let fd = unsafe { open("/nabi-no-such-dir/file", 0) };
let e = io::IoError::last_os_error();
println(f"{fd} {errno()} {e.kind()}");
set_errno(0);
println(f"{errno()}");
```

```output
-1 2 not found
0
```

沒有 `#[errno]` 就什麼都不會儲存，`errno()` 保留的仍是先前某次呼叫的值。在呼叫前執行 `set_errno(0)`，是給只透過 errno 回報錯誤的 C 函式用的，例如 `strtol`。

## 回呼中的 panic

panic 不可以穿過 C 的堆疊框架，離開 C 所呼叫的函式：C 對 panic 一無所知，而結束程式會在損壞的狀態上執行主程式的清理。因此，C 所呼叫的 `extern "C" fn` 中發生 panic 時，會印出報告，再多印一行，然後中止行程（在 Unix 上結束狀態為 134），而不是以 101 結束：

```nabi norun
use bare::gc::pin;
use bare::ptr::Ptr;

extern "C" {
    fn qsort(base: Ptr<i32>, n: c_size_t, size: c_size_t, cmp: extern "C" fn(Ptr<i32>, Ptr<i32>) -> c_int);
}

extern "C" fn cmp(a: Ptr<i32>, b: Ptr<i32>) -> c_int {
    panic("cannot compare")
}

let xs: List<i32> = [5, 1, 4];
let p = pin(xs);
unsafe { qsort(p.as_data_ptr(), 3, 4, cmp) };
```

```text
thread 'main' panicked at main.nabi:9:5:
cannot compare
panic in a function C called: aborting (`#[on_panic(return = v)]` on the `extern "C" fn` returns `v` to C instead)
```

若要繼續執行，就指定函式在 panic 時要回傳給 C 什麼：`#[on_panic(return = v)]`，其中 `v` 是一個數字、`true` / `false`，或代表 null 指標的 `0`（沒有回傳值的函式只寫 `#[on_panic]`）。panic 仍會被回報，然後 C 得到 `v`。`std::panic::set_hook` 會在每次 panic 時先執行你的函式，用來記錄它：

```nabi
use bare::gc::pin;
use bare::mem::size_of;
use bare::ptr::Ptr;

extern "C" {
    fn qsort(base: Ptr<i32>, n: c_size_t, size: c_size_t, cmp: extern "C" fn(Ptr<i32>, Ptr<i32>) -> c_int);
}

#[on_panic(return = 0)]
extern "C" fn by_value(a: Ptr<i32>, b: Ptr<i32>) -> c_int {
    let (x, y) = unsafe { (a.read(), b.read()) };
    if x == 13 || y == 13 { panic("unlucky number") }
    if x < y { -1 } else if x > y { 1 } else { 0 }
}

std::panic::set_hook(|info| { eprintln(f"logged: {info.message}"); });
let xs: List<i32> = [5, 13, 1];
let p = pin(xs);
unsafe { qsort(p.as_data_ptr(), xs.len(), size_of::<i32>(), by_value) };
println(f"still running, {xs.len()} items");
```

```output
still running, 3 items
```

panic 發生處到該函式之間的程式碼會被略過，其中的 `Drop` 也不會執行：它們原本要釋放的東西會洩漏，在那裡鎖上的 `Mutex` 也會一直鎖著。持有鎖時可能失敗的回呼，應該回傳錯誤值，而不是觸發 panic。在 wasm32 上，回呼中的 panic 一律中止。

## 在執行時載入函式庫：`Library`

`#[link]`（見下文）在程式連結時繫結函式庫。若要在程式執行時才選擇函式庫，就像外掛系統那樣，可以用 `bare::ffi::Library::open(path)` 載入，再用 `symbol::<F>(name)` 以 C 函式指標型別 `F` 查找函式。檔案或符號不存在時，兩者都會帶著系統提供的原因擲回錯誤。

```nabi
use bare::ffi::Library;

#[cfg(target_os = "linux")]
const LIBM: str = "libm.so.6";
#[cfg(target_os = "macos")]
const LIBM: str = "/usr/lib/libSystem.B.dylib";

fn main() throws (bare::ffi::LoadError | bare::ffi::SymbolError) {
    let lib = try Library::open(LIBM);
    let cos = try unsafe { lib.symbol::<extern "C" fn(f64) -> f64>("cos") };
    println(f"cos(0) = {unsafe { cos(0.0) }}");
    try unsafe { lib.close() };
}
```

```output
cos(0) = 1
```

`symbol` 是 `unsafe`，因為 `F` 必須是函式真正的簽章；不是 C 函式指標的型別會觸發 panic。函式庫在你 `close` 之前都會保持載入，而 `close` 也是 `unsafe`：之後從它取得的每個函式指標都會成為懸置指標。`Library::this()` 是程式本身，包括它所連結的 C 函式庫。

## 匯出給 C：`#[export]`

要讓 C 能以名稱找到 Nabi 函式，就把它定義為 `pub extern "C" fn`，並加上 `#[export]`：它的符號就是它自己的名稱。`#[export_name("sym")]` 則以另一個名稱匯出。下面的範例在同一個程式中把這兩個符號再宣告為 C 函式，然後呼叫它們。

```nabi
#[export]
pub extern "C" fn nabi_add(a: c_int, b: c_int) -> c_int { a + b }

#[export_name("nabi_mul")]
pub extern "C" fn mul(a: c_int, b: c_int) -> c_int { a * b }

extern "C" {
    #[link_name = "nabi_add"]
    fn add_c(a: c_int, b: c_int) -> c_int;
    fn nabi_mul(a: c_int, b: c_int) -> c_int;
}

println(f"{unsafe { add_c(2, 3) }} {unsafe { nabi_mul(4, 5) }}");
println(f"{nabi_add(4, 5)}");
```

```output
5 20
9
```

沒有 `extern "C"` 的匯出函式會使用 Nabi 自己的呼叫慣例，而 C 並不使用它，所以這是錯誤，並告訴你該寫什麼：`` `compute` is exported to C but uses the Nabi ABI: write `pub extern "C" fn compute(..)` ``。

### 給 C 用的共用函式庫：`--crate-type=cdylib`

另一個方向：C 程式碼，或透過 `ctypes` 的 Python，載入以 Nabi 撰寫的函式庫。把它要呼叫的函式以 `#[export] pub extern "C" fn` 的形式放在套件的 `src/lib.nabi` 中，並以 `--crate-type=cdylib` 建置：

```nabi
#[export]
pub extern "C" fn geo_sum_squares(n: c_int) -> c_int {
    let xs: List<c_int> = [];
    for i in 0..n { xs.push(i * i); }
    let s: c_int = 0;
    for x in xs { s += x; }
    s
}
```

```text
$ nabi build --crate-type=cdylib
$ python3 -c 'import ctypes; g = ctypes.CDLL("target/debug/libgeo.so"); print(g.geo_sum_squares(10))'
285
```

檔案在 Linux 上是 `libgeo.so`，在 macOS 上是 `libgeo.dylib`，在 Windows 上是 `geo.dll`（以及匯入函式庫 `geo.lib`）。只有匯出的函式從外部可見。函式庫內含標準函式庫和它自己的執行環境，執行環境會在函式庫載入時啟動，所以 C 那一端不需要先呼叫任何東西，而且可以從任何執行緒呼叫。跨越邊界的值都是 C 的值：數字、指標和 `#[repr(C)]` 結構。函式庫內的 panic 會中止主程式的行程，和任何回呼一樣；請為可能觸發 panic 的匯出函式加上主程式能檢查的 `#[on_panic(return = v)]`，並用 `std::panic::set_hook` 記錄。

在 `nabi.nabion` 中寫上 `lib: { kind: "cdylib" }`，每次 `nabi build` 都會產生共用函式庫。這時它會採用套件的 `version`：在 Linux 上，版本 0.3.x 的函式庫 soname 是 `libgeo.so.0.3`（從 1.0 起是 `libgeo.so.1`），旁邊會有一個同名的連結；在 macOS 上，它就是函式庫的 current version 與 compatibility version。

```nabion
Package {
    name: "geo",
    version: "0.3.1",
    lib: { kind: "cdylib" },
}
```

建置也會寫出一個 C 標頭檔 `target/debug/include/geo.h`，內含每個匯出的函式、它們用到的 `#[repr(C)]` 結構和 `#[repr(i32)]` 列舉，以及 `///` 註解。C 引入這個標頭檔，而不是手寫一份可能與實際脫節的複本：

```text
$ cat target/debug/include/geo.h
...
int64_t geo_sum_squares(int64_t n);
...
$ cc -I target/debug/include main.c -L target/debug -lgeo -o main
```

`nabic build --emit=c-header` 只寫出同樣的標頭檔。

### 給 C 和 Swift 的單一靜態函式庫：`--crate-type=staticlib`

共用函式庫必須和程式一起發布。若要把 Nabi 程式碼連結進程式本身，就改為建置單一靜態函式庫：`--crate-type=staticlib` 會把套件、它的相依套件、它用到的標準函式庫和執行環境放進 `target/debug/static/libgeo.a`，並在旁邊寫出標頭檔和 module map。C 只需連結這一個封存檔：

```text
$ nabi build --crate-type=staticlib
$ cc -I target/debug/include main.c target/debug/static/libgeo.a -o main
```

執行環境會在第一次呼叫匯出函式時啟動，在發出呼叫的那個執行緒上。`target/debug/include/module.modulemap` 讓標頭檔成為 clang 模組，所以 Swift 寫 `import geo` 就會連結這個函式庫。iOS 應用程式就是這樣使用 Nabi 程式碼的（[第 28 章](28-ios.html)）。在 `nabi.nabion` 中寫上 `lib: { kind: "staticlib" }`，每次 `nabi build` 都會產生它。

## 連結函式庫：`#[link]`

libc 的函式不需要額外設定就會被連結。其他函式庫則要在 `extern` 區塊上加上 `#[link("name")]`。這等同於把 `-lname` 傳給連結器。

```nabi
#[link("m")]
extern "C" {
    fn sqrt(x: f64) -> f64;
    fn pow(x: f64, y: f64) -> f64;
}

println(f"{unsafe { sqrt(2.0) }:.4}");
println(f"{unsafe { pow(2.0, 10.0) }}");
```

```output
1.4142
1024
```

要選用靜態函式庫，寫 `#[link(name = "foo", kind = "static")]`。

libc 和 libm 一律會被連結。其他任何函式庫都需要 `#[link]`，例如下面的 zlib。手上有 C 原始碼的函式庫不需要先建置：套件會編譯它（[C 原始檔](#c-sources-nabi-cc)）。

```nabi
use bare::ffi::CStr;
use bare::ptr::ConstPtr;

#[link("z")]
extern "C" {
    fn zlibVersion() -> ConstPtr<c_char>;
}
let v = unsafe { CStr::from_ptr(zlibVersion()) } ?? panic("no version");
println(f"zlib {try! v.to_str()}");
```

對於龐大的 C API，`nabi bindgen libfoo.h > libfoo_sys.nabi` 會從它的標頭檔寫出宣告；這不需要 C 編譯器。輸出只針對一個目標（`--target`，預設為主機），每個 C 型別都採用該目標的大小。


## 安裝在電腦上的函式庫：`Dep::system`

`#[link("z")]` 只會在連結器的慣用目錄中尋找 `libz`。用套件管理工具安裝的函式庫常常放在別處：Homebrew 把許多函式庫放在 `/opt/homebrew/opt/<name>` 下，vcpkg 則放在它自己的目錄樹中。在 `nabi.nabion` 中以 `Dep::system` 指名它，`nabi` 就會在每次建置時找到它：它會讀取 pkg-config 的 `.pc` 檔（不需要 `pkg-config` 程式），接著在 Homebrew、vcpkg 和 macOS SDK 中尋找。

```nabion
Package {
    name: "app",
    native: {
        "z": Dep::system(pkg_config: "zlib", brew: "zlib", apt: "zlib1g-dev"),
        "sqlite3": Dep::system(brew: "sqlite", apt: "libsqlite3-dev"),
    },
}
```

程式碼照常宣告函式，不需要 `#[link]`：

```text
extern "C" {
    fn crc32(crc: u64, buf: str, len: u32) -> u64;
    fn sqlite3_libversion_number() -> i32;
}
```

函式庫沒有安裝時，建置會說明如何在這台電腦上安裝它：

```text
$ nabi build
nabi: error: system library `z` (pkg-config `zlib`) not found for x86_64-unknown-linux-gnu (looked in pkg-config (zlib.pc)): install it with `sudo apt install zlib1g-dev` (or your distribution's package of it), or set PKG_CONFIG_PATH to the directory holding zlib.pc
```

## Apple 框架

在 macOS 上，系統 API 以框架（framework）的形式提供。用 `kind = "framework"`，或在 `nabi.nabion` 中用 `Dep::framework()` 連結框架，然後呼叫它的 C 函式：

```text
#[link(name = "CoreFoundation", kind = "framework")]
extern "C" {
    fn CFStringCreateWithCString(alloc: usize, s: str, encoding: u32) -> usize;
    fn CFStringGetLength(s: usize) -> isize;
    fn CFRelease(p: usize);
}

let s = unsafe { CFStringCreateWithCString(0, "hello, nabi", 0x0800_0100) };
println(f"{unsafe { CFStringGetLength(s) }}");
unsafe { CFRelease(s) };
```

為 Linux 或 Windows 建置時，`Dep::framework()` 會被略過；但在那些平台上寫 `#[link(kind = "framework")]` 是錯誤，所以請把這樣的區塊放在 `#[cfg(target_os = "macos")]` 之下。


## C 原始檔：`nabi cc`

Nabi 有自己的 C 編譯器，所以手上有原始碼的 C 程式碼不需要其他編譯器，也不必先建置成函式庫。它以 `nabicc` 的名稱安裝在 `nabi` 旁邊，而 `nabi cc` 執行的是同一個編譯器。它接受 gcc 和 clang 所接受的選項，會找到目標的 C 標頭檔，並以 Nabi 的程式碼產生器和連結器編譯，採用和 `extern "C"` 相同的 C ABI。

### 編譯 C 檔案

```c
// hello.c
#include <stdio.h>

int main(void) {
    printf("hello from C\n");
    return 0;
}
```

```text
$ nabi cc -O2 hello.c -o hello
$ ./hello
hello from C
```

不加 `-o` 時，程式是 `a.out`（在 Windows 上是 `a.exe`）。`-c` 會停在目的檔（`hello.o`，Windows 則是 `hello.obj`），`-E` 印出前置處理後的原始碼，`-g` 為 gdb、lldb 或 WinDbg 加上偵錯資訊。預設標準是 `gnu17`；`-std=c99`、`-std=c11` 或 `-std=c23` 會相應地設定 `__STDC_VERSION__`。

### 搭配 make 與 CMake

命令列和 gcc 相同，所以建置系統可以用任一個名稱作為 C 編譯器：

```bash
make CC="nabi cc"
make CC=nabicc
cmake -DCMAKE_C_COMPILER=nabicc -B build && cmake --build build
```

警告選項（`-Wall`、`-Wextra`）、`-fPIC`、`-pthread` 以及其他 `-f` 和 `-m` 選項都會被接受，但不會改變任何東西；程式碼一律與位置無關。`-MD` 和 `-MMD` 會寫出 make 用來追蹤標頭檔的 `.d` 檔。

### 套件中的 C

套件可以以原始碼形式帶著一個 C 函式庫，並從 Nabi 呼叫它。把它的檔案列在 `nabi.nabion` 的 `c` 欄位中：`nabi build` 會用內建編譯器、以建置的最佳化等級編譯它們，並與套件連結在一起。

```text
geom/
├── nabi.nabion
├── src/
│   └── main.nabi
└── vendor/
    ├── area.c
    └── area.h
```

```c
// vendor/area.h
double circle_area(double r);
int scaled(int x);
```

```c
// vendor/area.c
#include "area.h"

#ifndef SCALE
#define SCALE 1
#endif

double circle_area(double r) { return 3.14159265358979 * r * r; }
int scaled(int x) { return x * SCALE; }
```

```nabion
Package {
    name: "geom",
    c: {
        sources: ["vendor/area.c"],
        include: ["vendor"],
        define: { "SCALE": "10" },
        std: "c17",
        flags: ["-Wall"],
    },
}
```

`src/main.nabi` 在 `extern "C"` 區塊中宣告這些 C 函式，和使用任何 C 函式庫時一樣：

```nabi norun
extern "C" {
    fn circle_area(r: f64) -> f64;
    fn scaled(x: i32) -> i32;
}

println(f"{unsafe { circle_area(2.0) }:.3}");
println(f"{unsafe { scaled(4) }}");
```

```text
$ nabi run -q
12.566
40
```

`sources` 和 `include` 是相對於 `nabi.nabion` 的路徑。`define` 會成為 `-D` 選項；`std` 就是 `-std=`（預設為 `gnu17`）。`flags` 只接受 `-O*`、`-g`、`-W*`、`-f*` 和 `-m*`；寫在那裡的 `-O` 會取代該次建置對 C 程式碼使用的最佳化等級。`flags` 中的其他任何東西都是錯誤，錯誤會列出它接受的選項。工具鏈之外的東西一概不會執行：沒有建置指令碼，也沒有外部的 C 編譯器。

標頭檔中的 `static inline` 函式沒有可供連結的符號。`nabi bindgen --c-shim` 會為每一個寫出 C 包裝函式，並繫結到包裝函式；把包裝函式的檔案加進 `sources`：

```text
$ nabi bindgen vendor/area.h -o src/area.nabi --c-shim vendor/area_shim.c
```

```nabion
    c: {
        sources: ["vendor/area.c", "vendor/area_shim.c"],
        include: ["vendor"],
    },
```

### 交叉編譯

`--target` 用於 C 時，和用於 Nabi 程式碼時一樣。它接受 Nabi 的目標名稱，以及 gcc 和 clang 使用的 triple：

```text
$ nabi cc --target=aarch64-unknown-linux-gnu hello.c -o hello-arm64
$ nabi cc --target=x86_64-linux-gnu -c hello.c
$ nabi build --target=wasm32-wasip2
```

最後一個指令也會把套件的 C 程式碼編譯成 wasm32。Linux glibc 和 wasm32 目標不需要其他東西，因為工具鏈附帶了它們的 C 標頭檔和 C 函式庫（musl 目標則請用 `--sysroot` 提供標頭檔）。Apple 目標使用 macOS SDK，而 Windows 目標在 Windows 主機上使用 Windows SDK 和 Visual Studio 的標頭檔。

### 無法編譯的內容

有些 C 在 Nabi 的程式碼產生器中還沒有對應的功能。這些都是錯誤，而每個錯誤都會告訴你改寫成什麼：

- 行內組語（空的 `asm("")` 屏障沒有問題）、`&&label` 以及計算式 `goto *p`
- 可變長度陣列與 `alloca`
- `_Complex` 運算、`__int128`、比 `double` 更寬的 `long double`、`_Float16` 與 `__bf16` 運算
- 執行緒區域變數（`_Thread_local`、`__thread`）
- 向量型別與 SIMD 內建函式；`__SSE2__`、`__ARM_NEON` 和其他向量巨集都沒有定義，所以函式庫會走它們可攜的 C 路徑
- K&R 風格的定義，以及呼叫未宣告的函式
- `-shared`、`.s` / `.S` 原始檔，以及 C++ 或 Objective-C 原始檔

原子操作內建函式（`__atomic_*`、`__sync_*`）目前會編譯成一般的載入與儲存，所以在執行緒之間並不是原子的。在它們支援之前，請不要用 `nabi cc` 編譯在執行緒之間共用記憶體的 C 程式碼。

## Windows 注意事項

- `c_long` 和 `c_ulong` 在 Windows 上是 32 位元。以 `c_long` 表示 C 的 `long` 的程式碼不需修改就能建置；寫成 `i64` 的程式碼在那裡則與 C 那一端不符。
- Windows API 函式使用 `extern "system"`，它在 32 位元 x86 以外的平台上都是 `"C"`，在 32 位元 x86 上則是 stdcall。
- `W` 函式接受 UTF-16 字串：傳入 `WideCString::as_ptr()`，並用 `str::from_utf16` 讀取函式所填入的緩衝區（一個被固定的 `List<u16>`）。
- Windows 區塊上的 `#[errno]` 也會儲存 `GetLastError`：`bare::ffi::last_error()` 讀取它，而 `io::IoError::last_os_error()` 會優先使用它，其次才是 errno。
- `Library::open` 以 `LoadLibraryW` 載入 DLL（`"kernel32.dll"`）；`symbol::<extern "system" fn(..) -> R>` 取得它的函式。

```nabi norun
use bare::ffi::{WideCString, last_error};
use bare::gc::pin;
use bare::ptr::{ConstPtr, Ptr};

#[errno]
extern "system" {
    fn SetEnvironmentVariableW(name: ConstPtr<u16>, value: ConstPtr<u16>) -> i32;
    fn GetEnvironmentVariableW(name: ConstPtr<u16>, buf: Ptr<u16>, size: u32) -> u32;
}

let name = try! WideCString::new("NABI_WIDE_TEST");
let value = try! WideCString::new("h\u{E9}\u{1F600}");
unsafe { SetEnvironmentVariableW(name.as_ptr(), value.as_ptr()) };

let buf: List<u16> = [];
for _ in 0..1024 { buf.push(0u16); }
let held = pin(buf);
let n = unsafe { GetEnvironmentVariableW(name.as_ptr(), held.as_data_ptr(), 1024u32) };
if n == 0 {
    println(f"failed: {io::IoError::last_os_error()} ({last_error()})");
} else {
    println(try! str::from_utf16(buf[0:n as usize]));
}
```

## 小結

- 以 `extern "C" { fn name(..) -> T; }` 宣告 C 函式，並在 `unsafe { }` 中呼叫它們。C 的型別寫成 `c_int`、`c_long`、`c_size_t` 等；`void*` 是 `CPtr<c_void>`。
- C 的 `char` 是 `c_char` / `c'A'`；可能為 NULL 的指標是 `ConstPtr<T> | none`；C 字串以 `CStr::from_ptr` 複製；`wchar_t` 字串是 `WideCString`。
- 每次呼叫若會取得串列（或切片、`StrBuf`）的位址，就用 `gc::pin` 固定它；被固定的串列無法增長，所以要先 `reserve`。
- 以 `Ptr` / `ConstPtr` 上的 `read` / `write` / `add` / `cast` 以及 `bare::alloc` 處理 C 的記憶體；在 `Drop` 包裝型別中釋放 C 的資源。
- 結構使用 `#[repr(C)]`，回呼使用 `extern "C" fn`，而由 C 替你保管的值（串列、閉包）放進 `Handle`。
- `#[errno]` 會儲存 errno，供 `errno()` 和 `IoError::last_os_error()` 使用。
- C 所呼叫的函式中發生 panic 會中止程式；`#[on_panic(return = v)]` 則改為回傳 `v`，而 `std::panic::set_hook` 能看到每一次 panic。
- `Library::open` 在執行時載入共用函式庫；`#[link]` 在建置時連結函式庫。
- `#[export] pub extern "C" fn` 把函式匯出給 C。
- `nabi build --crate-type=cdylib`（或 `lib: { kind: "cdylib" }`）會產生 C 或 Python 能載入的共用函式庫，以及它的 C 標頭檔。
- `Dep::system(..)` 找出已安裝的 C 函式庫；`Dep::framework()` 和 `kind = "framework"` 連結 Apple 框架。
- `nabi cc`（也就是 `nabicc`）以 gcc 的選項編譯 C，搭配 make 時用 `CC="nabi cc"`；`nabi.nabion` 的 `c` 欄位會用它編譯套件的 C 原始檔。
- 把危險的呼叫侷限在小型的 `unsafe` 函式中，對外公開安全的函式。
- `unsafe trait`（例如 `Pod`）以 `unsafe impl` 實作；`#[derive(Pod)]` 會為 `#[repr(C)]` 結構寫出這個 impl。

## 延伸閱讀

- 規格：[5.9 extern 區塊](../spec/refs-lang/005-modules/509-extern-block.html)、[9.13 C 編譯器](../spec/refs-lang/009-tools/913-c-compiler.html)、[4.15 unsafe](../spec/refs-lang/004-functions/415-unsafe.html)、[4.15.4 unsafe trait](../spec/refs-lang/004-functions/415-unsafe.html#4154-in-traits)、[6.2.3 link 屬性](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#623-link-naming-an-external-library)、[與 C 相容的字元](../spec/refs-lang/001-lexical/108-character-literals.html)
- 標準函式庫規格：[FFI](../spec/refs-standard-libraries/100-bare/107-ffi.html)、[GC 固定](../spec/refs-standard-libraries/100-bare/106-gc-pin.html)、[指標](../spec/refs-standard-libraries/100-bare/101-ptr.html)、[`Pod` trait](../spec/refs-standard-libraries/100-bare/103-mem.html#133-the-pod-trait)、[panic hook](../spec/refs-standard-libraries/300-std/312-panic.html)
- API：[bare::ffi](../std/bare/ffi.html)、[bare::gc](../std/bare/gc.html)、[bare::ptr](../std/bare/ptr.html)、[bare::alloc](../std/bare/alloc.html)、[bare::mem](../std/bare/mem.html)、[std::panic](../std/std/panic.html)
- 下一章：[15. NABION](15-nabion.html)
