# Next-generation PTA implementations (Rust and others) and architecture building blocks for a CLI + Python accounting core (state as of Sept 2026)

Research date: 2026-09-25. Star counts and versions come from GitHub search, crates.io and PyPI API queries run that day, unless noted otherwise. Anything older than 2025 is marked **[older]**.

---

## Q1. Which newer or alternative implementations already exist (rustledger, limabean, tackler, parser crates, Beancount v3 C++, Go clones, others)? For each: status, maturity, features, performance, Python bindings, syntax compatibility, license, and 2025-2026 activity

### Takeaway
There is already a serious Rust Beancount reimplementation: **rustledger**. It was created in January 2026 and reached v0.24.0 on 2026-09-06, with 396 stars, BQL, 7 booking methods, 31 plugins, an LSP, WASM builds and a Fava fork. It has **no PyO3/pip library API**, and it is **GPL-3.0-only with a CLA**. The other projects are narrower. limabean is a Clojure frontend over a Rust parser and booking engine. beancount-parser-lima and beancount-parser are parsers only. tackler uses its own simplified format. okane handles Ledger syntax. uromyces is a toy with Python bindings. No Rust clone of the Ledger/hledger language is anywhere near hledger's feature set. The Beancount C++ rewrite is shelved.

### Cited Findings

