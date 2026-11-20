# 26. Hot Reload

`nabi dev` runs your program and applies your edits while it runs. Save a file, and a few moments later the running program uses the new code, with everything it had built up still there: the level you were playing, the window you were tuning, the data you loaded. It is made for programs you watch while you work on them, such as games, simulations and tools with a user interface.

This chapter runs a small game loop under `nabi dev`, changes a function, then changes a type.

## Starting a session

Take a package with a loop in `main`:

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

Run it with `nabi dev` instead of `nabi run`:

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

`std::reload::point()` marks a good place to take new code: between frames, with no half-done work. Outside `nabi dev` it does nothing, so the same program runs unchanged under `nabi run`.

## Changing a function

Edit `draw` and save:

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

Only `game` was rebuilt, and only what changed in it was compiled again. The loop in `main` kept running; its next call to `draw` went to the new body.

A save that does not compile changes nothing in the program:

```text
error[E0034]: no field `hpp` on `Player`
 --> src/main.nabi:9:28
        Kept the running code of `game`: fix the error and save
```

Fix it, save again, and the change goes in.

## Changing a type

Give `Player` a new field. While the program runs, every `Player` it has must get a value for it, so a new field needs a default:

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

The `Player` that `main` made before the edit now has `mana: 5`. Without the default, the build stops with what to write:

```text
error: new field `mana` needs a default while the program runs: write `mana: i32 = 0;` (or restart with `r`)
```

Removing a field, reordering fields and adding enum variants all apply. Renaming keeps the value when you say where it came from:

```nabi norun
struct Player {
    name: str;
    #[renamed_from("hp")]
    health: i32;
}
```

Changing a field's type cannot carry the old value over; the error says to give the new field its own name with a default, or to restart.

## Where state lives

A layout change can only apply when no running function holds a value of the changed type in a local variable. `main`'s loop holds `p` the whole time, so when `Player` changes, the change waits and says why:

```text
     Waiting game v3: `main` (src/main.nabi:13) holds a `Player` across `reload::point()`; keep it in a `reload::State`: `let p = reload::State::new(Player { .. });`
```

Keep long-lived state in a `std::reload::State`. It is a box that the reload knows how to update; outside `nabi dev` it is a plain box:

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

Two more things to know:

- `main`'s own loop is old code for the whole session. Put the work of each frame in functions that `main` calls, as `draw` is here.
- A closure keeps the body it was made with. New closures, made by new code, use new bodies.

To run code after each reload, for example to rebuild a cache, register a hook. Hooks are kept by name, so registering again from new code replaces the old one:

```nabi norun
reload::on_reload("cache", |r| println(f"reloaded v{r.version}"));
```

## Restarting

Some changes cannot be made in place. `nabi dev` restarts the program, and says why, when `nabi.nabion` or `nabi.lock` changes (a new dependency, a new version, other features) or when the toolchain changes:

```text
  Restarting nabi.nabion changed
```

Press `r` to restart by hand, for example after a change the program refused, and `q` to quit. The program's standard input is empty while `nabi dev` runs, because the keys are `nabi dev`'s.

## What reloads

Your own packages reload: the package you run and its `path` dependencies, including workspace members. Registry and git packages are part of the host, which is built once per session, so changing them restarts. Hot reload builds debug code only: `nabi dev --release` is an error. Programs built with `nabi build` or `nabi run` have none of this machinery and pay nothing for it.

## Summary

- `nabi dev` runs the program and applies each save while it runs; a compile error keeps the old code.
- Call `std::reload::point()` once per frame.
- A new field needs a default; `#[renamed_from]` keeps a renamed field's value.
- Keep long-lived state in `reload::State`, and the per-frame work in functions.
- `r` restarts, `q` quits; manifest, lock and toolchain changes restart by themselves.

## Further reading

- Spec: [Hot reload](../spec/refs-lang/009-tools/914-hot-reload.html), [`nabi dev`](../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-dev-run-with-hot-reload), [`std::reload`](../spec/refs-standard-libraries/300-std/313-reload.html)
- Loading code yourself: [23. Plugins](23-plugins.html)
- Tools: [16. Tools](16-tools.html)
