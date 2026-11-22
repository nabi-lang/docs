# Security Advisories

A security advisory says that some versions of a package have a vulnerability, which versions, and which version fixes it. The registry publishes them at [packages.nabi-lang.org/advisories](https://packages.nabi-lang.org/advisories), in the [OSV](https://ossf.github.io/osv-schema/) format other ecosystems use, and `nabi audit` checks your project against them.

## Checking a project: `nabi audit`

```sh
nabi audit
```

`nabi audit` reads `nabi.lock` and checks every registry package in it, the official packages the toolchain carries included, against the advisories of its registry. For each one that applies it prints the advisory's id and summary, its severity when the advisory gives one, and the version to upgrade to; then run `nabi update <name>` (chapter 11) or raise the requirement in `nabi.nabion`. Path and git dependencies and native packages are not checked, and it says which it skipped. Without a `nabi.lock` it resolves one first.

The exit status says what it found:

| Status | Meaning |
|--------|---------|
| 0 | No advisory applies. |
| 1 | At least one does. |
| 2 | The check could not run: no network and no cached list, an unreadable lock file, or an error from the registry. The message says which. |

The list is fetched once, with one request per registry, and kept for an hour under `$NABI_HOME/cache/advisories`. `--offline` uses that copy whatever its age, and a failed fetch falls back to it.

| Flag | Effect |
|------|--------|
| `--json` | One JSON object per line: a record per finding, one per package not checked, then a summary. Same as `--format json`. |
| `--ignore <id>` | Skip an advisory, by its id (`NABI-2026-0001`) or an alias (`CVE-..`, `GHSA-..`). Repeat it for more. |
| `--offline` | Use only the cached list. |

In CI, run it after the build; a status of 1 fails the job:

```yaml
- run: nabi audit
```

Ignore an advisory only after reading it and deciding that your program does not reach the affected code; say why next to the flag.

## Reading an advisory

Each advisory has a page on the registry, and the pages of the versions it affects carry a banner that links it. An advisory names:

- **the affected versions**: ranges such as "from 0.1.0, fixed in 0.2.3", or a list of versions;
- **the fixed versions**, when there are any: upgrade to one of them;
- **its severity**, a CVSS vector, when the reporter gave one;
- **references and credits**: the fix, the report, other ids for the same flaw.

A withdrawn advisory stays listed, marked withdrawn, and `nabi audit` skips it.

The whole list is also an API, for other tools: `GET https://packages.nabi-lang.org/api/v1/advisories?format=json` gives every advisory as OSV JSON, and `/api/v1/packages/<name>/advisories` those of one package. The [API reference](https://packages.nabi-lang.org/api) has the fields.

## Reporting a vulnerability

Found one in a package? Tell its owners privately first; their repository's security policy says how. If they do not answer, or for a flaw in the registry or the toolchain, write to [security@nabi-lang.org](mailto:security@nabi-lang.org).

Owners of a package can draft an advisory for it through the API (`POST /api/v1/advisories` with an account token that has the `owners` scope); the registry's maintainers review it and publish it, and every owner of the package gets a mail when it is published.
