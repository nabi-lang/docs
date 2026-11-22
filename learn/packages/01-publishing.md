# 1. Publishing a Package

Publishing adds one line to the registry index, pointing at a commit of your git repository ([how the registry works](index.html#how-the-registry-works)). This page goes from the manifest to a published version, and how to withdraw one.

## What a package needs

- `nabi.nabion` at the root of the repository, with `name`, `version` and `nabi_version`.
- A `src/lib.nabi` (a library), a `src/main.nabi` (a program for `nabi install`) or both, and a readme.
- A clean git work tree with a remote `origin`, whose url is `https://` on a public host, and the commit pushed to a branch or tag of it (a commit only in a fork does not count).
- Every normal dependency from the registry. Git dependencies are rejected, and so is a path dependency unless it also names the version to publish in its place, `Dep::path("../geom", version: "^0.1")` (publish `geom` first). Path and git `dev_deps` without a `version:` are dropped.
- No `publish: false` in the manifest.

Set `nabi_version` to the oldest nabi you have built the package with: `nabi publish` needs it, and resolution gives each user the newest version their toolchain builds (`"0.1.3"` admits nabi 0.1.3 and later 0.1.x, [chapter 1](../01-getting-started.html#managing-toolchains)). Each package page shows it as "Requires". `nabi new` writes the version that made the package.

Fill in `description`, `license` and `repository`: they show up in `nabi search` and on the website. The other fields a package page shows are optional, as in Cargo:

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

## `nabi login`

Sign in at [packages.nabi-lang.org/me](https://packages.nabi-lang.org/me) with GitHub and create an account token there. Then hand it to `nabi`:

```text
$ nabi login
a token from https://packages.nabi-lang.org/me (not shown; Ctrl-C to cancel):
   Logged in as your-name at https://packages.nabi-lang.org (token in ~/.nabi/tokens.nabion)
```

Without an argument, the token is read from standard input without echoing. `nabi login` asks the registry who the token belongs to and refuses a token it rejects, saving nothing. A good token is kept in `~/.nabi/tokens.nabion` (readable only by you), filed under the index it was made for, and is sent only to that index; `nabi login --registry <url>` keeps one for another index. `nabi logout` removes it, and `nabi logout --all` removes every index's token. In CI, set `NABI_REGISTRY_TOKEN` instead; it is sent only to packages.nabi-lang.org. From GitHub Actions, [trusted publishing](02-trusted-publishing.html) needs no stored token at all.

## `nabi publish`

`nabi publish --dry-run` prints the index entry it would add, and nothing leaves your machine. `nabi publish` publishes, as `cargo publish` does.

```text
$ nabi publish --dry-run
Package { name: "textwrap", version: "0.1.0", deps: [], source: Dep::git("https://github.com/you/textwrap.git", rev: "8e2afb20.."), tree: "823a3592..", yanked: false, nabi_version: "0.1.0" }
note: dry run; `nabi publish` sends it to the registry API of https://github.com/nabi-lang/packages-index (needs `nabi login`), which adds it as te/xt/textwrap
```

The entry holds only what resolving needs: the HEAD commit and its tree hash, the dependencies, and `nabi_version`. Push that commit first. The description, license and other metadata travel beside it to the registry, which shows them with who published each version. `nabi publish` sends both with your token (`Published textwrap 0.1.0 to https://packages.nabi-lang.org`), and the version is live once the index has it. A version cannot be published twice.

A package with an executable is marked **bin** in lists and search, and its page shows `nabi install <name>` (`nabi install --bin <name>` when it has a library too).

## Names and owners

A name is at most 64 characters. A name differing only in case or `-` / `_` from an existing package is taken. The standard library's tiers `std`, `core` and `bare` are entries of the index owned by the Nabi project: they ship with the toolchain, and nothing is published under their names. Names of the toolchain (`nabi`, `nabic`, ...) are reserved, and a new name one letter away from a name in the index, or the same once look-alike characters are folded (`js0n` for `json`), is refused; the [usage policy](https://packages.nabi-lang.org/policy) says how to ask for one anyway.

Whoever publishes a package's first version owns it. Owners invite other accounts to share it on the [account page](https://packages.nabi-lang.org/me); an invited account owns it once it accepts.

## API docs on the registry

The registry makes each version's API docs from the source at that commit, reading it without building or running it: [packages.nabi-lang.org/docs/textwrap/0.1.0](https://packages.nabi-lang.org/docs/textwrap/0.1.0), and `/docs/textwrap/latest` for the newest version. The pages are laid out as `nabi doc` lays them out: the root module there, each other public module on a page of its own (`/docs/textwrap/0.1.0/<module>`), and the same anchors (`#fn.wrap`). The root module's `//!` text is the first thing a reader sees, so it is the place for a package's introduction and tutorial; its code blocks are doc tests that `nabi test` runs. A module's doc is its file's leading `//!` lines; plain `//` comments are not shown. An item you re-export from a private module with `pub use` is shown there as if it were declared where you re-export it. A package with only an executable has no API, and so no Docs tab. If the docs cannot be made, the version is published anyway: `nabi publish` prints the reason as a warning, and the docs page shows it.

## `nabi yank`

A published version is never deleted. A version with a serious bug can be yanked: new resolutions skip it, but projects whose `nabi.lock` already pins it keep building, so yanking never breaks anyone. `--undo` reverses it.

```bash
nabi yank textwrap@0.1.0 --message "wraps CJK text wrong; use 0.1.1"
nabi yank textwrap@0.1.0 --undo
```

`--message` says why; the registry shows it on the package's page.

## Further reading

- Spec: [9.10.4a Registry packages](../../spec/refs-lang/009-tools/910-nabi-frontend.html#9104a-registry-packages), [`nabi search` / `publish` / `yank` / `login`](../../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-search--publish--yank--login--logout), [`nabi.nabion`](../../spec/refs-lang/009-tools/910-nabi-frontend.html#9103-nabinabion--nabilock)
- Next: [2. Trusted publishing](02-trusted-publishing.html)
