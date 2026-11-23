# 11. 모듈과 패키지

프로그램이 커지면 코드를 나누고 다른 사람이 작성한 라이브러리를 써야 합니다. Nabi에서 코드를 나누는 단위는 **모듈**(module)이고, 빌드하고 배포하는 단위는 **패키지**(package)입니다.

## `mod`와 경로

`mod name { .. }`는 그 자리에 모듈을 정의합니다. 모듈 안의 항목은 `module::item` 경로로 가리킵니다. 모듈 안에 다시 모듈을 둘 수 있습니다.

```nabi
mod geom {
    pub struct Point { pub x: f64; pub y: f64; }

    pub fn origin() -> Point { Point { x: 0.0, y: 0.0 } }

    pub mod dist {
        use sup::Point;

        pub fn manhattan(a: Point, b: Point) -> f64 {
            abs(a.x - b.x) + abs(a.y - b.y)
        }

        fn abs(v: f64) -> f64 { if v < 0.0 { -v } else { v } }
    }
}

let p = geom::Point { x: 3.0, y: -4.0 };
println(f"{geom::dist::manhattan(geom::origin(), p)}");
```

```output
7
```

경로는 정해진 단어로만 시작할 수 있습니다.

| 시작     | 뜻                                                           |
|----------|--------------------------------------------------------------|
| `pkg::`  | 현재 패키지의 루트부터                                       |
| `sup::`  | 부모 모듈부터(더 위로 가려면 `sup::sup::`)                  |
| `std::`, `core::`, `bare::` | 표준 라이브러리의 계층 |
| `::std::` | 지역 `std`가 있어도 표준 라이브러리 |
| 이름     | 현재 모듈의 항목이나 자식 모듈, 또는 `use`로 들여온 이름    |

이름 찾기는 부모 모듈을 자동으로 뒤지지 않습니다. 그래서 `dist` 안에서 `Point`를 쓰려면 `use sup::Point;`로 명시적으로 들여와야 합니다. 덕분에 이름이 어디서 왔는지 언제나 파일 안에서 알 수 있습니다.

`std`, `core`, `bare`는 일반 이름입니다. 직접 만든 모듈이나 의존성을 `std`라고 부를 수 있습니다. 그 모듈이 있는 곳에서 `std::`는 내 것을 가리키고, `::std::`는 여전히 표준 라이브러리를 가리킵니다.

```nabi
mod std { pub fn hello() -> str { "mine" } }

fn main() {
    println(std::hello());                       // mine
    println(::std::path::Path::empty().is_relative());  // 라이브러리의 것
}
```

## `use`로 들여오기

긴 경로를 매번 쓰는 대신 `use`로 이름을 들여옵니다. `{..}`로 여러 개를 들여오고, `as`로 이름을 바꾸고, `*`로 모듈의 공개 항목을 모두 들여올 수 있습니다.

```nabi
use report::show;
use shapes::*;

mod shapes {
    pub fn square(x: i32) -> i32 { x * x }
    pub fn cube(x: i32) -> i32 { x * x * x }
    pub const UNIT: i32 = 1;
}

mod report {
    use pkg::shapes::{square, cube as pow3};

    pub fn show(x: i32) {
        println(f"{x}: {square(x)} {pow3(x)}");
    }
}

show(3);
println(f"{cube(UNIT + 1)}");
```

```output
3: 9 27
8
```

이름을 지정해 들여온 이름은 같은 모듈에서 다시 정의할 수 없습니다. 정의가 아무 말 없이 가져온 이름을 가리기 때문입니다. glob(`use shapes::*;`)으로 들여온 이름은 모듈 자신의 항목에 자리를 내줍니다.

```nabi norun
use shapes::square;
fn square(x: f64) -> f64 { x * x }
```

```text check=checks/11-import-clash.nabi
error[E0032]: `square` is both imported from `shapes::square` and defined in this module
  --> main.nabi:1:5
   |
 1 | use shapes::square;
   |     ^^^^^^ imported here
 2 | fn square(x: f64) -> f64 { x * x }
   | -- defined here, which would hide the import
   |
   = help: import it under another name, `use shapes::square as shapes_square;`, or remove this `use` if the module does not need it
```

