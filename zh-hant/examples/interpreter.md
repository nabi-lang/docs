# 小型 Lisp 直譯器

建立一個小型 Lisp：包含語彙分析器、遞迴下降剖析器，以及攜帶環境的求值器，支援 `define`、`if`、`lambda` 和閉包，並透過 `throws` 回報執行時的錯誤。

## 程式碼

```nabi
// A tiny Lisp: integers, define, if, lambda with closures.
enum Value {
    Num(i64),
    Bool(bool),
    Sym(str),
    List(List<Value>),
    Lambda(List<str>, List<Value>, Env),   // params, body, captured scope
}

// Scopes from outermost to innermost. Dicts are shared, so a closure
// sees definitions made after it was created (needed for recursion).
struct Env { frames: List<Dict<str, Value>>; }

struct LispError { msg: str; }
impl Display for LispError { fn fmt(self, ref f: Formatter) throws (FmtError) { try f.write_str(self.msg) } }
impl Debug for LispError { fn fmt(self, ref f: Formatter) throws (FmtError) { try f.write_str(self.msg) } }
impl Error for LispError {}
fn err(msg: str) -> LispError { LispError { msg: msg } }
fn tokenize(src: str) -> List<str> {
    src.replace("(", " ( ").replace(")", " ) ").split_whitespace().to_list()
}

fn parse(tokens: List<str>, ref pos: usize) throws (LispError) -> Value {
    if pos >= tokens.len() { throw err("unexpected end of input"); }
    let tok = tokens[pos];
    pos += 1;
    if tok == ")" { throw err("unexpected `)`"); }
    if tok != "(" {
        return match try? tok.parse::<i64>() {
            n: i64 => Value::Num(n),
            none => Value::Sym(tok),
        };
    }
    let items: List<Value> = [];
    while pos < tokens.len() && tokens[pos] != ")" {
        items.push(try parse(tokens, ref pos));
    }
    if pos >= tokens.len() { throw err("missing `)`"); }
    pos += 1;
    Value::List(items)
}

fn eval(x: Value, env: Env) throws (LispError) -> Value {
    let items = match x {
        Value::Sym(name) => {
            for i in (0..env.frames.len()).rev() {
                if let v = env.frames[i].get(name) { return v; }
            }
            throw err(f"undefined symbol `{name}`");
        },
        Value::List(items) => items,
        _ => return x,    // numbers and booleans evaluate to themselves
    };
    if items.is_empty() { return x; }
    let head = match items[0] { Value::Sym(s) => s, _ => "" };
    match head {
        "define" => {
            let Value::Sym(name) = items[1] else { throw err("define needs a name"); };
            let v = try eval(items[2], env);
            env.frames[env.frames.len() - 1].insert(name, v);
            return v;
        },
        "if" => {
            let truthy = match try eval(items[1], env) { Value::Bool(b) => b, _ => true };
            return try eval(items[if truthy { 2 } else { 3 }], env);
        },
        "lambda" => {
            let params: List<str> = [];
            if let Value::List(ps) = items[1] {
                for p in ps {
                    match p { Value::Sym(s) => params.push(s), _ => { throw err("parameters must be symbols"); } }
                }
            }
            return Value::Lambda(params, [items[2]], env);
        },
        _ => {},
    }
    let args: List<Value> = [];
    for i in 1..items.len() { args.push(try eval(items[i], env)); }
    if head in ["+", "-", "*", "/", "<", "="] {
        return try arith(head, args);
    }
    try apply(try eval(items[0], env), args)
}

fn arith(op: str, args: List<Value>) throws (LispError) -> Value {
    let (a, b) = match (args[0], args[1]) {
        (Value::Num(a), Value::Num(b)) => (a, b),
        _ => { throw err(f"`{op}` needs two numbers"); },
    };
    match op {
        "+" => Value::Num(a + b),
        "-" => Value::Num(a - b),
        "*" => Value::Num(a * b),
        "/" => if b == 0 { throw err("division by zero"); } else { Value::Num(a / b) },
        "<" => Value::Bool(a < b),
        _ => Value::Bool(a == b),
    }
}

fn apply(f: Value, args: List<Value>) throws (LispError) -> Value {
    let Value::Lambda(params, body, captured) = f else { throw err(f"{show(f)} is not a function"); };
    if params.len() != args.len() { throw err(f"expected {params.len()} arguments, got {args.len()}"); }
    let frame = Dict::new();   // on top of the captured scopes
    for (p, a) in params.zip(args.iter()) { frame[p] = a; }
    try eval(body[0], Env { frames: captured.frames + [frame] })
}

fn show(v: Value) -> str {
    match v {
        Value::Num(n) => f"{n}",
        Value::Bool(b) => if b { "#t" } else { "#f" },
        Value::Sym(s) => s,
        Value::List(items) => f"<list of {items.len()}>",
        Value::Lambda(params, _, _) => f"<lambda/{params.len()}>",
    }
}

let global = Env { frames: [Dict::new()] };
let program = [
    "(define square (lambda (x) (* x x)))",
    "(square 12)",
    "(define fact (lambda (n) (if (< n 2) 1 (* n (fact (- n 1))))))",
    "(fact 20)",
    "(define make-adder (lambda (k) (lambda (x) (+ x k))))",
    "(define add5 (make-adder 5))",
    "(add5 37)",
    "(/ 10 0)",
    "(undefined-thing 1)",
    "(5 3)",
    "(+ 1 (* 2 3)",
];

for src in program {
    let pos = 0;
    let result = try {
        show(try eval(try parse(tokenize(src), ref pos), global))
    } catch e: LispError {
        f"error: {e}"
    };
    println(f"{src}\n  => {result}");
}
```

