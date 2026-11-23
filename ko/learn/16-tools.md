# 16. 도구

이 장에서는 코드를 쓰면서 늘 곁에 두는 도구를 둘러봅니다. 포매터, 린터, 테스트와 벤치마크, 문서 생성, 에디터 지원(LSP), REPL과 Jupyter, 그리고 컴파일러 안을 들여다보는 디버그 옵션입니다. 모두 `nabi` 명령 하나에서 시작합니다.

이 장의 출력은 아래 패키지를 실제로 실행해 얻은 것입니다. `nabi new mathx`로 만들고 `src/main.nabi`를 다음과 같이 바꿨습니다.

```nabi build
//! Small math helpers.

/// Greatest common divisor of `a` and `b`.
pub fn gcd(a: i64, b: i64) -> i64 {
    if b == 0 { a } else { gcd(b, a % b) }
}

/// The `n`-th Fibonacci number.
pub fn fib(n: i64) -> i64 {
    if n < 2 { return n; }
    fib(n - 1) + fib(n - 2)
}

println(f"gcd(12, 18) = {gcd(12, 18)}, fib(10) = {fib(10)}");

#[test]
fn gcd_works() {
    assert_eq(gcd(12, 18), 6);
    assert_eq(gcd(7, 0), 7);
}

#[test]
fn fib_works() {
    assert_eq(fib(10), 55);
}

#[test]
fn fib_wrong() {
    assert_eq(fib(5), 6, "deliberately wrong");
}

#[bench]
fn fib_20(b: bench::Bench) { b.iter(|| fib(20)); }
```

## `nabi test`: 테스트

`#[test]`를 붙인 함수가 테스트입니다. 테스트 함수는 보통 빌드에서 빠지고, `nabi test`가 테스트 함수를 모아 따로 빌드합니다. 검사는 `assert`, `assert_eq`, `assert_ne`로 합니다. 검사가 실패하면 panic이 일어납니다. 마지막 인자로 메시지를 줄 수 있습니다.

각 테스트는 자기 프로세스에서 실행되므로 하나가 panic으로 끝나도 나머지는 계속 실행됩니다. 위 패키지의 `fib_wrong`은 일부러 틀리게 만든 것입니다.

```text
$ nabi test
   Compiling mathx (.)
    Finished in 0.69s

running 3 tests
test fib_works ... ok
test fib_wrong ... FAILED
test gcd_works ... ok

failures:

---- fib_wrong ----
thread 'main' panicked at src/main.nabi:29:5:
assertion `left == right` failed: deliberately wrong
  left: 5
 right: 6
(panicked (exit 101))

failures:
    fib_wrong

test result: FAILED. 2 passed; 1 failed; 0 ignored; 0 filtered out; finished in 0.22s
```

하나라도 실패하면 종료 코드가 1이므로 CI에서 그대로 쓸 수 있습니다. 이름의 일부(또는 `^`, `$`, `.`, `*`, `+`, `?`를 쓰는 작은 정규식)를 인자로 주면 맞는 테스트만 실행합니다.

```text
$ nabi test gcd
   Compiling mathx (.)
    Finished in 0.37s

running 1 test
test gcd_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 2 filtered out; finished in 0.22s
```

통과한 테스트의 출력도 보려면 `--show-output`을, 동시에 실행할 테스트 수를 정하려면 `--threads=N`을 씁니다. CI 대시보드 같은 도구에는 `--format json`을 씁니다. 표준 출력에 테스트마다 하나, 마지막에 요약 하나씩 JSON 객체를 한 줄에 하나씩 쓰고, 사람이 읽는 글은 표준 에러로 보냅니다.

```text
$ nabi test --format json 2>/dev/null
{"schema":1,"type":"case","kind":"test","name":"fib_works","result":"ok","ms":87.057}
{"schema":1,"type":"case","kind":"test","name":"fib_wrong","result":"failed","ms":87.144,"reason":"panicked (exit 101)"}
{"schema":1,"type":"case","kind":"test","name":"gcd_works","result":"ok","ms":87.05}
{"schema":1,"type":"summary","kind":"test","result":"failed","passed":2,"failed":1,"ignored":0,"filtered_out":0,"ms":87.276}
```

`--exact`를 주면 필터가 이름 전체와만 맞고, `--skip <pat>`은 맞는 테스트를 뺍니다(여러 번 주면 여럿을 뺍니다). `--list`는 아무것도 실행하지 않고 이름만 출력합니다.

### panic을 기대하는 테스트, 무시하는 테스트, 시간 제한

panic해야 하는 테스트에는 `#[should_panic]`을 붙입니다. `expected`를 주면 panic 메시지에 그 글자도 들어 있어야 합니다. 느린 테스트나 CI에 없는 것이 필요한 테스트에는 `#[ignore]`를 붙이고, 이유를 적을 수 있습니다.

```nabi norun
#[test]
#[should_panic(expected = "deliberately")]
fn fib_wrong_panics() {
    assert_eq(fib(5), 6, "deliberately wrong");
}

#[test]
#[ignore = "takes a minute"]
fn fib_large() {
    assert(fib(90) > 0);
}
```

```text
test fib_large ... ignored, takes a minute
test fib_wrong_panics ... ok
```

`nabi test --ignored`는 무시한 테스트만, `--include-ignored`는 나머지와 함께 그 테스트도 실행합니다. 멈춘 테스트가 전체 실행을 멈추지는 않습니다. 600초가 지나면 강제 종료되고 실패로 보고됩니다. 다른 제한은 `--timeout=SECS`(또는 `NABI_TEST_TIMEOUT`)로 정하고, `--timeout=0`이면 제한이 없습니다.

### 문서 테스트

라이브러리에서는 `nabi test`가 문서 속 코드 예제도 실행합니다. `///`, `//!` 주석과 `README.md`의 ```` ```nabi ```` 블록마다, 라이브러리를 쓰는 작은 프로그램으로 빌드해 실행합니다. `use mathx::*;`를 쓴 것처럼 라이브러리의 공개 아이템을 이름만으로 쓸 수 있습니다. 더는 컴파일되지 않거나 `assert`가 실패하는 예제는 테스트 실행을 실패시키므로 문서가 늘 맞게 유지됩니다.

````text
/// The greatest common divisor.
///
/// ```nabi
/// # let (a, b) = (12, 18);
/// assert(mathx::gcd(a, b) == 6);
/// ```
pub fn gcd(a: i64, b: i64) -> i64 {
    // ...
}
````

`# `로 시작하는 줄은 실행되지만 `nabi doc`에는 보이지 않으므로, 예제에서 준비 코드를 감출 수 있습니다. fence에서 `nabi` 뒤의 단어가 검사 내용을 바꿉니다. `nabi,norun`은 빌드만 하고(파일을 읽거나 네트워크를 기다리는 코드), `nabi,should_panic`은 panic을 기대하고, `nabi,compile_fail`은 빌드되지 않기를 기대하며(특정 에러는 `compile_fail,E0040`), `nabi,ignore`는 건너뜁니다. 다른 언어(```` ```text ````, ```` ```nabion ````)는 실행하지 않습니다.

```text
running 2 tests
test doc README.md:9 ... ok
test doc src/lib.nabi:14 (gcd) ... ok
```

