# 18. 功能特性與發布

[第 11 章](11-modules-packages.html)建立了套件並新增了相依套件。本章更進一步：介紹讓函式庫使用者開啟或關閉部分功能的**功能特性**（feature）、Nabi 專案維護的套件，以及如何把你自己的函式庫發布到套件登錄庫 [packages.nabi-lang.org](https://packages.nabi-lang.org)。

## 定義功能特性

功能特性是 `nabi.nabion` 中 `features` 字典裡的一個名稱，對應到它要開啟的項目串列。在程式碼中，已啟用的功能特性就是條件 `feature = "<name>"`（[第 17 章](17-conditional-compilation.html)）。這裡的 `geom` 函式庫有一個預設開啟的 `circle` 功能特性，以及一個會引入選用相依套件的 `pretty` 功能特性。

```nabion
Package {
    name: "geom",
    version: "0.1.0",
    deps: {
        "fancy": Dep::path("../fancy", optional: true),
    },
    features: {
        "default": ["circle"],
        "circle": [],
        "pretty": ["dep:fancy"],
    },
}
```

```nabi norun
// geom/src/lib.nabi
pub fn shapes() -> List<str> {
    let out: List<str> = ["square"];
    #[cfg(feature = "circle")]
    out.push("circle");
    out
}

#[cfg(feature = "pretty")]
pub fn show(s: str) -> str { fancy::frame(s) }

#[cfg(not(feature = "pretty"))]
pub fn show(s: str) -> str { s }
```

功能特性串列中的每個項目是以下其中之一：

| 項目        | 開啟什麼                                                            |
|-------------|---------------------------------------------------------------------|
| `"circle"`  | 本套件的另一個功能特性                                              |
| `"dep:fancy"` | 選用相依套件 `fancy`                                              |
| `"http/tls"` | 相依套件 `http` 的功能特性 `tls`（若 `http` 是選用的，也一併開啟 `http` 本身） |
| `"http?/tls"` | `http` 的功能特性 `tls`，但只在其他東西已經開啟 `http` 時 |

- `default` 是一般的功能特性，除非使用者關閉它，否則都是開啟的。
- 選用相依套件（`optional: true`）只有在某個已啟用的功能特性以 `dep:` 或 `<dep>/<f>` 指名它時，才會納入建置。它不會自動得到同名的功能特性，而沒有任何功能特性啟用的選用相依套件是錯誤。`nabi add fancy --path ../fancy --optional` 會同時寫入相依套件和 `"fancy": ["dep:fancy"]` 功能特性。
- 指名不存在的功能特性或相依套件、對非選用的相依套件使用 `dep:`，以及環狀相依，都是錯誤，並會回報 `nabi.nabion` 中的行與欄。

功能特性的名稱是說明它啟用什麼的小寫單字（`http1`、`tls`、`pretty`），而不是 `with-x` 或 `no-x`：功能特性永遠只會增加東西。

## 啟用功能特性

正在建置的套件從它的 `default` 功能特性開始。`--features` 可以再加上更多功能特性，可以是自己的（`fancy`），也可以是直接相依套件的（`geom/pretty`）；`--no-default-features` 和 `--all-features` 的作用正如其名。它們可以搭配 `run`、`build`、`test`、`check` 和其他建置指令使用。

```text
$ nabi run
square
circle
$ nabi run --features geom/pretty
*** square ***
*** circle ***
$ nabi run --no-default-features
square
circle
```

最後一個仍然印出 `circle`：`--no-default-features` 關閉的是正在建置的套件（`app`）的預設功能特性，而不是 `geom` 的。相依套件的功能特性，是在你宣告相依的地方選擇的。

```nabion
Package {
    name: "app",
    version: "0.1.0",
    deps: {
        "geom": Dep::path("../geom", default_features: false, features: ["pretty"]),
    },
    features: {
        "fancy": ["geom/pretty"],
    },
}
```

功能特性會被**統一**（unified）：每個套件在整個建置中只有一組功能特性，也就是所有相依於它的套件所要求的功能特性的聯集。所以如果兩個函式庫都使用 `geom`，一個要 `pretty`，另一個不要，兩者都會得到 `pretty`。這就是功能特性只能增加 API、絕不能改變既有程式碼行為的原因。`nabi deps` 會以 `--extern-feature` 行顯示結果。

鎖定檔會固定相依關係圖中的每個套件，包括選用的套件，所以切換功能特性絕不會改變 `nabi.lock`。

## Nabi 套件

Nabi 專案在標準函式庫之外，把一組函式庫發布為獨立的套件：資料格式（`json`、`toml`、`csv`）、`http`、`markdown`、`regex`、`spatial`、`nabi-syntax`，以及[第 22 章](22-science.html)的科學套件。每一個都是純 Nabi 寫成的。它們在登錄庫上標示為 **Official**，登錄庫也[列出了全部](https://packages.nabi-lang.org/packages?q=official:yes)，每個套件的文件都從使用方式開始說明。

`json` 和 `toml` 依循 std 的 [`nabion`](15-nabion.html) 模組的形式：`parse` / `to_str` / `from_str`，加上 `to_value` / `from_value`，以及用於你自己型別的 derive。

```nabi norun
#[derive(json::Json)]
struct Server { host: str; port: u16 = 80; }

let s: Server = try json::from_str("{\"host\": \"localhost\"}");
println(json::to_str(s));      // {"host":"localhost","port":80}
```

套件的 derive 要加上套件名稱寫成 `#[derive(json::Json)]`，或在 `use json::Json;` 之後直接寫出名稱。derive 展開後的內容是套件自己的程式碼：它可以呼叫套件的內部輔助函式，而你的程式碼不行。

它們的新增方式和其他套件一樣。沒有指定需求時，`nabi add` 會寫入 `^` 和最新版本：

```bash
nabi add json
nabi add http --no-default-features --features http1,tls
```

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "json": Dep::version("^0.1.0"),
    "http": Dep::version("^0.1.0", default_features: false, features: ["http1", "tls"]),
} }
```

在指令碼中，把檔案放在最後：`nabi add json app.nabi` 會把相依套件寫進指令碼的 `//!` 區塊（[第 11 章](11-modules-packages.html#single-file-scripts-and-environments)）。

