# 27. Objective-C 与 Apple 框架

在 macOS 上，系统自带的库，如 Foundation、AppKit、Metal 等，都是 Objective-C 类。Nabi 可以直接调用它们：在 `extern "objc"` 块中声明要用到的类和方法，每个方法就成为一个普通的 Nabi 方法，调用时发送对应的 Objective-C 消息。整个过程不需要 C 编译器，也不需要胶水代码。对于 Apple 的框架，你很少需要亲手写这些块：`ffi::objc::import` 会读取 SDK 的头文件并替你生成（[导入 SDK 的头文件](#导入-sdk-的头文件)）。前面几节先手写声明，借此说明导入能为你提供什么。

本章的内容只适用于 Apple 目标（`aarch64-apple-darwin`）。构建网站时不会运行本章的代码块；每一段都在 Mac 上编译运行过，并给出了输出。

## 声明类

一个 `extern "objc"` 块声明一个类。`#[objc::class(NSArray)]` 生成类型 `NSArray`；它的方法写法与 `extern "C"` 函数相同，实例方法带 `self`：

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

- 不带 `self` 的方法是类方法，在类型上调用：`NSMutableArray::array()`。
- `extends(NSArray)` 让 `NSArray` 成为超类，`NSMutableArray` 因此获得它的方法：`count` 和 `componentsJoinedByString` 都可以用在 `fruit` 上。一个类的方法是为它及其子类实现的 trait，所以在其他模块中需要 `use` 该模块的项。不写 `extends` 时超类是 `NSObject`，它的方法（`description`、`isEqual`、`hash` 等）每个对象都有。
- `Id<Self>` 表示调用该方法的那个类，即 Objective-C 的 `instancetype`。
- 对象的类型是 `Id<T>`。声明为 `Id<NSObject>` 的参数接受任何对象，所以 `ns(f)` 这个 `Id<NSString>` 可以直接传进去。
- `{x}` 打印对象的 `description`，与 `NSLog` 的输出相同。

## 方法名就是选择器

方法名就是它的 Objective-C 选择器（selector），选择器各部分之间的 `:` 写成 `_`：`componentsJoinedByString(sep)` 发送 `componentsJoinedByString:`，`initWithBytes_length_encoding(b, n, e)` 则发送 `initWithBytes:length:encoding:`。因此 Apple 的文档可以一一对照着读。想让 Nabi 中的名字不同时，直接给出选择器：

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

`NSString` 已由 `bare::ffi::objc` 声明，所以这个块是一个*类别*（category）：`#[objc::category(NSString)]` 为在别处声明的类添加方法。`ns("text")` 从 `str` 创建一个 `NSString`，`to_str()` 再把它复制回来。

如果名字的各部分与参数对不上，编译器会指出问题，并给出两种正确写法：

```text
error[E0084]: `initWithBytes_length` has 3 parameter(s), so its selector has 3 part(s): name it after the selector with `_` between the parts (`initWithBytes_length` for `initWithBytes:length:`), or give it with `#[selector("..")]`
```

## nil、`none` 与错误

声明返回 `Id<T>` 的方法如果得到 nil，就会触发 panic，并指明是哪个方法。如果 nil 是正常的结果，就声明为 `Id<T> | none`：

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

Cocoa 通过最后一个 `NSError **` 参数报告错误。省去这个参数，改写 `throws (objc::Error)`：调用时就需要 `try`，错误的捕获方式与其他错误相同（[第 10 章](10-errors.html)）。名字保留 `_error` 部分，因为选择器里有它。

## 块就是闭包

闭包类型的参数传递的是 Objective-C 块（block）。像传给任何 Nabi 函数一样传入闭包即可：

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

只要 Objective-C 还持有这个块，闭包就一直存活。完成处理程序（completion handler）可能在其他线程上运行；只与它共享线程安全的值，例如 `Atomic`、`Mutex` 或通道。

## 谁来释放对象

一个 `Id<T>` 持有其对象的一次 retain，`Id` 的各个副本共享这一次 retain。Nabi 遵循 Cocoa 的命名规则：`alloc`、`new`、`copy`、`mutableCopy` 和 `init` 返回的对象本来就归你所有，其他方法返回的对象会替你 retain。无论哪种情况，`Id` 最终都恰好持有一次 retain。

当一个 `Id` 的副本都不在了，垃圾回收器就会释放它的对象，与它释放其他值一样。对于持有稀缺资源（文件、窗口）的对象，`release_now()` 会立即释放；之后再使用任何副本都会触发 panic 并给出消息，而不会访问已释放的内存。Objective-C 自动释放（autorelease）的对象由自动释放池释放；创建大量此类对象的循环，用 `objc::autoreleasepool` 包起来：

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

`objc::WeakId::new(id)` 引用一个对象，但不让它保持存活；对象消失后，`load()` 返回 `none`。用它打破循环引用，例如 delegate 持有窗口，而窗口又持有这个 delegate。

## Metal 计算内核

Metal 对象是通过协议（`MTLDevice`、`MTLBuffer` 等）而不是类来识别的：用 `#[objc::protocol(..)]` 声明它们。下面的程序从源代码编译一个内核，在 GPU 上对 1024 个数运行它，并检查结果：

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
cmd.addCompletedHandler(|buf| { status.store(buf.status() as i64); });   // 在 Metal 线程上运行
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

`MTLCreateSystemDefaultDevice` 是一个 C 函数，在 `extern "C"` 块中声明；它的结果是一个 `+1` 对象，由 `Id::from_retained` 接管。`MTLSize` 是普通的 `#[repr(C)]` 结构体，像 C 一样按值传递。

## 导入 SDK 的头文件

上面的块都是手写每个方法的声明，但 Apple 的框架有成千上万个方法。`ffi::objc::import` 读取 SDK 自带的头文件，替你写好这些块：

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

- 每个头文件写成 `Framework/Header.h`。把所有要调用其方法的框架放进同一个导入中：`{ "Foundation/Foundation.h", "AppKit/AppKit.h", "Metal/Metal.h" }`。
- 绑定会放进包含该导入的模块中。类的方法在那里是 trait，所以用 `use ns::*;` 把它们引入作用域，这同时也引入了 `Id`、`NSString` 等。
- 头文件说明了哪些值可能是 nil：`lastObject` 返回 `Id<NSObject> | none`，`uppercaseString` 返回 `Id<NSString>`。最后一个 `NSError **` 变成 `throws (objc::Error)`，`NS_OPTIONS` 类型支持 `|`、`&` 和 `contains`，`NSCocoaErrorDomain` 这类常量则是函数。
- 第一次构建会读取头文件，并把绑定写入工具链的缓存，之后的构建只读这一个文件。`nabi bindgen -x objective-c <header>` 会打印同样的绑定，可以用来查看导入为你生成了什么。

## 较新的 API

程序可以在其部署目标（deployment target）及之后的每个 macOS 版本上运行；除非清单写了 `deployment_target: "13.0"`，部署目标默认为 11.0。头文件标出了后来才加入的 API，如果某个调用可能运行在没有该 API 的 macOS 上，编译器会拒绝它：

```text
error[E0084]: `maximumConcurrentCompilationTaskCount` needs macOS 13.3, the deployment target is 11.0: wrap it in `if objc::available(macos: "13.3") { .. }`, or raise the deployment target (`deployment_target: "13.3"` in nabi.nabion, or `MACOSX_DEPLOYMENT_TARGET=13.3`)
```

在 `if objc::available(macos: "13.3") { .. }` 内部允许这样的调用，在较旧的 macOS 上会跳过这个块：

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

## 带 delegate 的窗口

AppKit 应用把事件交给 delegate，也就是你提供的某个类的对象。`ClassBuilder` 在运行时用 `extern "C"` 函数创建这样的类；每个函数先接收接收者和选择器，然后是方法的参数，类型编码（`"v@:@"`：不返回值，接收一个对象）告诉 Objective-C 它的签名。下面的程序打开一个窗口，像点击关闭按钮那样从运行循环中关闭它，并随着最后一个窗口关闭而结束：

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

- `setDelegate` 接受 `Id<NSApplicationDelegate>`：这里创建的类遵循该协议，`cast` 把这一点告诉 Nabi。
- `Block::new_main_thread` 用一个不是 `Send` 的闭包为主队列创建块；`dispatch_async` 从运行循环中运行它。`Block::new` 接受 `Send` 闭包，可用于任何队列。
- 窗口对象在主线程上释放，这是 AppKit 的要求，即使回收器是在其他线程上发现它们已不再使用。

## Metal 三角形

有了生成的绑定，Metal 就不需要任何自己的声明。下面的程序编译两个着色器，在一张 64x64 的纹理上画一个红色三角形，然后读回像素：

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

    // 在 GC 堆之外：Metal 会写入这里。
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

`MTLCreateSystemDefaultDevice` 返回 `Id<MTLDevice> | none`：头文件把它标为可为空（nullable），它的 `NS_RETURNS_RETAINED` 告诉 Nabi 结果已经归你所有。`MTLRegion` 和 `MTLClearColor` 这样的结构体是普通的 `#[repr(C)]` 结构体，按值传递。

## 自己创建块

接受块的 C 函数，例如 `dispatch_async`，接收一个 `Block`。`bare::ffi::objc` 提供了各个调度队列（dispatch queue）：

```nabi norun
use bare::ffi::objc::{Block, Queue, dispatch_async};

let total = Atomic::new(0);
for i in 1..=8 {
    dispatch_async(Queue::global(), Block::new(|| { total.fetch_add(i); }));
}
```

`Block::new` 需要一个 `Send` 闭包，因为队列会在其他线程上运行它。Objective-C 传进来的块以 `BlockRef` 的形式到达，仅在调用期间有效；用 `copy()` 可以保留它。

## 小结

- 在 `extern "objc"` 块上写 `#[objc::class(C)]` / `#[objc::class(C, extends(S))]` 声明一个类；`#[objc::protocol(P)]` 为通过协议识别的对象声明类型；`#[objc::category(C)]` 为已声明的类添加方法。
- 带 `self` 的方法在 `Id<C>` 上，不带的在 `C` 上；子类获得超类的方法。
- 方法名是它的选择器，内部的 `:` 换成 `_`，也可以用 `#[selector("..")]` 给出。
- 结果：`Id<T>`（nil 时触发 panic）、`Id<T> | none`；`throws (objc::Error)` 取代最后一个 `NSError **`。
- 闭包参数就是块。
- 一个 `Id` 持有一次 retain，最后一个副本消失后由回收器归还；`release_now()` 立即释放对象，`objc::autoreleasepool` 清空 Objective-C 自动释放的对象，`WeakId` 打破循环引用。
- `ffi::objc::import { "Framework/Header.h", .. }` 根据 SDK 的头文件写出这些块；`use` 该模块的项即可调用。
- 比部署目标新的 API 要在 `if objc::available(macos: "..") { .. }` 内部调用。
- `ClassBuilder` 用 `extern "C"` 函数创建 delegate 类；`Block` 和 `dispatch_async` 在调度队列上运行闭包。

## 延伸阅读

- 标准库规范：[Objective-C](../spec/refs-standard-libraries/100-bare/110-objc.html)、[FFI](../spec/refs-standard-libraries/100-bare/107-ffi.html)
- [14. FFI 基础](14-ffi.html)、[10. 错误处理](10-errors.html)
- 下一章：[28. iOS](28-ios.html)
