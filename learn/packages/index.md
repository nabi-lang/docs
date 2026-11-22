# Publishing Packages

How to put your own library or program on the package registry, [packages.nabi-lang.org](https://packages.nabi-lang.org), and keep it there. Using packages is covered in [chapter 11](../11-modules-packages.html#the-package-registry), and features in [chapter 18](../18-features-and-publishing.html).

> [!IMPORTANT]
> For now only the registry's maintainers can publish. Anyone can sign in and use packages, but `nabi login` refuses an account that cannot publish yet.

## How the registry works

The registry's data is a single git repository, the [index](https://github.com/nabi-lang/packages-index). Each version of a package is one line in it, naming a commit of the package's own git repository and that commit's tree hash. You upload no source: when a version is published, the registry fetches that commit, keeps an archive of it and writes the archive's SHA-256 into the line. Users download the archive and check it against the index, so a version outlives its repository. Lines are written only by the registry: the index takes no pull requests. The line format is described in the [index README](https://github.com/nabi-lang/packages-index#readme).

Packages published from the nabi-lang organization are marked **Official**; they come with the toolchain ([chapter 11](../11-modules-packages.html#the-package-registry)).

## The pages

| Page | What it covers |
|------|----------------|
| [Publishing a package](01-publishing.html) | what a package needs, `nabi login`, `nabi publish`, names and owners, the docs the registry makes, `nabi yank` |
| [Trusted publishing](02-trusted-publishing.html) | publishing from GitHub Actions with no stored token |
| [Keeping an API stable](03-stable-api.html) | `#[deprecated]`, `#[non_exhaustive]` and `#[sealed]` |

The registry's own rules are on the registry: the [usage policy](https://packages.nabi-lang.org/policy), the [terms of use](https://packages.nabi-lang.org/terms) and the [privacy notice](https://packages.nabi-lang.org/privacy).
