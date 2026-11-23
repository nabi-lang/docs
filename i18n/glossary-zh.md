# Chinese glossary

The terms every Chinese translation of Nabi follows: the site (nabi-lang/www), the docs and the spec. zh-Hans is Simplified Chinese as written in mainland China; zh-Hant is Traditional Chinese as written in Taiwan. zh-Hant is its own text with Taiwan terms (程式, 套件, 預設, 記憶體), never a character conversion of zh-Hans.

When a term is missing, take the one the Chinese editions of the Rust or Go docs use, add it here in the same change, and keep using it.

## Rules

- **Nabi** is a name and is never translated or transliterated. Lowercase `nabi` is the command and stays in code font.
- Keywords, identifiers, types, commands, flags, file names and paths stay in English and in code font: `struct`, `match`, `throws`, `pub(pkg)`, `List`, `nabi build`, `nabi.nabion`. Do not translate a word in code font, and do not add a Chinese word inside it.
- English names stay as they are: NABION, NABIHIR, HIR, MIR, AOT, JIT, GC, ABI, FFI, TLS, REPL, LLVM, MLIR, WebAssembly, WASI, SHA-256, minisign, GitHub, Discussions, issue, pull request.
- **Punctuation.** Chinese prose uses full-width punctuation: ，。、；：？！（）「」. Code, URLs, version numbers and text that is entirely Latin use half-width. A parenthesis in a Chinese sentence is full-width even when its content is Latin: `Windows 11（x64 或 arm64）`.
- **Quotation marks.** zh-Hans uses “ ” and ‘ ’; zh-Hant uses 「 」 and 『 』.
- **Spacing (盘古之白): yes.** Put one half-width space between Han characters and Latin letters, numbers or inline code: `运行 nabi build 即可`, `共 20 个官方包`, ``用 `nabi add` 添加``. No space next to full-width punctuation (`` `nabi`。 ``), between a number and `%` or `°`, or inside a word in code font. Units keep a space: `5 GiB`, `32 KiB`. A link follows the same rule as its text.
- **Bold and italics** end before full-width punctuation, not after it: `**草稿**。本文`, not `**草稿。** 本文`. Markdown does not close `**` that follows punctuation and precedes a Han character.
- **Lists of Latin items** in Chinese prose are joined with 、 (`json、toml 和 csv`); a list that is all code font may keep half-width commas inside one code span only.
- **Headings** are short noun phrases or questions, without a final 。. A heading's anchor is derived from its text, so a link to a Chinese heading uses that heading's own slug (lowercase, spaces become `-`, punctuation dropped): `## 验证下载的文件` is `#验证下载的文件`.
- **Register.** Plain, declarative technical prose, as in good Chinese docs for Rust or Go. Translate the meaning, not the English word order. No chatty fillers (呢、哦、啦) and no exclamation marks. Address the reader as 你 in both variants; legal texts (privacy notice, terms of use) use 您.
- **Tone of instructions.** Use the bare imperative or 可以…: `运行 nabi test。`, `可以用 nabi update 更新。` 请 / 請 is fine when pointing the reader somewhere (`请见`, `請見`) or asking them not to do something; do not stack courtesy words.
- The first mention of a term on a page may add the English in parentheses, `和类型（sum type）`; later mentions use the Chinese alone.
- **trait** stays as the English word trait in prose, in both variants; it takes code font only as the keyword `trait`. 特征 / 特徵 is ambiguous with feature (特性) and is not used. The same holds for a few other terms of art: panic, issue, pull request and triple.
- 并行 differs: in zh-Hans 并行 is parallel and 并发 is concurrent; in zh-Hant 平行 is parallel and 並行 is concurrent. Never convert one into the other.

## Terms