지금까지 `use` 없이 써 온 `List`, `Dict`, `println` 같은 이름은 *프렐류드*(prelude)에서 옵니다. 모든 파일은 암묵적인 `use core::prelude::*;`로 시작합니다. 프렐류드는 정해진 짧은 목록입니다. 자주 쓰는 타입(`List`, `Dict`, `Result`, `Duration`, `Mutex` 등), `impl` 머리와 바운드에 쓰는 트레잇(`Clone`, `Hash`, `Display`, `Iterator`, `From` 등), 출력과 panic 함수, 그리고 `core`의 최상위 모듈(`task`, `par`, `ops`, `num`, `sync`, `iter` 등)이 들어 있습니다. `core`의 나머지는 이 모듈들을 거쳐 부르거나 가져옵니다.

```nabi norun
task::sleep(Duration::ms(1));          // the module `task` is in the prelude
impl ops::Add for V { .. }             // operator traits live in `ops`
use core::sync::Frozen;           // or import the name
```

직접 정의한 이름은 언제나 프렐류드의 이름보다 우선합니다. enum 변형은 프렐류드에 없습니다. 기대하는 타입이 enum을 정해 주는 자리에서는 변형 이름만 씁니다([6장](06-structs-enums-traits.html)). 스코프에 없는 이름을 쓰면 에러 메시지가 써야 할 경로와 그 이름을 가져오는 `use`를 알려 줍니다. 파일, 네트워크, 스레드 같은 `std`의 모듈은 `use std::fs;`처럼 명시적으로 들여옵니다.

트레잇도 같습니다. 트레잇의 메서드는 그 트레잇이 스코프에 있을 때만 찾으므로, 다른 모듈이나 패키지의 트레잇은 `use`(`use zoo::Animal;`)해야 `x.name()`이 해소됩니다. 표준 라이브러리 트레잇(`core`, `std`)의 메서드는 `use` 없이 어디서나 동작합니다.

매크로, derive, 메타 속성은 그것을 정의한 패키지에 속합니다. 자기 패키지와 표준 라이브러리의 것은 이름만 씁니다(`log!(..)`, `#[derive(Inspect)]`). 다른 패키지의 것은 패키지 이름을 붙여 `json::log!(..)`, `#[derive(json::Json)]`처럼 쓰거나, `use json::log!;`로 한 번 들여옵니다. 다른 패키지의 매크로를 이름만 쓰면 에러이고, help가 두 형태를 알려 줍니다.

## 가시성

기본적으로 항목은 자기 모듈 안에서만 보입니다. 밖에서 보이게 하려면 가시성 키워드를 붙입니다.

| 키워드     | 보이는 곳                               |
|------------|-----------------------------------------|
| (없음)     | 같은 모듈                               |
| `pub(sub)` | 같은 모듈과 그 하위 모듈                |
| `pub(pkg)` | 같은 패키지 어디서나                    |
| `pub`      | 다른 패키지에서도                       |

struct 필드도 하나씩 공개해야 합니다. 그래서 위의 `Point`는 `pub x: f64`처럼 필드마다 `pub`을 붙였습니다. 비공개 항목을 밖에서 쓰면 컴파일러가 어디에 `pub`을 붙여야 하는지 알려 줍니다.

```nabi
mod bank {
    pub struct Account { pub owner: str; balance: i64; }

    pub fn open(owner: str) -> Account { Account { owner: owner, balance: 0 } }

    pub fn deposit(a: Account, amount: i64) -> Account {
        Account { owner: a.owner, balance: a.balance + check(amount) }
    }

    pub fn balance(a: Account) -> i64 { a.balance }

    fn check(amount: i64) -> i64 { if amount < 0 { 0 } else { amount } }
}

let a = bank::deposit(bank::open("kim"), 500);
println(f"{a.owner}: {bank::balance(a)}");
```

```output
kim: 500
```

`balance` 필드와 `check` 함수는 `bank` 밖에서 보이지 않으므로 잔액은 `bank`의 함수로만 바꿀 수 있습니다. 같은 이유로 `bank` 밖의 코드는 `bank::Account { owner: "kim" }`라고 쓸 수 없습니다. struct 리터럴은 기본값이 있는 비공개 필드만 생략할 수 있기 때문입니다. 그래서 계좌는 `bank::open`이 만듭니다.