문서 테스트의 이름은 파일과 줄, 그리고 문서화하는 아이템입니다. 예제 안의 에러는 소스 파일의 그 줄을 가리킵니다. `nabi test --doc`은 문서 테스트만 실행하고, `--no-doc`은 문서 테스트를 건너뜁니다.

## `nabi bench`: 벤치마크

`#[bench]` 함수는 `Bench`를 받아, 시간을 잴 코드를 `b.iter`에 넘깁니다. `nabi bench`는 이 함수들을 최적화 빌드(`-O2`)로 만들고, 각 클로저를 여러 번 호출해 호출당 중앙값 시간과 그 편차를 출력합니다. `b.iter`를 한 번도 호출하지 않는 벤치마크에는 경고가 나옵니다. 수치는 기계와 실행마다 다릅니다.

```text
$ nabi bench
   Compiling mathx (.)
    Finished in 0.42s

benchmarking 1 bench
  fib_20      12.99 us +/- 120.52 ns

bench result: ok. 1 run; 0 failed; 0 filtered out; finished in 2.63s
```

`--iters=N`은 표본당 호출 횟수(기본값은 100ms쯤 걸리는 횟수), `--samples=N`은 표본 수(기본값 10)입니다. 필터는 `nabi test`와 같이 동작합니다. 출력은 터미널에서나 파이프에서나 같습니다. 결과를 읽는 도구는 `--format=nabibench`를 주면 벤치마크마다 `#nabibench` 줄 하나를 받고, `--format json`을 주면 `nabi test`와 같은 JSON 줄을 받습니다. 벤치마크마다 `median_ns`, `mad_ns`, `samples`가 들어 있습니다.

## 가비지 컬렉터 일시 정지

가비지 컬렉터는 짧은 일시 정지 동안 모든 스레드를 멈춥니다. 환경 변수 두 개로 이 일시 정지를 볼 수 있습니다. `NABIR_GC_LOG=1`은 수집마다 stderr에 한 줄을 쓰고, `NABIR_GC_STATS=1`은 프로그램이 끝날 때 요약을 씁니다.

```text
$ NABIR_GC_LOG=1 ./target/release/server
nabi gc: #41 minor: 0.802 ms pause, 21046 objects marked, 141312 -> 104160 KiB; ttsp 9 us, mark 410 us, sweep 352 us, evac 0 us, threads 3, at 1520.114 ms
```

일시 정지는 컬렉터가 스레드에 멈추라고 요청한 순간부터 셉니다. `ttsp`(time to safepoint)는 그중 스레드들이 멈추기를 기다린 시간입니다. `ttsp`가 길면 컬렉터가 느린 것이 아니라 어떤 스레드가 계속 돌고 있었다는 뜻입니다. 긴 문자열 검색과 세기, UTF-8 검사, 큰 복사는 일시 정지를 붙잡지 않습니다. 이런 작업 옆에서는 컬렉터가 함께 돕니다. `NABIR_GC_STATS=1`의 요약 끝에는 일시 정지 종류(minor, full, 동시에 mark하는 full 수집의 시작과 끝)마다 p50, p90, p99와 가장 긴 값을 담은 줄이 있습니다.

프로그램은 같은 카운터를 `bare::gc::stats()`로 읽습니다. 지표를 내보내거나 프레임 예산을 확인할 때 씁니다. 컬렉터의 lock을 잠깐 잡으므로 자주 도는 루프가 아니라 가끔 읽으세요.

```nabi
use bare::gc;

let s = gc::stats();
println(f"{s.collections} collections, longest pause {s.pause_max_ns / 1000} us");
```

`bare::gc::collect()`는 full 수집을 하고 그동안 모든 스레드를 멈춥니다. 프레임 사이에 "정리"하려고 부르면 프레임마다 더 길어집니다. 일시 정지를 프레임 밖에 두려면 `collect_young`, 일시 정지 목표, `no_collect`, arena를 씁니다([25장](25-gc-pauses.html)).

## 다른 타깃

`--target`을 주면 `nabi test`, `nabi bench`, `nabi run`은 빌드한 프로그램을 러너를 거쳐 이 기계에서 실행합니다. `wasm32-wasip2`와 `wasm32-wasip1-threads`는 `wasmtime`, `wasm32-unknown-unknown`은 `node`, Linux에서 CPU가 다른 Linux 타깃은 `qemu-<cpu>`(qemu-user)를 씁니다. 모두 `PATH`에 있어야 하고, 없으면 무엇을 설치할지 에러가 알려 줍니다. 기계가 직접 실행하는 프로그램에는 러너가 필요 없습니다. x86-64 Linux의 `i686-unknown-linux-gnu`, 그리고 CPU가 32비트 ARM 코드도 실행하는 arm64 Linux(Neoverse N1과 Cortex-A72는 실행하고, Neoverse N2와 Apple silicon은 못 함)에 armhf C 라이브러리를 설치했을 때의 `armv7-unknown-linux-gnueabihf`가 그렇습니다(Debian, Ubuntu: `dpkg --add-architecture armhf` 다음 `apt install libc6:armhf`). `armv7-unknown-linux-musleabihf` 프로그램은 정적이라 거기서 C 라이브러리가 필요 없습니다.

```text
$ nabi test --target wasm32-wasip2 gcd
   Compiling mathx (.)
    Finished in 0.61s

running 1 test
test gcd_works ... ok

test result: ok. 1 passed; 0 failed; 2 filtered out; finished in 0.05s
```

다른 러너를 쓰거나 러너가 없는 타깃을 실행하려면 `NABI_RUNNER`에 명령을 적습니다. 프로그램과 그 인자는 그 뒤에 붙습니다.

```text
$ NABI_RUNNER="qemu-aarch64 -L /opt/aarch64-sysroot" nabi test --target aarch64-unknown-linux-gnu
```

Xcode가 있는 Mac에서는 `aarch64-apple-ios-sim` 프로그램이 부팅된 iOS 시뮬레이터에서 돕니다(다른 기기는 `NABI_SIM_DEVICE`로 지정). [28장](28-ios.html)을 보세요.

WASI 프로그램에는 작업 디렉터리가 없어서 상대 경로를 `/`에서 찾습니다. wasm에서 도는 테스트에는 절대 경로를 쓰세요.

### Android

`x86_64-linux-android`는 Android 10 이상용 프로그램과 공유 라이브러리를 빌드하고, 그 결과는 Android 에뮬레이터(x86_64 시스템 이미지)에서 돕니다. arm64 휴대폰과 태블릿용 빌드(`aarch64-linux-android`)는 진행 중입니다. 이 타깃은 아직 기기에서 테스트하지 않았고 이후 릴리스에서 나옵니다. 링크에는 Android NDK가 필요합니다. Android Studio의 SDK Manager(SDK Tools, NDK)나 NDK 다운로드 페이지에서 받습니다. `nabi`는 Android SDK 안에서 NDK를 찾고, 다른 곳에 두었다면 `ANDROID_NDK_HOME`을 그 디렉터리로 설정합니다.

```text
$ export ANDROID_NDK_HOME=~/Android/Sdk/ndk/30.0.16248370
$ nabi build --target x86_64-linux-android --release
$ adb push target/x86_64-linux-android/release/hello /data/local/tmp/
$ adb shell /data/local/tmp/hello
Hello, world!
```

