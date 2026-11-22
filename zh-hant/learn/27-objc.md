# 27. Objective-C 與 Apple 框架

在 macOS 上，系統本身的函式庫，也就是 Foundation、AppKit、Metal 等，都是 Objective-C 類別。Nabi 直接呼叫它們：在 `extern "objc"` 區塊中宣告你用到的類別與方法，每個方法就成為一個普通的 Nabi 方法，負責送出對應的 Objective-C 訊息。過程中不需要 C 編譯器，也不需要膠合程式碼。對於 Apple 的框架，你很少需要自己寫這些區塊：`ffi::objc::import` 會讀取 SDK 的標頭檔並替你產生（[匯入 SDK 的標頭檔](#importing-the-sdks-headers)）。前面幾節先手動宣告，藉此說明匯入會帶給你什麼。

本章的內容只能在 Apple 目標（`aarch64-apple-darwin`）上運作。建置網站時不會執行其中的程式碼區塊；每個區塊都已在 Mac 上編譯並執行過，並附上輸出。

## 宣告類別

一個 `extern "objc"` 區塊宣告一個類別。`#[objc::class(NSArray)]` 產生型別 `NSArray`；它的方法寫法與 `extern "C"` 函式相同，實體方法以 `self` 表示：

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

- 沒有 `self` 的方法是類別方法，在型別上呼叫：`NSMutableArray::array()`。
- `extends(NSArray)` 讓 `NSArray` 成為父類別，`NSMutableArray` 因此取得它的方法：`count` 與 `componentsJoinedByString` 都能用在 `fruit` 上。類別的方法是為它及其子類別實作的 trait，所以在其他模組中使用時，需要 `use` 那個模組的項目。沒有 `extends` 時，父類別是 `NSObject`，它的方法（`description`、`isEqual`、`hash` 等）每個物件都有。
- `Id<Self>` 是呼叫該方法的那個類別，相當於 Objective-C 的 `instancetype`。
- 物件的型別是 `Id<T>`。宣告為 `Id<NSObject>` 的參數接受任何物件，所以 `Id<NSString>` 型別的 `ns(f)` 可以直接傳入。
- `{x}` 會印出物件的 `description`，與 `NSLog` 的做法相同。

## 方法名稱就是選擇器

方法的名稱就是它的 Objective-C 選擇器（selector），選擇器中各部分之間的 `:` 改寫為 `_`：`componentsJoinedByString(sep)` 送出 `componentsJoinedByString:`，而 `initWithBytes_length_encoding(b, n, e)` 會送出 `initWithBytes:length:encoding:`。因此 Apple 的文件可以一對一對照閱讀。若希望 Nabi 名稱不同，就明確給出選擇器：

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

`NSString` 已由 `bare::ffi::objc` 宣告，所以這個區塊是一個*分類*（category）：`#[objc::category(NSString)]` 為在別處宣告的類別新增方法。`ns("text")` 從 `str` 產生一個 `NSString`，`to_str()` 則將其複製回來。

如果名稱的部分數與參數不符，編譯器會指出問題，並列出兩種正確寫法：

```text
error[E0084]: `initWithBytes_length` has 3 parameter(s), so its selector has 3 part(s): name it after the selector with `_` between the parts (`initWithBytes_length` for `initWithBytes:length:`), or give it with `#[selector("..")]`
```

## nil、`none` 與錯誤

宣告回傳 `Id<T>` 的方法若收到 nil 就會觸發 panic，並指出是哪個方法。當 nil 是正常的回答時，宣告為 `Id<T> | none`：

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

Cocoa 透過最後一個 `NSError **` 參數回報錯誤。省略這個參數，改寫 `throws (objc::Error)`：此時呼叫需要 `try`，錯誤也能像其他錯誤一樣攔截（[第 10 章](10-errors.html)）。名稱保留 `_error` 部分，因為選擇器本身有這個部分。

## 區塊就是閉包

閉包型別的參數會傳遞一個 Objective-C 區塊（block）。傳入閉包的方式與傳給任何 Nabi 函式相同：

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

只要 Objective-C 還保留著該區塊，閉包就會一直存在。完成處理常式（completion handler）可能在另一個執行緒上執行；只與它共用執行緒安全的值，例如 `Atomic`、`Mutex` 或通道。

## 誰負責釋放物件

一個 `Id<T>` 持有其物件的一次 retain，`Id` 的各個副本共用這一次 retain。Nabi 遵循 Cocoa 的命名規則：`alloc`、`new`、`copy`、`mutableCopy` 與 `init` 回傳的物件本來就歸你所有，其他物件則會替你 retain。無論哪種情況，`Id` 最後都恰好持有一次 retain。

當某個 `Id` 的副本全部不存在時，垃圾回收器就會釋放它的物件，就像釋放其他值一樣。`release_now()` 會立即釋放物件，適用於持有稀缺資源（檔案、視窗）的物件；之後再使用任何副本都會觸發帶有訊息的 panic，而不會存取已釋放的記憶體。Objective-C 自動釋放（autorelease）的物件由自動釋放池釋放；會產生大量這類物件的迴圈，請用 `objc::autoreleasepool` 包起來：

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

`objc::WeakId::new(id)` 參照一個物件，但不讓它保持存活；物件消失後，`load()` 會回傳 `none`。用它來打破環狀參照，例如持有視窗、同時又被該視窗持有的委派物件（delegate）。

## Metal 運算核心函式

Metal 物件是以協定（`MTLDevice`、`MTLBuffer` 等）而非類別來識別：用 `#[objc::protocol(..)]` 宣告它們。這個程式從原始碼編譯一個核心函式，在 GPU 上對 1024 個數字執行，並檢查結果：

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
cmd.addCompletedHandler(|buf| { status.store(buf.status() as i64); });   // 在 Metal 執行緒上執行
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

`MTLCreateSystemDefaultDevice` 是 C 函式，宣告在 `extern "C"` 區塊中；它的結果是一個 `+1` 物件，由 `Id::from_retained` 接手。`MTLSize` 是一般的 `#[repr(C)]` 結構，以 C 的方式按值傳遞。

## 匯入 SDK 的標頭檔

上面的區塊都是手動宣告每個方法，但 Apple 的框架有數以千計的方法。`ffi::objc::import` 會讀取 SDK 本身的標頭檔，替你寫出這些區塊：

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

- 每個標頭檔都寫成 `Framework/Header.h`。把你要呼叫其方法的所有框架放進同一個匯入：`{ "Foundation/Foundation.h", "AppKit/AppKit.h", "Metal/Metal.h" }`。
- 繫結會放進包含該匯入的模組。類別的方法在那裡是 trait，所以用 `use ns::*;` 將它們帶入作用域，這同時也會帶入 `Id`、`NSString` 等。
- 標頭檔會標明哪些值可能是 nil：`lastObject` 回傳 `Id<NSObject> | none`，`uppercaseString` 回傳 `Id<NSString>`。最後一個 `NSError **` 會變成 `throws (objc::Error)`，`NS_OPTIONS` 型別可以使用 `|`、`&` 與 `contains`，而 `NSCocoaErrorDomain` 之類的常數則是函式。
- 第一次建置時會讀取標頭檔；繫結會寫入工具鏈的快取，之後的建置只讀取那一個檔案。`nabi bindgen -x objective-c <header>` 會印出相同的繫結，方便你查看匯入產生了什麼。

## 較新的 API

程式可以在從其部署目標（deployment target）起的每個 macOS 版本上執行；除非清單中寫了 `deployment_target: "13.0"`，否則部署目標是 11.0。標頭檔會標示後來才加入的 API，若某個呼叫可能在沒有該 API 的 macOS 上執行，編譯器就會拒絕：

```text
error[E0084]: `maximumConcurrentCompilationTaskCount` needs macOS 13.3, the deployment target is 11.0: wrap it in `if objc::available(macos: "13.3") { .. }`, or raise the deployment target (`deployment_target: "13.3"` in nabi.nabion, or `MACOSX_DEPLOYMENT_TARGET=13.3`)
```

在 `if objc::available(macos: "13.3") { .. }` 之內，這個呼叫是允許的；在較舊的 macOS 上，該區塊會被略過：

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

## 帶有委派物件的視窗

AppKit 應用程式把事件交給委派物件，也就是由你提供的類別所建立的物件。`ClassBuilder` 在執行時從 `extern "C"` 函式建立這樣的類別；每個函式都先接收接收者（receiver）與選擇器，接著才是方法的引數，而型別編碼（`"v@:@"`：不回傳任何值，接收一個物件）則告訴 Objective-C 它的簽章。這個程式開啟一個視窗，像點擊一樣從執行迴圈（run loop）關閉它，並在最後一個視窗關閉時結束：

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

- `setDelegate` 接收 `Id<NSApplicationDelegate>`：這裡建立的類別遵循該協定，而 `cast` 把這件事告訴 Nabi。
- `Block::new_main_thread` 從一個不是 `Send` 的閉包建立區塊，供主佇列使用；`dispatch_async` 從執行迴圈執行它。`Block::new` 接收 `Send` 閉包，可用於任何佇列。
- 視窗物件會依 AppKit 的要求在主執行緒上釋放，即使回收器是在其他執行緒上發現它們已不再使用。

## Metal 三角形

有了產生的繫結，Metal 就不需要任何自己的宣告。這個程式編譯兩個著色器，在 64x64 的紋理上畫出一個紅色三角形，再把像素讀回來：

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

    // 位於 GC 堆積之外：Metal 會寫入這裡。
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

`MTLCreateSystemDefaultDevice` 回傳 `Id<MTLDevice> | none`：標頭檔將它標示為可為 nil，而它的 `NS_RETURNS_RETAINED` 告訴 Nabi 結果已歸你所有。`MTLRegion` 與 `MTLClearColor` 之類的結構都是一般的 `#[repr(C)]` 結構，按值傳遞。

## 自己的區塊

接收區塊的 C 函式（例如 `dispatch_async`）會得到一個 `Block`。`bare::ffi::objc` 提供了 dispatch 佇列：

```nabi norun
use bare::ffi::objc::{Block, Queue, dispatch_async};

let total = Atomic::new(0);
for i in 1..=8 {
    dispatch_async(Queue::global(), Block::new(|| { total.fetch_add(i); }));
}
```

`Block::new` 需要 `Send` 閉包，因為佇列會在另一個執行緒上執行它。Objective-C 傳進來的區塊會以 `BlockRef` 的形式抵達，只在呼叫期間有效；`copy()` 可以保留它。

## 小結

- 在 `extern "objc"` 區塊上加 `#[objc::class(C)]` / `#[objc::class(C, extends(S))]` 會宣告一個類別；`#[objc::protocol(P)]` 宣告以協定識別的物件型別；`#[objc::category(C)]` 為已宣告的類別新增方法。
- 有 `self` 的方法位於 `Id<C>` 上，沒有 `self` 的位於 `C` 上；子類別會取得父類別的方法。
- 方法名稱就是它的選擇器，內部的 `:` 改寫為 `_`，或以 `#[selector("..")]` 明確給出。
- 結果：`Id<T>`（nil 會觸發 panic）、`Id<T> | none`；`throws (objc::Error)` 取代最後一個 `NSError **`。
- 閉包參數就是區塊。
- 一個 `Id` 持有一次 retain，在最後一個副本消失後由回收器歸還；`release_now()` 立即釋放物件，`objc::autoreleasepool` 清空 Objective-C 自動釋放的物件，`WeakId` 打破環狀參照。
- `ffi::objc::import { "Framework/Header.h", .. }` 從 SDK 的標頭檔寫出這些區塊；`use` 該模組的項目即可呼叫。
- 比部署目標更新的 API，要在 `if objc::available(macos: "..") { .. }` 之內呼叫。
- `ClassBuilder` 從 `extern "C"` 函式建立委派類別；`Block` 與 `dispatch_async` 在 dispatch 佇列上執行閉包。

## 延伸閱讀

- 標準函式庫規格：[Objective-C](../spec/refs-standard-libraries/100-bare/110-objc.html)、[FFI](../spec/refs-standard-libraries/100-bare/107-ffi.html)
- 前面的章節：[14. FFI 基礎](14-ffi.html)、[10. 錯誤處理](10-errors.html)
