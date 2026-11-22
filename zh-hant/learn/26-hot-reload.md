# 26. 熱重載

`nabi dev` 會執行你的程式，並在它執行時套用你的修改。儲存檔案後，過一會兒，執行中的程式就會改用新的程式碼，而它累積下來的一切都還在：你正在玩的關卡、你正在調整的視窗、你載入的資料。這是為一邊開發一邊觀察的程式設計的，例如遊戲、模擬程式，以及有使用者介面的工具。

本章會在 `nabi dev` 底下執行一個小型的遊戲迴圈，修改一個函式，再修改一個型別。

## 開始一個工作階段

拿一個 `main` 中有迴圈的套件來看：

```nabi norun
use core::time::Duration;

struct Player {
    name: str;
    hp: i32;
}

fn draw(p: Player) {
    println(f"{p.name} has {p.hp} hp");
}

fn main() {
    let p = Player { name: "ann", hp: 10 };
    loop {
        draw(p);
        std::reload::point();
        std::thread::sleep(Duration::ms(500));
    }
}
```

用 `nabi dev` 取代 `nabi run` 來執行它：

```bash
nabi dev
```

```text
    Building the host of `game`
     Running `game` (1.4s)
    Watching press `r` to restart, `q` to quit
ann has 10 hp
ann has 10 hp
```

`std::reload::point()` 標出一個適合接收新程式碼的位置：在兩個影格之間，沒有做到一半的工作。在 `nabi dev` 之外它什麼都不做，所以同一個程式不需修改就能在 `nabi run` 底下執行。

## 修改函式

編輯 `draw` 並儲存：

```nabi norun
fn draw(p: Player) {
    println(f"{p.name}: {p.hp} hp");
}
```

```text
  Rebuilding `game`
     Applied game v2: 1 function
ann: 10 hp
ann: 10 hp
```

只有 `game` 被重新建置，而且其中只有變更的部分被重新編譯。`main` 中的迴圈持續執行；它下一次呼叫 `draw` 時就進入了新的主體。

無法編譯的儲存不會改變程式中的任何東西：

```text
error[E0034]: no field `hpp` on `Player`
 --> src/main.nabi:9:28
        Kept the running code of `game`: fix the error and save
```

修正錯誤、再次儲存，修改就會套用進去。

## 修改型別

為 `Player` 加上一個新欄位。程式執行期間，它擁有的每個 `Player` 都必須為這個欄位取得一個值，所以新欄位需要預設值：

```nabi norun
struct Player {
    name: str;
    hp: i32;
    mana: i32 = 5;
}

fn draw(p: Player) {
    println(f"{p.name}: {p.hp} hp, {p.mana} mana");
}
```

```text
     Applied game v3: 1 function, 1 type (Player: +mana), 1 object migrated
ann: 10 hp, 5 mana
```

`main` 在修改之前建立的 `Player` 現在有了 `mana: 5`。如果沒有預設值，建置會停下來，並告訴你該寫什麼：

```text
error: new field `mana` needs a default while the program runs: write `mana: i32 = 0;` (or restart with `r`)
```

移除欄位、調整欄位順序以及新增列舉變體都可以套用。重新命名時，只要說明它原本叫什麼，值就會保留：

```nabi norun
struct Player {
    name: str;
    #[renamed_from("hp")]
    health: i32;
}
```

改變欄位的型別則無法沿用舊值；錯誤訊息會告訴你為新欄位取一個自己的名稱並給它預設值，或是重新啟動。

## 狀態放在哪裡

只有在沒有任何執行中的函式於區域變數中持有被修改型別的值時，物件配置的變更才能套用。`main` 的迴圈從頭到尾都持有 `p`，所以當 `Player` 改變時，這個變更會等待，並說明原因：

```text
     Waiting game v3: `main` (src/main.nabi:13) holds a `Player` across `reload::point()`; keep it in a `reload::State`: `let p = reload::State::new(Player { .. });`
```

把存活時間長的狀態放在 `std::reload::State` 中。它是一個重載機制知道如何更新的容器（box）；在 `nabi dev` 之外，它就是一個普通的容器：

```nabi norun
use core::time::Duration;
use std::reload;

fn main() {
    let p = reload::State::new(Player { name: "ann", hp: 10, mana: 5 });
    loop {
        draw(p.get());
        reload::point();
        std::thread::sleep(Duration::ms(500));
    }
}
```

還有兩件事要知道：

- `main` 自己的迴圈在整個工作階段中都是舊程式碼。把每個影格的工作放在 `main` 呼叫的函式中，就像這裡的 `draw`。
- 閉包會保留它建立時的主體。由新程式碼建立的新閉包則使用新的主體。

若要在每次重載後執行程式碼，例如重建快取，可以註冊一個 hook。hook 依名稱保存，所以從新程式碼再次註冊會取代舊的：

```nabi norun
reload::on_reload("cache", |r| println(f"reloaded v{r.version}"));
```

## 重新啟動

有些變更無法就地套用。當 `nabi.nabion` 或 `nabi.lock` 改變（新的相依套件、新的版本、不同的功能特性），或工具鏈改變時，`nabi dev` 會重新啟動程式，並說明原因：

```text
  Restarting nabi.nabion changed
```

按 `r` 可以手動重新啟動，例如在程式拒絕某個變更之後；按 `q` 則結束。`nabi dev` 執行期間，程式的標準輸入是空的，因為按鍵歸 `nabi dev` 使用。

## 哪些會重載

你自己的套件會重載：你執行的套件及其 `path` 相依套件，包括工作區成員。登錄庫和 git 套件屬於主機（host）的一部分，而主機在每個工作階段只建置一次，所以修改它們會導致重新啟動。熱重載只建置偵錯版程式碼：`nabi dev --release` 是錯誤。用 `nabi build` 或 `nabi run` 建置的程式完全沒有這套機制，也不必為它付出任何代價。

## 小結

- `nabi dev` 執行程式，並在執行期間套用每次儲存；編譯錯誤時保留舊的程式碼。
- 每個影格呼叫一次 `std::reload::point()`。
- 新欄位需要預設值；`#[renamed_from]` 會保留重新命名欄位的值。
- 把存活時間長的狀態放在 `reload::State` 中，把每個影格的工作放在函式中。
- `r` 重新啟動，`q` 結束；清單、鎖定檔和工具鏈的變更會自動重新啟動。

## 延伸閱讀

- 規格：[熱重載](../spec/refs-lang/009-tools/914-hot-reload.html)、[`nabi dev`](../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-dev-run-with-hot-reload)、[`std::reload`](../spec/refs-standard-libraries/300-std/313-reload.html)
- 自行載入程式碼：[23. 外掛](23-plugins.html)
- 工具：[16. 工具](16-tools.html)
