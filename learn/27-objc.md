# 27. Objective-C and Apple Frameworks

On macOS the system's own libraries, Foundation, AppKit, Metal and the rest, are Objective-C classes. Nabi calls them directly: you declare the classes and methods you use in an `extern "objc"` block, and each method becomes an ordinary Nabi method that sends the Objective-C message. No C compiler or glue code is involved. For Apple's frameworks you rarely write those blocks yourself: `ffi::objc::import` reads the SDK's headers and writes them ([Importing the SDK's headers](#importing-the-sdks-headers)). The first sections declare by hand, which shows what an import gives you.

Everything in this chapter works only on Apple targets (`aarch64-apple-darwin`). Its code blocks are not run when the site is built; each was compiled and run on a Mac, with the output shown.

## Declaring a class

An `extern "objc"` block declares one class. `#[objc::class(NSArray)]` makes the type `NSArray`; its methods are written like `extern "C"` functions, with `self` for instance methods:

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

- A method without `self` is a class method, called on the type: `NSMutableArray::array()`.
- `extends(NSArray)` makes `NSArray` the superclass, and `NSMutableArray` gets its methods: `count` and `componentsJoinedByString` work on `fruit`. A class's methods are traits implemented for it and its subclasses, so in another module they need `use` of that module's items. Without `extends` the superclass is `NSObject`, whose methods (`description`, `isEqual`, `hash`, ..) every object has.
- `Id<Self>` is the class the method is called on, Objective-C's `instancetype`.
- Objects are `Id<T>`. A parameter declared `Id<NSObject>` takes any object, so `ns(f)`, an `Id<NSString>`, goes straight in.
- `{x}` prints an object's `description`, as `NSLog` would.

## Method names are selectors

The name of a method is its Objective-C selector, with `_` where the selector has a `:` between parts: `componentsJoinedByString(sep)` sends `componentsJoinedByString:`, and `initWithBytes_length_encoding(b, n, e)` would send `initWithBytes:length:encoding:`. So Apple's documentation reads one to one. When the Nabi name should differ, give the selector:

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

`NSString` is already declared by `bare::ffi::objc`, so this block is a *category*: `#[objc::category(NSString)]` adds methods to a class declared elsewhere. `ns("text")` makes an `NSString` from a `str` and `to_str()` copies one back.

If a name's parts do not match its parameters, the compiler says so and shows both ways to write it:

```text
error[E0084]: `initWithBytes_length` has 3 parameter(s), so its selector has 3 part(s): name it after the selector with `_` between the parts (`initWithBytes_length` for `initWithBytes:length:`), or give it with `#[selector("..")]`
```

## nil, `none` and errors

A method declared to return `Id<T>` panics if it gets nil, naming the method. When nil is a normal answer, declare `Id<T> | none`:

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

Cocoa reports errors through a last `NSError **` parameter. Leave it out and write `throws (objc::Error)`: the call then needs `try`, and the error is caught like any other ([chapter 10](10-errors.html)). The name keeps the `_error` part, because the selector does.

## Blocks are closures

A parameter of a closure type passes an Objective-C block. Pass a closure as you would to any Nabi function:

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

The closure lives as long as Objective-C keeps the block. A completion handler may run on another thread; share only thread-safe values with it, such as an `Atomic`, a `Mutex` or a channel.

## Who frees an object

An `Id<T>` holds one retain of its object, and copies of the `Id` share it. Nabi follows Cocoa's naming rule: what `alloc`, `new`, `copy`, `mutableCopy` and `init` return is yours already, everything else is retained for you. Either way the `Id` ends up with exactly one retain.

When no copy of an `Id` is left, the garbage collector releases its object, like any other value it frees. `release_now()` releases it at once, for an object that holds something scarce (a file, a window); using any copy afterwards panics with a message instead of touching freed memory. Objects Objective-C autoreleases are released by an autorelease pool; wrap loops that make many of them in `objc::autoreleasepool`:

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

`objc::WeakId::new(id)` refers to an object without keeping it alive; `load()` gives `none` once it is gone. Use it to break a cycle, such as a delegate that holds the window holding it.

## A Metal compute kernel

Metal objects are known by protocols (`MTLDevice`, `MTLBuffer`, ..) rather than classes: declare them with `#[objc::protocol(..)]`. This program compiles a kernel from source, runs it on the GPU over 1024 numbers and checks the result:

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
cmd.addCompletedHandler(|buf| { status.store(buf.status() as i64); });   // runs on a Metal thread
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

`MTLCreateSystemDefaultDevice` is a C function, declared in an `extern "C"` block; its result is a `+1` object, which `Id::from_retained` takes over. `MTLSize` is a plain `#[repr(C)]` struct, passed by value as C passes it.

## Importing the SDK's headers

Declaring every method by hand is how the blocks above work, but Apple's frameworks have thousands of them. `ffi::objc::import` reads the SDK's own headers and writes those blocks for you:

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

- Name each header as `Framework/Header.h`. Put every framework whose methods you call in one import: `{ "Foundation/Foundation.h", "AppKit/AppKit.h", "Metal/Metal.h" }`.
- The bindings go into the module that holds the import. A class's methods are traits there, so bring them into scope with `use ns::*;`, which also brings `Id`, `NSString` and the rest.
- The headers say what may be nil: `lastObject` returns `Id<NSObject> | none`, `uppercaseString` an `Id<NSString>`. A last `NSError **` becomes `throws (objc::Error)`, `NS_OPTIONS` types take `|`, `&` and `contains`, and constants such as `NSCocoaErrorDomain` are functions.
- The first build reads the headers; it writes the bindings into the toolchain's cache, and later builds read that one file. `nabi bindgen -x objective-c <header>` prints the same bindings, to read what an import gives you.

## Newer APIs

A program runs on every macOS from its deployment target on, 11.0 unless the manifest says `deployment_target: "13.0"`. The headers mark APIs that came later, and the compiler refuses a call that might run on a macOS without it:

```text
error[E0084]: `maximumConcurrentCompilationTaskCount` needs macOS 13.3, the deployment target is 11.0: wrap it in `if objc::available(macos: "13.3") { .. }`, or raise the deployment target (`deployment_target: "13.3"` in nabi.nabion, or `MACOSX_DEPLOYMENT_TARGET=13.3`)
```

Inside `if objc::available(macos: "13.3") { .. }` the call is allowed, and on an older macOS the block is skipped:

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

## A window with a delegate

An AppKit app hands events to a delegate, an object of a class you provide. `ClassBuilder` makes such a class at run time from `extern "C"` functions; each takes the receiver and the selector first, then the method's arguments, and the type encoding (`"v@:@"`: returns nothing, takes an object) tells Objective-C its signature. This program opens a window, closes it from the run loop as a click would, and ends with its last window:

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

- `setDelegate` takes an `Id<NSApplicationDelegate>`: the class made here conforms to the protocol, which `cast` tells Nabi.
- `Block::new_main_thread` makes a block from a closure that is not `Send`, for the main queue; `dispatch_async` runs it from the run loop. `Block::new` takes a `Send` closure for any queue.
- Window objects are released on the main thread, as AppKit requires, even when the collector finds them dead on another.

## A Metal triangle

With the generated bindings Metal needs no declarations of its own. This program compiles two shaders, draws one red triangle into a 64x64 texture and reads the pixels back:

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

    // Outside the GC heap: Metal writes into it.
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

`MTLCreateSystemDefaultDevice` returns `Id<MTLDevice> | none`: the header marks it nullable, and its `NS_RETURNS_RETAINED` tells Nabi the result is already owned. Structs such as `MTLRegion` and `MTLClearColor` are plain `#[repr(C)]` structs, passed by value.

## Blocks of your own

A C function that takes a block, such as `dispatch_async`, gets a `Block`. `bare::ffi::objc` has the dispatch queues:

```nabi norun
use bare::ffi::objc::{Block, Queue, dispatch_async};

let total = Atomic::new(0);
for i in 1..=8 {
    dispatch_async(Queue::global(), Block::new(|| { total.fetch_add(i); }));
}
```

`Block::new` needs a `Send` closure, since the queue runs it on another thread. A block Objective-C passes in arrives as a `BlockRef`, valid during the call; `copy()` keeps it.

## Summary

- `#[objc::class(C)]` / `#[objc::class(C, extends(S))]` on an `extern "objc"` block declares a class; `#[objc::protocol(P)]` a type for objects known by a protocol; `#[objc::category(C)]` adds methods to a declared class.
- Methods with `self` are on `Id<C>`, those without on `C`; a subclass gets its superclass's methods.
- A method's name is its selector with `_` for the inner `:`s, or `#[selector("..")]` gives it.
- Results: `Id<T>` (nil panics), `Id<T> | none`; `throws (objc::Error)` replaces a last `NSError **`.
- A closure parameter is a block.
- An `Id` holds one retain, which the collector gives back after the last copy; `release_now()` frees the object at once, `objc::autoreleasepool` drains what Objective-C autoreleased, `WeakId` breaks cycles.
- `ffi::objc::import { "Framework/Header.h", .. }` writes the blocks from the SDK's headers; `use` the module's items to call them.
- An API newer than the deployment target is called inside `if objc::available(macos: "..") { .. }`.
- `ClassBuilder` makes a delegate class from `extern "C"` functions; `Block` and `dispatch_async` run closures on dispatch queues.

## Further reading

- Standard library spec: [Objective-C](../spec/refs-standard-libraries/100-bare/110-objc.html), [FFI](../spec/refs-standard-libraries/100-bare/107-ffi.html)
- [14. FFI Basics](14-ffi.html), [10. Error Handling](10-errors.html)
- Next chapter: [28. iOS](28-ios.html)
