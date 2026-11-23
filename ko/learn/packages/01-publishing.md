# 1. 패키지 공개하기

공개는 레지스트리 index에 줄 하나를 더하는 일입니다. 그 줄은 내 git 저장소의 커밋 하나를 가리킵니다([레지스트리의 동작 방식](index.html#레지스트리의-동작-방식)). 이 페이지는 매니페스트에서 시작해 버전을 공개하기까지, 그리고 공개한 버전을 철회하는 방법을 다룹니다.

## 패키지에 필요한 것

- 저장소 루트에 `name`, `version`, `nabi_version`을 적은 `nabi.nabion`.
- `src/lib.nabi`(라이브러리), `src/main.nabi`(`nabi install`로 설치하는 프로그램) 중 하나나 둘 다, 그리고 readme.
- 원격 `origin`이 있는 깨끗한 git 작업 트리. `origin`의 url은 공개 호스트의 `https://`여야 하고, 커밋은 그 저장소의 브랜치나 태그에 push되어 있어야 합니다(fork에만 있는 커밋은 인정되지 않습니다).
- 레지스트리에서 오는 일반 의존성. git 의존성은 거부됩니다. 경로 의존성도 거부되지만, 경로 대신 공개할 버전을 함께 적으면 됩니다: `Dep::path("../geom", version: "^0.1")`(`geom`을 먼저 공개합니다). `version:`이 없는 경로와 git의 `dev_deps`는 빠집니다.
- 매니페스트에 `publish: false`가 없을 것.

`nabi_version`에는 패키지를 빌드해 본 가장 오래된 nabi를 적으세요. `nabi publish`에 필요하고, 해소는 사용자마다 그 toolchain이 빌드하는 가장 새 버전을 줍니다(`"0.1.3"`은 nabi 0.1.3과 그 뒤의 0.1.x를 허용합니다. [1장](../01-getting-started.html#toolchain-관리)). 패키지 페이지에는 이 값이 "Requires"로 나옵니다. `nabi new`는 패키지를 만든 버전을 적습니다.

`description`, `license`, `repository`를 채우세요. 이 값이 `nabi search`와 웹사이트에 나타납니다. 패키지 페이지에 나오는 나머지 필드는 Cargo처럼 선택입니다.

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

`keywords`와 `categories`는 각각 5개까지 적을 수 있습니다. `include` / `exclude`는 패키지에 담을 파일을 고릅니다. `license_file`, `readme`, `links`, `default_run`, 자유 형식의 `metadata` 테이블도 있습니다. `nabi`가 모르는 필드는 에러가 아니라 경고이고, 레지스트리에는 그대로 보내집니다.

## `nabi login`

[packages.nabi-lang.org/me](https://packages.nabi-lang.org/me)에서 GitHub로 로그인하고 거기서 계정 토큰을 만듭니다. 그런 다음 `nabi`에 넘겨줍니다.

```text
$ nabi login
a token from https://packages.nabi-lang.org/me (not shown; Ctrl-C to cancel):
   Logged in as your-name at https://packages.nabi-lang.org (token in ~/.nabi/tokens.nabion)
```

인자가 없으면 토큰을 화면에 보이지 않게 표준 입력에서 읽습니다. `nabi login`은 레지스트리에 토큰이 누구의 것인지 묻고, 레지스트리가 거부하는 토큰은 아무것도 저장하지 않고 거절합니다. 올바른 토큰은 `~/.nabi/tokens.nabion`(본인만 읽을 수 있음)에 그 토큰을 만든 index별로 저장되고, 그 index에만 보내집니다. 다른 index의 토큰은 `nabi login --registry <url>`로 저장합니다. `nabi logout`은 그 토큰을 지우고, `nabi logout --all`은 모든 index의 토큰을 지웁니다. CI에서는 대신 `NABI_REGISTRY_TOKEN`을 설정합니다. 이 토큰은 packages.nabi-lang.org에만 보내집니다. GitHub Actions에서는 [신뢰할 수 있는 공개](02-trusted-publishing.html)를 쓰면 저장해 둔 토큰이 아예 필요 없습니다.

## `nabi publish`

`nabi publish --dry-run`은 더할 index 항목을 출력할 뿐, 아무것도 기계 밖으로 보내지 않습니다. `nabi publish`는 `cargo publish`처럼 바로 공개합니다.

```text
$ nabi publish --dry-run
Package { name: "textwrap", version: "0.1.0", deps: [], source: Dep::git("https://github.com/you/textwrap.git", rev: "8e2afb20.."), tree: "823a3592..", yanked: false, nabi_version: "0.1.0" }
note: dry run; `nabi publish` sends it to the registry API of https://github.com/nabi-lang/packages-index (needs `nabi login`), which adds it as te/xt/textwrap
```

항목에는 의존성 해소에 필요한 것만 들어갑니다. HEAD 커밋과 그 트리 해시, 의존성, `nabi_version`입니다. 그러니 그 커밋을 먼저 push하세요. 설명, 라이선스 같은 나머지 메타데이터는 항목과 함께 레지스트리로 가고, 레지스트리는 버전마다 누가 공개했는지와 함께 이를 보여 줍니다. `nabi publish`는 토큰과 함께 둘을 보냅니다(`Published textwrap 0.1.0 to https://packages.nabi-lang.org`). index에 항목이 들어가면 그 버전이 공개됩니다. 한 버전은 두 번 공개할 수 없습니다.

실행 파일이 있는 패키지는 목록과 검색 결과에 **bin** 표시가 붙고, 패키지 페이지에 `nabi install <name>`이 나옵니다(라이브러리도 있으면 `nabi install --bin <name>`).

## 이름과 소유자

이름은 64자까지입니다. 기존 패키지와 대소문자나 `-` / `_`만 다른 이름은 이미 쓰인 것으로 봅니다. 표준 라이브러리의 계층 `std`, `core`, `bare`는 Nabi 프로젝트가 소유한 index 항목입니다. 이들은 toolchain과 함께 오고, 그 이름으로는 아무것도 공개되지 않습니다. toolchain의 이름(`nabi`, `nabic`, ...)은 예약되어 있습니다. index에 있는 이름과 한 글자만 다르거나, 비슷하게 생긴 글자를 맞추면 같아지는 새 이름(`json`에 대한 `js0n`)은 거절됩니다. 그래도 그 이름이 필요할 때 요청하는 방법은 [이용 정책](https://packages.nabi-lang.org/policy)에 있습니다.

패키지의 첫 버전을 공개한 사람이 그 패키지를 소유합니다. 소유자는 [계정 페이지](https://packages.nabi-lang.org/me)에서 다른 계정을 초대해 소유권을 나눌 수 있고, 초대받은 계정은 수락하면 소유자가 됩니다.

## API 문서

레지스트리는 버전마다 그 커밋의 소스로 API 문서를 만듭니다. 소스는 읽기만 하고 빌드하거나 실행하지 않습니다. 문서는 docs.rs가 crates.io에 하듯 문서 사이트가 보여 줍니다. `docs.nabi-lang.org/packages/textwrap/0.1.0`에 있고, 최신 버전의 문서는 `/packages/textwrap/latest`입니다(실제 예는 [json의 문서](https://docs.nabi-lang.org/packages/json/latest)). 레지스트리의 패키지 페이지는 Docs 탭에서 그곳으로 연결합니다. 페이지 구성은 `nabi doc`과 같습니다. 루트 모듈은 그 주소에, 다른 public 모듈은 각자의 페이지(`/packages/textwrap/0.1.0/<module>`)에 있고, 앵커도 같습니다(`#fn.wrap`). 독자가 가장 먼저 보는 것은 루트 모듈의 `//!` 글이므로, 패키지 소개와 튜토리얼은 거기에 씁니다. 그 안의 코드 블록은 `nabi test`가 실행하는 문서 테스트이고, `# `로 시작하는 줄은 페이지에 나오지 않는 준비 코드입니다. 모듈의 문서는 그 파일 맨 앞의 `//!` 줄입니다. 일반 `//` 주석은 나오지 않습니다. private 모듈의 항목을 `pub use`로 다시 내보내면, 그 항목은 다시 내보낸 곳에 선언된 것처럼 보입니다. 실행 파일만 있는 패키지는 API가 없으므로 문서도 없습니다. 문서를 만들지 못해도 버전은 공개됩니다. `nabi publish`가 이유를 경고로 출력하고, 문서 페이지에도 그 이유가 나옵니다.

## `nabi yank`

공개한 버전은 지워지지 않습니다. 심각한 버그가 있는 버전은 철회(yank)할 수 있습니다. 새로 의존성을 해소할 때는 그 버전을 건너뜁니다. 하지만 `nabi.lock`이 이미 그 버전을 고정한 프로젝트는 계속 빌드되므로 철회가 누군가의 빌드를 깨뜨리는 일은 없습니다. `--undo`로 되돌립니다.

```bash
nabi yank textwrap@0.1.0 --message "wraps CJK text wrong; use 0.1.1"
nabi yank textwrap@0.1.0 --undo
```

`--message`로 이유를 적으면 레지스트리가 패키지 페이지에 보여 줍니다.

## 더 읽을거리

- 사양: [9.10.4a 레지스트리 패키지](../../spec/refs-lang/009-tools/910-nabi-frontend.html#9104a-레지스트리-패키지), [`nabi search` / `publish` / `yank` / `login`](../../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-search--publish--yank--login--logout), [`nabi.nabion`](../../spec/refs-lang/009-tools/910-nabi-frontend.html#9103-nabinabion--nabilock)
- 다음: [2. 신뢰할 수 있는 공개](02-trusted-publishing.html)
