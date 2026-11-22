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

The Nabi project publishes a set of libraries as packages of their own, outside the standard library: data formats (`json`, `toml`, `csv`), `http`, `markdown`, `regex`, `spatial`, `nabi-syntax`, and the science packages of [chapter 22](22-science.html). Each is pure Nabi. They are marked **Official** on the registry, which [lists them all](https://packages.nabi-lang.org/packages?q=official:yes), and each one's docs start with how to use it.

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

Publishing adds one line to the registry index, pointing at a commit of your git repository; the registry keeps an archive of that commit, so the version outlives your repository. In short:

```bash
nabi login                # a token from packages.nabi-lang.org/me
nabi publish --dry-run    # print the index entry; nothing leaves your machine
nabi publish
nabi yank textwrap@0.1.0 --message "wraps CJK text wrong; use 0.1.1"
```

Publishing has its own section of the guide, [Publishing Packages](packages/index.html): what a package needs, [`nabi login` and `nabi publish`](packages/01-publishing.html), [trusted publishing](packages/02-trusted-publishing.html) from GitHub Actions, and [keeping an API stable](packages/03-stable-api.html) with `#[deprecated]`, `#[non_exhaustive]` and `#[sealed]`.

> [!IMPORTANT]
> For now only the registry's maintainers can publish. Anyone can sign in and use packages, but `nabi login` refuses an account that cannot publish yet.

## Summary

- `features` in `nabi.nabion` maps names to entries: other features, `dep:<d>`, `<d>/<f>`, `<d>?/<f>`. `default` is on unless turned off.
- Optional dependencies (`optional: true`) are built only when a feature enables them.
- Choose features with `--features`, `--no-default-features`, `--all-features`, and per dependency with `features:` / `default_features: false`. Features are unified across the build, so they must only add.
- Code tests them with `#[cfg(feature = "..")]` and `cfg!(feature = "..")`.
- The Nabi project's packages are marked Official on the registry.
- Publish with `nabi login`, `nabi publish --dry-run` and `nabi publish`, and withdraw with `nabi yank`; the details are in [Publishing Packages](packages/index.html). Publishing is open to maintainers only for now.

## Further reading

- Spec: [9.10.4c Features](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104c-features), [9.10.4a Registry packages](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104a-registry-packages), [`nabi search` / `publish` / `yank` / `login`](../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-search--publish--yank--login--logout), [`nabi.nabion`](../spec/refs-lang/009-tools/910-nabi-frontend.html#9103-nabinabion--nabilock)
- [Publishing Packages](packages/index.html), and [packages.nabi-lang.org](https://packages.nabi-lang.org), the package registry
- Next chapter: [19. Cryptography and TLS](19-crypto-and-tls.html)
