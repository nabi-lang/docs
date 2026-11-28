# 14. FFI 기초

이 장에서는 C로 작성된 함수를 Nabi에서 호출하는 방법, Nabi 함수를 C에 넘기는 방법, Nabi 자체의 C 컴파일러로 C 코드를 컴파일하는 방법을 보여 줍니다. 운영체제와 수많은 라이브러리가 C ABI로 기능을 공개합니다. 그래서 FFI(Foreign Function Interface)를 알면 표준 라이브러리에 없는 기능도 쓸 수 있습니다.

모든 예제는 macOS / Linux의 libc를 상대로 실행됩니다. Windows에서 달라지는 점은 끝에 모아 두었습니다.

## `extern "C"` 블록

C 함수를 쓰려면 먼저 `extern "C" { ... }` 블록에 그 시그니처를 선언합니다. 본체는 없고 선언은 `;`으로 끝납니다. 이름은 C 심볼 이름 그대로입니다.

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

C의 타입은 C 이름으로 씁니다. `c_int`, `c_uint`, `c_long`, `c_ulong`, `c_size_t`, `c_double`과 나머지가 prelude에 있습니다. 각 이름은 타깃에 맞는 크기의 Nabi 수 타입(`c_int`는 `i32`, `c_size_t`는 `usize`)이므로 값이 변환 없이 오갑니다. 고정 크기 타입 대신 이 이름을 쓰세요. C의 `long`은 Linux와 macOS에서 64비트지만 Windows에서는 32비트입니다. 그래서 `fn labs(x: i64) -> i64`는 한쪽에서만 맞고, `fn labs(x: c_long) -> c_long`은 어디서나 맞습니다. `void*`는 `CPtr<c_void>`입니다.

`_Bool`은 `bool`입니다. `_Float16`과 `__bf16`은 `f16`과 `bf16`이고, 8바이트나 16바이트 벡터(`float32x4_t`, `__m128`)는 arm64와 x86-64에서 `Simd`입니다. `str` 매개변수는 NUL로 끝나는 `char*`로 넘어갑니다. 필요하면 Nabi가 NUL로 끝나는 복사본을 만듭니다.

C가 담을 수 있는 타입만 경계를 넘을 수 있고, 컴파일러는 별칭과 struct 필드까지 꿰뚫어 하나하나 검사합니다. 튜플, 유니코드 `char`, 일반 enum, `List`는 대신 쓸 것(`struct`, `u32`, `#[repr(i32)]`, 포인터와 길이)을 알려 주는 에러입니다.

`extern` 블록은 모듈 최상위에만 올 수 있고, ABI 문자열(`"C"`)을 적어야 합니다.

## `unsafe`가 필요한 이유

컴파일러는 C 함수가 시그니처대로 동작하는지 검사할 방법이 없습니다. 잘못된 포인터를 넘기거나 선언을 틀리면 메모리가 망가질 수 있습니다. 그래서 `extern "C"` 블록의 모든 함수는 암묵적으로 `unsafe`이고, `unsafe { ... }` 안에서만 호출할 수 있습니다.

```text check=checks/14-unsafe-call.nabi
error[E0061]: call to unsafe function `abs` outside an `unsafe` block or `unsafe fn`
  --> main.nabi:4:12
   |
 4 | println(f"{abs(-1)}");
   |            ^^^^^^^
   |
   = help: wrap the call in `unsafe { .. }`; every `extern` function, `extern "nabi"` included, is unsafe to call
```

`extern` 블록에 선언한 C 전역 변수(`static mut optind: c_int;`)도 마찬가지로 검사할 수 없습니다. 그래서 읽거나 쓸 때도 `unsafe`가 필요합니다.

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

