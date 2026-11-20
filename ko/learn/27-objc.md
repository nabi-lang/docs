# 27. Objective-C와 Apple 프레임워크

macOS에서는 Foundation, AppKit, Metal 같은 시스템 라이브러리 자체가 Objective-C 클래스입니다. Nabi는 이것을 직접 부릅니다. 쓰려는 클래스와 메서드를 `extern "objc"` 블록에 선언하면 메서드마다 Objective-C 메시지를 보내는 평범한 Nabi 메서드가 됩니다. C 컴파일러도 접착 코드도 필요 없습니다. Apple 프레임워크라면 그 블록을 직접 쓸 일은 드뭅니다. `ffi::objc::import`가 SDK 헤더를 읽어 대신 씁니다([SDK 헤더 가져오기](#sdk-헤더-가져오기)). 앞의 절들은 손으로 선언하며, import가 무엇을 주는지 보여 줍니다.

이 장의 내용은 모두 Apple 타깃(`aarch64-apple-darwin`)에서만 동작합니다. 그래서 코드 블록은 사이트를 빌드할 때 실행하지 않습니다. 각 블록은 Mac에서 컴파일하고 실행했으며, 그 출력을 함께 보입니다.

## 클래스 선언하기

`extern "objc"` 블록 하나가 클래스 하나를 선언합니다. `#[objc::class(NSArray)]`가 타입 `NSArray`를 만들고, 메서드는 `extern "C"` 함수처럼 쓰되 인스턴스 메서드에는 `self`를 붙입니다.

```nabi norun
use bare::ffi::objc::{Id, NSObject, NSString, ns};

#[link(name = "Foundation", kind = "framework")]
#[objc::class(NSArray)]
extern "objc" {
    fn array() -> Id<Self>;
    fn count(self) -> usize;
    fn objectAtIndex(self, i: usize) -> Id<NSObject>;
    fn componentsJoinedByString(self, sep: Id<NSString>) -> Id<NSString>;
}

#[objc::class(NSMutableArray, extends(NSArray))]
extern "objc" {
    fn addObject(self, o: Id<NSObject>);
}

let fruit = NSMutableArray::array();
for f in ["pear", "apple", "fig"] {
    fruit.addObject(ns(f));
}
println(f"{fruit.count()} fruits: {fruit.componentsJoinedByString(ns(", "))}");
println(f"second: {fruit.objectAtIndex(1)}");
```

```output
3 fruits: pear, apple, fig
second: apple
```

- `self`가 없는 메서드는 클래스 메서드이고 타입에 대고 부릅니다: `NSMutableArray::array()`.
- `extends(NSArray)`는 `NSArray`를 슈퍼클래스로 정하고, `NSMutableArray`는 그 메서드를 물려받습니다. 그래서 `fruit`에서 `count`와 `componentsJoinedByString`이 동작합니다. 클래스의 메서드는 그 클래스와 하위 클래스에 구현된 트레잇이므로, 다른 모듈에서는 그 모듈의 항목을 `use`해야 합니다. `extends`가 없으면 슈퍼클래스는 `NSObject`이고, 그 메서드(`description`, `isEqual`, `hash`, ..)는 모든 객체에 있습니다.
- `Id<Self>`는 메서드를 부른 클래스, 곧 Objective-C의 `instancetype`입니다.
- 객체는 `Id<T>`입니다. `Id<NSObject>`로 선언한 매개변수는 아무 객체나 받으므로 `Id<NSString>`인 `ns(f)`를 그대로 넘깁니다.
- `{x}`는 `NSLog`처럼 객체의 `description`을 출력합니다.

## 메서드 이름이 곧 셀렉터

메서드 이름은 Objective-C 셀렉터에서 부분 사이의 `:`를 `_`로 쓴 것입니다. `componentsJoinedByString(sep)`은 `componentsJoinedByString:`를 보내고, `initWithBytes_length_encoding(b, n, e)`라면 `initWithBytes:length:encoding:`를 보냅니다. 그래서 Apple 문서와 일대일로 읽힙니다. Nabi 이름을 다르게 하고 싶으면 셀렉터를 직접 줍니다.

```nabi norun
use bare::ffi::objc::{Id, NSString, ns};

#[link(name = "Foundation", kind = "framework")]
#[objc::category(NSString)]
extern "objc" {
    fn length(self) -> usize;
    fn uppercaseString(self) -> Id<NSString>;
    #[selector("isEqualToString:")]
    fn same_text(self, other: Id<NSString>) -> bool;
}

let s = ns("héllo");
println(f"{s.length()} {s.uppercaseString()} {s.same_text(ns("héllo"))}");
println(s.uppercaseString().to_str());
```

```output
5 HÉLLO true
HÉLLO
```

`NSString`은 `bare::ffi::objc`가 이미 선언했으므로 이 블록은 *카테고리*입니다. `#[objc::category(NSString)]`는 다른 곳에서 선언한 클래스에 메서드를 더합니다. `ns("text")`는 `str`로 `NSString`을 만들고, `to_str()`은 다시 복사해 옵니다.

이름의 부분이 매개변수와 맞지 않으면 컴파일러가 알려 주고 두 가지 쓰는 법을 보여 줍니다.

```text
error[E0084]: `initWithBytes_length` has 3 parameter(s), so its selector has 3 part(s): name it after the selector with `_` between the parts (`initWithBytes_length` for `initWithBytes:length:`), or give it with `#[selector("..")]`
```

## nil, `none`, 오류

`Id<T>`를 돌려준다고 선언한 메서드가 nil을 받으면 메서드 이름을 대며 패닉합니다. nil이 정상적인 답이라면 `Id<T> | none`으로 선언합니다.

```nabi norun
use bare::ffi::objc;
use bare::ffi::objc::{Id, NSObject, NSString, ns};

#[link(name = "Foundation", kind = "framework")]
#[objc::class(NSMutableDictionary)]
extern "objc" {
    fn dictionary() -> Id<Self>;
    fn setObject_forKey(self, o: Id<NSObject>, key: Id<NSObject>);
    fn objectForKey(self, key: Id<NSObject>) -> Id<NSObject> | none;
}

#[objc::class(NSFileManager)]
extern "objc" {
    fn defaultManager() -> Id<Self>;
    fn contentsOfDirectoryAtPath_error(self, path: Id<NSString>) throws (objc::Error) -> Id<NSArray>;
}

#[objc::class(NSArray)]
extern "objc" {
    fn count(self) -> usize;
}

let ages = NSMutableDictionary::dictionary();
ages.setObject_forKey(ns("31"), ns("ana"));
println(f"ana: {ages.objectForKey(ns("ana")) ?? ns("?").as_object()}");
println(f"bo: {ages.objectForKey(ns("bo")) ?? ns("unknown").as_object()}");

let fm = NSFileManager::defaultManager();
let n = try fm.contentsOfDirectoryAtPath_error(ns("/no/such/dir")).count()
    catch e: objc::Error { println(f"{e.domain()} {e.code()}: {e}"); 0 };
println(f"{n} entries");
```

```output
ana: 31
bo: unknown
NSCocoaErrorDomain 260: The folder “dir” doesn’t exist.
0 entries
```

Cocoa는 마지막 `NSError **` 매개변수로 오류를 알립니다. 그 매개변수는 빼고 `throws (objc::Error)`라고 씁니다. 그러면 호출에 `try`가 필요하고, 오류는 다른 오류처럼 잡습니다([10장](10-errors.html)). 셀렉터에 `error:`가 있으므로 이름에는 `_error` 부분이 남습니다.

## 블록은 클로저

클로저 타입 매개변수는 Objective-C 블록을 넘깁니다. 다른 Nabi 함수에 넘기듯 클로저를 넘기면 됩니다.

```nabi norun
use bare::ffi::objc::{Id, NSObject, NSString, ns};

#[link(name = "Foundation", kind = "framework")]
#[objc::class(NSMutableArray)]
extern "objc" {
    fn array() -> Id<Self>;
    fn addObject(self, o: Id<NSObject>);
    fn enumerateObjectsUsingBlock(self, f: fn(Id<NSObject>, usize, bare::ptr::CPtr<bool>));
    fn sortedArrayUsingComparator(self, cmp: fn(Id<NSObject>, Id<NSObject>) -> isize) -> Id<NSMutableArray>;
}

#[objc::category(NSString)]
extern "objc" {
    fn compare(self, other: Id<NSString>) -> isize;
}

let words = NSMutableArray::array();
for w in ["pear", "apple", "fig"] {
    words.addObject(ns(w));
}
let text = |o: Id<NSObject>| o.downcast::<NSString>() ?? ns("");
let sorted = words.sortedArrayUsingComparator(|a, b| text(a).compare(text(b)));
let seen: List<str> = [];
sorted.enumerateObjectsUsingBlock(|o, i, stop| { seen.push(f"{i}={o}"); });
println(seen.join(" "));
```

```output
0=apple 1=fig 2=pear
```

클로저는 Objective-C가 블록을 쥐는 동안 살아 있습니다. 완료 핸들러는 다른 스레드에서 실행될 수 있으니, `Atomic`, `Mutex`, 채널처럼 스레드에 안전한 값만 함께 쓰십시오.

## 객체는 누가 해제하나

`Id<T>`는 객체의 retain 하나를 쥐고, `Id`의 사본은 그것을 함께 씁니다. Nabi는 Cocoa의 이름 규칙을 따릅니다. `alloc`, `new`, `copy`, `mutableCopy`, `init`이 돌려주는 것은 이미 내 것이고, 나머지는 대신 retain합니다. 어느 쪽이든 `Id`에는 retain이 정확히 하나 남습니다.

`Id`의 사본이 하나도 남지 않으면 가비지 컬렉터가 다른 값처럼 그 객체를 해제합니다. 귀한 것(파일, 창)을 쥔 객체는 `release_now()`로 곧바로 해제합니다. 그 뒤에 사본을 쓰면 해제된 메모리를 건드리는 대신 메시지와 함께 패닉합니다. Objective-C가 autorelease한 객체는 autorelease pool이 해제합니다. 그런 객체를 많이 만드는 루프는 `objc::autoreleasepool`로 감쌉니다.

```nabi norun
use bare::ffi::objc;
use bare::ffi::objc::{Id, NSString, ns};

#[link(name = "Foundation", kind = "framework")]
#[objc::category(NSString)]
extern "objc" {
    fn stringByAppendingString(self, s: Id<NSString>) -> Id<NSString>;
}

let base = ns("item ");
let last = objc::autoreleasepool(|| {
    let s = base;
    for i in 0..10000 {
        let t = base.stringByAppendingString(ns(f"{i}"));
        if i == 9999 { s = t; } else { t.release_now(); }
    }
    s
});
println(last.to_str());
last.release_now();
println(f"released: {last.is_released()}");
```

```output
item 9999
released: true
```

`objc::WeakId::new(id)`는 객체를 살려 두지 않고 가리킵니다. 객체가 사라지면 `load()`가 `none`을 줍니다. 자기를 쥔 창을 쥐는 delegate처럼 순환을 끊을 때 씁니다.

## Metal 컴퓨트 커널

Metal 객체는 클래스가 아니라 프로토콜(`MTLDevice`, `MTLBuffer`, ..)로 알려져 있으므로 `#[objc::protocol(..)]`로 선언합니다. 이 프로그램은 소스에서 커널을 컴파일하고, GPU에서 숫자 1024개에 실행한 뒤 결과를 확인합니다.

```nabi norun
use bare::ffi::objc;
use bare::ffi::objc::{Id, NSObject, NSString, ns};
use bare::ptr::CPtr;

#[repr(C)]
struct MTLSize { width: usize; height: usize; depth: usize; }

#[link(name = "Metal", kind = "framework")]
#[link(name = "Foundation", kind = "framework")]
extern "C" {
    fn MTLCreateSystemDefaultDevice() -> CPtr<u8>;
}

#[objc::protocol(MTLDevice)]
extern "objc" {
    fn name(self) -> Id<NSString>;
    fn newLibraryWithSource_options_error(self, source: Id<NSString>, options: Id<NSObject> | none) throws (objc::Error) -> Id<MTLLibrary>;
    fn newComputePipelineStateWithFunction_error(self, f: Id<MTLFunction>) throws (objc::Error) -> Id<MTLComputePipelineState>;
    fn newCommandQueue(self) -> Id<MTLCommandQueue>;
    fn newBufferWithLength_options(self, length: usize, options: usize) -> Id<MTLBuffer>;
}

#[objc::protocol(MTLLibrary)]
extern "objc" {
    fn newFunctionWithName(self, name: Id<NSString>) -> Id<MTLFunction> | none;
}

#[objc::protocol(MTLFunction)]
extern "objc" {}

#[objc::protocol(MTLComputePipelineState)]
extern "objc" {
    fn maxTotalThreadsPerThreadgroup(self) -> usize;
}

#[objc::protocol(MTLBuffer)]
extern "objc" {
    fn contents(self) -> CPtr<f32>;
}

#[objc::protocol(MTLCommandQueue)]
extern "objc" {
    fn commandBuffer(self) -> Id<MTLCommandBuffer>;
}

#[objc::protocol(MTLCommandBuffer)]
extern "objc" {
    fn computeCommandEncoder(self) -> Id<MTLComputeCommandEncoder>;
    fn addCompletedHandler(self, handler: fn(Id<MTLCommandBuffer>));
    fn commit(self);
    fn waitUntilCompleted(self);
    fn status(self) -> usize;
}

#[objc::protocol(MTLComputeCommandEncoder)]
extern "objc" {
    fn setComputePipelineState(self, state: Id<MTLComputePipelineState>);
    fn setBuffer_offset_atIndex(self, buffer: Id<MTLBuffer> | none, offset: usize, index: usize);
    fn dispatchThreads_threadsPerThreadgroup(self, threads: MTLSize, per_group: MTLSize);
    fn endEncoding(self);
}

const KERNEL: str = "
#include <metal_stdlib>
using namespace metal;
kernel void square_plus(device const float *in [[buffer(0)]],
                        device float *out [[buffer(1)]],
                        uint i [[thread_position_in_grid]]) {
    out[i] = in[i] * in[i] + 1.0;
}
";

let n = 1024usize;
let raw = unsafe { MTLCreateSystemDefaultDevice() };
let device = unsafe { Id::<MTLDevice>::from_retained(raw) } ?? panic("no Metal device");
let library = try! device.newLibraryWithSource_options_error(ns(KERNEL), none);
let function = library.newFunctionWithName(ns("square_plus")) ?? panic("no kernel");
let pipeline = try! device.newComputePipelineStateWithFunction_error(function);
let input = device.newBufferWithLength_options(n * 4, 0);
let output = device.newBufferWithLength_options(n * 4, 0);
let src = input.contents().as_ptr() ?? panic("no contents");
for i in 0..n {
    unsafe { src.add(i).write(i as f32) };
}

let cmd = device.newCommandQueue().commandBuffer();
let enc = cmd.computeCommandEncoder();
enc.setComputePipelineState(pipeline);
enc.setBuffer_offset_atIndex(input, 0, 0);
enc.setBuffer_offset_atIndex(output, 0, 1);
let group = pipeline.maxTotalThreadsPerThreadgroup().min(n);
enc.dispatchThreads_threadsPerThreadgroup(MTLSize { width: n, height: 1, depth: 1 }, MTLSize { width: group, height: 1, depth: 1 });
enc.endEncoding();
let status = Atomic::new(0i64);
cmd.addCompletedHandler(|buf| { status.store(buf.status() as i64); });   // Metal 스레드에서 실행
cmd.commit();
cmd.waitUntilCompleted();

let out = output.contents().as_ptr() ?? panic("no contents");
let wrong = 0;
for i in 0..n {
    if unsafe { out.add(i).read() } != (i as f32) * (i as f32) + 1.0 { wrong += 1; }
}
println(f"GPU done (status {status.load()}): {n} results, {wrong} wrong");
```

```output
GPU done (status 4): 1024 results, 0 wrong
```

`MTLCreateSystemDefaultDevice`는 C 함수이므로 `extern "C"` 블록에 선언합니다. 그 결과는 `+1` 객체이고 `Id::from_retained`가 넘겨받습니다. `MTLSize`는 평범한 `#[repr(C)]` 구조체이고, C가 넘기듯 값으로 넘어갑니다.

## SDK 헤더 가져오기

위의 블록은 메서드를 하나하나 손으로 선언하지만, Apple 프레임워크에는 메서드가 수천 개 있습니다. `ffi::objc::import`는 SDK 헤더를 직접 읽어 그 블록을 대신 씁니다.

```nabi norun
mod ns {
    ffi::objc::import { "Foundation/Foundation.h" }
}
use ns::*;
use bare::ffi::objc;
use bare::ffi::objc::ns;

let s = ns("hello, objc");
let parts = s.componentsSeparatedByString(ns(", "));
let last = parts.lastObject() ?? panic("empty");
println(f"{s.length()} {s.uppercaseString().to_str()} {parts.count()} {last}");

let flags = NSCaseInsensitiveSearch | NSLiteralSearch;
println(f"options {flags.contains(NSLiteralSearch)} {flags.contains(NSBackwardsSearch)}");

try NSString::stringWithContentsOfFile_encoding_error(ns("/nonexistent/nabi"), NSUTF8StringEncoding)
    catch e: objc::Error { println(f"error {e.domain()} {e.code()}"); ns("") };
println(f"domain {NSCocoaErrorDomain().to_str()}");
```

```output
11 HELLO, OBJC 2 objc
options true false
error NSCocoaErrorDomain 260
domain NSCocoaErrorDomain
```

- 헤더는 `Framework/Header.h`로 씁니다. 메서드를 부르는 프레임워크는 모두 import 하나에 넣습니다. `{ "Foundation/Foundation.h", "AppKit/AppKit.h", "Metal/Metal.h" }`처럼 씁니다.
- 바인딩은 import가 있는 모듈에 들어갑니다. 클래스의 메서드는 그곳의 트레잇이므로 `use ns::*;`로 스코프에 가져옵니다. 이것으로 `Id`, `NSString` 같은 것도 함께 들어옵니다.
- 무엇이 nil일 수 있는지는 헤더가 알려 줍니다. `lastObject`는 `Id<NSObject> | none`을, `uppercaseString`은 `Id<NSString>`을 돌려줍니다. 마지막 `NSError **`는 `throws (objc::Error)`가 되고, `NS_OPTIONS` 타입은 `|`, `&`, `contains`를 받으며, `NSCocoaErrorDomain` 같은 상수는 함수입니다.
- 첫 빌드는 헤더를 읽고 바인딩을 toolchain 캐시에 씁니다. 다음 빌드부터는 그 파일 하나만 읽습니다. `nabi bindgen -x objective-c <header>`는 같은 바인딩을 출력하므로, import가 무엇을 주는지 읽어 볼 수 있습니다.

## 새로운 API

프로그램은 배포 타깃 이후의 모든 macOS에서 돕니다. 매니페스트에 `deployment_target: "13.0"`이 없으면 11.0입니다. 헤더는 나중에 생긴 API를 표시하고, 컴파일러는 그 API가 없는 macOS에서 돌 수도 있는 호출을 거부합니다.

```text
error[E0084]: `maximumConcurrentCompilationTaskCount` needs macOS 13.3, the deployment target is 11.0: wrap it in `if objc::available(macos: "13.3") { .. }`, or raise the deployment target (`deployment_target: "13.3"` in nabi.nabion, or `MACOSX_DEPLOYMENT_TARGET=13.3`)
```

`if objc::available(macos: "13.3") { .. }` 안에서는 호출할 수 있고, 더 오래된 macOS에서는 그 블록을 건너뜁니다.

```nabi norun
mod mtl {
    ffi::objc::import { "Metal/Metal.h" }
}
use mtl::*;
use bare::ffi::objc;

let d = MTLCreateSystemDefaultDevice() ?? panic("no Metal device");
if objc::available(macos: "13.3") {
    println(f"compile tasks {d.maximumConcurrentCompilationTaskCount()}");
}
```

```output
compile tasks 6
```

## delegate가 있는 창

AppKit 앱은 이벤트를 delegate에게 넘깁니다. delegate는 내가 정한 클래스의 객체입니다. `ClassBuilder`는 `extern "C"` 함수로 그런 클래스를 실행 중에 만듭니다. 함수는 저마다 수신자와 셀렉터를 먼저 받고 그다음에 메서드의 인자를 받으며, 타입 인코딩(`"v@:@"`: 아무것도 돌려주지 않고 객체 하나를 받음)이 Objective-C에 그 시그니처를 알립니다. 이 프로그램은 창을 열고, 클릭이 그러듯 런 루프에서 창을 닫고, 마지막 창과 함께 끝납니다.

```nabi norun
mod ak {
    ffi::objc::import { "Foundation/Foundation.h", "AppKit/AppKit.h" }
}
use ak::*;
use bare::ffi::objc::{Block, ClassBuilder, Queue, Sel, dispatch_async, ns};
use bare::mem::transmute_unchecked;
use bare::ptr::CPtr;

extern "C" fn did_finish_launching(this: CPtr<u8>, _cmd: Sel, _note: CPtr<u8>) {
    let rect = CGRect { origin: CGPoint { x: 200.0, y: 200.0 }, size: CGSize { width: 320.0, height: 200.0 } };
    let style = NSWindowStyleMaskTitled | NSWindowStyleMaskClosable;
    let w = NSWindow::alloc().initWithContentRect_styleMask_backing_defer(rect, style, NSBackingStoreBuffered, false);
    w.setReleasedWhenClosed(false);
    w.setTitle(ns("Nabi"));
    let delegate = unsafe { Id::<NSObject>::retain(this) } ?? panic("no delegate");
    w.setDelegate(unsafe { delegate.cast::<NSWindowDelegate>() });
    w.makeKeyAndOrderFront(none);
    println(f"launched: window \"{w.title().to_str()}\" {w.frame().size.width}x{w.frame().size.height} visible {w.isVisible()}");
    dispatch_async(Queue::main(), Block::new_main_thread(|| { w.performClose(none); }));
}

extern "C" fn window_should_close(_this: CPtr<u8>, _cmd: Sel, _sender: CPtr<u8>) -> bool {
    println("delegate: windowShouldClose:");
    true
}

extern "C" fn window_will_close(_this: CPtr<u8>, _cmd: Sel, _note: CPtr<u8>) {
    println("delegate: windowWillClose:");
}

extern "C" fn last_window_closed(_this: CPtr<u8>, _cmd: Sel, _app: CPtr<u8>) -> bool { true }

extern "C" fn will_terminate(_this: CPtr<u8>, _cmd: Sel, _note: CPtr<u8>) {
    println("delegate: applicationWillTerminate:");
}

fn imp(f: extern "C" fn(CPtr<u8>, Sel, CPtr<u8>)) -> usize { unsafe { transmute_unchecked(f) } }
fn imp_bool(f: extern "C" fn(CPtr<u8>, Sel, CPtr<u8>) -> bool) -> usize { unsafe { transmute_unchecked(f) } }

let app = NSApplication::sharedApplication();
_ = app.setActivationPolicy(NSApplicationActivationPolicyRegular);
let class = ClassBuilder::new("AppDelegate", NSObject::class()) ?? panic("the class exists");
let class = unsafe {
    class
        .method(Sel::register("applicationDidFinishLaunching:"), imp(did_finish_launching), "v@:@")
        .method(Sel::register("applicationWillTerminate:"), imp(will_terminate), "v@:@")
        .method(Sel::register("applicationShouldTerminateAfterLastWindowClosed:"), imp_bool(last_window_closed), "B@:@")
        .method(Sel::register("windowShouldClose:"), imp_bool(window_should_close), "B@:@")
        .method(Sel::register("windowWillClose:"), imp(window_will_close), "v@:@")
};
let class = class.protocol("NSApplicationDelegate").protocol("NSWindowDelegate").register();
let delegate = class.new_object();
app.setDelegate(unsafe { delegate.cast::<NSApplicationDelegate>() });
app.run();
```

```output
launched: window "Nabi" 320x232 visible true
delegate: windowShouldClose:
delegate: windowWillClose:
delegate: applicationWillTerminate:
```

- `setDelegate`는 `Id<NSApplicationDelegate>`를 받습니다. 여기서 만든 클래스는 그 프로토콜을 따르며, `cast`가 그 사실을 Nabi에 알립니다.
- `Block::new_main_thread`는 `Send`가 아닌 클로저로 메인 큐용 블록을 만들고, `dispatch_async`는 그것을 런 루프에서 실행합니다. `Block::new`는 어느 큐에서든 도는 `Send` 클로저를 받습니다.
- 창 객체는 AppKit이 요구하는 대로 메인 스레드에서 해제됩니다. 수집기가 다른 스레드에서 죽은 것을 찾아도 그렇습니다.

## Metal 삼각형

생성된 바인딩이 있으면 Metal은 따로 선언할 것이 없습니다. 이 프로그램은 셰이더 둘을 컴파일하고, 64x64 텍스처에 빨간 삼각형 하나를 그리고, 픽셀을 다시 읽어 옵니다.

```nabi norun
mod mtl {
    ffi::objc::import { "Foundation/Foundation.h", "Metal/Metal.h" }
}
use mtl::*;
use bare::alloc::alloc_array;
use bare::ffi::objc::ns;
use bare::ptr::ConstPtr;

const SHADERS: str = "
#include <metal_stdlib>
using namespace metal;
vertex float4 vmain(uint v [[vertex_id]]) {
    float2 p[3] = { float2(0.0, 0.8), float2(-0.8, -0.8), float2(0.8, -0.8) };
    return float4(p[v], 0.0, 1.0);
}
fragment float4 fmain() { return float4(1.0, 0.0, 0.0, 1.0); }
";

const W: u64 = 64;

fn main() throws {
    let device = MTLCreateSystemDefaultDevice() ?? panic("no Metal device");
    let lib = try device.newLibraryWithSource_options_error(ns(SHADERS), none);
    let desc = MTLRenderPipelineDescriptor::new();
    desc.setVertexFunction(lib.newFunctionWithName(ns("vmain")));
    desc.setFragmentFunction(lib.newFunctionWithName(ns("fmain")));
    desc.colorAttachments().objectAtIndexedSubscript(0).setPixelFormat(MTLPixelFormatRGBA8Unorm);
    let pipeline = try device.newRenderPipelineStateWithDescriptor_error(desc);

    let td = MTLTextureDescriptor::texture2DDescriptorWithPixelFormat_width_height_mipmapped(MTLPixelFormatRGBA8Unorm, W, W, false);
    td.setUsage(MTLTextureUsageRenderTarget | MTLTextureUsageShaderRead);
    td.setStorageMode(MTLStorageModeShared);
    let target = device.newTextureWithDescriptor(td) ?? panic("no texture");

    let pass = MTLRenderPassDescriptor::renderPassDescriptor();
    let color = pass.colorAttachments().objectAtIndexedSubscript(0);
    color.setTexture(target);
    color.setLoadAction(MTLLoadActionClear);
    color.setStoreAction(MTLStoreActionStore);
    color.setClearColor(MTLClearColor { red: 0.0, green: 0.0, blue: 0.0, alpha: 1.0 });

    let cmd = (device.newCommandQueue() ?? panic("no queue")).commandBuffer() ?? panic("no command buffer");
    let enc = cmd.renderCommandEncoderWithDescriptor(pass) ?? panic("no encoder");
    enc.setRenderPipelineState(pipeline);
    enc.drawPrimitives_vertexStart_vertexCount(MTLPrimitiveTypeTriangle, 0, 3);
    enc.endEncoding();
    let done = Atomic::new(0);
    cmd.addCompletedHandler(|_b| { done.fetch_add(1); });
    cmd.commit();
    cmd.waitUntilCompleted();

    // GC 힙 밖: Metal이 여기에 씁니다.
    let buf = unsafe { alloc_array::<u8>((W * W * 4) as usize) };
    let px = |i: u64| unsafe { ConstPtr::<u8>::dangling().with_addr(buf.addr() + i as usize).read() };
    let region = MTLRegion { origin: MTLOrigin { x: 0, y: 0, z: 0 }, size: MTLSize { width: W, height: W, depth: 1 } };
    unsafe { target.getBytes_bytesPerRow_fromRegion_mipmapLevel(buf, W * 4, region, 0) };
    let at = |x: u64, y: u64| {
        let i = (y * W + x) * 4;
        f"{px(i)},{px(i + 1)},{px(i + 2)},{px(i + 3)}"
    };
    let red = (0..W * W).filter(|i| px(i * 4) == 255u8).count();
    println(f"center {at(32, 32)} corner {at(1, 1)} red {red} of {W * W}");
    println(f"completed handler ran {done.load()}");
}
```

```output
center 255,0,0,255 corner 0,0,0,255 red 1352 of 4096
completed handler ran 1
```

`MTLCreateSystemDefaultDevice`는 `Id<MTLDevice> | none`을 돌려줍니다. 헤더가 nullable로 표시했고, `NS_RETURNS_RETAINED`가 결과가 이미 내 것임을 알려 줍니다. `MTLRegion`, `MTLClearColor` 같은 구조체는 평범한 `#[repr(C)]` 구조체이고 값으로 넘깁니다.

## 직접 만드는 블록

`dispatch_async`처럼 블록을 받는 C 함수에는 `Block`을 넘깁니다. dispatch 큐는 `bare::ffi::objc`에 있습니다.

```nabi norun
use bare::ffi::objc::{Block, Queue, dispatch_async};

let total = Atomic::new(0);
for i in 1..=8 {
    dispatch_async(Queue::global(), Block::new(|| { total.fetch_add(i); }));
}
```

큐가 다른 스레드에서 실행하므로 `Block::new`는 `Send` 클로저가 필요합니다. Objective-C가 넘겨주는 블록은 `BlockRef`로 오며 그 호출 동안 유효합니다. `copy()`로 간직합니다.

## 정리

- `extern "objc"` 블록에 `#[objc::class(C)]` / `#[objc::class(C, extends(S))]`를 붙이면 클래스를, `#[objc::protocol(P)]`를 붙이면 프로토콜로 알려진 객체의 타입을 선언합니다. `#[objc::category(C)]`는 선언된 클래스에 메서드를 더합니다.
- `self`가 있는 메서드는 `Id<C>`에, 없는 메서드는 `C`에 있습니다. 하위 클래스는 슈퍼클래스의 메서드를 물려받습니다.
- 메서드 이름은 셀렉터에서 안쪽 `:`를 `_`로 쓴 것이고, `#[selector("..")]`로 직접 줄 수도 있습니다.
- 결과: `Id<T>`(nil이면 패닉), `Id<T> | none`. `throws (objc::Error)`가 마지막 `NSError **`를 대신합니다.
- 클로저 매개변수는 블록입니다.
- `Id`는 retain 하나를 쥐고, 마지막 사본이 사라지면 수집기가 돌려줍니다. `release_now()`는 객체를 곧바로 해제하고, `objc::autoreleasepool`은 Objective-C가 autorelease한 것을 비우며, `WeakId`는 순환을 끊습니다.
- `ffi::objc::import { "Framework/Header.h", .. }`가 SDK 헤더에서 블록을 씁니다. 부르려면 그 모듈의 항목을 `use`합니다.
- 배포 타깃보다 새로운 API는 `if objc::available(macos: "..") { .. }` 안에서 부릅니다.
- `ClassBuilder`는 `extern "C"` 함수로 delegate 클래스를 만들고, `Block`과 `dispatch_async`는 클로저를 dispatch 큐에서 실행합니다.

## 더 읽기

- 표준 라이브러리 명세: [Objective-C](../spec/refs-standard-libraries/100-bare/110-objc.html), [FFI](../spec/refs-standard-libraries/100-bare/107-ffi.html)
- 앞 장: [14. FFI 기초](14-ffi.html), [10. 오류](10-errors.html)
