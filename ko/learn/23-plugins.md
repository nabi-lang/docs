# 23. 플러그인

플러그인은 프로그램이 실행 중에 로드하는 패키지입니다. 게임의 레벨, 편집기 도구, 사용자가 폴더에 넣는 필터 같은 것입니다. Nabi에서 플러그인은 매니페스트에 한 줄을 더한 평범한 라이브러리 패키지입니다. 플러그인은 프로그램의 런타임 위에서 돌기 때문에, 그것이 돌려주는 문자열, 리스트, 클로저, `any` 값은 모두 프로그램의 평범한 값이고, 그 함수도 다른 함수처럼 부릅니다.

이 장에서는 플러그인을 로드하고, 부르고, 언로드하는 프로그램을 만듭니다.

## 플러그인 패키지

플러그인은 라이브러리입니다. `src/lib.nabi`가 있고 `src/main.nabi`는 없습니다. `nabi.nabion`에 `lib: { kind: "plugin" }`을 쓰면 `nabi build`가 플러그인으로 링크합니다.

```nabion
Package {
    name: "greeter",
    version: "0.1.0",
    lib: { kind: "plugin" },
    deps: {},
}
```

프로그램이 플러그인에서 얻는 것은 그 `pub fn`입니다.

```nabi norun
pub fn greet(name: str) -> str {
    f"hello, {name}"
}

pub fn squares(n: i32) -> List<i32> {
    let xs: List<i32> = List::new();
    for i in 0..n {
        xs.push(i * i);
    }
    xs
}

pub fn counter() -> fn() -> i32 {
    let n = [0];
    || {
        n[0] += 1;
        n[0]
    }
}
```

```bash
nabi build
```

플러그인은 macOS에서 `target/debug/libgreeter.dylib`, Linux에서 `libgreeter.so`, Windows에서 `greeter.dll`입니다. `std::plugin::file("greeter")`는 빌드하는 플랫폼에 맞는 이름을 줍니다.

## 로드하기

프로그램은 `plugin::load`로 파일을 로드하고, 함수마다 경로와 타입으로 달라고 합니다. 타입은 확인됩니다. 플러그인이 `fn(str) -> str`로 선언한 것을 `fn(i32) -> str`로 달라고 하면 크래시가 아니라 에러입니다.

```nabi norun
use std::plugin;

fn main() throws {
    let file = plugin::file("greeter");
    let p = try plugin::load(f"../greeter/target/debug/{file}");
    let greet = try p.get::<fn(str) -> str>("greeter::greet");
    println(greet("Ada"));

    let squares = try p.get::<fn(i32) -> List<i32>>("greeter::squares");
    println(squares(5));

    // 플러그인이 만든 클로저는 상태를 유지합니다.
    let next = try p.get::<fn() -> fn() -> i32>("greeter::counter")();
    println(f"{next()} {next()} {next()}");
}
```

```text
hello, Ada
[0, 1, 4, 9, 16]
1 2 3
```

프로그램 쪽에서 따로 선언할 것은 없습니다. `std::plugin`을 쓰는 프로그램은 플러그인을 받을 수 있게 빌드됩니다.

경로를 쓰는 대신 `nabi`에게 플러그인 빌드를 맡겨도 됩니다. `plugin: true`인 의존성은 프로그램 옆에(같은 프로필, 같은 타깃으로) 플러그인으로 빌드되고, `plugin::beside(name)`이 그것을 찾습니다.

```nabion
Package {
    name: "app",
    deps: { "greeter": Dep::path("../greeter", plugin: true) },
}
```

```nabi norun
let p = try plugin::load(plugin::beside("greeter"));
```

## 타입 공유

프로그램과 플러그인은 둘 다 아는 타입의 값을 주고받을 수 있습니다. 그 타입을 둘이 함께 의존하는 패키지에 두면 됩니다.

```nabion
// game/nabi.nabion: 프로그램
Package { name: "game", deps: { "engine": Dep::path("../engine") } }

// enemies/nabi.nabion: 플러그인
Package { name: "enemies", lib: { kind: "plugin" }, deps: { "engine": Dep::path("../engine") } }
```

