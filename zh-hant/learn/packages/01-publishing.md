# 1. 發布一個套件

發布就是在登錄庫索引中加上一行，指向你 git 儲存庫中的一個提交（[登錄庫如何運作](index.html#登錄庫如何運作)）。本頁從清單一路說明到發布一個版本，以及如何撤回版本。

## 套件需要什麼

- 儲存庫根目錄下的 `nabi.nabion`，其中有 `name`、`version` 和 `nabi_version`。
- 一個 `src/lib.nabi`（函式庫）、一個 `src/main.nabi`（給 `nabi install` 安裝的程式），或兩者都有，以及一份 readme。
- 一個乾淨的 git 工作樹，有遠端 `origin`，其 url 是公開主機上的 `https://` 網址，而且提交已推送到它的某個分支或標籤（只存在於分叉中的提交不算）。
- 所有一般相依套件都來自登錄庫。git 相依套件會被拒絕；路徑相依套件也會被拒絕，除非它同時寫明發布時用來取代它的版本：`Dep::path("../geom", version: "^0.1")`（先發布 `geom`）。沒有 `version:` 的路徑與 git `dev_deps` 會被移除。
- 清單中沒有 `publish: false`。

把 `nabi_version` 設為你建置過這個套件的最舊 nabi：`nabi publish` 需要它，而解析時會給每位使用者其工具鏈能建置的最新版本（`"0.1.3"` 接受 nabi 0.1.3 及之後的 0.1.x，請見[第 1 章](../01-getting-started.html#管理工具鏈)）。每個套件頁面會把它顯示為「Requires」。`nabi new` 會寫入建立這個套件的版本。

請填寫 `description`、`license` 和 `repository`：它們會出現在 `nabi search` 和網站上。套件頁面顯示的其他欄位都是選用的，與 Cargo 相同：

```nabion
Package {
    name: "textwrap",
    version: "0.1.0",
    nabi_version: "0.1.0",
    description: "Word wrapping for terminals",
    license: "MIT OR Apache-2.0",
    repository: "https://github.com/you/textwrap",
    authors: ["You <you@example.com>"],
    homepage: "https://example.com/textwrap",
    documentation: "https://example.com/textwrap/docs",
    keywords: ["text", "terminal"],
    categories: ["text-processing"],
    exclude: ["bench/**"],
    deps: {},
}
```

`keywords` 和 `categories` 各最多 5 項；`include` / `exclude` 選擇套件包含哪些檔案；另外還有 `license_file`、`readme`、`links`、`default_run`，以及一個自由格式的 `metadata` 表。`nabi` 不認得的欄位只會產生警告而不是錯誤，並且照樣送到登錄庫。

## `nabi login`

用 GitHub 登入 [packages.nabi-lang.org/me](https://packages.nabi-lang.org/me)，在那裡建立帳戶權杖，再把它交給 `nabi`：

```text
$ nabi login
a token from https://packages.nabi-lang.org/me (not shown; Ctrl-C to cancel):
   Logged in as your-name at https://packages.nabi-lang.org (token in ~/.nabi/tokens.nabion)
```

不帶引數時，權杖會從標準輸入讀取，不會回顯。`nabi login` 會向登錄庫詢問權杖屬於誰，登錄庫拒絕的權杖會被拒收，什麼都不儲存。有效的權杖會存在 `~/.nabi/tokens.nabion`（只有你能讀取）中，依建立它的索引分開存放，並且只會送給那個索引；`nabi login --registry <url>` 會為另一個索引保存權杖。`nabi logout` 會移除它，`nabi logout --all` 會移除所有索引的權杖。在 CI 中則改為設定 `NABI_REGISTRY_TOKEN`，它只會送給 packages.nabi-lang.org。從 GitHub Actions 發布時，[受信任發布](02-trusted-publishing.html)完全不需要儲存權杖。

## `nabi publish`

`nabi publish --dry-run` 會印出它將加入的索引項目，不會把任何東西送出你的電腦。`nabi publish` 則直接發布，與 `cargo publish` 相同。

```text
$ nabi publish --dry-run
Package { name: "textwrap", version: "0.1.0", deps: [], source: Dep::git("https://github.com/you/textwrap.git", rev: "8e2afb20.."), tree: "823a3592..", yanked: false, nabi_version: "0.1.0" }
note: dry run; `nabi publish` sends it to the registry API of https://github.com/nabi-lang/packages-index (needs `nabi login`), which adds it as te/xt/textwrap
```

項目只包含解析所需的內容：HEAD 提交與它的樹雜湊、相依套件，以及 `nabi_version`。請先推送那個提交。描述、授權條款等其他中繼資料會隨項目一起送到登錄庫，登錄庫會連同每個版本的發布者一起顯示它們。`nabi publish` 會用你的權杖把兩者一起送出（`Published textwrap 0.1.0 to https://packages.nabi-lang.org`），索引收錄之後，這個版本就上線了。同一個版本不能發布兩次。

含有執行檔的套件在清單與搜尋結果中會標示為 **bin**，它的頁面會顯示 `nabi install <name>`（同時也有函式庫時則是 `nabi install --bin <name>`）。

## 名稱與擁有者

名稱最多 64 個字元。與現有套件只差在大小寫或 `-` / `_` 的名稱，會被視為已被使用。標準函式庫的層級 `std`、`core` 和 `bare` 是索引中由 Nabi 專案擁有的項目：它們隨工具鏈一起提供，不會有任何東西以這些名稱發布。工具鏈的名稱（`nabi`、`nabic` 等）是保留的；與索引中某個名稱只差一個字母，或是把外觀相似的字元歸併後相同（`js0n` 之於 `json`）的新名稱會被拒絕；若仍想申請這樣的名稱，[使用政策](https://packages.nabi-lang.org/policy)說明了做法。

發布套件第一個版本的人就是它的擁有者。擁有者可以在[帳戶頁面](https://packages.nabi-lang.org/me)邀請其他帳戶共同擁有它；受邀的帳戶接受之後就成為擁有者。

## API 文件

登錄庫會從每個版本所在提交的原始碼產生它的 API 文件，只讀取原始碼，不建置也不執行；文件網站負責呈現，就像 docs.rs 之於 crates.io：`docs.nabi-lang.org/packages/textwrap/0.1.0`，最新版本則是 `/packages/textwrap/latest`（實際的例子請見 [json 的文件](https://docs.nabi-lang.org/packages/json/latest)）。登錄庫上的套件頁面會在 Docs 分頁連結到那裡。頁面的編排與 `nabi doc` 相同：根模組在那個網址，其他每個公開模組各有自己的頁面（`/packages/textwrap/0.1.0/<module>`），錨點也相同（`#fn.wrap`）。讀者最先看到的是根模組的 `//!` 文字，所以套件的介紹與教學就寫在那裡；其中的程式碼區塊是 `nabi test` 會執行的文件測試，以 `# ` 開頭的行是頁面上省略的準備程式碼。模組的文件是它的檔案開頭的 `//!` 行；一般的 `//` 註解不會顯示。用 `pub use` 從私有模組重新匯出的項目，會顯示成宣告在重新匯出的位置。只有執行檔的套件沒有 API，因此也沒有文件。如果文件無法產生，版本照樣發布：`nabi publish` 會以警告印出原因，文件頁面也會顯示它。

## `nabi yank`

已發布的版本永遠不會被刪除。有嚴重錯誤的版本可以撤回：新的解析會跳過它，但 `nabi.lock` 已經固定它的專案仍能照常建置，所以撤回不會破壞任何人的建置。`--undo` 可以取消撤回。

```bash
nabi yank textwrap@0.1.0 --message "wraps CJK text wrong; use 0.1.1"
nabi yank textwrap@0.1.0 --undo
```

`--message` 用來說明原因，登錄庫會在套件頁面上顯示它。

## 延伸閱讀

- 規格：[9.10.4a 登錄庫套件](../../spec/refs-lang/009-tools/910-nabi-frontend.html#9104a-登錄庫套件)、[`nabi search` / `publish` / `yank` / `login`](../../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-search--publish--yank--login--logout)、[`nabi.nabion`](../../spec/refs-lang/009-tools/910-nabi-frontend.html#9103-nabinabion--nabilock)
- 下一頁：[2. 受信任發布](02-trusted-publishing.html)
