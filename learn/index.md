# Learn Nabi

A guide for newcomers to Nabi, from installation through error handling, concurrency, FFI and C sources, NABION, tools, packages, cryptography, a cookbook, arrays and numerics, the science packages, plugins, reflection, garbage collector pauses, hot reload, Objective-C and Apple frameworks, and iOS. Each chapter has a short explanation and code you can run right away.

Every code block in this guide is compiled and run with the real toolchain when the site is built, except a few marked as not run (fragments, REPL sessions, programs that never end), which are only shown. Where an output block follows a program, its output is checked against the one shown. Copy any of them and run it with `nabi run file.nabi`.

Coming from Python? [Nabi for Pythonistas](nabi-for-python.html) maps Python idioms and standard library modules to Nabi.

Coming from Rust? [Nabi for Rustaceans](nabi-for-rust.html) shows what to unlearn about ownership, errors and `Option`, and where Cargo's commands live.

Doing science or data analysis? [Science with Nabi](22-science.html) maps the science packages, and each package's [docs](https://docs.nabi-lang.org/packages) start with a tutorial, from a first CSV analysis to linear algebra, statistics, signals, fitting, differential equations and derivatives. The [Rosetta](rosetta/index.html) pages solve well-known small tasks (FizzBuzz, a prime sieve, eight queens, word counts) in Nabi.

The complete rules are defined in the [language reference](../spec/refs-lang/index.html) and the [standard library specification](../spec/refs-standard-libraries/index.html); functions and types are listed in the [standard library API](../std/index.html). Small complete programs are collected in [Examples](../examples/index.html). For quick answers to everyday tasks (reading a file, timing, threads, JSON, HTTP), see [How Do I...](20-cookbook.html). When the compiler stops you, look its code up in the [error code index](../errors/index.html).

## Contents