표준 라이브러리의 `pub` 항목 가운데 몇 개에는 `#[internal]`이 붙어 있습니다. 툴체인의 패키지끼리만 나눠 쓰는 항목이어서, 다른 패키지에서 쓰면 "internal to the Nabi toolchain" 에러가 납니다. API 문서와 편집기에도 나오지 않습니다.

## 파일로 나누기

실제 패키지에서는 모듈을 파일로 나눕니다. `mod shapes;`처럼 본체 없이 모듈을 선언하면 같은 디렉터리의 `shapes.nabi`가 그 모듈의 본체가 됩니다. 하위 모듈이 있으면 `shapes.nabi` 옆에 같은 이름의 `shapes/` 디렉터리를 둡니다. `mod.nabi` 같은 특별한 파일 이름은 없습니다.

```text
geom/
├── nabi.nabion
└── src/
    ├── lib.nabi          // mod shapes;
    ├── shapes.nabi       // shapes module body, declares mod circle;
    └── shapes/
        └── circle.nabi   // shapes::circle
```

패키지의 소스는 `src/`에 있습니다. 패키지 루트 모듈은 실행 파일이면 `src/main.nabi`, 라이브러리면 `src/lib.nabi`입니다. 실행되는 최상위 문장은 `src/main.nabi`에만 올 수 있고, 다른 모듈에는 항목만 들어갑니다.

## 패키지와 `nabi.nabion`

