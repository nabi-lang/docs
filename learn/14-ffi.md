# 14. FFI Basics

This chapter shows how to call functions written in C from Nabi, how to hand Nabi functions to C, and how to compile C code with Nabi's own C compiler. Operating systems and countless libraries expose their functionality through the C ABI, so knowing FFI (foreign function interface) lets you use features the standard library does not provide.

All examples run against libc on macOS / Linux. What differs on Windows is collected at the end.

## `extern "C"` blocks

To use a C function, first declare its signature in an `extern "C" { ... }` block. There is no body; the declaration ends with `;`. The name is the C symbol name as is.

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

Write C's types with their C names: `c_int`, `c_uint`, `c_long`, `c_ulong`, `c_size_t`, `c_double` and the rest are in the prelude. Each is the Nabi number type of the right size for the target (`c_int` is `i32`, `c_size_t` is `usize`), so values pass between them without a conversion. Use them rather than a fixed size: C's `long` is 64 bits on Linux and macOS but 32 bits on Windows, so `fn labs(x: i64) -> i64` is right on one and wrong on the other, while `fn labs(x: c_long) -> c_long` is right everywhere. A `void*` is `CPtr<c_void>`.

`_Bool` is `bool`; `_Float16` and `__bf16` are `f16` and `bf16`, and an 8- or 16-byte vector (`float32x4_t`, `__m128`) is a `Simd` on arm64 and x86-64. A `str` parameter is passed as a NUL-terminated `char*`; Nabi makes a NUL-terminated copy when needed.

Only types C can hold may cross, and the compiler checks every one, through aliases and struct fields: a tuple, a Unicode `char`, a plain enum or a `List` is an error that says what to write instead (a `struct`, `u32`, `#[repr(i32)]`, a pointer and a length).

`extern` blocks may only appear at module top level, and must name the ABI string (`"C"`).

## Why `unsafe` is required

The compiler has no way to check that a C function behaves as its signature says. Passing a bad pointer or getting the declaration wrong can corrupt memory. So every function in an `extern "C"` block is implicitly `unsafe` and can only be called inside `unsafe { ... }`.

```text check=checks/14-unsafe-call.nabi
error[E0061]: call to unsafe function `abs` outside an `unsafe` block or `unsafe fn`
  --> main.nabi:4:12
   |
 4 | println(f"{abs(-1)}");
   |            ^^^^^^^
   |
   = help: wrap the call in `unsafe { .. }`; every `extern` function, `extern "nabi"` included, is unsafe to call
```

A C global declared in an `extern` block (`static mut optind: c_int;`) is unchecked in the same way, so reading or writing it also needs `unsafe`:

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

Outside `unsafe` it is an error whose help shows the fix (`unsafe { optind = 1; }`). C's `errno` is not such a global (it is a per-thread macro); [Errors from C](#errors-from-c-errno) shows how to read it.

An `unsafe` block says "I guarantee safety here." Good practice is to keep these blocks small and wrap dangerous calls in safe Nabi functions, so only a safe API is visible from outside.

## Strings

### C characters and `CStr`

A C `char` is one byte; in Nabi it is the `c_char` type with `c'A'` literals. It is a different type from the Unicode `char` (`'A'`), so code touching the FFI boundary stands out.

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

If a C function can return NULL, declare its return type as `ConstPtr<T> | none`: one address, with NULL as `none`. (`CPtr<T>` holds NULL as a value instead, as a C pointer does.) A C string comes back this way, as a `char*` whose bytes belong to C: `strerror`'s buffer changes on the next call, and `getenv`'s after a `setenv`. `CStr::from_ptr` copies the bytes up to the NUL, and `to_str` checks that they are UTF-8. The dangerous part is confined to `from_c`, and only the safe `env_var` is exposed.

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

A `str` result is an error, with help that points here: Nabi cannot know how long C keeps the bytes, whether the caller must free them, or that they are UTF-8. To pass a string built from bytes, make a `CString` (`try CString::new(bytes)`, which refuses a NUL inside) and pass `cs.as_ptr()` while `cs` is in scope.

### Wide strings: `WideCString`

