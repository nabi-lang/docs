# 11. 模組與套件

程式變大之後，你需要把程式碼拆開，也需要使用別人寫的函式庫。在 Nabi 中，劃分程式碼的單位是**模組**（module），建置與發布的單位是**套件**（package）。

## `mod` 與路徑

`mod name { .. }` 會就地定義一個模組。模組內的項目以 `module::item` 路徑參照，模組之中也可以再包含模組。

```nabi
mod geom {
    pub struct Point { pub x: f64; pub y: f64; }

    pub fn origin() -> Point { Point { x: 0.0, y: 0.0 } }

    pub mod dist {
        use sup::Point;

        pub fn manhattan(a: Point, b: Point) -> f64 {
            abs(a.x - b.x) + abs(a.y - b.y)
        }

        fn abs(v: f64) -> f64 { if v < 0.0 { -v } else { v } }
    }
}

let p = geom::Point { x: 3.0, y: -4.0 };
println(f"{geom::dist::manhattan(geom::origin(), p)}");
```

```output
7
```

只有特定的字可以作為路徑的開頭。

| 開頭     | 意義                                                         |
|----------|--------------------------------------------------------------|
| `pkg::`  | 從目前套件的根開始                                           |
| `sup::`  | 從上層模組開始（要再往上，寫 `sup::sup::`）                   |
| `std::`、`core::`、`bare::` | 標準函式庫的各層級 |
| `::std::` | 即使本地有 `std`，也指標準函式庫 |
| 一個名稱 | 目前模組的項目或子模組，或用 `use` 引入的名稱 |

名稱查找不會自動搜尋上層模組。因此在 `dist` 裡使用 `Point` 時，必須以 `use sup::Point;` 明確引入。也正因如此，你在一個檔案裡永遠看得出每個名稱來自哪裡。

`std`、`core` 和 `bare` 都是一般的名稱。你的模組或相依套件也可以叫 `std`；在擁有它的模組中，`std::` 指的是你的那一個，而 `::std::` 仍然指標準函式庫：

```nabi
mod std { pub fn hello() -> str { "mine" } }

fn main() {
    println(std::hello());                       // mine
    println(::std::path::Path::empty().is_relative());  // 函式庫的
}
```

## 用 `use` 匯入

不必每次都寫長長的路徑，可以用 `use` 把名稱引入。用 `{..}` 可以一次匯入多個，用 `as` 可以改名，用 `*` 可以匯入一個模組的所有公開項目。

```nabi
use report::show;
use shapes::*;

mod shapes {
    pub fn square(x: i32) -> i32 { x * x }
    pub fn cube(x: i32) -> i32 { x * x * x }
    pub const UNIT: i32 = 1;
}

mod report {
    use pkg::shapes::{square, cube as pow3};

    pub fn show(x: i32) {
        println(f"{x}: {square(x)} {pow3(x)}");
    }
}

show(3);
println(f"{cube(UNIT + 1)}");
```

```output
3: 9 27
8
```

以名稱匯入的名稱，不能在同一個模組裡再定義一次；否則定義會不聲不響地遮蔽匯入。萬用匯入（glob，`use shapes::*;`）則會讓位給模組自己的項目。

```nabi norun
use shapes::square;
fn square(x: f64) -> f64 { x * x }
```

```text check=checks/11-import-clash.nabi
error[E0032]: `square` is both imported from `shapes::square` and defined in this module
  --> main.nabi:1:5
   |
 1 | use shapes::square;
   |     ^^^^^^ imported here
 2 | fn square(x: f64) -> f64 { x * x }
   | -- defined here, which would hide the import
   |
   = help: import it under another name, `use shapes::square as shapes_square;`, or remove this `use` if the module does not need it
```

到目前為止不用 `use` 就能使用的 `List`、`Dict`、`println` 等名稱，來自*預載模組*（prelude）：每個檔案都以隱含的 `use core::prelude::*;` 開頭。預載模組是一份固定且簡短的清單：常用型別（`List`、`Dict`、`Result`、`Duration`、`Mutex` 等）、寫在 `impl` 標頭和約束中的 trait（`Clone`、`Hash`、`Display`、`Iterator`、`From` 等）、輸出與 panic 函式，以及 `core` 的頂層模組（`task`、`par`、`ops`、`num`、`sync`、`iter` 等）。`core` 中的其他一切都要透過這些模組之一取用，或另外匯入：

