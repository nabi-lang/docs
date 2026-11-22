# JSON 解析器

用枚举表示 JSON 值，用一个字符级的递归下降解析器通过 `throws` 报告错误位置，再加上一个带缩进的美化输出器。

## 代码

```nabi
// A small JSON parser: recursive descent over chars, errors with positions, pretty printer.

enum Json { Null, Bool(bool), Num(f64), Str(str), Arr(List<Json>), Obj(Dict<str, Json>) }

struct ParseError { msg: str; pos: usize; }

struct Parser { cs: List<char>; pos: usize = 0; }

impl Parser {
    fn peek(self) -> char | none { self.cs.get(self.pos) }

    fn fail(self, msg: str) -> ParseError { ParseError { msg: msg, pos: self.pos } }

    fn skip_ws(ref self) {
        while self.peek()?.is_whitespace() ?? false { self.pos += 1; }
    }

    // Skip whitespace, then consume `c` if it is next.
    fn eat(ref self, c: char) -> bool {
        self.skip_ws();
        if self.peek() != c { return false; }
        self.pos += 1;
        true
    }

    fn expect(ref self, c: char) throws (ParseError) {
        if !self.eat(c) { throw self.fail(f"expected '{c}'"); }
    }

    fn value(ref self) throws (ParseError) -> Json {
        self.skip_ws();
        let c = self.peek() else { throw self.fail("unexpected end of input"); };
        match c {
            'n' => try self.word("null", Json::Null),
            't' => try self.word("true", Json::Bool(true)),
            'f' => try self.word("false", Json::Bool(false)),
            '"' => Json::Str(try self.string()),
            '[' => try self.array(),
            '{' => try self.object(),
            _ => if c == '-' || c.is_ascii_digit() { try self.number() } else { throw self.fail(f"unexpected '{c}'"); },
        }
    }

    fn word(ref self, w: str, v: Json) throws (ParseError) -> Json {
        for c in w.chars() {
            if self.peek() != c { throw self.fail(f"invalid literal, expected {w}"); }
            self.pos += 1;
        }
        v
    }

    fn number(ref self) throws (ParseError) -> Json {
        let start = self.pos;
        let sb = StrBuf::new();
        while let c = self.peek() {
            if !(c.is_ascii_digit() || "+-.eE".contains(f"{c}")) { break; }
            sb.push(c);
            self.pos += 1;
        }
        let text = sb.to_str();
        let n = try text.parse::<f64>() catch _ { throw ParseError { msg: f"bad number '{text}'", pos: start }; };
        Json::Num(n)
    }

    fn string(ref self) throws (ParseError) -> str {
        self.pos += 1; // opening quote
        let sb = StrBuf::new();
        loop {
            let c = self.peek() else { throw self.fail("unterminated string"); };
            self.pos += 1;
            if c == '"' { break; }
            if c != '\\' { sb.push(c); continue; }
            let e = self.peek() else { throw self.fail("unterminated string"); };
            self.pos += 1;
            match e {
                'n' => sb.push('\n'),
                't' => sb.push('\t'),
                '"' | '\\' | '/' => sb.push(e),
                _ => throw ParseError { msg: f"unknown escape '\\{e}'", pos: self.pos - 2 },
            }
        }
        sb.to_str()
    }

    fn array(ref self) throws (ParseError) -> Json {
        self.pos += 1;
        let items: List<Json> = [];
        if self.eat(']') { return Json::Arr(items); }
        loop {
            items.push(try self.value());
            if self.eat(']') { return Json::Arr(items); }
            try self.expect(',');
        }
    }

    fn object(ref self) throws (ParseError) -> Json {
        self.pos += 1;
        let fields: Dict<str, Json> = Dict::new();
        if self.eat('}') { return Json::Obj(fields); }
        loop {
            self.skip_ws();
            if self.peek() != '"' { throw self.fail("expected a string key"); }
            let key = try self.string();
            try self.expect(':');
            fields[key] = try self.value();
            if self.eat('}') { return Json::Obj(fields); }
            try self.expect(',');
        }
    }
}

fn parse(text: str) throws (ParseError) -> Json {
    let p = Parser { cs: text.chars().to_list() };
    let v = try p.value();
    p.skip_ws();
    if p.pos < p.cs.len() { throw p.fail("trailing characters"); }
    v
}

// Render `parts` one per line between `open` and `close`.
fn block(open: str, close: str, parts: List<str>, depth: usize) -> str {
    if parts.is_empty() { return f"{open}{close}"; }
    let sb = StrBuf::new();
    sb.push_str(open);
    for (i, p) in parts.enumerate() {
        let comma = if i + 1 < parts.len() { "," } else { "" };
        sb.push_str(f"\n{"  ".repeat(depth + 1)}{p}{comma}");
    }
    sb.push_str(f"\n{"  ".repeat(depth)}{close}");
    sb.to_str()
}

fn pretty(v: Json, depth: usize) -> str {
    match v {
        Json::Null => "null",
        Json::Bool(b) => f"{b}",
        Json::Num(n) => f"{n}",
        Json::Str(s) => f"{s:?}",
        Json::Arr(items) => block("[", "]", items.map(|x| pretty(x, depth + 1)).to_list(), depth),
        Json::Obj(fields) => block("{", "}", fields.map(|(k, x)| f"{k:?}: {pretty(x, depth + 1)}").to_list(), depth),
    }
}

fn main() {
    let doc = "{\"name\": \"nabi\", \"tags\": [\"fast\", \"safe\"], \"version\": 0.1, \"stars\": 1.2e3,
               \"stable\": false, \"deps\": {}, \"notes\": null, \"quote\": \"say \\\"hi\\\"\"}";
    let v = try parse(doc) catch e { println(f"error: {e.msg} at {e.pos}"); return; };
    println(pretty(v, 0));

    for bad in ["[1, 2,]", "{\"a\" 1}", "[tru]", "\"open", "[1] x", "-"] {
        try parse(bad) catch e { println(f"{bad:<9} -> {e.msg} at offset {e.pos}"); };
    }
}
```

