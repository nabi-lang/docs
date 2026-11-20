# 18. Features and Publishing

[Chapter 11](11-modules-packages.html) built a package and added dependencies. This chapter goes further: **features**, which let the users of a library turn parts of it on and off; the packages the Nabi project maintains; and publishing your own library to the package registry, [packages.nabi-lang.org](https://packages.nabi-lang.org).

## Defining features

A feature is a name in the `features` dict of `nabi.nabion`, mapped to the list of what it turns on. In code, an enabled feature is the condition `feature = "<name>"` ([chapter 17](17-conditional-compilation.html)). Here a `geom` library has a `circle` feature, on by default, and a `pretty` feature that pulls in an optional dependency.

```nabion
Package {
    name: "geom",
    version: "0.1.0",
    deps: {
        "fancy": Dep::path("../fancy", optional: true),
    },
    features: {
        "default": ["circle"],
        "circle": [],
        "pretty": ["dep:fancy"],
    },
}
```

```nabi norun
// geom/src/lib.nabi
pub fn shapes() -> List<str> {
    let out: List<str> = ["square"];
    #[cfg(feature = "circle")]
    out.push("circle");
    out
}

#[cfg(feature = "pretty")]
pub fn show(s: str) -> str { fancy::frame(s) }

#[cfg(not(feature = "pretty"))]
pub fn show(s: str) -> str { s }
```

Each entry of a feature's list is one of:

| Entry       | Turns on                                                            |
|-------------|---------------------------------------------------------------------|
| `"circle"`  | Another feature of this package                                     |
| `"dep:fancy"` | The optional dependency `fancy`                                   |
| `"http/tls"` | Feature `tls` of the dependency `http` (and `http` itself, if it is optional) |
| `"http?/tls"` | Feature `tls` of `http`, only if something else already turned `http` on |

- `default` is an ordinary feature that is on unless the user turns it off.
- An optional dependency (`optional: true`) is part of the build only when an enabled feature names it with `dep:` or `<dep>/<f>`. It does not get a feature of its own name automatically, and an optional dependency no feature enables is an error. `nabi add fancy --path ../fancy --optional` writes both the dependency and a `"fancy": ["dep:fancy"]` feature.
- Naming a feature or dependency that does not exist, using `dep:` on a non-optional dependency, and cycles are errors, reported with the line and column in `nabi.nabion`.

Feature names are lowercase words that say what they enable (`http1`, `tls`, `pretty`), not `with-x` or `no-x`: features only ever add.

## Enabling features

The package being built starts from its `default` feature. `--features` adds more, either its own (`fancy`) or a direct dependency's (`geom/pretty`); `--no-default-features` and `--all-features` do what they say. They work with `run`, `build`, `test`, `check` and the other build commands.

```text
$ nabi run
square
circle
$ nabi run --features geom/pretty
*** square ***
*** circle ***
$ nabi run --no-default-features
square
circle
```

The last one still prints `circle`: `--no-default-features` turns off the defaults of the package being built (`app`), not those of `geom`. A dependency's features are chosen where you depend on it.

```nabion
Package {
    name: "app",
    version: "0.1.0",
    deps: {
        "geom": Dep::path("../geom", default_features: false, features: ["pretty"]),
    },
    features: {
        "fancy": ["geom/pretty"],
    },
}
```

Features are **unified**: each package has one feature set for the whole build, the union of everything every dependent asks of it. So if two libraries both use `geom`, one with `pretty` and one without, both get `pretty`. That is why a feature must only add API and never change what existing code does. `nabi deps` shows the result as `--extern-feature` lines.

The lockfile pins every package of the graph, optional ones included, so switching features never changes `nabi.lock`.

## The Nabi packages

The Nabi project maintains a few libraries as packages of their own, outside the standard library. Each is pure Nabi with no dependencies beyond `std`.

| Package      | What it is                                                                    |
|------------|-------------------------------------------------------------------------------|
| `json`     | JSON (RFC 8259): a `Value` tree, a strict parser with `line:col` errors, a streaming tokenizer, typed encode / decode with `#[derive(json::Json)]`, JSON Pointer |
| `toml`     | TOML 1.1.0: a `Value` tree with typed date-times, a parser with `line:col` errors, a writer, typed encode / decode with `#[derive(toml::Toml)]` |
| `csv`      | CSV (RFC 4180 plus dialects): a streaming reader over a `Read`, a `str` or bytes, typed rows with `#[derive(Csv)]`, a writer with quoting styles, errors with line and column |
| `http`     | HTTP/1.1 and HTTP/2 client and server: strict parsing, HPACK, streaming bodies, connection pooling, `https` over std's TLS. Features `http1`, `http2`, `tls`, all on by default |
| `markdown` | CommonMark plus GitHub extensions (tables, callouts, footnotes, heading anchors) to HTML. It renders this site |
| `spatial`  | Game and graphics math: `Vec2/3/4`, `Quat`, `Mat2/3/4`, affine transforms, projections and shapes, generic over `f32` and `f64`. Every type is `#[repr(C)]`; `Mat4` products run on SIMD lanes |
| `nabi-syntax` | A tokenizer and HTML highlighter for Nabi and NABION source. It highlights the code on this site |

`json` and `toml` follow the shape of std's [`nabion`](15-nabion.html) module: `parse` / `to_str` / `from_str`, plus `to_value` / `from_value`, and a derive for your own types.

```nabi norun
#[derive(json::Json)]
struct Server { host: str; port: u16 = 80; }

let s: Server = try json::from_str("{\"host\": \"localhost\"}");
println(json::to_str(s));      // {"host":"localhost","port":80}
```

A package's derive is written with the package's name, `#[derive(json::Json)]`, or bare after `use json::Json;`. What the derive expands to is the package's own code: it may call the package's internal helpers, which your code cannot.

They are added like any other package. Without a requirement, `nabi add` writes `^` and the newest version:

```bash
nabi add json
nabi add http --no-default-features --features http1,tls
```

```nabion
Package { name: "app", version: "0.1.0", deps: {
    "json": Dep::version("^0.1.0"),
    "http": Dep::version("^0.1.0", default_features: false, features: ["http1", "tls"]),
} }
```

In a script, name the file last: `nabi add json app.nabi` writes the dependency into the script's `//!` block ([chapter 11](11-modules-packages.html#single-file-scripts-and-environments)).

With `http`, an HTTPS request is a few lines. The package opens the TCP connection, runs the TLS 1.3 handshake from [`std::net::tls`](19-crypto-and-tls.html) against the OS trust store, offers HTTP/2 through ALPN and falls back to HTTP/1.1:

```nabi norun
let r = try await http::get("https://example.com/");
println(f"{r.status} {r.version:?} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

## Publishing a package

Publishing adds one line to the registry index, pointing at a commit of your git repository. You upload no source: the registry fetches that commit, keeps an archive of it and writes the archive's SHA-256 into the line (`archive`), so users download the archive and the version outlives your repository. The package must have a `src/lib.nabi` (a library), a `src/main.nabi` (a program for `nabi install`) or both, and a readme, at the root of a clean git work tree with a remote `origin`, and every normal dependency must itself come from the registry: git dependencies are rejected, and so is a path dependency unless it also names the version to publish in its place, `Dep::path("../geom", version: "^0.1")` (publish `geom` first). Path and git `dev_deps` without a `version:` are dropped. Set `nabi_version` to the oldest nabi you have built the package with: `nabi publish` needs it, and resolution gives each user the newest version their toolchain builds (`"0.1.3"` admits nabi 0.1.3 and later 0.1.x, [chapter 1](01-getting-started.html#managing-toolchains)). `nabi new` writes the version that made the package. Fill in `description`, `license` and `repository`: they show up in `nabi search` and on the website. The other fields a package page shows are optional, as in Cargo:

```nabion
Package {
    name: "textwrap",
    version: "0.1.0",
    nabi_version: "0.1.0",
    description: "Word wrapping for terminals",
    license: "MIT OR Apache-2.0",
    repository: "https://github.com/you/textwrap",
    authors: ["You <you@example.com>"],
    homepage: "https://example.com/textwrap",
    documentation: "https://example.com/textwrap/docs",
    keywords: ["text", "terminal"],
    categories: ["text-processing"],
    exclude: ["bench/**"],
    deps: {},
}
```

`keywords` and `categories` take at most 5 entries each; `include` / `exclude` choose the files the package holds; `license_file`, `readme`, `links`, `default_run` and a free-form `metadata` table are there too. A field `nabi` does not know is a warning, not an error, and is still sent to the registry.

> [!IMPORTANT]
> For now only the registry's maintainers can publish. Anyone can sign in and use packages, but `nabi login` refuses an account that cannot publish yet.

### `nabi login`

Sign in at [packages.nabi-lang.org/me](https://packages.nabi-lang.org/me) with GitHub and create an account token there. Then hand it to `nabi`:

```text
$ nabi login
a token from https://packages.nabi-lang.org/me (not shown; Ctrl-C to cancel):
   Logged in as your-name at https://packages.nabi-lang.org (token in ~/.nabi/tokens.nabion)
```

Without an argument, the token is read from standard input without echoing. `nabi login` asks the registry who the token belongs to and refuses a token it rejects, saving nothing. A good token is kept in `~/.nabi/tokens.nabion` (readable only by you), filed under the index it was made for, and is sent only to that index; `nabi login --registry <url>` keeps one for another index. `nabi logout` removes it, and `nabi logout --all` removes every index's token. In CI, set `NABI_REGISTRY_TOKEN` instead; it is sent only to packages.nabi-lang.org.

### `nabi publish`

`nabi publish --dry-run` prints the index entry it would add, and nothing leaves your machine. `nabi publish` publishes, as `cargo publish` does.

```text
$ nabi publish --dry-run
Package { name: "textwrap", version: "0.1.0", deps: [], source: Dep::git("https://github.com/you/textwrap.git", rev: "8e2afb20.."), tree: "823a3592..", yanked: false, nabi_version: "0.1.0" }
note: dry run; `nabi publish` sends it to the registry API of https://github.com/nabi-lang/packages-index (needs `nabi login`), which adds it as te/xt/textwrap
```

The entry holds only what resolving needs: the HEAD commit and its tree hash, the dependencies, and `nabi_version`. Push that commit first. The description, license and other metadata travel beside it to the registry, which shows them with who published each version. `nabi publish` sends both with your token (`Published textwrap 0.1.0 to https://packages.nabi-lang.org`), and the version is live once the index has it. A version cannot be published twice, and a name differing only in case or `-` / `_` from an existing package is taken.

The registry also makes the version's API docs from the source at that commit, reading it without building or running it: [packages.nabi-lang.org/packages/textwrap/0.1.0/docs](https://packages.nabi-lang.org/packages/textwrap/0.1.0/docs), and `/packages/textwrap/latest/docs` for the newest version. The pages are laid out as `nabi doc` lays them out: the root module at `docs`, each other public module on a page of its own (`docs/<module>`), and the same anchors (`docs#fn.wrap`). An item you re-export from a private module with `pub use` is shown there as if it were declared where you re-export it. If the docs cannot be made, the version is published anyway: `nabi publish` prints the reason as a warning, and the docs page shows it.

### `nabi yank`

A version with a serious bug can be yanked. New resolutions skip it, but projects whose `nabi.lock` already pins it keep building, so yanking never breaks anyone. `--undo` reverses it.

```bash
nabi yank textwrap@0.1.0 --message "wraps CJK text wrong; use 0.1.1"
nabi yank textwrap@0.1.0 --undo
```

`--message` says why; the registry shows it on the package's page.

### Changing a published API

When a function, method, field or type of your package should go away, mark it `#[deprecated]` instead of removing it. Code that uses it keeps building and gets a warning that says what to write instead; remove it in the next breaking release.

```nabi norun
#[deprecated(since = "0.3.0", note = "use `wrap_width`")]
pub fn wrap(s: str) -> str { wrap_width(s, 80) }
```

```text
warning[E0131]: use of deprecated function `wrap`
  --> src/main.nabi:6:9
   |
 6 | println(textwrap::wrap("hi"));
   |         ^^^^^^^^^^^^^^^^^^^^
   |
   = note: deprecated since 0.3.0
   = note: use `wrap_width`
   = note: `#[warn(deprecated)]` on by default
```

Two attributes keep room to grow. `#[non_exhaustive]` on an enum makes a `match` in other packages end with `_ =>`, so you can add variants later. On a struct, it marks a record you may add fields to: other packages still build it with a literal, leaving out fields that have defaults, but a pattern on it ends in `..`. Give every field you add later a default, so those literals keep compiling.

```nabi norun
#[non_exhaustive]
pub struct WrapOptions {
    pub width: i32 = 80;
    pub break_words: bool = false;
}

// in another package
let o = textwrap::WrapOptions { width: 60 };
let textwrap::WrapOptions { width, .. } = o;
```

`#[sealed]` on a trait means only your package implements it. Other packages still use it in bounds and call its methods, but an `impl` of it elsewhere is an error, so you may rely on knowing every implementation.

## Summary

- `features` in `nabi.nabion` maps names to entries: other features, `dep:<d>`, `<d>/<f>`, `<d>?/<f>`. `default` is on unless turned off.
- Optional dependencies (`optional: true`) are built only when a feature enables them.
- Choose features with `--features`, `--no-default-features`, `--all-features`, and per dependency with `features:` / `default_features: false`. Features are unified across the build, so they must only add.
- Code tests them with `#[cfg(feature = "..")]` and `cfg!(feature = "..")`.
- `json`, `toml`, `csv`, `http`, `markdown`, `spatial` and `nabi-syntax` are the Nabi project's packages.
- Publish with `nabi login` (a registry token), `nabi publish --dry-run` and `nabi publish`; withdraw with `nabi yank name@version --message <why>`, and retire an API with `#[deprecated]` before removing it. `#[non_exhaustive]` and `#[sealed]` leave room to add variants, fields and impls. Publishing is open to maintainers only for now.

## Further reading

- Spec: [9.10.4c Features](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104c-features), [9.10.4a Registry packages](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104a-registry-packages), [`nabi search` / `publish` / `yank` / `login`](../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-search--publish--yank--login--logout), [`nabi.nabion`](../spec/refs-lang/009-tools/910-nabi-frontend.html#9103-nabinabion--nabilock)
- [packages.nabi-lang.org](https://packages.nabi-lang.org), the package registry
- Next chapter: [19. Cryptography and TLS](19-crypto-and-tls.html)