```nabi norun
task::sleep(Duration::ms(1));          // the module `task` is in the prelude
impl ops::Add for V { .. }             // operator traits live in `ops`
use core::sync::Frozen;           // or import the name
```

你自己的名稱永遠優先於預載模組的名稱。列舉的變體不在預載模組中；在預期型別能決定是哪個列舉的地方，變體直接寫出即可（[第 6 章](06-structs-enums-traits.html)）。當某個名稱不在作用域內時，錯誤訊息會列出要寫的路徑以及匯入它的 `use`。`std` 中的檔案、網路、執行緒等模組要明確匯入，例如 `use std::fs;`。

trait 也是同樣的道理：只有在 trait 位於作用域內時，才找得到它的方法，所以來自其他模組或套件的 trait，需要先 `use`（`use zoo::Animal;`），`x.name()` 才能解析。標準函式庫（`core`、`std`）中 trait 的方法則在任何地方都能用，不需要 `use`。

巨集、derive 和 meta 屬性屬於定義它們的套件。你自己的套件和標準函式庫中的，直接寫出名稱即可（`log!(..)`、`#[derive(Inspect)]`）；其他套件的則要加上套件名稱，寫成 `json::log!(..)`、`#[derive(json::Json)]`，或以 `use json::log!;` 匯入一次。直接寫出其他套件巨集的名稱是錯誤，錯誤的說明會列出這兩種寫法。

## 可見性

預設情況下，項目只在自己的模組內可見。要讓它在外部可見，請加上可見性關鍵字。

| 關鍵字     | 可見範圍                                |
|------------|-----------------------------------------|
| （無）     | 同一個模組                              |
| `pub(sub)` | 同一個模組及其子模組                    |
| `pub(pkg)` | 同一個套件內的任何地方                  |
| `pub`      | 其他套件也可見                          |

結構的欄位也必須個別設為公開。因此上面的 `Point` 在每個欄位前都加了 `pub`，例如 `pub x: f64`。如果你從外部使用私有項目，編譯器會告訴你要在哪裡加上 `pub`。

```nabi
mod bank {
    pub struct Account { pub owner: str; balance: i64; }

    pub fn open(owner: str) -> Account { Account { owner: owner, balance: 0 } }

    pub fn deposit(a: Account, amount: i64) -> Account {
        Account { owner: a.owner, balance: a.balance + check(amount) }
    }

    pub fn balance(a: Account) -> i64 { a.balance }

    fn check(amount: i64) -> i64 { if amount < 0 { 0 } else { amount } }
}

let a = bank::deposit(bank::open("kim"), 500);
println(f"{a.owner}: {bank::balance(a)}");
```

```output
kim: 500
```

`balance` 欄位和 `check` 函式在 `bank` 之外不可見，所以餘額只能透過 `bank` 的函式改變。基於同樣的理由，`bank` 之外的程式碼不能寫 `bank::Account { owner: "kim" }`：結構字面值只有在私有欄位有預設值時，才能省略該欄位。這就是由 `bank::open` 建立帳戶的原因。

標準函式庫中有少數 `pub` 項目標有 `#[internal]`：它們由工具鏈自己的套件共用，其他任何套件使用它們都是錯誤，錯誤訊息為「internal to the Nabi toolchain」。API 文件和編輯器都不會列出它們。

## 拆分成檔案

在實際的套件中，模組會拆分到不同檔案。像 `mod shapes;` 這樣宣告不帶主體的模組，會讓同一目錄下的 `shapes.nabi` 成為該模組的主體。如果它有子模組，就在 `shapes.nabi` 旁邊放一個同名的 `shapes/` 目錄（沒有 `mod.nabi` 之類的特殊檔名）。

```text
geom/
├── nabi.nabion
└── src/
    ├── lib.nabi          // mod shapes;
    ├── shapes.nabi       // shapes module body, declares mod circle;
    └── shapes/
        └── circle.nabi   // shapes::circle
```