앱의 Java나 Kotlin 코드가 쓸 라이브러리는 JNI 이름으로 함수를 내보내는 `cdylib`입니다. `.so`를 앱의 `jniLibs/x86_64/`(기기용 빌드가 나오면 `arm64-v8a/`)에 두고 `System.loadLibrary`로 읽습니다.

```nabi
use bare::ptr::CPtr;

#[export_name("Java_com_example_app_Native_add")]
pub extern "C" fn add(env: CPtr<c_void>, class: CPtr<c_void>, a: i32, b: i32) -> i32 { a + b }
```

```text
$ nabi build --target x86_64-linux-android --release --crate-type=cdylib
```

프로그램은 Android 10(API 레벨 29) 이상에서 돕니다. `ANDROID_PLATFORM=34`는 더 높은 레벨용으로 빌드합니다. 프로그램과 라이브러리는 페이지가 4 KB인 기기에서도 16 KB인 기기에서도 로드됩니다. Google Play가 Android 15 이상용 앱에 요구하는 것입니다. 커널이 Linux이므로 Linux 프로그램은 대부분 그대로 돕니다. 다른 점은 [플랫폼 노트](../spec/refs-lang/000-meta/006-platform-targets.html#android)에 있습니다.

## `nabi fmt`: 포매터

`nabi fmt`는 패키지의 `.nabi`와 `.nabion` 파일을 표준 형식으로 다시 씁니다. Nabi의 포매터는 일부러 보수적입니다. **토큰, 줄 안의 공백, 주석은 그대로 두고** 들여쓰기(괄호 깊이마다 4칸), 줄 끝 공백, 연속된 빈 줄을 고칩니다. 옮기는 것은 모듈 맨 위의 `mod` / `use` 줄뿐입니다. 이 줄들을 `pub mod`, `mod`, `pub use`, `use` 순서로 묶고, 묶음마다 정렬하고, 묶음 사이에 빈 줄 하나를 둡니다. 그래서 포매터가 코드의 의미나 손으로 맞춘 정렬을 바꾸는 일은 없습니다.

포매터는 연산자 토큰을 컴파일러와 같게 읽습니다. 패키지가 직접 등록한 `#[infix]` 토큰과 의존성이 내보내는 토큰을 알며(아무것도 내려받지 않고 `nabi.lock`에서 찾습니다), 그래서 `a ⊛ b`나 `x |> f`를 쓰는 테스트, 벤치마크, 의존하는 패키지도 포맷합니다.

들여쓰기가 엉망이고 탭이 섞이고 줄 끝 공백이 있는 `fib`를 `--check`로 검사해 봅니다. 포매터는 파일을 건드리지 않고 바뀔 줄을 보여 준 뒤 종료 코드 1로 끝납니다. 이 모드는 CI에 잘 맞습니다.

```text
$ nabi fmt --check
Diff in ./src/main.nabi:
-   10   if n < 2 { return n; }
-   11   let unused = 1;
-   12 	fib(n - 1) + fib(n - 2)
+   10     if n < 2 { return n; }
+   11     let unused = 1;
+   12     fib(n - 1) + fib(n - 2)
        note: 1 file needs formatting; `nabi fmt` without `--check` rewrites it
$ nabi fmt
   Formatted 1 file, 1 already formatted
$ nabi fmt --check
        note: 2 files already formatted
```

`--check` 없이 실행하면 파일을 고치고 몇 개를 고쳤는지 출력합니다. `-q`를 주면 그 줄을 생략합니다. 에디터에서 저장할 때마다 실행하면 편리합니다.

패키지 안에서는 어느 하위 디렉터리에서 실행해도 `nabi fmt`가 패키지 전체를 포맷합니다. `target/`, 숨김 디렉터리, `node_modules/`, `.gitignore`에 있는 경로는 건너뜁니다. 패키지 밖에서는 포맷할 대상을 지정합니다. 파일(`nabi fmt hi.nabi`)이나 디렉터리(`nabi fmt scripts/`)를 줍니다. 경로가 없으면 멈추고 그렇게 알려 줍니다.

```text
$ nabi fmt
nabi: error: not inside a package; pass a file or directory: `nabi fmt <path>`
```

## `nabi lint`와 `nabi check`

`nabi lint`는 컴파일하지 않고 타입 검사와 린트만 실행합니다. 위에 남겨 둔 `let unused = 1;`을 잡아냅니다.

```text
$ nabi lint
    Checking mathx (.)
warning[E0131]: unused variable `unused`
   --> src/main.nabi:11:9
    |
 11 |     let unused = 1;
    |         ^^^^^^
    |
    = note: if this is intentional, prefix it with an underscore: `_unused`
    = note: `#[warn(unused_variables)]` on by default
     Checked mathx: 0 errors, 1 warning
    Finished in 0.05s
```

모든 린트의 기본 수준은 `warn`입니다. 다만 늘 실수인 `nan_comparison`과 `self_assignment`는 `deny`이고, `missing_docs`(`///` 주석이 없는 `pub` 항목)는 직접 켜기 전까지 `allow`입니다. 수준은 세 곳에서 바꿀 수 있고, 뒤의 것이 우선합니다.

1. `nabi.nabion`의 `lints` 필드: `lints: { "unused": "deny", "dead_code": "allow" }`
2. 명령줄의 `-A` / `-W` / `-D` (allow / warn / deny)
3. 소스의 속성: 항목에 `#[allow(dead_code)]`, 파일 맨 위에 `#![deny(unused)]`

린트 하나(`unused_variables`) 대신 그룹(`unused`, `nonstandard_style`, `warnings`, `all`)을 지정할 수도 있습니다. `deny` 린트가 하나라도 걸리면 종료 코드는 1입니다.

`unused_result`는 `xs.len();`이나 `parse(s);`처럼 값을 버리는 문장에 경고합니다. 대개 실수이기 때문입니다. 정말 버리려는 것이면 `_ = expr;`라고 씁니다. `Dict::insert`, `List::pop`, `Atomic::fetch_add`처럼 호출하는 쪽에서 결과가 필요 없는 경우가 많은 함수에는 `#[discardable]`이 붙어 있어서, 이런 함수를 부르는 문장은 경고하지 않습니다. 직접 만든 함수에도 같은 식으로 붙입니다.

```text
$ nabi lint -D unused
    Checking mathx (.)
error[E0131]: unused variable `unused`
   --> src/main.nabi:11:9
    |
 11 |     let unused = 1;
    |         ^^^^^^
    |
    = note: if this is intentional, prefix it with an underscore: `_unused`
    = note: `-D unused` on the command line
     Checked mathx: 1 error, 0 warnings
    Finished in 0.04s
```

경고가 있으면 CI를 실패시키려면 `warnings`를 deny하세요(`lints: { "warnings": "deny" }` 또는 `-D warnings`). rustc와 같이 이 그룹은 이후 릴리스가 추가하는 린트까지 포함해 경고 수준의 린트를 모두 담습니다. 새 릴리스 때문에 CI 결과가 바뀌면 안 된다면 `nabi.nabion`에 toolchain을 정확한 버전으로 고정하세요(`toolchain: "=0.1.0"`). `all`은 모든 린트입니다. `nabi.nabion`에 `"warnings": "deny"`를 적으면 마지막 note가 그 수준이 어디서 왔는지 알려 줍니다.

```text
$ nabi lint
    Checking mathx (.)
error[E0131]: unused variable `unused`
   --> src/main.nabi:11:9
    |
 11 |     let unused = 1;
    |         ^^^^^^
    |
    = note: if this is intentional, prefix it with an underscore: `_unused`
    = note: `"warnings": "deny"` in `lints` of nabi.nabion
     Checked mathx: 1 error, 0 warnings
    Finished in 0.11s
```

어떤 결과에는 그것을 푸는 편집이 딸려 있습니다. 쓰지 않는 `let`에는 앞에 `_`가 붙고, 쓰지 않는 `use` 줄은 지워지고, `while true`는 `loop`가, `x == true`는 `x`가 되고, 불필요한 `as`나 `.clone()`은 사라지고, `{name}`을 담은 평범한 문자열에는 `f`가 붙습니다. 에디터는 이를 하나씩 빠른 수정으로 보여 주고, `nabi fix`는 패키지 전체에 모두 적용한 뒤 남은 것이 없을 때까지 다시 검사합니다.

```text
$ nabi fix
       Fixed src/main.nabi (2 fixes)
```

`nabi fix`는 내 패키지의 파일만 바꾸고, 작업 트리에 커밋하지 않은 변경이 있으면 거절합니다. 그래서 모든 수정이 `git diff`에 보이고 되돌릴 수 있습니다(`--allow-dirty`는 이 검사를 건너뜀). `nabi fix --diff`는 수정을 쓰지 않고 출력만 합니다. 린트 수준도 따릅니다. `nabi fix -A unused_imports`는 import를 그대로 둡니다. 컴파일 에러가 있으면 아무것도 바꾸기 전에 멈추므로 그 에러는 먼저 직접 고칩니다.

`deprecated`는 `#[deprecated]`가 붙은 항목을 쓰면 경고합니다. 함수, 메서드, 필드, 타입, 상수, static 모두 해당하고, 내 코드든 의존성이든 표준 라이브러리든 상관없습니다. 에디터는 그런 사용처에 취소선을 긋습니다. 전체 목록은 `nabi lint --list`로 봅니다. 린트마다 추가된 릴리스도 함께 나옵니다. 출력을 `head`로 파이프해도 됩니다. 읽는 쪽이 먼저 끝나면 `nabi`와 `nabic`는 에러 없이 조용히 멈춥니다. `nabi check`는 한 걸음 더 나아가 코드 생성까지 실행하지만, 링크하거나 실행 파일을 만들지는 않습니다. 빌드보다 빠르므로 컴파일되는지만 확인할 때 씁니다. 여기서부터의 예제는 에러와 저수준 출력이 `mathx`를 어지럽히지 않도록 버려도 되는 `hello` 패키지를 씁니다.

```text
$ nabi check
    Checking hello (.)
error[E0040]: mismatched types: `let s` expects `str`, found `i32`
  --> src/main.nabi:2:14
   |
 2 | let s: str = square(7);
   |              ^^^^^^^^^
   |
   = help: to make text from it, write `f"{square(7)}"`
     Checked hello: 1 error, 0 warnings
    Finished in 0.04s
```

`nabi build`와 `nabi run`도 같은 진단을 출력하고, 빌드에 실패하면 `Finished` 대신 개수를 세어 끝냅니다.

```text
$ nabi build
   Compiling hello (.)
...
error: could not build hello (1 error, 0 warnings)
```

기계가 읽을 출력이 필요하면 `--message-format=json`을 붙입니다. 진단 하나가 JSON 한 줄(`"schema": 1`)로 출력됩니다. 코드와 종류, 메시지, 시작과 끝의 줄·열이 담긴 위치, `notes`와 `help` 줄, 진단이 제안하는 `fixes`, 린트 경고라면 그 `lint` 이름이 들어 있습니다.

대괄호 안의 코드는 에러를 가리키며, 에러 종류마다 코드가 하나씩 있습니다(린트는 모두 `E0131`을 쓰고 노트에 자기 이름을 밝힙니다). `nabic --explain E0040`(또는 `nabi --explain E0040`)은 그 코드의 뜻과 고치는 법을 출력하고, [에러 코드 목록](../errors/index.html)에는 모든 코드가 있습니다. nabic 자체가 멈추면(내부 컴파일러 에러) 종료 코드 70으로 끝납니다. panic이 난 프로그램의 101과는 다릅니다. 이런 일이 생기면 알려 주세요.

## `nabi doc`: 문서 주석과 API 문서

`///`는 바로 아래 항목을 문서화하고, `//!`는 주석이 들어 있는 모듈(파일)을 문서화합니다. 위 패키지에서는 첫 줄 `//! Small math helpers.`가 모듈 설명이 되고, `gcd`와 `fib` 위의 `///`가 함수 설명이 됩니다. 평범한 `//` 주석은 파일 맨 위에 있어도 문서가 아닙니다. 라이브러리의 루트 파일이나 `pub` 모듈의 파일이 `//!` 없이 `//` 줄로 시작하면 `nabi lint`가 경고하고(`plain_module_doc`), 도움말에 써야 할 `//!` 줄을 보여 줍니다. 문서 텍스트에는 제목, 목록, 인라인 코드, 코드 블록, 링크 같은 Markdown 문법을 쓸 수 있습니다. 그 안의 ```` ```nabi ```` 예제는 `nabi test`가 테스트합니다([문서 테스트](#문서-테스트) 참고).

```text
$ nabi doc
 Documenting mathx (.)
    Finished in 0.10s
   Generated ./target/doc/mathx/index.html
```

`nabi doc --open`은 그 페이지를 브라우저에서 엽니다.

`target/doc/`에 패키지와 그 의존성의 공개(`pub`) 항목의 HTML과 검색 색인이 생깁니다. 패키지는 `target/doc/<package>/index.html`이고, 공개 모듈마다 페이지가 하나씩 있습니다(`<package>/geo.html`, `<package>/geo/shapes.html`). 각 페이지 사이드바에 패키지의 의존성이 그 페이지 링크와 함께 나오고, 검색은 문서화된 모든 패키지에서 항목을 찾습니다. 비공개 모듈의 항목을 `pub use`로 내보내면 그 항목은 내보내는 쪽 페이지에 그곳에서 선언한 것처럼 보이고, 공개 모듈이 보여 주는 항목을 다시 내보내면 그 모듈 페이지로 링크합니다. 페이지는 이 사이트의 [표준 라이브러리 API](../../std/core/index.html)와 같은 모습이고 같은 규칙을 따르며, 디스크에서 바로 열어도 인터넷 없이 동작합니다.

## 에디터: `nabils`

`nabils`는 LSP(Language Server Protocol) 서버입니다. 진단, 호버, 정의로 이동, 참조 찾기, 이름 바꾸기, 문서 심볼, 자동 완성, 시그니처 도움말, 빠른 수정, 포맷 기능이 있습니다. 자동 완성은 범위 안의 지역 변수, 값의 멤버(`xs.`는 `List`의 공개 메서드를 보여 줍니다), `use` 뒤의 경로를 제안합니다. 호버와 정의로 이동은 표준 라이브러리 안까지 따라갑니다. 빠른 수정은 진단이 제안한 수정을 적용하고, 빠진 `;`나 `use`를 넣고, 없는 패키지에는 `nabi add`를 실행합니다. 이름 바꾸기는 라이브러리 항목, 키워드, 다른 파일의 이름이면 거절하고 이유를 알려 줍니다. 진단은 저장할 때가 아니라 입력할 때마다 갱신됩니다. 편집할 때마다 백그라운드에서 다시 검사하고, 그동안 호버, 자동 완성, 정의로 이동은 마지막으로 통과한 검사로 바로 답합니다. 편집할 때마다 다시 검사하므로, 웹 페이지에서 딸려 온 문자(폭 없는 공백, `-` 자리의 `−`, 둥근 따옴표), 홀로 쓴 결합 부호, 잘못된 숫자, 잘못된 이스케이프, 처리기가 없는 리터럴 접미사(`hz`가 등록돼 있는데 쓴 `10Hz`)가 바로 표시됩니다. 메시지는 컴파일러의 것과 같고, 고칠 방법이 분명하면 그대로 써 주는 빠른 수정이 함께 옵니다. 각 파일은 그 파일이 속한 패키지의 의존성으로 검사하므로 패키지 여러 개를 담은 폴더를 열어도 되고, 빠른 수정으로, 또는 `nabi.nabion`을 고쳐 저장해서 추가한 의존성도 서버를 다시 시작하지 않고 찾습니다. `nabi lsp`도 같은 서버입니다. 표준 입출력으로 통신하므로 LSP를 지원하는 에디터라면 실행할 명령만 알려 주면 됩니다(에디터가 붙이는 `--stdio`도 받습니다). 파일에 타입 에러가 있어도 호버는 계속 답합니다. 그 검사가 타입을 정한 부분에서, 아니면 마지막으로 통과한 검사에서 답합니다. 터미널에서 직접 실행하면 `nabils`는 에디터가 실행하는 서버라고만 알려 줍니다.

공식 확장은 `nabils`를 알아서 실행하고 구문 강조를 더합니다.

- VS Code: [nabi-lang/nabi.vscode](https://github.com/nabi-lang/nabi.vscode). 아직 Marketplace에 없으니 클론한 저장소에서 `npm ci && npm run package`로 `nabi-<version>.vsix`를 만들고 `code --install-extension nabi-<version>.vsix`로 설치합니다.
- Vim과 Neovim: [nabi-lang/nabi.vim](https://github.com/nabi-lang/nabi.vim). 플러그인 관리자에 이 저장소를 지정합니다(vim-plug라면 `Plug 'nabi-lang/nabi.vim'`). Neovim에서는 `nabils`도 실행합니다.
- Zed: [nabi-lang/nabi.zed](https://github.com/nabi-lang/nabi.zed). 아직 Zed 확장 레지스트리에 없으니 rustup으로 Rust를 설치하고, 저장소를 클론한 뒤 명령 팔레트에서 `zed: install dev extension`을 실행해 클론한 디렉터리를 고릅니다.

플러그인 없이 Neovim(0.11 이상)을 쓴다면 `init.lua`에 다음을 넣습니다.

```text
vim.filetype.add({ extension = { nabi = "nabi", nabion = "nabion" } })
vim.lsp.config("nabils", {
  cmd = { "nabils" },
  filetypes = { "nabi", "nabion" },
  root_markers = { "nabi.nabion" },
})
vim.lsp.enable("nabils")
```

Helix에서는 `languages.toml`에 다음을 넣습니다.

```text
[language-server.nabils]
command = "nabils"

[[language]]
name = "nabi"
scope = "source.nabi"
file-types = ["nabi"]
roots = ["nabi.nabion"]
language-servers = ["nabils"]
```

다른 에디터에서는 에디터의 LSP 클라이언트나 범용 LSP 클라이언트 확장에 `nabils`를 등록합니다. `nabils`는 `PATH`에 있어야 합니다.

## REPL과 Jupyter

1장의 REPL(`nabi` 또는 `nabi repl`)을 좀 더 자세히 봅니다. 터미널에서는 두 줄로 시작합니다. toolchain 버전과 호스트, 그리고 도움말 안내입니다. `nabi repl -q`는 이 두 줄을 생략합니다. 입력이 파이프로 들어오면 처음부터 나오지 않습니다.

```text
$ nabi repl
Nabi 0.1.3 (<commit> <date>) on aarch64-apple-darwin
Type :help for help, :exit to quit.
>>> let xs = [3, 1, 2]
>>> xs.len()
3
>>> it * 10
30
>>> xs.sort(); xs?
[1, 2, 3]
>>> :type xs
List<i64>
>>> fn twice(x: i32) -> i32 {
...     x * 2
... }
>>> twice(21)
42
>>> exit
```

프롬프트는 `>>>`이고, 셀이 이어지는 동안에는 `...`입니다. 입력이 분명히 끝나지 않은 경우가 아니면 셀은 줄 끝에서 끝납니다. 끝나지 않은 경우는 세 가지입니다. 괄호, 문자열, 블록 주석이 아직 열려 있는 경우, 줄이 연산자, `.`, `,`, `=`로 끝나는 경우, `fn f(x: i32)`처럼 본문을 기다리는 항목 머리인 경우입니다. 셀의 마지막 문장이 `let`이나 대입이면 `;`을 생략할 수 있습니다.

각 셀은 한 번 실행됩니다. 최상위 `let` 바인딩과 `fn` / `struct` / `impl` 정의는 이후 셀로 이어집니다. 셀의 최상위에서 `await`를 쓸 수 있습니다. 식 셀은 값을 `Display`로, 없으면 `Debug`로, 그것도 없으면 타입만 출력합니다. 끝에 `?`를 붙이면 대신 `Debug` 형태를 출력합니다. `let` 셀은 아무것도 보여 주지 않습니다. 문장이 여럿인 셀은 마지막이 식으로 끝나면 그 식의 값을 보여 줍니다. 셀이 보여 준 값은 `it`으로 남으므로 다음 셀에서 쓸 수 있습니다. 직접 정의한 `it`이 우선합니다.

일반 소스와 달리 같은 이름을 `let`으로 다시 선언할 수 있고, 타입이 달라도 됩니다. `fn`이나 `struct`를 다시 정의하면 새 정의로 바뀌고, 이전 셀의 코드도 새 버전을 호출합니다. 컴파일에 실패하거나 panic이 일어나거나 에러를 던진 셀은 세션을 바꾸지 않습니다. 이미 쓴 출력은 남습니다. `:reset`은 빈 세션에서 다시 시작합니다.

에러는 입력한 셀 안을 `<repl>:line:col`로 가리킵니다. panic과 이전 셀의 `fn` 안에서 난 에러는 그 셀을 `<repl cell N>:line:col`로 가리킵니다. N은 지금까지 입력한 셀의 번호입니다. Jupyter에서 N은 노트북 셀의 실행 번호이고, REPL 셀 여러 개로 나뉘는 노트북 셀은 `N.1`, `N.2`처럼 번호를 붙입니다.

```text
>>> fn boom(n: i32) {
...     panic(f"boom {n}");
... }
>>> boom(5)
thread 'main' panicked at <repl cell 1>:2:5:
boom 5
error: the cell panicked; the session is unchanged
```

| 명령 | 하는 일 |
|------|---------|
| `expr?` | `Display` 대신 `Debug` 형태를 출력 |
| `:type expr` (`:t`) | 실행하지 않고 타입을 출력 |
| `:doc name` | 이름의 시그니처와 문서 주석을 출력. 메서드는 `:doc xs.len` |
| `:time expr` | 식을 실행하고 단계마다 걸린 시간을 출력 |
| `:vars` | 바인딩을 `name: type`으로 나열한 뒤 항목을 나열 |
| `:load file`, `:reload` | 프로그램을 세션에 로드. 마지막 것을 다시 로드. `:save`로 만든 스크립트는 셀 단위로 다시 실행 |
| `:edit` | `$VISUAL` / `$EDITOR`에서 셀을 쓴 뒤 실행 |
| `:save file` | 세션을 셀마다 표시한 스크립트로 저장 |
| `:opt [level]` | 이후 셀의 최적화 수준을 보거나 설정(`0` ~ `3`, `s`) |
| `:reset` | 모든 셀을 잊고 다시 시작 |
| `:add name ..` | 의존성 추가. 환경에는 `nabi install`, 패키지에서는 `nabi add`. 셀이 쓰는 공식 패키지는 이것 없이도 추가됨 |
| `:help` (`:h`) | 모든 명령을 나열 |
| `:exit`, `:quit` (`:q`) | 종료. `exit`나 `quit`만 쓰거나 입력이 끝나도 종료 |

변수를 캡처하는 클로저는 셀의 최상위 `let`에 남겨 둘 수 없습니다. 클로저마다 이름 없는 자기 타입이 있기 때문입니다. REPL은 그런 셀을 거부합니다.

```text
>>> let x = 3
>>> let f = |n: i64| n * x
error: `f: closure(i64) -> i64` cannot be kept for later cells (its type has no name)
```

바인딩에 함수 타입을 주면(`let f: fn(i64) -> i64 = |n: i64| n * x`) 이후 셀로 이어집니다. `fn`도 이어지지만 `x` 같은 셀 변수는 보지 못하므로 인자로 넘기세요. `xs.map(|n| n * x)`처럼 셀 안에서 쓰는 클로저는 평소대로 동작합니다.

터미널에서 REPL은 줄 편집을 직접 합니다.

- Tab은 이름을 완성합니다. 대상은 범위 안의 지역 변수, 세션의 바인딩과 항목, core 라이브러리, 키워드이고, `x.` 뒤에서는 `x`의 공개 필드와 메서드, `a::` 뒤에서는 모듈 `a`의 항목입니다. 줄 맨 앞에서 Tab은 들여씁니다.
- 기록은 셀 단위로 남고, 세션이 끝나도 유지됩니다(`$NABI_HOME/repl_history`). 위와 아래 화살표로 오갑니다. 텍스트를 조금 입력해 두면 그 텍스트로 시작하는 셀만 거칩니다. Ctrl-R은 거꾸로 검색합니다.
- 붙여 넣은 블록은 줄마다 실행되지 않고, Enter를 누르면 셀 하나가 됩니다.
- 입력은 이 가이드처럼 색칠됩니다. 색을 끄려면 `NO_COLOR`를 설정합니다.

표준 입력을 읽는 셀은 터미널을 읽습니다. REPL 자신의 입력이 파이프로 들어오면 셀은 빈 입력을 봅니다.

`nabi repl path`는 패키지나 파일을 세션에 로드합니다. 라이브러리라면 패키지를 의존성으로 추가하고 `use name::*;`를 먼저 실행합니다. `:reset` 뒤에도 다시 실행합니다. 라이브러리를 만들면서 함수를 시험해 보기에 편리합니다. 프로그램이라면 프로그램을 실행한 뒤 그 바인딩과 항목을 세션에 남깁니다. 패키지 안에서 `nabi repl .`은 현재 패키지를 로드하고, 그냥 `nabi repl`은 빈 상태로 시작합니다.

같은 셀 엔진을 Jupyter 노트북에서도 쓸 수 있습니다. 커널을 한 번 등록하면 Jupyter, VS Code 노트북 등에서 `nabi` 커널을 고를 수 있습니다.

```text
$ nabi jupyter install
   Installed kernel `nabi` (/Users/me/Library/Jupyter/kernels/nabi/kernel.json)
```

macOS에서는 `~/Library/Jupyter` 아래에, Windows에서는 `%APPDATA%\jupyter`에, 그 밖에서는 `$XDG_DATA_HOME/jupyter`(기본값 `~/.local/share/jupyter`)에 설치됩니다. `JUPYTER_DATA_DIR`로 위치를 바꿀 수 있습니다. `nabi jupyter install --env name`은 이름 있는 환경의 커널을 설치합니다.

노트북 셀은 REPL의 규칙을 따르고 마지막 식의 값을 보여 줍니다. 그래프의 그림처럼 타입이 `RichDisplay`를 구현한 값은 `mime_bundle()`이 주는 것(SVG 이미지, HTML)으로 보이고, `display(x)`는 셀 중간에 그런 값을 보여 줍니다. 출력은 끝에서만이 아니라 셀이 실행되는 동안 나타납니다. 표준 입력을 읽는 셀은 노트북에 입력 상자를 띄웁니다. Tab 완성과 Shift-Tab 도움말은 REPL과 같은 이름을 씁니다. 노트북 프런트엔드가 Nabi를 알게 될 때까지 코드는 Rust 규칙으로 강조됩니다.

## 최적화 수준

`nabi build`는 `debug` 프로필을, `--release`는 `release` 프로필을 빌드합니다. `nabi`와 `nabic`에는 수준을 직접 줄 수도 있습니다.

| 수준 | 쓰는 곳 |
|------|---------|
| `-O0` | 기본값. 최적화 없이 가장 빨리 빌드하고, 모든 값이 디버거가 찾는 곳에 있음 |
| `-O1` | 빠르게 도는 디버그 빌드. 루프는 `-O2`만큼 최적화하고(리스트를 도는 `for`가 카운트 루프가 되고, 되면 벡터화됨), 그 밖에서는 인라인을 적게 하며, 디버그 정보를 넣음 |
| `-O2` | `--release`. 빠른 코드 |
| `-O3` | 코드를 더 써서 조금 더 빠르게. 루프 안에서 더 많이 인라인하고, 스칼라 루프(float 합, 본문 안의 호출)를 펼침 |
| `-Os` | 작은 코드. 작은 함수만 인라인하고, 루프는 여전히 카운트 루프가 되며, 짧으면 벡터화됨 |

`-O2`, `-O3`, `-Os`는 `release` 프로필을, `-O0`과 `-O1`은 `debug` 프로필을 빌드합니다. `-g`는 release 빌드의 코드는 그대로 두고 디버그 정보를 더합니다.

## 빌드 캐시

빌드는 프로그램이 닿는 표준 라이브러리 함수를 컴파일하고 그 코드를 toolchain의 캐시에 보관하므로, 어떤 패키지든 다음 빌드는 거기서 가져옵니다. 캐시는 많아야 5 GiB입니다. 이를 넘으면 빌드가 가장 오래전에 쓴 것부터 지워 4 GiB를 남기고, 한 달 동안 어떤 빌드도 쓰지 않은 것도 지웁니다. 상한을 바꾸려면 `NABI_CACHE_MAX_SIZE`를 설정합니다(`NABI_CACHE_MAX_SIZE=20G`, 상한을 없애려면 `off`). `nabi cache`로 보고 줄입니다.

```bash
nabi cache dir                  # 위치
nabi cache prune                # 지금 상한까지 줄임
nabi cache prune --max-size 1G  # 많아야 1 GiB만 남김
nabi cache clean                # 비움
```

toolchain에는 공식 패키지도 읽기 전용 캐시로 들어 있고, 의존성 해석은 그것을 먼저 시도합니다. `nabi component list`는 그것이 있는지 알려 주고, `nabi component remove packages`는 지웁니다. `nabi component add packages`는 다시 받습니다.

## 컴파일러 안 들여다보기

빌드 단계마다 걸리는 시간을 보려면 `-v`를 붙입니다.

```text
$ nabi build -v
   Compiling hello (.)
   Stage: front end (111ms)
      Reused: 168 of 169 files parsed, 0 of 1 bodies lowered, the library's items collected
    Building ./target/debug/hello
       Units 26 reused, 2 compiled
   Stage: codegen + link (22ms)
    Finished in 0.14s
```

`--emit`은 중간 산출물을 `target/debug/build/`에 씁니다. 종류는 `source`(메타 전개와 디슈거링을 마친 패키지 전체를 Nabi 소스로 쓴 것), `hir`, `mir`(코드 생성기가 컴파일하는 것, `-O0`보다 높으면 최적화한 것), `obj`(오브젝트 파일), `asm`(오브젝트를 어셈블리 소스로 쓴 것, 아래 참고), `listing`(기계어를 오프셋, 바이트와 함께 쓴 것, 아래 참고), `expansions`(메타 전개 하나하나의 출력), `c-header`(`#[export]` 함수의 C 헤더, [14장](14-ffi.html#c를-위한-공유-라이브러리---crate-typecdylib) 참고)입니다. `nabi emit`만 쓰면 `mir`을 씁니다.

```text
$ nabi emit
   Compiling hello (.)
       Wrote ./target/debug/build/hello.mir
    Finished in 0.04s
$ grep -A 6 '^fn square' target/debug/build/hello.mir
fn square(_0: i32) -> i32 {
    let _1: i32;

    bb0:
        _1 = Mul<i32>(_0, _0);
        return _1
}
```

`asm`은 오브젝트를 `clang -S`처럼 어셈블리 소스로 씁니다. 섹션, 심볼, 지역 레이블과 재배치 자리에 심볼 이름을 쓴 함수별 명령어(`call nabi_panic_at@PLT`), 데이터가 들어 있습니다. 어셈블하면(`clang -c`) 코드, 재배치, 심볼이 같은 오브젝트가 나오므로 오브젝트와 똑같이 링크되고 실행됩니다. 어셈블러가 텍스트에서 그렇게 인코딩하지 않을 명령어는 강제하거나(`{disp32}`, `mov.s`) 바이트로 쓰고 명령어는 주석에 둡니다. 언와인드 테이블은 어셈블러가 그것을 만들어 내는 지시어(`.cfi_*`, x64 Windows는 `.seh_*`, 32비트 ARM은 `.fnstart` .. `.fnend`, arm64 Windows는 데이터)이고, 디버그 정보는 데이터입니다. wasm32에서는 LLVM의 WebAssembly 어셈블리이고, `clang --target=wasm32-wasi -msimd128 -c`로 어셈블합니다.

```text
$ nabi emit --emit=asm --target x86_64-unknown-linux-gnu
   Compiling hello (.)
       Wrote ./target/x86_64-unknown-linux-gnu/debug/build/hello.s
    Finished in 0.60s
$ grep -A 17 '^_Y.*square_:' target/x86_64-unknown-linux-gnu/debug/build/hello.s
_Y957555df__square_:
	.cfi_startproc
	push	rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	mov	rbp, rsp
	.cfi_def_cfa_register 6
	.byte	0x48, 0x81, 0xec, 0x30, 0x00, 0x00, 0x00	# sub rsp, 0x30
	mov	dword ptr [rsp], edi
.LBB0_0:
	movsxd	rcx, dword ptr [rsp]
	mov.s	rax, rcx
	imul	eax, ecx
	{disp32} jo	.LBB0_1
	movsxd	rax, eax
	mov	rsp, rbp
	pop	rbp
	ret
$ clang --target=x86_64-linux-gnu -c target/x86_64-unknown-linux-gnu/debug/build/hello.s
```

`listing`은 함수마다 기계어를 `objdump -d`가 보여 주는 형식으로 나열합니다(x86_64와 x86은 Intel 문법). 오프셋, 바이트, 명령어를 쓰고, 코드 생성기의 레이블(`.L0:`)과 명령어를 채우는 재배치를 그 명령어 아래에 씁니다. 바이트는 `obj`의 것과 똑같습니다. 모든 대상에서 쓸 수 있고, 내보내기는 링크하지 않으므로 크로스 컴파일할 때도 됩니다.

```text
$ nabi emit --emit=listing --target x86_64-unknown-linux-gnu
   Compiling hello (.)
       Wrote ./target/x86_64-unknown-linux-gnu/debug/build/hello.lst
    Finished in 0.85s
$ grep -A 12 'square_>:' target/x86_64-unknown-linux-gnu/debug/build/hello.lst
<_Y957555df__square_>:
       0: 55                           	push	rbp
       1: 48 89 e5                     	mov	rbp, rsp
       4: 48 81 ec 30 00 00 00         	sub	rsp, 0x30
       b: 89 3c 24                     	mov	dword ptr [rsp], edi
.L0:
       e: 48 63 0c 24                  	movsxd	rcx, dword ptr [rsp]
      12: 48 8b c1                     	mov	rax, rcx
      15: 0f af c1                     	imul	eax, ecx
      18: 0f 80 08 00 00 00            	jo	0x26 <.L1>
      1e: 48 63 c0                     	movsxd	rax, eax
      21: 48 89 ec                     	mov	rsp, rbp
      24: 5d                           	pop	rbp
```

오브젝트 파일에서는, 그래서 디버거나 프로파일러에서도, 함수 이름이 `_Y`로 시작하는 맹글링된 심볼입니다. `nabi demangle`은 심볼을 그 이름으로 풀어 줍니다(`-v`는 심볼이 담은 부분을 보여 줌). 심볼을 주지 않으면 텍스트를 거르는 필터가 되어, 프로파일러나 `nm` 출력을 Nabi 이름으로 읽을 수 있습니다.

```text
$ nabi demangle _Y957555df__square_
square
$ perf report --stdio | nabi demangle
$ nm target/debug/hello | nabi demangle
```

`-g`로 빌드하면 DWARF 디버그 정보가 들어가므로 `lldb` / `gdb`에서 소스 줄 단위로 따라갈 수 있습니다(Windows 빌드는 같은 정보를 PDB의 CodeView로 받으므로, Visual Studio 디버거와 WinDbg에서 소스 줄, 매개변수, 지역 변수와 그 타입을 볼 수 있습니다). `-g`는 최적화 빌드(`-O1` 이상)의 코드를 바꾸지 않습니다. 디버거는 인라인된 함수를 별도의 프레임으로 보여 주고, 최적화된 코드가 더는 값을 갖고 있지 않은 변수는 optimized out으로 보여 줍니다.

toolchain에는 lldb, gdb, natvis(Visual Studio, WinDbg)용 포매터가 들어 있습니다. `str`은 그 텍스트로, `List`, `Dict`, `Set`, `SortedDict`, `Slice`, `Array`는 크기와 원소로, 합 타입은 (lldb에서) 활성 변형으로 보여 줍니다. `nabic --print lldbinit`, `--print gdbinit`, `--print natvis`가 포매터를 로드하는 것을 출력합니다.

```text
$ lldb -O "$(nabic --print lldbinit)" ./target/debug/hello
(lldb) frame variable xs m
(core::collections::list::List<i64>) xs = size=3 {
  [0] = 1
  [1] = 2
  [2] = 3
}
(core::collections::dict::Dict<str, i64, RandomState>) m = size=1 {
  ["a"] = 1
}
$ gdb -iex "$(nabic --print gdbinit)" ./target/debug/hello
(gdb) print xs
$1 = size=3 = {1, 2, 3}
$ nabic --print lldbinit >> ~/.lldbinit      # or load them every time
$ nabic --print gdbinit >> ~/.gdbinit
```

cdb나 WinDbg에서는 natvis 파일을 `.nvload <path>`로 로드합니다(경로는 `nabic --print natvis`가 출력). Visual Studio에서는 이 파일을 `Documents\Visual Studio 2022\Visualizers`에 복사합니다.

WinDbg나 cdb에서 최적화가 인라인한 함수는 자기 주소가 없습니다. `bm hello!sq`는 그 함수가 인라인된 모든 곳에 중단점을 걸고, `k`는 그 함수를 인라인 프레임으로 보여 줍니다. `-g`로 빌드한 wasm32 모듈은 wasmtime을 거쳐 디버깅할 수 있습니다. wasmtime이 모듈의 디버그 정보를 gdb나 lldb용 네이티브 디버그 정보로 바꿔 줍니다. wasmtime 자체의 최적화가 일부 값을 잃을 수 있는데, `-O opt-level=0`을 주면 그 값이 남습니다.

```text
$ gdb --args wasmtime run -D debug-info -O opt-level=0 hello.wasm
```

## 도움말 보기

`nabi help`는 명령을 묶어서 보여 주고, `nabi help <command>`(또는 `nabi <command> --help`)는 명령 하나의 사용법, 예, 옵션을 보여 줍니다.

```text
$ nabi help fmt
usage: nabi fmt [path] [options]

Format .nabi and .nabion files in place: <path> is one file, or a directory
whose files are all formatted (default: the package the current directory
is in).

examples:
  nabi fmt                       format everything here
  nabi fmt --check               exit 1 if a file would change (CI)
  nabi fmt --diff src/main.nabi  show the change, write nothing

options:
  --check  exit 1 if a file would change
  --diff   show the changes, write nothing

global options (-q, -v, --color, ...): `nabi help`
```

명령이나 옵션의 철자를 틀리면 가장 가까운 이름을 알려 줍니다.

```text
$ nabi tset
nabi: unknown command `tset`; did you mean `test`?
run `nabi help` for usage
```

`-q`, `-v`, `--color` 같은 전역 옵션은 명령 앞에도 뒤에도 쓸 수 있습니다.

## 요약

- `nabi test` / `nabi bench`는 `#[test]` / `#[bench]` 함수를 따로 빌드해 실행합니다. 테스트는 각자 자기 프로세스에서 시간 제한을 두고 돕니다. `#[should_panic]`과 `#[ignore]`는 테스트가 확인하는 것과 실행 시점을 바꿉니다. 라이브러리에서는 `nabi test`가 문서와 README의 ```` ```nabi ```` 예제도 실행합니다. `--target`을 주면 `wasmtime`, `node`, `qemu-<cpu>`나 `NABI_RUNNER`를 거쳐 실행합니다.
- `nabi fmt`는 들여쓰기와 빈 줄만 고칩니다. CI에서는 `nabi fmt --check`를 씁니다. 패키지 밖에서는 경로를 줍니다.
- `nabi lint`는 린트를 실행하고 `nabi fix`는 그 수정을 적용하며, `nabi check`는 코드 생성까지 갑니다. 린트 수준은 `nabi.nabion` < `-A/-W/-D` < 소스 속성 순으로 정해집니다.
- `nabi doc`은 `///`와 `//!` 문서 주석으로 `target/doc/`에 HTML을 생성합니다.
- VS Code, Vim/Neovim, Zed용 공식 확장을 설치하거나, 에디터에 `nabils`를 LSP 서버로 등록합니다.
- REPL의 프롬프트는 `>>>`이고, 마지막 값을 `it`에 두며, `:type`, `:doc`, `:vars`, `:reset` 같은 명령이 있습니다. `nabi repl .`은 현재 패키지를 로드하고, 노트북에는 `nabi jupyter install`을 씁니다.
- 문제를 파고들 때는 `-v`, `--emit`, `nabi demangle`, `-g`를 씁니다. 디버거에서는 `nabic --print lldbinit` / `gdbinit` / `natvis`가 로드하는 포매터와 함께 씁니다.
- `nabi help <command>`는 명령 하나의 사용법과 옵션을 보여 줍니다.
- `NABIR_GC_LOG=1` / `NABIR_GC_STATS=1`은 컬렉터의 일시 정지와 safepoint까지 걸린 시간을 보여 주고, 코드에서는 `bare::gc::stats()`가 카운터를 줍니다.

## 더 읽을거리

- 사양: [nabic CLI](../spec/refs-lang/009-tools/901-nabic-cli.html), [표준 포맷](../spec/refs-lang/009-tools/910-nabi-frontend.html#표준-형식), [린트 제어](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#629-allow--deny--warn-린트-제어), [Nabi 프런트엔드](../spec/refs-lang/009-tools/910-nabi-frontend.html), [JIT 셀과 REPL](../spec/refs-lang/009-tools/907-jit-cells.html), [REPL 줄 편집](../spec/refs-lang/009-tools/907-jit-cells.html#줄-편집), [REPL 명령](../spec/refs-lang/009-tools/909-debug-tools.html#999-repl--jupyter-디버그), [Jupyter](../spec/refs-lang/009-tools/908-jupyter-integration.html), [디버그 도구](../spec/refs-lang/009-tools/909-debug-tools.html), [진단](../spec/refs-lang/009-tools/909-debug-tools.html#994-진단--에러-메시지), [심볼 맹글링](../spec/refs-lang/009-tools/905-symbol-mangling.html)
- [에러 코드](../errors/index.html)
- 실행 중에 코드 로드: [23. 플러그인](23-plugins.html)
- 실행 중에 편집 적용: [26. 핫 리로드](26-hot-reload.html)
- API: [core의 assert 함수](../../std/core/panic.html)
- 다음 장: [17. 조건부 컴파일](17-conditional-compilation.html)
