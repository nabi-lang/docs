# 文字冒险游戏

一个有房间、出口、物品和一扇锁着的门的小型文字冒险游戏，按一份预先写好的命令列表一直玩到结局。

## 代码

```nabi
enum Dir { North, South, East, West }

enum Command {
    Go(Dir),
    Take(str),
    Use(str),
    Look,
    Inventory,
    Unknown(str),
}

struct Room {
    name: str;
    desc: str;
    exits: Dict<str, usize>;    // direction name -> room index
    items: List<str>;
}

struct Game {
    rooms: List<Room>;
    here: usize;
    bag: List<str>;
    door_open: bool;
    won: bool;
}

fn dir_name(d: Dir) -> str {
    match d { Dir::North => "north", Dir::South => "south", Dir::East => "east", Dir::West => "west" }
}

fn parse(line: str) -> Command {
    let words = line.trim().to_lowercase().split_whitespace().to_list();
    if words.is_empty() { return Command::Unknown(""); }
    let arg = if words.len() > 1 { words[1] } else { "" };
    match words[0] {
        "n" | "north" => Command::Go(Dir::North),
        "s" | "south" => Command::Go(Dir::South),
        "e" | "east" => Command::Go(Dir::East),
        "w" | "west" => Command::Go(Dir::West),
        "go" => parse(arg),
        "take" | "get" => Command::Take(arg),
        "use" => Command::Use(arg),
        "look" | "l" => Command::Look,
        "i" | "inventory" => Command::Inventory,
        _ => Command::Unknown(line),
    }
}

impl Game {
    fn describe(self) {
        let room = self.rooms[self.here];
        println(f"== {room.name} ==\n{room.desc}");
        if !room.items.is_empty() { println(f"You see: {room.items:?}"); }
    }

    fn step(ref self, cmd: Command) {
        match cmd {
            Command::Go(d) => {
                let way = dir_name(d);
                match self.rooms[self.here].exits.get(way) {
                    next: usize => {
                        if self.rooms[next].name == "Vault" && !self.door_open {
                            println("The vault door is locked.");
                        } else {
                            self.here = next;
                            self.describe();
                        }
                    },
                    none => println(f"You can't go {way}."),
                }
            },
            Command::Take(item) => {
                let items = self.rooms[self.here].items;
                match items.position(|x| x == item) {
                    i: usize => {
                        self.bag.push(items.remove(i));
                        println(f"Taken: {item}.");
                    },
                    none => println(f"There is no {item} here."),
                }
            },
            Command::Use(item) => {
                if item !in self.bag {
                    println(f"You don't have a {item}.");
                } else if item == "key" && self.rooms[self.here].name == "Hall" {
                    self.door_open = true;
                    println("The key turns. The vault door swings open.");
                } else if item == "lamp" && self.rooms[self.here].name == "Vault" {
                    self.won = true;
                    println("The lamp reveals a pile of gold. You win!");
                } else {
                    println(f"Nothing happens.");
                }
            },
            Command::Look => self.describe(),
            Command::Inventory => println(f"You carry: {self.bag:?}"),
            Command::Unknown(text) => println(f"I don't understand \"{text}\"."),
        }
    }
}

fn room(name: str, desc: str, exits: Dict<str, usize>, items: List<str>) -> Room {
    Room { name: name, desc: desc, exits: exits, items: items }
}

let game = Game {
    rooms: [
        room("Cellar", "Damp stone walls. Stairs lead north.", {"north": 1}, ["lamp"]),
        room("Hall", "A long hall with a heavy door to the east.", {"south": 0, "west": 2, "east": 3}, []),
        room("Library", "Dusty shelves everywhere.", {"east": 1}, ["book", "key"]),
        room("Vault", "It is pitch dark.", {"west": 1}, []),
    ],
    here: 0,
    bag: [],
    door_open: false,
    won: false,
};

// Scripted input instead of stdin.
let script = ["look", "take lamp", "north", "east", "go west", "take key", "take sword",
              "dance", "e", "use key", "e", "i", "use lamp"];

game.describe();
for line in script {
    if game.won { break; }
    println(f"\n> {line}");
    game.step(parse(line));
}
```

```output
== Cellar ==
Damp stone walls. Stairs lead north.
You see: ["lamp"]

> look
== Cellar ==
Damp stone walls. Stairs lead north.
You see: ["lamp"]

> take lamp
Taken: lamp.

> north
== Hall ==
A long hall with a heavy door to the east.

> east
The vault door is locked.

> go west
== Library ==
Dusty shelves everywhere.
You see: ["book", "key"]

> take key
Taken: key.

> take sword
There is no sword here.

> dance
I don't understand "dance".

> e
== Hall ==
A long hall with a heavy door to the east.

> use key
The key turns. The vault door swings open.

> e
== Vault ==
It is pitch dark.

> i
You carry: ["lamp", "key"]

> use lamp
The lamp reveals a pile of gold. You win!
```

## 讲解

- 玩家的输入先被解析成 `Command` 枚举，游戏逻辑只处理这个枚举。字符串处理和状态变化分开，所以添加新命令很容易，编译器也会检查 `match` 是否覆盖了每个变体（[和类型](../learn/05-sum-types.html)，[3.12 match](../spec/refs-lang/003-expressions/312-match-expression.html)）。
- `"n" | "north" => ...` 这样的或模式把别名归为一组，`"go"` 对剩下的单词再次调用 `parse`，所以 `go west` 和 `west` 成为同一个命令（[3.11 模式](../spec/refs-lang/003-expressions/311-patterns.html#或模式)）。
- `Dict::get` 返回 `usize | none`。分成 `next: usize => ...` 和 `none => ...` 两个分支，就自然地处理了“那个方向没有出口”的情况（[`Dict`](../../std/core/collections/dict.html#struct.Dict)）。
- 游戏状态 `Game` 通过 `step(ref self, ...)` 就地修改。房间的 `items` 是 `List`，所以用 `let items = self.rooms[self.here].items;` 取出它再调用 `remove`，也会修改房间自己的列表（[集合与迭代器](../learn/08-collections-iterators.html)）。
- `script` 列表代替标准输入，一次送入一行。要做成真正的游戏，把这个循环换成 `while let line = try io::input("> ") { .. }`，它会逐行读取，直到输入结束（[I/O](../learn/13-io.html)）。
