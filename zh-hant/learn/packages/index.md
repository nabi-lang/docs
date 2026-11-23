# 發布套件

這一部分說明如何把你自己的函式庫或程式放上套件登錄庫 [packages.nabi-lang.org](https://packages.nabi-lang.org)，並持續維護。如何使用套件請見[第 11 章](../11-modules-packages.html#套件登錄庫)，功能特性請見[第 18 章](../18-features-and-publishing.html)。

> [!IMPORTANT]
> 目前只有登錄庫的維護者可以發布。任何人都可以登入並使用套件，但 `nabi login` 會拒絕還不能發布的帳戶。

## 登錄庫如何運作

登錄庫的資料是一個 git 儲存庫，也就是[索引](https://github.com/nabi-lang/packages-index)。套件的每個版本是其中的一行，記錄套件本身 git 儲存庫中的一個提交，以及該提交的樹雜湊。你不必上傳原始碼：版本發布時，登錄庫會取得那個提交、保存它的封存檔，並把封存檔的 SHA-256 寫進這一行。使用者下載封存檔後會對照索引檢查，所以即使儲存庫不在了，版本仍然存在。這些行只由登錄庫寫入，索引不接受 pull request。行的格式請見[索引的 README](https://github.com/nabi-lang/packages-index#readme)。

從 nabi-lang 組織發布的套件會標示為 **Official**，它們隨工具鏈一起提供（[第 11 章](../11-modules-packages.html#套件登錄庫)）。

## 各頁內容

| 頁面 | 涵蓋內容 |
|------|----------------|
| [發布一個套件](01-publishing.html) | 套件需要什麼、`nabi login`、`nabi publish`、名稱與擁有者、登錄庫產生的文件、`nabi yank` |
| [受信任發布](02-trusted-publishing.html) | 不儲存權杖，從 GitHub Actions 發布 |
| [保持 API 穩定](03-stable-api.html) | `#[deprecated]`、`#[non_exhaustive]` 和 `#[sealed]` |
| [安全性公告](04-security-advisories.html) | `nabi audit`、閱讀公告、回報漏洞 |

登錄庫本身的規則放在登錄庫上：[使用政策](https://packages.nabi-lang.org/policy)、[使用條款](https://packages.nabi-lang.org/terms)與[隱私權聲明](https://packages.nabi-lang.org/privacy)。