C's `wchar_t` functions, and the Windows `W` APIs, take strings of `c_wchar` units. `WideCString::new(s)` makes one from a `str`, NUL included: UTF-16 on Windows, one unit per character elsewhere. `as_ptr()` is the `const wchar_t*` to pass, and `to_str()` gives the string back. `WideCStr::from_ptr(p)` copies one that C returns, as `CStr::from_ptr` does.

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

## Buffers

### Passing a list: `gc::pin`

A C function that fills or reads a buffer takes a pointer and a length. A `List` lives on the GC heap, and the collector may move it. Every C call lets a collection run, however short the call, so the list must be pinned while C uses its address: `gc::pin(xs)` keeps it in place while the `Pinned` value is in scope, and `as_data_ptr()` is the address of its first element.

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

The compiler never adds a pin for you. A `List` in the signature itself is an error, and its help shows the declaration and the call to write: `ptr: ConstPtr<T>, len: usize`, then `p.as_data_ptr()` and `xs.len()` from `let p = gc::pin(xs);`. A `str` argument needs no pin: Nabi keeps it in place for the call.

A slice of a list pins the same way (`pin(xs[1:3])`, whose `as_data_ptr()` is the slice's first element), and so does a `StrBuf` (`as_bytes_ptr()` is its first byte):

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

A pinned list cannot grow: growing would move its elements to a new buffer while C may still hold the old address. Reserve the room before you pin, and fill it while the pin lasts:

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

One more `ys.push(9)` there would need a bigger buffer, so it panics and says what to do:

```text
cannot grow a list while it is pinned (C may hold its address): call `xs.reserve(n)` before `gc::pin(xs)`
```

A `StrBuf` follows the same rule.

### Memory C owns: `bare::ptr` and `bare::alloc`

Raw pointers come from the `bare` layer:

| Type | C equivalent | null |
|------|--------|------|
| `Ptr<T>`      | `T*`        | no (null is expressed as `none`) |
| `ConstPtr<T>` | `const T*`  | no |
| `CPtr<T>`     | `T*`        | yes |

The basic operations are `read()` / `write()` through the pointer, `add(n)` (advance by n elements), and `cast::<U>()`. Memory from the C heap comes from `bare::alloc`, and you return it yourself. The GC does not manage this memory, so forgetting `dealloc` leaks it.

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

When exchanging structs with C, add `#[repr(C)]` to say the layout is C's (the default layout already is). A `union` crosses the same way: C passes it by its members, and so does Nabi.

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

## Promises the compiler cannot check: `unsafe trait`

Some traits promise something about every type that implements them that the compiler has no way to check. Such a trait is declared `unsafe trait`, and each impl of it is written `unsafe impl`: the `unsafe` is the author's word that the promise holds, just as an `unsafe { }` block is for a call. Using the trait (a bound, a call) needs no `unsafe`.

The standard library's one is `bare::mem::Pod`, "plain old data": every bit pattern of the type is a value. Integers, floats and `c_char` are `Pod` already. `reinterpret` takes the bits of one `Pod` value as another of the same size. A `#[repr(C)]` struct of `Pod` fields opts in with `#[derive(Pod)]`, or by hand with `unsafe impl Pod for T {}`; either way the compiler still checks the layout, the fields and that there is no `Drop`.

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

A plain `impl` of an `unsafe trait` is an error that gives the form to write ([E0078](../errors/E0078.html)):

```text check=checks/14-unsafe-impl.nabi
error[E0078]: `Zeroable` is an `unsafe trait`: the impl promises what the compiler cannot check; write `unsafe impl Zeroable for i32`
  --> main.nabi:5:1
   |
 5 | impl Zeroable for i32 {
   | ^^^^
```

## Callbacks: `extern "C" fn`

For C functions that take a function pointer, such as `qsort`, compile a Nabi function with the C ABI and pass it. Put `extern "C"` before the function definition. Its parameters and return type must be FFI-compatible (integers, floats, pointers, `#[repr(C)]` structs).

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

`str` is not an FFI type here, so it cannot be a parameter or result of an `extern "C" fn`. The compiler tells you to take a `ConstPtr<c_char>` and copy it with `CStr::from_ptr`.

C code may also call an `extern "C" fn` from a thread it started itself, as a thread pool or an audio library does. Nabi attaches such a thread on its first call, so the callback can allocate and use any Nabi code; nothing has to be set up on the C side.

An `extern "C" fn` is not a closure: it cannot capture surrounding variables, and passing a closure where C takes a function is an error. Its help points to the next section.

## Handles: Nabi values C holds

A C API that calls back usually takes a `void* user_data` next to the function pointer and hands it to each call. `Handle::new(value)` (`bare::ffi`) turns a Nabi value into such a pointer: `h.as_ptr()` is the `void*` to pass, and the callback gets the value back with `Handle::<T>::from_ptr(p).get()`. A `List` or `Dict` comes back as the same object, so the callback can fill it. The handle keeps the value alive and in place until `h.release()`.

A closure is a value too. To call one from C, keep it in a handle, stored as a `fn` type, and pass a one-line `extern "C" fn` that reads it back and calls it.

Below, `each` stands in for a C library that calls back with the pointer it was given:

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

The pointer is a handle number, not an address: C only passes it along. A `Handle` is pointer-sized and `Copy`, so it can be a field of a `#[repr(C)]` struct. Call `release` once C is done with the pointer; if C keeps a second copy, `retain` adds one more `release`. A mistake panics with what to write: `get` after the last `release` says `handle already released ...`, and `from_ptr` with another type than the handle was made with says `` this handle holds `List<str>`, not `Dict<str, i64>`: write `Handle::<List<str>>::from_ptr(p)` ``.

### Closing what C opened: `Drop`

The GC frees Nabi memory but runs no finalizers, so a resource C hands out (a `FILE*`, a `malloc`ed block, a library handle) is released by a wrapper type that implements `Drop`. The compiler calls `drop` where the value's owner leaves scope, or at the end of the `using` block that binds it.

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

## Errors from C: `#[errno]`

Many C functions report failure through `errno`. Put `#[errno]` on the `extern` block (or on one declaration): each call then saves errno right after it returns, before anything else can change it, and `bare::ffi::errno()` reads the saved value. `io::IoError::last_os_error()` turns it into an `IoError` with the OS's kind and message, so a wrapper can `throw` it.

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

Without `#[errno]` nothing is saved, and `errno()` still holds an older call's value. `set_errno(0)` before a call is for C functions that report errors only through errno, such as `strtol`.

## Panics in callbacks

A panic must not leave a function C called through C's frames: C knows nothing of it, and exiting would run the host's cleanup on broken state. So a panic in an `extern "C" fn` that C called prints its report, then one more line, and aborts the process (exit status 134 on Unix) instead of exiting with 101:

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

To keep going, say what the function returns to C when it panics: `#[on_panic(return = v)]`, where `v` is a number, `true` / `false`, or `0` for a null pointer (`#[on_panic]` alone for a function without a result). The panic is still reported, then C gets `v`. `std::panic::set_hook` runs a function of yours on every panic first, to log it:

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

The code between the panic and the function is skipped, and its `Drop`s do not run: what they would free leaks, and a `Mutex` locked there stays locked. A callback that can fail while it holds a lock should return an error value instead of panicking. On wasm32 a panic in a callback always aborts.

## Loading a library at run time: `Library`

`#[link]` (below) binds a library when the program is linked. To choose one while the program runs, as a plugin system does, `bare::ffi::Library::open(path)` loads it, and `symbol::<F>(name)` looks up a function as the C function pointer type `F`. Both throw with the system's reason when the file or the symbol is missing.

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

`symbol` is `unsafe` because `F` must be the function's real signature; a type that is not a C function pointer panics. A library stays loaded until you `close` it, which is `unsafe` too: every function pointer taken from it dangles afterwards. `Library::this()` is the program itself, with the C library it was linked with.

## Exporting to C: `#[export]`

To let C find a Nabi function by name, define it as `pub extern "C" fn` and put `#[export]` on it: its symbol is then its own name. `#[export_name("sym")]` exports it under another name. The example below declares both symbols again as C functions in the same program and calls them.

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

An exported function without `extern "C"` would use Nabi's own calling convention, which C does not, so it is an error that says what to write: `` `compute` is exported to C but uses the Nabi ABI: write `pub extern "C" fn compute(..)` ``.

### Shared libraries for C: `--crate-type=cdylib`

The other direction: C code, or Python through `ctypes`, loads a library written in Nabi. Put the functions it calls in the package's `src/lib.nabi` as `#[export] pub extern "C" fn`s and build with `--crate-type=cdylib`:

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

The file is `libgeo.so` on Linux, `libgeo.dylib` on macOS and `geo.dll` (with the import library `geo.lib`) on Windows. Only the exported functions are visible from outside. The library carries the standard library and its own runtime, which starts when the library is loaded, so the C side calls nothing first, from any thread. Values that cross are C values: numbers, pointers and `#[repr(C)]` structs. A panic inside the library aborts the host process, as in any callback; give an exported function that can panic an `#[on_panic(return = v)]` the host can check, and log with `std::panic::set_hook`.

Put `lib: { kind: "cdylib" }` in `nabi.nabion` to make the shared library on every `nabi build`. It then takes the package's `version`: on Linux the library's soname is `libgeo.so.0.3` for version 0.3.x (`libgeo.so.1` from 1.0), with a link of that name beside it; on macOS it is the library's current and compatibility version.

```nabion
Package {
    name: "geo",
    version: "0.3.1",
    lib: { kind: "cdylib" },
}
```

The build also writes a C header, `target/debug/include/geo.h`, with every exported function and the `#[repr(C)]` structs and `#[repr(i32)]` enums they use, and the `///` comments. C includes it instead of a hand-written copy that can drift:

```text
$ cat target/debug/include/geo.h
...
int64_t geo_sum_squares(int64_t n);
...
$ cc -I target/debug/include main.c -L target/debug -lgeo -o main
```

`nabic build --emit=c-header` writes the same header alone.

### One static library for C and Swift: `--crate-type=staticlib`

A shared library has to ship beside the program. To link Nabi code into the program itself, build one static library instead: `--crate-type=staticlib` puts the package, its dependencies, the standard library it uses and the runtime into `target/debug/static/libgeo.a`, and writes the header and a module map beside it. C links that one archive and nothing else:

```text
$ nabi build --crate-type=staticlib
$ cc -I target/debug/include main.c target/debug/static/libgeo.a -o main
```

The runtime starts on the first exported call, on whatever thread makes it. `target/debug/include/module.modulemap` makes the header a clang module, so Swift says `import geo` and links the library. This is how an iOS app takes Nabi code ([chapter 28](28-ios.html)). `lib: { kind: "staticlib" }` in `nabi.nabion` makes it on every `nabi build`.

## Linking libraries: `#[link]`

libc functions are linked without anything extra. For other libraries, put `#[link("name")]` on the `extern` block. This is the same as passing `-lname` to the linker.

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

To select a static library, write `#[link(name = "foo", kind = "static")]`.

libc and libm are always linked. Any other library needs `#[link]`, like zlib below. A library you have as C source does not need building first: the package compiles it ([C sources](#c-sources-nabi-cc)).

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

For a large C API, `nabi bindgen libfoo.h > libfoo_sys.nabi` writes the declarations from its header; it needs no C compiler. The output is for one target (`--target`, the host by default), with each C type at that target's size. C's integer types are written with their C names (`int` as `c_int`, `long` as `c_long`), so the declarations keep fitting where sizes differ, and `void *` as `CPtr<c_void>`.


## Libraries installed on the machine: `Dep::system`

`#[link("z")]` finds `libz` only in the linker's usual directories. A library installed with a package manager often lives elsewhere: Homebrew keeps many under `/opt/homebrew/opt/<name>`, vcpkg under its own tree. Name it in `nabi.nabion` with `Dep::system`, and `nabi` finds it on each build: it reads pkg-config `.pc` files (no `pkg-config` program needed), then looks in Homebrew, vcpkg and the macOS SDK.

```nabion
Package {
    name: "app",
    native: {
        "z": Dep::system(pkg_config: "zlib", brew: "zlib", apt: "zlib1g-dev"),
        "sqlite3": Dep::system(brew: "sqlite", apt: "libsqlite3-dev"),
    },
}
```

The code declares the functions as usual, without `#[link]`:

```text
extern "C" {
    fn crc32(crc: u64, buf: str, len: u32) -> u64;
    fn sqlite3_libversion_number() -> i32;
}
```

When it is not installed, the build says how to install it on this machine:

```text
$ nabi build
nabi: error: system library `z` (pkg-config `zlib`) not found for x86_64-unknown-linux-gnu (looked in pkg-config (zlib.pc)): install it with `sudo apt install zlib1g-dev` (or your distribution's package of it), or set PKG_CONFIG_PATH to the directory holding zlib.pc
```

## Apple frameworks

On macOS, system APIs come as frameworks. Link one with `kind = "framework"`, or with `Dep::framework()` in `nabi.nabion`, and call its C functions:

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

A build for Linux or Windows leaves a `Dep::framework()` out; a `#[link(kind = "framework")]` there is an error, so put such a block under `#[cfg(target_os = "macos")]`.


## C sources: `nabi cc`

Nabi has its own C compiler, so C code you have as source needs no other compiler and does not have to be built into a library first. It is installed as `nabicc` beside `nabi`, and `nabi cc` runs the same compiler. It takes the options gcc and clang take, finds the C headers for the target, and compiles with Nabi's code generator and linker, under the same C ABI that `extern "C"` uses.

### Compiling a C file

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

Without `-o` the program is `a.out` (`a.exe` on Windows). `-c` stops at the object (`hello.o`, or `hello.obj` for Windows), `-E` prints the preprocessed source, and `-g` adds debug information for gdb, lldb or WinDbg. The default standard is `gnu17`; `-std=c99`, `-std=c11` or `-std=c23` set `__STDC_VERSION__` to match.

### With make and CMake

The command line is gcc's, so a build system takes either name as its C compiler:

```bash
make CC="nabi cc"
make CC=nabicc
cmake -DCMAKE_C_COMPILER=nabicc -B build && cmake --build build
```

Warning options (`-Wall`, `-Wextra`), `-fPIC`, `-pthread` and the other `-f` and `-m` options are accepted and change nothing; the code is always position independent. `-MD` and `-MMD` write the `.d` files make uses to track headers.

### C in a package

A package can carry a C library as source and call it from Nabi. List its files in the `c` field of `nabi.nabion`: `nabi build` compiles them with the built-in compiler, at the build's optimization level, and links them with the package.

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

`src/main.nabi` declares the C functions in an `extern "C"` block, as for any C library:

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

`sources` and `include` are relative to `nabi.nabion`. `define` becomes `-D` options; `std` is the `-std=` (default `gnu17`). `flags` takes only `-O*`, `-g`, `-W*`, `-f*` and `-m*`; an `-O` there replaces the build's level for the C code. Anything else in `flags` is an error that lists what it takes. Nothing outside the toolchain runs: there is no build script and no external C compiler.

A header's `static inline` functions have no symbol to link. `nabi bindgen --c-shim` writes a C wrapper for each and binds the wrapper; add the wrapper file to `sources`:

```text
$ nabi bindgen vendor/area.h -o src/area.nabi --c-shim vendor/area_shim.c
```

```nabion
    c: {
        sources: ["vendor/area.c", "vendor/area_shim.c"],
        include: ["vendor"],
    },
```

### Cross-compiling

`--target` works for C as it does for Nabi code. It takes Nabi's target names and the triples gcc and clang use:

```text
$ nabi cc --target=aarch64-unknown-linux-gnu hello.c -o hello-arm64
$ nabi cc --target=x86_64-linux-gnu -c hello.c
$ nabi build --target=wasm32-wasip2
```

The last one compiles the package's C for wasm32 too. Linux glibc and wasm32 targets need nothing more, because the toolchain ships their C headers and C library (for musl targets, give the headers with `--sysroot`). Apple targets use the macOS SDK, and Windows targets the Windows SDK and Visual Studio headers on a Windows host.

### What it does not compile

Some C has no counterpart in Nabi's code generator yet. These are errors, and each error says what to write instead:

- inline assembly (an empty `asm("")` barrier is fine), `&&label` and computed `goto *p`
- variable length arrays and `alloca`
- `_Complex` arithmetic, `__int128`, a `long double` wider than `double`, `_Float16` and `__bf16` arithmetic
- thread-local variables (`_Thread_local`, `__thread`)
- vector types and SIMD intrinsics; `__SSE2__`, `__ARM_NEON` and the other vector macros are not defined, so libraries take their portable C path
- K&R-style definitions and calls of undeclared functions
- `-shared`, `.s` / `.S` sources, and C++ or Objective-C sources

The atomic builtins (`__atomic_*`, `__sync_*`) compile as plain loads and stores for now, so they are not atomic across threads. Keep C code that shares memory between threads out of `nabi cc` until they are.

## Windows notes

- `c_long` and `c_ulong` are 32 bits on Windows. Code that writes `c_long` for C's `long` builds unchanged; code that wrote `i64` does not match the C side there.
- Windows API functions use `extern "system"`, which is `"C"` everywhere but 32-bit x86, where it is stdcall.
- The `W` functions take UTF-16 strings: pass `WideCString::as_ptr()`, and read a buffer the function fills (a pinned `List<u16>`) with `str::from_utf16`.
- `#[errno]` on a Windows block also saves `GetLastError`: `bare::ffi::last_error()` reads it, and `io::IoError::last_os_error()` uses it before errno.
- `Library::open` loads a DLL (`"kernel32.dll"`) with `LoadLibraryW`; `symbol::<extern "system" fn(..) -> R>` gives its functions.

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

## Summary

- Declare C functions with `extern "C" { fn name(..) -> T; }` and call them inside `unsafe { }`. Write C's types as `c_int`, `c_long`, `c_size_t` and so on; a `void*` is `CPtr<c_void>`.
- A C `char` is `c_char` / `c'A'`; a pointer that may be NULL is `ConstPtr<T> | none`; a C string is copied with `CStr::from_ptr`; a `wchar_t` string is a `WideCString`.
- Pin a list (or a slice, or a `StrBuf`) with `gc::pin` for every call that gets its address; a pinned list cannot grow, so `reserve` first.
- Handle C memory with `read` / `write` / `add` / `cast` on `Ptr` / `ConstPtr`, and with `bare::alloc`; release C resources in a `Drop` wrapper.
- Structs use `#[repr(C)]`, callbacks use `extern "C" fn`, and a value C carries for you (a list, a closure) goes in a `Handle`.
- `#[errno]` saves errno for `errno()` and `IoError::last_os_error()`.
- A panic in a function C called aborts; `#[on_panic(return = v)]` returns `v` instead, and `std::panic::set_hook` sees every panic.
- `Library::open` loads a shared library at run time; `#[link]` links one at build time.
- `#[export] pub extern "C" fn` exports a function to C.
- `nabi build --crate-type=cdylib` (or `lib: { kind: "cdylib" }`) makes a shared library that C or Python loads, and its C header.
- `Dep::system(..)` finds an installed C library; `Dep::framework()` and `kind = "framework"` link an Apple framework.
- `nabi cc` (also `nabicc`) compiles C with gcc's options, `CC="nabi cc"` for make; the `c` field of `nabi.nabion` compiles a package's C sources with it.
- Confine dangerous calls to small `unsafe` functions and expose safe functions to the outside.
- An `unsafe trait` (such as `Pod`) is implemented with `unsafe impl`; `#[derive(Pod)]` writes that impl for a `#[repr(C)]` struct.

## Further reading

- Spec: [5.9 extern blocks](../spec/refs-lang/005-modules/509-extern-block.html), [9.13 the C compiler](../spec/refs-lang/009-tools/913-c-compiler.html), [4.15 unsafe](../spec/refs-lang/004-functions/415-unsafe.html), [4.15.4 unsafe trait](../spec/refs-lang/004-functions/415-unsafe.html#4154-in-traits), [6.2.3 the link attribute](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#623-link-naming-an-external-library), [C-compatible characters](../spec/refs-lang/001-lexical/108-character-literals.html)
- Standard library spec: [FFI](../spec/refs-standard-libraries/100-bare/107-ffi.html), [GC pinning](../spec/refs-standard-libraries/100-bare/106-gc-pin.html), [Pointers](../spec/refs-standard-libraries/100-bare/101-ptr.html), [The `Pod` trait](../spec/refs-standard-libraries/100-bare/103-mem.html#133-the-pod-trait), [The panic hook](../spec/refs-standard-libraries/300-std/312-panic.html)
- API: [bare::ffi](../std/bare/ffi.html), [bare::gc](../std/bare/gc.html), [bare::ptr](../std/bare/ptr.html), [bare::alloc](../std/bare/alloc.html), [bare::mem](../std/bare/mem.html), [std::panic](../std/std/panic.html)
- Next chapter: [15. NABION](15-nabion.html)
