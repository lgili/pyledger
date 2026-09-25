# Ledger (ledger-cli, C++) and hledger (Haskell) as of September 2026: features, maintenance, Python usability, pain points

Method note: release dates, commit counts and changelog contents below were checked directly against git clones of `github.com/ledger/ledger` and `github.com/simonmichael/hledger` (now `github.com/hledgerorg/hledger`) made on 2026-09-25, plus the official docs. Package-index facts were checked against the PyPI JSON API and packages.debian.org on the same day. When a claim comes only from a search-engine snippet or a vendor blog, it is marked that way.

## 1. Ledger: current version, release cadence, maintenance status, core features, syntax quirks, performance

### Takeaway
Ledger's latest tagged release is **3.4.1 (2025-10-25)**. It came after 3.4.0 (2025-10-22), which ended a gap of about 2.5 years since 3.3.2 (2023-03-30). From Feb 2026 onward, John Wiegley did a very large AI-assisted rewrite and hardening effort (~1,700 commits in 2026, many with Claude co-author trailers). That work is written up in NEWS.md as **3.5.0 ("more than 575 merged pull requests")**, but as of 2026-09-25 **3.5.0 has no tag**. The feature set is the richest in plain text accounting: value expressions, automated and periodic transactions, virtual postings, lot annotations, `--market`/`--exchange`, and csv/xml/lisp output. Much of the fix work, including automatic FIFO/LIFO lot matching and most Python binding crash fixes, exists only in unreleased `main`.

