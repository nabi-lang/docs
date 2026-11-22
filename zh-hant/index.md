<div class="hero">
<img src="../logo.svg" alt="">

# Nabi 文件

<div class="lead">Nabi 的指南、範例、語言參考與標準函式庫，對應上方選單中選擇的版本。</div>
<div class="install"><pre><code>curl -fsSL https://nabi.sh | sh</code></pre><p>適用於 macOS、Linux 和 FreeBSD。在 Windows 上，請使用 <a href="learn/01-getting-started.html#安裝指令碼">install.ps1</a>。</p></div>
</div>

<div class="cards">
<a class="card" href="learn/index.html"><span class="tag">learn/</span><h3>學習 Nabi</h3><p>二十一章，從安裝到並行、FFI、套件、陣列，以及實用範例。網站建置時會驗證每一段程式碼範例。</p></a>
<a class="card" href="examples/index.html"><span class="tag">examples/</span><h3>範例</h3><p>小而完整的程式：CLI 工具、檔案處理、TCP echo 伺服器、平行 map、直譯器等等。</p></a>
<a class="card" href="spec/refs-lang/index.html"><span class="tag">spec/</span><h3>語言參考</h3><p>十章，從詞法結構到型別、運算式、模組、元系統、執行環境與 ABI。</p></a>
<a class="card" href="spec/refs-standard-libraries/index.html"><span class="tag">std/</span><h3>標準函式庫</h3><p><code>bare</code> / <code>core</code> / <code>std</code> 三個層級的規格。</p></a>
<a class="card" href="../std/index.html"><span class="tag">api/</span><h3>API</h3><p>標準函式庫的每個模組、型別與函式，由原始碼產生。</p></a>
<a class="card" href="spec/refs-lang/009-tools/index.html"><span class="tag">tools/</span><h3>工具</h3><p><code>nabic</code> 編譯器、<code>nabi</code> 套件工具、<code>nabicc</code> C 編譯器、REPL、Jupyter 核心、LSP 伺服器，以及偵錯工具。</p></a>
<a class="card" href="errors/index.html"><span class="tag">errors/</span><h3>錯誤碼</h3><p>每個 <code>nabic</code> 錯誤碼一頁，附上產生錯誤的範例與修正方式。</p></a>
</div>

## 版本

每個發行系列都有自己的文件：Nabi 0.1.x 的文件在 `docs.nabi-lang.org/0.1/`，依此類推；`latest/` 是最新版本，`dev/` 跟隨 main 分支。只有最新的系列會收到修正。語言的動機與理念請見[規格簡介](spec/index.html)。套件放在套件登錄庫 [packages.nabi-lang.org](https://packages.nabi-lang.org)；[發布套件](learn/packages/index.html)說明如何把你的套件放上去。