| English | zh-Hans | zh-Hant (Taiwan) | Notes |
|---|---|---|---|
| Nabi | Nabi | Nabi | the name; never translated |
| program | 程序 | 程式 | |
| programming language | 编程语言 | 程式語言 | |
| source code | 源代码 | 原始碼 | |
| code | 代码 | 程式碼 | |
| source file | 源文件 | 原始檔 | |
| file | 文件 | 檔案 | zh-Hans 文件 also means document; use 文档 for docs |
| directory | 目录 | 目錄 | |
| path (file system) | 路径 | 路徑 | |
| data | 数据 | 資料 | |
| memory | 内存 | 記憶體 | |
| documentation, docs | 文档 | 文件 | zh-Hant 文件 is docs, 檔案 is a file |
| example | 示例 | 範例 | |
| default | 默认 | 預設 | |
| settings | 设置 | 設定 | |
| configuration | 配置 | 設定 | configuration file: 配置文件 / 設定檔 |
| search | 搜索 | 搜尋 | |
| download | 下载 | 下載 | |
| install | 安装 | 安裝 | |
| install script | 安装脚本 | 安裝指令碼 | |
| script | 脚本 | 指令碼 | single-file script: 单文件脚本 / 單檔指令碼 |
| log in, sign in | 登录 | 登入 | |
| account | 账户 | 帳戶 | |
| user | 用户 | 使用者 | |
| support (v.) | 支持 | 支援 | |
| network | 网络 | 網路 | |
| server | 服务器 | 伺服器 | |
| browser | 浏览器 | 瀏覽器 | |
| site, website | 网站 | 網站 | |
| page | 页面 | 頁面 | |
| information | 信息 | 資訊 | |
| message | 消息 | 訊息 | |
| performance | 性能 | 效能 | |
| hardware | 硬件 | 硬體 | |
| software | 软件 | 軟體 | |
| device | 设备 | 裝置 | |
| command | 命令 | 指令 | |
| subcommand | 子命令 | 子指令 | |
| command line | 命令行 | 命令列 | |
| terminal | 终端 | 終端機 | |
| run (a program) | 运行 | 執行 | |
| execute | 执行 | 執行 | |
| compile | 编译 | 編譯 | |
| compiler | 编译器 | 編譯器 | `nabic` |
| build (v., n.) | 构建 | 建置 | build target: 构建目标 / 建置目標 |
| build cache | 构建缓存 | 建置快取 | |
| cache | 缓存 | 快取 | |
| incremental build | 增量构建 | 增量建置 | |
| toolchain | 工具链 | 工具鏈 | |
| component (toolchain) | 组件 | 元件 | `nabi component add` |
| target | 目标 | 目標 | target triple: 目标三元组 / 目標三元組; in tables just "triple" |
| host | 主机 | 主機 | the machine the toolchain runs on |
| platform | 平台 | 平台 | |
| cross-compilation, cross-compile | 交叉编译 | 交叉編譯 | |
| emulation, emulator | 模拟，模拟器 | 模擬，模擬器 | |
| simulator (iOS) | 模拟器 | 模擬器 | iOS 模拟器 / iOS 模擬器 |
| real hardware | 真实硬件 | 實體硬體 | |
| architecture | 架构 | 架構 | |
| 32-bit, 64-bit | 32 位 | 32 位元 | |
| little-endian | 小端 | 小端序 | |
| hard float | 硬浮点 | 硬體浮點 | |
| embedded | 嵌入式 | 嵌入式 | |
| static linking, static | 静态链接，静态 | 靜態連結，靜態 | |
| WebAssembly, wasm | WebAssembly，wasm | WebAssembly，wasm | not translated |
| executable | 可执行文件 | 執行檔 | |
| binary | 二进制文件 | 二進位檔 | |
| library | 库 | 函式庫 | |
| standard library | 标准库 | 標準函式庫 | |
| standard library tier (`bare`, `core`, `std`) | 标准库层级 | 標準函式庫層級 | tier names stay in code font |
| shared library | 共享库 | 共用函式庫 | |
| static library | 静态库 | 靜態函式庫 | |
| runtime | 运行时 | 執行環境 | at run time: 运行时 / 執行時 |
| code generator | 代码生成器 | 程式碼產生器 | |
| codegen backend | 代码生成后端 | 程式碼產生後端 | `-C codegen-backend` |
| backend | 后端 | 後端 | |
| front end | 前端 | 前端 | |
| linker | 链接器 | 連結器 | |
| link (v.) | 链接 | 連結 | also a hyperlink |
| optimizer | 优化器 | 最佳化器 | |
| optimize, optimization | 优化 | 最佳化 | optimization level: 优化级别 / 最佳化等級 |
| inlining | 内联 | 內嵌 | |
| intermediate representation | 中间表示 | 中間表示 | HIR, MIR stay |
| lowering | 降级 | 降階 | spec term; may add (lowering) |
| calling convention | 调用约定 | 呼叫慣例 | |
| call (v.) | 调用 | 呼叫 | |
| symbol | 符号 | 符號 | |
| symbol mangling | 符号修饰 | 符號修飾 | |
| ABI | ABI | ABI | not translated |
| FFI | FFI | FFI | 外部函数接口 / 外部函式介面 on first mention |
| C header | C 头文件 | C 標頭檔 | |
| binding | 绑定 | 繫結 | `nabi bindgen`; also a pattern binding |
| JIT | JIT（即时编译） | JIT（即時編譯） | |
| AOT | AOT（预先编译） | AOT（預先編譯） | |
| REPL | REPL | REPL | not translated |
| Jupyter kernel | Jupyter 内核 | Jupyter 核心 | |
| notebook | 笔记本 | 筆記本 | a Jupyter notebook: Jupyter Notebook |
| language server | 语言服务器 | 語言伺服器 | `nabils` |
| editor support | 编辑器支持 | 編輯器支援 | |
| editor extension | 编辑器扩展 | 編輯器擴充功能 | |
| formatter | 格式化工具 | 格式化工具 | `nabi fmt` |
| linter, lint | lint 工具，lint | lint 工具，lint | `nabi lint` |
| debugger, debug | 调试器，调试 | 偵錯工具，偵錯 | |
| diagnostic | 诊断 | 診斷 | |
| error code | 错误码 | 錯誤碼 | |
| compile error | 编译错误 | 編譯錯誤 | |
| warning | 警告 | 警告 | |
| comment | 注释 | 註解 | |
| doc comment | 文档注释 | 文件註解 | |
| API docs | API 文档 | API 文件 | |
| test | 测试 | 測試 | `nabi test` |
| doc test | 文档测试 | 文件測試 | |
| benchmark | 基准测试 | 基準測試 | `nabi bench` |
| fuzzing | 模糊测试 | 模糊測試 | |
| parse, parser | 解析，解析器 | 剖析，剖析器 | |
| syntax highlighting | 语法高亮 | 語法標示 | |
| compute kernel (MLIR, offload) | 内核 | 核心函式 | |
| show and tell (Discussions) | 作品展示 | 作品分享 | |
| plugin | 插件 | 外掛 | |
| hot reload | 热重载 | 熱重載 | |
| reflection | 反射 | 反射 | |
| conditional compilation | 条件编译 | 條件式編譯 | `#[cfg]` |
| package | 包 | 套件 | zh-Hans 软件包 only when "package" alone is unclear |
| official package, first-party package | 官方包 | 官方套件 | first-party = 官方 |
| package registry, registry | 包注册表，注册表 | 套件登錄庫，登錄庫 | packages.nabi-lang.org; may be named by its address |
| registry index | 注册表索引 | 登錄庫索引 | `packages-index` |
| index | 索引 | 索引 | |
| publish | 发布 | 發布 | `nabi publish` |
| publisher | 发布者 | 發布者 | |
| trusted publishing | 可信发布 | 受信任發布 | trusted publisher: 可信发布者 / 受信任的發布者 |
| yank, unyank | 撤回，取消撤回 | 撤回，取消撤回 | a yanked version stays downloadable for locked builds |
| owner (of a package) | 所有者 | 擁有者 | |
| maintainer | 维护者 | 維護者 | |
| contributor | 贡献者 | 貢獻者 | |
| manifest | 清单 | 清單 | `nabi.nabion` |
| lockfile | 锁文件 | 鎖定檔 | `nabi.lock` |
| dependency | 依赖 | 相依套件 | depends on: 依赖 / 相依於 |
| dependency graph, build graph | 依赖图，构建图 | 相依關係圖，建置圖 | |
| version | 版本 | 版本 | |
| release (n.) | 版本，发布 | 版本，發行 | a release = 一个版本 / 一個版本 |
| release (v.) | 发布 | 發行 | |
| release notes | 发布说明 | 發行說明 | |
| release archive, archive | 发布包，压缩包 | 發行封存檔，壓縮檔 | the `.tar.xz` / `.zip` files |
| archive (`.a`) | 归档文件 | 封存檔 | compiled package output |
| pin (a version) | 固定 | 固定 | |
| semver, semantic versioning | 语义化版本 | 語意化版本 | |
| major / minor / patch version | 主 / 次 / 修订版本号 | 主 / 次 / 修訂版本號 | |
| series (0.1 series) | 系列 | 系列 | 0.1 系列 |
| breaking change | 破坏性变更 | 破壞性變更 | |
| stable, stability | 稳定，稳定性 | 穩定，穩定性 | |
| deprecate | 弃用 | 棄用 | Nabi does not deprecate; it changes |
| feature (package) | 特性 | 功能特性 | `#[cfg(feature = "...")]`; Cargo zh uses 特性 |
| feature (capability, in prose) | 功能 | 功能 | |
| workspace | 工作区 | 工作區 | |
| environment | 环境 | 環境 | spec term, the unit `nabi` keeps a dependency set in |
| profile (build) | 构建配置 | 建置設定檔 | `profile: { "release": ... }` |
| advisory, security advisory | 安全公告 | 安全性公告 | `nabi audit` |
| vulnerability | 漏洞 | 漏洞 | |
| audit (n.) | 审计 | 稽核 | `nabi audit` stays |
| token (credential) | 令牌 | 權杖 | publish token: 发布令牌 / 發布權杖 |
| token (lexical) | 词法单元 | 語彙單元 | |
| signature, sign | 签名 | 簽章 | |
| signing key | 签名密钥 | 簽署金鑰 | |
| public key | 公钥 | 公開金鑰 | |
| checksum | 校验和 | 總和檢查碼 | |
| hash | 哈希 | 雜湊 | |
| build provenance attestation | 构建来源证明 | 建置來源證明 | GitHub artifact attestation |
| code signing | 代码签名 | 程式碼簽署 | |
| notarization | 公证 | 公證 | macOS |
| cryptography | 密码学 | 密碼學 | `core::crypto` |
| encryption | 加密 | 加密 | |
| key material | 密钥材料 | 金鑰材料 | |
| sandbox | 沙箱 | 沙箱 | |
| rate limit | 速率限制 | 速率限制 | |
| repository | 仓库 | 儲存庫 | |
| commit (n., v.) | 提交 | 提交 | |
| sign-off | 签署（sign-off） | 簽署（sign-off） | DCO |
| issue | issue | issue | not translated |
| pull request | pull request | pull request | not translated |
| fork | 复刻 | 分叉 | GitHub's zh-CN term; 派生 is derive |
| milestone | 里程碑 | 里程碑 | |
| roadmap | 路线图 | 路線圖 | |
| community | 社区 | 社群 | |
| governance | 治理 | 治理 | |
| code of conduct | 行为准则 | 行為準則 | |
| license | 许可证 | 授權條款 | |
| open source | 开源 | 開源 | |
| specification, spec | 规范 | 規格 | the language reference |
| language reference | 语言参考 | 語言參考 | |
| chapter / section | 章 / 节 | 章 / 節 | |
| guide | 指南 | 指南 | |
| getting started | 入门 | 入門 | |
| cookbook | 实用示例 | 實用範例 | |
| type | 类型 | 型別 | |
| type system | 类型系统 | 型別系統 | |
| static typing, statically typed | 静态类型 | 靜態型別 | |
| type inference | 类型推断 | 型別推論 | |
| type parameter | 类型参数 | 型別參數 | |
| sum type | 和类型 | 和型別 | `A \| B` |
| closed (sum type) | 封闭的 | 封閉的 | all variants known |
| variant | 变体 | 變體 | |
| tagged union | 带标签联合 | 帶標籤聯集 | |
| union | 联合体 | 聯集 | `union` |
| optional type `T?` | 可选类型 | 可選型別 | |
| `none` | `none` | `none` | keyword; never 空 or null in prose about Nabi |
| null | null | null | "no null": 没有 null / 沒有 null |
| narrowing | 类型收窄 | 型別收窄 | `is` |
| struct | 结构体 | 結構 | the keyword is `struct` |
| enum | 枚举 | 列舉 | the keyword is `enum` |
| enum case | 枚举项 | 列舉項 | |
| field | 字段 | 欄位 | |
| field default | 字段默认值 | 欄位預設值 | |
| method | 方法 | 方法 | |
| trait | trait | trait | see Rules; keyword `trait` |
| trait bound | trait 约束 | trait 約束 | |
| trait-as-type ban | trait 不能作类型 | trait 不能作為型別 | spec 2.12 |
| impl | 实现 | 實作 | the keyword is `impl` |
| implement | 实现 | 實作 | |
| default method | 默认方法 | 預設方法 | |
| associated type / constant | 关联类型 / 常量 | 關聯型別 / 常數 | |
| generic, generics | 泛型 | 泛型 | |
| const generics | 常量泛型 | 常數泛型 | |
| monomorphization | 单态化 | 單態化 | |
| specialize | 特化 | 特化 | |
| concrete | 具体的 | 具體的 | |
| dispatch | 分派 | 分派 | |
| `any` slot | `any` 槽 | `any` 槽位 | |
| lang slot, magic | lang 槽，魔法 | lang 槽位，魔法 | |
| niche optimization | niche 优化 | niche 最佳化 | |
| interface | 接口 | 介面 | |
| object | 对象 | 物件 | |
| value | 值 | 值 | |
| handle | 句柄 | 控制代碼 | a reference value |
| capability | 能力 | 能力 | `Dir` is a capability |
| variable | 变量 | 變數 | |
| constant | 常量 | 常數 | `const` |
| static | 静态变量 | 靜態變數 | `static` |
| immutable / mutable | 不可变 / 可变 | 不可變 / 可變 | |
| reassignment | 重新赋值 | 重新指派 | |
| assignment | 赋值 | 指派 | |
| expression | 表达式 | 運算式 | |
| statement | 语句 | 陳述式 | |
| statement terminator | 语句终止符 | 陳述式終止符 | |
| block | 块 | 區塊 | |
| body | 主体 | 主體 | function body: 函数体 / 函式主體 |
| operator | 运算符 | 運算子 | |
| operand | 操作数 | 運算元 | |
| precedence | 优先级 | 優先順序 | |
| literal | 字面量 | 字面值 | |
| identifier | 标识符 | 識別字 | |
| keyword | 关键字 | 關鍵字 | |
| reserved word | 保留字 | 保留字 | |
| function | 函数 | 函式 | |
| function signature | 函数签名 | 函式簽章 | |
| parameter | 参数 | 參數 | |
| argument | 实参 | 引數 | zh-Hans 参数 when no contrast with parameter |
| named argument | 具名参数 | 具名引數 | |
| default value | 默认值 | 預設值 | |
| return (v.) | 返回 | 回傳 | |
| return value | 返回值 | 回傳值 | |
| return type | 返回类型 | 回傳型別 | |
| early return | 提前返回 | 提前回傳 | |
| closure | 闭包 | 閉包 | |
| lambda | lambda | lambda | |
| trailing closure | 尾随闭包 | 尾隨閉包 | |
| capture | 捕获 | 擷取 | |
| pipe `\|>` | 管道 | 管線 | |
| entry point | 入口点 | 進入點 | |
| nested function | 嵌套函数 | 巢狀函式 | |
| generator | 生成器 | 產生器 | `yield` |
| iterator | 迭代器 | 迭代器 | |
| iterator adapter | 迭代器适配器 | 迭代器配接器 | |
| iteration | 迭代 | 迭代 | |
| loop | 循环 | 迴圈 | |
| label | 标签 | 標籤 | |
| control flow | 控制流 | 控制流程 | |
| pattern | 模式 | 模式 | |
| pattern matching | 模式匹配 | 模式比對 | |
| arm (of `match`) | 分支 | 分支 | |
| exhaustiveness, exhaustive | 穷尽性，穷尽的 | 窮盡性，窮盡的 | exhaustiveness check: 穷尽检查 / 窮盡檢查 |
| destructuring | 解构 | 解構 | |
| optional chaining / coalescing | 可选链 / 空值合并 | 可選鏈結 / 空值合併 | `?.` / `??` |
| range | 区间 | 範圍 | |
| broadcast | 广播 | 廣播 | |
| collection | 集合类型 | 集合型別 | the container types |
| list | 列表 | 串列 | `List` |
| array | 数组 | 陣列 | `Array` (core's n-dimensional array) |
| fixed-size array | 定长数组 | 固定長度陣列 | |
| tuple | 元组 | 元組 | |
| dictionary, map | 字典，映射 | 字典，對應 | `Dict` |
| set | 集合 | 集合 | `Set` |
| slice, slicing | 切片 | 切片 | |
| index (v., subscript) | 索引 | 索引 | |
| string | 字符串 | 字串 | `str` |
| string builder | 字符串构建器 | 字串建構器 | `StrBuf` |
| f-string | f-string | f-string | |
| format specifier | 格式说明符 | 格式規範 | |
| byte string | 字节串 | 位元組字串 | |
| raw string | 原始字符串 | 原始字串 | |
| escape | 转义 | 跳脫 | |
| char, character | 字符 | 字元 | `char` |
| byte | 字节 | 位元組 | |
| bool, boolean | 布尔值 | 布林值 | `bool` |
| integer | 整数 | 整數 | |
| integer width (8/16/32/64-bit) | 整数位宽 | 整數位元寬度 | `i8` ... `u64` |
| signed / unsigned | 有符号 / 无符号 | 有號 / 無號 | |
| big integer | 大整数 | 大整數 | `int` |
| floating-point number | 浮点数 | 浮點數 | |
| complex number | 复数 | 複數 | |
| overflow | 溢出 | 溢位 | integer overflow: 整数溢出 / 整數溢位 |
| implicit conversion / promotion | 隐式转换 / 隐式提升 | 隱含轉換 / 隱含提升 | "no implicit numeric promotion" |
| cast | 强制转换 | 強制轉型 | `as` |
| Unicode normalization / grapheme | Unicode 规范化 / 字素 | Unicode 正規化 / 字素 | |
| module | 模块 | 模組 | `mod` |
| submodule | 子模块 | 子模組 | |
| visibility | 可见性 | 可見性 | `pub`, `pub(sub)`, `pub(pkg)` stay in code font |
| public / private | 公开 / 私有 | 公開 / 私有 | |
| item | 项 | 項目 | |
| import | 导入 | 匯入 | `use` |
| re-export | 重新导出 | 重新匯出 | |
| export (to C) | 导出 | 匯出 | `#[export]` |
| attribute | 属性 | 屬性 | `#[...]`; property also renders 属性, so keep the `#[...]` form near |
| derive | 派生 | 衍生 | `#[derive(...)]` |
| macro | 宏 | 巨集 | Nabi has no user macros; used for `include_str!`, `cfg!` |
| meta-system, meta block | 元系统，元块 | 元系統，元區塊 | |
| template token | 模板记号 | 樣板記號 | |
| error | 错误 | 錯誤 | |
| error handling | 错误处理 | 錯誤處理 | |
| effect, `throws` | 错误效应，`throws` | 錯誤效應，`throws` | "errors in function signatures": 错误写在函数签名中 / 錯誤寫在函式簽章中 |
| throw / catch | 抛出 / 捕获 | 擲回 / 攔截 | keywords `throw`, `try`, `catch` |
| error type | 错误类型 | 錯誤型別 | |
| error union | 错误联合 | 錯誤聯集 | the set of types a `throws` clause names |
| exception | 异常 | 例外 | Nabi has no exceptions; use for comparisons |
| panic | panic | panic | not translated; verb: 触发 panic / 觸發 panic |
| undefined behavior | 未定义行为 | 未定義行為 | |
| unsafe | 不安全 | 不安全 | `unsafe` block / fn stay in code font |
| `bare` | `bare` | `bare` | the lowest library tier, not translated |
| pointer / raw pointer | 指针 / 裸指针 | 指標 / 原始指標 | |
| resource cleanup | 资源清理 | 資源清理 | `using` |
| ownership | 所有权 | 所有權 | Nabi has no ownership or borrow checker (GC) |
| borrow, borrow checker | 借用，借用检查器 | 借用，借用檢查器 | Rust comparisons only |
| move (a value into a thread) | 移动 | 移動 | `[move x]` |
| garbage collector, GC | 垃圾回收器，GC | 垃圾回收器，GC | |
| garbage collection | 垃圾回收 | 垃圾回收 | |
| generational / moving GC | 分代 / 移动式 GC | 分代 / 移動式 GC | |
| GC pause | GC 停顿 | GC 暫停 | |
| heap / stack | 堆 / 栈 | 堆積 / 堆疊 | |
| allocation | 分配 | 配置 | |
| object layout | 对象布局 | 物件配置 | |
| root / barrier | 根 / 屏障 | 根 / 屏障 | |
| safepoint | 安全点 | 安全點 | |
| memory model | 内存模型 | 記憶體模型 | |
| concurrency, concurrent | 并发 | 並行 | see Rules |
| parallelism, parallel | 并行 | 平行 | see Rules; `par` |
| thread | 线程 | 執行緒 | |
| thread-local | 线程局部 | 執行緒區域 | |
| async, asynchronous | 异步 | 非同步 | |
| task | 任务 | 任務 | |
| spawn (a thread, a task) | 启动 | 啟動 | `thread::spawn`, `task::spawn`; not 派生, which is derive |
| coroutine | 协程 | 協程 | |
| scheduler | 调度器 | 排程器 | |
| channel | 通道 | 通道 | |
| stream | 流 | 串流 | |
| mutex | 互斥锁 | 互斥鎖 | |
| atomic | 原子 | 原子 | |
| freeze | 冻结 | 凍結 | `freeze` |
| I/O | I/O | I/O | |
| child process | 子进程 | 子行程 | |
| standard input / output | 标准输入 / 输出 | 標準輸入 / 輸出 | |
| NABION | NABION | NABION | not translated; `.nabion` files |
| SIMD vector | SIMD 向量 | SIMD 向量 | |
| reduction | 归约 | 歸約 | |
| missing value | 缺失值 | 缺失值 | |
| playground | Playground | Playground | playground.nabi-lang.org; name kept |
| scope (lexical) | 作用域 | 作用域 | |
| scope (OAuth, token) | 权限范围 | 權限範圍 | |
| privacy notice | 隐私声明 | 隱私權聲明 | |
| terms of use | 使用条款 | 使用條款 | |
| draft | 草稿 | 草稿 | |
| processor (GDPR) | 处理者 | 處理者 | |
| legitimate interest | 正当利益 | 正當利益 | |
| performance of a contract | 履行合同 | 履行契約 | |
| data protection authority | 数据保护机构 | 資料保護主管機關 | |
| moderation | 内容管理 | 內容管理 | |
| warranty / liability | 担保 / 责任 | 擔保 / 責任 | |
| shipped (roadmap) | 已发布 | 已發布 | |
| in progress | 进行中 | 進行中 | |
| next / later (roadmap) | 下一步 / 以后 | 下一步 / 之後 | |
| supported (tier) | 已支持 | 已支援 | |
| news | 新闻 | 最新消息 | |
| announcement | 公告 | 公告 | "Announcing Nabi 0.1.0": Nabi 0.1.0 发布 / Nabi 0.1.0 正式發布 |
| FAQ | 常见问题 | 常見問題 | |
| brand, mark, wordmark | 品牌，标志，文字标志 | 品牌，標誌，文字標誌 | |
| slogan | 标语 | 標語 | "Clear rules, free as a butterfly." is 规则清晰，如蝶般自由。 in zh-Hans and 規則清晰，如蝶般自由。 in zh-Hant |
| code point | 码位 | 碼位 | Unicode code point |
| clamp | 钳制 | 夾限 | to clamp a value into a range |
| stop-the-world (pause) | 全局停顿（stop-the-world） | stop-the-world 暫停 | zh-Hant keeps the English term |
| trust store | 信任存储 | 信任存放區 | the OS root certificates |
| desugar, desugaring | 脱糖 | 去語法糖 | |
