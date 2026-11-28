# 14. FFI 基础

本章介绍如何从 Nabi 调用用 C 编写的函数，如何把 Nabi 函数交给 C，以及如何用 Nabi 自带的 C 编译器编译 C 代码。操作系统和无数的库都通过 C ABI 提供功能，所以掌握 FFI（外部函数接口，foreign function interface），就能使用标准库没有提供的功能。

所有示例都在 macOS / Linux 上针对 libc 运行。Windows 上的不同之处集中放在最后。

## `extern "C"` 块

要使用 C 函数，先在 `extern "C" { ... }` 块中声明它的签名。声明没有函数体，以 `;` 结尾。名字就是 C 的符号名。

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

C 的类型用它们的 C 名字来写：`c_int`、`c_uint`、`c_long`、`c_ulong`、`c_size_t`、`c_double` 等都在 prelude 中。每个都是与目标上对应大小一致的 Nabi 数字类型（`c_int` 是 `i32`，`c_size_t` 是 `usize`），所以值在它们之间传递时无需转换。请使用这些类型，而不是固定大小的类型：C 的 `long` 在 Linux 和 macOS 上是 64 位，在 Windows 上却是 32 位，所以 `fn labs(x: i64) -> i64` 在一个平台上正确，在另一个平台上就错了，而 `fn labs(x: c_long) -> c_long` 在所有平台上都正确。`void*` 是 `CPtr<c_void>`。

`_Bool` 是 `bool`；`_Float16` 和 `__bf16` 是 `f16` 和 `bf16`，8 字节或 16 字节的向量（`float32x4_t`、`__m128`）在 arm64 和 x86-64 上是 `Simd`。`str` 参数以 NUL 结尾的 `char*` 传递；需要时 Nabi 会制作一份以 NUL 结尾的副本。

只有 C 能容纳的类型才能跨越边界，编译器会检查每一个类型，包括别名和结构体字段：元组、Unicode `char`、普通枚举或 `List` 都是错误，错误信息会说明应该怎么写（`struct`、`u32`、`#[repr(i32)]`、指针加长度）。

`extern` 块只能出现在模块顶层，并且必须写明 ABI 字符串（`"C"`）。

## 为什么需要 `unsafe`

编译器无法检查 C 函数的行为是否与它的签名一致。传入错误的指针或写错声明都可能破坏内存。所以 `extern "C"` 块中的每个函数都隐式地是 `unsafe`，只能在 `unsafe { ... }` 中调用。

```text check=checks/14-unsafe-call.nabi
error[E0061]: call to unsafe function `abs` outside an `unsafe` block or `unsafe fn`
  --> main.nabi:4:12
   |
 4 | println(f"{abs(-1)}");
   |            ^^^^^^^
   |
   = help: wrap the call in `unsafe { .. }`; every `extern` function, `extern "nabi"` included, is unsafe to call
```

在 `extern` 块中声明的 C 全局变量（`static mut optind: c_int;`）同样不受检查，所以读写它也需要 `unsafe`：

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

