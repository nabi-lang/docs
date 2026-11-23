# 1. 发布一个包

发布就是在注册表索引中添加一行，指向你 git 仓库中的一个提交（[注册表如何运作](index.html#注册表如何运作)）。本页从清单讲到发布一个版本，以及如何撤回一个版本。

## 包需要什么

- 仓库根目录下的 `nabi.nabion`，其中有 `name`、`version` 和 `nabi_version`。
- 一个 `src/lib.nabi`（库）、一个 `src/main.nabi`（供 `nabi install` 安装的程序）或两者都有，以及一个 readme。
- 一个干净的 git 工作树，带有远程仓库 `origin`，其 url 是公开主机上的 `https://` 地址，并且提交已推送到它的某个分支或标签（只存在于复刻仓库中的提交不算）。
- 所有普通依赖都来自注册表。git 依赖会被拒绝；路径依赖也会被拒绝，除非它同时写明发布时用来代替它的版本：`Dep::path("../geom", version: "^0.1")`（先发布 `geom`）。没有 `version:` 的路径和 git `dev_deps` 会被去掉。
- 清单中没有 `publish: false`。

把 `nabi_version` 设为你构建过这个包的最旧的 nabi：`nabi publish` 需要它，而解析时会给每个用户其工具链能构建的最新版本（`"0.1.3"` 接受 nabi 0.1.3 及之后的 0.1.x，见[第 1 章](../01-getting-started.html#管理工具链)）。每个包的页面把它显示为“Requires”。`nabi new` 会写入创建这个包的版本。

填写 `description`、`license` 和 `repository`：它们会出现在 `nabi search` 和网站上。包页面显示的其他字段都是可选的，与 Cargo 相同：

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

`keywords` 和 `categories` 各自最多 5 项；`include` / `exclude` 选择包中包含的文件；还有 `license_file`、`readme`、`links`、`default_run` 和一个自由格式的 `metadata` 表。`nabi` 不认识的字段只是警告而不是错误，并且照样发送给注册表。

## `nabi login`

用 GitHub 登录 [packages.nabi-lang.org/me](https://packages.nabi-lang.org/me)，在那里创建一个账户令牌，然后把它交给 `nabi`：

```text
$ nabi login
a token from https://packages.nabi-lang.org/me (not shown; Ctrl-C to cancel):
   Logged in as your-name at https://packages.nabi-lang.org (token in ~/.nabi/tokens.nabion)
```

不带参数时，令牌从标准输入读取，不回显。`nabi login` 会向注册表询问令牌属于谁，注册表拒绝的令牌会被拒收，什么也不保存。有效的令牌保存在 `~/.nabi/tokens.nabion`（只有你能读取）中，按签发它的索引分别存放，并且只发送给那个索引；`nabi login --registry <url>` 为另一个索引保存令牌。`nabi logout` 删除它，`nabi logout --all` 删除所有索引的令牌。在 CI 中改为设置 `NABI_REGISTRY_TOKEN`，它只会发送给 packages.nabi-lang.org。从 GitHub Actions 发布时，[可信发布](02-trusted-publishing.html)完全不需要存储令牌。

## `nabi publish`

`nabi publish --dry-run` 打印它将要添加的索引条目，不向外发送任何东西。`nabi publish` 则直接发布，与 `cargo publish` 相同。

```text
$ nabi publish --dry-run
Package { name: "textwrap", version: "0.1.0", deps: [], source: Dep::git("https://github.com/you/textwrap.git", rev: "8e2afb20.."), tree: "823a3592..", yanked: false, nabi_version: "0.1.0" }
note: dry run; `nabi publish` sends it to the registry API of https://github.com/nabi-lang/packages-index (needs `nabi login`), which adds it as te/xt/textwrap
```

条目只包含解析所需的内容：HEAD 提交及其树哈希、依赖和 `nabi_version`。请先推送那个提交。描述、许可证等其他元数据随条目一起发送到注册表，注册表会连同每个版本的发布者一起显示它们。`nabi publish` 用你的令牌把两者一起发送（`Published textwrap 0.1.0 to https://packages.nabi-lang.org`），索引收录之后，这个版本就上线了。同一个版本不能发布两次。

带有可执行文件的包在列表和搜索结果中标为 **bin**，它的页面会显示 `nabi install <name>`（同时有库时显示 `nabi install --bin <name>`）。

## 名称与所有者

名称最多 64 个字符。与已有的包只差大小写或 `-` / `_` 的名称视为已被占用。标准库的层级 `std`、`core` 和 `bare` 是索引中归 Nabi 项目所有的条目：它们随工具链一起提供，没有任何东西以这些名称发布。工具链的名称（`nabi`、`nabic` 等）是保留的；与索引中某个名称只差一个字母，或者把形近字符归并后相同（`js0n` 之于 `json`）的新名称会被拒绝；如果仍想申请这样的名称，[使用政策](https://packages.nabi-lang.org/policy)说明了怎么做。

发布一个包第一个版本的人就是它的所有者。所有者可以在[账户页面](https://packages.nabi-lang.org/me)邀请其他账户共同拥有它；被邀请的账户接受后即成为所有者。

## API 文档

注册表根据每个版本所在提交的源代码生成它的 API 文档，只读取源代码，不构建也不运行；文档站点负责展示，就像 docs.rs 之于 crates.io：`docs.nabi-lang.org/packages/textwrap/0.1.0`，最新版本则是 `/packages/textwrap/latest`（真实的例子见 [json 的文档](https://docs.nabi-lang.org/packages/json/latest)）。注册表上包的页面在 Docs 标签页中链接到那里。页面的布局与 `nabi doc` 相同：根模块在那个地址，其他每个公开模块各有自己的页面（`/packages/textwrap/0.1.0/<module>`），锚点也相同（`#fn.wrap`）。读者最先看到的是根模块的 `//!` 文字，所以包的介绍和教程就写在那里；其中的代码块是 `nabi test` 运行的文档测试，以 `# ` 开头的行是页面上省略的准备代码。模块的文档是它的文件开头的 `//!` 行；普通的 `//` 注释不会显示。用 `pub use` 从私有模块重新导出的项，会显示为声明在重新导出的位置。只有可执行文件的包没有 API，因此也没有文档。如果文档无法生成，版本照样发布：`nabi publish` 以警告打印原因，文档页面也会显示它。

## `nabi yank`

已发布的版本永远不会被删除。有严重 bug 的版本可以撤回：新的解析会跳过它，但 `nabi.lock` 已经固定了它的项目照常构建，所以撤回不会破坏任何人的构建。`--undo` 取消撤回。

```bash
nabi yank textwrap@0.1.0 --message "wraps CJK text wrong; use 0.1.1"
nabi yank textwrap@0.1.0 --undo
```

`--message` 说明原因，注册表会在包的页面上显示它。

## 延伸阅读

- 规范：[9.10.4a 注册表包](../../spec/refs-lang/009-tools/910-nabi-frontend.html#9104a-注册表包)、[`nabi search` / `publish` / `yank` / `login`](../../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-search--publish--yank--login--logout)、[`nabi.nabion`](../../spec/refs-lang/009-tools/910-nabi-frontend.html#9103-nabinabion--nabilock)
- 下一页：[2. 可信发布](02-trusted-publishing.html)