**rustledger (Rust, Beancount-compatible)**
- Repo created 2026-01-04. As of 2026-09-25 it had 396 stars, 39 forks and 20 open issues, and was updated the same day. Description: "Modern plain text accounting. Beancount compatible." — [GitHub search API result for rustledger/rustledger](https://github.com/rustledger/rustledger)
- Latest crate is 0.24.0 (published 2026-09-06), described as "Drop-in replacement for Beancount. Pure Rust, 10-30x faster." It has about 2.2k total downloads. The license is **GPL-3.0-only**; rustledger-core has the same license and about 5.3k downloads. — [crates.io rustledger](https://crates.io/crates/rustledger); [crates.io rustledger-core](https://crates.io/crates/rustledger-core)
- The GitHub page shows 3,221 commits. It is a workspace of about 15 published crates: core (Amount/Position/Inventory), parser, loader, booking (STRICT, STRICT_WITH_SIZE, FIFO, LIFO, HIFO, AVERAGE, NONE), validate, query (BQL), completion, plugin (31 built-ins plus Python plugins), importer (CSV/OFX plus WASM importers), ops (ML categorization, dedup), lsp, wasm (JS/TS via npm) and ffi-component (WASI Preview 2 / Component Model). It also ships an MCP server and `ag-rledger`, a CLI that returns JSON envelopes. — [rustledger README](https://github.com/rustledger/rustledger)
- The announcement came on 2026-01-29 from robcohen. It claimed "100% compatibility on 759 real-world test files", full BQL, 20 built-in plugins at the time, and asked for testers. — [PTA forum announcement](https://forum.plaintextaccounting.org/t/announcing-rustledger-beancount-reimagined-in-rust/725)
- **Python story:** there are no PyO3 bindings. Python *plugins* run in "a sandboxed CPython compiled to WebAssembly" and are "compatible with most pure-Python beancount plugins", but plugins with C extensions are not supported. `rledger compat install` installs `bean-check`/`bean-query` wrapper scripts. The README tells users to stay on Python beancount if they "need Fava web interface (until rustledger integration)". — [rustledger README](https://github.com/rustledger/rustledger)
- ADR-0004 (Phase 3, May 2026) generates a JSON Schema from the Rust DTOs and uses `datamodel-code-generator` to emit `bindings/types.py` (Pydantic v2) for Python consumers. It calls the Python compat layer (`crates/rustledger-ffi-wasi/python/compat.py`) hand-maintained. It lists `.pyi` stubs as "explicitly out of scope", and the old wasip1 JSON-RPC FFI is "slated for removal". — [rustledger ADR 0004](https://github.com/rustledger/rustledger/blob/main/docs/reference/adr/0004-ts-types-from-rust-dtos.md); [rustledger README crate table](https://github.com/rustledger/rustledger)
- The roadmap names "Arbitrary-precision decimal" as "the single most-cited reason a ledger could produce different numbers". It also plans "range-based reparse for LSP responsiveness" and a browser playground. It does not mention PyO3, pip, Arrow or DataFrames. — [rustledger roadmap](https://rustledger.github.io/roadmap/)
- **rustfava** is a fork of Fava whose parser layer is rustledger. It "runs the rustledger engine as an in-process WebAssembly component, so the `wasmtime` Python package ships as a dependency". It is MIT-licensed, has 54 stars and 3,665 commits, and ships a Tauri desktop app. — [rustfava GitHub](https://github.com/rustledger/rustfava). PyPI has rustfava 1.32.0 (uploaded 2026-08-22), which requires Python >=3.13. — [PyPI rustfava](https://pypi.org/project/rustfava/)
- Development process: the repo root contains `CLAUDE.md` ("Claude Code Context ... All development work in this repository MUST use git worktrees"), `AGENTS.md`, `opencode.jsonc`, `Dockerfile.opencode` and `CLA.md`. That points to a heavily AI-agent-assisted workflow and a contributor license agreement. — [rustledger repo root](https://github.com/rustledger/rustledger/blob/main/CLAUDE.md) (inspected via a shallow clone on 2026-09-25)

**limabean + beancount-parser-lima + limabean-booking (Rust + Clojure)**
- limabean is "a new implementation of Beancount using Rust and Clojure and the Lima parser". The only user interface is the Clojure REPL, with "no support for Beancount Query Language nor Python". Rust handles only parsing and booking. It runs on Linux and macOS, and Windows needs WSL. 47 stars, MIT OR Apache-2.0. — [limabean GitHub](https://github.com/tesujimath/limabean)
- Crate limabean is at 0.6.1 (2026-05-25, about 339 downloads). Booking tags go up to booking-0.8.0. — [crates.io limabean](https://crates.io/crates/limabean)
- The booking algorithm lives in a separate crate, **limabean-booking**, which is "entirely generic, having no dependencies on either the Lima parser types nor limabean itself". — [lib.rs limabean](https://lib.rs/crates/limabean); [PTA forum thread](https://forum.plaintextaccounting.org/t/limabean-a-new-implementation-of-beancount-in-rust-and-clojure-reddit/740)
- There was a Show HN, "Limabean – a new implementation of Beancount in Clojure/Rust" (2026). I could not read the discussion because of HTTP 429. — [HN 47239083](https://news.ycombinator.com/item?id=47239083)
- beancount-parser-lima is a zero-copy parser built on Logos (lexer) and Chumsky (combinators), with Ariadne error reports. It has no booking. Its Python bindings "removed to a separate repo" and "demoted in status to proof-of-concept". The test suite is based on Beancount's own tests, with differences marked "ANOMALY". License is MIT/Apache-2.0, except that the protobuf test files are GPLv2. — [docs.rs beancount_parser_lima](https://docs.rs/beancount-parser-lima/latest/beancount_parser_lima/)
- Version 0.16.3 (2026-05-10), about 12.1k downloads, 23 stars. — [crates.io](https://crates.io/crates/beancount-parser-lima)

**beancount-parser (jcornaz)**
- Version 2.6.0 (2026-02-19), Unlicense, about 63k downloads (the most-downloaded Beancount parser crate). Built on `nom` 8, `nom_locate` and `miette`. 29 stars. — [crates.io beancount-parser](https://crates.io/crates/beancount-parser); [GitHub](https://github.com/jcornaz/beancount-parser)

**tackler / tackler-ng (Rust, own format)**
- "Fast, reliable bookkeeping engine with native GIT SCM support". Apache-2.0, 162 stars. It is a Rust reimplementation of an earlier Scala tool. The README claims it "can process 900 000 transactions per second on modern laptop", and says it is "tested with 520 tracked test vectors". — [tackler GitHub](https://github.com/tackler-ng/tackler). An older lib.rs listing says 700,000 txn/s. — [lib.rs tackler](https://lib.rs/crates/tackler)
- Version 26.8.1 (2026-08-20; tags use calendar versioning, e.g. v26.01.2 through v26.08.1). About 9.6k downloads. — [crates.io tackler](https://crates.io/crates/tackler); [docs](https://tackler.fi/docs/tackler/latest/)
- The format is its own "simplified syntax" and is not Ledger-compatible ("Tackler Ain't Calculator and Kernel for Ledger Equivalent Records"). It has ISO-8601 timestamps down to nanoseconds with time zones, UUIDs that are mandatory in audit mode, geo-location, and a strict chart of accounts. Account names cannot contain spaces. The docs mention no conversion tools. — [tackler journal format](https://tackler.fi/docs/journal/format/)
- Audit mode requires UUIDs and produces SHA-256/SHA-512 checksums of transaction sets. Reports cover balance, balance-group, register, equity, identity and JSON. — [tackler GitHub](https://github.com/tackler-ng/tackler)
- tackler-core 0.18.0 depends on `winnow` 1.0, `gix` (git), `jiff` (time), `rust_decimal`, `mimalloc` and `sha2`/`sha3`. — [crates.io tackler-core](https://crates.io/crates/tackler-core)
- There is a synthetic test-data generator, **pta-generator**. — [tackler-ng/pta-generator](https://github.com/tackler-ng/pta-generator)

**Other Rust projects**
- **uromyces** calls itself "a toy Rust re-implementation of parts of Beancount's functionality". It uses a tree-sitter grammar, exposes Python bindings (`uromyces.load_file`) and is aimed at Fava. It mentions possibly using salsa-rs for incremental computation. Claimed speedups over Beancount: parsing about 2x, booking about 40x, plugins about 10x. Currency inference and interpolation are weaker than Beancount's, and error messages differ. 7 stars, no releases. — [yagebu/uromyces](https://github.com/yagebu/uromyces)
- **zhang** (账) is "compatible with beancount but more powerful". It deprecates `note`, `pad` and `push_tag`, and adds datetimes, budgets and document management. It is Rust with a web UI and has zhang-core/cli/server/sql components. Apache-2.0, 188 stars, updated 2026-08-15. — [zhang-accounting/zhang](https://github.com/zhang-accounting/zhang)
- **okane** handles the Ledger-cli format in Rust. MIT, crate 0.21.1 (2026-08-20), about 20k downloads, 13 stars, active 2026-09-24. Built on `winnow` 1.0, `rust_decimal` and `bumpalo`. — [xkikeg/okane](https://github.com/xkikeg/okane); [crates.io okane](https://crates.io/crates/okane); [crates.io okane-core deps](https://crates.io/crates/okane-core)
- **ledger-parser / ledger-utils** (marek-g) parse Ledger files and compute balances. Unlicense. Last versions were 7.0.0 (2024-06-06) and 0.6.0 (2024-03-14), so both look **dormant since 2024**. — [crates.io ledger-parser](https://crates.io/crates/ledger-parser); [crates.io ledger-utils](https://crates.io/crates/ledger-utils)
- **Transity** has 654 stars. GitHub lists its primary language as Rust, and its topics still include "purescript". Updated 2026-09-20. — [ad-si/Transity](https://github.com/ad-si/Transity)
- **beancount-language-server** (Rust LSP) has 250 stars and was updated 2026-09-22. **tree-sitter-beancount** has 56 stars. **twilco/beancount** ("Rust tooling surrounding beancount") has 88 stars. **bean-rs** and **lumi** are small. — [GitHub search results](https://github.com/polarmutex/beancount-language-server); [tree-sitter-beancount](https://github.com/polarmutex/tree-sitter-beancount); [twilco/beancount](https://github.com/twilco/beancount)
- **Klirr** is *not* a PTA tool. It is a Rust invoice generator (Sajjon/klirr, 140 stars). — [Sajjon/klirr](https://github.com/Sajjon/klirr)

**Go**
- **howeyc/ledger** is a Go command-line double-entry program with a Ledger-like format, CSV import/export and a web UI. 505 stars, updated 2026-09-16. — [howeyc/ledger](https://github.com/howeyc/ledger); [pkg.go.dev](https://pkg.go.dev/github.com/howeyc/ledger)
- **tn47/goledger** is another Go Ledger implementation. — [tn47/goledger](https://github.com/tn47/goledger)
- **robinvdvleuten/beancount** is a "Fast, lightweight Beancount parser, formatter and editor written in Go". 5 stars, active 2026-09-25. — [robinvdvleuten/beancount](https://github.com/robinvdvleuten/beancount)

**Incumbents (for context)**
- **Beancount v3** is still a Python core plus a C parser. The repo has `beancount/parser/lexer.l`, `grammar.y` and `decimal.c`, uses `mesonpy` as the build backend, and is GPL-2.0-only with requires-python >=3.9. — [beancount repo](https://github.com/beancount/beancount) (pyproject and tree inspected). The latest PyPI release is 3.2.3 (2026-05-05). — [PyPI beancount](https://pypi.org/project/beancount/)
- **Beancount C++ rewrite**: on 2024-06-16, Blais moved the C++/Bazel work to a `cpp` branch. v3 is "pretty stripped down" and split into beanquery, beangulp, beanprice and beangrow. He said he might "salvage bits and pieces ... into a 'rust' branch". **[older, 2024]** — [Beancount mailing list announcement](https://groups.google.com/g/beancount/c/iTdRuvZnE4E). Search summaries describe the C++ core as "on ice". — [Beancount v3 docs / groups](https://groups.google.com/g/beancount/c/yaMv6kdS_7k)
- **hledger**: 1.52.4 and 1.99.4 ("2.0 preview 4") were both released 2026-09-10. hledger 2 previews add "automated lot tracking and capital gains calculation" with Beancount-style cost annotations and FIFO/LIFO/HIFO/AVERAGE, plus a new `holdings` report. — [hledger release notes](https://hledger.org/relnotes.html). License GPL-3.0-or-later. — [hledger LICENSE](https://github.com/simonmichael/hledger/blob/master/LICENSE)
- **Ledger**: latest tag v3.4.1, BSD-3-style license ("Copyright (c) 2003-2025, John Wiegley"). It needs Boost ≥1.72 and GMP/MPFR. The Python bridge is optional and off by default (`option(USE_PYTHON ... OFF)`). — [ledger LICENSE.md](https://github.com/ledger/ledger/blob/master/LICENSE.md); [ledger CMakeLists.txt](https://github.com/ledger/ledger/blob/master/CMakeLists.txt)
- Fava 1.30.16 (2026-08-18). — [PyPI fava](https://pypi.org/project/fava/)

### Inferences
- rustledger already covers most of the "fast Beancount-compatible core + CLI + LSP + WASM" space. Rebuilding a Beancount-compatible Rust core would duplicate it. The gaps it leaves are exactly what this project wants: a **first-class, in-process Python library** (PyO3, native objects, DataFrames), arbitrary-precision decimals, and Ledger/hledger syntax.
- Licensing decides a lot. Any PyO3 wrapper that links rustledger crates must be distributed under GPL-3.0-only, and contributing upstream needs their CLA. The permissive building blocks are beancount-parser-lima, limabean-booking (MIT/Apache), beancount-parser and ledger-parser (Unlicense), okane (MIT) and tackler (Apache-2.0).
- rustledger has one visible human maintainer, a very high commit velocity (about 3.2k commits in about 9 months) and an explicit AI-agent workflow. It may move fast, but API stability and bus factor are open risks for a downstream dependency.
- rustfava shows that one team chose **wasmtime-in-Python** (WASM component) over PyO3 for Python embedding. That gives a single portable artifact but pays serialization costs at the boundary.
- For Ledger/hledger syntax there is no mature Rust core. okane is the most active option (MIT, winnow). hledger 2.0 is converging toward Beancount-style lot tracking.

### Gaps
- I could not confirm whether "pta-rs" exists. No repository with that name turned up.
- Exact GitHub star counts for beancount/beancount, hledger and ledger were not retrieved (the GitHub REST API is blocked in this session).
- The HN comments on limabean are unread (rate-limited), so I have no community reaction data.
- rustledger's Python-plugin compatibility rate on real ledgers is not quantified.

---

## Q2. Do projects exist that expose a ledger as Arrow/Polars/pandas DataFrames or let you query it with real SQL (DuckDB/SQLite)?

### Takeaway
Only thin or niche options exist: Beancount's own SQL-like BQL (beanquery), hledger's `print -O sql`, a 1-star DuckDB extension for Ledger files, and a few tiny pandas converters. I found **no mature project that exposes a ledger as Arrow/Polars DataFrames or runs real SQL over booked postings with exact decimals**. That is a clear open niche.

### Cited Findings
- **beanquery** is "A customizable lightweight SQL query tool that works on tabular data, including Beancount". 65 stars, updated 2026-09-25. Latest PyPI release is 0.2.0 (2025-03-24), GPL-2.0. — [beancount/beanquery](https://github.com/beancount/beanquery); [PyPI beanquery](https://pypi.org/project/beanquery/)
- The BQL docs explain the choice of a custom language: the old `bean-sql` converted a ledger to SQLite, but "the results are not great"; queries were painful, and handling lots held at cost was difficult. — [Beancount Query Language doc](https://beancount.github.io/docs/beancount_query_language/). There is a GitHub issue "Should `bean-sql` be removed in v3?" — [beancount#661](https://github.com/beancount/beancount/issues/661). A search summary of a mailing-list thread also says SQLite lacks a real decimal type (I did not verify this in the original thread). — [Conversion from Beancount to SQLite3](https://groups.google.com/g/beancount/c/wpzfjzXBJaY)
- **hledger**: "Only hledger's print command outputs SQL", aimed at SQLite, MySQL and Postgres. It emits a `postings` table (`amount numeric, commodity text, credit numeric, debit numeric, ...`). On SQLite, `id serial` produces NULL ids, and the documented workaround is a `sed` rewrite before piping into `sqlite3`. — [hledger and SQLite](https://hledger.org/sqlite.html); [hledger#2017](https://github.com/simonmichael/hledger/issues/2017). Other hledger docs pages cover dsq/DataStation and Ultorg. — [hledger dsq](https://hledger.org/dsq.html); [hledger Ultorg](https://hledger.org/ultorg.html). The 2026 release notes mention SQL output format across more commands. — [hledger relnotes](https://hledger.org/relnotes.html)
- **duckdb_ledger** is a "DuckDB extension to query Ledger plain-text accounting files". 1 star, created 2026-02-06, updated 2026-07-19. — [caetanosauer/duckdb_ledger](https://github.com/caetanosauer/duckdb_ledger)
- Small pandas converters: **beancount2dataframe** (5 stars, last updated 2024-06) and **beancount_pandas** (0 stars, 2019). **[older]** — [dimonf/beancount2dataframe](https://github.com/dimonf/beancount2dataframe); [dimonf/beancount_pandas](https://github.com/dimonf/beancount_pandas)
- **beancount-sqlite** has 2 stars (2025). — [BarrySong97/beancount-sqlite](https://github.com/BarrySong97/beancount-sqlite)
- A 2026 personal-finance repo advertises "a plain-text double-entry ledger an LLM can query over SQL" (1 star). — [geofftang/programmatic-finance-tracker](https://github.com/geofftang/programmatic-finance-tracker)
- **zhang** has a `zhang-sql` component (SQL storage backend). — [zhang](https://github.com/zhang-accounting/zhang)
- **rustledger**'s query engine is its own BQL implementation in Rust (rustledger-query depends on chumsky and rayon). Its README and roadmap mention no Arrow, DataFrame or SQL-database export. — [crates.io rustledger-query](https://crates.io/crates/rustledger-query); [rustledger roadmap](https://rustledger.github.io/roadmap/)

### Inferences
- A Rust core that emits Arrow RecordBatches (postings, balances, prices, lots) through the Arrow PyCapsule interface would get Polars, pandas, PyArrow and DuckDB SQL "for free" (see Q3). None of the surveyed tools does this.
- Decimal fidelity is the recurring pain point in SQL exports (SQLite has no decimal type; hledger exports `numeric`). Arrow `Decimal128(38, s)` (or string-encoded amounts plus commodity) should be designed deliberately. This is an inference, not sourced.

### Gaps
- I did not check whether Fava has an Arrow or pandas export path.
- The duckdb_ledger implementation approach (native C++ versus shell wrapper; GitHub reports "Shell" as its language) was not inspected.

---

## Q3. What architecture building blocks and precedents exist (PyO3/maturin, Arrow zero-copy, exact decimals, parser choices, incremental parsing/LSP, WASM, file watching)?

### Takeaway
The Rust core + PyO3 + maturin pattern is mature and widely proven (Polars, pydantic-core, tokenizers, orjson, cryptography, tiktoken, datafusion-python). Arrow zero-copy to Python is solved by pyo3-arrow and the Arrow PyCapsule interface. Rust PTA projects have converged on `rust_decimal`, which is 96-bit with about 28 significant digits and *not* arbitrary precision. rustledger itself flags that as its top compatibility risk. For parsing, projects use logos + chumsky, winnow, nom or tree-sitter. rowan (lossless CST), tree-sitter (incremental) and salsa (incremental computation) are the standard LSP-grade tools.

### Cited Findings

**PyO3 / maturin**
- PyO3 has 16.2k stars and is at version 0.29.2. It supports CPython 3.9+, PyPy 7.3 (3.11+) and GraalPy 25+. The README names **maturin** as the primary build tool and lists users including polars, pydantic-core, tokenizers, orjson, cryptography, tiktoken, datafusion-python, deltalake-python, opendal, obstore, arro3, jsonschema and granian. — [PyO3 GitHub](https://github.com/PyO3/pyo3)
- Stable ABI: PyO3 supports abi3/abi3t (e.g. `features = ["abi3-py310", "abi3t-py315"]`). Free-threaded CPython 3.14t cannot use abi3, so maturin builds a version-specific `cp314-cp314t` wheel. abi3t (PEP 803) is aimed at 3.15. — [PyO3 building and distribution](https://pyo3.rs/v0.29.2/building-and-distribution); [maturin bindings guide](https://www.maturin.rs/bindings); [maturin#3064](https://github.com/PyO3/maturin/issues/3064)
- watchfiles (a Rust-based file watcher for Python) builds abi3 wheels with maturin, which shows the file-watching use case already ships this way. — [watchfiles PR #396](https://github.com/samuelcolvin/watchfiles/pull/396)

**Arrow zero-copy**
- pyo3-arrow "implements zero-copy FFI conversions between Python objects and Rust representations using the arrow crate", relying on the **Arrow PyCapsule Interface**. It negotiates schemas on export and works with PyArrow, Polars, DuckDB, nanoarrow and pandas (ArrowDtype). — [lib.rs pyo3-arrow](https://lib.rs/crates/pyo3-arrow); [docs.rs pyo3_arrow](https://docs.rs/pyo3-arrow); [arro3](https://github.com/kylebarron/arro3)
- pyo3-arrow is at 0.19.0 (2026-06-15), MIT/Apache, about 7.3M downloads. — [crates.io pyo3-arrow](https://crates.io/crates/pyo3-arrow)
- Polars 1.44.2 was released on PyPI on 2026-09-09. — [PyPI polars](https://pypi.org/project/polars/)

**Exact decimals**
- `rust_decimal` is a 128-bit value: a 96-bit integer mantissa plus scale (0–28) plus sign, giving "roughly 28 base-10 digits (29 in some cases)". It is **not** arbitrary precision; the docs point to `bigdecimal`/`decimal-rs` for that. It offers configurable RoundingStrategy and serde (float/string/arbitrary-precision modes). — [docs.rs rust_decimal](https://docs.rs/rust_decimal/latest/rust_decimal/). Version 1.43.0 (2026-09-02), MIT, about 148M downloads. — [crates.io rust_decimal](https://crates.io/crates/rust_decimal)
- rust_decimal is used by rustledger (workspace dep `rust_decimal = 1.42`), beancount-parser-lima, tackler-core and okane. — [rustledger Cargo.toml](https://github.com/rustledger/rustledger/blob/main/Cargo.toml); [crates.io deps for lima](https://crates.io/crates/beancount-parser-lima), [tackler-core](https://crates.io/crates/tackler-core), [okane-core](https://crates.io/crates/okane-core)
- rustledger's roadmap calls arbitrary-precision decimals the "single most-cited reason a ledger could produce different numbers". — [rustledger roadmap](https://rustledger.github.io/roadmap/)
- Ledger (C++) links GMP and MPFR (arbitrary-precision rationals and floats). — [ledger CMakeLists.txt](https://github.com/ledger/ledger/blob/master/CMakeLists.txt)
- Beancount uses Python's `decimal` (the C parser has `decimal.c`). Python's `decimal` offers correctly rounded decimal arithmetic with user-settable precision (default context precision 28). — [beancount repo tree](https://github.com/beancount/beancount); [Python decimal docs](https://docs.python.org/3/library/decimal.html)

**Parsers and incremental tooling (what existing Rust PTA projects actually use)**
- rustledger uses `logos` 0.16 (lexer), `rowan` 0.17 (lossless CST, from rust-analyzer), `chumsky` 1.0-alpha (BQL), `ropey` (rope for LSP), `imbl` (persistent collections), `jiff` (dates), `miette` (diagnostics) and `wasm-bindgen`. — [rustledger Cargo.toml](https://github.com/rustledger/rustledger/blob/main/Cargo.toml)
- beancount-parser-lima uses logos, chumsky and ariadne (zero-copy). beancount-parser uses nom 8. okane and tackler use winnow 1.0. uromyces and beancount-language-server use tree-sitter. — [crates.io dependency lists](https://crates.io/crates/beancount-parser-lima); [uromyces](https://github.com/yagebu/uromyces)
- Crate snapshots on 2026-09-25: winnow 1.0.4 (about 935M downloads, MIT), logos 0.16.1, rowan 0.18.0-alpha.1 (2026-09-14), salsa 0.28.5 (2026-09-24), tree-sitter 0.27.0 (2026-08-30). — [crates.io winnow](https://crates.io/crates/winnow); [logos](https://crates.io/crates/logos); [rowan](https://crates.io/crates/rowan); [salsa](https://crates.io/crates/salsa); [tree-sitter](https://crates.io/crates/tree-sitter)
- rowan is "a generic library for lossless syntax trees", originally built for rust-analyzer. — [docs.rs rowan](https://docs.rs/rowan)
- Tree-sitter is "a parser generator tool and an incremental parsing library" that updates the syntax tree efficiently as the file is edited. — [tree-sitter GitHub](https://github.com/tree-sitter/tree-sitter)

**WASM**
- rustledger-wasm compiles the engine to WASM (browser and Node via npm). rustledger-ffi-component targets WASI Preview 2 / Component Model. rustfava embeds that component in Python via `wasmtime`. — [rustledger README](https://github.com/rustledger/rustledger); [rustfava](https://github.com/rustledger/rustfava)

### Inferences
- A sensible layering, following precedent: a pure-Rust core crate with no Python types, then a thin `pyo3` crate (maturin, abi3 wheels plus separate free-threaded wheels), `pyo3-arrow` for DataFrame export, a `wasm-bindgen` crate and a CLI crate. This mirrors polars (polars crates + py-polars) and pydantic-core.
- The decimal choice should be made on purpose. rust_decimal is fast and ubiquitous but capped at about 28 digits. That diverges from Python Decimal and Ledger's GMP in edge cases, which is rustledger's own top-cited divergence. Options include rust_decimal with overflow detection, `bigdecimal`/`fastnum`, or an i128 fixed-point with a per-commodity scale. This needs benchmarking.
- For an LSP later: a lossless CST (rowan) or tree-sitter for the editor layer, plus salsa-style incremental queries (uromyces already considered salsa for Fava). rustledger has not yet shipped incremental reparse ("range-based reparse" is on its roadmap).

### Gaps
- No PTA-specific benchmark compares rust_decimal against bigdecimal against Python Decimal.
- I did not research file-watching crates (`notify`) beyond noting that watchfiles uses the same PyO3/maturin pattern.
- I did not confirm whether ruff is built with PyO3. The PyO3 README list does not include ruff; ruff ships as a Rust binary distributed via PyPI wheels, but that was not verified in this session.

---

## Q4. Which Ledger/hledger/Beancount test suites could serve as conformance suites, and under what licenses?

### Takeaway
There are four large, reusable corpora. Ledger has about 4,000 regression tests plus about 300 baseline tests under a BSD license. hledger has about 560 shelltest files, including 352 adapted from Ledger's baseline, under GPL-3.0+. Beancount has 81 Python `*_test.py` modules under GPL-2.0-only. rustledger's **pta-standards** repo is a new MIT-licensed attempt at cross-implementation conformance specs.

### Cited Findings
- **Ledger** `test/` holds `baseline` (331 files, 312 `.test`), `regress` (4,145 files, 4,043 `.test`), `fuzz` (255), `input` (14), `manual` (11), `unit` (12), `python` (5), `semantic` (3) and `todo` (15). Runners include `RegressTests.py`, `CheckBaselineTests.py`, `LedgerHarness.py` and `DocTests.py`. — [ledger/test](https://github.com/ledger/ledger/tree/master/test) (file counts from a shallow clone of master @ 2026-09-22)
- The Ledger license is BSD-style ("Copyright (c) 2003-2025, John Wiegley. All rights reserved. Redistribution and use ... permitted"). — [ledger LICENSE.md](https://github.com/ledger/ledger/blob/master/LICENSE.md)
- **hledger** functional tests use shelltestrunner, with `*.test` files under `hledger/test/` grouped by component and error-message tests in `hledger/test/errors/`. Unit tests (tasty) ship inside the executable (`hledger test`), and there are doctests too. — [hledger TESTS](https://hledger.org/TESTS.html)
- Counts: 562 `.test` files in total, 553 of them under `hledger/test/`. Of those, **352 are in `hledger/test/ledger-compat`** (Ledger baseline tests run against hledger), 45 journal, 38 errors, 29 balance, 10 register and 10 print. — [hledger/test](https://github.com/simonmichael/hledger/tree/master/hledger/test) (shallow clone @ 2026-09-24)
- The hledger changelog says it tests reading Ledger's baseline sample journals, with a success rate that improved from 80% to 90% (search summary; exact version not verified). — [hledger CHANGES](https://hledger.org/CHANGES-cli.html)
- hledger's license is GPL-3.0-or-later. — [hledger LICENSE](https://github.com/simonmichael/hledger/blob/master/LICENSE)
- **Beancount** has 81 `*_test.py` files plus a C `tokens_test.c`, under GPL-2.0-only. — [beancount repo](https://github.com/beancount/beancount); [COPYING](https://github.com/beancount/beancount/blob/master/COPYING)
- beancount-parser-lima's test suite is "based on" Beancount's tests (GPLv2 protobuf test files), with anomalies annotated. — [docs.rs beancount_parser_lima](https://docs.rs/beancount-parser-lima/latest/beancount_parser_lima/)
- **pta-standards** (rustledger org) provides draft specs for Beancount v3, Ledger v1 and hledger v1. It includes EBNF/ABNF grammars, JSON Schema and Protobuf ASTs, tree-sitter grammars, Alloy formal models and conformance suites. Nightly results show 266–267 passing tests against Beancount v3, with 2–3 failures per implementation. Code and tests are MIT, docs CC-BY-4.0. 17 stars, "in development". — [rustledger/pta-standards](https://github.com/rustledger/pta-standards)
- rustledger runs a nightly compatibility workflow with a badge, and claims 100% on 759 real-world files. — [rustledger README](https://github.com/rustledger/rustledger); [announcement](https://forum.plaintextaccounting.org/t/announcing-rustledger-beancount-reimagined-in-rust/725)
- tackler has 520 tracked test vectors in a separate repo (Apache-2.0 project). — [tackler](https://github.com/tackler-ng/tackler)

### Inferences
- **Using the tests as external oracles** (run your binary against their inputs and compare outputs, without copying them into your repo) avoids most license entanglement. Vendoring hledger's (GPL-3+) or Beancount's (GPL-2-only) test files into a permissively licensed repo is riskier. Ledger's BSD-licensed tests are the easiest to vendor. This is a legal inference, not legal advice.
- pta-standards (MIT) is the most directly reusable cross-format conformance asset, but it is young and controlled by the rustledger project.

### Gaps
- I did not verify how many of Ledger's 4,043 regress tests are fuzz-generated versus hand-written.
- I did not confirm the exact size of rustledger's "759 real-world test files" corpus or whether it is published.

---

## Q5. What performance benchmarks exist comparing Ledger, hledger, Beancount and the Rust implementations on large journals?

### Takeaway
On a 10k-transaction ledger, rustledger reports about 34–43 ms against Ledger's 63–114 ms, hledger's about 500 ms and Beancount's about 0.8–1.0 s. The hardware is unspecified and the numbers are self-published. hledger's own 2026 numbers are 17k–52k txns/s on 100k transactions. tackler claims 700k–900k txns/s in its own format. For personal-scale ledgers every tool finishes in seconds or less. Beancount's time goes into booking and plugins more than parsing.

### Cited Findings
- **rustledger (self-reported, nightly, 10k txns):** validation takes rustledger about 43 ms, Beancount about 789 ms, Ledger about 108 ms and hledger about 498 ms. A balance report takes about 34 ms, 1,046 ms, 63 ms and 538 ms respectively. Memory use is "3-5x less" than Python Beancount. No hardware is stated. — [rustledger comparison](https://rustledger.github.io/about/comparison.html)
- The January 2026 announcement gave 34 ms (rustledger), 880 ms (Beancount) and 534 ms (hledger) for 10k txns, "~25x faster". — [announcement](https://forum.plaintextaccounting.org/t/announcing-rustledger-beancount-reimagined-in-rust/725). A search snippet also quoted Ledger at 114 ms. — [rustledger site via search](https://rustledger.github.io/)
- The benchmarks can be reproduced with `nix develop .#bench ./scripts/bench.sh` or `cargo bench` (Criterion). — [rustledger GitHub](https://github.com/rustledger/rustledger)
- **hledger (MacBook Pro M5 Pro, `examples/100ktxns-1kaccts.journal`):**

  | Version | stats | balance | print | register | Throughput |
  |---|---|---|---|---|---|
  | hledger 1.25 | 2.70 s | 2.68 s | 3.24 s | 71.99 s | 37k txns/s |
  | hledger 1.52 | 4.29 s | 4.06 s | 4.42 s | 20.73 s | about 23k txns/s |
  | hledger 1.99.4 | 5.96 s | 5.80 s | 6.32 s | 19.02 s | about 17k txns/s |
  | hledger main (2026) | 2.00 s | 2.15 s | 2.84 s | 14.17 s | 52k txns/s |

  The page describes main as "about 3x faster than 1.99.4, 2x faster than 1.52". — [hledger and Ledger](https://hledger.org/ledger.html)
- hledger documents its benchmarking method (quickbench, `hledger stats` txns/s, `_perf.test`) but does not publish tables on that page. — [hledger BENCHMARKS](https://hledger.org/BENCHMARKS.html). Issue #2153 tracks a "performance regression with many accounts" in versions 1.29–1.32.2. — [hledger#2153](https://github.com/simonmichael/hledger/issues/2153)
- **tackler:** "900 000 transactions per second on modern laptop" (current README); an older listing says 700,000. — [tackler README](https://github.com/tackler-ng/tackler); [lib.rs](https://lib.rs/crates/tackler)
- **uromyces** versus Beancount: parsing about 2x, booking about 40x and plugins about 10x faster. — [uromyces](https://github.com/yagebu/uromyces)
- **Beancount real-world breakdown [older, Feb 2019, Beancount v2]:** Blais's roughly 4 MB ledger loaded in 4,529 ms on an Intel NUC. Parsing took 740 ms, booking 1,219 ms, plugins 1,470 ms and validation 450 ms. Another user's single-account journal was approaching 11 MB. — [Beancount with large journals](https://groups.google.com/g/beancount/c/t22lS5635nA/m/k8IIBxiBGAAJ)
- **Fava in 2025:** a ledger of 15k+ entries over 5+ years took "several seconds" to render the journal page (28.5 MB response). The bottleneck was Jinja template rendering, not Beancount parsing, and pagination fixed it. — [Xidorn's blog, 2025-11-04](https://blog.upsuper.org/en/2025/fava-journal/)
- A vendor blog (beancount.io) claims hledger handles 1M transactions in about 80 s (about 12.5k txns/s) and that Beancount loads "hundreds of thousands" of transactions in about 2 s. It is unverified marketing content, not a primary benchmark, and was only seen in a search summary. — [beancount.io blog](https://beancount.io/blog/2025/07/22/beancounts-technical-edge-a-deep-dive-on-performance-python-api-and-data-integrity-vs-ledger-hledger-and-gnucash)

### Inferences
- Ledger (C++) is only about 2–3x slower than rustledger on these self-reported numbers. The big gap is against Beancount (about 20–30x) and hledger (about 10–15x).
- For typical personal ledgers (roughly 5k–50k transactions), a Rust core turns load times from about 1–5 s into under 100 ms. That matters most for interactive uses (LSP, watch mode, web UI reloads, notebooks re-running loads) rather than one-shot reports.
- Because Beancount's cost is dominated by booking and plugins, and Fava's by rendering, a fast parser alone does not fix end-to-end latency. Booking, plugins and the output and presentation layers matter as much.

### Gaps
- No independent, same-hardware benchmark of rustledger, Ledger, hledger, Beancount, tackler and limabean exists. The rustledger and tackler numbers are self-reported with unspecified hardware.
- Ledger's 2026 numbers on hledger's 100k journal were not extracted from the hledger page.
- limabean has no published performance numbers.

---

## Q6. What are the trade-offs of a pure Python core versus a Rust core with Python bindings for a personal-finance tool (journal sizes, dev velocity, contributor accessibility, packaging)?

### Takeaway
Personal ledgers are small, typically thousands to low tens of thousands of transactions and a few MB, so a pure-Python core is "fast enough" for batch reports; Beancount loaded about 4 MB in about 4.5 s in 2019. A Rust core pays off for interactive and embedded uses (LSP, watch, WASM, notebooks, big multi-year journals), and PyO3/maturin packaging is now routine. The costs are contributor accessibility, a dual-language build, decimal-semantics decisions and ABI/wheel matrices. There is also a strategic question: build on rustledger (GPL-3, CLA, no PyO3) or on permissive crates (lima/limabean-booking, okane).

### Cited Findings
- Real ledger sizes: 15k+ entries over 5+ years (2025 blog); about 4 MB for Blais and about 11 MB for a heavy user **[2019]**. — [Xidorn 2025](https://blog.upsuper.org/en/2025/fava-journal/); [Beancount large journals thread](https://groups.google.com/g/beancount/c/t22lS5635nA/m/k8IIBxiBGAAJ)
- Where time goes in Python Beancount: booking (1.2 s) plus plugins (1.5 s) outweigh parsing (0.74 s) on a 4 MB ledger **[2019]**. — [Beancount large journals thread](https://groups.google.com/g/beancount/c/t22lS5635nA/m/k8IIBxiBGAAJ)
- Rust speedups reported: 10–30x overall (rustledger); about 40x on booking and about 10x on plugins (uromyces). — [rustledger comparison](https://rustledger.github.io/about/comparison.html); [uromyces](https://github.com/yagebu/uromyces)
- Python extensibility is the ecosystem's main asset. rustledger had to embed a CPython-WASI sandbox to run existing Python plugins, and it cannot run plugins with C extensions. — [rustledger README](https://github.com/rustledger/rustledger)
- A pure-Rust core can drop users' in-process Python API. limabean exposes no Python at all, and rustledger exposes Python only via WASM/JSON DTOs (hand-maintained compat layer, generated Pydantic types). — [limabean](https://github.com/tesujimath/limabean); [rustledger ADR 0004](https://github.com/rustledger/rustledger/blob/main/docs/reference/adr/0004-ts-types-from-rust-dtos.md)
- PyO3 packaging: abi3 gives one wheel per platform for all GIL CPython versions ≥ the minimum. Free-threaded 3.14t needs its own wheel, and abi3t is coming with 3.15. — [PyO3 building and distribution](https://pyo3.rs/v0.29.2/building-and-distribution); [maturin bindings](https://www.maturin.rs/bindings)
- Large production projects (polars, pydantic-core, cryptography, tokenizers, orjson) ship PyO3/maturin wheels. — [PyO3 README](https://github.com/PyO3/pyo3)
- Decimal semantics: rust_decimal's roughly 28-digit cap differs from Python Decimal and from GMP-backed Ledger, and rustledger cites decimals as the #1 source of divergent numbers. — [docs.rs rust_decimal](https://docs.rs/rust_decimal/latest/rust_decimal/); [rustledger roadmap](https://rustledger.github.io/roadmap/)
- Licenses constrain reuse. rustledger is GPL-3.0-only with a CLA, Beancount GPL-2.0-only and hledger GPL-3.0-or-later. Ledger is BSD, tackler Apache-2.0, lima and limabean MIT/Apache, okane MIT, and beancount-parser and ledger-parser Unlicense. — [crates.io rustledger-core](https://crates.io/crates/rustledger-core); [beancount COPYING](https://github.com/beancount/beancount/blob/master/COPYING); [hledger LICENSE](https://github.com/simonmichael/hledger/blob/master/LICENSE); [ledger LICENSE.md](https://github.com/ledger/ledger/blob/master/LICENSE.md); [crates.io licenses](https://crates.io/crates/beancount-parser-lima)
- Alternative embedding route: rustfava uses wasmtime to run the Rust engine as a WASM component inside Python. That means one artifact for all platforms, but data crosses the boundary serialized. — [rustfava](https://github.com/rustledger/rustfava)

### Inferences
- **Recommended direction (for the report writer to weigh):** a Rust core plus a thin PyO3 layer that exposes (a) native Python objects for directives and postings with Decimal fidelity and (b) Arrow tables for DataFrame and DuckDB work. That is the differentiator no existing project offers. Keep plugin and extension hooks in Python, because Python extensibility is what users value most in Beancount.
- **Do not rebuild what rustledger already does** (Beancount parsing, BQL, LSP, WASM) unless licensing or Python-API goals require it. The options are:
  - (a) If GPL-3 is acceptable, wrap rustledger crates with PyO3, and possibly upstream the bindings. This is the lowest effort, but it adds a dependency on a fast-moving single-maintainer project.
  - (b) If a permissive license is required, compose beancount-parser-lima with limabean-booking (MIT/Apache) and okane or a new winnow-based parser for Ledger/hledger syntax.
  - (c) Stay pure Python and treat Rust as an optional accelerator later. This gives the fastest dev velocity and easiest contributions, and is sufficient for typical journal sizes.
- A pure-Python-first core with a clean, typed data model, later swapped for or accelerated by Rust behind the same API (the pydantic v1→v2 path), lowers early risk. That only works if the data model and Decimal semantics are fixed up front.
- Contributor accessibility: Rust raises the bar for plugin and report authors. Keeping reports and plugins in Python, with only parse, book, validate and query in Rust, preserves accessibility.

### Gaps
- There is no survey data on the distribution of personal journal sizes across PTA users. The size figures above are anecdotal (a few individuals).
- I found no published postmortem comparing dev velocity or contributor counts between pure-Python and Rust+PyO3 finance tools.
- CI and wheel-build cost (time, platform matrix) for a PyO3 PTA library was not measured.
