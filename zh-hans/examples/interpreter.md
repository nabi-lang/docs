# 一个小型 Lisp 解释器

构建一个小型 Lisp，包含词法分析器、递归下降解析器和带环境的求值器，支持 `define`、`if`、`lambda` 和闭包，并通过 `throws` 报告运行时错误。

## 代码

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

## 讲解

- 程序和值共用一个枚举 `Value`。Lisp 的代码和数据形状相同，所以解析器产生 `Value`，求值器接受一个 `Value` 并返回一个 `Value`（[和类型](../learn/05-sum-types.html)，[结构体、枚举与 trait](../learn/06-structs-enums-traits.html)）。
- 解析器通过 `ref pos: usize` 与调用方共享读取位置。它是一个经典的递归下降解析器，调用自身来读取括号中的内容（[4.7 `ref` 关键字](../spec/refs-lang/004-functions/407-ref-keyword.html)）。
- 不是整数的词法单元都是符号。`try? tok.parse::<i64>()` 把解析错误变成 `none`，所以一个有两个分支的 `match` 就能区分这两种情况（[3.14.3 `try?` 与 `try!`](../spec/refs-lang/003-expressions/314-error-control-flow.html#3143-try-与-try)）。在只接受一个变体的地方，`let Value::Sym(name) = items[1] else { throw ...; };` 这样的 [let-else](../spec/refs-lang/003-expressions/316-let-else.html) 要么绑定它，要么离开函数。
- 运行时错误 `throw` 一个 `LispError`，每个函数都在签名中用 `throws (LispError)` 写明这一点。底部的 `try { ... } catch e: LispError { ... }` 把一行的错误变成消息，所以下一行仍然会运行（[错误处理](../learn/10-errors.html)）。
- `lambda` 捕获创建它时的 `Env`。`Dict` 是共享的引用，所以 `make-adder` 返回的函数记得 `k`，而 `fact` 即使是之后才定义的，也能找到自身。

> [!NOTE]
> 直接包含自身的枚举（`Lambda(..., Value, ...)`）大小是无限的，所以编译器会拒绝它；lambda 的主体改为存放在 `List<Value>` 中。
