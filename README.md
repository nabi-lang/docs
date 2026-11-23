# Nabi docs

[![Content](https://github.com/nabi-lang/docs/actions/workflows/content.yml/badge.svg)](https://github.com/nabi-lang/docs/actions/workflows/content.yml) [![License: CC BY 4.0](https://img.shields.io/badge/license-CC%20BY%204.0-blue)](LICENSE)

The guide, examples and cookbook for [Nabi](https://nabi-lang.org), in four editions: English, Korean (`ko/`), Simplified Chinese (`zh-hans/`) and Traditional Chinese (`zh-hant/`). (The language spec is in [nabi-lang/spec](https://github.com/nabi-lang/spec).) The site at docs.nabi-lang.org renders these files; this repository holds only content.

## Branches

- `main`: the next release, served at docs.nabi-lang.org/dev/
- `vX.Y.Z`: the docs for that release, served at /X.Y.Z/ (and /X.Y/ for the newest patch of a minor). Fix a mistake in a released version on its branch.

A release is a `vX.Y.Z` branch here and in nabi-lang/spec; the site finds them on its own and serves the newest at /latest/.

## Layout

- `index.md`: the docs home
- `learn/`: the guide, one chapter per file, with `learn/packages/` (publishing packages) and `learn/rosetta/` (small well-known tasks)
- `examples/`: complete programs
- `errors/`: one page per nabic error code
- `ko/`, `zh-hans/`, `zh-hant/`: the Korean, Simplified Chinese and Traditional Chinese pages, the same layout as the English ones at the root
- `i18n/`: the translation glossaries
- `checks/`: the sources whose `nabi check` output the pages' ` ```text check=` blocks must match

English is the source. A change to an English page updates its `ko/` page in the same pull request; `zh-hans/` and `zh-hant/` too, or a maintainer follows up.

Every ` ```nabi ` block is compiled and run with the matching toolchain before a change goes live.
