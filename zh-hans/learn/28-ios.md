# 28. iOS

Nabi 可以在 Apple silicon Mac 上为 iOS 模拟器构建（`aarch64-apple-ios-sim`）。面向 iPhone 和 iPad 的构建（`aarch64-apple-ios`）还在进行中：这个目标尚未在真实设备上测试，会在以后的版本中提供，所以目前含有 Nabi 代码的应用只能在模拟器中运行。应用的界面用 Swift 编写，因此 Nabi 部分是一个由 Xcode 链接进来的库，Swift 通过 C ABI 调用它。命令行程序也能在模拟器上运行，Nabi 自己的测试就是这样在模拟器上运行的。

本章配置 Xcode，为应用构建一个库，从 Swift 调用它，并在模拟器上运行一个程序。

## 准备工作

为 iOS 构建需要 Xcode，仅有命令行工具是不够的：只有 Xcode 带有 iOS SDK 和模拟器。从 App Store 安装 Xcode，然后让工具指向它，并让它完成安装：

```text
$ sudo xcode-select -s /Applications/Xcode.app
$ sudo xcodebuild -runFirstLaunch
$ xcodebuild -downloadPlatform iOS
```

最后一条命令下载 iOS 模拟器，大约 8 GB；为应用构建库只需要前两条。没有 Xcode 时，面向 iOS 目标的构建会停止，并告诉你需要安装什么：

```text
$ nabi build --target aarch64-apple-ios-sim
error[E0126]: the link failed:
no iPhoneSimulator SDK in /Library/Developer/CommandLineTools: install Xcode from the App Store, then run `sudo xcode-select -s /Applications/Xcode.app`, or set SDKROOT to an iPhoneSimulator.sdk
```

## 为应用构建的库

把 Swift 要调用的函数放在 `src/lib.nabi` 中，写成 `#[export_name("..")] pub extern "C" fn`：

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

然后在 `nabi.nabion` 中要求生成静态库：

```nabion
Package {
    name: "calc",
    version: "0.1.0",
    lib: { kind: "staticlib", xcframework: true },
}
```

为模拟器构建：

```text
$ nabi build --release --target aarch64-apple-ios-sim
```

每次构建都会在 `target/<target>/release/` 下写入三个文件：

- `static/libcalc.a`：包本身、它用到的标准库和运行时，打包在一个归档文件中。
- `include/calc.h`：导出函数的 C 声明。
- `include/module.modulemap`：把头文件变成名为 `calc` 的模块，供 Swift 导入。

`xcframework: true`（或命令行上的 `--xcframework`）还会把构建结果放进 `target/release/calc.xcframework`，其中有一个模拟器切片。每次构建都会添加它自己的切片，所以等设备构建可用后，为 `aarch64-apple-ios` 构建会把设备切片加进同一个 XCFramework。

## 从 Swift 调用

在 Xcode 项目中，把 `calc.xcframework` 拖进应用目标的 Frameworks, Libraries, and Embedded Content（选择 Do Not Embed：它是静态库）。选择一个模拟器作为运行目标；Xcode 会在切片中找到头文件和模块映射。然后：

```text
import calc

let sum = calc_add(2, 3)        // Int32
let avg = calc_mean(10)         // Double
```

Nabi 运行时在第一次调用时启动，不需要事先做任何设置。调用可以来自任何线程，但 UIKit 仍然要求它自己的调用在主线程上进行，所以不要在主线程上做耗时的 Nabi 工作，用 `DispatchQueue.main.async` 把结果送回主线程。跨越边界的值是 C 值：数字、指针和 `#[repr(C)]` 结构体。

应用的最低 iOS 版本是 13.4。要求更新的版本时，在 `nabi.nabion` 中指定：

```nabion
Package {
    name: "calc",
    deployment_target: { ios: "16.0" },
}
```

`IPHONEOS_DEPLOYMENT_TARGET` 对单次构建起同样的作用。

比这个最低版本更新的 Objective-C API 用法与 macOS 上相同（[第 27 章](27-objc.html)）：绑定会标出它们需要的 iOS 版本，编译器要求把调用放在 `if objc::available(ios: "16.4") { .. }` 中。一次检查可以同时写出两个系统，`objc::available(macos: "13.3", ios: "16.4")`；每个操作系统读取自己的版本。

## 模拟器上的程序

含有 `src/main.nabi` 的包会构建成可在模拟器上运行的程序。先启动一次模拟器设备（`xcrun simctl boot "iPhone 17"`，或者打开 Simulator 应用），然后像平常一样使用 `nabi run` 和 `nabi test`：

```text
$ nabi run --target aarch64-apple-ios-sim
hello from the simulator
```

它们在已启动的设备上用 `xcrun simctl spawn` 运行程序，并把你的环境变量传给它。启动了多个设备时，用 `NABI_SIM_DEVICE` 指定要使用的那一个。

## 设备

设备构建还在进行中。为 `aarch64-apple-ios` 构建会停止，并给出目前应使用的目标：

```text
$ nabi build --target aarch64-apple-ios
nabi: error: target `aarch64-apple-ios` (iOS devices) is in progress: it is not tested on real hardware yet and comes in a later release; for now build for the simulator with `--target aarch64-apple-ios-sim`
```

设备构建可用之后，会面临所有 iPhone 和 iPad 应用都有的限制：系统只运行应用签名时包含的内容，并且运行在沙箱中。应用不能启动其他程序（`Command` 抛出 `Unsupported`），运行时不加载任何东西（没有插件或热重载），为设备签名由 Xcode 负责，主线程的栈是 1 MB，比 Mac 上小。
