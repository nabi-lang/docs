# 24. 反射

Nabi 程序可以在运行时查看自己的类型：结构体的字段及其类型、枚举的变体，以及你为工具附加的属性。编辑器的属性面板、游戏存档系统或通用序列化器都建立在这之上。本章介绍如何获取类型的描述，通过它读取和修改字段，附加自己的属性，以及在不为每个类型手写代码的情况下保存和加载整个场景。

每个类型都已经知道自己的名字和大小（`T::type_name()`、`T::size()`）。其余功能需要主动启用：在想要查看的类型上加 `#[derive(Inspect)]`。从不查询某个类型的程序不会为这个 derive 付出任何代价。

## 类型的描述

`T::type_info()` 返回一个 `TypeInfo`。它直接读取编译器写入程序的数据，因此读取时不分配内存。要读取或修改的值放进 `any` 中。

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

- `fields()` 按顺序列出所有字段，包括私有字段：derive 代表类型作者的同意。`field(name)` 查找单个字段。
- `Field::get(owner)` 读取字段，`get_as::<T>(owner)` 以 `T` 读取，`set(ref owner, value)` 写入。类型不对的值会被拒绝（`false`），但能容纳的整数会被转换。
- `core::reflect::info_of(value)` 给出 `any` 中装的是哪个类型。
- 枚举的字段属于它的变体：`variants()`、`variant_of(value)` 和 `Variant::fields()`。

## 自定义属性

标有 `#[runtime_attribute(name)]` 的结构体会成为一个属性，可以写在类型、字段和变体上。它的实参像结构体字面量一样被检查，因此写错了会是编译错误，并提示应该怎么写。`attr::<A>()` 读回该值。

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

属性的字段可以是数字、`bool`、`char` 或 `str`。`#[inspect(skip)]` 对反射隐藏一个字段，`#[renamed_from("old")]` 为读取旧数据的代码记录字段以前的名字。

## 保存和加载场景

`std::nabion::to_value_any` 把任何有描述的值转换为 NABION `Value`（[第 15 章](15-nabion.html)），`from_value_any` 则根据 `Value` 和 `TypeInfo` 构造回来。场景类型中除了 derive 之外不需要手写任何东西。文本中省略的字段取默认值，就像老鼠的 `hp` 那样。

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

`T | none` 上的 `?` 只能用在本身可能返回 `none` 的函数中。在会抛出错误的函数中，写 `x ?? throw err` 把缺失的值变成错误，`load` 就是这样做的。

## 何时使用

当代码必须处理自己不知道的类型时，就用反射，例如编辑器、调试器、存档格式或测试数据加载器。对于一个已知类型，`#[derive(Nabion)]` 或普通函数更短也更快。完整的 API 见标准库规范（[2.12](../spec/refs-standard-libraries/200-core/212-reflect.html)），规则见语言参考（[6.10](../spec/refs-lang/006-meta-system/610-reflection.html)）。
