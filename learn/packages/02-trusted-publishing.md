# 2. Trusted Publishing

A GitHub Actions workflow can publish a package without any token stored in the repository. GitHub gives each job an OIDC token that says which repository, workflow and environment it runs for; the registry checks GitHub's signature on it and, when the package trusts that workflow, trades it for a publish token that lasts 30 minutes and works for that package only. It is the same scheme as crates.io's, PyPI's and npm's trusted publishing.

Set it up once per package:

1. On your [account page](https://packages.nabi-lang.org/me), under **Trusted publishers**, add the package, its GitHub repository (`owner/repo`), the workflow's file name (`publish.yml`) and, if the job runs in a GitHub environment, its name. Only the package's owners can add one, and while publishing is not open yet only the registry's maintainers. A package with no owner on record (one the registry's maintainers wrote into the index directly with `nabi publish --index-dir`) gets its first trusted publisher from them.
2. Add the workflow below to the repository as `.github/workflows/publish.yml`. It runs when a `v*` tag is pushed, checks that the tag matches `version` in `nabi.nabion`, trades the job's OIDC token (audience `packages.nabi-lang.org`) for a publish token, logs in with it and publishes.

```yaml
name: Publish / Package to the registry

on:
  push:
    tags: ['v*']
  # a tag pushed before this file was in the repository
  workflow_dispatch:
    inputs:
      tag:
        description: Tag to publish, e.g. v0.3.1
        required: true

permissions:
  contents: read
  id-token: write

jobs:
  publish:
    name: Publish to packages.nabi-lang.org
    runs-on: ubuntu-24.04
    steps:
      - name: Check out the tag
        uses: actions/checkout@v5
        with:
          ref: ${{ inputs.tag || github.ref }}

      - name: Install Nabi
        run: |
          v=$(curl -fsSL https://github.com/nabi-lang/nabi/releases/latest/download/nabi-version)
          curl -fsSL https://nabi.sh | NABI_NO_MODIFY_PATH=1 sh -s -- -q --version "$v"
          echo "$HOME/.nabi/bin" >> "$GITHUB_PATH"

      - name: Check the tag, tests and publish dry run
        env:
          TAG: ${{ inputs.tag || github.ref_name }}
        run: |
          want=$(sed -n 's/^ *version: "\(.*\)",*$/\1/p' nabi.nabion | head -1)
          [ "v$want" = "$TAG" ] || { echo "tag $TAG, but nabi.nabion has version $want" >&2; exit 1; }
          nabi test
          nabi publish --dry-run

      - name: Log in to packages.nabi-lang.org with the OIDC token
        run: |
          jwt=$(curl -fsS -H "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
            "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=packages.nabi-lang.org" | sed -n 's/.*"value":"\([^"]*\)".*/\1/p')
          out=$(curl -sS --fail-with-body -H 'Content-Type: application/nabion' \
            --data-binary "TrustedPublishing { jwt: \"$jwt\" }" \
            https://packages.nabi-lang.org/api/v1/trusted-publishing/token) || { echo "$out" >&2; exit 1; }
          tok=$(printf '%s' "$out" | sed -n 's/.*token: "\(pkt_[0-9a-f]*\)".*/\1/p')
          [ -n "$tok" ]
          echo "::add-mask::$tok"
          printf '%s\n' "$tok" | nabi login

      - name: Publish with nabi publish
        run: nabi publish
```

Then release with `git tag v0.3.1 && git push origin v0.3.1`. For a tag pushed before the workflow was in the repository, run it by hand from the Actions tab (or `gh workflow run publish.yml -f tag=v0.3.1`): it runs the workflow from the default branch and publishes the tag's commit. Instead of `nabi login`, the step can put the token in `NABI_REGISTRY_TOKEN` for the publish step (`echo "NABI_REGISTRY_TOKEN=$tok" >> "$GITHUB_ENV"`).

Any ref of the named workflow file may publish. Give the publisher an environment: make a GitHub environment (`release`) with required reviewers and a deployment rule for `v*` tags, set `environment: release` on the job, and name it here; the registry then refuses tokens from runs outside it. Without one, protect your `v*` tags if others can push to the repository. The version page names the workflow that published it.

A version already in the index without its metadata (one the registry's maintainers wrote directly with `nabi publish --index-dir`) gets it from the same command: `nabi publish` of the same tree fills in the description, license, README and the rest, and changes nothing else.

## Further reading

- Spec: [`nabi search` / `publish` / `yank` / `login`](../../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-search--publish--yank--login--logout)
- Next: [3. Keeping an API stable](03-stable-api.html)
