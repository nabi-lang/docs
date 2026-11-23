# 26. 핫 리로드

`nabi dev`는 프로그램을 실행하면서 편집한 내용을 실행 중에 적용합니다. 파일을 저장하면 잠시 뒤 실행 중인 프로그램이 새 코드를 쓰고, 그때까지 쌓은 것은 그대로 남습니다. 하던 레벨, 다듬던 창, 불러온 데이터가 모두 그렇습니다. 게임, 시뮬레이션, 사용자 인터페이스가 있는 도구처럼 작업하면서 지켜보는 프로그램을 위한 기능입니다.

이 장에서는 작은 게임 루프를 `nabi dev`로 실행하고, 함수를 바꾼 다음 타입을 바꿉니다.

## 세션 시작

`main`에 루프가 있는 패키지를 봅시다.

```nabi norun
use core::time::Duration;

struct Player {
    name: str;
    hp: i32;
}

fn draw(p: Player) {
    println(f"{p.name} has {p.hp} hp");
}

fn main() {
    let p = Player { name: "ann", hp: 10 };
    loop {
        draw(p);
        std::reload::point();
        std::thread::sleep(Duration::ms(500));
    }
}
```

`nabi run` 대신 `nabi dev`로 실행합니다.

```bash
nabi dev
```

```text
    Building the host of `game`
     Running `game` (1.4s)
    Watching press `r` to restart, `q` to quit
ann has 10 hp
ann has 10 hp
```

`std::reload::point()`는 새 코드를 받기 좋은 곳을 표시합니다. 프레임 사이, 하다 만 일이 없는 곳입니다. `nabi dev` 밖에서는 아무 일도 하지 않으므로, 같은 프로그램이 `nabi run`에서도 그대로 돕니다.

## 함수 바꾸기

`draw`를 고치고 저장합니다.

```nabi norun
fn draw(p: Player) {
    println(f"{p.name}: {p.hp} hp");
}
```

```text
  Rebuilding `game`
     Applied game v2: 1 function
ann: 10 hp
ann: 10 hp
```

다시 빌드한 것은 `game`뿐이고, 그 안에서도 바뀐 부분만 다시 컴파일했습니다. `main`의 루프는 계속 돌았고, 다음 `draw` 호출이 새 본문으로 갔습니다.

컴파일되지 않는 저장은 프로그램에 아무것도 바꾸지 않습니다.

```text
error[E0034]: no field `hpp` on `Player`
 --> src/main.nabi:9:28
        Kept the running code of `game`: fix the error and save
```

고치고 다시 저장하면 변경이 들어갑니다.

## 타입 바꾸기

`Player`에 새 필드를 줍니다. 프로그램이 도는 동안에는 프로그램이 가진 `Player`마다 그 필드의 값이 있어야 하므로, 새 필드에는 기본값이 필요합니다.

```nabi norun
struct Player {
    name: str;
    hp: i32;
    mana: i32 = 5;
}

fn draw(p: Player) {
    println(f"{p.name}: {p.hp} hp, {p.mana} mana");
}
```

```text
     Applied game v3: 1 function, 1 type (Player: +mana), 1 object migrated
ann: 10 hp, 5 mana
```

편집 전에 `main`이 만든 `Player`에 이제 `mana: 5`가 있습니다. 기본값이 없으면 빌드가 멈추고 무엇을 쓸지 알려 줍니다.

```text
error: new field `mana` needs a default while the program runs: write `mana: i32 = 0;` (or restart with `r`)
```

필드 제거, 필드 순서 변경, 열거형 variant 추가는 모두 적용됩니다. 이름을 바꿀 때는 어디서 왔는지 밝히면 값이 이어집니다.

```nabi norun
struct Player {
    name: str;
    #[renamed_from("hp")]
    health: i32;
}
```

필드의 타입을 바꾸면 옛 값을 옮길 수 없습니다. 에러는 새 필드에 기본값과 함께 따로 이름을 주거나, 재시작하라고 알려 줍니다.

## 상태를 두는 곳