`nabi new`는 패키지를 만듭니다. 실행 파일이면 `src/main.nabi`를, `--lib`을 주면 예제 함수와 `#[test]` 하나가 든 `src/lib.nabi`를 만듭니다. 매니페스트 `nabi.nabion`은 [NABION](../spec/refs-standard-libraries/300-std/308-nabion.html) 형식입니다. 이름과 버전 말고도 `nabi_version`을 적어 둡니다. 패키지를 만든 nabi, 곧 빌드된다고 알려진 가장 오래된 nabi이고, 공개한 패키지의 사용자에게 필요합니다([18장](18-features-and-publishing.html)). 패키지를 만든 toolchain의 시리즈인 `toolchain: "0.1"`도 적습니다. 그래서 나중에 다른 시리즈의 toolchain이 이 패키지를 뜻하지 않게 빌드하는 일이 없습니다([1장](01-getting-started.html#toolchain-관리)). 표준 라이브러리는 의존성이 아닙니다. `std`, `core`, `bare`는 toolchain과 함께 오고, `deps`에 `"std": ..` 줄을 두면 그 줄을 지우라는 에러입니다.

```bash
nabi new --lib geom
nabi new app
```

```text
     Created library package (geom)
             cd geom && nabi test
     Created executable package (app)
             cd app && nabi run
```

`geom`에 모듈을 두 개 만듭니다. 하위 모듈의 항목을 패키지 루트에서 `pub use`로 다시 내보내면(re-export) 라이브러리 사용자는 내부 구조를 알 필요가 없습니다.

```nabi norun
// geom/src/lib.nabi
mod shapes;

pub use shapes::Circle;

pub(pkg) const PI: f64 = 3.14159;

pub fn area_sum(cs: List<Circle>) -> f64 {
    cs.map(|c| c.area()).sum()
}
```

```nabi norun
// geom/src/shapes.nabi
pub struct Circle { pub r: f64; }

impl Circle {
    pub fn area(self) -> f64 { pkg::PI * self.r * self.r }
}
```

`app`에서 `nabi add`로 의존성을 추가하면 `nabi.nabion`의 `deps`에 들어갑니다. 의존 패키지의 이름이 최상위 경로가 됩니다(`-`는 `_`로 바뀝니다).

```bash
cd app
nabi add geom --path ../geom
```

```text
       Added geom v0.1.0
```

```nabion
Package {
    name: "app",
    version: "0.1.0",
    nabi_version: "0.1.0",
    toolchain: "0.1",
    deps: { "geom": Dep::path("../geom") },
}
```

```nabi norun
// app/src/main.nabi
use geom::{Circle, area_sum};

println(f"{area_sum([Circle { r: 1.0 }, Circle { r: 2.0 }])}");
```

```text
$ nabi run
   Compiling geom (../geom)
   Compiling app (.)
    Finished in 0.22s
     Running `target/debug/app`
15.70795
```

`PI`는 `pub(pkg)`이므로 `geom` 안에서는 어디서나 쓸 수 있지만 `app`에서는 보이지 않습니다. git 저장소는 `nabi add parser --git <url> --tag v1.2`처럼 추가합니다. git과 레지스트리 의존성은 정확한 커밋이 `nabi.lock`에 고정됩니다. 그러니 `nabi.lock`도 버전 관리에 커밋하세요. 고정된 버전을 앞으로 옮기는 것은 `nabi update`뿐입니다.

## 패키지 레지스트리

기본 패키지 레지스트리는 [packages.nabi-lang.org](https://packages.nabi-lang.org)입니다. 이름으로 검색하고 버전 요구 사항과 함께 추가합니다. 버전 요구 사항은 Cargo와 같이 동작합니다. `^1.2`는 `>=1.2.0, <2.0.0`을 뜻합니다. 요구 사항을 생략하면 최신 버전이 `^`과 함께 기록됩니다.

```text
$ nabi search json
    Updating index https://github.com/nabi-lang/packages-index
json 0.2.0 - JSON (RFC 8259): Value tree, strict parser, streaming tokenizer, typed encode/decode, JSON Pointer
$ nabi add json@^0.2
 Downloading json 0.2.0
       Added json v0.2.0
```

```nabion
Package {
    name: "app",
    version: "0.1.0",
    nabi_version: "0.1.0",
    toolchain: "0.1",
    deps: { "json": Dep::version("^0.2") },
}
```

이름을 잘못 쓰면 가장 가까운 패키지를 알려 줍니다.

```text
$ nabi add jsn
    Updating index https://github.com/nabi-lang/packages-index
nabi: error: no package named `jsn` in https://github.com/nabi-lang/packages-index
help: did you mean `json`? (or search: `nabi search jsn`)
```

그냥 문자열은 `Dep::version`의 축약이고, 연산자가 없는 요구 사항은 `^`을 뜻합니다. 그래서 다음 세 줄은 같은 것을 요청합니다. 패키지의 README에는 짧은 형태인 `"json": "0.2"`가 자주 나옵니다. 이 형태는 `>=0.2.0, <0.3.0`을 뜻합니다.

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "textwrap": "1.2",
    // "textwrap": "^1.2",
    // "textwrap": Dep::version("^1.2"),
} }
```

경로는 버전이 아닙니다. `"../geom"`이 아니라 `Dep::path("../geom")`로 씁니다.

의존성에는 키가 두 개 더 있습니다. `package:`는 이름을 바꿉니다. 아래에서는 패키지 `json5`를 코드에서 `j5`로 부릅니다. `target:`은 맞는 타깃을 빌드할 때만 그 의존성을 씁니다.

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "j5": Dep::version("^0.2", package: "json5"),
    "nix": Dep::version("^0.4", target: "cfg(target_family = \"unix\")"),
} }
```

패키지는 자기 의존성만, 자기가 붙인 이름으로 봅니다. 여기서 `j5::..`는 `json5`이고, 빌드의 다른 패키지가 `json5`를 그 이름으로 의존하더라도 `json5::..`는 아무것도 아닙니다. 의존성의 이름이 자식 모듈의 이름이기도 하면 `name::..`가 둘 다를 뜻하므로 에러입니다. 의존성은 `::name::..`, 모듈은 `pkg::name::..`로 쓰거나, `package:`로 의존성의 이름을 바꿉니다.

버전 요구만 적은 의존성은 언제나 기본 레지스트리의 index를 뜻합니다. 그 index를 미러에서 받으려면 `~/.nabi/sources.nabion`에 미러를 적습니다. `nabi.lock`에는 그대로 기본 레지스트리의 index가 적히므로, 미러를 쓰든 안 쓰든 lock은 같습니다.

```nabion
Sources { replace: { "https://github.com/nabi-lang/packages-index": "https://mirror.example.com/packages-index" } }
```

레지스트리는 게시된 버전마다 아카이브를 보관합니다. `nabi`는 그 아카이브를 받고(`Downloading json 0.2.0`) SHA-256을 index와 대조하므로, 패키지의 git 저장소가 사라져도 패키지는 계속 쓸 수 있습니다. 레지스트리에 아카이브가 없을 때만 `nabi`가 패키지의 git 저장소를 가져옵니다.

Nabi 프로젝트가 게시하는 패키지(`json`, `toml`, `http`, `markdown`, ...)는 **공식** 패키지이고, 이런 패키지는 `nabi add`를 건너뛰어도 됩니다. 코드에서 쓰기만 하면 다음 `nabi run`(또는 build, check, test, REPL 셀)이 대신 추가하고 그렇다고 알려 줍니다.

```text
$ cat src/main.nabi
use json::parse;
...
$ nabi run
 Downloading json 0.2.0
       Added json ^0.2 (official)
```

이제 `nabi.nabion`에 `"json": "^0.2"`가 있고 `nabi.lock`에 고정되어 있습니다. `nabi add json`을 실행한 것과 같습니다. 이렇게 들어오는 것은 공식 패키지뿐이고, 다른 이름은 여전히 `nabi add <name>`을 실행하라고 알려 줍니다. `--locked`(또는 `--frozen`)이거나 CI에서(`CI` 변수가 설정됨) `nabi`는 매니페스트를 고치지 않고, 패키지를 직접 추가하라고 알려 줍니다.

toolchain에는 공식 패키지가 들어 있고, 그 버전이 요구 사항에 맞으면 그것이 골라집니다. 인덱스도 다운로드도 필요 없으므로 네트워크 없이 됩니다. `nabi update`는 그래도 가장 새 버전으로 옮기고, `nabi component update packages`는 모든 공식 패키지의 가장 새 버전을 미리 받아 둡니다.

한 패키지에 대한 요구는 모두 셉니다. `app`이 `json` `>=0.1`을, `table`이 `json` `^0.1`을 요구하면 `nabi`는 둘 다 받아들이는 가장 새 `0.1.x`를 고르고, 그러려면 사이에 있는 패키지의 더 오래된 버전으로 되돌아가기도 합니다. 맞는 조합이 없으면 에러가 누가 무엇을 요구하는지, 무엇을 바꾸면 되는지 보여 줍니다.

```text
nabi: error: no version of package `b` meets every requirement on it:
  `^2` (required by `a` 1.4.0)
  `^1` (required by `c` 0.3.0)
... has b 1.2.0, 2.0.1; none of them meets all of these
how those packages came in:
  `a` 1.4.0: the root package requires `a` `^1.0`
  `c` 0.3.0: the root package requires `c` `~0.3`
one build has one version of a package
help: `c` 0.4.0 gets along with the other requirements on `b`: change the root package's requirement on it, `"c": "^0.4.0"`
```

### 보안 권고: `nabi audit`

레지스트리는 [보안 권고](https://packages.nabi-lang.org/ko/advisories)를 공개합니다. 어떤 패키지의 어느 버전에 알려진 취약점이 있는지, 어느 버전에서 고쳐졌는지를 밝힙니다. `nabi audit`은 `nabi.lock`의 패키지를 보안 권고와 대조하고, 어느 버전으로 올려야 하는지 알려 줍니다.

```sh
nabi audit
```

해당하는 권고가 없으면 0, 있으면 1(그래서 CI 작업이 실패합니다), 확인하지 못했으면 2로 끝납니다. 권고 목록은 레지스트리마다 한 번 받아 `$NABI_HOME/cache/advisories`에 한 시간 동안 두고, `--offline`은 그 사본만 씁니다. 읽어 보고 받아들이기로 한 권고는 `--ignore <id>`로 건너뛰고, 도구에서 쓸 때는 `--json`으로 레코드를 출력합니다. 경로·git 의존성과 네이티브 패키지는 확인하지 않습니다.

취약점을 찾았다면 먼저 그 패키지의 소유자에게 비공개로 알리세요. 답이 없거나 레지스트리·툴체인의 문제라면 [security@nabi-lang.org](mailto:security@nabi-lang.org)로 보내 주세요. 자세한 내용은 [보안 권고](packages/04-security-advisories.html)에 있습니다.

라이브러리에는 사용자가 켜고 끌 수 있는 **feature**를 둘 수 있습니다. `nabi login`과 `nabi publish`로 내 라이브러리를 패키지로 공개할 수도 있습니다. 둘 다 Nabi 프로젝트가 관리하는 패키지(`json`, `toml`, `csv`, `http`, `markdown`, `spatial`, `nabi-syntax`)와 함께 [18장](18-features-and-publishing.html)에서 다룹니다.

## 워크스페이스

함께 개발하는 패키지들, 이를테면 앱과 그 앱이 쓰는 라이브러리는 **워크스페이스** 하나에 둘 수 있습니다. 맨 위 디렉터리에는 패키지를 기술하는 대신 멤버를 나열하는 `nabi.nabion`을 둡니다.

```nabion
Workspace { members: ["app", "libs/*"] }
```

```text
shop/
|-- nabi.nabion          Workspace { members: [..] }
|-- nabi.lock            모든 멤버의 lock 하나
|-- target/              빌드 디렉터리 하나
|-- app/                 Package { name: "app", deps: { "cart": Dep::path("../libs/cart") } }
`-- libs/cart/           Package { name: "cart", .. }
```

`libs/*`는 `libs/` 바로 아래의 패키지 전부를 뜻합니다. 멤버들은 `nabi.lock`과 `target/`을 함께 쓰므로 모두 같은 버전의 의존성으로 빌드되고, 라이브러리는 모두를 위해 한 번만 빌드됩니다. 멤버끼리는 `Dep::path`로 씁니다.

맨 위에서 명령을 실행하면 모든 멤버에서 돌고, `-p`로 멤버를 고를 수 있습니다. 멤버 안에서는 `--workspace`가 모든 멤버에 닿습니다.

```text
$ nabi test                 # 맨 위에서: 모든 멤버
$ nabi run -p app           # 멤버 하나
$ cd libs/cart && nabi test --workspace
```

## 단일 파일 스크립트와 환경

패키지까지 만들 필요가 없는 작은 스크립트는 파일 맨 위의 `//!` 주석 안에 있는 ```` ```nabion ```` 블록에 의존성을 둡니다. 스크립트를 마지막 인자로 주고 `nabi add`를 실행하면 이 블록을 대신 써 줍니다. 경로는 스크립트 기준의 상대 경로로 기록되고, lockfile은 캐시에 보관됩니다. 스크립트가 쓰는 공식 패키지는 처음 실행할 때 블록에 추가됩니다.

```bash
nabi add geom --path ../geom tool.nabi
```

```text
       Added geom to tool.nabi
```

````nabi norun
//! ```nabion
//! Package {
//!     deps: {
//!         "geom": Dep::path("../geom"),
//!     },
//! }
//! ```
mod helper;

use geom::{Circle, area_sum};
println(f"{area_sum([Circle { r: helper::radius() }])}");
````

```bash
nabi run tool.nabi
```

스크립트에도 파일 모듈을 둘 수 있습니다. 패키지의 `src/`에서와 마찬가지로 `mod helper;`는 옆에 있는 `helper.nabi`를 불러옵니다. 모듈 파일에도 자기 `//!` 블록을 둘 수 있고(`nabi add <name> helper.nabi`), 모든 파일의 블록이 합쳐집니다. 스크립트는 `nabi install`로 설치한 라이브러리를 보지 않고, 블록에 적힌 것만 셉니다. 스크립트가 한 파일로 부족해지면 `nabi init --script tool.nabi`로 패키지로 바꿉니다. 블록은 `nabi.nabion`이 되고, `tool.nabi`는 `src/main.nabi`가 되며, 그 모듈 파일은 `src/`로 옮겨집니다.

이름 붙은 환경(environment)은 REPL과 Jupyter에서 씁니다. `nabi install`로 환경에 라이브러리를 설치합니다. 환경은 `--env`로 고르거나 `nabi env use`로 기본값을 바꿉니다. 따로 활성화하는 단계는 없습니다. 환경이 아직 없으면 `nabi env list`가 만드는 방법을 알려 줍니다. 환경은 `~/.nabi/envs` 아래의 디렉터리이므로, 이름은 어느 시스템에서나 파일 이름으로 쓸 수 있어야 합니다. `/`, `\`를 넣거나 `.`으로 시작할 수 없고, 길이는 64자까지이며, `con` 같은 Windows 장치 이름은 쓸 수 없습니다. 대소문자만 다른 이름은 같은 이름으로 보고, `current`는 `nabi env use`로 고른 환경을 가리키는 데 쓰이므로 예약되어 있습니다.

```text
$ nabi env create sci
     Created environment `sci` (~/.nabi/envs/sci)
$ nabi install --env sci geom --path ~/work/geom
   Installed geom into environment `sci` (for `nabi repl` / `nabi jupyter`; a script gets it with `nabi add geom <file.nabi>`)
$ nabi env use sci
$ nabi env list
* sci (active)
```

패키지는 환경을 쓰지 않습니다. 패키지 안에서는 자기 `nabi.nabion`만이 기준입니다.

## 프로그램 설치

`nabi install`은 `cargo install`처럼 프로그램도 설치합니다. `src/main.nabi`가 있는 패키지(레지스트리, git, 경로에서)나 스크립트 파일을 `--release`로 빌드해 `~/.nabi/bin`에 넣습니다. 레지스트리의 같은 버전이나 같은 git 커밋을 다시 설치하려 하면 아무것도 하지 않고, 다시 빌드하는 방법을 알려 줍니다. 새 버전은 이전 버전을 대체합니다. 경로나 스크립트 파일은 언제나 다시 빌드합니다.

```text
$ nabi install tool
   Installed tool 0.1.0 (~/.nabi/bin/tool)
$ nabi install tool
`tool` 0.1.0 is installed already (~/.nabi/bin/tool); to rebuild it, run `nabi install tool --force`
$ nabi install tool --force
 Reinstalled tool 0.1.0 (~/.nabi/bin/tool)
$ nabi install tool@0.2
    Upgraded tool from 0.1.0 to 0.2.0 (~/.nabi/bin/tool)
```

`nabi install --list`는 설치된 것을 보여 주고, `nabi uninstall tool`은 설치한 것을 제거합니다.

## 정리

- `mod`로 모듈을 만들고, `pkg::` / `sup::` / `std::` 경로와 `use`로 이름을 가리킵니다.
- `use` 없이 쓰는 이름은 프렐류드에서 옵니다. 프렐류드는 자주 쓰는 `core`의 이름과 모듈을 담은 정해진 목록이고, 나머지는 모듈을 거쳐 부르거나(`task::sleep`, `ops::Add`) 가져옵니다.
- 항목은 기본적으로 비공개입니다. `pub(sub)`, `pub(pkg)`, `pub`으로 범위를 넓힙니다.
- `mod x;`는 `x.nabi`(와 `x/`)에 대응합니다. 루트는 `main.nabi` 또는 `lib.nabi`입니다.
- `nabi new [--lib]`으로 패키지를 만들고 `nabi add`로 의존성(경로, git, 레지스트리)을 추가합니다. `nabi.lock`은 커밋합니다.
- 워크스페이스(`Workspace { members: [..] }`)는 함께 개발하는 패키지들에 lock 하나와 `target/` 하나를 줍니다. `-p <name>`과 `--workspace`로 멤버를 고릅니다.
- 레지스트리 패키지는 `nabi search` / `nabi add name@^1.2`로 다룹니다. feature와 공개는 [18장](18-features-and-publishing.html)에 있습니다.
- 스크립트는 `//! ```nabion` 블록을 쓰고(`nabi add <name> <file.nabi>`), `nabi init --script`로 패키지로 바꿀 수 있습니다. `nabi env`는 REPL / Jupyter용입니다.

## 더 읽을거리

- 사양: [모듈](../spec/refs-lang/005-modules/504-modules.html), [`use`](../spec/refs-lang/005-modules/505-use.html), [가시성](../spec/refs-lang/005-modules/506-visibility.html), [경로](../spec/refs-lang/005-modules/507-paths.html), [re-export](../spec/refs-lang/005-modules/508-re-export.html)
- 도구: [`nabi` 프런트엔드](../spec/refs-lang/009-tools/910-nabi-frontend.html), [`nabi.nabion` / `nabi.lock`](../spec/refs-lang/009-tools/910-nabi-frontend.html#9103-nabinabion--nabilock), [레지스트리 패키지](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104a-레지스트리-패키지), [환경](../spec/refs-lang/009-tools/910-nabi-frontend.html#9105-환경), [인라인 매니페스트](../spec/refs-lang/009-tools/910-nabi-frontend.html#9107-단일-파일-스크립트의-인라인-매니페스트)
- [1. 시작하기](01-getting-started.html)
- 다음 장: [12. 동시성](12-concurrency.html)