在 `unsafe` 之外这样做是错误，帮助信息会给出修正方法（`unsafe { optind = 1; }`）。C 的 `errno` 不是这样的全局变量（它是一个线程各自的宏）；如何读取它，请见[来自 C 的错误](#来自-c-的错误errno)。

`unsafe` 块表示“我保证这里是安全的”。好的做法是让这些块尽量小，并把危险的调用包在安全的 Nabi 函数中，这样从外部只能看到安全的 API。

## 字符串

### C 字符与 `CStr`

C 的 `char` 是一个字节；在 Nabi 中它是 `c_char` 类型，字面量写作 `c'A'`。它与 Unicode `char`（`'A'`）是不同的类型，所以涉及 FFI 边界的代码会很显眼。

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

如果 C 函数可能返回 NULL，就把它的返回类型声明为 `ConstPtr<T> | none`：一个地址，NULL 表示为 `none`。（`CPtr<T>` 则像 C 指针一样把 NULL 作为一个值来保存。）C 字符串就是这样返回的，它是一个字节归 C 所有的 `char*`：`strerror` 的缓冲区在下一次调用时会改变，`getenv` 的则在 `setenv` 之后改变。`CStr::from_ptr` 复制到 NUL 为止的字节，`to_str` 检查它们是否是 UTF-8。危险的部分被限制在 `from_c` 中，对外只公开安全的 `env_var`。

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

`str` 作为结果是错误，帮助信息会指向这里：Nabi 无法知道 C 会把这些字节保留多久、调用者是否必须释放它们，也无法知道它们是否是 UTF-8。要传递由字节构建的字符串，先创建一个 `CString`（`try CString::new(bytes)`，它拒绝内部含有 NUL 的字节），然后在 `cs` 仍在作用域内时传入 `cs.as_ptr()`。

### 宽字符串：`WideCString`

C 的 `wchar_t` 函数以及 Windows 的 `W` API 接受由 `c_wchar` 单元组成的字符串。`WideCString::new(s)` 从 `str` 创建这样的字符串，包含 NUL：在 Windows 上是 UTF-16，在其他平台上是每个字符一个单元。`as_ptr()` 是要传入的 `const wchar_t*`，`to_str()` 把字符串还原回来。`WideCStr::from_ptr(p)` 复制 C 返回的宽字符串，就像 `CStr::from_ptr` 那样。

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

## 缓冲区

### 传递列表：`gc::pin`

填充或读取缓冲区的 C 函数接受一个指针和一个长度。`List` 位于 GC 堆上，回收器可能会移动它。每次 C 调用都可能让一次回收运行，不论调用多短，所以在 C 使用列表地址期间必须固定它：`gc::pin(xs)` 在 `Pinned` 值处于作用域内时让列表保持原位，`as_data_ptr()` 是其第一个元素的地址。

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

编译器从不替你添加固定。签名中直接出现 `List` 是错误，帮助信息会给出应该写的声明和调用：`ptr: ConstPtr<T>, len: usize`，然后用 `let p = gc::pin(xs);` 得到的 `p.as_data_ptr()` 和 `xs.len()`。`str` 实参不需要固定：Nabi 会在调用期间让它保持原位。

列表的切片也以同样的方式固定（`pin(xs[1:3])`，其 `as_data_ptr()` 是切片的第一个元素），`StrBuf` 也是如此（`as_bytes_ptr()` 是它的第一个字节）：

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

被固定的列表不能增长：增长会把元素移到新的缓冲区，而 C 可能仍持有旧地址。请在固定之前预留空间，并在固定期间填充：

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

如果在那里再 `ys.push(9)` 一次，就需要更大的缓冲区，于是它会触发 panic，并说明该怎么做：

```text
cannot grow a list while it is pinned (C may hold its address): call `xs.reserve(n)` before `gc::pin(xs)`
```

`StrBuf` 遵循同样的规则。

### C 拥有的内存：`bare::ptr` 与 `bare::alloc`

裸指针来自 `bare` 层：

| 类型 | C 对应 | null |
|------|--------|------|
| `Ptr<T>`      | `T*`        | 否（null 用 `none` 表示） |
| `ConstPtr<T>` | `const T*`  | 否 |
| `CPtr<T>`     | `T*`        | 是 |

基本操作是通过指针的 `read()` / `write()`、`add(n)`（前进 n 个元素）和 `cast::<U>()`。C 堆上的内存来自 `bare::alloc`，要由你自己归还。GC 不管理这部分内存，所以忘记 `dealloc` 就会泄漏。

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

与 C 交换结构体时，加上 `#[repr(C)]` 表明其布局是 C 的布局（默认布局本来就是）。`union` 也以同样的方式跨越边界：C 按其成员传递它，Nabi 也一样。

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

## 编译器无法检查的承诺：`unsafe trait`

有些 trait 对实现它们的每个类型都做出某种承诺，而编译器无法检查这种承诺。这样的 trait 声明为 `unsafe trait`，它的每个 impl 都写作 `unsafe impl`：这里的 `unsafe` 是作者对承诺成立的保证，就像 `unsafe { }` 块之于调用一样。使用这个 trait（作为约束、调用它）不需要 `unsafe`。

标准库中的这类 trait 是 `bare::mem::Pod`，即“plain old data”：该类型的每种位模式都是一个值。整数、浮点数和 `c_char` 已经是 `Pod`。`reinterpret` 把一个 `Pod` 值的位当作另一个大小相同的值。由 `Pod` 字段组成的 `#[repr(C)]` 结构体可以用 `#[derive(Pod)]` 选择加入，也可以手写 `unsafe impl Pod for T {}`；无论哪种方式，编译器仍会检查布局、字段以及是否没有 `Drop`。

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

`unsafe trait` 的普通 `impl` 是错误，错误信息会给出应该写的形式（[E0078](../errors/E0078.html)）：

```text check=checks/14-unsafe-impl.nabi
error[E0078]: `Zeroable` is an `unsafe trait`: the impl promises what the compiler cannot check; write `unsafe impl Zeroable for i32`
  --> main.nabi:5:1
   |
 5 | impl Zeroable for i32 {
   | ^^^^
```

## 回调：`extern "C" fn`

对于接受函数指针的 C 函数，例如 `qsort`，把一个 Nabi 函数按 C ABI 编译后传入。在函数定义前加上 `extern "C"`。它的参数和返回类型必须与 FFI 兼容（整数、浮点数、指针、`#[repr(C)]` 结构体）。

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

`str` 在这里不是 FFI 类型，所以不能作为 `extern "C" fn` 的参数或结果。编译器会告诉你改为接受 `ConstPtr<c_char>`，并用 `CStr::from_ptr` 复制它。

C 代码也可能从它自己启动的线程中调用 `extern "C" fn`，线程池或音频库就是这样做的。Nabi 会在这样的线程第一次调用时接管它，所以回调可以分配内存，也可以使用任何 Nabi 代码；C 一侧不需要做任何设置。

`extern "C" fn` 不是闭包：它不能捕获周围的变量，在 C 接受函数的地方传入闭包是错误。帮助信息会指向下一节。

## 句柄：由 C 持有的 Nabi 值

会回调的 C API 通常在函数指针旁边接受一个 `void* user_data`，并在每次调用时把它交回。`Handle::new(value)`（`bare::ffi`）把一个 Nabi 值变成这样的指针：`h.as_ptr()` 是要传入的 `void*`，回调用 `Handle::<T>::from_ptr(p).get()` 取回该值。`List` 或 `Dict` 取回时是同一个对象，所以回调可以填充它。句柄让值保持存活、保持原位，直到 `h.release()`。

闭包也是值。要从 C 调用闭包，把它以 `fn` 类型保存在句柄中，再传入一个只有一行的 `extern "C" fn`，由它读回闭包并调用。

下面的 `each` 代表一个用收到的指针回调的 C 库：

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

这个指针是句柄编号，而不是地址：C 只负责传递它。`Handle` 与指针一样大，并且是 `Copy`，所以可以作为 `#[repr(C)]` 结构体的字段。C 用完指针后调用 `release`；如果 C 保留了第二份，`retain` 会多要求一次 `release`。出错时会触发 panic，并说明应该怎么写：在最后一次 `release` 之后 `get` 会提示 `handle already released ...`，用与创建句柄时不同的类型调用 `from_ptr` 会提示 `` this handle holds `List<str>`, not `Dict<str, i64>`: write `Handle::<List<str>>::from_ptr(p)` ``。

### 关闭 C 打开的资源：`Drop`

GC 会释放 Nabi 内存，但不运行终结器，所以 C 交出的资源（`FILE*`、`malloc` 分配的内存块、库句柄）要由实现了 `Drop` 的包装类型来释放。编译器在值的所有者离开作用域的地方，或者在绑定它的 `using` 块结束时调用 `drop`。

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

## 来自 C 的错误：`#[errno]`

许多 C 函数通过 `errno` 报告失败。在 `extern` 块（或单个声明）上加上 `#[errno]`：之后每次调用都会在返回后、在其他任何东西改变它之前立即保存 errno，`bare::ffi::errno()` 读取保存的值。`io::IoError::last_os_error()` 把它变成带有操作系统的错误种类和消息的 `IoError`，于是包装函数可以 `throw` 它。

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

没有 `#[errno]` 时什么都不会保存，`errno()` 保存的仍是更早某次调用的值。在调用前 `set_errno(0)` 用于只通过 errno 报告错误的 C 函数，例如 `strtol`。

## 回调中的 panic

panic 不能穿过 C 的栈帧离开一个由 C 调用的函数：C 对它一无所知，而退出会在损坏的状态上运行宿主的清理代码。所以在由 C 调用的 `extern "C" fn` 中发生 panic 时，会打印 panic 报告，再多打印一行，然后中止进程（在 Unix 上退出状态为 134），而不是以 101 退出：

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

要继续运行，就指明函数在 panic 时返回给 C 什么：`#[on_panic(return = v)]`，其中 `v` 是一个数字、`true` / `false`，或者表示空指针的 `0`（对没有结果的函数只写 `#[on_panic]`）。panic 仍会被报告，然后 C 得到 `v`。`std::panic::set_hook` 会在每次 panic 时先运行你的一个函数，用来记录日志：

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

panic 与该函数之间的代码会被跳过，其中的 `Drop` 也不会运行：它们本该释放的东西会泄漏，在那里锁住的 `Mutex` 会一直保持锁定。持有锁时可能失败的回调，应该返回一个错误值，而不是触发 panic。在 wasm32 上，回调中的 panic 总是中止进程。

## 在运行时加载库：`Library`

`#[link]`（见下文）在程序链接时绑定库。要像插件系统那样在程序运行时选择库，用 `bare::ffi::Library::open(path)` 加载它，用 `symbol::<F>(name)` 以 C 函数指针类型 `F` 查找函数。文件或符号不存在时，两者都会带着系统给出的原因抛出错误。

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

`symbol` 是 `unsafe` 的，因为 `F` 必须是函数真正的签名；不是 C 函数指针的类型会触发 panic。库在你 `close` 它之前一直保持加载，而 `close` 也是 `unsafe` 的：从它取得的每个函数指针在此之后都会悬空。`Library::this()` 是程序本身，连同它所链接的 C 库。

## 导出给 C：`#[export]`

要让 C 按名字找到一个 Nabi 函数，就把它定义为 `pub extern "C" fn`，并加上 `#[export]`：它的符号就是它自己的名字。`#[export_name("sym")]` 以另一个名字导出它。下面的示例在同一个程序中再次把两个符号声明为 C 函数，然后调用它们。

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

不带 `extern "C"` 的导出函数会使用 Nabi 自己的调用约定，而 C 不使用这种约定，所以这是错误，错误信息会说明应该怎么写：`` `compute` is exported to C but uses the Nabi ABI: write `pub extern "C" fn compute(..)` ``。

### 供 C 使用的共享库：`--crate-type=cdylib`

反方向：C 代码，或者通过 `ctypes` 的 Python，加载用 Nabi 编写的库。把被调用的函数作为 `#[export] pub extern "C" fn` 放在包的 `src/lib.nabi` 中，并用 `--crate-type=cdylib` 构建：

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

该文件在 Linux 上是 `libgeo.so`，在 macOS 上是 `libgeo.dylib`，在 Windows 上是 `geo.dll`（以及导入库 `geo.lib`）。从外部只能看到导出的函数。库带有标准库和它自己的运行时，运行时在库加载时启动，所以 C 一侧不需要先调用任何东西，可以从任何线程调用。跨越边界的值是 C 的值：数字、指针和 `#[repr(C)]` 结构体。库内部的 panic 会中止宿主进程，和任何回调中一样；请给可能 panic 的导出函数加上宿主可以检查的 `#[on_panic(return = v)]`，并用 `std::panic::set_hook` 记录日志。

在 `nabi.nabion` 中写上 `lib: { kind: "cdylib" }`，每次 `nabi build` 都会生成共享库。此时它会采用包的 `version`：在 Linux 上，版本 0.3.x 的库 soname 是 `libgeo.so.0.3`（从 1.0 起是 `libgeo.so.1`），旁边有一个同名的链接；在 macOS 上它是库的当前版本和兼容版本。

```nabion
Package {
    name: "geo",
    version: "0.3.1",
    lib: { kind: "cdylib" },
}
```

构建还会写出一个 C 头文件 `target/debug/include/geo.h`，其中包含每个导出函数、它们用到的 `#[repr(C)]` 结构体和 `#[repr(i32)]` 枚举，以及 `///` 注释。C 包含这个头文件，而不是包含一份可能与实际脱节的手写副本：

```text
$ cat target/debug/include/geo.h
...
int64_t geo_sum_squares(int64_t n);
...
$ cc -I target/debug/include main.c -L target/debug -lgeo -o main
```

`nabic build --emit=c-header` 只写出同样的头文件。

### 供 C 和 Swift 使用的单个静态库：`--crate-type=staticlib`

共享库必须与程序一起分发。要把 Nabi 代码链接进程序本身，改为构建一个静态库：`--crate-type=staticlib` 把包、它的依赖、它用到的标准库和运行时放进 `target/debug/static/libgeo.a`，并在旁边写出头文件和一个 module map。C 只需链接这一个归档文件：

```text
$ nabi build --crate-type=staticlib
$ cc -I target/debug/include main.c target/debug/static/libgeo.a -o main
```

运行时在第一次调用导出函数时启动，在发起调用的那个线程上。`target/debug/include/module.modulemap` 让头文件成为一个 clang 模块，所以 Swift 写 `import geo` 就能链接这个库。iOS 应用就是这样使用 Nabi 代码的（[第 28 章](28-ios.html)）。在 `nabi.nabion` 中写上 `lib: { kind: "staticlib" }`，每次 `nabi build` 都会生成它。

## 链接库：`#[link]`

libc 的函数无需额外设置就会被链接。对于其他库，在 `extern` 块上加上 `#[link("name")]`。这与向链接器传入 `-lname` 相同。

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

要选择静态库，写 `#[link(name = "foo", kind = "static")]`。

libc 和 libm 总是会被链接。其他任何库都需要 `#[link]`，例如下面的 zlib。以 C 源代码形式拥有的库不需要先构建：包会编译它（[C 源代码](#c-源代码nabi-cc)）。

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

对于大型的 C API，`nabi bindgen libfoo.h > libfoo_sys.nabi` 会根据头文件写出声明；它不需要 C 编译器。输出针对一个目标（`--target`，默认为主机），每个 C 类型采用该目标上的大小。C 的整数类型用它们的 C 名称书写（`int` 写作 `c_int`，`long` 写作 `c_long`），因此在大小不同的目标上声明依然匹配；`void *` 写作 `CPtr<c_void>`。


## 机器上安装的库：`Dep::system`

`#[link("z")]` 只在链接器通常的目录中查找 `libz`。用包管理器安装的库往往在别处：Homebrew 把许多库放在 `/opt/homebrew/opt/<name>` 下，vcpkg 放在它自己的目录树下。在 `nabi.nabion` 中用 `Dep::system` 指明它，`nabi` 会在每次构建时找到它：先读取 pkg-config 的 `.pc` 文件（不需要 `pkg-config` 程序），再查找 Homebrew、vcpkg 和 macOS SDK。

```nabion
Package {
    name: "app",
    native: {
        "z": Dep::system(pkg_config: "zlib", brew: "zlib", apt: "zlib1g-dev"),
        "sqlite3": Dep::system(brew: "sqlite", apt: "libsqlite3-dev"),
    },
}
```

代码照常声明函数，不需要 `#[link]`：

```text
extern "C" {
    fn crc32(crc: u64, buf: str, len: u32) -> u64;
    fn sqlite3_libversion_number() -> i32;
}
```

没有安装时，构建会说明如何在这台机器上安装它：

```text
$ nabi build
nabi: error: system library `z` (pkg-config `zlib`) not found for x86_64-unknown-linux-gnu (looked in pkg-config (zlib.pc)): install it with `sudo apt install zlib1g-dev` (or your distribution's package of it), or set PKG_CONFIG_PATH to the directory holding zlib.pc
```

## Apple 框架

在 macOS 上，系统 API 以框架的形式提供。用 `kind = "framework"` 或在 `nabi.nabion` 中用 `Dep::framework()` 链接一个框架，然后调用它的 C 函数：

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

面向 Linux 或 Windows 的构建会忽略 `Dep::framework()`；而在那里 `#[link(kind = "framework")]` 是错误，所以要把这样的块放在 `#[cfg(target_os = "macos")]` 之下。


## C 源代码：`nabi cc`

Nabi 有自己的 C 编译器，所以你以源代码形式拥有的 C 代码不需要其他编译器，也不必先构建成库。它以 `nabicc` 的名字安装在 `nabi` 旁边，`nabi cc` 运行的是同一个编译器。它接受 gcc 和 clang 接受的选项，会找到目标的 C 头文件，并用 Nabi 的代码生成器和链接器编译，采用与 `extern "C"` 相同的 C ABI。

### 编译一个 C 文件

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

不加 `-o` 时程序是 `a.out`（在 Windows 上是 `a.exe`）。`-c` 在生成目标文件后停止（`hello.o`，Windows 上是 `hello.obj`），`-E` 打印预处理后的源代码，`-g` 为 gdb、lldb 或 WinDbg 添加调试信息。默认标准是 `gnu17`；`-std=c99`、`-std=c11` 或 `-std=c23` 会相应地设置 `__STDC_VERSION__`。

### 与 make 和 CMake 配合

命令行与 gcc 的一样，所以构建系统可以用这两个名字中的任意一个作为 C 编译器：

```bash
make CC="nabi cc"
make CC=nabicc
cmake -DCMAKE_C_COMPILER=nabicc -B build && cmake --build build
```

警告选项（`-Wall`、`-Wextra`）、`-fPIC`、`-pthread` 以及其他 `-f` 和 `-m` 选项都会被接受，但不改变任何东西；代码总是位置无关的。`-MD` 和 `-MMD` 写出 make 用来跟踪头文件的 `.d` 文件。

### 包中的 C

包可以以源代码形式携带一个 C 库，并从 Nabi 调用它。在 `nabi.nabion` 的 `c` 字段中列出它的文件：`nabi build` 用内置编译器以构建的优化级别编译它们，并把它们与包链接在一起。

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

`src/main.nabi` 在 `extern "C"` 块中声明这些 C 函数，与任何 C 库一样：

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

`sources` 和 `include` 相对于 `nabi.nabion`。`define` 变成 `-D` 选项；`std` 就是 `-std=`（默认为 `gnu17`）。`flags` 只接受 `-O*`、`-g`、`-W*`、`-f*` 和 `-m*`；其中的 `-O` 会替换 C 代码的构建优化级别。`flags` 中的其他内容都是错误，错误信息会列出它接受的选项。工具链之外的东西一概不运行：没有构建脚本，也没有外部 C 编译器。

头文件中的 `static inline` 函数没有可链接的符号。`nabi bindgen --c-shim` 为每个这样的函数写出一个 C 包装函数并绑定该包装函数；请把包装文件加入 `sources`：

```text
$ nabi bindgen vendor/area.h -o src/area.nabi --c-shim vendor/area_shim.c
```

```nabion
    c: {
        sources: ["vendor/area.c", "vendor/area_shim.c"],
        include: ["vendor"],
    },
```

### 交叉编译

`--target` 对 C 的作用与对 Nabi 代码相同。它接受 Nabi 的目标名，也接受 gcc 和 clang 使用的三元组：

```text
$ nabi cc --target=aarch64-unknown-linux-gnu hello.c -o hello-arm64
$ nabi cc --target=x86_64-linux-gnu -c hello.c
$ nabi build --target=wasm32-wasip2
```

最后一条命令也会把包中的 C 编译为 wasm32。Linux glibc 和 wasm32 目标不需要其他东西，因为工具链自带它们的 C 头文件和 C 库（对于 musl 目标，用 `--sysroot` 提供头文件）。Apple 目标使用 macOS SDK，Windows 目标在 Windows 主机上使用 Windows SDK 和 Visual Studio 的头文件。

### 它不能编译什么

有些 C 在 Nabi 的代码生成器中还没有对应的实现。这些都是错误，每个错误都会说明应该改写成什么：

- 内联汇编（空的 `asm("")` 屏障可以）、`&&label` 和计算跳转 `goto *p`
- 变长数组和 `alloca`
- `_Complex` 运算、`__int128`、比 `double` 更宽的 `long double`、`_Float16` 和 `__bf16` 运算
- 线程局部变量（`_Thread_local`、`__thread`）
- 向量类型和 SIMD 内建函数；`__SSE2__`、`__ARM_NEON` 和其他向量宏没有定义，所以库会走它们可移植的 C 路径
- K&R 风格的定义，以及对未声明函数的调用
- `-shared`、`.s` / `.S` 源文件，以及 C++ 或 Objective-C 源文件

原子内建函数（`__atomic_*`、`__sync_*`）目前编译为普通的加载和存储，所以它们在线程之间并不是原子的。在它们支持之前，请不要用 `nabi cc` 编译在线程之间共享内存的 C 代码。

## Windows 注意事项

- `c_long` 和 `c_ulong` 在 Windows 上是 32 位。用 `c_long` 表示 C 的 `long` 的代码无需修改就能构建；写成 `i64` 的代码在那里与 C 一侧不匹配。
- Windows API 函数使用 `extern "system"`，它在 32 位 x86 以外的所有平台上都是 `"C"`，在 32 位 x86 上则是 stdcall。
- `W` 函数接受 UTF-16 字符串：传入 `WideCString::as_ptr()`，并用 `str::from_utf16` 读取函数填充的缓冲区（一个被固定的 `List<u16>`）。
- Windows 块上的 `#[errno]` 还会保存 `GetLastError`：`bare::ffi::last_error()` 读取它，`io::IoError::last_os_error()` 会优先于 errno 使用它。
- `Library::open` 用 `LoadLibraryW` 加载 DLL（`"kernel32.dll"`）；`symbol::<extern "system" fn(..) -> R>` 取得其中的函数。

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

## 小结

- 用 `extern "C" { fn name(..) -> T; }` 声明 C 函数，在 `unsafe { }` 中调用它们。C 的类型写作 `c_int`、`c_long`、`c_size_t` 等；`void*` 是 `CPtr<c_void>`。
- C 的 `char` 是 `c_char` / `c'A'`；可能为 NULL 的指针是 `ConstPtr<T> | none`；C 字符串用 `CStr::from_ptr` 复制；`wchar_t` 字符串是 `WideCString`。
- 每次调用要取得列表（或切片、`StrBuf`）的地址时，都用 `gc::pin` 固定它；被固定的列表不能增长，所以要先 `reserve`。
- 用 `Ptr` / `ConstPtr` 上的 `read` / `write` / `add` / `cast` 以及 `bare::alloc` 处理 C 内存；在 `Drop` 包装类型中释放 C 资源。
- 结构体使用 `#[repr(C)]`，回调使用 `extern "C" fn`，由 C 替你携带的值（列表、闭包）放进 `Handle`。
- `#[errno]` 为 `errno()` 和 `IoError::last_os_error()` 保存 errno。
- 由 C 调用的函数中发生 panic 会中止进程；`#[on_panic(return = v)]` 改为返回 `v`，`std::panic::set_hook` 可以看到每次 panic。
- `Library::open` 在运行时加载共享库；`#[link]` 在构建时链接库。
- `#[export] pub extern "C" fn` 把函数导出给 C。
- `nabi build --crate-type=cdylib`（或 `lib: { kind: "cdylib" }`）生成供 C 或 Python 加载的共享库及其 C 头文件。
- `Dep::system(..)` 查找已安装的 C 库；`Dep::framework()` 和 `kind = "framework"` 链接 Apple 框架。
- `nabi cc`（也可用 `nabicc`）用 gcc 的选项编译 C，make 中用 `CC="nabi cc"`；`nabi.nabion` 的 `c` 字段用它编译包中的 C 源代码。
- 把危险的调用限制在小的 `unsafe` 函数中，对外公开安全的函数。
- `unsafe trait`（例如 `Pod`）用 `unsafe impl` 实现；`#[derive(Pod)]` 为 `#[repr(C)]` 结构体写出这个 impl。

## 延伸阅读

- 规范：[5.9 extern 块](../spec/refs-lang/005-modules/509-extern-block.html)、[9.13 C 编译器](../spec/refs-lang/009-tools/913-c-compiler.html)、[4.15 unsafe](../spec/refs-lang/004-functions/415-unsafe.html)、[4.15.4 unsafe trait](../spec/refs-lang/004-functions/415-unsafe.html#4154-在-trait-中)、[6.2.3 link 属性](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#623-link-指定外部库)、[与 C 兼容的字符](../spec/refs-lang/001-lexical/108-character-literals.html)
- 标准库规范：[FFI](../spec/refs-standard-libraries/100-bare/107-ffi.html)、[GC 固定](../spec/refs-standard-libraries/100-bare/106-gc-pin.html)、[指针](../spec/refs-standard-libraries/100-bare/101-ptr.html)、[`Pod` trait](../spec/refs-standard-libraries/100-bare/103-mem.html#133-pod-trait)、[panic 钩子](../spec/refs-standard-libraries/300-std/312-panic.html)
- API：[bare::ffi](../../std/bare/ffi.html)、[bare::gc](../../std/bare/gc.html)、[bare::ptr](../../std/bare/ptr.html)、[bare::alloc](../../std/bare/alloc.html)、[bare::mem](../../std/bare/mem.html)、[std::panic](../../std/std/panic.html)
- 下一章：[15. NABION](15-nabion.html)
