<div class="hero">
<img src="logo.svg" alt="">

# Nabi Docs

<div class="lead">The guide, examples, language reference and standard library of Nabi, for the version in the menu above.</div>
<div class="install"><pre><code>curl -fsSL https://nabi.sh | sh</code></pre><p>macOS, Linux and FreeBSD. On Windows, use <a href="learn/01-getting-started.html#install-script">install.ps1</a>.</p></div>
</div>

<div class="cards">
<a class="card" href="learn/index.html"><span class="tag">learn/</span><h3>Learn Nabi</h3><p>Twenty-eight chapters, from installation to concurrency, FFI, packages, arrays, plugins, hot reload, Objective-C, iOS, and a cookbook. Every code sample is checked when the site is built.</p></a>
<a class="card" href="examples/index.html"><span class="tag">examples/</span><h3>Examples</h3><p>Small complete programs: CLI tools, file processing, a TCP echo server, parallel map, an interpreter, and more.</p></a>
<a class="card" href="spec/refs-lang/index.html"><span class="tag">spec/</span><h3>Language reference</h3><p>Ten chapters, from lexical structure through types, expressions, modules, the meta system, the runtime, and the ABI.</p></a>
<a class="card" href="spec/refs-standard-libraries/index.html"><span class="tag">std/</span><h3>Standard library</h3><p>The specification of the three tiers <code>bare</code> / <code>core</code> / <code>std</code>.</p></a>
<a class="card" href="std/index.html"><span class="tag">api/</span><h3>API</h3><p>Every module, type and function of the standard library, generated from its sources.</p></a>
<a class="card" href="spec/refs-lang/009-tools/index.html"><span class="tag">tools/</span><h3>Tools</h3><p>The <code>nabic</code> compiler, the <code>nabi</code> package tool, the <code>nabicc</code> C compiler, the REPL, the Jupyter kernel, the LSP server, and debugging tools.</p></a>
<a class="card" href="errors/index.html"><span class="tag">errors/</span><h3>Error codes</h3><p>One page per <code>nabic</code> error code, with an example of the error and its fix.</p></a>
</div>

## Versions

Each release series has its own docs: `docs.nabi-lang.org/0.1/` for Nabi 0.1.x, and so on; `latest/` is the newest release and `dev/` follows the main branch. Only the latest series gets fixes. The [spec introduction](spec/index.html) gives the motivation and philosophy of the language. Packages are on the package registry, [packages.nabi-lang.org](https://packages.nabi-lang.org); [Publishing Packages](learn/packages/index.html) says how to put yours there.
