# 28. iOS

Nabi 可以在 Apple silicon Mac 上為 iOS 模擬器（`aarch64-apple-ios-sim`）建置。為 iPhone 和 iPad（`aarch64-apple-ios`）建置的支援仍在進行中：這個目標尚未在裝置上測試，會在之後的版本推出，所以目前含有 Nabi 程式碼的 app 只在模擬器中執行。app 的畫面以 Swift 撰寫，因此 Nabi 的部分是一個由 Xcode 連結進去的函式庫，Swift 透過 C ABI 呼叫它。命令列程式也能在模擬器上執行，Nabi 自己的測試就是這樣在上面執行的。

本章會設定 Xcode、為 app 建置函式庫、從 Swift 呼叫它，並在模擬器上執行程式。

## 環境設定

為 iOS 建置需要 Xcode，只有命令列工具（command line tools）是不夠的：只有 Xcode 才有 iOS SDK 和模擬器。從 App Store 安裝 Xcode，接著讓工具指向它，並讓它完成安裝：

```text
$ sudo xcode-select -s /Applications/Xcode.app
$ sudo xcodebuild -runFirstLaunch
$ xcodebuild -downloadPlatform iOS
```

最後一個指令會下載 iOS 模擬器，大約 8 GB；只為 app 建置函式庫的話，前兩個指令就夠了。沒有 Xcode 時，為 iOS 目標建置會停下來，並告訴你要安裝什麼：

```text
$ nabi build --target aarch64-apple-ios-sim
error[E0126]: the link failed:
no iPhoneSimulator SDK in /Library/Developer/CommandLineTools: install Xcode from the App Store, then run `sudo xcode-select -s /Applications/Xcode.app`, or set SDKROOT to an iPhoneSimulator.sdk
```

## 給 app 用的函式庫

把 Swift 要呼叫的函式寫在 `src/lib.nabi` 中，宣告為 `#[export_name("..")] pub extern "C" fn`：

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

並在 `nabi.nabion` 中要求產生靜態函式庫：

```nabion
Package {
    name: "calc",
    version: "0.1.0",
    lib: { kind: "staticlib", xcframework: true },
}
```

為模擬器建置：

```text
$ nabi build --release --target aarch64-apple-ios-sim
```

每次建置會在 `target/<target>/release/` 底下寫出三個檔案：

- `static/libcalc.a`：套件本身、它用到的標準函式庫以及執行環境，全部放在一個封存檔中。
- `include/calc.h`：匯出函式的 C 宣告。
- `include/module.modulemap`：把標頭檔變成名為 `calc` 的模組，供 Swift 匯入。

`xcframework: true`（或在命令列上加 `--xcframework`）還會把建置結果放進 `target/release/calc.xcframework`，其中含有一個給模擬器用的 slice。每次建置都會加入自己的 slice，所以等裝置建置推出後，為 `aarch64-apple-ios` 建置就會把裝置的 slice 加進同一個 XCFramework。

## 從 Swift 呼叫

在 Xcode 專案中，把 `calc.xcframework` 拖進 app 目標的 Frameworks, Libraries, and Embedded Content（選 Do Not Embed：它是靜態函式庫）。選一個模擬器作為執行目的地；Xcode 會在 slice 中找到標頭檔和 module map。接著：

```text
import calc

let sum = calc_add(2, 3)        // Int32
let avg = calc_mean(10)         // Double
```

Nabi 執行環境會在第一次呼叫時啟動，不需要事先設定任何東西。任何執行緒都可以呼叫，但 UIKit 仍然要求它自己的呼叫在主執行緒上進行，所以耗時的 Nabi 工作不要放在主執行緒上，並用 `DispatchQueue.main.async` 把結果送回去。跨越邊界的值都是 C 的值：數字、指標和 `#[repr(C)]` 結構。

app 的最低 iOS 版本是 13.4。若要求更新的版本，在 `nabi.nabion` 中指定：

```nabion
Package {
    name: "calc",
    deployment_target: { ios: "16.0" },
}
```

`IPHONEOS_DEPLOYMENT_TARGET` 可以對單次建置做同樣的事。

比這個最低版本更新的 Objective-C API，用法和在 macOS 上相同（[第 27 章](27-objc.html)）：繫結會標出它們需要的 iOS 版本，編譯器要求呼叫寫在 `if objc::available(ios: "16.4") { .. }` 之內。一次檢查可以同時指定兩個系統，`objc::available(macos: "13.3", ios: "16.4")`；每個作業系統會讀取自己的版本。

## 模擬器上的程式

含有 `src/main.nabi` 的套件會建置成模擬器可以執行的程式。先啟動一個模擬器裝置（`xcrun simctl boot "iPhone 17"`，或打開 Simulator app），之後照常使用 `nabi run` 和 `nabi test`：

```text
$ nabi run --target aarch64-apple-ios-sim
hello from the simulator
```

它們會用 `xcrun simctl spawn` 在已啟動的裝置上執行程式，並把你的環境變數傳給它。若同時啟動了多個裝置，用 `NABI_SIM_DEVICE` 指定要使用哪一個。

## 裝置

裝置建置仍在進行中。為 `aarch64-apple-ios` 建置會停下來，並指出目前該用的目標：

```text
$ nabi build --target aarch64-apple-ios
nabi: error: target `aarch64-apple-ios` (iOS devices) is in progress: it is not tested on real hardware yet and comes in a later release; for now build for the simulator with `--target aarch64-apple-ios-sim`
```

等它推出後，你會遇到所有 iPhone 或 iPad app 都要面對的限制：系統只執行 app 簽署時包含的內容，而且都在它的沙箱之內。app 不能啟動其他程式（`Command` 會擲回 `Unsupported`），執行時不會載入任何東西（沒有外掛，也沒有熱重載），為裝置簽署是 Xcode 的工作，而且主執行緒的堆疊是 1 MB，比 Mac 上小。
