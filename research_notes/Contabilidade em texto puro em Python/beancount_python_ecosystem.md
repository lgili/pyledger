# Beancount (v2/v3), Fava and the Python plain-text-accounting ecosystem (state as of Sept 2026)

Method note: facts below come from primary sources: PyPI JSON metadata, git clones of the repositories (commit/tag dates read directly), the Beancount docs sources (github.com/beancount/docs), mailing-list threads, GitHub issues and blog posts. Benchmarks marked "own measurement" were run for this research on 2026-09-25 (Beancount 3.2.3 wheel, beanquery 0.2.0, CPython 3.11.15, 4-vCPU Intel Xeon @ 2.80 GHz VM). They are single runs on synthetic data, so treat them as indicative only. beancount.io is a **third-party commercial site** (not the Beancount project) with marketing-style content. Its claims are flagged where used.

## 1. Beancount history, v2 vs v3 status, the package split, and maintenance 2024-2026

### Takeaway
Beancount v3 shipped on 2024-06-16 as a *trimmed* Python release. The long-planned C++ core was not included and has been shelved. Tooling was split into beanquery, beangulp and beanprice. Development continues at a slow, mostly single-maintainer pace: the latest release is 3.2.3 (2026-05-05). Martin Blais has since run Rust, Go and DuckDB experiments, but none of them has shipped.

