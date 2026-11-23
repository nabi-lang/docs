# 2. 受信任發布

GitHub Actions 工作流程可以在儲存庫中不存放任何權杖的情況下發布套件。GitHub 會給每個工作一個 OIDC 權杖，說明它是為哪個儲存庫、工作流程與環境執行；登錄庫會檢查權杖上 GitHub 的簽章，如果套件信任那個工作流程，就把它換成一個效期 30 分鐘、只對該套件有效的發布權杖。這和 crates.io、PyPI 與 npm 的受信任發布是同一種機制。

每個套件只需設定一次：

1. 在你的[帳戶頁面](https://packages.nabi-lang.org/me)的 **Trusted publishers** 下，加入套件、它的 GitHub 儲存庫（`owner/repo`）、工作流程的檔名（`publish.yml`），如果工作在某個 GitHub 環境中執行，再加上環境的名稱。只有套件的擁有者可以加入；在發布尚未開放期間，只有登錄庫的維護者可以加入。沒有登記擁有者的套件（登錄庫維護者用 `nabi publish --index-dir` 直接寫入索引的套件），由維護者為它加入第一個受信任的發布者。
2. 把下面的工作流程以 `.github/workflows/publish.yml` 加入儲存庫。它會在推送 `v*` 標籤時執行，檢查標籤與 `nabi.nabion` 中的 `version` 相符，把工作的 OIDC 權杖（audience 為 `packages.nabi-lang.org`）換成發布權杖，用它登入並發布。

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

之後用 `git tag v0.3.1 && git push origin v0.3.1` 發行新版本。如果標籤是在工作流程加入儲存庫之前推送的，可以從 Actions 分頁手動執行它（或用 `gh workflow run publish.yml -f tag=v0.3.1`）：它會執行預設分支上的工作流程，並發布該標籤的提交。除了 `nabi login`，這個步驟也可以把權杖放進 `NABI_REGISTRY_TOKEN` 給發布步驟使用（`echo "NABI_REGISTRY_TOKEN=$tok" >> "$GITHUB_ENV"`）。

指定的工作流程檔案不論在哪個 ref 上執行都能發布。因此請為發布者指定一個環境：建立一個設有必要審查者與 `v*` 標籤部署規則的 GitHub 環境（`release`），在工作上設定 `environment: release`，並在這裡填入它的名稱；之後登錄庫就會拒絕來自該環境以外的執行的權杖。如果不使用環境，而其他人可以推送到儲存庫，請保護你的 `v*` 標籤。版本頁面會註明發布它的工作流程。

已在索引中但沒有中繼資料的版本（登錄庫維護者用 `nabi publish --index-dir` 直接寫入的版本），也用同一個指令補上中繼資料：對同一棵樹執行 `nabi publish`，會補齊描述、授權條款、README 等內容，其他都不會改變。

## 延伸閱讀

- 規格：[`nabi search` / `publish` / `yank` / `login`](../../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-search--publish--yank--login--logout)
- 下一頁：[3. 保持 API 穩定](03-stable-api.html)
