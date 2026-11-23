# 2. 可信发布

GitHub Actions 工作流可以在仓库中不存储任何令牌的情况下发布包。GitHub 为每个任务提供一个 OIDC 令牌，说明它为哪个仓库、工作流和环境运行；注册表检查令牌上 GitHub 的签名，如果包信任那个工作流，就把它换成一个有效期 30 分钟、只对该包有效的发布令牌。这与 crates.io、PyPI 和 npm 的可信发布是同一种机制。

每个包只需设置一次：

1. 在你的[账户页面](https://packages.nabi-lang.org/me)的 **Trusted publishers** 下，添加包、它的 GitHub 仓库（`owner/repo`）、工作流的文件名（`publish.yml`），如果任务在某个 GitHub 环境中运行，再加上环境的名称。只有包的所有者可以添加；在发布尚未开放期间，只有注册表的维护者可以添加。没有登记所有者的包（注册表维护者用 `nabi publish --index-dir` 直接写入索引的包）由维护者为它添加第一个可信发布者。
2. 把下面的工作流以 `.github/workflows/publish.yml` 添加到仓库中。它在推送 `v*` 标签时运行，检查标签与 `nabi.nabion` 中的 `version` 一致，把任务的 OIDC 令牌（audience 为 `packages.nabi-lang.org`）换成发布令牌，用它登录并发布。

```yaml
name: Publish / Package to the registry

on:
  push:
    tags: ['v*']
  # a tag pushed before this file was in the repository
  workflow_dispatch:
    inputs:
      tag:
        description: Tag to publish, e.g. v0.3.1
        required: true

permissions:
  contents: read
  id-token: write

jobs:
  publish:
    name: Publish to packages.nabi-lang.org
    runs-on: ubuntu-24.04
    steps:
      - name: Check out the tag
        uses: actions/checkout@v5
        with:
          ref: ${{ inputs.tag || github.ref }}

      - name: Install Nabi
        run: |
          v=$(curl -fsSL https://github.com/nabi-lang/nabi/releases/latest/download/nabi-version)
          curl -fsSL https://nabi.sh | NABI_NO_MODIFY_PATH=1 sh -s -- -q --version "$v"
          echo "$HOME/.nabi/bin" >> "$GITHUB_PATH"

      - name: Check the tag, tests and publish dry run
        env:
          TAG: ${{ inputs.tag || github.ref_name }}
        run: |
          want=$(sed -n 's/^ *version: "\(.*\)",*$/\1/p' nabi.nabion | head -1)
          [ "v$want" = "$TAG" ] || { echo "tag $TAG, but nabi.nabion has version $want" >&2; exit 1; }
          nabi test
          nabi publish --dry-run

      - name: Log in to packages.nabi-lang.org with the OIDC token
        run: |
          jwt=$(curl -fsS -H "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
            "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=packages.nabi-lang.org" | sed -n 's/.*"value":"\([^"]*\)".*/\1/p')
          out=$(curl -sS --fail-with-body -H 'Content-Type: application/nabion' \
            --data-binary "TrustedPublishing { jwt: \"$jwt\" }" \
            https://packages.nabi-lang.org/api/v1/trusted-publishing/token) || { echo "$out" >&2; exit 1; }
          tok=$(printf '%s' "$out" | sed -n 's/.*token: "\(pkt_[0-9a-f]*\)".*/\1/p')
          [ -n "$tok" ]
          echo "::add-mask::$tok"
          printf '%s\n' "$tok" | nabi login

      - name: Publish with nabi publish
        run: nabi publish
```

之后用 `git tag v0.3.1 && git push origin v0.3.1` 发布新版本。对于工作流加入仓库之前就已推送的标签，可以在 Actions 标签页手动运行它（或用 `gh workflow run publish.yml -f tag=v0.3.1`）：它运行默认分支上的工作流，并发布该标签的提交。这一步也可以不用 `nabi login`，而是把令牌放进 `NABI_REGISTRY_TOKEN` 供发布步骤使用（`echo "NABI_REGISTRY_TOKEN=$tok" >> "$GITHUB_ENV"`）。

指定的工作流文件在任何 ref 上运行都可以发布。因此请给发布者指定一个环境：创建一个带有必需审阅者和 `v*` 标签部署规则的 GitHub 环境（`release`），在任务上设置 `environment: release`，并在这里填写它的名称；之后注册表会拒绝该环境之外的工作流运行所带的令牌。如果不用环境，而其他人可以向仓库推送，请保护你的 `v*` 标签。版本页面会写明发布它的工作流。

已在索引中但没有元数据的版本（注册表维护者用 `nabi publish --index-dir` 直接写入的版本）也用同一个命令补上元数据：对同一棵树运行 `nabi publish`，会补全描述、许可证、README 等内容，不改变其他任何东西。

## 延伸阅读

- 规范：[`nabi search` / `publish` / `yank` / `login`](../../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-search--publish--yank--login--logout)
- 下一页：[3. 保持 API 稳定](03-stable-api.html)