套件的原始檔放在 `src/` 中。套件的根模組，執行檔是 `src/main.nabi`，函式庫是 `src/lib.nabi`。可執行的頂層陳述式只能出現在 `src/main.nabi` 中；其他模組只包含項目。

## 套件與 `nabi.nabion`

`nabi new` 會建立一個套件：執行檔會有 `src/main.nabi`；加上 `--lib` 時則有 `src/lib.nabi`，裡面有一個範例函式和一個 `#[test]`。清單 `nabi.nabion` 採用 [NABION](../spec/refs-standard-libraries/300-std/308-nabion.html) 格式。除了名稱和版本之外，它還記錄 `nabi_version`，也就是建立它的 nabi 版本，因此也是已知能建置它的最舊版本（這是已發布套件的使用者需要的資訊，[第 18 章](18-features-and-publishing.html)），以及 `toolchain: "0.1"`，也就是建立套件時的工具鏈系列，讓之後另一個系列的工具鏈不會出乎意料地建置它（[第 1 章](01-getting-started.html#管理工具鏈)）。標準函式庫不是相依套件：`std`、`core` 和 `bare` 隨工具鏈提供，`deps` 中若有 `"std": ..` 這一行，會出現要你移除它的錯誤。

```bash
nabi new --lib geom
nabi new app
```

```text
     Created library package (geom)
             cd geom && nabi test
     Created executable package (app)
             cd app && nabi run
```

我們在 `geom` 中建立兩個模組。如果用 `pub use` 從套件根重新匯出子模組的項目，使用者就不需要知道內部結構。

```nabi norun
// geom/src/lib.nabi
mod shapes;

pub use shapes::Circle;

pub(pkg) const PI: f64 = 3.14159;

pub fn area_sum(cs: List<Circle>) -> f64 {
    cs.map(|c| c.area()).sum()
}
```

```nabi norun
// geom/src/shapes.nabi
pub struct Circle { pub r: f64; }

impl Circle {
    pub fn area(self) -> f64 { pkg::PI * self.r * self.r }
}
```

在 `app` 中用 `nabi add` 新增相依套件時，它會寫進 `nabi.nabion` 的 `deps`。相依套件的名稱會成為它的頂層路徑（`-` 會變成 `_`）。

```bash
cd app
nabi add geom --path ../geom
```

```text
       Added geom v0.1.0
```

```nabion
Package {
    name: "app",
    version: "0.1.0",
    nabi_version: "0.1.0",
    toolchain: "0.1",
    deps: { "geom": Dep::path("../geom") },
}
```

```nabi norun
// app/src/main.nabi
use geom::{Circle, area_sum};

println(f"{area_sum([Circle { r: 1.0 }, Circle { r: 2.0 }])}");
```

```text
$ nabi run
   Compiling geom (../geom)
   Compiling app (.)
    Finished in 0.22s
     Running `target/debug/app`
15.70795
```

`PI` 是 `pub(pkg)`，所以在 `geom` 內的任何地方都能使用，但在 `app` 中看不到。git 儲存庫可以用類似 `nabi add parser --git <url> --tag v1.2` 的方式新增。對於 git 和登錄庫的相依套件，確切的提交會固定在 `nabi.lock` 中，所以也要把 `nabi.lock` 提交到版本控制。只有 `nabi update` 會把固定的版本往前推進。

## 套件登錄庫

預設的套件登錄庫是 [packages.nabi-lang.org](https://packages.nabi-lang.org)。你可以依名稱搜尋，並連同版本需求一起新增。版本需求的規則與 Cargo 相同：`^1.2` 表示 `>=1.2.0, <2.0.0`。省略需求時，會以 `^` 記錄最新版本。

```text
$ nabi search json
    Updating index https://github.com/nabi-lang/packages-index
json 0.2.0 - JSON (RFC 8259): Value tree, strict parser, streaming tokenizer, typed encode/decode, JSON Pointer
$ nabi add json@^0.2
 Downloading json 0.2.0
       Added json v0.2.0
```

```nabion
Package {
    name: "app",
    version: "0.1.0",
    nabi_version: "0.1.0",
    toolchain: "0.1",
    deps: { "json": Dep::version("^0.2") },
}
```

名稱拼錯時，會得到最接近的套件：

```text
$ nabi add jsn
    Updating index https://github.com/nabi-lang/packages-index
nabi: error: no package named `jsn` in https://github.com/nabi-lang/packages-index
help: did you mean `json`? (or search: `nabi search jsn`)
```

單純的字串是 `Dep::version` 的簡寫，而沒有運算子的需求表示 `^`，所以下面這三行要求的是同一件事。套件的 README 常用簡寫形式（`"json": "0.2"`，表示 `>=0.2.0, <0.3.0`）。

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "textwrap": "1.2",
    // "textwrap": "^1.2",
    // "textwrap": Dep::version("^1.2"),
} }
```

路徑不是版本：請寫 `Dep::path("../geom")`，不要寫 `"../geom"`。

相依套件還有兩個鍵：`package:` 用來改名，讓程式碼以 `j5` 稱呼 `json5` 套件；`target:` 則把它限制在符合條件的目標的建置中。

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "j5": Dep::version("^0.2", package: "json5"),
    "nix": Dep::version("^0.4", target: "cfg(target_family = \"unix\")"),
} }
```

