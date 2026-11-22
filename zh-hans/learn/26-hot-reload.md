# 26. 热重载

`nabi dev` 运行你的程序，并在程序运行期间应用你的修改。保存文件后片刻，正在运行的程序就会使用新代码，而它已经积累的一切都还在：你正在玩的关卡、正在调整的窗口、已经加载的数据。它是为那些你一边开发一边观察的程序准备的，例如游戏、模拟和带用户界面的工具。

本章在 `nabi dev` 下运行一个小游戏循环，先修改一个函数，再修改一个类型。

## 开始会话

取一个在 `main` 中有循环的包：

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

用 `nabi dev` 而不是 `nabi run` 运行它：

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

`std::reload::point()` 标记一个适合接收新代码的位置：在两帧之间，没有做到一半的工作。在 `nabi dev` 之外它什么也不做，所以同一个程序在 `nabi run` 下无需修改即可运行。

## 修改函数

编辑 `draw` 并保存：

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

只有 `game` 被重新构建，并且其中只有改动的部分被重新编译。`main` 中的循环一直在运行；它下一次调用 `draw` 时进入的就是新的函数体。

无法编译的保存不会改变程序中的任何东西：

```text
error[E0034]: no field `hpp` on `Player`
 --> src/main.nabi:9:28
        Kept the running code of `game`: fix the error and save
```

修好后再保存一次，修改就会生效。

## 修改类型

给 `Player` 添加一个新字段。程序运行期间，它已有的每个 `Player` 都必须为这个字段取得一个值，因此新字段需要默认值：

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

`main` 在编辑之前创建的 `Player` 现在有了 `mana: 5`。没有默认值时，构建会停止，并提示应该写什么：

```text
error: new field `mana` needs a default while the program runs: write `mana: i32 = 0;` (or restart with `r`)
```

删除字段、调整字段顺序和添加枚举变体都能生效。重命名时，只要说明值从哪里来，值就会保留：

```nabi norun
struct Player {
    name: str;
    #[renamed_from("hp")]
    health: i32;
}
```

修改字段的类型无法沿用旧值；错误会提示为新字段另起一个带默认值的名字，或者重启。

## 状态放在哪里

只有当没有正在运行的函数在局部变量中持有被修改类型的值时，布局变更才能生效。`main` 的循环始终持有 `p`，所以 `Player` 改变时，变更会等待并说明原因：

```text
     Waiting game v3: `main` (src/main.nabi:13) holds a `Player` across `reload::point()`; keep it in a `reload::State`: `let p = reload::State::new(Player { .. });`
```

把长期存在的状态放在 `std::reload::State` 中。它是一个重载机制知道如何更新的盒子；在 `nabi dev` 之外，它就是一个普通的盒子：

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

还有两点需要了解：

- 在整个会话中，`main` 自己的循环都是旧代码。把每一帧的工作放在 `main` 调用的函数中，就像这里的 `draw`。
- 闭包保留创建它时的函数体。由新代码创建的新闭包使用新的函数体。

要在每次重载后运行代码，例如重建缓存，可以注册一个钩子。钩子按名字保存，因此在新代码中再次注册会替换旧的钩子：

```nabi norun
reload::on_reload("cache", |r| println(f"reloaded v{r.version}"));
```

## 重启

有些修改无法就地完成。当 `nabi.nabion` 或 `nabi.lock` 改变（新的依赖、新的版本、其他特性）或工具链改变时，`nabi dev` 会重启程序并说明原因：

```text
  Restarting nabi.nabion changed
```

按 `r` 手动重启，例如在程序拒绝了某个修改之后；按 `q` 退出。`nabi dev` 运行期间，程序的标准输入是空的，因为按键归 `nabi dev` 使用。

## 哪些会重载

你自己的包会重载：你运行的包及其 `path` 依赖，包括工作区成员。注册表和 git 上的包属于宿主，每个会话只构建一次，所以修改它们会导致重启。热重载只构建调试代码：`nabi dev --release` 是错误。用 `nabi build` 或 `nabi run` 构建的程序不包含这些机制，也不为此付出任何代价。

## 小结

- `nabi dev` 运行程序，并在运行期间应用每次保存；编译错误时保留旧代码。
- 每帧调用一次 `std::reload::point()`。
- 新字段需要默认值；`#[renamed_from]` 保留被重命名字段的值。
- 把长期存在的状态放在 `reload::State` 中，把每帧的工作放在函数中。
- `r` 重启，`q` 退出；清单、锁文件和工具链的变更会自动重启。

## 延伸阅读

- 规范：[热重载](../spec/refs-lang/009-tools/914-hot-reload.html)、[`nabi dev`](../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-dev带热重载运行)、[`std::reload`](../spec/refs-standard-libraries/300-std/313-reload.html)
- 自己加载代码：[23. 插件](23-plugins.html)
- 工具：[16. 工具](16-tools.html)
