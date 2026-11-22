# 24. 反射

Nabi 程式可以在執行時檢視自己的型別：結構的欄位及其型別、列舉的變體，以及你為工具附加的屬性。編輯器的屬性面板、遊戲存檔系統或泛用的序列化工具，都建立在這上面。本章說明如何取得型別的描述、透過描述讀取和修改欄位、附加你自己的屬性，以及在不為每個型別手寫程式碼的情況下儲存和載入整個場景。

每個型別本來就知道自己的名稱和大小（`T::type_name()`、`T::size()`）。其餘部分需要自行選用：在你想檢視的型別上加上 `#[derive(Inspect)]`。從不查詢某個型別的程式，不會因為這個衍生付出任何代價。

## 型別的描述

`T::type_info()` 回傳一個 `TypeInfo`。它直接讀取編譯器寫進程式的資料，所以讀取時不會配置任何記憶體。想讀取或修改的值要放進 `any`。

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

- `fields()` 依序列出每個欄位，私有欄位也包括在內：加上衍生就代表型別作者同意。`field(name)` 找出其中一個。
- `Field::get(owner)` 讀取欄位，`get_as::<T>(owner)` 以 `T` 讀取，`set(ref owner, value)` 寫入。型別不符的值會被拒絕（`false`），但放得下的整數會自動轉換。
- `core::reflect::info_of(value)` 告訴你 `any` 裝的是哪個型別。
- 列舉的欄位屬於它的變體：`variants()`、`variant_of(value)` 和 `Variant::fields()`。

## 自訂屬性

標上 `#[runtime_attribute(name)]` 的結構，會成為可以寫在型別、欄位和變體上的屬性。它的引數會像結構字面值一樣受到檢查，所以寫錯會是編譯錯誤，並告訴你該怎麼寫。`attr::<A>()` 讀回這個值。

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

屬性的欄位可以是數字、`bool`、`char` 或 `str`。`#[inspect(skip)]` 讓欄位不被反射看見，`#[renamed_from("old")]` 則記錄欄位以前的名稱，供讀取舊資料的程式碼使用。

## 儲存與載入場景

`std::nabion::to_value_any` 把任何有描述的值轉成 NABION `Value`（[第 15 章](15-nabion.html)），`from_value_any` 則從 `Value` 和 `TypeInfo` 重建出值。場景的型別除了衍生之外，沒有任何手寫的程式碼。文字中省略的欄位會取預設值，就像 rat 的 `hp` 一樣。

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

對 `T | none` 使用 `?`，只能在本身可能回傳 `none` 的函式中進行。在會擲回錯誤的函式中，寫 `x ?? throw err` 把缺失值轉成錯誤，就像 `load` 的做法。

## 何時使用

當程式碼必須處理它事先不知道的型別時，就用反射，例如編輯器、偵錯工具、存檔格式或測試資料載入器。若只針對一個已知的型別，`#[derive(Nabion)]` 或一般函式會更短也更快。完整的 API 在標準函式庫規格中（[2.12](../spec/refs-standard-libraries/200-core/212-reflect.html)），規則則在語言參考中（[6.10](../spec/refs-lang/006-meta-system/610-reflection.html)）。