套件只看得到自己的相依套件，而且使用的是它自己給的名稱。這裡 `j5::..` 是 `json5`，而 `json5::..` 什麼都不是，即使建置中的另一個套件以這個名稱相依於 `json5` 也一樣。當相依套件的名稱同時也是子模組的名稱時，`name::..` 會同時代表兩者，因此這是錯誤：相依套件請寫 `::name::..`，模組請寫 `pkg::name::..`，或用 `package:` 為相依套件改名。

單純的需求永遠指向預設登錄庫的索引。若要從鏡像站取得該索引，請把鏡像站寫在 `~/.nabi/sources.nabion` 中；`nabi.lock` 仍然記錄預設登錄庫的索引，所以不論有沒有鏡像站，鎖定檔都一樣。

```nabion
Sources { replace: { "https://github.com/nabi-lang/packages-index": "https://mirror.example.com/packages-index" } }
```

登錄庫會保存每個已發布版本的封存檔，`nabi` 會下載它（`Downloading json 0.2.0`），並以索引核對它的 SHA-256，所以即使套件的 git 儲存庫消失了，套件仍然能用。只有在登錄庫沒有封存檔時，`nabi` 才會去取得套件的 git 儲存庫。

Nabi 專案發布的套件（`json`、`toml`、`http`、`markdown` 等）是**官方套件**，使用它們可以省略 `nabi add`：只要使用其中一個，下一次 `nabi run`（或 build、check、test、REPL 儲存格）就會替你新增它，並告訴你這件事。

```text
$ cat src/main.nabi
use json::parse;
...
$ nabi run
 Downloading json 0.2.0
       Added json ^0.2 (official)
```

現在 `"json": "^0.2"` 已寫入 `nabi.nabion` 並固定在 `nabi.lock` 中，就跟你執行了 `nabi add json` 一樣。只有官方套件會以這種方式加入；其他任何名稱仍會要求你執行 `nabi add <name>`。使用 `--locked`（或 `--frozen`）時，或在 CI 中（設定了 `CI` 變數），`nabi` 絕不會修改你的清單，而是告訴你自己新增該套件。

工具鏈內附官方套件，當它內附的版本符合你的需求時，就會選用那個版本：不用索引也不用下載，所以沒有網路也能運作。`nabi update` 仍然會移到最新版本，而 `nabi component update packages` 會預先下載每個官方套件的最新版本。

對一個套件的每項需求都算數。如果 `app` 需要 `json` `>=0.1`，而 `table` 需要 `json` `^0.1`，`nabi` 會選擇兩者都接受的最新 `0.1.x`，必要時也會把中間套件退回到較舊的版本。當沒有任何版本符合時，錯誤訊息會列出誰要求什麼，以及該改什麼：

```text
nabi: error: no version of package `b` meets every requirement on it:
  `^2` (required by `a` 1.4.0)
  `^1` (required by `c` 0.3.0)
... has b 1.2.0, 2.0.1; none of them meets all of these
how those packages came in:
  `a` 1.4.0: the root package requires `a` `^1.0`
  `c` 0.3.0: the root package requires `c` `~0.3`
one build has one version of a package
help: `c` 0.4.0 gets along with the other requirements on `b`: change the root package's requirement on it, `"c": "^0.4.0"`
```

