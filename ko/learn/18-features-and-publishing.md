# 18. Feature와 공개

[11장](11-modules-packages.html)에서는 패키지를 만들고 의존성을 추가했습니다. 이 장은 거기서 더 나아갑니다. 라이브러리 사용자가 그 일부를 켜고 끌 수 있게 하는 **feature**, Nabi 프로젝트가 관리하는 패키지, 그리고 자기 라이브러리를 패키지 레지스트리 [packages.nabi-lang.org](https://packages.nabi-lang.org)에 공개하는 법을 다룹니다.

## feature 정의하기

feature는 `nabi.nabion`의 `features` 딕셔너리에 있는 이름입니다. 이름마다 그 feature가 켜는 것의 리스트가 붙습니다. 코드에서 켜진 feature는 `feature = "<name>"` 조건입니다([17장](17-conditional-compilation.html)). 여기서 `geom` 라이브러리에는 기본으로 켜진 `circle` feature와 선택 의존성을 끌어들이는 `pretty` feature가 있습니다.

```nabion
Package {
    name: "geom",
    version: "0.1.0",
    deps: {
        "fancy": Dep::path("../fancy", optional: true),
    },
    features: {
        "default": ["circle"],
        "circle": [],
        "pretty": ["dep:fancy"],
    },
}
```

```nabi norun
// geom/src/lib.nabi
pub fn shapes() -> List<str> {
    let out: List<str> = ["square"];
    #[cfg(feature = "circle")]
    out.push("circle");
    out
}

#[cfg(feature = "pretty")]
pub fn show(s: str) -> str { fancy::frame(s) }

#[cfg(not(feature = "pretty"))]
pub fn show(s: str) -> str { s }
```

feature 리스트의 각 항목은 다음 중 하나입니다.

| 항목        | 켜는 것                                                             |
|-------------|---------------------------------------------------------------------|
| `"circle"`  | 이 패키지의 다른 feature                                            |
| `"dep:fancy"` | 선택 의존성 `fancy`                                               |
| `"http/tls"` | 의존성 `http`의 feature `tls`(`http`가 선택 의존성이면 `http` 자체도) |
| `"http?/tls"` | `http`의 feature `tls`. 다른 무언가가 이미 `http`를 켰을 때만   |

- `default`는 사용자가 끄지 않는 한 켜져 있는 보통의 feature입니다.
- 선택 의존성(`optional: true`)은 켜진 feature가 `dep:`이나 `<dep>/<f>`로 가리킬 때만 빌드에 들어갑니다. 자기 이름의 feature가 저절로 생기지는 않습니다. 어떤 feature도 켜지 않는 선택 의존성은 에러입니다. `nabi add fancy --path ../fancy --optional`은 의존성과 `"fancy": ["dep:fancy"]` feature를 함께 씁니다.
- 없는 feature나 의존성을 가리키는 것, 선택 의존성이 아닌 의존성에 `dep:`을 쓰는 것, 순환은 에러입니다. 이 에러는 `nabi.nabion`의 줄과 열로 보고됩니다.

feature 이름은 무엇을 켜는지 말해 주는 소문자 단어(`http1`, `tls`, `pretty`)로 짓고, `with-x`나 `no-x`는 쓰지 않습니다. feature는 언제나 더하기만 하기 때문입니다.

## feature 켜기

빌드하는 패키지는 자기 `default` feature에서 시작합니다. `--features`는 자기 feature(`fancy`)나 직접 의존성의 feature(`geom/pretty`)를 더합니다. `--no-default-features`와 `--all-features`는 이름 그대로 동작합니다. `run`, `build`, `test`, `check`와 그 밖의 빌드 명령에서 쓸 수 있습니다.

```text
$ nabi run
square
circle
$ nabi run --features geom/pretty
*** square ***
*** circle ***
$ nabi run --no-default-features
square
circle
```

마지막 명령도 `circle`을 출력합니다. `--no-default-features`는 `geom`이 아니라 빌드하는 패키지(`app`)의 기본값을 끄기 때문입니다. 의존성의 feature는 그 의존성을 적은 자리에서 고릅니다.

```nabion
Package {
    name: "app",
    version: "0.1.0",
    deps: {
        "geom": Dep::path("../geom", default_features: false, features: ["pretty"]),
    },
    features: {
        "fancy": ["geom/pretty"],
    },
}
```

feature는 **합쳐집니다**(unified). 빌드 전체에서 패키지마다 feature 집합은 하나뿐이고, 이 집합은 그 패키지에 의존하는 모든 패키지가 요구한 것의 합집합입니다. 그래서 두 라이브러리가 모두 `geom`을 쓰는데 하나는 `pretty`를 켜고 하나는 켜지 않으면 둘 다 `pretty`를 받습니다. feature가 API를 더하기만 하고 기존 코드의 동작을 바꾸면 안 되는 이유가 이것입니다. `nabi deps`는 결과를 `--extern-feature` 줄로 보여 줍니다.

lockfile은 선택 의존성까지 그래프의 모든 패키지를 고정하므로 feature를 바꿔도 `nabi.lock`은 바뀌지 않습니다.

## Nabi 프로젝트의 패키지

Nabi 프로젝트는 여러 라이브러리를 표준 라이브러리 밖의 독립된 패키지로 공개합니다. 데이터 형식(`json`, `toml`, `csv`), `http`, `markdown`, `regex`, `spatial`, `nabi-syntax`, 그리고 [22장](22-science.html)의 과학 패키지입니다. 모두 순수 Nabi로 짰습니다. 레지스트리에서는 **Official** 표시가 붙고, [전체 목록](https://packages.nabi-lang.org/packages?q=official:yes)이 있으며, 각 패키지의 문서는 쓰는 법으로 시작합니다.

`json`과 `toml`은 std의 [`nabion`](15-nabion.html) 모듈과 같은 모양을 따릅니다. `parse` / `to_str` / `from_str`, 여기에 `to_value` / `from_value`, 그리고 자기 타입을 위한 derive입니다.

```nabi norun
#[derive(json::Json)]
struct Server { host: str; port: u16 = 80; }

let s: Server = try json::from_str("{\"host\": \"localhost\"}");
println(json::to_str(s));      // {"host":"localhost","port":80}
```

패키지의 derive는 패키지 이름을 붙여 `#[derive(json::Json)]`으로 쓰거나, `use json::Json;` 뒤에 이름만 씁니다. derive가 펼쳐 낸 코드는 패키지 자신의 코드이므로 패키지 내부의 도우미를 부를 수 있지만, 여러분의 코드는 부를 수 없습니다.

다른 패키지와 똑같이 추가합니다. 버전 요구 사항을 적지 않으면 `nabi add`는 `^`과 최신 버전을 씁니다.

```bash
nabi add json
nabi add http --no-default-features --features http1,tls
```

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "json": Dep::version("^0.1.0"),
    "http": Dep::version("^0.1.0", default_features: false, features: ["http1", "tls"]),
} }
```

스크립트에서는 파일 이름을 마지막에 적습니다. `nabi add json app.nabi`는 의존성을 스크립트의 `//!` 블록에 씁니다([11장](11-modules-packages.html#단일-파일-스크립트와-환경)).

