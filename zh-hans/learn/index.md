# 学习 Nabi

这是一份面向 Nabi 新手的指南，从安装讲起，涵盖错误处理、并发、FFI 与 C 源代码、NABION、工具、包、密码学、实用示例、数组与数值计算、科学计算包、反射，以及垃圾回收器的停顿。每一章都有简短的说明和可以直接运行的代码。

本指南中的每个代码块都会在构建网站时用真实的工具链编译并运行，并检查输出是否与页面上给出的一致。任意复制一段，用 `nabi run file.nabi` 运行即可。

从 Python 转过来？[写给 Python 用户的 Nabi](nabi-for-python.html) 把 Python 的惯用写法和标准库模块对应到 Nabi。

从 Rust 转过来？[写给 Rust 用户的 Nabi](nabi-for-rust.html) 说明在所有权、错误和 `Option` 上需要改掉哪些习惯，以及 Cargo 的命令在 Nabi 中对应什么。

做科学计算或数据分析？[用 Nabi 做科学计算](22-science.html) 梳理了各个科学计算包，每个包的[文档](https://docs.nabi-lang.org/packages)都从一篇教程开始，从第一次 CSV 分析讲到线性代数、统计、信号、拟合、微分方程和导数。[Rosetta](rosetta/index.html) 页面用 Nabi 解决一些广为人知的小问题（FizzBuzz、素数筛、八皇后、单词计数）。

完整的规则定义在[语言参考](../spec/refs-lang/index.html)和[标准库规范](../spec/refs-standard-libraries/index.html)中；函数和类型列在[标准库 API](../../std/index.html) 中。小而完整的程序收集在[示例](../examples/index.html)中。日常任务（读文件、计时、线程、JSON、HTTP）的快速解答，请见[怎么做……](20-cookbook.html)。编译器报错时，到[错误码索引](../errors/index.html)中查找对应的错误码。

## 目录