### 安全性公告：`nabi audit`

登錄庫會發布[安全性公告](https://packages.nabi-lang.org/advisories)：說明套件的哪些版本有已知漏洞，以及哪個版本修正了它。`nabi audit` 會拿 `nabi.lock` 中的套件與公告比對，並告訴你該升級到哪個版本：

```sh
nabi audit
```

沒有適用的公告時，結束碼為 0；有適用的公告時為 1（因此 CI 工作會失敗）；無法檢查時為 2。`--ignore <id>` 會在你讀過某則公告之後接受它，`--json` 會為工具輸出紀錄。詳細內容以及如何回報漏洞，請見[安全性公告](packages/04-security-advisories.html)。

函式庫可以提供讓使用者開啟或關閉的**功能特性**（feature），你也可以用 `nabi login` 和 `nabi publish` 把自己的函式庫發布為套件。功能特性在[第 18 章](18-features-and-publishing.html)中說明，該章也介紹 Nabi 專案維護的套件（`json`、`toml`、`csv`、`http`、`markdown`、`spatial`、`nabi-syntax`）；發布則請見[發布套件](packages/index.html)。

## 工作區

一起開發的套件，例如一個應用程式和它使用的函式庫，可以放在同一個**工作區**（workspace）中。最上層目錄會有一個 `nabi.nabion`，它列出成員，而不是描述一個套件：

```nabion
Workspace { members: ["app", "libs/*"] }
```

```text
shop/
|-- nabi.nabion          Workspace { members: [..] }
|-- nabi.lock            所有成員共用一個 lock
|-- target/              一個建置目錄
|-- app/                 Package { name: "app", deps: { "cart": Dep::path("../libs/cart") } }
`-- libs/cart/           Package { name: "cart", .. }
```

`libs/*` 表示直接位於 `libs/` 底下的每個套件。成員共用 `nabi.lock` 和 `target/`，所以它們都以相依套件的相同版本建置，一個函式庫也只為所有成員建置一次。成員之間透過 `Dep::path` 互相使用。

在最上層執行指令，就會對每個成員執行，也可以用 `-p` 挑選成員；在某個成員內，`--workspace` 會涵蓋所有成員：

```text
$ nabi test                 # 在最上層：所有成員
$ nabi run -p app           # 一個成員
$ cd libs/cart && nabi test --workspace
```

## 單檔指令碼與環境

對於不值得建立套件的小指令碼，它的相依套件放在檔案最開頭 `//!` 註解中的 ```` ```nabion ```` 區塊裡。把指令碼當作最後一個引數執行 `nabi add`，就會替你寫好這個區塊；路徑以相對於指令碼的方式寫入，鎖定檔則保存在快取中。指令碼使用的官方套件，會在第一次執行時加入它的區塊。

```bash
nabi add geom --path ../geom tool.nabi
```

```text
       Added geom to tool.nabi
```

````nabi norun
//! ```nabion
//! Package {
//!     deps: {
//!         "geom": Dep::path("../geom"),
//!     },
//! }
//! ```
mod helper;

use geom::{Circle, area_sum};
println(f"{area_sum([Circle { r: helper::radius() }])}");
````

```bash
nabi run tool.nabi
```

指令碼也可以有檔案模組：`mod helper;` 會載入它旁邊的 `helper.nabi`，就跟在套件的 `src/` 中一樣。模組檔案可以有自己的 `//!` 區塊（`nabi add <name> helper.nabi`），所有檔案的區塊會合併在一起。指令碼永遠看不到用 `nabi install` 安裝的函式庫；只有它們的區塊算數。當指令碼大到一個檔案放不下時，`nabi init --script tool.nabi` 會把它轉成套件：區塊變成 `nabi.nabion`，`tool.nabi` 變成 `src/main.nabi`，它的模組檔案則移到 `src/` 中。

具名環境是給 REPL 和 Jupyter 用的。用 `nabi install` 把函式庫安裝到環境中，再用 `--env` 選擇環境，或用 `nabi env use` 變更預設環境；不需要啟用步驟。還沒有任何環境時，`nabi env list` 會告訴你怎麼建立。環境是 `~/.nabi/envs` 底下的一個目錄，所以它的名稱必須在每個系統上都能當作檔名：不能有 `/`、`\` 或開頭的 `.`，最多 64 個字元，也不能是 `con` 之類的 Windows 裝置名稱。只有大小寫不同的名稱視為相同，`current` 則保留給 `nabi env use` 選定的那一個。

```text
$ nabi env create sci
     Created environment `sci` (~/.nabi/envs/sci)
$ nabi install --env sci geom --path ~/work/geom
   Installed geom into environment `sci` (for `nabi repl` / `nabi jupyter`; a script gets it with `nabi add geom <file.nabi>`)
$ nabi env use sci
$ nabi env list
* sci (active)
```

套件永遠不使用環境：在套件內，只有它自己的 `nabi.nabion` 算數。

## 安裝程式

`nabi install` 也能安裝程式，就像 `cargo install`：有 `src/main.nabi` 的套件（來自登錄庫、git 或路徑）或指令碼檔案，會以 `--release` 建置到 `~/.nabi/bin` 中。再次安裝同一個登錄庫版本或 git 提交時，什麼都不會做，並告訴你如何重新建置；新版本則會取代舊版本。路徑或指令碼檔案每次都會重新建置。

```text
$ nabi install tool
   Installed tool 0.1.0 (~/.nabi/bin/tool)
$ nabi install tool
`tool` 0.1.0 is installed already (~/.nabi/bin/tool); to rebuild it, run `nabi install tool --force`
$ nabi install tool --force
 Reinstalled tool 0.1.0 (~/.nabi/bin/tool)
$ nabi install tool@0.2
    Upgraded tool from 0.1.0 to 0.2.0 (~/.nabi/bin/tool)
```

`nabi install --list` 會列出已安裝的程式，`nabi uninstall tool` 則會移除它。

## 小結

- 用 `mod` 建立模組，並以 `pkg::` / `sup::` / `std::` 路徑和 `use` 參照名稱。
- 不用 `use` 就能使用的名稱來自預載模組，這是一份固定的清單，包含 `core` 的常用名稱和模組；其餘的透過模組取用（`task::sleep`、`ops::Add`）或另外匯入。
- 項目預設為私有。用 `pub(sub)`、`pub(pkg)` 和 `pub` 擴大範圍。
- `mod x;` 對應到 `x.nabi`（以及 `x/`）。根是 `main.nabi` 或 `lib.nabi`。
- 用 `nabi new [--lib]` 建立套件，用 `nabi add` 新增相依套件（路徑、git、登錄庫）。要提交 `nabi.lock`。
- 工作區（`Workspace { members: [..] }`）讓一起開發的套件共用一個鎖定檔和一個 `target/`；`-p <name>` 和 `--workspace` 用來挑選成員。
- 用 `nabi search` / `nabi add name@^1.2` 使用登錄庫的套件；功能特性和發布在[第 18 章](18-features-and-publishing.html)。
- 指令碼使用自己的 `//! ```nabion` 區塊（`nabi add <name> <file.nabi>`），`nabi init --script` 可以把指令碼轉成套件；`nabi env` 是給 REPL / Jupyter 用的。

## 延伸閱讀

- 規格：[模組](../spec/refs-lang/005-modules/504-modules.html)、[`use`](../spec/refs-lang/005-modules/505-use.html)、[可見性](../spec/refs-lang/005-modules/506-visibility.html)、[路徑](../spec/refs-lang/005-modules/507-paths.html)、[重新匯出](../spec/refs-lang/005-modules/508-re-export.html)
- 工具：[`nabi` 前端](../spec/refs-lang/009-tools/910-nabi-frontend.html)、[`nabi.nabion` / `nabi.lock`](../spec/refs-lang/009-tools/910-nabi-frontend.html#9103-nabinabion--nabilock)、[登錄庫套件](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104a-登錄庫套件)、[環境](../spec/refs-lang/009-tools/910-nabi-frontend.html#9105-環境)、[內嵌清單](../spec/refs-lang/009-tools/910-nabi-frontend.html#9107-單檔指令碼的行內資訊清單)
- [1. 入門](01-getting-started.html)
- 下一章：[12. 並行](12-concurrency.html)