`unsafe` 밖에서 쓰면 에러이고, 도움말이 고친 형태(`unsafe { optind = 1; }`)를 보여 줍니다. C의 `errno`는 이런 전역 변수가 아닙니다(스레드마다 다른 매크로입니다). 읽는 방법은 [C의 에러](#c의-에러-errno)에 있습니다.

`unsafe` 블록은 "여기서는 내가 안전을 보장한다"는 뜻입니다. 이 블록은 작게 유지하세요. 위험한 호출은 안전한 Nabi 함수로 감싸서 밖에서는 안전한 API만 보이게 하는 편이 좋습니다.

## 문자열

### C 문자와 `CStr`

C의 `char`는 1바이트이고, Nabi에서는 `c'A'` 리터럴을 쓰는 `c_char` 타입입니다. 유니코드 `char`(`'A'`)와는 다른 타입이므로 FFI 경계를 건드리는 코드가 눈에 띕니다.

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

C 함수가 NULL을 반환할 수 있으면 반환 타입을 `ConstPtr<T> | none`으로 선언합니다. 주소 하나이고 NULL이 `none`입니다. (`CPtr<T>`는 C 포인터처럼 NULL을 값으로 담습니다.) C 문자열도 이렇게 돌아옵니다. 바이트가 C의 것인 `char*`입니다. `strerror`의 버퍼는 다음 호출에서 바뀌고, `getenv`의 버퍼는 `setenv` 뒤에 바뀝니다. `CStr::from_ptr`는 NUL까지 바이트를 복사하고, `to_str`는 UTF-8인지 검사합니다. 위험한 부분은 `from_c`에 가두고, 안전한 `env_var`만 드러냅니다.

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

`str` 결과는 에러이고, 도움말이 이 방법을 알려 줍니다. C가 바이트를 언제까지 두는지, 호출자가 해제해야 하는지, UTF-8인지 Nabi로서는 알 수 없기 때문입니다. 바이트로 만든 문자열을 넘기려면 `CString`을 만들고(`try CString::new(bytes)`, 안에 NUL이 있으면 거부), `cs`가 스코프에 있는 동안 `cs.as_ptr()`를 넘깁니다.

### wide 문자열: `WideCString`

C의 `wchar_t` 함수와 Windows의 `W` API는 `c_wchar` 단위의 문자열을 받습니다. `WideCString::new(s)`는 `str`에서 NUL까지 포함해 그런 문자열을 만듭니다. Windows에서는 UTF-16, 그 밖에서는 문자 하나에 단위 하나입니다. `as_ptr()`가 넘길 `const wchar_t*`이고, `to_str()`이 문자열을 되돌려 줍니다. C가 돌려준 것은 `CStr::from_ptr`처럼 `WideCStr::from_ptr(p)`가 복사합니다.

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

## 버퍼

### 리스트 넘기기: `gc::pin`

버퍼를 채우거나 읽는 C 함수는 포인터와 길이를 받습니다. `List`는 GC 힙에 살고, 수집기가 옮길 수 있습니다. 아무리 짧은 호출이라도 모든 C 호출 중에 수집이 돌 수 있으므로, C가 주소를 쓰는 동안 리스트를 pin해야 합니다. `gc::pin(xs)`는 `Pinned` 값이 스코프에 있는 동안 리스트를 제자리에 두고, `as_data_ptr()`가 첫 원소의 주소입니다.

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

컴파일러가 pin을 대신 넣어 주는 일은 없습니다. 시그니처에 `List`를 쓰면 에러이고, 그 help가 쓸 선언과 호출을 보여 줍니다. `ptr: ConstPtr<T>, len: usize`로 선언하고, `let p = gc::pin(xs);`의 `p.as_data_ptr()`와 `xs.len()`을 넘깁니다. `str` 인자는 pin이 필요 없습니다. 호출 동안 Nabi가 제자리에 둡니다.

리스트의 slice도 같은 방법으로 pin합니다(`pin(xs[1:3])`, `as_data_ptr()`는 slice의 첫 원소). `StrBuf`도 그렇습니다(`as_bytes_ptr()`는 첫 바이트).

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

pin된 리스트는 늘어날 수 없습니다. 늘어나면 원소가 새 버퍼로 옮겨지는데, C는 아직 옛 주소를 쥐고 있을 수 있기 때문입니다. pin하기 전에 공간을 잡아 두고, pin이 이어지는 동안 채웁니다.

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

거기서 `ys.push(9)`를 하나 더 하면 더 큰 버퍼가 필요하므로, panic하면서 할 일을 알려 줍니다.

```text
cannot grow a list while it is pinned (C may hold its address): call `xs.reserve(n)` before `gc::pin(xs)`
```

`StrBuf`도 같은 규칙을 따릅니다.

### C가 가진 메모리: `bare::ptr`와 `bare::alloc`

원시 포인터는 `bare` 계층에 있습니다.

| 타입 | C 대응 | null |
|------|--------|------|
| `Ptr<T>`      | `T*`        | 담을 수 없음(null은 `none`으로 표현) |
| `ConstPtr<T>` | `const T*`  | 담을 수 없음 |
| `CPtr<T>`     | `T*`        | 담을 수 있음 |

기본 연산은 포인터로 하는 `read()` / `write()`, `add(n)`(원소 n개만큼 전진), `cast::<U>()`입니다. C 힙의 메모리는 `bare::alloc`으로 받고 직접 돌려줍니다. 이 메모리는 GC가 관리하지 않으므로 `dealloc`을 잊으면 새어 나갑니다.

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

C와 struct를 주고받을 때는 레이아웃이 C의 것임을 밝히도록 `#[repr(C)]`를 붙입니다(기본 레이아웃이 이미 그렇습니다). `union`도 같은 방식으로 넘어갑니다. C가 멤버를 보고 넘기듯 Nabi도 그렇게 넘깁니다.

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

## 컴파일러가 검사할 수 없는 약속: `unsafe trait`

어떤 트레잇은 그것을 구현하는 모든 타입에 대해 컴파일러가 검사할 방법이 없는 약속을 합니다. 그런 트레잇은 `unsafe trait`로 선언하고, 그 impl은 모두 `unsafe impl`로 씁니다. 호출에 쓰는 `unsafe { }` 블록처럼, 이 `unsafe`는 약속이 지켜진다는 작성자의 보증입니다. 트레잇을 쓰는 쪽(바운드, 호출)에는 `unsafe`가 필요 없습니다.

표준 라이브러리에서는 `bare::mem::Pod`("plain old data")가 그렇습니다. 그 타입의 모든 비트 패턴이 값이라는 뜻입니다. 정수, 부동소수점, `c_char`는 이미 `Pod`입니다. `reinterpret`은 한 `Pod` 값의 비트를 같은 크기의 다른 `Pod` 값으로 읽습니다. `Pod` 필드로 된 `#[repr(C)]` struct는 `#[derive(Pod)]`로, 또는 직접 `unsafe impl Pod for T {}`로 참여합니다. 어느 쪽이든 컴파일러는 레이아웃과 필드, `Drop`이 없는지를 여전히 검사합니다.

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

`unsafe trait`를 그냥 `impl`로 구현하면 에러이고, 적어야 할 형태를 알려 줍니다([E0078](../errors/E0078.html)).

```text check=checks/14-unsafe-impl.nabi
error[E0078]: `Zeroable` is an `unsafe trait`: the impl promises what the compiler cannot check; write `unsafe impl Zeroable for i32`
  --> main.nabi:5:1
   |
 5 | impl Zeroable for i32 {
   | ^^^^
```

## 콜백: `extern "C" fn`

`qsort`처럼 함수 포인터를 받는 C 함수에는 Nabi 함수를 C ABI로 컴파일해 넘깁니다. 함수 정의 앞에 `extern "C"`만 붙이면 됩니다. 매개변수와 반환 타입은 정수, 부동소수점, 포인터, `#[repr(C)]` struct 같은 FFI 호환 타입이어야 합니다.

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

여기서 `str` 타입은 FFI 타입이 아니므로 `extern "C" fn`의 매개변수나 결과가 될 수 없습니다. 컴파일러가 대신 `ConstPtr<c_char>`를 받아 `CStr::from_ptr`로 복사하라고 알려 줍니다.

스레드 풀이나 오디오 라이브러리처럼 C 코드가 직접 만든 스레드에서 `extern "C" fn`을 부를 수도 있습니다. Nabi는 그런 스레드를 첫 호출 때 등록하므로, 콜백 안에서 메모리를 할당하고 어떤 Nabi 코드든 쓸 수 있습니다. C 쪽에서 따로 준비할 것은 없습니다.

`extern "C" fn`은 클로저가 아닙니다. 주변 변수를 캡처할 수 없고, C가 함수를 받는 자리에 클로저를 넘기면 에러입니다. 그 help가 다음 절을 가리킵니다.

## Handle: C가 쥐는 Nabi 값

콜백하는 C API는 보통 함수 포인터 옆에 `void* user_data`를 받아 호출마다 넘겨줍니다. `Handle::new(value)`(`bare::ffi`)는 Nabi 값을 그런 포인터로 바꿉니다. `h.as_ptr()`가 넘길 `void*`이고, 콜백은 `Handle::<T>::from_ptr(p).get()`으로 값을 되찾습니다. `List`나 `Dict`는 같은 객체로 돌아오므로 콜백이 그것을 채울 수 있습니다. handle은 `h.release()`까지 값을 살려 두고 제자리에 둡니다.

클로저도 값입니다. C에서 클로저를 부르려면 `fn` 타입으로 handle에 담고, 그것을 다시 읽어 부르는 한 줄짜리 `extern "C" fn`을 넘깁니다.

아래의 `each`는 받은 포인터로 콜백하는 C 라이브러리를 대신합니다.

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

포인터는 주소가 아니라 handle 번호입니다. C는 그것을 넘겨주기만 합니다. `Handle`은 포인터 크기이고 `Copy`이므로 `#[repr(C)]` struct의 필드가 될 수 있습니다. C가 포인터를 다 쓰면 `release`를 부릅니다. C가 복사본을 하나 더 두면 `retain`이 `release`를 하나 더 요구합니다. 실수하면 쓸 것을 알려 주며 panic합니다. 마지막 `release` 뒤의 `get`은 `handle already released ...`라고 하고, handle을 만들 때와 다른 타입의 `from_ptr`는 `` this handle holds `List<str>`, not `Dict<str, i64>`: write `Handle::<List<str>>::from_ptr(p)` ``라고 합니다.

### C가 연 것 닫기: `Drop`

GC는 Nabi 메모리를 해제하지만 finalizer는 돌리지 않습니다. 그래서 C가 내준 자원(`FILE*`, `malloc`한 블록, 라이브러리 handle)은 `Drop`을 구현한 래퍼 타입이 놓아줍니다. 컴파일러는 값의 주인이 스코프를 떠나는 곳, 또는 그 값을 묶은 `using` 블록의 끝에서 `drop`을 부릅니다.

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

## C의 에러: `#[errno]`

많은 C 함수가 `errno`로 실패를 알립니다. `extern` 블록(또는 선언 하나)에 `#[errno]`를 붙이면, 호출마다 돌아오자마자 다른 무엇이 바꾸기 전에 errno를 저장하고, `bare::ffi::errno()`가 저장된 값을 읽습니다. `io::IoError::last_os_error()`는 그 값을 OS의 종류와 메시지를 담은 `IoError`로 바꾸므로, 래퍼가 그것을 `throw`할 수 있습니다.

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

`#[errno]`가 없으면 아무것도 저장하지 않으므로, `errno()`에는 더 앞선 호출의 값이 남아 있습니다. 호출 전의 `set_errno(0)`은 `strtol`처럼 errno로만 에러를 알리는 C 함수를 위한 것입니다.

## 콜백 안의 panic

panic은 C가 부른 함수를 C의 프레임 너머로 떠나면 안 됩니다. C는 그것을 전혀 모르고, exit하면 망가진 상태 위에서 호스트의 정리 코드가 돕니다. 그래서 C가 부른 `extern "C" fn`에서 난 panic은 보고를 출력하고 한 줄을 더 쓴 뒤, 101로 끝나는 대신 프로세스를 abort합니다(Unix에서 종료 상태 134).

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

계속 가려면 함수가 panic할 때 C에 돌려줄 값을 적습니다. `#[on_panic(return = v)]`이고, `v`는 숫자, `true` / `false`, null 포인터면 `0`입니다(결과가 없는 함수는 `#[on_panic]`만). panic은 여전히 보고되고, 그다음 C가 `v`를 받습니다. `std::panic::set_hook`은 그보다 먼저 모든 panic에서 내 함수를 돌려 기록을 남기게 합니다.

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

panic과 함수 사이의 코드는 건너뛰고, 그 `Drop`은 돌지 않습니다. 해제했어야 할 것이 새고, 거기서 잠근 `Mutex`는 잠긴 채로 남습니다. lock을 쥔 동안 실패할 수 있는 콜백은 panic하지 말고 에러 값을 돌려주세요. wasm32에서는 콜백 안의 panic이 언제나 abort합니다.

## 실행 중에 라이브러리 불러오기: `Library`

`#[link]`(아래)는 프로그램을 링크할 때 라이브러리를 묶습니다. 플러그인 시스템처럼 실행 중에 고르려면 `bare::ffi::Library::open(path)`로 불러오고, `symbol::<F>(name)`으로 함수를 C 함수 포인터 타입 `F`로 찾습니다. 파일이나 심볼이 없으면 둘 다 시스템이 준 이유와 함께 던집니다.

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

`F`가 함수의 진짜 시그니처여야 하므로 `symbol`은 `unsafe`입니다. C 함수 포인터가 아닌 타입은 panic합니다. 라이브러리는 `close`할 때까지 불러온 채로 남습니다. `close`도 `unsafe`입니다. 닫은 뒤에는 거기서 얻은 모든 함수 포인터가 매달리기 때문입니다. `Library::this()`는 프로그램 자신이고, 함께 링크된 C 라이브러리도 담습니다.

## C로 내보내기: `#[export]`

C가 Nabi 함수를 이름으로 찾게 하려면 `pub extern "C" fn`으로 정의하고 `#[export]`를 붙입니다. 그러면 심볼이 함수 자신의 이름입니다. 다른 이름으로 내보내려면 `#[export_name("sym")]`을 씁니다. 아래 예제는 같은 프로그램 안에서 두 심볼을 C 함수로 다시 선언해 호출합니다.

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

`extern "C"`가 없이 내보낸 함수는 C가 쓰지 않는 Nabi 고유의 호출 규약을 쓰게 되므로, 쓸 것을 알려 주는 에러입니다: `` `compute` is exported to C but uses the Nabi ABI: write `pub extern "C" fn compute(..)` ``.

### C를 위한 공유 라이브러리: `--crate-type=cdylib`

반대 방향도 됩니다. C 코드나 `ctypes`를 쓰는 Python이 Nabi로 쓴 라이브러리를 불러옵니다. 그쪽에서 부를 함수를 패키지의 `src/lib.nabi`에 `#[export] pub extern "C" fn`으로 두고 `--crate-type=cdylib`으로 빌드합니다.

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

파일은 Linux에서 `libgeo.so`, macOS에서 `libgeo.dylib`, Windows에서 `geo.dll`(import 라이브러리 `geo.lib`와 함께)입니다. 밖에서 보이는 것은 내보낸 함수뿐입니다. 라이브러리에는 표준 라이브러리와 자체 런타임이 들어 있고, 런타임은 라이브러리를 불러올 때 시작합니다. 그래서 C 쪽은 어느 스레드에서든 먼저 부를 것이 없습니다. 오가는 값은 C 값입니다. 숫자, 포인터, `#[repr(C)]` struct입니다. 라이브러리 안의 panic은 다른 콜백에서처럼 호스트 프로세스를 abort합니다. panic할 수 있는 내보낸 함수에는 호스트가 확인할 수 있는 `#[on_panic(return = v)]`를 붙이고, `std::panic::set_hook`으로 기록을 남기세요.

`nabi.nabion`에 `lib: { kind: "cdylib" }`를 두면 `nabi build`마다 공유 라이브러리를 만듭니다. 이때 패키지의 `version`을 씁니다. Linux에서 라이브러리의 soname은 0.3.x 버전이면 `libgeo.so.0.3`(1.0부터는 `libgeo.so.1`)이고, 옆에 그 이름의 링크가 생깁니다. macOS에서는 라이브러리의 현재 버전과 호환 버전이 됩니다.

```nabion
Package {
    name: "geo",
    version: "0.3.1",
    lib: { kind: "cdylib" },
}
```

빌드는 C 헤더 `target/debug/include/geo.h`도 씁니다. 내보낸 함수마다 선언하고, 그 함수들이 쓰는 `#[repr(C)]` struct와 `#[repr(i32)]` enum, `///` 주석도 담습니다. C는 손으로 쓴 사본 대신 이것을 include하므로 둘이 어긋날 일이 없습니다.

```text
$ cat target/debug/include/geo.h
...
int64_t geo_sum_squares(int64_t n);
...
$ cc -I target/debug/include main.c -L target/debug -lgeo -o main
```

`nabic build --emit=c-header`는 같은 헤더만 씁니다.

### C와 Swift를 위한 정적 라이브러리 하나: `--crate-type=staticlib`

공유 라이브러리는 프로그램 옆에 함께 배포해야 합니다. Nabi 코드를 프로그램 안에 링크하려면 대신 정적 라이브러리 하나를 빌드하세요. `--crate-type=staticlib`은 패키지, 의존성, 패키지가 쓰는 표준 라이브러리, 런타임을 `target/debug/static/libgeo.a`에 담고, 그 옆에 헤더와 module map을 씁니다. C는 그 아카이브 하나만 링크하면 됩니다.

```text
$ nabi build --crate-type=staticlib
$ cc -I target/debug/include main.c target/debug/static/libgeo.a -o main
```

런타임은 처음 export가 불릴 때, 그 호출이 일어난 스레드에서 시작합니다. `target/debug/include/module.modulemap`은 헤더를 clang 모듈로 만들므로 Swift에서 `import geo`로 쓰고 라이브러리도 링크합니다. iOS 앱은 이렇게 Nabi 코드를 받습니다([28장](28-ios.html)). `nabi.nabion`에 `lib: { kind: "staticlib" }`를 두면 `nabi build`마다 만듭니다.

## 라이브러리 링크: `#[link]`

libc 함수는 따로 할 일 없이 링크됩니다. 다른 라이브러리를 쓰려면 `extern` 블록에 `#[link("name")]`을 붙입니다. 링커에 `-lname`을 넘기는 것과 같습니다.

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

정적 라이브러리를 고르려면 `#[link(name = "foo", kind = "static")]`으로 씁니다.

libc와 libm은 언제나 링크됩니다. 그 밖의 라이브러리는 아래의 zlib처럼 `#[link]`가 필요합니다. C 소스로 가진 라이브러리는 먼저 빌드해 둘 필요가 없습니다. 패키지가 컴파일합니다([C 소스](#c-소스-nabi-cc)).

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

C API가 크면 `nabi bindgen libfoo.h > libfoo_sys.nabi`가 헤더에서 선언을 써 줍니다. C 컴파일러는 필요 없습니다. 출력은 한 타깃(`--target`, 기본은 호스트)의 것이고, 각 C 타입은 그 타깃의 크기입니다. C의 정수 타입은 C 이름으로 쓰므로(`int`는 `c_int`, `long`은 `c_long`) 크기가 다른 타깃에서도 선언이 맞고, `void *`는 `CPtr<c_void>`로 씁니다.


## 기계에 설치된 라이브러리: `Dep::system`

`#[link("z")]`는 링커가 늘 보는 디렉터리에서만 `libz`를 찾습니다. 패키지 관리자로 설치한 라이브러리는 다른 곳에 있는 일이 많습니다. Homebrew는 많은 라이브러리를 `/opt/homebrew/opt/<name>`에, vcpkg는 자기 트리에 둡니다. `nabi.nabion`에 `Dep::system`으로 적으면 `nabi`가 빌드마다 찾습니다. pkg-config `.pc` 파일을 직접 읽고(`pkg-config` 프로그램은 필요 없음), 그다음 Homebrew, vcpkg, macOS SDK를 봅니다.

```nabion
Package {
    name: "app",
    native: {
        "z": Dep::system(pkg_config: "zlib", brew: "zlib", apt: "zlib1g-dev"),
        "sqlite3": Dep::system(brew: "sqlite", apt: "libsqlite3-dev"),
    },
}
```

코드는 `#[link]` 없이 평소처럼 함수를 선언합니다.

```text
extern "C" {
    fn crc32(crc: u64, buf: str, len: u32) -> u64;
    fn sqlite3_libversion_number() -> i32;
}
```

설치되어 있지 않으면 빌드가 이 기계에서 설치하는 방법을 알려 줍니다.

```text
$ nabi build
nabi: error: system library `z` (pkg-config `zlib`) not found for x86_64-unknown-linux-gnu (looked in pkg-config (zlib.pc)): install it with `sudo apt install zlib1g-dev` (or your distribution's package of it), or set PKG_CONFIG_PATH to the directory holding zlib.pc
```

## Apple 프레임워크

macOS에서 시스템 API는 프레임워크로 옵니다. `kind = "framework"`나 `nabi.nabion`의 `Dep::framework()`로 링크하고 C 함수를 부릅니다.

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

Linux나 Windows용 빌드에서 `Dep::framework()`는 빠집니다. 거기서 `#[link(kind = "framework")]`는 에러이므로, 그런 블록은 `#[cfg(target_os = "macos")]` 아래에 둡니다.


## C 소스: `nabi cc`

Nabi에는 자체 C 컴파일러가 있습니다. 그래서 소스로 가진 C 코드는 다른 컴파일러가 필요 없고, 먼저 라이브러리로 빌드해 둘 필요도 없습니다. 컴파일러는 `nabi` 옆에 `nabicc`로 설치되고, `nabi cc`는 같은 컴파일러를 실행합니다. gcc와 clang이 받는 옵션을 받고, 타깃의 C 헤더를 찾아, Nabi의 코드 생성기와 링커로 컴파일합니다. 호출 규약은 `extern "C"`가 쓰는 C ABI와 같습니다.

### C 파일 컴파일하기

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

`-o`가 없으면 프로그램은 `a.out`(Windows는 `a.exe`)입니다. `-c`는 오브젝트(`hello.o`, Windows는 `hello.obj`)에서 멈추고, `-E`는 전처리한 소스를 출력하며, `-g`는 gdb, lldb, WinDbg용 디버그 정보를 넣습니다. 기본 표준은 `gnu17`이고, `-std=c99`, `-std=c11`, `-std=c23`은 `__STDC_VERSION__`을 그에 맞춥니다.

### make와 CMake에서

명령줄이 gcc의 것이므로 빌드 시스템은 두 이름 중 어느 것이든 C 컴파일러로 받습니다.

```bash
make CC="nabi cc"
make CC=nabicc
cmake -DCMAKE_C_COMPILER=nabicc -B build && cmake --build build
```

경고 옵션(`-Wall`, `-Wextra`), `-fPIC`, `-pthread`, 그 밖의 `-f`와 `-m` 옵션은 받되 아무것도 바꾸지 않습니다. 코드는 언제나 위치 독립적입니다. `-MD`와 `-MMD`는 make가 헤더를 추적하는 `.d` 파일을 씁니다.

### 패키지 안의 C

패키지는 C 라이브러리를 소스로 담아 Nabi에서 부를 수 있습니다. 그 파일을 `nabi.nabion`의 `c` 필드에 적으면, `nabi build`가 내장 컴파일러로 빌드의 최적화 수준에 맞춰 컴파일하고 패키지와 함께 링크합니다.

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

`src/main.nabi`는 다른 C 라이브러리와 마찬가지로 `extern "C"` 블록에 C 함수를 선언합니다.

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

`sources`와 `include`는 `nabi.nabion` 기준 상대 경로입니다. `define`은 `-D` 옵션이 되고, `std`는 `-std=`입니다(기본 `gnu17`). `flags`는 `-O*`, `-g`, `-W*`, `-f*`, `-m*`만 받습니다. 거기 적은 `-O`는 C 코드에 대해 빌드의 수준을 대신합니다. `flags`에 그 밖의 것이 있으면 받는 옵션을 나열하는 에러입니다. toolchain 밖의 것은 아무것도 실행하지 않습니다. 빌드 스크립트도 외부 C 컴파일러도 없습니다.

헤더의 `static inline` 함수에는 링크할 심벌이 없습니다. `nabi bindgen --c-shim`은 함수마다 C 래퍼를 쓰고 그 래퍼를 바인딩합니다. 래퍼 파일을 `sources`에 더하세요.

```text
$ nabi bindgen vendor/area.h -o src/area.nabi --c-shim vendor/area_shim.c
```

```nabion
    c: {
        sources: ["vendor/area.c", "vendor/area_shim.c"],
        include: ["vendor"],
    },
```

### 크로스 컴파일

`--target`은 Nabi 코드에서와 마찬가지로 C에서도 동작합니다. Nabi의 타깃 이름과, gcc와 clang이 쓰는 트리플을 받습니다.

```text
$ nabi cc --target=aarch64-unknown-linux-gnu hello.c -o hello-arm64
$ nabi cc --target=x86_64-linux-gnu -c hello.c
$ nabi build --target=wasm32-wasip2
```

마지막 명령은 패키지의 C도 wasm32용으로 컴파일합니다. Linux glibc와 wasm32 타깃은 더 필요한 것이 없습니다. toolchain이 그 C 헤더와 C 라이브러리를 담고 있기 때문입니다(musl 타깃은 `--sysroot`로 헤더를 주세요). Apple 타깃은 macOS SDK를, Windows 타깃은 Windows 호스트의 Windows SDK와 Visual Studio 헤더를 씁니다.

### 컴파일하지 않는 것

어떤 C는 아직 Nabi의 코드 생성기에 대응하는 것이 없습니다. 다음은 에러이고, 각 에러는 대신 무엇을 쓸지 알립니다.

- 인라인 어셈블리(빈 `asm("")` 배리어는 괜찮습니다), `&&label`과 계산된 `goto *p`
- 가변 길이 배열과 `alloca`
- `_Complex` 산술, `__int128`, `double`보다 넓은 `long double`, `_Float16`과 `__bf16` 산술
- 스레드 지역 변수(`_Thread_local`, `__thread`)
- 벡터 타입과 SIMD intrinsic. `__SSE2__`, `__ARM_NEON` 같은 벡터 매크로를 정의하지 않으므로 라이브러리는 이식 가능한 C 경로를 탑니다
- K&R 형식 정의와 선언하지 않은 함수의 호출
- `-shared`, `.s` / `.S` 소스, C++나 Objective-C 소스

atomic builtin(`__atomic_*`, `__sync_*`)은 지금은 일반 load와 store로 컴파일되므로 스레드 사이에서 원자적이지 않습니다. 그때까지는 스레드 사이에 메모리를 공유하는 C 코드를 `nabi cc`로 컴파일하지 마세요.

## Windows에서

- Windows에서 `c_long`과 `c_ulong`은 32비트입니다. C의 `long`에 `c_long`을 쓴 코드는 그대로 빌드되고, `i64`를 쓴 코드는 거기서 C 쪽과 맞지 않습니다.
- Windows API 함수는 `extern "system"`을 씁니다. 32비트 x86에서는 stdcall이고, 그 밖에서는 `"C"`입니다.
- `W` 함수는 UTF-16 문자열을 받습니다. `WideCString::as_ptr()`를 넘기고, 함수가 채운 버퍼(pin한 `List<u16>`)는 `str::from_utf16`으로 읽습니다.
- Windows 블록의 `#[errno]`는 `GetLastError`도 저장합니다. `bare::ffi::last_error()`가 그것을 읽고, `io::IoError::last_os_error()`는 errno보다 그것을 먼저 씁니다.
- `Library::open`은 `LoadLibraryW`로 DLL(`"kernel32.dll"`)을 불러오고, `symbol::<extern "system" fn(..) -> R>`이 그 함수를 줍니다.

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

## 정리

- C 함수는 `extern "C" { fn name(..) -> T; }`로 선언하고 `unsafe { }` 안에서 호출합니다. C의 타입은 `c_int`, `c_long`, `c_size_t` 등으로 쓰고, `void*`는 `CPtr<c_void>`입니다.
- C의 `char`는 `c_char` / `c'A'`이고, NULL일 수 있는 포인터는 `ConstPtr<T> | none`이며, C 문자열은 `CStr::from_ptr`로 복사합니다. `wchar_t` 문자열은 `WideCString`입니다.
- 주소를 넘기는 모든 호출에서 리스트(또는 slice, `StrBuf`)를 `gc::pin`으로 pin합니다. pin된 리스트는 늘어날 수 없으므로 먼저 `reserve`합니다.
- C 메모리는 `Ptr` / `ConstPtr`의 `read` / `write` / `add` / `cast`와 `bare::alloc`으로 다루고, C 자원은 `Drop` 래퍼에서 놓아줍니다.
- struct는 `#[repr(C)]`, 콜백은 `extern "C" fn`을 쓰고, C가 대신 들고 다니는 값(리스트, 클로저)은 `Handle`에 담습니다.
- `#[errno]`는 `errno()`와 `IoError::last_os_error()`를 위해 errno를 저장합니다.
- C가 부른 함수의 panic은 abort합니다. `#[on_panic(return = v)]`는 대신 `v`를 돌려주고, `std::panic::set_hook`은 모든 panic을 봅니다.
- `Library::open`은 실행 중에 공유 라이브러리를 불러오고, `#[link]`는 빌드할 때 링크합니다.
- `#[export] pub extern "C" fn`은 함수를 C로 내보냅니다.
- `nabi build --crate-type=cdylib`(또는 `lib: { kind: "cdylib" }`)은 C나 Python이 불러오는 공유 라이브러리와 그 C 헤더를 만듭니다.
- `Dep::system(..)`은 설치된 C 라이브러리를 찾고, `Dep::framework()`와 `kind = "framework"`는 Apple 프레임워크를 링크합니다.
- `nabi cc`(`nabicc`도)는 gcc의 옵션으로 C를 컴파일하고, make에서는 `CC="nabi cc"`로 씁니다. `nabi.nabion`의 `c` 필드는 패키지의 C 소스를 그것으로 컴파일합니다.
- 위험한 호출은 작은 `unsafe` 함수에 가두고, 밖에는 안전한 함수를 드러냅니다.
- `unsafe trait`(`Pod` 등)는 `unsafe impl`로 구현합니다. `#[repr(C)]` struct에는 `#[derive(Pod)]`가 그 impl을 써 줍니다.

## 더 읽을거리

- 사양: [5.9 extern 블록](../spec/refs-lang/005-modules/509-extern-block.html), [9.13 C 컴파일러](../spec/refs-lang/009-tools/913-c-compiler.html), [4.15 unsafe](../spec/refs-lang/004-functions/415-unsafe.html), [4.15.4 unsafe trait](../spec/refs-lang/004-functions/415-unsafe.html#4154-트레잇-안에서), [6.2.3 link 속성](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#623-link-외부-라이브러리-명시), [C 호환 문자](../spec/refs-lang/001-lexical/108-character-literals.html)
- 표준 라이브러리 사양: [FFI](../spec/refs-standard-libraries/100-bare/107-ffi.html), [GC pinning](../spec/refs-standard-libraries/100-bare/106-gc-pin.html), [포인터](../spec/refs-standard-libraries/100-bare/101-ptr.html), [`Pod` 트레잇](../spec/refs-standard-libraries/100-bare/103-mem.html#133-pod-트레잇), [panic hook](../spec/refs-standard-libraries/300-std/312-panic.html)
- API: [bare::ffi](../../std/bare/ffi.html), [bare::gc](../../std/bare/gc.html), [bare::ptr](../../std/bare/ptr.html), [bare::alloc](../../std/bare/alloc.html), [bare::mem](../../std/bare/mem.html), [std::panic](../../std/std/panic.html)
- 다음 장: [15. NABION](15-nabion.html)
