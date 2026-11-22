# 文字冒險

一個小型文字冒險遊戲，有房間、出口、物品和一扇上鎖的門，依照預先寫好的指令清單一路玩到結局。

## 程式碼

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

## 說明

- 玩家的輸入先剖析成 `Command` 列舉，遊戲邏輯只處理這個列舉。字串處理與狀態變更彼此分開，所以新增指令很容易，而編譯器會檢查 `match` 是否涵蓋每個變體（[和型別](../learn/05-sum-types.html)、[3.12 match](../spec/refs-lang/003-expressions/312-match-expression.html)）。
- `"n" | "north" => ...` 這類 or 模式把別名歸為一組，`"go"` 則對剩下的字再呼叫一次 `parse`，所以 `go west` 和 `west` 會成為同一個指令（[3.11 模式](../spec/refs-lang/003-expressions/311-patterns.html#or-模式)）。
- `Dict::get` 回傳 `usize | none`。分成 `next: usize => ...` 和 `none => ...` 兩個分支，自然就處理了「那個方向沒有出口」的情況（[`Dict`](../../std/core/collections/dict.html#struct.Dict)）。
- 遊戲狀態 `Game` 透過 `step(ref self, ...)` 就地改變。房間的 `items` 是 `List`，所以用 `let items = self.rooms[self.here].items;` 取出後呼叫 `remove`，也會改變房間本身的串列（[集合型別與迭代器](../learn/08-collections-iterators.html)）。
- `script` 串列一次送入一行，代替標準輸入。想做成真正的遊戲，可以把這個迴圈換成 `while let line = try io::input("> ") { .. }`，它會一直讀取行，直到輸入結束（[I/O](../learn/13-io.html)）。
