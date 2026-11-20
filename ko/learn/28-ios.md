# 28. iOS

Nabi는 Apple silicon Mac의 iOS 시뮬레이터(`aarch64-apple-ios-sim`)를 대상으로 빌드합니다. iPhone과 iPad용 빌드(`aarch64-apple-ios`)는 진행 중입니다. 이 타깃은 아직 기기에서 테스트하지 않았고 이후 릴리스에서 나오므로, 지금은 Nabi 코드가 든 앱을 시뮬레이터에서 실행합니다. 앱의 화면은 Swift로 만들므로, Nabi 부분은 Xcode가 링크해 넣는 라이브러리가 되고 Swift는 C ABI로 그것을 부릅니다. 명령줄 프로그램도 시뮬레이터에서 돌며, Nabi 자체 테스트도 그렇게 돕니다.

이 장에서는 Xcode를 준비하고, 앱용 라이브러리를 빌드해 Swift에서 부르고, 시뮬레이터에서 프로그램을 실행합니다.

## 준비

iOS용 빌드에는 명령줄 도구만으로는 안 되고 Xcode가 필요합니다. iOS SDK와 시뮬레이터는 Xcode에만 있습니다. App Store에서 설치한 뒤, 도구가 Xcode를 가리키게 하고 남은 설치를 마칩니다.

```text
$ sudo xcode-select -s /Applications/Xcode.app
$ sudo xcodebuild -runFirstLaunch
$ xcodebuild -downloadPlatform iOS
```

마지막 명령은 약 8 GB인 iOS 시뮬레이터를 내려받습니다. 앱용 라이브러리만 만든다면 앞의 두 명령이면 됩니다. Xcode가 없으면 iOS 타깃 빌드는 멈추고 무엇을 설치할지 알려 줍니다.

```text
$ nabi build --target aarch64-apple-ios-sim
error[E0126]: the link failed:
no iPhoneSimulator SDK in /Library/Developer/CommandLineTools: install Xcode from the App Store, then run `sudo xcode-select -s /Applications/Xcode.app`, or set SDKROOT to an iPhoneSimulator.sdk
```

## 앱을 위한 라이브러리

Swift가 부를 함수를 `src/lib.nabi`에 `#[export_name("..")] pub extern "C" fn`으로 둡니다.

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

그리고 `nabi.nabion`에서 정적 라이브러리를 요청합니다.

```nabion
Package {
    name: "calc",
    version: "0.1.0",
    lib: { kind: "staticlib", xcframework: true },
}
```

시뮬레이터용으로 빌드합니다.

```text
$ nabi build --release --target aarch64-apple-ios-sim
```

빌드마다 `target/<target>/release/` 아래에 파일 세 개를 씁니다.

- `static/libcalc.a`: 패키지, 패키지가 쓰는 표준 라이브러리, 런타임을 담은 아카이브 하나.
- `include/calc.h`: export한 함수의 C 선언.
- `include/module.modulemap`: 헤더를 `calc`라는 모듈로 만들어 Swift가 import합니다.

`xcframework: true`(명령줄에서는 `--xcframework`)는 빌드를 `target/release/calc.xcframework`에도 넣습니다. 시뮬레이터용 슬라이스가 들어 있습니다. 빌드마다 자기 슬라이스를 넣으므로, 기기용 빌드가 나오면 `aarch64-apple-ios`용 빌드가 같은 XCFramework에 기기용 슬라이스를 더합니다.

## Swift에서 부르기

Xcode 프로젝트에서 `calc.xcframework`를 앱 타깃의 Frameworks, Libraries, and Embedded Content로 끌어다 놓습니다(정적 라이브러리이므로 Do Not Embed). 실행 목적지로 시뮬레이터를 고르면 Xcode가 슬라이스 안의 헤더와 module map을 찾습니다. 그러면 이렇게 씁니다.

```text
import calc

let sum = calc_add(2, 3)        // Int32
let avg = calc_mean(10)         // Double
```

Nabi 런타임은 첫 호출에서 시작하므로 먼저 설정할 것은 없습니다. 어느 스레드에서 불러도 되지만 UIKit 호출은 여전히 메인 스레드에서 해야 하므로, 오래 걸리는 Nabi 작업은 메인 스레드 밖에서 돌리고 결과는 `DispatchQueue.main.async`로 돌려보내세요. 경계를 넘는 값은 C 값입니다. 숫자, 포인터, `#[repr(C)]` 구조체입니다.

앱의 최소 iOS는 13.4입니다. 더 새 버전을 요구하려면 `nabi.nabion`에 적습니다.

```nabion
Package {
    name: "calc",
    deployment_target: { ios: "16.0" },
}
```

빌드 한 번에만 바꾸려면 `IPHONEOS_DEPLOYMENT_TARGET`을 씁니다.

그 최소 버전보다 새로운 Objective-C API는 macOS에서와 같이 다룹니다([27장](27-objc.html)). 바인딩은 그 API가 필요로 하는 iOS 버전을 표시하고, 컴파일러는 호출을 `if objc::available(ios: "16.4") { .. }` 안에 두라고 합니다. 검사 하나에 두 OS를 함께 쓸 수 있습니다. `objc::available(macos: "13.3", ios: "16.4")`에서 각 OS는 자기 버전만 읽습니다.

## 시뮬레이터에서 프로그램 실행

`src/main.nabi`가 있는 패키지는 시뮬레이터가 실행하는 프로그램으로 빌드됩니다. 시뮬레이터 기기를 한 번 켜 두고(`xcrun simctl boot "iPhone 17"`, 또는 Simulator 앱을 엽니다) 평소처럼 `nabi run`과 `nabi test`를 씁니다.

```text
$ nabi run --target aarch64-apple-ios-sim
hello from the simulator
```

이 명령들은 부팅된 기기에서 `xcrun simctl spawn`으로 프로그램을 실행하고 환경 변수를 넘깁니다. 여러 기기가 켜져 있으면 `NABI_SIM_DEVICE`로 쓸 기기를 지정합니다.

## 기기

기기용 빌드는 진행 중입니다. `aarch64-apple-ios`용 빌드는 멈추고 지금 쓸 타깃을 알려 줍니다.

```text
$ nabi build --target aarch64-apple-ios
nabi: error: target `aarch64-apple-ios` (iOS devices) is in progress: it is not tested on real hardware yet and comes in a later release; for now build for the simulator with `--target aarch64-apple-ios-sim`
```

기기용 빌드가 나오면 iPhone과 iPad 앱이 늘 따르는 제약이 그대로 있습니다. 시스템은 앱이 서명한 것만, 앱의 샌드박스 안에서 실행합니다. 앱은 다른 프로그램을 시작할 수 없고(`Command`는 `Unsupported`를 던짐), 실행 중에 코드를 불러오지 않으며(플러그인과 핫 리로드 없음), 기기용 서명은 Xcode가 맡고, 메인 스레드의 스택은 1 MB로 Mac보다 작습니다.
