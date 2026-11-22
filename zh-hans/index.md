<div class="hero">
<img src="../logo.svg" alt="">

# Nabi 文档

<div class="lead">Nabi 的指南、示例、语言参考和标准库，对应上方菜单中选择的版本。</div>
<div class="install"><pre><code>curl -fsSL https://nabi.sh | sh</code></pre><p>适用于 macOS、Linux 和 FreeBSD。在 Windows 上使用 <a href="learn/01-getting-started.html#安装脚本">install.ps1</a>。</p></div>
</div>

<div class="cards">
<a class="card" href="learn/index.html"><span class="tag">learn/</span><h3>学习 Nabi</h3><p>共二十一章，从安装讲到并发、FFI、包、数组和实用示例。所有代码示例都在构建网站时检查。</p></a>
<a class="card" href="examples/index.html"><span class="tag">examples/</span><h3>示例</h3><p>小而完整的程序：CLI 工具、文件处理、TCP 回显服务器、并行 map、解释器等。</p></a>
<a class="card" href="spec/refs-lang/index.html"><span class="tag">spec/</span><h3>语言参考</h3><p>共十章，从词法结构讲到类型、表达式、模块、元系统、运行时和 ABI。</p></a>
<a class="card" href="spec/refs-standard-libraries/index.html"><span class="tag">std/</span><h3>标准库</h3><p><code>bare</code> / <code>core</code> / <code>std</code> 三个层级的规范。</p></a>
<a class="card" href="../std/index.html"><span class="tag">api/</span><h3>API</h3><p>标准库的所有模块、类型和函数，由源代码生成。</p></a>
<a class="card" href="spec/refs-lang/009-tools/index.html"><span class="tag">tools/</span><h3>工具</h3><p><code>nabic</code> 编译器、<code>nabi</code> 包工具、<code>nabicc</code> C 编译器、REPL、Jupyter 内核、LSP 服务器和调试工具。</p></a>
<a class="card" href="errors/index.html"><span class="tag">errors/</span><h3>错误码</h3><p>每个 <code>nabic</code> 错误码一页，给出出错的示例和修正方法。</p></a>
</div>

## 版本

每个发布系列有自己的文档：Nabi 0.1.x 的文档在 `docs.nabi-lang.org/0.1/`，依此类推。`latest/` 是最新版本，`dev/` 跟随 main 分支。只有最新系列会得到修正。语言的动机和理念见[规范简介](spec/index.html)。包在包注册表 [packages.nabi-lang.org](https://packages.nabi-lang.org) 上，[发布包](learn/packages/index.html)介绍了如何把你的包发布到那里。
