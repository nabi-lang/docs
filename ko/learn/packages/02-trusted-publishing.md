# 2. 신뢰할 수 있는 공개

GitHub Actions 워크플로는 저장소에 토큰을 하나도 저장하지 않고 패키지를 공개할 수 있습니다. GitHub는 작업(job)마다 어느 저장소, 워크플로, 환경에서 실행되는지 밝히는 OIDC 토큰을 줍니다. 레지스트리는 그 토큰에 있는 GitHub의 서명을 검사하고, 패키지가 그 워크플로를 신뢰하면 30분 동안 그 패키지에만 쓸 수 있는 공개 토큰으로 바꿔 줍니다. crates.io, PyPI, npm의 trusted publishing과 같은 방식입니다.

설정은 패키지마다 한 번 합니다.

1. [계정 페이지](https://packages.nabi-lang.org/me)의 **Trusted publishers**에서 패키지, 그 GitHub 저장소(`owner/repo`), 워크플로 파일 이름(`publish.yml`)을 더하고, 작업이 GitHub 환경(environment)에서 실행된다면 그 이름도 적습니다. 패키지의 소유자만 더할 수 있고, 공개가 아직 열리지 않은 지금은 레지스트리 관리자만 더할 수 있습니다. 기록된 소유자가 없는 패키지(레지스트리 관리자가 `nabi publish --index-dir`로 index에 직접 적은 패키지)는 첫 trusted publisher를 관리자에게서 받습니다.
2. 아래 워크플로를 저장소에 `.github/workflows/publish.yml`로 더합니다. 이 워크플로는 `v*` 태그가 push되면 실행되어, 태그가 `nabi.nabion`의 `version`과 맞는지 확인하고, 작업의 OIDC 토큰(audience `packages.nabi-lang.org`)을 공개 토큰으로 바꾼 뒤, 그 토큰으로 로그인해 공개합니다.

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

그다음 `git tag v0.3.1 && git push origin v0.3.1`로 릴리스합니다. 워크플로가 저장소에 들어오기 전에 push한 태그라면 Actions 탭에서 직접 실행합니다(또는 `gh workflow run publish.yml -f tag=v0.3.1`). 그러면 기본 브랜치의 워크플로가 실행되어 그 태그의 커밋을 공개합니다. `nabi login` 대신, 그 단계에서 공개 단계가 쓸 `NABI_REGISTRY_TOKEN`에 토큰을 넣어도 됩니다(`echo "NABI_REGISTRY_TOKEN=$tok" >> "$GITHUB_ENV"`).

지정한 워크플로 파일이라면 어떤 ref에서 실행되든 공개할 수 있습니다. 그러니 publisher에 환경을 지정하세요. 필수 검토자(required reviewers)와 `v*` 태그용 배포 규칙을 둔 GitHub 환경(`release`)을 만들고, 작업에 `environment: release`를 적고, 여기에도 그 이름을 적습니다. 그러면 레지스트리는 그 환경 밖에서 실행된 토큰을 거절합니다. 환경을 두지 않는다면, 다른 사람이 저장소에 push할 수 있을 때 `v*` 태그를 보호하세요. 버전 페이지에는 그 버전을 공개한 워크플로가 나옵니다.

메타데이터 없이 index에 이미 들어 있는 버전(레지스트리 관리자가 `nabi publish --index-dir`로 직접 적은 버전)도 같은 명령으로 메타데이터를 받습니다. 같은 트리를 `nabi publish`하면 설명, 라이선스, README 같은 나머지를 채우고, 다른 것은 바꾸지 않습니다.

## 더 읽을거리

- 사양: [`nabi search` / `publish` / `yank` / `login`](../../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-search--publish--yank--login--logout)
- 다음: [3. API를 안정적으로 유지하기](03-stable-api.html)
