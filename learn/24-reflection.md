# 24. Reflection

Nabi programs can look at their own types while they run: the fields of a struct with their types, an enum's variants, and attributes you attach for tools. This is what an editor's property panel, a save-game system or a generic serializer is built on. This chapter shows how to ask for a type's description, read and change fields through it, attach your own attributes, and save and load a whole scene with no hand-written code per type.

Every type already knows its name and size (`T::type_name()`, `T::size()`). The rest is opt-in: put `#[derive(Inspect)]` on the types you want to look into. A program that never asks about a type pays nothing for the derive.

## A type's description

`T::type_info()` returns a `TypeInfo`. It is read straight from data the compiler wrote into the program, so reading it allocates nothing. A value you want to read or change goes in an `any`.

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

- `fields()` lists every field in order, private ones too: the derive is the type author's consent. `field(name)` finds one.
- `Field::get(owner)` reads a field, `get_as::<T>(owner)` reads it as a `T`, and `set(ref owner, value)` writes one. A value of the wrong type is refused (`false`), but an integer that fits converts.
- `core::reflect::info_of(value)` tells which type an `any` holds.
- An enum's fields belong to its variants: `variants()`, `variant_of(value)`, and `Variant::fields()`.

## Your own attributes

A struct marked `#[runtime_attribute(name)]` becomes an attribute you can write on types, fields and variants. Its arguments are checked like a struct literal, so a mistake is a compile error that shows what to write. `attr::<A>()` reads the value back.

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

An attribute's fields are numbers, `bool`, `char` or `str`. `#[inspect(skip)]` hides a field from reflection, and `#[renamed_from("old")]` records a field's earlier name for code that reads old data.

## Saving and loading a scene

`std::nabion::to_value_any` turns any described value into a NABION `Value` ([chapter 15](15-nabion.html)), and `from_value_any` builds one back from a `Value` and a `TypeInfo`. Nothing in the scene types is written by hand except the derive. A field the text leaves out takes its default, as `hp` does for the rat.

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

`?` on a `T | none` works only in a function that may return `none` itself. In a function that throws, write `x ?? throw err` to turn a missing value into an error, as `load` does.

## When to use it

Reach for reflection when code must handle types it does not know, such as an editor, a debugger, a save format or a test fixture loader. For one known type, a `#[derive(Nabion)]` or a plain function is shorter and faster. The full API is in the standard library specification ([2.12](../spec/refs-standard-libraries/200-core/212-reflect.html)) and the rules in the language reference ([6.10](../spec/refs-lang/006-meta-system/610-reflection.html)).
