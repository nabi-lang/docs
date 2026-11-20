# 24. 리플렉션

Nabi 프로그램은 실행 중에 자기 타입을 들여다볼 수 있습니다. struct의 필드와 그 타입, enum의 variant, 도구를 위해 붙인 속성까지 볼 수 있습니다. 에디터의 속성 패널, 게임 저장 기능, 범용 직렬화기가 이 위에 만들어집니다. 이 장에서는 타입의 설명을 얻는 법, 그것으로 필드를 읽고 바꾸는 법, 직접 만든 속성을 붙이는 법, 타입마다 코드를 쓰지 않고 장면 전체를 저장하고 불러오는 법을 봅니다.

모든 타입은 자기 이름과 크기를 이미 압니다(`T::type_name()`, `T::size()`). 나머지는 직접 켭니다. 들여다볼 타입에 `#[derive(Inspect)]`를 붙이면 됩니다. 타입에 대해 묻지 않는 프로그램은 derive를 붙여도 비용이 없습니다.

## 타입의 설명

`T::type_info()`는 `TypeInfo`를 돌려줍니다. 컴파일러가 프로그램에 써 둔 데이터를 바로 읽으므로, 읽을 때 메모리를 할당하지 않습니다. 읽거나 바꿀 값은 `any`에 담습니다.

```nabi
#[derive(Inspect)]
struct Enemy {
    name: str;
    hp: i32 = 100;
    pub speed: f64 = 1.5;
}

fn main() {
    let t = Enemy::type_info();
    println(f"{t.name()}: {t.field_count()} fields, size matches: {t.size() == Enemy::size()}");
    for f in t.fields() {
        println(f"  {f.name()}: {f.type_ref().name()} pub={f.is_pub()} default={f.has_default()}");
    }

    let e: any = Enemy { name: "orc" };
    let hp = t.field("hp") ?? panic("no hp");
    println(f"hp = {hp.get_as::<i32>(e) ?? 0}");
    println(f"set: {hp.set(ref e, 40)}, then: {hp.set(ref e, "forty")}");
    println(f"hp = {hp.get_as::<i32>(e) ?? 0}");
    println(f"held: {core::reflect::info_of(e) ?? t:?}");
}
```

```output
Enemy: 3 fields, size matches: true
  name: str pub=false default=false
  hp: i32 pub=false default=true
  speed: f64 pub=true default=true
hp = 100
set: true, then: false
hp = 40
held: TypeInfo(Enemy)
```

- `fields()`는 모든 필드를 순서대로 보여 주고, 비공개 필드도 포함합니다. derive를 붙인 것이 타입 작성자의 동의이기 때문입니다. `field(name)`은 필드 하나를 찾습니다.
- `Field::get(owner)`는 필드를 읽고, `get_as::<T>(owner)`는 `T`로 읽고, `set(ref owner, value)`는 씁니다. 타입이 틀린 값은 거절하지만(`false`), 범위에 맞는 정수는 변환됩니다.
- `core::reflect::info_of(value)`는 `any`가 담은 타입을 알려 줍니다.
- enum의 필드는 variant에 속합니다: `variants()`, `variant_of(value)`, `Variant::fields()`.

## 직접 만든 속성

`#[runtime_attribute(name)]`을 붙인 struct는 타입, 필드, variant에 쓸 수 있는 속성이 됩니다. 인자는 struct 리터럴처럼 검사되므로, 실수는 무엇을 써야 하는지 보여 주는 컴파일 에러가 됩니다. `attr::<A>()`가 값을 다시 읽습니다.

```nabi
/// Shown as a slider in the editor.
#[runtime_attribute(range)]
pub struct Range { pub min: f64; pub max: f64; }

#[derive(Inspect)]
struct Enemy {
    #[range(0, 100)] hp: i32 = 100;
    #[range(min = 0.5, max = 4.0)] speed: f64 = 1.0;
    name: str = "orc";
}

fn main() {
    for f in Enemy::type_info().fields() {
        match f.attr::<Range>() {
            r: Range => println(f"{f.name()}: slider {r.min}..{r.max}"),
            none => println(f"{f.name()}: text box"),
        }
    }
}
```