그러면 `engine::Entity`는 양쪽에서 같은 타입이고, `p.get::<fn(engine::World) -> List<engine::Entity>>("enemies::spawn")`은 그 값을 그대로 주고받습니다. 플러그인은 `engine`을 프로그램에서 가져오므로 `engine`의 static은 하나뿐입니다. 플러그인만 쓰는 의존성은 함께 넣을 수 있습니다. `Dep::version("^1.2", bundle: true)`는 그것을 플러그인에 링크합니다.

`any`와 `any Trait` 값도 넘어갑니다. 플러그인이 자기 타입을 담은 `any Display`를 돌려주면, 프로그램의 `{v}`는 플러그인의 `fmt`로 그것을 출력합니다. 리플렉션도 그 타입을 봅니다. `core::reflect::info_of(v)`는 플러그인이 만든 값을 설명하고, `p.types()`는 플러그인이 설명하는 타입을 나열합니다.

## 로드되지 않을 때

에러마다 할 일을 알려 줍니다. 파일이 없거나, 플러그인이 아니거나, 다른 툴체인으로 빌드되었거나, 프로그램에 없는 패키지를 쓰거나, 함수가 없거나 타입이 다르면 모두 `PluginError`로 돌아옵니다.

```nabi norun
let p = try plugin::load("plugins/missing.so") catch e {
    println(e);
    return;
};
```

```text
no plugin at plugins/missing.so; build its package with `nabi build` and give the file it writes under target/<profile>/ (`plugin::file(name)` gives the file name)
```

플러그인과 프로그램은 같은 툴체인에서 나와야 합니다. `nabi toolchain`이 바뀌면 플러그인을 다시 빌드하십시오. 툴체인 버전 사이에 안정적인 플러그인 ABI는 없습니다.

## 언로드

`p.unload()`는 플러그인을 바로 찾을 수 없게 하지만, 그 무엇이든 살아 있는 동안은 메모리에 남겨 둡니다. 그 타입의 객체, 그것이 만든 클로저, 아직 돌고 있는 그 안의 호출이 그렇습니다. 결과가 어느 경우인지 알려 줍니다.

```nabi norun
match p.unload() {
    plugin::Unload::Done => println("gone"),
    plugin::Unload::Pending(why) => println(f"later: {why}"),
    plugin::Unload::Kept(why) => println(f"stays: {why}"),
}
```

자기 코드의 콜백을 C 라이브러리에 준 플러그인은 계속 남습니다. C가 아직 부를 수 있기 때문입니다.

## 플러그인이 되는 곳

플러그인은 macOS(arm64), Linux(glibc, x86-64와 arm64), Windows(x64와 arm64)에서 로드됩니다. WebAssembly와 정적 musl 타깃에서는 프로그램이 실행 중에 코드를 로드할 수 없습니다. 그래서 그 타깃으로 플러그인이나 `std::plugin`을 쓰는 프로그램을 빌드하면, 대신 프로그램 하나로 빌드하라는 에러가 납니다. 플러그인의 패키지를 평범한 의존성으로 만드십시오.

## 요약

- `lib: { kind: "plugin" }`은 라이브러리 패키지를 플러그인으로 빌드합니다. 의존성의 `plugin: true`는 그것을 프로그램 옆에 빌드합니다.
- `plugin::load(path)`로 로드하고, `p.get::<F>("pkg::name")`으로 정확히 타입 `F`인 `pub fn`을 얻습니다.
- 값은 그대로 넘어갑니다. 런타임도 힙도 하나입니다. 타입은 둘이 함께 의존하는 패키지로 공유합니다.
- `unload`는 플러그인의 무엇도 살아 있지 않을 때까지 기다립니다.

## 더 읽을거리

- 사양: [플러그인](../spec/refs-lang/009-tools/912-plugins.html), [모듈 디스크립터 ABI](../spec/refs-lang/008-abi/812-module-abi.html), [`std::plugin`](../spec/refs-standard-libraries/300-std/311-plugin.html)
- API: [std::plugin](../../std/std/plugin.html)
- 다음 장: [24. 리플렉션](24-reflection.html)