```output
(define square (lambda (x) (* x x)))
  => <lambda/1>
(square 12)
  => 144
(define fact (lambda (n) (if (< n 2) 1 (* n (fact (- n 1))))))
  => <lambda/1>
(fact 20)
  => 2432902008176640000
(define make-adder (lambda (k) (lambda (x) (+ x k))))
  => <lambda/1>
(define add5 (make-adder 5))
  => <lambda/1>
(add5 37)
  => 42
(/ 10 0)
  => error: division by zero
(undefined-thing 1)
  => error: undefined symbol `undefined-thing`
(5 3)
  => error: 5 is not a function
(+ 1 (* 2 3)
  => error: missing `)`
```

## 說明

- 程式和值共用同一個列舉 `Value`。由於 Lisp 的程式碼和資料形狀相同，剖析器產生 `Value`，求值器接受一個 `Value` 並回傳一個 `Value`（[和型別](../learn/05-sum-types.html)、[結構、列舉與 trait](../learn/06-structs-enums-traits.html)）。
- 剖析器透過 `ref pos: usize` 與呼叫端共用讀取位置。它是典型的遞迴下降剖析器，讀取括號內的內容時會呼叫自己（[4.7 `ref` 關鍵字](../spec/refs-lang/004-functions/407-ref-keyword.html)）。
- 不是整數的語彙單元都是符號。`try? tok.parse::<i64>()` 把剖析錯誤轉成 `none`，所以用有兩個分支的 `match` 就能區分兩種情況（[3.14.3 `try?` 與 `try!`](../spec/refs-lang/003-expressions/314-error-control-flow.html#3143-try-與-try)）。只接受一種變體的地方，用 `let Value::Sym(name) = items[1] else { throw ...; };` 這樣的 [let-else](../spec/refs-lang/003-expressions/316-let-else.html) 繫結它，否則離開函式。
- 執行時的錯誤會 `throw` 一個 `LispError`，每個函式都在簽章中以 `throws (LispError)` 寫明這一點。底部的 `try { ... } catch e: LispError { ... }` 把一行的錯誤轉成訊息，所以下一行仍會執行（[錯誤](../learn/10-errors.html)）。
- `lambda` 擷取它被建立時所在的 `Env`。`Dict` 是共用的參考，所以 `make-adder` 回傳的函式記得 `k`，而 `fact` 即使在之後才定義，也能找到自己。

> [!NOTE]
> 直接包含自身的列舉（`Lambda(..., Value, ...)`）大小會是無限的，所以編譯器會拒絕它；因此 lambda 主體改存放在 `List<Value>` 中。
