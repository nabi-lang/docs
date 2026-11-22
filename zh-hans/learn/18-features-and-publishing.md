# 18. 特性与发布

[第 11 章](11-modules-packages.html)构建了一个包并添加了依赖。本章更进一步：**特性**（feature）让库的用户开启或关闭库的某些部分；Nabi 项目维护的包；以及把你自己的库发布到包注册表 [packages.nabi-lang.org](https://packages.nabi-lang.org)。

## 定义特性

特性是 `nabi.nabion` 中 `features` 字典里的一个名字，映射到它所开启内容的列表。在代码中，已开启的特性就是条件 `feature = "<name>"`（[第 17 章](17-conditional-compilation.html)）。这里的 `geom` 库有一个默认开启的 `circle` 特性，以及一个引入可选依赖的 `pretty` 特性。

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

特性列表中的每一项是下列之一：

| 项          | 开启的内容                                                          |
|-------------|---------------------------------------------------------------------|
| `"circle"`  | 本包的另一个特性                                                    |
| `"dep:fancy"` | 可选依赖 `fancy`                                                  |
| `"http/tls"` | 依赖 `http` 的特性 `tls`（如果 `http` 是可选的，也开启 `http` 本身） |
| `"http?/tls"` | `http` 的特性 `tls`，仅当别处已经开启了 `http` 时生效             |

- `default` 是一个普通特性，除非用户关闭，否则它是开启的。
- 可选依赖（`optional: true`）只有在某个已开启的特性用 `dep:` 或 `<dep>/<f>` 指名它时，才会参与构建。它不会自动得到一个同名特性；没有任何特性开启的可选依赖是错误。`nabi add fancy --path ../fancy --optional` 会同时写入依赖和一个 `"fancy": ["dep:fancy"]` 特性。
- 指名不存在的特性或依赖、对非可选依赖使用 `dep:`，以及循环，都是错误，报告时会给出在 `nabi.nabion` 中的行号和列号。

特性名是说明其开启内容的小写单词（`http1`、`tls`、`pretty`），而不是 `with-x` 或 `no-x`：特性只做加法。

## 开启特性

正在构建的包从它的 `default` 特性开始。`--features` 可以添加更多特性，既可以是它自己的（`fancy`），也可以是直接依赖的（`geom/pretty`）；`--no-default-features` 和 `--all-features` 的作用正如其名。它们适用于 `run`、`build`、`test`、`check` 以及其他构建命令。

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

最后一条仍然输出 `circle`：`--no-default-features` 关闭的是正在构建的包（`app`）的默认特性，而不是 `geom` 的。依赖的特性在声明依赖的地方选择。

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

特性是**统一**的：每个包在整个构建中只有一个特性集合，即所有依赖它的包所要求的并集。所以如果两个库都使用 `geom`，一个带 `pretty`，一个不带，两者都会得到 `pretty`。因此，特性只能添加 API，绝不能改变已有代码的行为。`nabi deps` 以 `--extern-feature` 行显示结果。

锁文件固定依赖图中的每个包，包括可选的包，所以切换特性永远不会改变 `nabi.lock`。

## Nabi 官方包

Nabi 项目把一组库作为独立的包发布，放在标准库之外：数据格式（`json`、`toml`、`csv`）、`http`、`markdown`、`regex`、`spatial`、`nabi-syntax`，以及[第 22 章](22-science.html)的科学计算包。它们都是纯 Nabi 实现。它们在注册表上标为 **Official**，注册表[列出了全部官方包](https://packages.nabi-lang.org/packages?q=official:yes)，每个包的文档都从用法讲起。

`json` 和 `toml` 沿用标准库 [`nabion`](15-nabion.html) 模块的形式：`parse` / `to_str` / `from_str`，加上 `to_value` / `from_value`，以及一个用于你自己类型的 derive。

```nabi norun
#[derive(json::Json)]
struct Server { host: str; port: u16 = 80; }

let s: Server = try json::from_str("{\"host\": \"localhost\"}");
println(json::to_str(s));      // {"host":"localhost","port":80}
```

包提供的 derive 带包名书写，即 `#[derive(json::Json)]`，或者在 `use json::Json;` 之后直接写名字。derive 展开成的是包自己的代码：它可以调用包的内部辅助函数，而你的代码不能。

它们和其他包一样添加。不写版本要求时，`nabi add` 写入 `^` 和最新版本：

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

在脚本中，把文件名写在最后：`nabi add json app.nabi` 会把依赖写入脚本的 `//!` 块（[第 11 章](11-modules-packages.html#单文件脚本与环境)）。

有了 `http`，一个 HTTPS 请求只要几行。包会打开 TCP 连接，用操作系统的信任库运行 [`std::net::tls`](19-crypto-and-tls.html) 的 TLS 1.3 握手，通过 ALPN 提供 HTTP/2，并回退到 HTTP/1.1：

```nabi norun
let r = try await http::get("https://example.com/");
println(f"{r.status} {r.version:?} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

## 发布包

发布会在注册表索引中添加一行，指向你 git 仓库中的一个提交；注册表会保存该提交的归档，所以这个版本比你的仓库活得更久。简而言之：

```bash
nabi login                # a token from packages.nabi-lang.org/me
nabi publish --dry-run    # print the index entry; nothing leaves your machine
nabi publish
nabi yank textwrap@0.1.0 --message "wraps CJK text wrong; use 0.1.1"
```

发布在指南中有专门的部分，即[发布包](packages/index.html)：包需要什么、[`nabi login` 和 `nabi publish`](packages/01-publishing.html)、从 GitHub Actions 进行[可信发布](packages/02-trusted-publishing.html)，以及用 `#[deprecated]`、`#[non_exhaustive]` 和 `#[sealed]` [保持 API 稳定](packages/03-stable-api.html)。

> [!IMPORTANT]
> 目前只有注册表的维护者可以发布。任何人都可以登录并使用包，但 `nabi login` 会拒绝尚不能发布的账户。

## 小结

- `nabi.nabion` 中的 `features` 把名字映射到各项：其他特性、`dep:<d>`、`<d>/<f>`、`<d>?/<f>`。`default` 除非被关闭，否则是开启的。
- 可选依赖（`optional: true`）只有在某个特性开启它时才会构建。
- 用 `--features`、`--no-default-features`、`--all-features` 选择特性，对单个依赖则用 `features:` / `default_features: false`。特性在整个构建中统一，所以只能做加法。
- 代码用 `#[cfg(feature = "..")]` 和 `cfg!(feature = "..")` 检测特性。
- Nabi 项目的包在注册表上标为 Official。
- 用 `nabi login`、`nabi publish --dry-run` 和 `nabi publish` 发布，用 `nabi yank` 撤回；详情见[发布包](packages/index.html)。目前只对维护者开放发布。

## 延伸阅读

- 规范：[9.10.4c 特性](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104c-features)、[9.10.4a 注册表包](../spec/refs-lang/009-tools/910-nabi-frontend.html#9104a-registry-packages)、[`nabi search` / `publish` / `yank` / `login`](../spec/refs-lang/009-tools/910-nabi-frontend.html#nabi-search--publish--yank--login--logout)、[`nabi.nabion`](../spec/refs-lang/009-tools/910-nabi-frontend.html#9103-nabinabion--nabilock)
- [发布包](packages/index.html)，以及包注册表 [packages.nabi-lang.org](https://packages.nabi-lang.org)
- 下一章：[19. 密码学与 TLS](19-crypto-and-tls.html)
