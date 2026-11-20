# 28. iOS

Nabi builds for the iOS simulator on Apple silicon Macs (`aarch64-apple-ios-sim`). Builds for iPhones and iPads (`aarch64-apple-ios`) are in progress: that target is not tested on a device yet and comes in a later release, so for now an app with Nabi code runs in the simulator. An app's screens are Swift, so the Nabi part is a library that Xcode links in, and Swift calls it through the C ABI. Command-line programs also run on the simulator, which is how Nabi's own tests run there.

This chapter sets up Xcode, builds a library for an app, calls it from Swift, and runs a program on the simulator.

## Setting up

Building for iOS needs Xcode, not just the command line tools: only Xcode has the iOS SDKs and the simulator. Install it from the App Store, then point the tools at it and let it finish installing:

```text
$ sudo xcode-select -s /Applications/Xcode.app
$ sudo xcodebuild -runFirstLaunch
$ xcodebuild -downloadPlatform iOS
```

The last command downloads the iOS simulator, about 8 GB; a library for an app needs only the first two. Without Xcode, a build for an iOS target stops and says what to install:

```text
$ nabi build --target aarch64-apple-ios-sim
error[E0126]: the link failed:
no iPhoneSimulator SDK in /Library/Developer/CommandLineTools: install Xcode from the App Store, then run `sudo xcode-select -s /Applications/Xcode.app`, or set SDKROOT to an iPhoneSimulator.sdk
```

## A library for an app

Put the functions Swift calls in `src/lib.nabi` as `#[export_name("..")] pub extern "C" fn`s:

```nabi
#[export_name("calc_add")]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[export_name("calc_mean")]
pub extern "C" fn mean(n: i32) -> f64 {
    let xs: List<f64> = [];
    for i in 0..n { xs.push(i as f64); }
    let s = 0.0;
    for x in xs { s += x; }
    s / (n as f64)
}
```

and ask for a static library in `nabi.nabion`:

```nabion
Package {
    name: "calc",
    version: "0.1.0",
    lib: { kind: "staticlib", xcframework: true },
}
```

Build it for the simulator:

```text
$ nabi build --release --target aarch64-apple-ios-sim
```

Each build writes three files under `target/<target>/release/`:

- `static/libcalc.a`: the package, the standard library it uses and the runtime, in one archive.
- `include/calc.h`: the C declarations of the exported functions.
- `include/module.modulemap`: makes the header a module named `calc`, which Swift imports.

`xcframework: true` (or `--xcframework` on the command line) also puts the build into `target/release/calc.xcframework`, with a slice for the simulator. Each build adds its own slice, so once device builds come, a build for `aarch64-apple-ios` adds the device slice to the same XCFramework.

## Calling it from Swift

In the Xcode project, drag `calc.xcframework` into the app target's Frameworks, Libraries, and Embedded Content (Do Not Embed: it is a static library). Choose a simulator as the run destination; Xcode finds the headers and the module map in the slice. Then:

```text
import calc

let sum = calc_add(2, 3)        // Int32
let avg = calc_mean(10)         // Double
```

The Nabi runtime starts on the first call; nothing needs to set it up first. Calls work from any thread, but UIKit still wants its own calls on the main thread, so keep long Nabi work off it and send the result back with `DispatchQueue.main.async`. Values that cross are C values: numbers, pointers and `#[repr(C)]` structs.

The app's minimum iOS is 13.4. To require a newer one, give it in `nabi.nabion`:

```nabion
Package {
    name: "calc",
    deployment_target: { ios: "16.0" },
}
```

`IPHONEOS_DEPLOYMENT_TARGET` does the same for one build.

Objective-C APIs newer than that minimum work as on macOS ([chapter 27](27-objc.html)): the bindings mark them with the iOS version they need, and the compiler wants a call inside `if objc::available(ios: "16.4") { .. }`. One check can name both systems, `objc::available(macos: "13.3", ios: "16.4")`; each OS reads its own version.

## Programs on the simulator

A package with `src/main.nabi` builds into a program the simulator runs. Start a simulator device once (`xcrun simctl boot "iPhone 17"`, or open the Simulator app), then use `nabi run` and `nabi test` as usual:

```text
$ nabi run --target aarch64-apple-ios-sim
hello from the simulator
```

They run the program with `xcrun simctl spawn` on the booted device and pass it your environment variables. With several devices booted, `NABI_SIM_DEVICE` names the one to use.

## Devices

Device builds are in progress. A build for `aarch64-apple-ios` stops and names the target to use now:

```text
$ nabi build --target aarch64-apple-ios
nabi: error: target `aarch64-apple-ios` (iOS devices) is in progress: it is not tested on real hardware yet and comes in a later release; for now build for the simulator with `--target aarch64-apple-ios-sim`
```

When they come, expect what any iPhone or iPad app lives with: the system runs only what the app was signed with, inside its sandbox. An app cannot start other programs (`Command` throws `Unsupported`), nothing is loaded at run time (no plugins or hot reload), signing for a device is Xcode's job, and the main thread's stack is 1 MB, smaller than on a Mac.