```output
hp: slider 0..100
speed: slider 0.5..4
name: text box
```

속성의 필드는 수, `bool`, `char`, `str`입니다. `#[inspect(skip)]`은 필드를 리플렉션에서 숨기고, `#[renamed_from("old")]`은 예전 데이터를 읽는 코드를 위해 필드의 예전 이름을 남깁니다.

## 장면 저장과 불러오기

`std::nabion::to_value_any`는 설명이 있는 값을 NABION `Value`로 바꾸고([15장](15-nabion.html)), `from_value_any`는 `Value`와 `TypeInfo`로 값을 다시 만듭니다. 장면 타입에는 derive 말고는 손으로 쓴 코드가 없습니다. 텍스트에 없는 필드는 기본값을 씁니다. rat의 `hp`가 그렇습니다.

```nabi
use core::reflect;
use std::nabion;
use std::nabion::NabionError;

#[runtime_attribute(range)]
pub struct Range { pub min: f64; pub max: f64; }

#[derive(Inspect, Clone)]
struct Vec2 { x: f64 = 0.0; y: f64 = 0.0; }

#[derive(Inspect, Clone)]
enum Kind { Player, Enemy { level: i32 }, Prop }

#[derive(Inspect, Clone)]
struct Entity { name: str; pos: Vec2; kind: Kind; #[range(0, 100)] hp: i32 = 100; }

#[derive(Inspect, Clone)]
struct Scene { entities: List<Entity>; tags: Dict<str, i32>; }

fn load<T: Inspect>(text: str) throws (NabionError) -> T {
    let v = try nabion::from_value_any(T::type_info(), try nabion::parse(text));
    return reflect::cast::<T>(v) ?? throw NabionError::new(f"not a {T::type_name()}");
}

fn main() throws (NabionError) {
    let tags: Dict<str, i32> = Dict::new();
    tags.insert("level", 1);
    let scene = Scene {
        entities: [
            Entity { name: "hero", pos: Vec2 { x: 1.0, y: 2.0 }, kind: Kind::Player },
            Entity { name: "orc", pos: Vec2 { x: 5.0 }, kind: Kind::Enemy { level: 3 }, hp: 40 },
        ],
        tags: tags,
    };
    let text = try (try nabion::to_value_any(scene)).to_str();
    println(text);
    let back = try load::<Scene>(text);
    let e = back.entities;
    println(f"{e.len()} {e[0].pos.y} {e[1].hp} {e[1].pos.y} {back.tags["level"]} {e[1].kind.variant_name()}");
    let one = try load::<Entity>("Entity { name: \"rat\", pos: Vec2 {}, kind: Kind::Prop }");
    println(f"default hp {one.hp}");
    let hp = Entity::type_info().field("hp") ?? throw NabionError::new("no hp");
    let r = hp.attr::<Range>() ?? throw NabionError::new("no range");
    println(f"hp in {r.min}..{r.max}");
}
```

```output
Scene {
    entities: [
        Entity { name: "hero", pos: Vec2 { x: 1.0, y: 2.0 }, kind: Kind::Player, hp: 100 },
        Entity {
            name: "orc",
            pos: Vec2 { x: 5.0, y: 0.0 },
            kind: Kind::Enemy { level: 3 },
            hp: 40,
        },
    ],
    tags: { "level": 1 },
}
2 2 40 0 1 Enemy
default hp 100
hp in 0..100
```

`T | none`에 쓰는 `?`는 `none`을 돌려줄 수 있는 함수 안에서만 됩니다. throw하는 함수에서는 `x ?? throw err`로 없는 값을 에러로 바꿉니다. `load`가 그렇게 합니다.

## 언제 쓰나

코드가 모르는 타입을 다뤄야 할 때 리플렉션을 씁니다. 에디터, 디버거, 저장 형식, 테스트 데이터 로더가 그렇습니다. 아는 타입 하나라면 `#[derive(Nabion)]`이나 평범한 함수가 더 짧고 빠릅니다. 전체 API는 표준 라이브러리 사양([2.12](../spec/refs-standard-libraries/200-core/212-reflect.html))에, 규칙은 언어 레퍼런스([6.10](../spec/refs-lang/006-meta-system/610-reflection.html))에 있습니다.
