# 23. 外掛

外掛（plugin）是程式在執行時載入的套件：遊戲關卡、編輯器工具、使用者放進資料夾裡的濾鏡。在 Nabi 中，外掛就是一般的函式庫套件，只是清單裡多寫一行。它在你的程式的執行環境上執行，所以它回傳的一切，不論是字串、串列、閉包或 `any` 值，都是你的程式中的一般值，而你呼叫它的函式也和呼叫其他函式沒有兩樣。

本章會建立一個程式，載入外掛、呼叫它，然後將它卸載。

## 外掛套件

外掛是一個函式庫：有 `src/lib.nabi`，沒有 `src/main.nabi`。在它的 `nabi.nabion` 中寫上 `lib: { kind: "plugin" }`，`nabi build` 就會把它連結成外掛。

```nabion
Package {
    name: "greeter",
    version: "0.1.0",
    lib: { kind: "plugin" },
    deps: {},
}
```

程式能從外掛取得的，就是它的 `pub fn`：

```nabi norun
pub fn greet(name: str) -> str {
    f"hello, {name}"
}

pub fn squares(n: i32) -> List<i32> {
    let xs: List<i32> = List::new();
    for i in 0..n {
        xs.push(i * i);
    }
    xs
}

pub fn counter() -> fn() -> i32 {
    let n = [0];
    || {
        n[0] += 1;
        n[0]
    }
}
```

```bash
nabi build
```

外掛在 macOS 上是 `target/debug/libgreeter.dylib`，在 Linux 上是 `libgreeter.so`，在 Windows 上是 `greeter.dll`。`std::plugin::file("greeter")` 會依你建置的平台給出這個檔名。

## 載入外掛

程式用 `plugin::load` 載入檔案，再以路徑和型別逐一取得函式。型別會經過檢查：外掛宣告的是 `fn(str) -> str`，你卻要求 `fn(i32) -> str`，得到的是錯誤，而不是當機。

```nabi norun
use std::plugin;

fn main() throws {
    let file = plugin::file("greeter");
    let p = try plugin::load(f"../greeter/target/debug/{file}");
    let greet = try p.get::<fn(str) -> str>("greeter::greet");
    println(greet("Ada"));

    let squares = try p.get::<fn(i32) -> List<i32>>("greeter::squares");
    println(squares(5));

    // 外掛建立的閉包會保留自己的狀態。
    let next = try p.get::<fn() -> fn() -> i32>("greeter::counter")();
    println(f"{next()} {next()} {next()}");
}
```

```text
hello, Ada
[0, 1, 4, 9, 16]
1 2 3
```

程式這一端不需要任何宣告：使用 `std::plugin` 的程式，建置出來就能承載外掛。

也可以不給路徑，讓 `nabi` 替你建置外掛：設定了 `plugin: true` 的相依套件會被建置成外掛，放在你的程式旁邊（相同的建置設定檔、相同的目標），而 `plugin::beside(name)` 會在那裡找到它。

```nabion
Package {
    name: "app",
    deps: { "greeter": Dep::path("../greeter", plugin: true) },
}
```

```nabi norun
let p = try plugin::load(plugin::beside("greeter"));
```

## 共用型別

程式和外掛可以交換雙方都認得的型別的值：把這個型別放進兩者都相依的套件裡。

```nabion
// game/nabi.nabion: 程式
Package { name: "game", deps: { "engine": Dep::path("../engine") } }

// enemies/nabi.nabion: 外掛
Package { name: "enemies", lib: { kind: "plugin" }, deps: { "engine": Dep::path("../engine") } }
```

這樣 `engine::Entity` 在兩邊就是同一個型別，`p.get::<fn(engine::World) -> List<engine::Entity>>("enemies::spawn")` 會原封不動地來回傳遞這些值。外掛從程式取得 `engine`，所以 `engine` 的靜態變數只有一份。只有外掛用到的相依套件可以跟著外掛一起：`Dep::version("^1.2", bundle: true)` 會把它連結進外掛。

`any` 與 `any Trait` 值也能跨越邊界：外掛可以回傳一個裝著它自己型別的 `any Display`，程式中的 `{v}` 會用外掛的 `fmt` 印出它。反射也看得到外掛的型別：`core::reflect::info_of(v)` 能描述外掛建立的值，`p.types()` 則列出外掛所描述的型別。

## 載入失敗時

每個錯誤都會告訴你該怎麼做。檔案不存在、檔案不是外掛、外掛由另一個工具鏈建置、程式缺少某個套件、函式不存在或型別不同，這些都會以 `PluginError` 回傳：

```nabi norun
let p = try plugin::load("plugins/missing.so") catch e {
    println(e);
    return;
};
```

```text
no plugin at plugins/missing.so; build its package with `nabi build` and give the file it writes under target/<profile>/ (`plugin::file(name)` gives the file name)
```

外掛和程式必須來自同一個工具鏈：`nabi toolchain` 改變之後，請重新建置你的外掛。不存在能在不同工具鏈版本之間保持穩定的外掛 ABI。

## 卸載

`p.unload()` 會立刻讓外掛無法再被取用，但只要外掛的任何東西還活著，它就會留在記憶體中：它的某個型別的物件、它建立的閉包、仍在執行中的對它的呼叫。回傳的結果會說明是哪一種情況：

```nabi norun
match p.unload() {
    plugin::Unload::Done => println("gone"),
    plugin::Unload::Pending(why) => println(f"later: {why}"),
    plugin::Unload::Kept(why) => println(f"stays: {why}"),
}
```

如果外掛曾把自己程式碼中的回呼交給 C 函式庫，它就會永久保留，因為 C 仍可能呼叫它。

## 外掛可用的平台

外掛可以在 macOS（arm64）、Linux（glibc，x86-64 與 arm64）和 Windows（x64 與 arm64）上載入。在 WebAssembly 和靜態 musl 目標上，程式無法在執行時載入程式碼，所以為這些目標建置外掛，或建置使用 `std::plugin` 的程式，都會以錯誤停止，並告訴你改為建置單一程式：把外掛的套件改成一般的相依套件。

## 小結

- `lib: { kind: "plugin" }` 把函式庫套件建置成外掛；在相依套件上寫 `plugin: true`，會把它建置在你的程式旁邊。
- `plugin::load(path)` 載入外掛，`p.get::<F>("pkg::name")` 取得型別恰好為 `F` 的 `pub fn`。
- 值原封不動地跨越邊界：同一個執行環境、同一個堆積。透過兩者都相依的套件共用型別。
- `unload` 會等到外掛的任何東西都不再存活。

## 延伸閱讀

- 規格：[外掛](../spec/refs-lang/009-tools/912-plugins.html)、[模組描述子 ABI](../spec/refs-lang/008-abi/812-module-abi.html)、[`std::plugin`](../spec/refs-standard-libraries/300-std/311-plugin.html)
- API：[std::plugin](../../std/std/plugin.html)
- 上一章：[22. 用 Nabi 做科學計算](22-science.html)
