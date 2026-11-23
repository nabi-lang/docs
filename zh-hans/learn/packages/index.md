# 发布包

本部分介绍如何把你自己的库或程序放到包注册表 [packages.nabi-lang.org](https://packages.nabi-lang.org) 上并持续维护。如何使用包见[第 11 章](../11-modules-packages.html#包注册表)，特性见[第 18 章](../18-features-and-publishing.html)。

> [!IMPORTANT]
> 目前只有注册表的维护者可以发布。任何人都可以登录并使用包，但 `nabi login` 会拒绝尚不能发布的账户。

## 注册表如何运作

注册表的数据是一个 git 仓库，即[索引](https://github.com/nabi-lang/packages-index)。包的每个版本是其中的一行，记录包自己 git 仓库中的一个提交以及该提交的树哈希。你不需要上传源代码：一个版本发布时，注册表获取那个提交，保存它的压缩包，并把压缩包的 SHA-256 写进这一行。用户下载压缩包并对照索引检查，所以即使仓库不在了，版本也依然存在。这些行只由注册表写入，索引不接受 pull request。行的格式见[索引的 README](https://github.com/nabi-lang/packages-index#readme)。

从 nabi-lang 组织发布的包标为 **Official**，它们随工具链一起提供（[第 11 章](../11-modules-packages.html#包注册表)）。

## 各页内容

| 页面 | 内容 |
|------|----------------|
| [发布一个包](01-publishing.html) | 包需要什么、`nabi login`、`nabi publish`、名称与所有者、注册表生成的文档、`nabi yank` |
| [可信发布](02-trusted-publishing.html) | 不存储令牌，从 GitHub Actions 发布 |
| [保持 API 稳定](03-stable-api.html) | `#[deprecated]`、`#[non_exhaustive]` 和 `#[sealed]` |
| [安全公告](04-security-advisories.html) | `nabi audit`、阅读公告、报告漏洞 |

注册表自己的规则在注册表上：[使用政策](https://packages.nabi-lang.org/policy)、[使用条款](https://packages.nabi-lang.org/terms)和[隐私声明](https://packages.nabi-lang.org/privacy)。