### Cited Findings
**Versions and cadence (from git tags)**
- Tag dates: 3.1.2 (2019-02-12), 3.1.3 (2019-03-31), 3.2.0 (2020-05-01), 3.2.1 (2020-05-18), 3.3.0 (2023-02-08), 3.3.1 (2023-03-03), 3.3.2 (2023-03-30), **3.4.0 (2025-10-22), 3.4.1 (2025-10-25)**. No v3.5.0 tag exists as of 2026-09-25. — [ledger tags](https://github.com/ledger/ledger/tags)
- 3.4.1's full changelog is two items: "Fix version number in binary" and "Fix crash in optimized builds when exceptions are thrown (issue #693)". — [NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md)
- 3.4.0 highlights: segfault/divide-by-zero/use-after-free fixes; `--hashes` hash chaining for transaction integrity; commodity swaps over a base commodity; `==~` regex capture operator; `--align-intervals`; nested `account` sub-directives; Python min version raised to 3.10 (3.9 minimum), Python 3.12 deprecation fixes, Python 2 test support removed; C++17, Boost 1.86 (min 1.72), CMake 3.16.2. — [NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md)
- A commit on 2026-04-30 is titled "Draft NEWS.md entry for the upcoming v3.5.0 release". Commits continue after it: fixes such as #3252 and #3268, a fuzzing-driven fix "recursion depth limit to expression parser", and use-after-free fixes. The most recent commit is 2026-09-22. — [ledger commits](https://github.com/ledger/ledger/commits/main)

**Maintenance and activity (git clone, 2026-09-25)**
- Monthly commit counts: 1–2 per month in 2022–2024 (e.g., 2024-04: 2, 2024-09: 1, 2024-12: 1). Then 2025-08: 50, 2025-12: 50, **2026-02: 666, 2026-03: 646, 2026-04: 282**, 2026-05: 49, 2026-06: 18, 2026-07: 9, 2026-08: 13, 2026-09: 8. Since 2025-01-01, John Wiegley authored 1,798 commits. The next-largest contributors had 41, 24 and 18. — [ledger commits](https://github.com/ledger/ledger/commits/main)
- 969 commit messages since 2025 carry `Co-Authored-By: Claude …` trailers (e.g., 480 "Claude Opus 4.6", 341 "Claude Sonnet 4.6"). The repo has a `CLAUDE.md`. In Aug 2026 it added a `ledger-semantics` submodule (Lean formalization, repo `github.com/ledger/ledger-semantics`) and a "semantic-bisimulation harness" that runs in CI, gated on the Lean "oracle". — [ledger commits](https://github.com/ledger/ledger/commits/main), [.gitmodules](https://github.com/ledger/ledger/blob/main/.gitmodules)
- The GitHub repo page reported ~6,000 stars, 549 forks, **15 open issues** and 2 open PRs. (These numbers come from a page fetch and could not be cross-checked through the API, which is blocked in this environment.) — [github.com/ledger/ledger](https://github.com/ledger/ledger)
- The draft 3.5.0 notes say the release "represent[s] more than 575 merged pull requests". They also list: 419 new regression tests, source coverage raised to 89.6% with a 90% CI gate, regression tests for "153 previously-uncovered open issues (#3011)", a libFuzzer harness on the journal parser run nightly, and ASan/UBSan CI. — [NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md)
- The draft 3.5.0 notes also cover a C++17 modernization: `boost::filesystem`, `boost::any` and `boost::variant` were replaced by `std::` types. "Embedders should note that most public headers now traffic in `std::optional`, `std::any`, and `std::variant` … downstream code that named the Boost types explicitly will need a one-line typedef or search-and-replace." The default branch was renamed master→main. — [NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md)
- Build dependencies: CMake ≥3.16.2, Boost ≥1.72, GMP ≥6.1.2, MPFR ≥4.0.2, utfcpp. Optional: ICU, gettext, libedit, Python ≥3.10, Gpgmepp. — [README.md](https://github.com/ledger/ledger/blob/main/README.md)
- Debian ships ledger 3.3.0 in bookworm, 3.3.2 in trixie and 3.4.1 in sid. — [packages.debian.org](https://packages.debian.org/search?keywords=ledger&searchon=names&suite=all&section=all)

**Core features (manual, ledger3.texi on main)**
- The manual has sections for: Virtual postings, Expression amounts, Balance assertions, Balance assignments, Total and virtual posting costs, Commodity prices, Fixated prices and costs, Commodity swaps, Lot dates, Lot notes, Automatic Lot Matching, Lot value expressions, Automated Transactions (amount multipliers, access to the matched posting's amount and account, metadata), Effective Dates, Periodic Transactions, Named Automated Transactions, Asset Allocation, and Visualizing with Gnuplot. — [ledger3.texi](https://github.com/ledger/ledger/blob/main/doc/ledger3.texi) / [ledger manual](https://ledger-cli.org/doc/ledger3.html)
- Built-in report commands in `src/report.cc`: accounts, balance/bal, budget, csv, cleared, convert (CSV import), commodities, draft/entry/xact, equity, emacs/lisp, echo, print, prices, pricedb, pricemap, payees, register/reg, reload, stats, **select** (SQL-like), tags, **xml**. There is **no JSON output command**. In `src/select.cc`, the `style` clause accepts csv/xml/json/emacs, but every branch is an empty placeholder. — [src/report.cc](https://github.com/ledger/ledger/blob/main/src/report.cc), [src/select.cc](https://github.com/ledger/ledger/blob/main/src/select.cc)
- Automatic FIFO/LIFO lot matching, where selling without a `{cost}` annotation pairs the sale with open lots and computes gain/loss, is **new in the unreleased 3.5.0**. It closes the very old issue #164 and adds `--lot-matching fifo|lifo`, `--lots-fifo`/`--lots-lifo`, `--plopen` (unrealized P&L on open positions) and `--gain-since`. In released 3.4.x, the user must annotate which lot is being sold. — [NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md), [ledger3.texi "Automatic Lot Matching"](https://github.com/ledger/ledger/blob/main/doc/ledger3.texi)
- The draft 3.5.0 notes list many correctness fixes in valuation and lots. This shows how fragile these areas were in released versions: `-V -M` dropped revaluations (#2131), `-H` with `-X` (#1738), `--exchange` for quoted commodities (#2321), silently accepted imbalanced transactions with lot-price annotations, unrealized gains with several prices on the same day (#1821), and more. — [NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md)
- The draft 3.5.0 notes also cover a forecast overhaul closing #591, #1044, #1113, #1141, #1148, #1155, #1161, #1576, #1605 and crash #2043. They add `payee-rewrite`/`account-rewrite` directives, directives to enable/disable automated transactions, and `$expr` in automated-transaction account names. Budget periods now default to monthly. — [NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md)
- Balance-assertion semantics change in 3.5.0: assertions are filtered "by date rather than file order", with `--check-in-file-order` to restore the pre-3.5 behavior (#3186). — [NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md)

**Syntax quirks / compatibility**
- Ledger supports features hledger does not: value expressions (an embedded expression language), parenthesized amount expressions like `($10 / 3)`, embedded `python` blocks, and directives such as `apply fixed`, `assert`, `check`, `define`, `eval/expr`, `value`, `capture`, `bucket`, `tag`. hledger accepts but ignores most of these. — [hledger manual "Other Ledger directives"](https://hledger.org/hledger.html), [hledger and Ledger](https://hledger.org/ledger.html)
- Ledger's lot syntax is `{LOTUNITCOST}`/`{{LOTTOTALCOST}}`, `[LOTDATE]`, `(LOTNOTE)`, plus fixated `{=COST}`. Ledger also has "virtual costs" `(@)`/`(@@)`, which do not generate market prices. — [hledger manual "Ledger cost basis"](https://hledger.org/hledger.html), [ledger manual "Fixing Lot Prices"](https://www.ledger-cli.org/3.0/doc/ledger3.html#Fixing-Lot-Prices)
- The 3.5.0 draft removes the old 4096-byte parser line-length limit. It also documents that `capture` was never implemented and removes its documentation. — [NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md)
- The PTA FAQ (2022 assessment) describes Ledger as "the oldest and best known, with many features and long-standing quirks". — [plaintextaccounting.org FAQ](https://plaintextaccounting.org/FAQ)

**Performance**
- 3.5.0 draft: register sped up 46% "on large journals (≈50K postings: 6.0s → 3.3s on the benchmark)". The accounts/payees/tags/commodities listings were moved to a fast path; before that they were "6-10× slower than `stats`" on a 20K-posting journal. The benchmark harness in `benchmark/` uses hyperfine and a generated journal. — [NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md), [benchmark/](https://github.com/ledger/ledger/tree/main/benchmark)
- The traditional claim is that Ledger "does core calculations 10x faster on big data". hledger's maintainer wrote (issue opened 2025-04-02, still open): "for a number of years I have been unable to repeat these results", though "recently others have". — [hledger #2363](https://github.com/simonmichael/hledger/issues/2363)
- hledger's own comparison page still lists "more speed with large files" as a Ledger advantage. — [hledger and Ledger](https://hledger.org/ledger.html)

### Inferences
- Ledger's maintenance is **bursty and depends on one person**. After a near-dormant 2022–mid-2025, it had a single maintainer's AI-assisted sprint in 2026. Quality has clearly gone up (fuzzing, sanitizers, 89.6% coverage, a Lean semantics oracle), but those gains are not yet in a release. Anyone depending on packaged Ledger (Debian, Homebrew) still gets 3.3.x/3.4.x behavior with the known lot and valuation bugs.
- Because a large share of 2026 fixes touch valuation, lots and forecasting, released Ledger results in those areas should be treated as historically unreliable. Any tool meant to match Ledger's semantics has a moving target.
- The C++ public headers switched from Boost to std types. The C++ embedding API is therefore not stable across 3.4→3.5.

### Gaps
- No official 3.5.0 release date was found. NEWS.md calls it "upcoming", but nothing gives a date.
- No independent, recent (2025–2026), same-machine benchmark comparing Ledger 3.4/main against hledger 1.52/2.0 was found. The hledger issue #2363 thread's numbers could not be retrieved.
- The historical count of open Ledger issues before the 2026 cleanup could not be verified. The "15 open issues" figure comes from a single page fetch.

## 2. Ledger's Python bindings, and other ways to drive Ledger from Python

### Takeaway
Ledger's Python support is a **Boost.Python extension compiled into the C++ build**: 14 `src/py_*.cc` binding files, off by default, enabled with `./acprep --python`. It is **not on PyPI**; the `ledger` name there belongs to an unrelated package. You get it by building from source or from distro packages (Debian `python3-ledger`). It has a long history of crashes (segfaults, use-after-free), most of which are fixed only in unreleased 3.5.0. The API allows only one active query per journal, and the docs still show Python 2 syntax. In practice, most people script Ledger through subprocess with `csv`/`xml` output or custom `--format` strings. Ledger has no JSON output.

### Cited Findings
- The README says: "Ledger includes optional Python bindings built with Boost.Python. Python support is off by default and must be explicitly enabled." It requires Python 3.10+. — [github.com/ledger/ledger README](https://github.com/ledger/ledger)
- Binding sources in the tree: `py_account, py_amount, py_balance, py_commodity, py_expr, py_format, py_item, py_journal, py_post, py_session, py_times, py_utils, py_value, py_xact` (14 files). Python tests live in `test/python/`. There is **no `pyproject.toml` or `setup.py`** in the repo. — [ledger/src](https://github.com/ledger/ledger/tree/main/src), [ledger/python](https://github.com/ledger/ledger/tree/main/python)
- PyPI: the `ledger` project is an unrelated "Geektrust ledger problem" package (1.0.1, 2022-07-11). `ledger-py`/`ledgerpy` are for Ledger Nano S hardware wallets. `pyledger` is "A simple ledger for smart contracts" (0.5, 2017-05-10), so **the name `pyledger` is also taken on PyPI**. — [pypi.org/project/ledger](https://pypi.org/project/ledger/), [pypi.org/project/pyledger](https://pypi.org/project/pyledger/)
- Debian ships a binary `python3-ledger` package matching the ledger version: bookworm 3.3.0, trixie 3.3.2, sid 3.4.1. — [packages.debian.org](https://packages.debian.org/search?keywords=ledger&searchon=names&suite=all&section=all)
- Build FAQ: "Whenever I try to use the Python support, I get a segfault … Make sure that the boost_python library you linked against is using the exact same Python as the Ledger executable." Another FAQ entry reads "When I run `make check`, the Python unit tests always crash". — [INSTALL.md](https://github.com/ledger/ledger/blob/main/INSTALL.md)
- Forum post (igbanam, 2025-02-10): "The documentation says this is possible, but for the life of me, I found it hard to get `import ledger` to work." The workaround was building with `./acprep --python` and copying `ledger.so` into the project. The poster asked for "this as a package in PyPi". The reply (2025-02-13) suggested exporting CSV with hledger instead. — [PTA forum: Extending ledger-cli with Python](https://forum.plaintextaccounting.org/t/extending-ledger-cli-with-python/493)
- Older mailing-list threads report the same class of problems: a Python/Boost.Python version mismatch (e.g., Python 3.9 vs Boost bindings for 3.8), acprep failing to find Boost, and Homebrew needing a separate `boost-python3`. — [ledger-cli Google Group: Getting Python bindings working](https://groups.google.com/g/ledger-cli/c/bcfZ7Rf68bY), [building python package](https://groups.google.com/g/ledger-cli/c/spA9Ns42YSY/m/j0jghLhvAgAJ)
- Unreleased 3.5.0 fixes "many crashes in the Python bindings and embedded interpreter":
  - segfaults in `journal.query()` with `--pivot`, `--account` or `-M` (#1193, #1858)
  - use-after-free in Python journal queries (#2152)
  - segfaults in `close_journal_files()` when Python holds commodity references (#977, #978)
  - a dangling session pointer when ledger is imported as a module (#2163)
  - a stack-smash in the optional converter

  It also allows ledger "to be loaded as a Python extension module (#513)", makes `xacts`/`posts` iterable (#682), fixes `xact.posts` (#2453), fixes the `TypeError` on `boost::optional` fields like `note`, and exposes `price_point_t`. — [NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md)
- API model (manual): a `Session` holds commodities, and `ledger.read_journal(path)` returns a Journal. Iterating `journal.xacts` gives "raw" data. `journal.query("…")` accepts "every argument you can specify on the command-line" and returns "cooked" postings, with automated transactions applied. "Since a query 'cooks' the journal it applies to, only one query may be active for that journal at a given time." Amount arithmetic is restricted to a single commodity; multi-commodity sums need `Balance()`. — [ledger3.texi "Extending with Python"](https://github.com/ledger/ledger/blob/main/doc/ledger3.texi)
- The manual's Python examples on `main` (Sept 2026) still use **Python 2 `print` statements**, e.g. `print "Transferring %s to/from %s" % (…)`. — [ledger3.texi](https://github.com/ledger/ledger/blob/main/doc/ledger3.texi)
- Embedded Python: a `python` directive in the journal defines functions that "become immediately available as valexpr functions", for example for `tag PATH` / `assert check_path(value)`. The README says some features, such as `--import`, "require building Ledger with Python support". — [ledger3.texi](https://github.com/ledger/ledger/blob/main/doc/ledger3.texi), [README.md](https://github.com/ledger/ledger/blob/main/README.md)
- Non-binding routes: Ledger's report commands include `csv`, `xml`, `emacs`/`lisp` (s-expressions), `print`, and user-defined `--format` strings. There is no JSON. — [src/report.cc](https://github.com/ledger/ledger/blob/main/src/report.cc)
- Third-party Python around Ledger:
  - `ledgerhelpers` "extends Ledger's python library". — [plaintextaccounting.org](https://plaintextaccounting.org/)
  - `ledger-autosync` 1.2.0 (2024-08-22), "Automatically sync your bank's data with ledger". — [PyPI ledger-autosync](https://pypi.org/project/ledger-autosync/)
  - `ledgertools` is abandoned (0.3, 2013). — [PyPI ledgertools](https://pypi.org/project/ledgertools/)
  - `pricehist` 1.4.16 (2026-05-15) fetches historical prices and outputs Ledger/hledger/Beancount price formats. — [PyPI pricehist](https://pypi.org/project/pricehist/)

### Inferences
- For a Python developer, Ledger-as-a-library means: compile C++ against a matching Boost.Python/Python ABI; no wheels; a mutable, stateful Session/Journal model with a single-active-query constraint; and, in every released version, known segfault paths. That is far from what `pip install`-and-go users expect.
- Subprocess plus `xml`/`csv` is the practical integration path. It has costs: the whole journal is re-parsed on every call, output formats are shaped around reports rather than a canonical data model, and there is no JSON.
- The absence of any maintained PyPI wheel for Ledger, in 2026, is itself a gap that a new Python-first core would fill.

### Gaps
- No quantitative user survey on how many people use Ledger's Python bindings was found.
- It was not verified whether Homebrew's current `ledger` formula builds with Python enabled. NEWS mentions "fix the Homebrew build with Python support (#2267)" but not the formula's default.
- There was no reliable evidence on whether `ledgerhelpers` works with Ledger 3.4+.

## 3. hledger: current version, release cadence, features, Ledger compatibility, performance

### Takeaway
hledger is on a **steady quarterly cadence** with frequent point releases. The stable line is **1.52.x (1.52 on 2026-03-20; 1.52.4 on 2026-09-10)**. hledger 2.0 is in preview as **1.99.1–1.99.4 (2026-03-28 → 2026-09-10)**, with "the final 2.0 release (coming later this year)". The previews add native **lot tracking** (FIFO/LIFO/HIFO/AVERAGE), capital-gains calculation and a **holdings** report with XIRR. hledger has the broadest set of built-in financial reports, the best CSV import system, JSON output on major reports, and CLI/TUI/web/JSON-API front ends. It does not support Ledger's value expressions or embedded scripting. Performance regressed from 1.25 to 1.99.4; on current `main` the gap is closed (~52k txns/s on the maintainer's machine).

### Cited Findings
**Versions and cadence**
- Major releases: 1.40 (2024-09-09), 1.41 (2024-12-09), 1.42 (2025-03-07), 1.43 (2025-06-01), 1.50 (2025-09-03), 1.51 (2025-12-05), 1.52 (2026-03-20).
- Point releases: 1.52.1 (2026-04-28), 1.52.2 (2026-08-24), 1.52.3 (2026-08-27), 1.52.4 (2026-09-10).
- 2.0 previews: 1.99.1 (2026-03-28), 1.99.2 (2026-04-28), 1.99.3 (2026-06-24), 1.99.4 (2026-09-10).

  — [hledger release notes](https://hledger.org/relnotes.html), [hledger tags](https://github.com/hledgerorg/hledger/tags)
- The release notes say: "Both hledger 1 and 2 (the 1.99.x preview releases) are suitable for daily use. hledger 1 is receiving only essential fixes; the hledger 2 preview releases are strictly better, highly compatible … the final 2.0 release (coming later this year)". — [relnotes.md](https://github.com/hledgerorg/hledger/blob/main/doc/relnotes.md)
- Activity: 2,726 commits since 2025-01-01, 2,499 of them by Simon Michael. Next contributors: Arthur Cinader 59, Dmitry Astapov 47, Henning Thielemann 23. Latest commit 2026-09-24. The repo moved to `github.com/hledgerorg/hledger`. — [hledger commits](https://github.com/hledgerorg/hledger/commits/main)
- The hledger comparison page claims "a new release every quarter". — [hledger and Ledger](https://hledger.org/ledger.html)
- "Thoughts on hledger 2" (#2547, opened 2026-02-01): lot tracking (#1015) "necessitates impactful data model changes". The 2.x line allows breaking changes with "a reasonably smooth path to migrate data from 1 to 2". — [hledger #2547](https://github.com/hledgerorg/hledger/issues/2547)
- AI policy (updated 2026-09-21):
  - "hledger 2.x (2026..) is developed with careful AI assistance".
  - hledger 1.x "was developed without AI assistance".
  - The maintainer says he needed AI "to fully design and implement robust tax lot tracking … I think it's unlikely hledger would have ever got this feature without them".
  - "hledger does not use AI at runtime".
  - A 6-month OSS AI credit was used in 2026.

  — [hledger AI policy](https://hledger.org/AI.html), [relnotes.md](https://github.com/hledgerorg/hledger/blob/main/doc/relnotes.md)
- Debian: bookworm ships 1.25, trixie 1.32.3, sid 1.52.1. — [packages.debian.org](https://packages.debian.org/search?keywords=ledger&searchon=names&suite=all&section=all)

**Features (manual and release notes)**
- Strict mode: by default hledger checks syntax, transaction balancing and balance assertions. `-s/--strict` adds: accounts declared, commodities declared, and "all commodity conversions declared explicitly". The `check` command runs individual checks. — [hledger manual: Strict mode](https://hledger.org/hledger.html)
- CSV import: a rules-file system with `if` matchers. `hledger import *.csv` "will (a) detect the new transactions, and (b) append just those transactions … It is idempotent" (state kept in `.latest.FILE.csv`). This works "Where records have a stable chronological order, and new records appear only at the new end". — [hledger manual: Deduplicating, importing](https://hledger.org/hledger.html)
- Other headline features by release:
  - 1.40: config file, sortable register, FODS output.
  - 1.41: robust export to Beancount, terminal pagination.
  - 1.42: `run` and `repl` commands, non-UTF8 CSV.
  - 1.43: `setup` command, better boolean queries, timeclock concurrent sessions.
  - 1.50: better transaction balancing, auto-posting account interpolation, CSV "data commands", import archiving.
  - 1.52: faster valuation, commodity tags, preserved/exportable cost basis annotations.
  - 1.99.3: conventional file layout, `get` command, commodity aliases, average-cost methods, `prices --summary`.
  - 1.99.4: holdings command, command aliases, a CSV rules precedence fix, `any:`/`all:` queries in posting reports.

  — [hledger release notes](https://hledger.org/relnotes.html)
- Reports and formats: `balancesheet`, `balancesheetequity`, `cashflow`, `incomestatement`, `balance` (including `--budget`), `register`, `aregister`, `holdings` and `print` support txt/html/csv/tsv/fods/json. `print` also supports `ledger`, `beancount` and `sql`. — [hledger manual: Output format](https://hledger.org/hledger.html)
- The manual also documents: forecasting (`--forecast` with periodic transactions), budget reports, valuation (`-B/-V/-X/--value`, with valuation date and commodity), tags, pivoting (`--pivot`), Timeclock and Timedot formats, auto postings (including "on forecast transactions only"), and `roi`. — [hledger manual](https://hledger.org/hledger.html)
- Lots (2.0 previews only):
  - Lot detection from `{COSTBASIS}`, `:{LOTNAME}` subaccounts, or `lots:` tags.
  - Booking methods FIFO/LIFO/HIFO/AVERAGE per account, and FIFOALL/LIFOALL/HIFOALL/AVERAGEALL global.
  - Automatic capital-gain calculation or checking; `--lots` mode; `check lots`; `close --clopen --lots` across year files.
  - A Beancount-like `{DATE, "LABEL", COST}` syntax, with Ledger `{COST} [DATE] (NOTE)` also accepted.
  - Breaking change: postings to Gain-type accounts are excluded from normal balancing in disposals.

  — [relnotes 1.99.1](https://hledger.org/relnotes.html), [SPEC-lots](https://hledger.org/SPEC-lots.html)
- 1.99.4 lot rework: "Several bugs that could silently produce wrong numbers are fixed". The new `holdings` report shows units, cost basis, price, market value, weight, unrealized and realized gains, and XIRR. A known limitation: "Balance assertions on lot subaccounts can't be checked correctly (assertions are checked before lots are calculated)". — [relnotes 1.99.4](https://hledger.org/relnotes.html)
- The lot design was still changing in Sept 2026. Issue #2731 (2026-09-14): "Lot tracking forces breaking the accounting equation / mis-valuing assets". The reporter argues "lot postings should participate in balancing at basis cost, not at transacted cost". The follow-up PR #2744 is titled "lots: balance disposals at cost basis (historical cost accounting)". The G (Gain) account type is no longer auto-detected from names, and a U (UnrealisedGain) type was added. — [hledger #2731](https://github.com/hledgerorg/hledger/issues/2731), [PR #2744](https://github.com/hledgerorg/hledger/pull/2744), [relnotes](https://hledger.org/relnotes.html)
- Front ends: CLI; hledger-ui (TUI); hledger-web (web UI plus JSON API). The comparison page lists "multiple officially-supported user interfaces: CLI, TUI, web, HTTP-JSON", "timedot time logging format", "a Haskell API". — [hledger and Ledger](https://hledger.org/ledger.html)

**Ledger compatibility**
- hledger does not support Ledger's value expressions. It rejects parenthesized amount expressions `($10 / 3)`. It ignores `((valuation))` expressions after amounts. It accepts but ignores `apply fixed`, `apply tag`, `assert`, `bucket`, `capture`, `check`, `define`, `eval/expr`, `python`, `tag`, `value` and `--command-line-flags`. It warns that "hledger's reports may differ from Ledger's if you use these". Ledger's virtual costs `(@)`/`(@@)` are treated as `@`/`@@`. — [hledger manual: Other Ledger directives / Ledger virtual costs](https://hledger.org/hledger.html)
- 1.99.4: "a single tab is also now accepted as the separator between account and amount, for improved compatibility with Ledger". — [relnotes 1.99.4](https://hledger.org/relnotes.html)
- Stated Ledger advantages: "more speed with large files", "support for embedded code in journals (value expressions, python expressions..)", "a C++ API". — [hledger and Ledger](https://hledger.org/ledger.html)

**Performance (maintainer measurements, 2026-09-23)**
- Setup: a synthetic journal of 100k transactions, 200k postings, 100k P directives, 1k accounts and 26 commodities (8 MB, 500k lines), measured on a MacBook Pro M5 Pro with GHC 9.14.1.
- `balance` wall time: 1.25 2.68s, 1.40 3.92s, 1.52 4.06s, 1.99.4 5.80s, **main 2.15s**.
- `register`: 1.25 71.99s → main 14.17s ("rendering a 26-commodity running balance for 200k lines").
- Throughput: **main 52k txns/s**, vs 37k (1.25), 25k (1.40), 23k (1.52), 17k (1.99.4).
- Memory: "~2.6 KB per transaction in memory (262 MB for 100k)", ~0.8 GB RSS. Parsing takes 55% of the run, and GC 31%.

  — [NOTE-performance.md](https://github.com/hledgerorg/hledger/blob/main/doc/NOTE-performance.md), [hledger and Ledger](https://hledger.org/ledger.html)
- Earlier regression issues: #2122 (1.26 about 50% slower than 1.25) and #2153 (1.29–1.32.2 slow with many accounts). — [hledger #2363](https://github.com/simonmichael/hledger/issues/2363), [hledger #2153](https://github.com/simonmichael/hledger/issues/2153)

### Inferences
- hledger is the healthiest of the two projects by release discipline: quarterly releases, a changelog for every point release, and an explicit 1.x/2.x migration story. It still depends on one person, with ~92% of commits from one maintainer.
- Investment and lot support, historically hledger's weakest area, is being built right now. It is only in preview, and its semantics were still being redesigned in Sept 2026. Real-world maturity of hledger lots (tax reporting, wash sales, corporate actions) cannot yet be judged.
- Whole-journal re-parse on every invocation, at ~2.6 KB RAM per transaction and ~2s per 100k transactions, is fine for personal finance but sets a floor for any interactive or embedded use through the CLI.

### Gaps
- No exact date for hledger 2.0 final beyond "later this year" (2026).
- No independent third-party benchmark of hledger 1.99.x/main vs Ledger was found.
- No data on hledger user counts or adoption trends was found.

## 4. Using hledger from Python

### Takeaway
There are **no Python bindings to hledger-lib**, which is Haskell-only. Python integration means one of: (a) **subprocess with `-O json` or `-O csv`**; (b) the **hledger-web JSON API** (read-mostly, with a PUT `/add`); or (c) third-party pure-Python re-implementations and add-ons, all small or stale. hledger's JSON is explicitly a verbose dump of internal Haskell types, and its field names change between versions (e.g., 1.99.1 renamed `ptype`→`preal`). It is usable but not a stable, designed library API.

### Cited Findings
- JSON output: "Our JSON is rather large and verbose, since it is a faithful representation of hledger's internal data types. To understand its structure, read the Haskell type definitions" (Hledger/Data/Types.hs). "In JSON output we round numbers to at most 10 decimal places" (see issue #1195). — [hledger manual: JSON output](https://hledger.org/hledger.html)
- JSON is available for aregister, balance, balancesheet, balancesheetequity, cashflow, holdings, incomestatement, print and register. It is **not** available for accounts, prices, stats, commodities, etc. — [hledger manual: Output format](https://hledger.org/hledger.html)
- JSON schema instability example (1.99.1): "Posting's `ptype` field has been renamed to `preal` … This changes JSON output." — [relnotes 1.99.1](https://hledger.org/relnotes.html)
- hledger-web JSON API (`--serve-api`): routes `/version`, `/accountnames`, `/transactions`, `/prices`, `/commodities`, `/accounts`, `/accounttransactions/ACCOUNTNAME`, `/openapi.json`. Writes go through a PUT to `/add`, and "The payload must be the full, exact JSON representation of a hledger transaction (partial data won't do)". The docs describe the OpenAPI spec as "basic". — [hledger-web manual: JSON API](https://hledger.org/hledger-web.html)
- hledger's scripting page covers shell, make/just, add-on executables (`hledger-NAME` in any language) and Haskell scripts using hledger-lib. It says nothing about Python bindings. — [Scripting hledger](https://hledger.org/scripting.html)
- The hledger repo itself ships Python glue scripts: `paypalcsv`, `simplefincsv`, `krakencsv`. — [Scripts and add-ons](https://hledger.org/scripts.html)
- Python add-ons and wrappers, with PyPI status on 2026-09-25:
  - `hledger-utils` 1.14.0, last upload **2023-11-17**. Provides `hledger-edit` and `hledger-plot` (matplotlib). — [PyPI hledger-utils](https://pypi.org/project/hledger-utils/)
  - `hledger-lots` 0.4.2, last upload **2023-05-21**. A FIFO lots report and sale-transaction generator, now superseded by native 2.0 lots. — [PyPI hledger-lots](https://pypi.org/project/hledger-lots/), [github edkedk99/hledger-lots](https://github.com/edkedk99/hledger-lots)
  - `hledger-textual` 0.3.7 (2026-06-17), first release 2026-02-27. A Textual TUI. — [PyPI hledger-textual](https://pypi.org/project/hledger-textual/)
  - Also: `hreports` (query shortcuts and PDF reports), `hledger-budget` (envelope budgeting wrapper), `hledger-tui`. — [PyPI hreports](https://pypi.org/project/hreports/), [PyPI hledger-budget](https://pypi.org/project/hledger-budget/1.3.0), [PyPI hledger-tui](https://pypi.org/project/hledger-tui/)
  - `ledgerkit`: "a deterministic, Python-native accounting and query engine". It re-implements a subset of hledger 1.52 journal format in pure Python: include/account/commodity/payee/alias/P/Y/D/apply account, plus balance/register/print/accounts/stats/check and pandas export. Around 4 stars and 68 commits. The README does not mention lots, auto postings or periodic transactions. — [github ctosullivan/ledgerkit](https://github.com/ctosullivan/ledgerkit)
  - `pyhledger`: "Python Library to read, manipulate and generate stats of hledger files". — [github btittelbach/pyhledger](https://github.com/btittelbach/pyhledger)
  - `hledger-sankey`: a plotly script. — [github adept/hledger-sankey](https://github.com/adept/hledger-sankey)
- No PyPI package named `hledger`, `hledger-py` or `pyhledger` exists. — [PyPI JSON API lookups](https://pypi.org/pypi/hledger/json)
- Forum thread (Aug 2024): a user asked, "I do not know how to execute hledger commands and get the output of the commands in Python". The maintainer (simonmic) pointed to hledger-plot, "It's python and uses matplotlib". — [PTA forum: Advice for python scripts with hledger](https://forum.plaintextaccounting.org/t/advice-for-python-scripts-with-hledger/337)

### Inferences
- A Python user of hledger has to pick between: subprocess+JSON, with a full re-parse on every call and a schema tied to Haskell internals that shifts across versions; a long-running hledger-web process, with limited routes, full-object writes and no report endpoints for balance sheets or budgets; or immature pure-Python re-implementations that trail hledger's semantics.
- The Python ecosystem around hledger is fragmented. Several add-ons (hledger-utils, hledger-lots) have had no PyPI release since 2023. This suggests demand exists but no core library has emerged to anchor it.

### Gaps
- No published measurement of subprocess+JSON overhead for typical Python workflows was found.
- It was not checked whether hledger's JSON output is covered by any compatibility or stability promise. None was found in the manual.

## 5. Main pain points, limitations and feature requests reported by users

### Takeaway
Common complaints are:
- the learning curve and forgiving-but-quirky syntax (Ledger especially);
- investments, lots and capital gains, which historically were weak in both tools and are only now being addressed in unreleased or preview code;
- categorization and import friction;
- scripting and extensibility: Ledger's Python bindings are hard to build and crash-prone; hledger has no embedded scripting and only a verbose JSON dump;
- performance on large journals, which is disputed and version-dependent;
- single-maintainer risk.

### Cited Findings
- PTA FAQ (2022): "Ledger and hledger have flexible file formats and parse forgivingly by default, with optional strictness. Beancount has a more restricted file format and always parses strictly." Beancount "has the most features for investing and trading, and the most support (and need) for customisation, via Python". — [plaintextaccounting.org FAQ](https://plaintextaccounting.org/FAQ)
- Investment and lot tracking:
  - Ledger's automatic lot matching request (#164) was open until the 2026 work, and is still unreleased.
  - hledger's lot tracking (#1015) is described by the maintainer as a feature "I have been wanting for many years, but it was just too big/intricate to tackle".

  — [ledger NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md), [hledger AI policy](https://hledger.org/AI.html)
- Correctness bugs in valuation and lots in released Ledger. The 3.5.0 draft fixes silently accepted imbalanced lot-price transactions, `-V -M` dropping revaluations (#2131), wrong unrealized gains with same-day prices (#1821), and more. — [ledger NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md)
- In hledger 2.0 previews, 1.99.4 fixed lot bugs "that could silently produce wrong numbers". A user reported in Sept 2026 that lot tracking broke the accounting equation (a $200 assets/equity discrepancy), which led to a redesign toward balancing at cost basis. — [relnotes 1.99.4](https://hledger.org/relnotes.html), [hledger #2731](https://github.com/hledgerorg/hledger/issues/2731)
- Import and categorization friction: a Nov 2024 HN comment described as the "major gripe" defining regexes/categories for dozens to hundreds of expense descriptions and "recompiling" hledger journals after every rule change. (This is from a search snippet; the page itself returned HTTP 429.) — [HN item 42127432](https://news.ycombinator.com/item?id=42127432)
- CSV rules pitfalls fixed only in 1.99.4:
  - When a directive was declared more than once, the first declaration had silently won, contrary to the manual.
  - `include` errors were reported at the wrong file and line.
  - A failing data command aborted the whole import.
  - `import` with `archive` could "stall, reprocessing the oldest file each time".

  — [relnotes 1.99.4](https://hledger.org/relnotes.html)
- hledger's import dedup assumes "a stable chronological order, and new records appear only at the new end". Other workflows need external tools. — [hledger manual](https://hledger.org/hledger.html)
- Extensibility:
  - Ledger's Python route: build difficulty in 2025, the segfault FAQ, the crash list in 3.5.0, and a request for a PyPI package. — [PTA forum 2025](https://forum.plaintextaccounting.org/t/extending-ledger-cli-with-python/493), [INSTALL.md](https://github.com/ledger/ledger/blob/main/INSTALL.md)
  - hledger lacks "embedded code in journals (value expressions, python expressions..)". — [hledger and Ledger](https://hledger.org/ledger.html)
  - Its JSON is an internal-type dump. — [hledger manual](https://hledger.org/hledger.html)
- Performance disputes: "Ledger does core calculations 10x faster on big data" vs hledger's maintainer being "unable to repeat these results" (2025). hledger itself regressed from 1.25 to 1.99.4 (balance 2.68s → 5.80s on 100k txns) before the Sept 2026 fixes. — [hledger #2363](https://github.com/simonmichael/hledger/issues/2363), [NOTE-performance.md](https://github.com/hledgerorg/hledger/blob/main/doc/NOTE-performance.md)
- A search snippet attributed to a user reports, for 17,385 transactions, hledger balance at 4.07s vs Ledger at 381 ms, and hledger print at 3.45s vs Ledger under 1s. The original page could not be located, so this is **unverified**. — search snippet only (no primary URL found)
- Numerical representation: hledger stores "up to 255 significant digits" (Decimal) but JSON output rounds to 10 decimal places (#1195). — [hledger manual: JSON output](https://hledger.org/hledger.html)
- Balance-assertion ordering semantics differ and have changed. Ledger 3.5 moves from file order to date order, with `--check-in-file-order`. hledger lot subaccount assertions "can't be checked correctly". — [ledger NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md), [relnotes 1.99.4](https://hledger.org/relnotes.html)
- Governance and AI controversy risk:
  - hledger's release notes: "If you disagree with it, please be patient while we navigate this period". The AI-free 1.x "can be revived or forked at any time".
  - Ledger's 2026 work is heavily AI-co-authored (969 trailers).

  — [hledger relnotes](https://hledger.org/relnotes.html), [hledger AI policy](https://hledger.org/AI.html), [ledger commits](https://github.com/ledger/ledger/commits/main)
- Beancount-vendor sources (beancount.io, a commercial Beancount host, so likely biased) cite "The Wall of Intimidation", a steep learning curve, tedious data entry, and "one misplaced comma can send you on a frustrating debugging quest". They also claim LLMs are now used for categorization and CSV conversion. — [beancount.io: LLM-assisted PTA feedback (2025-07-28)](https://beancount.io/blog/2025/07/28/user-experience-and-feedback-on-llm-assisted-plain-text-accounting)

### Inferences
- The recurring unmet needs are:
  1. trustworthy lot and capital-gains accounting;
  2. an embeddable, stable programmatic API, which neither tool has for Python;
  3. low-friction import and categorization;
  4. clear, strict validation with good errors.

  Both incumbents are addressing (1) in 2026, but only in unreleased or preview code, so the space is still in flux.
- Silent-wrong-number bugs keep appearing in lots and valuation in both tools. That points to the value of an explicit, test-heavy or formally specified semantics for those subsystems. Ledger is now pursuing this via Lean.

### Gaps
- Direct Reddit r/plaintextaccounting threads could not be retrieved. Search results did not surface specific threads, so there is no Reddit-sourced quote here.
- There is no quantitative data on how prevalent each pain point is (e.g., a user survey).
- The Nov 2024 HN thread content could not be fetched directly (HTTP 429).

## 6. Documented comparisons: Ledger vs hledger vs Beancount

### Takeaway
The main primary comparisons are hledger's own "hledger and Ledger" and "hledger and Beancount" pages, the plaintextaccounting.org FAQ and feature matrix, and Beancount's 2014 comparison doc. The consensus: Ledger is the most powerful and quirky, with embedded expressions and historically the fastest. hledger is the most user-friendly, the most actively maintained, has the best reports and CSV import, and has multiple UIs. Beancount is the strictest, the strongest for investments, and scriptable in Python. Many 2025–2026 "showdown" articles come from beancount.io, a commercial Beancount vendor, and should be treated as biased.

### Cited Findings
- hledger's view:
  - hledger "focusses strongly on UX, reliability, and real-world practicality".
  - Advantages over Ledger: "more active maintenance", "complete and accurate manual", "more built in reports, including standard financial reports", "multi-period reports", "easier query syntax", "battle-tested CSV/SSV/TSV import system", "CLI, TUI, web, HTTP-JSON" interfaces.
  - Ledger's advantages: "more speed with large files", embedded code, "a C++ API".
  - Performance: main at "52k" txns/s on 100k transactions, about 3x faster than 1.99.4 and 2x faster than 1.52.

  — [hledger and Ledger](https://hledger.org/ledger.html)
- PTA FAQ (2022): Ledger is "the oldest and best known, with many features and long-standing quirks". hledger is "a cleaned-up version of Ledger, the most actively maintained, and the most user-friendly". Beancount "has the most features for investing and trading". Migration is "relatively easy … eg using … ledger2beancount and beancount2ledger". — [plaintextaccounting.org FAQ](https://plaintextaccounting.org/FAQ)
- plaintextaccounting.org lists comparison resources: hledger and Ledger / hledger and Beancount / hledger and other software (2014–2023), Matthias Kauer's "Command Line Accounting – A look at the various ledger ports" (2015), Beancount's "A Comparison of Beancount and Ledger" (2014), and a "PTA apps feature matrix". — [plaintextaccounting.org](https://plaintextaccounting.org/)
- 2014 mailing-list thread: Martin Blais (Beancount author) said Beancount differs in how it "treats the booking of lots in inventories … how it treats cost basis", "enforces accounts to be in one of five types", and offers "extensibility (via Python instead of its own language)". He argued "Haskell does not confer any particular advantage" and that "speedy and trustable decimal or rational number representation" matters most. (Older, 2014.) — [ledger-cli Google Group (2014)](https://groups.google.com/g/ledger-cli/c/__yuMVjrOH0)
- Vendor comparisons (beancount.io, so potential bias): "Ledger remains the classic … favored by command-line purists and those who need ultimate speed". "hledger's CSV import system is particularly well-regarded". Beancount's mandatory `open` directives prevent typo-created accounts. — [beancount.io forum: Showdown 2025](https://beancount.io/forum/t/the-ultimate-plain-text-accounting-showdown-2025-beancount-v3-vs-hledger-vs-ledger/81), [beancount.io blog: Beancount vs hledger (2026-03-17)](https://beancount.io/blog/2026/03/17/beancount-vs-hledger-comparison-plain-text-accounting), [beancount.io blog: technical edge (2025-07-22)](https://beancount.io/blog/2025/07/22/beancounts-technical-edge-a-deep-dive-on-performance-python-api-and-data-integrity-vs-ledger-hledger-and-gnucash)
- hledger 1.99.1 improved `print -O beancount` export (booking methods from `lots` tags, cost basis before cost, market prices). 1.41 added "robust export to Beancount". Converting to Beancount is a first-class path. — [hledger relnotes](https://hledger.org/relnotes.html)
- hledger's AI policy lists other free software with lot tracking: "Beancount/Ledger/rustledger/BittyTax/rotki/RP2". So a Rust re-implementation ("rustledger") exists in the landscape. — [hledger AI policy](https://hledger.org/AI.html)

### Inferences
- No neutral, recent, quantitative three-way comparison with same-machine benchmarks was found. Most "2025 showdown" content comes from a Beancount vendor.
- The comparisons agree that "Python extensibility" is Beancount's differentiator, not Ledger's or hledger's. For the Ledger/hledger family, a Python-native core is an unfilled niche.

### Gaps
- The PTA feature-matrix contents were not extracted row-by-row.
- rustledger was not investigated (out of scope; flagged for other researchers).

## 7. Implications: would a new Python-usable PTA core be meaningfully better than Ledger/hledger?

### Takeaway
On the specific axis of **"powerful core usable both from a CLI and as a Python library"**, neither incumbent delivers:
- Ledger's Boost.Python bindings are not pip-installable, historically crash-prone (fixes unreleased), constrained to one active query, and documented in Python 2 syntax.
- hledger has no Python bindings; its JSON is an unstable dump of internal types, and hledger-web exposes only a handful of routes.

A new tool could be meaningfully better here. It would still have to match fast-moving incumbents on lots and gains (both shipping major lot work in 2026), CSV import (hledger's strong suit), reports, and Ledger-syntax compatibility.

### Cited Findings
- Ledger Python: not on PyPI, and distro-only (Debian `python3-ledger`). There is a segfault FAQ; crash fixes (#1193, #1858, #2152, #977, #978, #2163) are only in unreleased 3.5.0; and the constraint "only one query may be active for that journal at a given time" applies. — [INSTALL.md](https://github.com/ledger/ledger/blob/main/INSTALL.md), [NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md), [ledger3.texi](https://github.com/ledger/ledger/blob/main/doc/ledger3.texi), [packages.debian.org](https://packages.debian.org/search?keywords=ledger&searchon=names&suite=all&section=all)
- hledger Python: JSON "faithful representation of hledger's internal data types", with field renames across versions (`ptype`→`preal`). hledger-web writes need the "full, exact JSON representation". The official extension story is Haskell or add-on executables. — [hledger manual](https://hledger.org/hledger.html), [hledger-web manual](https://hledger.org/hledger-web.html), [Scripting hledger](https://hledger.org/scripting.html)
- The existing Python-side ecosystem is small or stale: hledger-utils (2023-11), hledger-lots (2023-05), ledgerkit (~4 stars, subset). — [PyPI hledger-utils](https://pypi.org/project/hledger-utils/), [PyPI hledger-lots](https://pypi.org/project/hledger-lots/), [ledgerkit](https://github.com/ctosullivan/ledgerkit)
- Incumbent performance benchmark: hledger main parses and reports 100k transactions (200k postings) in ~2.2s at ~0.8 GB RSS on an M5 Pro. Ledger register on ≈50K postings takes 3.3s after the 3.5.0 optimization (a different benchmark and machine, so not directly comparable). — [NOTE-performance.md](https://github.com/hledgerorg/hledger/blob/main/doc/NOTE-performance.md), [ledger NEWS.md](https://github.com/ledger/ledger/blob/main/NEWS.md)
- Naming: `pyledger` on PyPI is taken by an unrelated 2017 package ("A simple ledger for smart contracts", 0.5). — [pypi.org/project/pyledger](https://pypi.org/project/pyledger/)

### Inferences
- A differentiated design would offer:
  - a typed, documented, versioned Python object model (Journal/Transaction/Posting/Amount/Lot) with stable serialization;
  - an in-process API that parses once and can run many queries or reports, avoiding Ledger's single-active-query and hledger's re-parse-per-call;
  - pip-installable wheels;
  - exact decimal arithmetic;
  - a Ledger/hledger-compatible journal subset;
  - a CLI built on the same core.
- Performance targets in the order of 50k txns/s parse+report and under 1 KB RAM per transaction would be competitive with hledger main. A pure-Python core may struggle to reach this without a compiled parser (Rust/C extension) or aggressive caching. This is an inference, not measured.
- Lots and capital gains are where users most need correctness, and where both incumbents have shipped silent-wrong-number bugs as recently as 2026. A new core that makes booking methods explicit and heavily tested (FIFO/LIFO/HIFO/AVERAGE, transfers, fees, the accounting-equation invariants raised in hledger #2731) would be a genuine differentiator, but it is also the hardest part to get right.
- Competitive risk: if Ledger 3.5.0 ships with working PyPI-style bindings, or hledger 2.0 stabilizes its JSON, the gap narrows. As of 2026-09-25, neither has announced a PyPI package or a stable JSON contract.

### Gaps
- There is no evidence on whether Ledger's maintainer plans PyPI wheels for 3.5.0. NEWS mentions loading "as a Python extension module (#513)" but no packaging.
- No statement from hledger's maintainer about Python bindings (e.g., via a C FFI to hledger-lib) was found.