有了 `http`，一個 HTTPS 請求只要幾行。這個套件會開啟 TCP 連線，以作業系統的信任存放區執行 [`std::net::tls`](19-crypto-and-tls.html) 的 TLS 1.3 交握，透過 ALPN 提供 HTTP/2，並在必要時退回 HTTP/1.1：

```nabi norun
let r = try await http::get("https://example.com/");
println(f"{r.status} {r.version:?} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

## 發布套件

發布會在登錄庫索引中新增一行，指向你 git 儲存庫的某個提交；登錄庫會保存該提交的封存檔，所以這個版本會比你的儲存庫存在得更久。簡單來說：

```bash
nabi login                # a token from packages.nabi-lang.org/me
nabi publish --dry-run    # print the index entry; nothing leaves your machine
nabi publish
nabi yank textwrap@0.1.0 --message "wraps CJK text wrong; use 0.1.1"
```

發布在本指南中有獨立的一節，[發布套件](packages/index.html)：說明套件需要什麼、[`nabi login` 與 `nabi publish`](packages/01-publishing.html)、從 GitHub Actions 進行的[受信任發布](packages/02-trusted-publishing.html)，以及如何用 `#[deprecated]`、`#[non_exhaustive]` 和 `#[sealed]` [保持 API 穩定](packages/03-stable-api.html)。

> [!IMPORTANT]
> 目前只有登錄庫的維護者可以發布。任何人都可以登入並使用套件，但 `nabi login` 會拒絕還不能發布的帳戶。

## 小結

- `nabi.nabion` 中的 `features` 把名稱對應到項目：其他功能特性、`dep:<d>`、`<d>/<f>`、`<d>?/<f>`。`default` 除非被關閉，否則都是開啟的。
- 選用相依套件（`optional: true`）只有在某個功能特性啟用它時才會建置。
- 用 `--features`、`--no-default-features`、`--all-features` 選擇功能特性，對個別相依套件則用 `features:` / `default_features: false`。功能特性在整個建置中會被統一，所以只能增加東西。
- 程式碼用 `#[cfg(feature = "..")]` 和 `cfg!(feature = "..")` 檢查它們。
- Nabi 專案的套件在登錄庫上標示為 Official。
- 用 `nabi login`、`nabi publish --dry-run` 和 `nabi publish` 發布，用 `nabi yank` 撤回；詳細內容請見[發布套件](packages/index.html)。目前只開放維護者發布。

## 延伸閱讀

- 規格：[9.10.4c 功能特性](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104c-features)、[9.10.4a 登錄庫套件](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104a-registry-packages)、[`nabi search` / `publish` / `yank` / `login`](../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-search--publish--yank--login--logout)、[`nabi.nabion`](../spec/refs-lang/009-tools/910-nabi-frontend.html#9103-nabinabion--nabilock)
- [發布套件](packages/index.html)，以及套件登錄庫 [packages.nabi-lang.org](https://packages.nabi-lang.org)
- 下一章：[19. 密碼學與 TLS](19-crypto-and-tls.html)
