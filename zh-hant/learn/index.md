# 學習 Nabi

這是寫給 Nabi 新手的指南，從安裝開始，涵蓋錯誤處理、並行、FFI 與 C 原始檔、NABION、工具、套件、密碼學、實用範例、陣列與數值計算、科學計算套件、外掛、反射、垃圾回收器的暫停、熱重載、Objective-C 與 Apple 框架，以及 iOS。每一章都有簡短的說明和可以立刻執行的程式碼。

建置網站時，本指南的每個程式碼區塊都會用真正的工具鏈編譯並執行，只有少數標示為不執行的區塊（程式碼片段、REPL 工作階段、不會結束的程式）僅供展示。程式後面接著輸出區塊時，會檢查實際輸出是否與它相同。任何一段都可以複製下來，用 `nabi run file.nabi` 執行。

用過 Python？[給 Python 使用者的 Nabi](nabi-for-python.html) 列出 Python 的慣用寫法與標準函式庫模組在 Nabi 中對應的寫法。

用過 Rust？[給 Rust 使用者的 Nabi](nabi-for-rust.html) 說明關於所有權、錯誤和 `Option` 有哪些習慣要放下，以及 Cargo 的指令在 Nabi 的哪裡。

要做科學計算或資料分析？[用 Nabi 做科學計算](22-science.html) 整理了科學計算套件，每個套件的[文件](https://docs.nabi-lang.org/packages)都從教學開始，從第一次的 CSV 分析，到線性代數、統計、訊號、擬合、微分方程與導數。[Rosetta](rosetta/index.html) 頁面用 Nabi 解決一些知名的小題目（FizzBuzz、質數篩法、八皇后、單字計數）。

完整的規則定義在[語言參考](../spec/refs-lang/index.html)與[標準函式庫規格](../spec/refs-standard-libraries/index.html)中；函式與型別列在[標準函式庫 API](../../std/index.html) 裡。小而完整的程式收錄在[範例](../examples/index.html)中。日常工作（讀取檔案、計時、執行緒、JSON、HTTP）的快速解答，請見[怎麼做……](20-cookbook.html)。編譯器擋下你時，可以到[錯誤碼索引](../errors/index.html)查它的錯誤碼。

## 目錄