### Cited Findings
- **Origins.** Beancount was started around 2007 by Martin Blais. The README says v1 "was intended to be similar to and partially compatible with Ledger". Development of v1 stopped in 2013. v2 was "a complete rewrite… which introduced a number of constraints and a new grammar". v2 was in maintenance mode from 2020 to 2024 and is "now frozen and obsolete". v3 is "the current stable version of Beancount since June 2024… trimmed down from v2 and most of the tools the v2 branch included have been moved to their own independent projects" — [README.rst](https://github.com/beancount/beancount/blob/master/README.rst)
- **Release timeline** (git tags and PyPI):
  - 2.3.5: 2022-02 ("only a few minor bugfixes. Mainly released for 3.10")
  - 3.0.0: 2024-06-16
  - 3.1.0: 2025-01-19
  - 3.2.0: 2025-09-14
  - 3.2.1 and 3.2.2: 2026-04-28/30
  - 3.2.3: 2026-05-05 (latest on PyPI; requires Python >=3.9)
  - PyPI lists 37 releases in total — [PyPI beancount](https://pypi.org/project/beancount/); [CHANGES](https://github.com/beancount/beancount/blob/master/CHANGES)
- **The branch reorganisation at the v3 release (2024-06-16), from CHANGES:**
  - The previous master "with all C++ and Bazel build" became a `cpp` branch.
  - "All work on the C++ rewrite -- if it is to continue -- will live on that branch. If things do move to Rust, I will probably salvage bits and pieces of that manually into a 'rust' branch."
  - `v3` became "the official release branch".
  - PyPI releases were made for beangulp, beanquery, beanprice, beangrow and beancount2ledger.
  - Source: [CHANGES](https://github.com/beancount/beancount/blob/master/CHANGES)
- **The original v3 plan (July 2020).** The core, parser, booking and plugins were to be rewritten "in simple C++, outputting its parsed and booked contents as a stream of protobuf objects", built with Bazel. The query engine was to be forked into a separate, general-purpose tool "closer to that of the Pandas library". The public API was to be consolidated under `import beancount as bn`. The pickle cache was to be removed — [Beancount v3 Goals & Design (docs source)](https://github.com/beancount/docs/blob/master/docs/beancount_v3.md) / [rendered](https://beancount.github.io/docs/beancount_v3/)
- **The C++ work was shelved.** In an Oct 27-28, 2024 mailing-list thread, Martin Blais "noted the C++ rewrite project is shelved, with preserved source code in a separate branch" — [Beancount v3 upgrade documentation thread](https://groups.google.com/g/beancount/c/3ZSrTKDpPy0). The last commit on the `cpp` branch is dated 2024-11-07 — [cpp branch](https://github.com/beancount/beancount/tree/cpp)
- **What v3 actually is:**
  - Still Python, with a C flex/bison lexer and parser that calls back into Python: `beancount/parser/lexer.l`, `grammar.y`, `parser.c`.
  - Built with meson / meson-python.
  - Runtime dependencies: only `click`, `python-dateutil` and `regex`.
  - Console scripts are reduced to `bean-check`, `bean-doctor`, `bean-example`, `bean-format` and `treeify`. There is no `bean-query`, `bean-report`, `bean-web`, `bean-extract` or `bean-price` in core.
  - Sources: [pyproject.toml](https://github.com/beancount/beancount/blob/master/pyproject.toml); [parser dir](https://github.com/beancount/beancount/tree/master/beancount/parser)
- **Split-out packages** (PyPI and git):
  - **beanquery**: 0.1.dev0 (2024-06-16), then 0.2.0 (2025-03-24). Last commit 2026-06-11. Only 3 PyPI releases, still 0.x — [PyPI beanquery](https://pypi.org/project/beanquery/); [repo](https://github.com/beancount/beanquery)
  - **beangulp**: 0.1.0 (2024-05-29), then 0.2.0 (2025-01-20). Last commit 2026-05-30 — [PyPI beangulp](https://pypi.org/project/beangulp/); [repo](https://github.com/beancount/beangulp)
  - **beanprice**: 2.1.0 (2025-10-18) — [PyPI beanprice](https://pypi.org/project/beanprice/)
  - **beangrow** (portfolio returns): last commit 2025-10-14 — [repo](https://github.com/beancount/beangrow)
- **Maintenance activity since 2024-01-01** (git log on master):
  - Commit bursts in 2024-06 (49), 2024-11 (46), 2024-12 (39), 2025-01 (17), 2026-01 (15) and 2026-04 (15). Most other months have 1-8 commits.
  - Authors: blais/Martin Blais (~137), Trim21 (~45), Daniele Nicolodi (36), Jakob Schnitzer (9).
  - Source: [commit history](https://github.com/beancount/beancount/commits/master)
- **Notable changes after 3.0:**
  - Type annotations, `py.typed` and mypy (Dec 2024).
  - Windows/MSVC builds (Nov 2024).
  - `import beancount as bn` root API in `beancount/api.py`. Its docstring says the "API may change over time, though we're not expecting to remove any symbols on the v3 branch".
  - Balance directives allowed on non-leaf accounts (2025-01-14).
  - New `display_precision` option; `inferred_tolerance_multiplier` renamed to `tolerance_multiplier` (May 2025).
  - Python 3.14 support and a `--json` flag for errors (Jan 2026).
  - A rounding inconsistency in transaction interpolation fixed in 3.2.1 (Apr 2026), plus a new `use_precise_interpolation` option to make that fix optional (May 2026).
  - Sources: [commit history](https://github.com/beancount/beancount/commits/master); [api.py](https://github.com/beancount/beancount/blob/master/beancount/api.py)
- **Experimental branches:**
  - `rust` (last merge 2025-05-04; contains `experiments/rust/…` including an "llm-conversion" of the C core and a prost/protobuf crate) — [rust branch](https://github.com/beancount/beancount/tree/rust)
  - `golang`, whose last commit is "Gemini 3.1 port of Beancount to Go (experiment)" (2026-03-22) — [golang branch](https://github.com/beancount/beancount/tree/golang)
  - `grammar-in-c.codex` ("Dispatch parser grammar actions directly in C", 2026-04-16) — [branch](https://github.com/beancount/beancount/tree/grammar-in-c.codex)
- **DuckDB experiment.** On 2026-04-18 master gained an experiment titled "Implement Beancount to DuckDB adapter and advanced interactive CLI". It flattens postings into a DuckDB `postings` table with `DECIMAL(38,18)` and STRUCT types (Amount, Cost, Position), aiming for "column name compatibility with beanquery" and "DuckDB's columnar storage and vectorized execution for fast aggregations" — [experiments/duckdb/design.md](https://github.com/beancount/beancount/blob/master/experiments/duckdb/design.md)
- **Documentation.** Docs are still authored in Google Docs and converted to Markdown ([docs README](https://github.com/beancount/beancount/tree/master/docs); [beancount/docs repo](https://github.com/beancount/docs), last commit 2026-04-19). Old `.html` doc URLs (e.g. `/docs/beancount_v3.html`) now return 404. The new directory-style URLs (`/docs/beancount_v3/`) return 200. This was observed on 2026-09-25 against [beancount.github.io/docs](https://beancount.github.io/docs/).
- **Community size** (plaintextaccounting.org apps table): Beancount has ~103 committers, ~5.5k stars, a mailing list of ~880 and a Fava Matrix room of ~260. hledger has ~194 committers and ~4.4k stars — [plaintextaccounting.org](https://plaintextaccounting.org/)

### Inferences
- "v3" means a repackaging and modularisation, not a faster engine. The performance-motivated redesign (C++, protobuf and no cache) never shipped. The maintainer is now exploring LLM-assisted ports (Rust, Go) and DuckDB, which signals openness to a different core, but nothing is committed.
- The 3.2.x rounding and interpolation changes show that numeric semantics are still being adjusted in 2026. A compatible reimplementation has to track these options.

### Gaps
- There is no official roadmap for Beancount v4 or for the Rust and Go experiments. Their status comes only from branch contents and commit messages.
- I could not locate the git tag for 3.2.3 in the shallow clone. Its date comes from PyPI.

## 2. Beancount syntax, philosophy, strictness, booking, plugins, and features missing compared with Ledger

### Takeaway
Beancount is deliberately "pessimistic" and minimal:
- Strict typing: 5 root account types, mandatory `open`, date-ordered (not file-ordered) assertions, and strict lot matching.
- A small directive set, extended by Python plugins over a directive stream.

It intentionally lacks several Ledger features: virtual postings, automated and periodic transactions, time of day, effective dates and an expression language. Average-cost booking is still unimplemented in 2026.

### Cited Findings
- **Philosophy.** "Ledger is optimistic… In contrast, Beancount is highly pessimistic. It assumes the user is unreliable." Assertions are dated rather than file-ordered. Beancount "does not provide support for unbalanced/virtual postings; it's not a shortcoming, it's on purpose" — [A Comparison of Beancount and Ledger (Sept 2014)](https://github.com/beancount/docs/blob/master/docs/a_comparison_of_beancount_and_ledger_hledger.md)
- **Five account types.** "Beancount accounts must have a particular type from one of five categories: Assets, Liabilities, Income, Expenses and Equity." Ledger has no such constraint — [comparison doc](https://beancount.github.io/docs/a_comparison_of_beancount_and_ledger_hledger/)
- **Strictness.** "Any transaction in an account needs to have an open directive… If you maintain a Beancount ledger, you can expect to have to normalize it to fix a number of common errors being reported." — [comparison doc](https://github.com/beancount/docs/blob/master/docs/a_comparison_of_beancount_and_ledger_hledger.md)
- **Directives in the language syntax doc:** open, close, commodity, transactions, balance, pad, note, document, price, event, query and custom, plus options such as operating currencies — [Beancount Language Syntax](https://github.com/beancount/docs/blob/master/docs/beancount_language_syntax.md)
- **Balance and pad semantics.**
  - A balance assertion "applies at the **beginning** of its date (i.e., midnight at the start of day)".
  - Every account has an implicit zero balance at its Open date.
  - `pad` "automatically inserts a transaction that will make the subsequent balance assertion succeed", where "subsequent" means date order.
  - Source: [syntax doc](https://github.com/beancount/docs/blob/master/docs/beancount_language_syntax.md)
- **Cost basis and conversions.** Beancount separates plain currency conversions (`@` price, no cost memory) from holdings "at cost" (`{}`), which keep per-lot cost and date and are reduced under strict matching rules. An account may not hold long and short lots of the same commodity at the same time — [comparison doc](https://github.com/beancount/docs/blob/master/docs/a_comparison_of_beancount_and_ledger_hledger.md)
- **Booking methods.** The v3 `Booking` enum contains STRICT, STRICT_WITH_SIZE, NONE, AVERAGE, FIFO, LIFO and HIFO — [core/data.py](https://github.com/beancount/beancount/blob/master/beancount/core/data.py)
  - AVERAGE is only an enum value. `booking_method_AVERAGE` returns the error `"AVERAGE method is not supported"`, and the old code is marked "DISABLED" — [parser/booking_method.py](https://github.com/beancount/beancount/blob/master/beancount/parser/booking_method.py)
  - Issue #940 "Enable AVERAGE cost booking" was opened 2025-02-09 and is still open. It notes that the feature was removed in 2021 (PR #591) and that France legally requires average cost ("also accepted in Singapore and Australia") — [issue #940](https://github.com/beancount/beancount/issues/940)
- **Built-in plugins (v3):** auto, auto_accounts, check_average_cost, check_closing, check_commodity, check_drained, close_tree, coherent_cost, commodity_attr, currency_accounts, implicit_prices, leafonly, noduplicates, nounused, onecommodity, pedantic, sellgains, unique_prices — [beancount/plugins](https://github.com/beancount/beancount/tree/master/beancount/plugins)
- **Extension model.** The loader produces "a single ordered list of directives". Plugins are Python functions that transform that stream, "not an API to access its data at the edges". Ledger's automated transactions are replaced by plugins — [comparison doc](https://github.com/beancount/docs/blob/master/docs/a_comparison_of_beancount_and_ledger_hledger.md)
- **No time of day or effective dates.** "Beancount does not represent the intra-day time of transactions, its granularity is a day". Effective (auxiliary) dates were removed. There is a settlement-date proposal — [comparison doc](https://github.com/beancount/docs/blob/master/docs/a_comparison_of_beancount_and_ledger_hledger.md)
  - An `adding_time` branch ("Added parsing of time for transactions only (no tests)") has been dormant since 2020-05-25 — [branch](https://github.com/beancount/beancount/tree/adding_time)
- **No forecasting or periodic transactions.** "Beancount has no support for generating periodic transactions for forecasting", apart from a simplistic example `forecast` plugin — [comparison doc](https://github.com/beancount/docs/blob/master/docs/a_comparison_of_beancount_and_ledger_hledger.md). Third-party tools fill the gap, e.g. beanahead ("Administer future transactions", uses pandas frequencies; last commit 2026-09-14) — [beanahead](https://github.com/maread99/beanahead)
- **Known modelling gaps listed by the author in the v3 doc (2020):**
  - Stock splits: "Right now, Beancount ignores the issue."
  - Self-reductions within a single transaction are unsupported, which is "unintuitive to some users".
  - Proposed but unimplemented: `balance … > amount` inequality assertions, `balanceall`, and debit/credit sign normalisation.
  - Source: [v3 doc](https://github.com/beancount/docs/blob/master/docs/beancount_v3.md)
- **v3 syntax changes.** "The syntax became more permissive, requiring no journal modifications for upgrades" (Oct 2024 thread) — [mailing list](https://groups.google.com/g/beancount/c/3ZSrTKDpPy0)

### Inferences
- Strictness is Beancount's main differentiator (it catches errors, and gives correct lot and capital-gains tracking). It is also the main source of friction: mandatory opens, no virtual postings, and day-only granularity.
- A new system could keep the strict validation model while adding the missing primitives as first-class, opt-in features: AVERAGE booking, time of day, periodic and automated transactions, and stock splits.

### Gaps
- I did not find an official, current (2025-2026) statement from Blais on AVERAGE booking beyond the open issue.
- I did not re-verify the full list of `option` names for v3; the options reference doc structure did not parse cleanly.

## 3. Beancount Python API ergonomics, BQL/beanquery, and performance

### Takeaway
The Python API is Beancount's strongest asset:
- `loader.load_file` returns plain, immutable, typed NamedTuples with `Decimal` numbers.
- Plugins and scripts work directly on this list.

Its limits:
- Performance is poor at scale: own measurement ≈13 s for a 100k-transaction ledger, ≈41 s for 300k, ~1.3 GB RSS.
- The pickle cache that works around this still exists.
- beanquery is a capable but 0.x, sparsely documented SQL dialect.
- There is no native DataFrame output.

### Cited Findings
- **Loader API.** `loader.load_file()` returns `(entries, errors, options_map)`. It still ships a pickle cache (`.{filename}.picklecache`, created when loading takes longer than `PICKLE_CACHE_THRESHOLD = 1.0` s), although the 2020 plan was to remove it — [loader.py](https://github.com/beancount/beancount/blob/master/beancount/loader.py); [v3 doc "Caching"](https://github.com/beancount/docs/blob/master/docs/beancount_v3.md)
- **Data model.** Directives such as `Transaction(NamedTuple)` (meta, date, flag, payee, narration, tags, links, postings) are immutable NamedTuples. They were rewritten in class syntax with type annotations in Dec 2024 ("refactor: rewrite NamedTuple with class", py.typed) — [core/data.py](https://github.com/beancount/beancount/blob/master/beancount/core/data.py); [commits](https://github.com/beancount/beancount/commits/master)
- **Other core modules.** `beancount/core` also contains realization.py (account tree realization), inventory.py, position.py, prices.py, convert.py, getters.py, interpolate.py and display_context.py — [core dir](https://github.com/beancount/beancount/tree/master/beancount/core)
- **v3 root API.** v3 exposes `import beancount as bn; bn.load_file(...)` "like e.g., numpy" — [api.py](https://github.com/beancount/beancount/blob/master/beancount/api.py)
- **beanquery features:**
  - Describes itself as a "customizable lightweight SQL query tool that works on tabular data, including Beancount ledger data" — [beanquery README](https://github.com/beancount/beanquery)
  - 0.1 added HAVING, `empty()`, `round()`, per-column ORDER direction and cast functions `int()`/`decimal()`/`str()`/`date()`. `str()` was renamed `repr()`, a breaking change for old queries — [CHANGES.rst](https://github.com/beancount/beanquery/blob/master/CHANGES.rst)
  - Early 2025 (released as 0.2.0 on 2025-03-24) added `CREATE TABLE AS`, CSV files as data sources in `beanquery.connect()`, table names without the `#` prefix, quoted identifiers and a structured date type — [beanquery commits](https://github.com/beancount/beanquery/commits/master)
- **beanquery documentation.** In Oct 2024 Daniele Nicolodi said beanquery documentation "is under development… but remains sparse" and suggested reading merged PRs and tests — [mailing list](https://groups.google.com/g/beancount/c/3ZSrTKDpPy0)
- **Author's own performance goal (July 2020).** "My current file takes 6 seconds on the souped-up NUC… but that's just too long… I really do want that 'instant' feeling… well under half a second." — [v3 doc](https://github.com/beancount/docs/blob/master/docs/beancount_v3.md)
- **Mailing-list profile** (~4 MB ledger, Intel NUC, circa the C++-plan era): total load 4,529 ms, of which parser 740 ms, booking 1,219 ms, plugins 1,470 ms and validation 450 ms. Blais called parsing, booking and plugins "the big hitters". Another user's journal approached 11 MB — ["Beancount with large journals"](https://groups.google.com/g/beancount/c/t22lS5635nA/m/k8IIBxiBGAAJ)
- **Own measurements** (Beancount 3.2.3, cache disabled):

  | Ledger | Transactions | Size | Full load | Parse only | Other |
  |---|---|---|---|---|---|
  | `bean-example` output, 2000-2025 (18,730 directives, 29,548 postings) | 9,560 | 2.9 MB | 1.57 s | 0.56 s | beanquery connect + GROUP BY account query: 2.22 s |
  | Synthetic | 100k (2 postings each) | 8.5 MB | 12.98 s | 5.13 s | RSS ≈448 MB |
  | Synthetic | 300k | 25.7 MB | 41.15 s | 16.15 s | RSS ≈1.3 GB |

  - With the pickle cache on the 100k file: the first run takes 17.4 s (it writes the cache) and the second 3.7 s. This is own measurement; see the method note.
- **Conflicting claim.** The third-party site beancount.io states that Beancount can process "hundreds of thousands of transactions in approximately 2 seconds" — [beancount.io blog (2025-07-22)](https://beancount.io/blog/2025/07/22/beancounts-technical-edge-a-deep-dive-on-performance-python-api-and-data-integrity-vs-ledger-hledger-and-gnucash). This is contradicted by the author's own 6 s figure for a personal ledger and by the measurements above. A 2022 HN user reported "2+ seconds" of parsing with hundreds of thousands of transactions — [HN 2022](https://news.ycombinator.com/item?id=30138434)

### Inferences
- Load time grows roughly linearly at about 130 µs per simple transaction on this VM. Parsing is only ~40% of it; booking, interpolation, plugins and validation dominate, which matches Blais's profile. Replacing only the parser will not fix it: booking, plugins and validation need a native core too.
- Ergonomic gaps a new library could close:
  - Columnar/DataFrame (Arrow, polars, pandas) views of postings, balances and prices.
  - Incremental or cached loads without pickle.
  - A stable, documented query layer (SQL via DuckDB is the direction Blais himself is exploring).

### Gaps
- I found no official benchmark suite for v3. `tools/benchmark.py` exists in the repo but was not run.
- The mailing-list profile post date was not visible in the fetched page.

## 4. Fava (web UI)

### Takeaway
Fava is the most actively maintained part of the ecosystem: 1.30.16 was released 2026-08-18 and it has 62 PyPI releases. It offers charts, a BQL query page with charts and exports, an editor with a tree-sitter grammar, entry forms, an import UI (beangulp), budgets via `custom` directives, and JS-capable extensions. It dropped Beancount v2 in May 2026. Its main complaint is performance on large ledgers, with a slow server-rendered journal.

### Cited Findings
- **Release history** — [Fava CHANGES](https://github.com/beancount/fava/blob/main/CHANGES):
  - Latest: 1.30.16 (2026-08-18); first PyPI release 1.3 in 2017-03; requires Python >=3.10 — [PyPI fava](https://pypi.org/project/fava/)
  - v1.30 (2024-12-29): Beancount 3 support; queries via beanquery ("minor differences in syntax… leads to breakage" for extensions using query_shell); beangulp importers on v3; duplicate detection no longer automatic; Svelte 5 frontend.
  - v1.30.13 (2026-05-19): "drops support for Beancount version 2… you will have to switch importers to use beangulp". Translations moved to Weblate. The `unrealized` fava-option was removed.
  - Unreleased (2026): fiscal-year and quarter intervals, a rewritten date-filter parser (the `to` range separator was dropped), and Decimal/Amount metadata round-tripping.
- **Dependencies.** Current Fava depends on `beancount>=3.2.0,<4`, `beangulp>=0.2`, `beanquery>=0.1,<0.3`, Flask, Jinja2, Werkzeug, cheroot and msgspec — [pyproject.toml](https://github.com/beancount/fava/blob/main/pyproject.toml)
- **Features:**
  - Editor with autocompletion.
  - Query page running bean-query queries. It shows line or treemap charts when there are two columns (date/string + inventory), and exports CSV (xlsx/ods with the `fava[excel]` extra).
  - Add-transaction form.
  - Up-to-date indicators driven by `fava-uptodate-indication` metadata.
  - External-editor links.
  - Source: [features.md](https://github.com/beancount/fava/blob/main/src/fava/help/features.md)
- **Budgets** are `custom "budget"` directives with daily, weekly, monthly, quarterly or yearly periods. They are "broken down to a daily budget" and shown in income-statement charts and reports — [budgets.md](https://github.com/beancount/fava/blob/main/src/fava/help/budgets.md)
- **Import UI.** It uses an `import-config` and `import-dirs` and supports only Transaction, Balance and Note entries. `__source__` metadata displays the raw CSV row or XML fragment. HOOKS can be beangulp-style — [import.md](https://github.com/beancount/fava/blob/main/src/fava/help/import.md)
- **Extensions.**
  - Since v1.25 (2023-07-17) extensions can ship frontend JavaScript; since v1.26 they can add endpoints.
  - v1.29 (2024-10-09) moved query results to frontend rendering, added dark mode, numerical filter comparisons and an optional polling watcher.
  - v1.28 switched to watchfiles.
  - Source: [CHANGES](https://github.com/beancount/fava/blob/main/CHANGES)
  - Popular extension fava-dashboards is very active: 113 commits since 2025, last 2026-09-15 — [fava-dashboards](https://github.com/andreasgerstmayr/fava-dashboards)
- **Performance** — [Xidorn's blog (2025-11-04)](https://blog.upsuper.org/en/2025/fava-journal/):
  - A user with more than 15,000 entries measured 2.08 s waiting time and a 28.50 MB response for the journal page.
  - The time was mostly spent in Jinja template rendering, with postings making up more than 60% of the rendered content.
  - Fixes contributed upstream: skip price entries, paginate and load the rest concurrently, and CSS flex layout.
  - Older issues: "fava very slow to start" — [#1031](https://github.com/beancount/fava/issues/1031); "Minimum system requirements" — [#582](https://github.com/beancount/fava/issues/582)

### Inferences
- Fava inherits Beancount's full-reload model: a file change triggers a full `load_file`. At the measured 13 s per 100k transactions, big ledgers make Fava sluggish no matter how the frontend is optimised.
- Fava's hard dependency on Beancount's Python data structures, and on beanquery below 0.3, means any faster core must provide a Beancount-compatible Python object model to reuse Fava. rustledger explicitly lists "Fava web interface (until rustledger integration)" as a reason to stay on Python Beancount.

### Gaps
- I found no systematic Fava benchmark. The evidence consists of individual reports.

## 5. Importing in the Beancount world (beangulp, smart_importer, beancount-import, reds importers, BeanHub)

### Takeaway
Importing is fragmented and was disrupted by v3:
- `beancount.ingest` became beangulp 0.2, with a script-based workflow and an API change.
- The largest third-party importer collection (reds) still had no released beangulp version as of its latest README.
- ML categorisation exists: smart_importer (scikit-learn SVC, "beta") and beancount-import (decision tree + web UI).
- BeanHub offers an MIT, YAML-rule-based alternative (beanhub-import/extract/cli) whose bank-sync "Connect" feature is paid SaaS.

### Cited Findings
- **beangulp.** beangulp "is the evolution of `beancount.ingest` in Beancount 2.3 and replaces it in the Beancount 3.0 release". Documentation is mostly "code docstrings and examples" — [beangulp README](https://github.com/beancount/beangulp). Migration requires converting config files into scripts and updating the importer base class and method signatures — [v3 upgrade thread](https://groups.google.com/g/beancount/c/3ZSrTKDpPy0); announcement: ["beancount.ingest is no more. Long live beangulp!"](https://groups.google.com/g/beancount/c/YhBQEh7xVdk)
- **smart_importer** — [smart_importer README](https://github.com/beancount/smart_importer):
  - Provides hooks (e.g. `PredictPostings`) that train on existing entries.
  - "The model is implemented using… scikit-learn… SVC (support vector machine)".
  - Status: "Working protoype, development status: beta".
  - Latest release 1.2 (2025-10-17); 31 commits since 2025.
- **beancount-import (jbms)** — [beancount-import](https://github.com/jbms/beancount-import); [PyPI](https://pypi.org/project/beancount-import/):
  - Offers a web UI, OFX, Mint, Amazon and Venmo sources, a decision-tree classifier for unknown legs, and matching/merging of imported and manual transactions.
  - PyPI last release 1.4.0 (2024-04-19). Only 5 commits since 2025 (last 2026-08-20).
- **beancount_reds_importers** — [reds README](https://github.com/redstreet/beancount_reds_importers):
  - Supports OFX, CSV, XLSX and PDF readers.
  - Its README says: "being transitioned from Beancount v2 to Beangulp… The Beangulp version is not yet ready for release… if you want v2, use version 0.10.0". PyPI 0.10.0 was released 2025-02-24.
  - The repo is active: 126 commits since 2025, last 2026-04-29.
- **BeanHub open source (all MIT)** — [beanhub-cli](https://github.com/LaunchPlatform/beanhub-cli); [BeanHub open source page](https://beanhub.io/open-source/):
  - beanhub-cli 3.3.0 (2026-06-21, 180 commits since 2025)
  - beanhub-import 1.4.1 (2026-06-20)
  - beanhub-extract 0.1.7 (2025-12-20), with extractors for chase, citi, csv, fidelity, mercury, plaid and wealthsimple
  - beanhub-forms, beanhub-inbox, beancount-parser (Lark-based, 1.2.3 on 2024-05-09)
  - beancount-black formatter (1.0.6 on 2026-09-03)
  - beancount-data (pydantic models)
  - beancount-exporter (GPL-2.0, JSON export)
- **beanhub-import design** — [beanhub-import](https://github.com/LaunchPlatform/beanhub-import):
  - "Simple declarative rules — a single import file for all imports" in YAML + Jinja2.
  - "Idempotent" output; it auto-updates existing transactions.
- **BeanHub SaaS-only features:**
  - "Connect" imports from "12,000+ financial institutions in 17 countries" and requires "a BeanHub paid account" — [BeanHub Connect blog](https://beanhub.io/blog/2024/06/24/introduction-of-beanhub-connect/); [beanhub-cli README](https://github.com/LaunchPlatform/beanhub-cli)
  - Inbox (email archiving + LLM extraction) needs a BeanHub account.
  - Most CLI features work without an account.

### Inferences
- Import tooling is the biggest day-to-day workload for plain-text-accounting users. The Python ecosystem offers several incompatible paradigms: class-based importers (beangulp), rule-based YAML (BeanHub), and interactive ML reconciliation (beancount-import). v3 broke the old one, and the main community library still lags behind.
- A new system could make the import interface a stable, first-class API with rules, ML and dedup built in, and offer adapters for beangulp importers.

### Gaps
- BeanHub pricing: I found no pricing page (`beanhub.io/pricing/` returned 404).
- I did not quantify how many institution-specific packages (beancount-dkb, beancount-n26, etc.) have migrated to beangulp.

## 6. Other Python tools around ledger/hledger formats, and DataFrame access

### Takeaway
Outside Beancount, Python plain-text-accounting tools are mostly small, single-purpose or dormant: ledger-autosync, ledgerhelpers, icsv2ledger (dormant since 2019), pyhledger, and Ledger's own Boost.Python bindings. reckon is Ruby, not Python. No mature library exposes a ledger as pandas/polars DataFrames:
- beancount2dataframe: 6 commits, last 2024.
- ledgerkit: hledger-format, pandas optional, first dev release in 2026.
- Blais's DuckDB experiment is unreleased.

### Cited Findings
- **ledger-autosync** (OFX/CSV sync to ledger): 1.2.0 (2024-08-22), last commit 2025-09-15. It can use "ledger python bindings… Note, however, they can be buggy, which is why they are disabled by default" — [ledger-autosync](https://github.com/egh/ledger-autosync); [PyPI](https://pypi.org/project/ledger-autosync/)
- **ledgerhelpers** (GUI data-entry helpers for Ledger; "extends Ledger's python library" per plaintextaccounting.org): PyPI last 0.3.10 (2022-12-02), last commit 2025-08-05 — [ledgerhelpers](https://github.com/Rudd-O/ledgerhelpers); [plaintextaccounting.org](https://plaintextaccounting.org/)
- **icsv2ledger** (interactive CSV→ledger): last commit 2019-03-26, not on PyPI — [icsv2ledger](https://github.com/quentinsf/icsv2ledger)
- **reckon** is **Ruby**. It outputs and learns in LEDGER or BEANCOUNT format; last commit 2025-07-24 — [reckon](https://github.com/cantino/reckon)
- **Ledger's own Python bindings** ([src/pyledger.cc](https://github.com/ledger/ledger/blob/master/src/pyledger.cc)): Ledger NEWS for 3.5.0 lists fixes for "many crashes in the Python bindings", including segfaults in `journal.query()` and use-after-free in Python journal queries — [Ledger NEWS.md](https://github.com/ledger/ledger/blob/master/NEWS.md)
- **pyhledger** — Python library/scripts to parse hledger files; last commit 2026-06-09, 37 commits — [pyhledger](https://github.com/btittelbach/pyhledger)
- **ledgerkit** — [ledgerkit](https://github.com/ctosullivan/ledgerkit); [PyPI ledgerkit](https://pypi.org/project/ledgerkit/):
  - "A deterministic, Python-native accounting and query engine with a documented hledger-compatible foundation", for hledger 1.52 journal format.
  - CLI: balance, register, print, accounts, stats, check.
  - "Optional pandas DataFrame export"; pure Python; GPLv3+.
  - First commit 2026-04-15; PyPI 1.0.0.dev1.
- **ledger-cli-toolkit (imports as "ledgerpy")** — reads .ledger into JSON-like structures, with CSV/PDF export and DB integration. PyPI 1.8.6; active 2025-01 to 2025-08 — [ledger-cli-toolkit](https://github.com/EddyBel/ledger-cli-toolkit)
- **beancount2dataframe** — "reads beancount files… and returns pandas DataFrames". 6 commits, 2019-10 to 2024-06 — [beancount2dataframe](https://github.com/dimonf/beancount2dataframe)
- **Other Beancount-side tools:**
  - beangrep 0.6.0 (2024-08-04), a grep-like filter — [PyPI](https://pypi.org/project/beangrep/)
  - pinto 0.1.4 (2020-06-14), a Beancount CLI (dormant) — [PyPI](https://pypi.org/project/pinto/)
  - beanahead (future/recurring transactions with pandas) — [beanahead](https://github.com/maread99/beanahead)
- **medici.** PyPI `medici` is an empty placeholder: version 0.0.1 with no files, pointing to github.com/stevewedig/medici, which could not be cloned — [PyPI medici](https://pypi.org/project/medici/)
- **plutus** (nickjj, 2025): a "zero dependency Python script" for income/expense tracking from a single CSV, i.e. not double-entry plain-text accounting. Last commit 2026-07-24 — [plutus](https://github.com/nickjj/plutus)
- **Workarounds.** The common way to get DataFrames is to run BQL queries and convert the rows manually, per community posts summarised by a search engine — [mailing list post](https://groups.google.com/g/beancount/c/_hWFZf2al68/m/1Wb2G9rfBQAJ). Blais's own 2026 DuckDB adapter design flattens postings into a columnar table — [design.md](https://github.com/beancount/beancount/blob/master/experiments/duckdb/design.md)

### Inferences
- "Ledger journal → typed DataFrame/Arrow table" is an open niche for both Beancount and ledger/hledger formats. Existing offerings are toy-scale, dormant or pre-release.

### Gaps
- Not verified: whether ledger-autosync can emit Beancount directly. Its README section I read did not mention it.
- I did not verify medici's original purpose (the repo is gone).

## 7. Faster reimplementations of Beancount (context for the "fast core" gap)

### Takeaway
A fast Beancount-compatible core now exists in Rust: rustledger, started 2025/2026 and very active in 2026. It claims 10-30x speed and ships a CLI, LSP, WASM, an MCP server and Python *plugin* compatibility via a WASI sandbox. It offers **no native Python library** (no PyO3 package on PyPI). Other attempts are narrower:
- limabean: Rust + Clojure, with no Python and no BQL.
- bean-rs: alpha PyO3, dormant since 2024.

### Cited Findings
- **rustledger overview** — [rustledger README](https://github.com/rustledger/rustledger); [releases](https://github.com/rustledger/rustledger/releases):
  - Describes itself as "A blazing-fast Rust implementation of Beancount… 10-30x faster".
  - "Drop-in replacement… Compatible `bean-*` CLI commands"; "BQL (100% compat)".
  - LSP server, "MCP server for AI assistants", WASM (npm `@rustledger/wasm`), 31 built-in plugins "plus Python plugin compatibility via WASI sandbox".
  - CSV/OFX import with dedup and categorisation, XIRR/TWR returns ("a built-in beangrow replacement"), Fava-compatible budgets, and capital gains per lot.
  - Licensed GPL-3.0.
- **rustledger's own comparison table** labels Python beancount "Speed: Slow" and "Active development: Maintenance". It says to use Python Beancount if "You need Fava web interface (until rustledger integration)". Its benchmarks run nightly on 10K-transaction ledgers and it claims "3-5x less memory than Python beancount" — [rustledger README](https://github.com/rustledger/rustledger)
- **rustledger activity.** First commit in the repo is 2026-01-03; 3,221 commits by 2026-09-24; tag v0.24.0 on 2026-09-06 (git clone). A LinuxLinks and search summary reported v0.18.0 on 2026-07-01 — [releases](https://github.com/rustledger/rustledger/releases). plaintextaccounting.org lists it with start 2025, ~1 committer, ~66 stars — [plaintextaccounting.org](https://plaintextaccounting.org/)
- **No native Python package.** No `pyo3` in rustledger's Cargo manifests (repo grep). The PyPI names `rustledger`, `rustledger-py` and `pyrustledger` do not exist. Python is supported only as sandboxed plugins ("File-based plugins (.py files) run in a sandboxed CPython compiled to WebAssembly") — [rustledger README](https://github.com/rustledger/rustledger)
- **limabean** — [limabean](https://github.com/tesujimath/limabean); [Show HN](https://news.ycombinator.com/item?id=47239083):
  - "A new implementation of Beancount using Rust and Clojure and the Lima parser".
  - UI is "solely the Clojure REPL, with no support for Beancount Query Language nor Python".
- **bean-rs** (PyPI 0.3.1, 2024-04-29): "Basic beancount clone (one day...) in Rust… Still very very alpha… Python bindings are a WIP using PyO3". Its to-do list includes includes, currency conversions and "Price/cost and FIFO" — [PyPI bean-rs](https://pypi.org/project/bean-rs/)
- **Other Rust Beancount parsers** (plaintextaccounting.org "Libraries"): beancount-parser-lima (Logos + Chumsky) and beancount-parser (nom) — [plaintextaccounting.org](https://plaintextaccounting.org/)

### Inferences
- The "fast core" half of the proposed system is already being addressed for Beancount syntax by rustledger. The **first-class Python library** half is not. No project offers native-speed loading *and* an idiomatic, typed Python object model with DataFrame output usable from notebooks and scripts (and by Fava).
- rustledger's single-maintainer profile (~1 committer) and very fast AI-assisted commit rate (3.2k commits in ~9 months) are a sustainability question. Its CLAUDE.md and AGENTS.md files are present in the repo.

### Gaps
- I could not independently run rustledger benchmarks (no Rust toolchain used). The 10-30x figure is the project's own claim.

## 8. Main complaints and pain points of Beancount/Fava users

### Takeaway
The recurring pain points are:
- Performance on large or long-lived ledgers.
- Strictness and learning curve (mandatory opens, lot matching, no virtual postings).
- v3 migration friction: removed tools, beangulp rewrite, sparse beanquery docs, a lagging importer library, and Fava v2 support dropped.
- Missing investment features: AVERAGE booking, stock splits.
- Weak budgeting and forecasting.
- Reporting that requires writing SQL.
- Mobile/data-entry friction.

### Cited Findings
- **v3 migration friction** (Oct 2024) — [mailing list](https://groups.google.com/g/beancount/c/3ZSrTKDpPy0):
  - "I haven't found any good Beancount v3 upgrade documentation" (Martin Michlmayr).
  - `bean-report`, `bean-extract` and `bean-web` were removed.
  - Fava was not yet compatible at the time.
  - beancount2ledger was untested with v3.
  - Beanquery docs are sparse.
- **Importers and Fava lag v3.** The reds importers' beangulp version was "not yet ready for release" ([reds README](https://github.com/redstreet/beancount_reds_importers)). Fava's v1.30 BQL move caused breakage for extensions ([Fava CHANGES](https://github.com/beancount/fava/blob/main/CHANGES)).
- **Investment tracking.** No AVERAGE booking, required in France and the Canadian default per community sources ([issue #940](https://github.com/beancount/beancount/issues/940); [mailing list "State of average cost booking?"](https://groups.google.com/g/beancount/c/SP5KeksHbCk)). Stock splits are ignored and self-reductions are unsupported ([v3 doc](https://github.com/beancount/docs/blob/master/docs/beancount_v3.md)).
- **Performance.** Evidence: the author's 6 s ledger; a 4 MB → 4.5 s profile; 11 MB journals; own measurement of 13 s per 100k transactions; Fava journal slowness ([v3 doc](https://github.com/beancount/docs/blob/master/docs/beancount_v3.md); [mailing list](https://groups.google.com/g/beancount/c/t22lS5635nA/m/k8IIBxiBGAAJ); [Xidorn 2025](https://blog.upsuper.org/en/2025/fava-journal/))
- **HN "Beancount: Double-entry accounting from text files" (2022-01-30)** — [HN 30138434](https://news.ycombinator.com/item?id=30138434):
  - "Beancounts budgeting system is fairly poor. It can't handle envelope budgeting"
  - reports "has never been as easy as QB. It often takes a little bit of Beancount SQL querying"
  - "I am not editing a structured text file on my phone"
  - strictness "feels at times like working with a static type system"
  - parsing "2+ seconds" with hundreds of thousands of transactions
  - praise for strict cost-basis handling and Python/pandas programmability
- **Strictness.** The author acknowledges users "can expect to have to normalize [the ledger] to fix a number of common errors"; virtual postings are refused ("If you feel strongly that you should need them, you should use Ledger") — [comparison doc](https://github.com/beancount/docs/blob/master/docs/a_comparison_of_beancount_and_ledger_hledger.md)
- **Numeric semantics churn.** Interpolation rounding was changed in 3.2.1 (Apr 2026), and the precise mode was made optional (`use_precise_interpolation`) in 3.2.3 — [commits](https://github.com/beancount/beancount/commits/master)
- **2026 usage signal.** An HN commenter (Apr-May 2026) reported switching from QuickBooks to Beancount + Fava for a sole proprietorship, adding text-based invoice and mileage tracking plus validators — [HN 47902612](https://news.ycombinator.com/item?id=47902612) (search snippet only).

### Inferences
- The complaints group into speed, stability of the toolchain and APIs, missing domain features, and ergonomics (entry, reporting). A new system with a fast core, a stable versioned Python API, and batteries-included import/report/DataFrame layers would address the first, second and fourth groups directly.

### Gaps
- Reddit r/plaintextaccounting threads could not be retrieved via search, so first-hand Reddit complaint quotes are missing.
- Multi-currency quirks (e.g. unrealized gains, `operating_currency` conversions, price-vs-cost semantics) lack a sourced, dated user complaint in these notes. The price/cost distinction itself is documented in the comparison doc.

## 9. Which "Python version of Ledger that isn't very good" did the user probably mean?

### Takeaway
- **Most likely overall: Beancount.** It is by far the best-known Python plain-text-accounting tool, and v1 began as a Ledger-like, partially compatible Python program. It is not a Ledger clone, though, and is widely regarded as capable, so "isn't very good" fits it less well.
- **Most likely literal match for "a Python version of Ledger that isn't very good": pledger** (pcapriotti/pledger, 2011). It reads most Ledger files but openly has far fewer features.
- **Also plausible:**
  - mafm/ledger.py ("like Ledger, but simpler", dormant since 2018).
  - Ledger's own buggy Python bindings.
- **Not a match:** everything literally named "pyledger" on PyPI/GitHub (smart-contract ledger, business-accounting apps, a YNAB clone).

### Cited Findings
- **Beancount v1:** "intended to be similar to and partially compatible with Ledger… Do not use this." — [Beancount README](https://github.com/beancount/beancount/blob/master/README.rst)
- **pcapriotti/pledger** — [pledger](https://github.com/pcapriotti/pledger):
  - "inspired by John Wiegley's ledger, and most ledger input files should work with pledger without modifications"; "pledger has nowhere near as many command line options as ledger"; "rules inside ledger files are ignored".
  - Commits: 112 in 2011, 1 in 2012, 1 in 2016, 36 in 2021, 5 in 2024 (last 2024-09-02). Not on PyPI (`pledger` not found).
- **mafm/ledger.py** — "Ledger.py is like John Wiegley's Ledger, but simpler". Uses its own syntax variant (e.g. `VERIFY-BALANCE`). Activity 2013-04 to 2018-06 (43 commits) — [ledger.py](https://github.com/mafm/ledger.py)
- **Ledger's Python bindings** (`import ledger`): ledger-autosync disables them by default because "they can be buggy" ([ledger-autosync](https://github.com/egh/ledger-autosync)); Ledger 3.5.0 NEWS lists many Python-binding crash fixes ([NEWS.md](https://github.com/ledger/ledger/blob/master/NEWS.md))
- **Projects named "pyledger":**
  - PyPI `pyledger` 0.5 (2017-05-10) is "A simple ledger for smart contracts written in Python" — [PyPI](https://pypi.org/project/pyledger/); [guillemborrell/pyledger](https://github.com/guillemborrell/pyledger) (2017-2018)
  - [dickhfchan/pyledger](https://github.com/dickhfchan/pyledger) (2025-07 to 2026-07) is a "headless Python accounting application" with SQLite, invoices, REST and MCP. It is not plain-text accounting.
  - [Omar-Abdalaziz/PyLedger](https://github.com/Omar-Abdalaziz/PyLedger) (created 2026-09-17), a general-ledger app.
  - [wmei1/pyLedger](https://github.com/wmei1/pyLedger) (2017), a "YNAB clone" in PyQt5/SQLite.
- **Other names checked:**
  - `pledger`, `python-ledger` and `medger`: not on PyPI.
  - `ledgerpy` / `ledger.py` on PyPI is "python module for Ledger Nano S" (the crypto hardware wallet, 2019) — [PyPI ledgerpy](https://pypi.org/project/ledgerpy/)
  - `ledger` on PyPI is a "Geektrust ledger problem" (2022) — [PyPI ledger](https://pypi.org/project/ledger/)
  - woodruffw/pledger is a **Rust** monthly-expense tool (archived) — [woodruffw/pledger](https://github.com/woodruffw/pledger)
  - "ledgerpy" is also the import name of EddyBel's ledger-cli-toolkit (2025) — [ledger-cli-toolkit](https://github.com/EddyBel/ledger-cli-toolkit)
  - No plain-text-accounting project named "medger" was found.

### Inferences
- If the user remembers a tool that *reads Ledger files* and is weak, it is pledger (or ledger.py). If they remember "the Python one" that is a different dialect, strict and slow, it is Beancount. A good question to ask them: did it read Ledger-syntax files, or did it have its own syntax with `open` directives? The answer separates the two.
- The name "pyledger" is effectively free as a plain-text-accounting brand on GitHub, but PyPI `pyledger` is taken (the 2017 smart-contract package).

### Gaps
- I cannot know the user's memory. The ranking is by name similarity, popularity and the "not very good" descriptor.

## 10. Gap a new system (fast core + first-class Python library + CLI) could fill

### Takeaway
No current option combines all three:
- A fast native core (rustledger has one but offers Python only as a WASI plugin sandbox).
- A first-class, stable, typed Python library with columnar/DataFrame output (Beancount has a good object model but is slow; ledgerkit and beancount2dataframe are toy-scale).
- A batteries-included CLI covering reports, queries and import (Beancount v3 moved these into 0.x side packages).

The niche is a native-core ledger engine exposed through an idiomatic Python API, ideally Beancount-syntax compatible or importable. Its requirements are listed in Inferences below.

### Cited Findings
- Beancount performance: 6 s author ledger; own measurement 13 s / 100k transactions; pickle cache still needed — see section 3 ([v3 doc](https://github.com/beancount/docs/blob/master/docs/beancount_v3.md); [loader.py](https://github.com/beancount/beancount/blob/master/beancount/loader.py))
- The C++ core is shelved, and the Rust/Go work is experimental — [CHANGES](https://github.com/beancount/beancount/blob/master/CHANGES); [mailing list](https://groups.google.com/g/beancount/c/3ZSrTKDpPy0); [golang branch](https://github.com/beancount/beancount/tree/golang)
- rustledger is fast but has no Python library; Fava integration is pending — [rustledger](https://github.com/rustledger/rustledger)
- Blais's own direction toward a DuckDB columnar model — [design.md](https://github.com/beancount/beancount/blob/master/experiments/duckdb/design.md)
- Missing features: AVERAGE booking ([issue #940](https://github.com/beancount/beancount/issues/940)); time of day, periodic transactions, virtual/automated postings ([comparison doc](https://github.com/beancount/docs/blob/master/docs/a_comparison_of_beancount_and_ledger_hledger.md))
- Import fragmentation and v3 breakage — [reds README](https://github.com/redstreet/beancount_reds_importers); [beangulp](https://github.com/beancount/beangulp); [BeanHub](https://beanhub.io/open-source/)

### Inferences
Requirements for a system that fills the gap:
- **Fast native core** (Rust + PyO3 is the obvious route; rustledger shows feasibility), with sub-second loads for 100k+ transactions and incremental reloads instead of pickle caches.
- **First-class Python API:** typed dataclasses/NamedTuples compatible with or convertible to `beancount.core.data`, so Fava and existing plugins could run; zero-copy Arrow export to polars/pandas/DuckDB; a stable semver API.
- **Batteries-included CLI:** check, format, balance/register/BS/IS reports, SQL queries, import and price fetching in one versioned package.
- **Correctness features Beancount lacks:** AVERAGE booking, stock splits, optional time of day, periodic/forecast and automated postings, budgets beyond Fava's `custom` directive.
- **Import as a stable plugin API,** with rule-based YAML/Python rules, ML categorisation and dedup built in.

Risks and competition:
- rustledger is moving extremely fast and could add PyO3 bindings.
- Beancount upstream might adopt DuckDB or Rust.
- GPL licensing: Beancount is GPL-2.0-only and rustledger is GPL-3.0. Reusing their code constrains the new project's license.

### Gaps
- No user survey quantifies demand for a Python-native fast core versus a standalone Rust CLI.
- I found no data on how many Fava users hit performance limits (ledger-size distributions).