레이아웃 변경은 실행 중인 함수가 바뀐 타입의 값을 지역 변수에 쥐고 있지 않을 때만 적용됩니다. `main`의 루프는 `p`를 줄곧 쥐고 있으므로, `Player`가 바뀌면 변경은 기다리며 이유를 말합니다.

```text
     Waiting game v3: `main` (src/main.nabi:13) holds a `Player` across `reload::point()`; keep it in a `reload::State`: `let p = reload::State::new(Player { .. });`
```

오래 사는 상태는 `std::reload::State`에 둡니다. 리로드가 갱신하는 법을 아는 상자이며, `nabi dev` 밖에서는 평범한 상자입니다.

```nabi norun
use core::time::Duration;
use std::reload;

fn main() {
    let p = reload::State::new(Player { name: "ann", hp: 10, mana: 5 });
    loop {
        draw(p.get());
        reload::point();
        std::thread::sleep(Duration::ms(500));
    }
}
```

두 가지를 더 알아 둡시다.

- `main` 자신의 루프는 세션 내내 옛 코드입니다. 프레임마다 하는 일은 여기의 `draw`처럼 `main`이 부르는 함수에 두십시오.
- 클로저는 만들어질 때의 본문을 유지합니다. 새 코드가 만든 새 클로저는 새 본문을 씁니다.

리로드마다 코드를 실행하려면, 예를 들어 캐시를 다시 만들려면, 훅을 등록합니다. 훅은 이름으로 보관하므로 새 코드에서 다시 등록하면 옛 것을 대체합니다.

```nabi norun
reload::on_reload("cache", |r| println(f"reloaded v{r.version}"));
```

## 재시작

제자리에서 할 수 없는 변경도 있습니다. `nabi.nabion`이나 `nabi.lock`이 바뀌거나(새 의존성, 새 버전, 다른 기능) toolchain이 바뀌면, `nabi dev`는 이유를 말하고 프로그램을 재시작합니다.

```text
  Restarting nabi.nabion changed
```

손으로 재시작하려면 `r`을 누르고(예를 들어 프로그램이 거부한 변경 뒤에), 끝내려면 `q`를 누릅니다. 키는 `nabi dev`의 것이므로, `nabi dev`가 도는 동안 프로그램의 표준 입력은 비어 있습니다.

## 무엇이 리로드되는가

여러분의 패키지가 리로드됩니다. 실행하는 패키지와 그 `path` 의존성이며, 워크스페이스 멤버도 들어갑니다. 레지스트리 패키지와 git 패키지는 세션마다 한 번 빌드하는 호스트에 들어 있으므로, 그것을 바꾸면 재시작합니다. 핫 리로드는 디버그 코드만 빌드합니다. `nabi dev --release`는 에러입니다. `nabi build`나 `nabi run`으로 빌드한 프로그램에는 이 장치가 전혀 없고 그 비용도 치르지 않습니다.

## 요약

- `nabi dev`는 프로그램을 실행하며 저장할 때마다 실행 중에 적용합니다. 컴파일 에러가 나면 옛 코드를 유지합니다.
- 프레임마다 `std::reload::point()`를 한 번 부릅니다.
- 새 필드에는 기본값이 필요합니다. `#[renamed_from]`은 이름을 바꾼 필드의 값을 유지합니다.
- 오래 사는 상태는 `reload::State`에, 프레임마다 하는 일은 함수에 둡니다.
- `r`은 재시작, `q`는 끝냄입니다. 매니페스트, 락 파일, toolchain이 바뀌면 저절로 재시작합니다.

## 더 읽을거리

- 사양: [핫 리로드](../spec/refs-lang/009-tools/914-hot-reload.html), [`nabi dev`](../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-dev-핫-리로드로-실행), [`std::reload`](../spec/refs-standard-libraries/300-std/313-reload.html)
- 코드를 직접 로드하기: [23. 플러그인](23-plugins.html)
- 도구: [16. 도구](16-tools.html)
- 다음 장: [27. Objective-C와 Apple 프레임워크](27-objc.html)