`http`를 쓰면 HTTPS 요청은 몇 줄이면 됩니다. 패키지가 TCP 연결을 열고 OS 신뢰 저장소를 기준으로 [`std::net::tls`](19-crypto-and-tls.html)의 TLS 1.3 핸드셰이크를 실행합니다. ALPN으로 HTTP/2를 제안하고, 안 되면 HTTP/1.1로 물러섭니다.

```nabi norun
let r = try await http::get("https://example.com/");
println(f"{r.status} {r.version:?} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

## 패키지 공개하기

공개는 레지스트리 index에 줄 하나를 더하는 일입니다. 그 줄은 git 저장소의 커밋 하나를 가리키고, 레지스트리가 그 커밋의 아카이브를 보관하므로 그 버전은 저장소가 없어져도 남습니다. 요약하면 다음과 같습니다.

```bash
nabi login                # a token from packages.nabi-lang.org/me
nabi publish --dry-run    # print the index entry; nothing leaves your machine
nabi publish
nabi yank textwrap@0.1.0 --message "wraps CJK text wrong; use 0.1.1"
```

공개는 가이드의 따로 된 부분인 [패키지 공개](packages/index.html)에서 다룹니다. 패키지에 필요한 것, [`nabi login`과 `nabi publish`](packages/01-publishing.html), GitHub Actions에서 하는 [신뢰할 수 있는 공개](packages/02-trusted-publishing.html), `#[deprecated]`, `#[non_exhaustive]`, `#[sealed]`로 [API를 안정적으로 유지하기](packages/03-stable-api.html)가 있습니다.

> [!IMPORTANT]
> 지금은 레지스트리 관리자만 패키지를 공개할 수 있습니다. 누구나 로그인해 패키지를 쓸 수 있지만, `nabi login`은 아직 공개 권한이 없는 계정을 거절합니다.

## 요약

- `nabi.nabion`의 `features`는 feature 이름마다 켤 항목을 적습니다. 항목은 다른 feature, `dep:<d>`, `<d>/<f>`, `<d>?/<f>`입니다. `default`는 끄지 않는 한 켜져 있습니다.
- 선택 의존성(`optional: true`)은 feature가 켤 때만 빌드됩니다.
- feature는 `--features`, `--no-default-features`, `--all-features`로, 의존성마다는 `features:` / `default_features: false`로 고릅니다. feature는 빌드 전체에서 합쳐지므로 더하기만 해야 합니다.
- 코드는 `#[cfg(feature = "..")]`와 `cfg!(feature = "..")`로 feature를 검사합니다.
- Nabi 프로젝트의 패키지는 레지스트리에서 Official 표시가 붙습니다.
- 공개는 `nabi login`(레지스트리 토큰), `nabi publish --dry-run`, `nabi publish` 순서로 하고, 철회는 `nabi yank name@version --message <이유>`로 합니다. API를 없앨 때는 먼저 `#[deprecated]`를 붙입니다. `#[non_exhaustive]`와 `#[sealed]`는 변형, 필드, 구현을 더할 여지를 남깁니다. 지금은 관리자만 공개할 수 있습니다.

## 더 읽을거리

- 사양: [9.10.4c feature](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104c-feature), [9.10.4a 레지스트리 패키지](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104a-레지스트리-패키지), [`nabi search` / `publish` / `yank` / `login`](../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-search--publish--yank--login--logout), [`nabi.nabion`](../spec/refs-lang/009-tools/910-nabi-frontend.html#9103-nabinabion--nabilock)
- 패키지 레지스트리 [packages.nabi-lang.org](https://packages.nabi-lang.org)
- 다음 장: [19. 암호와 TLS](19-crypto-and-tls.html)
