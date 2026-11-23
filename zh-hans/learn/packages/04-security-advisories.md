# 安全公告

安全公告说明一个包的某些版本存在漏洞、是哪些版本，以及哪个版本修复了它。注册表在 [packages.nabi-lang.org/advisories](https://packages.nabi-lang.org/advisories) 以其他生态系统也在使用的 [OSV](https://ossf.github.io/osv-schema/) 格式发布安全公告，`nabi audit` 用它们检查你的项目。

## 检查项目：`nabi audit`

```sh
nabi audit
```

`nabi audit` 读取 `nabi.lock`，把其中的每个注册表包（包括工具链自带的官方包）与其注册表的公告对照检查。对每条适用的公告，它打印公告的 id 和摘要、公告给出的严重程度（如果有），以及应升级到的版本；然后运行 `nabi update <name>`（第 11 章），或者在 `nabi.nabion` 中提高版本要求。路径依赖、git 依赖和原生包不检查，它会说明跳过了哪些。没有 `nabi.lock` 时，它会先解析出一个。

退出状态说明检查的结果：

| 状态 | 含义 |
|--------|---------|
| 0 | 没有适用的公告。 |
| 1 | 至少有一条适用。 |
| 2 | 无法检查：没有网络也没有缓存的列表、锁文件无法读取，或注册表返回错误。消息会说明是哪种情况。 |

列表只获取一次（每个注册表一次请求），并在 `$NABI_HOME/cache/advisories` 下缓存一小时。`--offline` 不论新旧都使用这份副本，获取失败时也会退回到它。

| 标志 | 作用 |
|------|--------|
| `--json` | 每行一个 JSON 对象：每项发现一条记录，每个未检查的包一条，最后是摘要。与 `--format json` 相同。 |
| `--ignore <id>` | 跳过一条公告，按它的 id（`NABI-2026-0001`）或别名（`CVE-..`、`GHSA-..`）。可重复使用。 |
| `--offline` | 只使用缓存的列表。 |

在 CI 中，在构建之后运行它；状态为 1 会使任务失败：

```yaml
- run: nabi audit
```

只有在读过一条公告、确定你的程序不会执行到受影响的代码之后，才忽略它；并在该标志旁边写明原因。

## 阅读公告

每条公告在注册表上都有一个页面，受它影响的版本的页面上有一条链接到它的横幅。公告写明：

- **受影响的版本**：例如“从 0.1.0 起，在 0.2.3 中修复”这样的区间，或一个版本列表；
- **已修复的版本**（如果有）：升级到其中之一；
- **严重程度**：报告者给出时，为一个 CVSS 向量；
- **参考与致谢**：修复、报告，以及同一缺陷的其他 id。

已撤销的公告仍会列出，并标为已撤销，`nabi audit` 会跳过它。

整个列表也是一个供其他工具使用的 API：`GET https://packages.nabi-lang.org/api/v1/advisories?format=json` 以 OSV JSON 给出所有公告，`/api/v1/packages/<name>/advisories` 给出一个包的公告。字段见 [API 参考](https://packages.nabi-lang.org/api)。

## 报告漏洞

在某个包中发现了漏洞？先私下告诉它的所有者，他们仓库的安全策略会说明方式。如果他们没有回应，或者缺陷在注册表或工具链中，请写信到 [security@nabi-lang.org](mailto:security@nabi-lang.org)。

包的所有者可以通过 API 为它起草公告（用带有 `owners` 权限范围的账户令牌调用 `POST /api/v1/advisories`）；注册表的维护者审核并发布它，发布时包的每位所有者都会收到一封邮件。
