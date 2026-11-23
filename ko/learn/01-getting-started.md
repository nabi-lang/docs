# 1. 시작하기

이 장에서는 nabi toolchain을 설치하고 첫 패키지를 만들어 실행합니다. 단일 파일 스크립트와 REPL도 씁니다. 이 장을 마치면 가이드의 나머지 예제를 직접 실행할 수 있습니다.

## 설치

### 준비물

`nabic`는 필요한 것을 모두 갖추고 있습니다. 자체 코드 생성기와 링커가 들어 있으므로 C 컴파일러나 링커를 따로 설치할 필요가 없습니다([9.2.18](../spec/refs-lang/009-tools/902-aot-pipeline.html#9218-자기-완결적-toolchain)).

- macOS: Xcode Command Line Tools (`xcode-select --install`). 마지막 링크 단계에 SDK가 필요합니다.
- Linux: 필요 없습니다. toolchain과 toolchain으로 빌드한 프로그램에는 glibc 2.34 이상(Ubuntu 22.04, Debian 12, RHEL 9 이후)이 필요합니다.
- Windows: 필요 없습니다.
- FreeBSD: 필요 없습니다. toolchain과 toolchain으로 빌드한 프로그램에는 FreeBSD 14 이상이 필요합니다.

릴리스 아카이브(archive)는 macOS arm64, Linux x86_64와 arm64(glibc와 정적 musl 둘 다), Windows x86_64와 arm64, FreeBSD x86_64용으로 빌드합니다. 다른 호스트용 릴리스 아카이브는 없습니다. 이런 호스트에서는 소스에서 Nabi를 빌드하세요.

### 설치 스크립트

설치 스크립트는 [GitHub releases](https://github.com/nabi-lang/nabi/releases)에서 플랫폼에 맞는 toolchain을 내려받아 릴리스의 `SHA256SUMS`로 검증하고 `~/.nabi`에 풉니다. `PATH`에 `minisign`이 있으면 먼저 `SHA256SUMS.minisig`를 릴리스 키로 검증합니다. 없으면 `SHA256SUMS`는 HTTPS로만 받아 오고, 스크립트가 서명을 직접 검증하는 방법을 알려 줍니다. 하나라도 맞지 않으면 아무것도 설치하지 않습니다.

```bash
curl -fsSL https://nabi.sh | sh
```

Windows에서는 PowerShell에서 다음을 실행합니다. `%USERPROFILE%\.nabi`에 설치하고, 그 아래 `bin`을 사용자 `PATH`에 추가합니다.

```powershell
powershell -ExecutionPolicy Bypass -c "irm https://nabi.sh/install.ps1 | iex"
```

`install.sh --version <v>`와 `install.ps1 -Version <v>`로 릴리스를 정확한 버전(`0.1.0`)으로 고릅니다. 적지 않으면 최신 릴리스를 설치합니다.

설치 위치는 `NABI_HOME` 환경 변수로 바꿀 수 있습니다(기본값 `~/.nabi`). 스크립트가 끝나면 다음 구조가 생깁니다.

```text
~/.nabi/
├── bin/                        # nabi, nabic, nabicc, nabils proxy
└── toolchains/
    └── 0.1.3-aarch64-apple-darwin/
        ├── bin/{nabi, nabic, nabicc, nabils}
        └── lib/                # runtime and standard library sources
```

방금 설치한 toolchain이 기본값이 됩니다. `~/.nabi/bin`의 네 실행 파일은 **프록시**입니다. 실제 일은 활성 toolchain에 있는 같은 이름의 실행 파일이 합니다. 그래서 여러 toolchain을 나란히 설치해 두고 하나를 골라 쓸 수 있습니다. rustup과 같은 방식입니다. 프록시는 언제나 설치된 toolchain 중 가장 새것에서 복사합니다. 가장 새 toolchain은 이전 toolchain들의 구조를 모두 알기 때문입니다. 그래서 이전 toolchain을 기본값으로 바꿔도 프록시는 그대로 둡니다.

스크립트는 `~/.nabi/bin`을 `PATH`에도 넣습니다. `~/.nabi/env`(와 `~/.nabi/env.fish`)를 만들고, `~/.zshenv`, `~/.bashrc`, `~/.profile` 같은 셸 시작 파일이 이 파일을 읽게 합니다. fish용으로는 `~/.config/fish/conf.d/nabi.fish`를 추가합니다. 바꾼 파일은 모두 이름을 출력합니다.

```text
 Downloading nabi 0.1.3 (aarch64-apple-darwin)
   Installed nabi 0.1.3 to ~/.nabi

    Modified ~/.zshenv (PATH)

A new shell finds nabi on PATH; to use it in this one, run:
  . "$HOME/.nabi/env"            (sh, bash, zsh)
  source "$HOME/.nabi/env.fish"  (fish)

Next steps:
  nabi --version
  nabi new hello && cd hello && nabi run
```

스크립트를 다시 실행해도 바뀌는 것은 없습니다. 셸 파일을 그대로 두려면 `--no-modify-path`(`install.ps1`은 `-NoModifyPath`)를 넘기거나 `NABI_NO_MODIFY_PATH=1`을 설정합니다. 그러면 스크립트는 `PATH`에 직접 추가할 줄을 출력합니다. toolchain에는 공식 패키지(`json`, `http`, `regex`, `table` 등)가 함께 들어 있으므로, 그것을 쓰는 코드는 네트워크 없이 빌드됩니다. 빼려면 `--no-packages`(`-NoPackages`, 또는 `NABI_NO_PACKAGES=1`)를 넘깁니다. `nabi`의 셸 자동 완성도 함께 설치됩니다(bash, zsh, fish, PowerShell). 다른 곳에 설치하려면 `nabi completions <shell>`로 자동 완성 스크립트를 출력합니다.

새 셸에서 버전을 확인합니다.

```bash
nabi --version
nabi toolchain list
```

`nabi --version -v`는 실행된 바이너리, 고른 toolchain과 그 이유, 함께 쓰는 `nabic`, 호스트까지 한 줄에 하나씩 `key: value` 형태로 알려 줍니다(`nabi version -v --format json`은 JSON 객체 하나로 출력합니다). 기대한 버전이 아닐 때 써 보세요.

```text
$ nabi --version -v
nabi 0.1.3 (<commit> <date>)
binary:    /Users/me/.nabi/toolchains/0.1.3-aarch64-apple-darwin/bin/nabi
toolchain: 0.1.3-aarch64-apple-darwin (the default)
nabic:     /Users/me/.nabi/toolchains/0.1.3-aarch64-apple-darwin/bin/nabic
host:      aarch64-apple-darwin
```

실행 파일 네 개가 설치됩니다.

| 실행 파일 | 하는 일 |
|------------|--------------|
| `nabic`     | 컴파일러. 빌드, 검사, 테스트와 벤치 하네스([9.1](../spec/refs-lang/009-tools/901-nabic-cli.html)) |
| `nabi`      | 패키지 / 빌드 프런트엔드와 도구. `nabi.nabion`, 의존성, 환경, 실행, 린트, 포맷, 문서, REPL, Jupyter([9.10](../spec/refs-lang/009-tools/910-nabi-frontend.html)) |
| `nabicc`    | C 컴파일러. `nabi cc`로도 실행합니다([9.13](../spec/refs-lang/009-tools/913-c-compiler.html), [14장](14-ffi.html#c-소스-nabi-cc)) |
| `nabils`    | 언어 서버(LSP) |

### toolchain 관리

`nabi toolchain`은 설치된 toolchain을 관리합니다. 활성 toolchain은 명령줄의 `+version`(`nabi +0.1 build`), `NABI_TOOLCHAIN` 환경 변수, 가장 가까운 `nabi.nabion`(또는 스크립트의 `//! nabion` 블록)의 `toolchain` 필드, 기본값 순서로 정해집니다. 고정 값으로는 시리즈(`"0.1"`, 설치된 0.1.x 릴리스 중 최신, 0.1 릴리스가 설치되지 않았으면 가장 새 릴리스 후보), 버전(`"0.1.0"`), 요구 조건(`"^0.1.0"`)을 쓸 수 있습니다. 명령 하나는 처음부터 끝까지 toolchain 하나로 실행됩니다. 어떤 toolchain의 `bin/nabi`를 전체 경로로 직접 실행해도 프로젝트가 고정한 toolchain에 명령을 넘기고, 각 toolchain의 `nabi`는 자기 `nabic`만 씁니다.

```bash
nabi toolchain list                     # installed toolchains
nabi toolchain default 0.1.1            # change the default
nabi toolchain install 0.1.1            # download 0.1.1 beside the default
nabi toolchain update                   # download the newest release and make it the default
nabi toolchain update 0.1.1             # exactly 0.1.1 instead
```

`nabi toolchain update`에는 시리즈가 아니라 정확한 버전을 적습니다. `nabi toolchain install`은 기본값을 바꾸지 않고 내려받기만 합니다(이미 가진 아카이브는 `--from <archive>`로 설치합니다). 내려받은 파일은 공개된 SHA-256으로 검증합니다. 어느 단계에서든 실패하면 이전 기본값이 그대로 남습니다. 이전 toolchain은 지우지 않습니다. 업데이트가 시리즈를 넘으면 안내가 붙습니다. 수정은 최신 시리즈에만 나오니 프로젝트를 그 시리즈로 옮기세요. 옮기기 전까지는 `toolchain: "0.1"`이 0.1을 고정하듯 이전 시리즈로 고정하거나 `nabi +0.1 ..`로 실행할 수 있습니다. 패키지의 `nabi.lock`에 고정된 버전을 올리는 `nabi update`와 헷갈리지 마세요([11장](11-modules-packages.html)).

같은 시리즈의 더 새 toolchain은 이전 toolchain이 빌드한 것을 빌드합니다. 1.0 아래에서 시리즈는 `0.y`입니다. 0.1.7은 0.1.3이 빌드한 것을 빌드하지만 0.2.0은 그렇지 않을 수 있습니다. 1.0부터는 `x`입니다. 1.9.0은 1.4.2가 빌드한 것을 빌드하지만 2.0.0은 그렇지 않을 수 있습니다. 패키지의 `nabi_version`도 이 규칙을 따릅니다. `"0.1.3"`은 nabi 0.1.3이나 그 뒤의 0.1.x를 뜻합니다.

패키지 관리자로 설치한 것처럼 다른 `nabi`가 `PATH`에서 `~/.nabi/bin`보다 앞에 있으면 새 기본값이 적용되지 않습니다. 그래서 `nabi toolchain update`와 `nabi toolchain default`가 경고합니다.

```text
nabi: warning: your shell runs /opt/homebrew/bin/nabi (nabi 0.1.3) before ~/.nabi/bin/nabi; put ~/.nabi/bin first on PATH or remove the other one
```

### 소스에서 빌드하기(선택)

toolchain 아카이브를 직접 빌드해 같은 스크립트로 설치할 수도 있습니다. Rust toolchain(`rustup`)이 필요합니다.

```bash
git clone https://github.com/nabi-lang/nabi && cd nabi
scripts/dist/native.sh                     # dist/nabi-<version>-<triple>.tar.gz
sh install.sh --archive dist/nabi-*.tar.gz  # into ~/.nabi
```

스크립트는 `targets/build.sh`로 타깃마다 런타임과 C 라이브러리도 빌드합니다. 여기에는 `curl`, `python3`, xz를 다루는 `tar`가 필요하고, 내려받은 파일은 `~/.cache/nabi`에 둡니다(`NABI_CACHE`로 옮길 수 있습니다). Linux에서는 `scripts/dist/gnu.sh`(podman 또는 docker)가 같은 아카이브를 컨테이너 안에서 빌드합니다. 그래서 결과물은 glibc 2.34 이상에서 돌아갑니다. 릴리스 아카이브도 이 방식으로 빌드합니다. 자세한 내용은 저장소의 README에 있습니다.

컴파일러를 개발하고 있다면 설치하지 않고 빌드 결과를 바로 써도 됩니다. `cargo build`는 실행 파일과 호스트의 런타임을 만듭니다. `targets/build.sh <triple>`은 타깃의 C 라이브러리와 런타임 파일을 빌드합니다. Linux 호스트는 프로그램을 링크하려면 자기 것이 필요하고, 크로스 타깃은 모두 이 파일이 필요합니다. macOS와 Windows 호스트에는 더 필요한 것이 없습니다.

```bash
cargo build --release
targets/build.sh "$(rustc -vV | sed -n 's/^host: //p')"
export PATH="$PWD/target/release:$PATH"
```

## 첫 패키지

`nabi new`는 패키지 디렉터리를 만들고, `target/`을 제외하는 `.gitignore`와 함께 git 저장소를 초기화합니다. 패키지 이름은 소문자, 숫자, `-`, `_`로 이루어지며 문자로 시작하고, 길이는 64자까지입니다. Windows가 장치 이름으로 쓰는 이름(`con`, `nul`, `com1` 등)은 그 이름의 파일을 만들 수 없으므로 거절합니다. `nabi`는 표준 라이브러리가 쓰는 이름이므로 패키지 이름으로 쓸 수 없습니다. 코드에서는 `-` 자리에 `_`를 씁니다(`nabi new my-lib`으로 만든 패키지는 `use my_lib::..`로 가져옵니다).

```bash
nabi new hello
cd hello
```

```text
     Created executable package (hello)
             cd hello && nabi run
```

두 번째 줄은 다음에 실행할 명령입니다. 파일은 두 개입니다. 매니페스트인 `nabi.nabion`에는 패키지 이름, 버전, 빌드되는 가장 오래된 nabi(`nabi_version`), 패키지를 만든 toolchain의 시리즈, 의존성이 들어 있습니다.

```nabion
Package { name: "hello", version: "0.1.0", nabi_version: "0.1.3", toolchain: "0.1", deps: {} }
```

소스는 `src/`에 있습니다. `src/main.nabi`는 인사를 출력하는 한 줄짜리 코드입니다.

```nabi
println("Hello, world!");
```

```output
Hello, world!
```

`nabi run`은 빌드하고 실행합니다. 결과물은 `target/debug/`에 생깁니다. 빌드는 표준 라이브러리 중 프로그램이 쓰는 부분만 컴파일하고 toolchain이 그것을 보관하므로, 이후 어떤 패키지를 빌드하든 그 캐시에서 가져옵니다.

```bash
nabi run
```

```text
   Compiling hello (.)
    Finished in 0.87s
     Running `target/debug/hello`
Hello, world!
```

`src/main.nabi`의 최상위 문은 위에서 아래로 실행되고, 이 문이 곧 프로그램입니다. 함수와 타입 선언은 최상위 문 사이 어디에 두어도 됩니다. 그 밖에 자주 쓰는 명령은 `nabi build`(빌드만), `nabi check`(타입 검사만), `nabi test`, `nabi fmt`입니다.

## 조금 더 큰 프로그램

`src/main.nabi`를 다음과 같이 바꾸고 `nabi run`을 다시 실행합니다.

```nabi
let names = ["ada", "grace", "linus"];
for name in names {
    greet(name);
}
println(f"{names.len()} people greeted");

fn greet(name: str) {
    println(f"Hello, {name}!");
}
```

```output
Hello, ada!
Hello, grace!
Hello, linus!
3 people greeted
```

`f"..."`는 f-string입니다. `{}` 안의 식을 평가해 그 자리에 넣습니다. `greet`는 호출하는 곳보다 아래에 선언되어 있지만 호출할 수 있습니다. 함수 같은 선언은 순서와 관계없이 보이기 때문입니다.

`println`은 값 여러 개를 받아 사이에 공백을 넣어 출력합니다. `sep:`와 `end:`는 값 사이와 끝에 넣을 문자열을 바꿉니다. 디버깅할 때는 `dbg(e)`가 식의 위치와 값을 stderr에 출력합니다.

```nabi
let n = 3;
println("total:", n, [1, 2]);
println(1, 2, 3, sep: ", ");
println(f"{n=}");
```

```output
total: 3 [1, 2]
1, 2, 3
n=3
```

최상위 문 대신 `fn main()`을 진입점으로 쓸 수도 있습니다. 둘 중 하나만 써야 합니다.

```nabi
fn main() {
    let total = add(2, 3);
    println(f"2 + 3 = {total}");
}

fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

```output
2 + 3 = 5
```

## 단일 파일 스크립트

간단히 실험할 때는 패키지가 필요 없습니다. `nabi run`에 파일 경로를 주면 그 파일 하나만 실행합니다. `nabi build hi.nabi`는 실행 파일 `hi`를 스크립트 옆에 만듭니다.

```bash
echo 'println("hi");' > hi.nabi
nabi run hi.nabi
```

```text
   Compiling hi (.)
    Finished in 0.05s
     Running hi.nabi
hi
```

스크립트에서도 의존성을 쓸 수 있습니다. `nabi add json hi.nabi`를 실행하면 파일 맨 위의 `//!` 주석에 의존성이 적힙니다([인라인 매니페스트](../spec/refs-lang/009-tools/910-nabi-frontend.html#9107-단일-파일-스크립트의-인라인-매니페스트), [11장](11-modules-packages.html#단일-파일-스크립트와-환경) 참고).

## REPL

인자 없이 `nabi`를 실행하면 REPL이 열립니다. 식을 입력하면 값을 출력합니다. 식 끝에 `?`를 붙이면 `Debug` 형태로 출력합니다. `Display`가 없는 값은 타입만 출력하고, `let`은 출력 없이 이름만 정의합니다.

```text
$ nabi
Nabi 0.1.3 (<commit> <date>) on aarch64-apple-darwin
Type :help for help, :exit to quit.
>>> let xs = [1, 2, 3]
>>> xs.len() * 2
6
>>> xs?
[1, 2, 3]
>>> :type xs
List<i64>
>>> :exit
```

`:type expr`는 식을 실행하지 않고 타입만 알려 줍니다. 가이드를 읽다가 타입이 궁금하면 REPL에서 확인해 보세요. 셀의 마지막 `let`에는 `;`이 필요 없습니다. 나머지는 [16장](16-tools.html#repl과-jupyter)에서 다룹니다([JIT 셀](../spec/refs-lang/009-tools/907-jit-cells.html)).

## 정리

- `curl -fsSL https://nabi.sh | sh`(Windows는 `install.ps1`)로 `~/.nabi`에 설치합니다. 스크립트는 `~/.nabi/bin`을 `PATH`에 넣고 셸 자동 완성을 설치합니다.
- `nabi new name`은 `nabi.nabion`과 `src/main.nabi`를 만들고, `nabi run`은 빌드한 뒤 실행합니다.
- `src/main.nabi`의 최상위 문이 프로그램입니다. `fn main()`을 쓸 수도 있습니다.
- `nabi run file.nabi`는 스크립트를 실행하고, 인자 없는 `nabi`는 REPL을 시작합니다.

## 더 읽을거리

- [설치 레이아웃](../spec/refs-lang/009-tools/902-aot-pipeline.html#9219-설치-레이아웃), [toolchain 고정](../spec/refs-lang/009-tools/910-nabi-frontend.html#9106-toolchain)
- [`nabi` 프런트엔드](../spec/refs-lang/009-tools/910-nabi-frontend.html), [`nabic` CLI](../spec/refs-lang/009-tools/901-nabic-cli.html)
- [진입점](../spec/refs-lang/004-functions/416-entry-point.html)
- [JIT 셀과 REPL](../spec/refs-lang/009-tools/907-jit-cells.html)
- 다음 장: [2. 값과 타입](02-values-and-types.html)