```output
{
  "name": "nabi",
  "tags": [
    "fast",
    "safe"
  ],
  "version": 0.1,
  "stars": 1200,
  "stable": false,
  "deps": {},
  "notes": null,
  "quote": "say \"hi\""
}
[1, 2,]   -> unexpected ']' at offset 6
{"a" 1}   -> expected ':' at offset 5
[tru]     -> invalid literal, expected true at offset 4
"open     -> unterminated string at offset 5
[1] x     -> trailing characters at offset 4
-         -> bad number '-' at offset 0
```

## 讲解

- JSON 值是有六个变体的枚举 `Json`。数组和对象以 `List<Json>` 和 `Dict<str, Json>` 递归地持有自身；`Dict` 保持键的顺序（[和类型](../learn/05-sum-types.html)，[结构体、枚举与 trait](../learn/06-structs-enums-traits.html)）。
- 解析器是一个持有位置 `pos` 的结构体，推进位置的方法接受 `ref self`。这样的方法返回（或抛出错误）时，更新后的 `self` 会写回调用方（[4.7.5 方法接收者](../spec/refs-lang/004-functions/407-ref-keyword.html#475-exception-for-method-receivers)）。
- 输入结束用 `let c = self.peek() else { throw ...; };` 检查。`peek()` 返回 `char | none`，所以遇到 `none` 时 `let-else` 离开，在它下面 `c` 是 `char`（[3.16 `let-else`](../spec/refs-lang/003-expressions/316-let-else.html#3161-form)）。
- 所有错误都是同一个携带消息和位置的 `ParseError`。标准库抛出的错误，例如数字解析失败，用 `catch _ { throw ParseError { ... }; }` 转换成带位置的错误（[错误处理](../learn/10-errors.html)）。
- 输出由返回字符串的递归函数 `pretty` 构造。对于数组和对象，先渲染每个元素，再由 `block` 加上缩进和逗号；字符串值用 `{s:?}`（Debug）输出引号和转义（[字符串](../learn/09-strings.html)）。
