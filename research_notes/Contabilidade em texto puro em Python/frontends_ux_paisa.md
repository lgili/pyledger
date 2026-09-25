# Plain Text Accounting Frontends and UX (Paisa focus) + UX Lessons from Mainstream Personal Finance Apps (state as of Sept 2026)

Research date: 2026-09-25. Dates in brackets are publication/release dates. Items older than 2025 are marked "(older)" where relevant. Some GitHub pages were read through a summarising fetch tool; where two readings of the same fact disagreed, both are noted.

## 1. Paisa: architecture, features, limitations, maintenance status, user complaints

### Takeaway
Paisa (Go backend + Svelte UI, AGPL-3.0, ~3.2k stars) is a reporting, import and editing layer that shells out to the `ledger`, `hledger` or `bean-*` binaries. It parses their CSV or JSON output into a local SQLite database, and it ships an embedded ledger binary. It has strong visual reports (net worth, allocation, gains, goals/retirement, budgets, recurring calendar, a Sheets calculator). But in 2025-2026 it is in maintenance mode: only 4 releases since Oct 2024, and those were mostly build or security fixes. Its Beancount 3 support has been broken since Mar 2025 (bean-report was removed from Beancount 3), multi-currency and commodity bugs are open, and there is no mobile app or bank sync.

### Cited Findings
**Maintenance status and release cadence**
- Release timestamps from GitHub's release feed: v0.6.7, v0.6.8, v0.6.9 and v0.7.0 all on 2024-08-26; v0.7.1 on 2024-10-20; v0.7.2 and v0.7.3 on 2025-02-23; v0.7.4 on 2025-08-03; v0.7.5 on 2026-08-16; v0.7.6 on 2026-09-06 (latest). That is a ~12-month gap between v0.7.4 and v0.7.5. — [Paisa releases.atom](https://github.com/ananthakumaran/paisa/releases.atom); consistent with [Paisa tags page](https://github.com/ananthakumaran/paisa/tags). (Note: one summarised read of the v0.7.6 release page reported "September 6, 2024"; that is inconsistent with version ordering (v0.7.1 = Oct 2024) and with the Atom feed's ISO timestamp 2026-09-06T08:35:17Z, so 2026 is correct.) — [v0.7.6 release](https://github.com/ananthakumaran/paisa/releases/tag/v0.7.6)
- Content of recent releases is maintenance only. v0.7.6 notes: "Include transaction id in the export", update Node.js and Go versions, remove Bun, fix the ledger formula, update fonts. The commits of 2026-08-16 are "bump go version", "bump nodejs", "remove bun", "fix version check", "fix ledger formula" and "update fonts". — [v0.7.6 release](https://github.com/ananthakumaran/paisa/releases/tag/v0.7.6); [commits](https://github.com/ananthakumaran/paisa/commits/master)
- Earlier feature releases (older, 2024): v0.6.3 (Jan 13, 2024) added the Sheets calculator; v0.6.4 (Jan 22, 2024) added checking-account balances on the dashboard; v0.6.5 (Feb 2, 2024) added a credit-card liabilities page, password-protected XLSX and timezone config; v0.6.6 (Feb 10, 2024) added sortable tables and improved allocation/goals pages; v0.7.0 (Aug 26, 2024) added Docker image variants for hledger and beancount; v0.7.1 (Oct 20, 2024) fixed the remote code execution vulnerability; v0.7.3 (Feb 2025) fixed the Yahoo price fetcher. — [Paisa releases](https://github.com/ananthakumaran/paisa/releases)
- Repo stats (Sept 2026): 3.2k stars, 214 forks, 82 open issues, 10 open PRs, AGPL-3.0, ~648 commits. There is a Matrix chat. The README says the author builds it primarily for personal use and welcomes bug reports. — [GitHub repo](https://github.com/ananthakumaran/paisa)

**Architecture: how it reads journals**
- Paisa shells out to the CLIs. The details below are from `internal/ledger/ledger.go`. — [ledger.go](https://github.com/ananthakumaran/paisa/blob/master/internal/ledger/ledger.go)
  - **ledger**: validates with `ledger --args-only [--pedantic] -f <file> balance`. It parses with `ledger csv --csv-format "%(quoted(date)),...%(quoted(xact.note))\n"`, which yields 17 fields: date, payee, account, commodity, quantity, market amount, filename, transaction id, status, line numbers, lot price/commodity, recurring/period tags and note. It reads prices via `pricesdb --pricedb-format "P %(datetime) %(display_account) ..."`.
  - **hledger**: validates with `hledger -f <file> --auto balance [--strict]`. It parses with `hledger -f <file> --auto print -Ojson`. It reads prices with `hledger commodities`, then `hledger --infer-market-prices --infer-costs prices`.
  - **beancount**: validates with `bean-check` and then `bean-report <file> bal`. It parses with `bean-query -f csv <file> select date,payee,narration,account,currency,...` (13 columns) and reads prices with `bean-report <file> pricesdb`.
- The parsed journal is stored in SQLite (`db_path: .../paisa.db`); `journal_path` points to the main journal and `ledger_cli` selects ledger, hledger or beancount. — [Paisa config reference](https://paisa.fyi/reference/config/)
- "Paisa ships with ledger binary. If you use hledger or beancount, make sure that the binaries are installed." — [Paisa ledger-cli reference](https://paisa.fyi/reference/ledger-cli/). "The prebuilt paisa binaries come with an embedded ledger binary and will use it if it's not already installed." — [Installation](https://paisa.fyi/getting-started/installation/)
- Configuration keys include `default_currency` (for example INR), `locale` (en-US and en-IN are listed), `financial_year_starting_month` (default 4, April, as in India), `strict`, `import_templates` and `user_accounts` (sha256-hashed passwords). — [Paisa config reference](https://paisa.fyi/reference/config/)

**Features**
- The homepage lists: built on ledger; price tracking via providers; expense tracking and budgets; data import ("flexible enough to handle most of the formats out there in the wild"); Sheets; goals; recurring transactions ("rent, emi, credit card bills"); tax and capital gains; privacy ("The app will never collect or send any data to any server"); plain-text storage. — [paisa.fyi](https://paisa.fyi/)
- Price providers: MF API (Indian mutual funds), Yahoo Finance, Alpha Vantage, Purified Bytes NPS, Purified Bytes Metals (gold and silver per gram), and manual `P` directives. Prices are fetched "only when you **update** the prices", from the UI or with `paisa update`. If a rate is unavailable, Paisa "will use previous known conversion rate". — [Paisa commodities](https://paisa.fyi/reference/commodities/)
- Budgets use ledger periodic transactions (`~ Monthly`) in an envelope style with automatic rollover, which can be disabled. You must write an interval such as `~ Monthly in 2023/08/01` because of a ledger-cli bug. — [Paisa budget](https://paisa.fyi/reference/budget/)
- Recurring transactions are identified by posting metadata `; Recurring: <name>`, usually attached through ledger automated transactions (for example `= payee=~/^PPF$/`). A `Period:` tag uses cron-like syntax (for example `L * ?` or `15W * ?`). "Recurring page will only display a transaction as recurring if there is more than one transaction with the same tag name." — [Paisa recurring](https://paisa.fyi/reference/recurring/)
- Retirement goal: the target is set explicitly or computed from yearly expenses (an explicit value or "the average of the last 3 year expenses") and the SWR. Savings accounts are chosen with wildcards (`Assets:Equity:*`). Goals are configured in `paisa.yaml`. — [Paisa retirement goals](https://paisa.fyi/reference/goals/retirement/)
- Sheets is an "experimental" notepad calculator. It has variables, user functions and `{query}` blocks that use the search syntax, plus the built-ins `cost()`, `balance()`, `fifo()` and `negate()`. Use cases include EMI and loan maths and capital-gains cost basis. — [Paisa sheets](https://paisa.fyi/reference/sheets/)
- Desktop apps exist for Linux (.deb), macOS (.dmg) and Windows (.exe), with data in `~/Documents/paisa`. Paisa is also available as a CLI (`paisa serve` on :7500), as Docker images (ledger, hledger and beancount variants) and as a Nix flake. The apps are **unsigned**: "Code signing would require me to pay $99 for Mac and approximately $300 for Windows each and every year." Authentication is optional and off by default. — [Installation](https://paisa.fyi/getting-started/installation/)
- Author's framing (older, Show HN, 2023-09-22): "Paisa is a reporting + import/editor tool. The goal is to have a app that handles all the common use cases without going to command line." He chose ledger over beancount because "Beancount doesn't support some features like periodic transactions (which I use for budgeting)." — [Show HN: Paisa](https://news.ycombinator.com/item?id=37613054)

**Documented limitations by backend**
- hledger: Recurring is "partial" because hledger "doesn't allow to add **only** metadata to transaction". Beancount: "Budget is based on periodic transactions that is no supported by beancount"; Recurring is partial because there are no automated transactions; the search filter does not support the note property. — [Paisa ledger-cli reference](https://paisa.fyi/reference/ledger-cli/)

**Security incident**
- Issue #294 (2024-10-20): Paisa ≤0.7.0 had an unauthenticated remote code execution vulnerability. The auth middleware checked `RequestURI` instead of `URL.Path`, so a URL-encoded path such as `/%61pi/config` bypassed auth. `/api/sheets/save` then allowed path traversal to overwrite the `ledger` binary, and `/api/editor/validate` executed it. v0.7.1 (2024-10-20) fixed it. — [Issue #294](https://github.com/ananthakumaran/paisa/issues/294); [releases](https://github.com/ananthakumaran/paisa/releases)

**User complaints and feature requests (GitHub issues)**
- #331 "beancount3 support broken" (opened 2025-03-15, open, 0 comments): "FATAL exec: 'bean-report': executable file not found in $PATH". Beancount 3 no longer ships `bean-report`. — [Issue #331](https://github.com/ananthakumaran/paisa/issues/331)
- #408 "Import enhancements for better automation" (2026-08-22, open) asks for YAML rule-driven import configuration, a CLI (`paisa import --all`), moving processed files to `imports/processed` or `imports/errors`, duplicate detection by fingerprint (date, amount, description, reference plus account) and web UI integration. — [Issue #408](https://github.com/ananthakumaran/paisa/issues/408)
- Open multi-currency and commodity bugs: #349 "Wrong calculation for commodities" (2025-06-03); #352 "Incorrect currency conversion: Yahoo prices are multiplied unnecessarily by exchange rates" (2025-06-23); #289 "Commodity names with numbers cannot be parsed" (2024-09-30). Closed but indicative: #235 "Assets -> Allocation doesn't respect default currency" (2024-05, 10 comments) and #238 "prices from MFAPI are not converted to default currency". — [GitHub issue search, ananthakumaran/paisa](https://github.com/ananthakumaran/paisa/issues?q=is%3Aissue+currency)
- Other open issues: #370 "Error during journal sync" (2025-11-14); #303 "Invalid time zone" (2024-11-08); #178 "Ignore virtual postings" (2024-02-13); #288 "Import from Beancount has issues" (2024-09-30); #312 "Average unit price of an asset" (2025-01-12); #221 "Echo function required" (2024-05-06). — [Paisa issues sorted by reactions](https://github.com/ananthakumaran/paisa/issues?q=is%3Aissue+sort%3Areactions-%2B1-desc)
- Show HN feedback (older, Sept 2023): lack of bank sync ("I simply cannot use this system manually because I have so many accounts"), no mobile app, limited multi-currency, no tag filtering, no OFX import. The author on OFX: "OFX is unheard of in my country." On Plaid: "as a free app I don't think it can support Plaid." On tax: "I think, I have to figure out a DSL." — [Show HN: Paisa](https://news.ycombinator.com/item?id=37613054)
- Praise: an HN commenter (Sept 2024) called Paisa's CSV import "very convenient... you upload csv, see the preview". — [HN: Plain Text Accounting (2024)](https://news.ycombinator.com/item?id=41550603)

### Inferences
- Paisa's CLI-scraping architecture (CSV format strings, `bean-report`, `bean-query` column lists) is brittle. Upstream CLI changes break it silently: Beancount 3 dropped `bean-report`, and Fava itself dropped Beancount 2 in May 2026 (see section 3). So Paisa's Beancount path is effectively stuck on an obsolete Beancount version. A new core should expose a stable in-process API or a versioned JSON schema instead of relying on text-output scraping.
- A second copy of the data in SQLite means a "sync" step and possible staleness or sync errors (#370). An in-memory or incremental model with file watching would avoid the explicit sync.
- Many "missing" features (budgets on beancount, recurring on hledger) come from feature gaps between ledger dialects, not from the UI. A new core that natively supports periodic/budget entries, automated metadata and recurring schedules removes the need for per-backend feature matrices.
- Paisa is strongly India-centric by default (INR, en-IN, April fiscal year, MF API and NPS providers). For a Brazilian user, pt-BR locale, BRL, comma decimal marks and OFX (common in Brazil) would have to be first-class. This is an inference from the documented locale options (en-US and en-IN only listed).
- The RCE shows that a local web UI that can write files and execute binaries needs auth-by-default, localhost binding and path sanitisation.
- The slow release cadence and the single maintainer make Paisa a good source of UX inspiration rather than a platform to depend on.

### Gaps
- Exact Paisa star or issue counts over time, and download numbers for the desktop app, were not found.
- Whether the desktop app uses Wails or another webview framework is not stated in the docs; the fetch tool guessed "webview-based".
- No official GitHub Security Advisory (GHSA) for #294 was found.
- Could not read GitHub Discussions (if enabled) or the Matrix chat for more complaints.

## 2. Paisa import template system (Handlebars, PDF-to-rows) and its weaknesses

### Takeaway
Paisa converts CSV, TXT, XLS, XLSX and PDF into rows (spreadsheet-like columns A, B, C…). The user writes a **Handlebars** template per bank that emits a ledger transaction for each row, with helper functions and a `predictAccount` helper that guesses the counter-account from similar existing transactions. There is a three-pane UI (file preview, template editor, ledger preview). Its weaknesses: it requires coding-style templates, PDF parsing is experimental, it is web-UI-only (no CLI or batch automation), it has no built-in dedup or file archiving, it has no learned rules outside the template, and it has no OFX, QIF or bank-sync support.

### Cited Findings
- Supported inputs are CSV, TXT, XLS, XLSX and PDF, and "PDF support is in an experimental stage and may not accurately detect rows." — [Paisa import docs](https://paisa.fyi/reference/import/)
- "Each row in a file becomes a single transaction." The UI has three components: file preview, ledger preview and template editor. — [Paisa import docs](https://paisa.fyi/reference/import/)
- "The template is written in Handlebars. Paisa provides a few helper functions to make it easier to write the template." Templates "are stored in the configuration file" (`import_templates` in `paisa.yaml`). Built-in templates ship with Paisa, and users "Save As" their own. — [Paisa import docs](https://paisa.fyi/reference/import/); [config](https://paisa.fyi/reference/config/)
- Template data: `ROW` is the current row, with columns `ROW.A`, `ROW.B`… and an index; `SHEET` is the whole sheet as an array, so any cell can be addressed as `SHEET.5.A`. That is useful for statement headers such as account numbers. — [Paisa import docs](https://paisa.fyi/reference/import/)
- Helpers:
  - logic: `eq`, `not`, `and`, `or`, `gte`, `gt`, `lte`, `lt`
  - transforms: `amount`, `round`, `date`, `trim`, `replace`, `toLowerCase`, `toUpperCase`, `capitalize`, `acronym`
  - validation: `isDate`, `isBlank`, `regexpTest`
  - extraction: `regexpMatch`, `textRange`, `findAbove`, `findBelow`
  - mapping: `predictAccount`, `match`

  — [Paisa import docs](https://paisa.fyi/reference/import/)
- `predictAccount()` "searches for similar transactions in existing ledger data". "Prediction will only work if you have similar transactions in ledger file." It returns "Unknown" if there is no match, and has an optional `prefix` filter. — [Paisa import docs](https://paisa.fyi/reference/import/)
- The docs admit: "The import system is designed to be extensible and might not be intuitive if you are not accustomed to coding." — [Paisa import docs](https://paisa.fyi/reference/import/)
- Users are asking for automation that is not there: a YAML rule config, a `paisa import` CLI, archiving processed files and fingerprint dedup (Issue #408, 2026-08-22). — [Issue #408](https://github.com/ananthakumaran/paisa/issues/408)
- Historical bugs and support requests: #54 "`regexpMatch` helper doesn't work in Import" (2023, closed); #377 "need help for convert csv from finances2" (2026-01-06, closed); #288 "Import from Beancount has issues" (open). — [#54](https://github.com/ananthakumaran/paisa/issues/54); [#377](https://github.com/ananthakumaran/paisa/issues/377); [#288](https://github.com/ananthakumaran/paisa/issues/288)
- Comparison with hledger: hledger uses declarative `.csv.rules` files (`fields`, `skip`, `currency`, `account1`, `if` blocks setting `account2`). `hledger import` "ignores transactions it has seen before, so it's safe to run it repeatedly" by keeping a `.latest.<file>` state file, and it has `--dry-run`. — [hledger CSV import tutorial](https://hledger.org/import-csv.html)
- Comparison with GnuCash: its import matcher uses a Bayesian approach. It tokenizes description, date and amount, builds per-account token frequency tables, picks the highest-scoring account, lets the user override the choice, and shows matching existing transactions so the user can mark duplicates. — [GnuCash manual 5: Importing Transactions](https://gnucash.org/docs/v5/C/gnucash-manual/trans-import.html)

### Inferences
- Handlebars templates are powerful (they can address any cell and handle header metadata in PDFs and XLSX) but they mix three concerns: parsing layout, mapping columns and categorising. Mainstream apps separate them: a column-mapping wizard, then a rules engine that learns from corrections, then dedup and matching. A new core should offer a declarative mapping (like hledger rules or a YAML/TOML schema) plus learned rules and a fingerprint-based dedup, with the template or scripting escape hatch kept for hard cases.
- `predictAccount` works only when similar history exists, and it lives inside the template. Moving categorisation into a global, learnable rules layer (Actual, Lunch Money, GnuCash Bayesian) would help new users on day one.
- PDF parsing being experimental and bank-specific suggests a plug-in point for extractors (table extraction, OCR, optional local LLM) that output normalised rows before mapping.
- Lack of a CLI or batch import and of dedup makes Paisa unsuitable for scheduled or automated pipelines. A new core should be headless-first (library plus CLI), with the UI as a client.

### Gaps
- The algorithm behind `predictAccount` (for example TF-IDF, BM25 or Bayes) is not documented on the import page; source code was not inspected.
- Whether Paisa does any duplicate detection on import is not documented. #408 implies it does not, but this is not confirmed by the maintainer.
- No list of the built-in bank templates (and their countries) was retrieved.

## 3. Other plain-text-accounting frontends and tooling (Fava, hledger-web/ui, ledger-mode, VS Code, LSPs, tree-sitter, mobile, Puffin, Klirr)

### Takeaway
The ecosystem is broad but fragmented:
- **Fava** (Beancount) is the most mature, actively maintained web UI.
- **hledger** ships `hledger-web` (browse and add), `hledger-ui` (TUI) and `add`. Its 2.0 previews (2026) add lot tracking, `holdings` and a `get` price command.
- Editor support is shifting to **language servers**: beancount-language-server (Rust with tree-sitter) and hledger-lsp (Go, own parser). No comparable ledger LSP was found.
- Mobile is the weakest area: Android entry apps (NanoLedger, MoLe via hledger-web) and PWAs (cashier). One old app (Cone) was discontinued after data-corruption reports.
- "Klirr" is not a PTA frontend. It is a Rust/Typst invoice generator.

### Cited Findings
**Directory views**
- plaintextaccounting.org lists:
  - Web UIs: Fava, hledger-web, BeanHub (Git-integrated Beancount UI), Paisa, ledger-analytics, ledgeraccounting, Ledger Web.
  - TUIs: hledger-ui, hledger add, hledger-iadd, Puffin, regdel, bean-add.
  - Desktop: Ledgera, hledger-macos, Prudent, Finzytrack ("Beancount GUI with AI features").
  - Mobile: MoLe, Cone, NanoLedger, Beancount Mobile CE, cashier.
  - Editors: ledger-mode, beancount-mode, vim-ledger, vim-beancount, vscode-beancount, hledger-vscode.

  — [plaintextaccounting.org](https://plaintextaccounting.org/)
- hledger's own UI page (2026) annotates third-party UIs with years and test notes:
  - TUIs: puffin (Go, 2023+); ldash (Rust, 2026); hledger-textual (Python TUI for viewing, entry and editing, 2026); dravik (Python, 2025, "build failures reported").
  - GUIs: hledger-macos (Swift, 2026); **Surebeans**, a "YNAB clone using hledger data format, providing data entry, budgeting, reports" (C#, 2026, closed source); fruit-credits (GNOME, 2024).
  - Web: hledger-webuix (2026); muhasib-e-hledger (Rust, 2024); Paisa (2022+); nextcloud-hledger (2021).

  — [hledger UIs](https://hledger.org/uis.html)

**Fava (Beancount web UI)**
- Changelog:
  - v1.26 (2023-09-04, older): charts improvements; extensions can provide endpoints.
  - v1.27 (2024-01-06): posting metadata in entry forms; faster editor.
  - v1.28 (2024-07-07): watchfiles-based change detection.
  - v1.29 (2024-10-09): query results rendered in the frontend, dark mode, numeric filters on units, price and cost.
  - v1.30 (2024-12-29): "Support for Beancount version 3 was added"; queries via `beanquery`; Svelte 5.
  - v1.30.13 (2026-05-19): **discontinued Beancount 2 support**, Weblate translations, Korean.

  — [Fava changelog](https://beancount.github.io/fava/changelog.html)
- A Fava release was published on PyPI on 2026-08-18 (version not captured in search snippet). — [fava on PyPI](https://pypi.org/project/fava/)
- ~2.6k GitHub stars. Described as "a web interface for the double-entry bookkeeping software Beancount with a focus on features and usability". — [beancount/fava](https://github.com/beancount/fava)
- HN users (2024) praise Fava's document attachments: "document tag and that document will show up directly associated in the ui". — [HN 2024](https://news.ycombinator.com/item?id=41550603)

**hledger built-in UIs and core**
- hledger-web (1.50 manual): "a simple web application for browsing and adding transactions". It is more user-friendly than the CLI or TUI, showing accounts, the current register and balance charts, with "history-aware data entry, interactive searching, and bookmarking". — [hledger-web manual 1.50](https://hledger.org/1.50/hledger-web.html) (via search snippet)
- hledger releases:
  - 1.51 (2025-12-05): `--find` for commodities, payees and tags; `accounts --tree` context.
  - 1.51.2 (2026-01-08).
  - 1.52 (2026-03-20): commodity tags; Gain account type; price lookup O(log n).
  - 1.52.2 and 1.52.3 (2026-08-24 and 08-27): **hledger-web XSS fixes**.
  - 1.52.4 (2026-09-10).
  - 2.0 previews 1.99.1 (2026-03-28) through 1.99.4 (2026-09-10): Beancount-like cost-basis syntax `{DATE, "LABEL", COST}`, automatic lot tracking and capital gains, `check lots`, a new **`get` command for fetching transaction data and market prices** (1.99.3, 2026-06-24), a new **`holdings`** command (1.99.4), command aliases in config, hledger-web CSP headers.
  - 1.99.1 was described as the "first AI-assisted development release with published policy".

  — [hledger release notes](https://hledger.org/relnotes.html)
- hledger-ui 1.50.x: `--watch` now detects changes made by apps that overwrite the file, such as VS Code. — [hledger release notes](https://hledger.org/relnotes.html) (via search snippet)
- Maintainer note (HN 2024): "Long-running apps like hledger-ui and hledger-web do parsing only once at startup." — [HN 2024](https://news.ycombinator.com/item?id=41550603)

**TUIs**
- Puffin: "Terminal dashboard to manage personal finances" built with hledger and Bubbletea (Go). It **wraps hledger by executing shell commands**. It shows assets, expenses, revenue, liabilities, register, accounts, the income statement and the balance sheet, with filters by account, date and period, and depth control. It has 571 stars. v3 was "paused as of February 2026". — [siddhantac/puffin](https://github.com/siddhantac/puffin)

**Editor tooling: LSPs and tree-sitter**
- beancount-language-server (polarmutex, Rust, ~250 stars):
  - Supports VS Code, Neovim, Helix, Zed, Emacs, Vim and Sublime.
  - Implemented: completions (accounts with hierarchy, directives), diagnostics (with bean-check integration), formatting "compatible with bean-format", rename, references, semantic highlighting and **inlay hints** (implicit balancing amounts, unbalanced warnings).
  - Planned: hover, go-to-definition, document symbols, code actions, folding.
  - It uses the tree-sitter parser `tree-sitter-beancount`.

  — [polarmutex/beancount-language-server](https://github.com/polarmutex/beancount-language-server); [lib.rs](https://lib.rs/crates/beancount-language-server)
- hledger-lsp (juev, Go, ~42 stars):
  - Completions for accounts, payees, commodities, tags and dates.
  - Diagnostics (balance checks) validated against "hledger 1.52.4 semantics" using **its own parser** (it does not shell out).
  - Formatting (amount alignment), **hover showing account balances**, go-to-definition, rename and semantic tokens.

  — [juev/hledger-lsp](https://github.com/juev/hledger-lsp)
- VS Code extensions for hledger include "hledger Language Support" (evsyukov: completion, amount alignment, project caching), juev/hledger-vscode (LSP-based, inline completions, transaction templates, on-type formatting) and patrickt's hledger-lsp extension. They were updated as recently as Sept 2026 per search results. — [evsyukov.hledger](https://marketplace.visualstudio.com/items?itemName=evsyukov.hledger); [juev/hledger-vscode](https://github.com/juev/hledger-vscode); [patrickt hledger-lsp on Open VSX](https://open-vsx.org/extension/patrickt/hledger-lsp-vscode/changes)
- Tree-sitter grammars:
  - tree-sitter-hledger (chrislloyd) — [repo](https://github.com/chrislloyd/tree-sitter-hledger)
  - tree-sitter-ledger (cbarrete): "Only the Nix toolchain is supported"; npm is best-effort and not actively maintained — [repo](https://github.com/cbarrete/tree-sitter-ledger)
  - tree-sitter-beancount (polarmutex), used by the Beancount LSP — [repo](https://github.com/polarmutex/tree-sitter-beancount)

**Mobile**
- hledger's mobile page lists:
  - NanoLedger (Android, 2023-, entry only, exports an h/ledger journal, active).
  - Cashier / Cashier II (PWA, 2022-2024, entry and reports, works offline).
  - MoLe (Android, 2018-2024, entry and reports, "Connects to a hledger-web server").
  - Cone (Android, 2019-2021, **discontinued**, "reports of data corruption and performance issues with synced files").
  - Generic expense apps that export CSV (GnuCash, MyExpenses, Money Manager Ex…).

  — [hledger mobile apps](https://hledger.org/mobile.html)
- NanoLedger supports ledger and hledger syntax, shows all transactions in a file and allows adding new ones "with auto-completion supported for payee, note and account names". v1.4.1 was added to F-Droid on 2026-04-07. — [NanoLedger on F-Droid](https://f-droid.org/en/packages/be.chvp.nanoledger/)
- MoLe requires a running hledger-web server reachable from the phone, and adds transactions directly without export or import. — [MoLe](https://mole.ktnx.net/); [F-Droid](https://f-droid.org/packages/net.ktnx.mobileledger/)

**Klirr**
- Klirr (Sajjon/Alexander Cyon) is "Zero-maintenance and smart FOSS generating beautiful invoices for services and expenses", written in Rust with Typst. The name comes from the Swedish "klirr i kassan". It is an invoicing tool, not a PTA frontend. — [Sajjon/klirr](https://github.com/Sajjon/klirr); [Rust forum](https://users.rust-lang.org/t/klirr-a-config-once-inter-month-idempotent-calendar-aware-capable-and-maintenance-free-invoice-solution-written-in-rust-typst/131260)

### Inferences
- The best-regarded frontends (Fava, hledger-web) are tightly coupled to their own core's in-process data model. Scrapers of CLI output (Paisa, Puffin) inherit fragility. A new Python core should expose a documented library API plus JSON output, and ideally a local HTTP/JSON API, so that web UIs, TUIs, mobile apps and LSPs don't reparse text.
- LSP features that users value (account completion, balance diagnostics, inlay hints for the implied amount, hover balances, formatting, rename of accounts) are cheap for a core that already has a parser with source positions. Shipping an LSP and a tree-sitter grammar from day one would beat ledger, which has neither an LSP nor a maintained npm tree-sitter.
- Mobile remains an unsolved gap. The viable patterns are (a) an offline entry app that appends to a synced file (NanoLedger/Syncthing style) and (b) a thin client over a server API (MoLe over hledger-web). The Cone history shows that sync conflicts and corruption are real risks. Append-only inbox files merged by the core would be safer.
- Fava dropping Beancount 2 and hledger adding lot tracking, `holdings` and a price-fetching `get` in 2.0 suggest the cores are converging on investment features and built-in price fetching. A new core should include both natively.

### Gaps
- Could not retrieve Fava's in-app help pages (editor autocompletion, import page with beangulp importers, budgets). Details of Fava's import UI are unverified here.
- Could not fetch the ledger-mode (Emacs) manual (404 or redirect). Its reconcile-mode workflow is not documented in these notes.
- No ledger (C++) language server was found. No version or date was captured for the latest beancount-language-server release (Homebrew lists 1.3.4 per [libraries.io](https://libraries.io/homebrew/beancount-language-server), date unknown).
- Beancount Mobile CE, BeanHub, Ledgera and Finzytrack were only seen as directory entries; they were not evaluated.
- The latest exact Fava version number (Aug 2026 release) was not captured.

## 4. Mainstream personal finance apps: what users love and what PTA lacks

### Takeaway
Mainstream apps win on **automation of the boring parts**:
- bank sync or easy multi-format import
- rules engines that **learn from the user's own corrections**
- automatic recurring or subscription detection
- a clear "review inbox" and reconciliation workflow
- polished budgeting methods (YNAB targets, Age of Money) and net-worth dashboards
- onboarding

PTA tools already have the rigour (double entry, balance assertions, version control) but lack these interaction layers.

### Cited Findings
**Actual Budget (open source, local-first)**
- Rules engine: "When importing or syncing transactions, they are run through a list of rules that can apply actions to the transaction."
  - Conditions: is / is not / contains / matches (regex) / one of.
  - Actions set category, payee, notes, cleared, account, date or amount.
  - Rules run in 3 stages, `pre`, `default` and `post`, and "Within each stage, rules are automatically ranked from least to most specific".
  - Rules are learned: "Actual will **automatically create rules for you** based on your behavior. As you rename payees or categorize transactions, it will use rules as a mechanism for writing down what you've done."

  — [Actual rules docs](https://actualbudget.org/docs/budgeting/rules/)
- Bank sync:
  - GoCardless (EU/UK) and SimpleFIN (US/CA, $1.50/month, up to 90 days of history).
  - From **July 2025 GoCardless stopped accepting new Bank Account Data accounts**.
  - Enable Banking was added as an experimental provider (26.6.0) and Akahu for NZ (26.7.0).
  - File imports: QIF, OFX, QFX, CAMT.053 and CSV.

  — [Actual bank sync](https://actualbudget.org/docs/advanced/bank-sync/); [GoCardless setup](https://actualbudget.org/docs/advanced/bank-sync/gocardless/); [SimpleFIN](https://actualbudget.org/docs/advanced/bank-sync/simplefin/); [Actual release notes](https://actualbudget.org/docs/releases/)
- Monthly releases in 2026 (CalVer):
  - 26.5.0 (2026-05-03): Sankey report, Age of Money (experimental), themes.
  - 26.6.0 (06-01): custom themes stable, Balance Forecast report, tag autocomplete.
  - 26.7.0 (07-01): **Actual CLI stable**, tag management, Monthly Spending report.
  - 26.8.0 (08-02): **redesigned onboarding**, **mobile account reconciliation**, bank sync setup from mobile.
  - 26.8.1: performance hotfix for large budgets.
  - 26.9.0 (09-01): customizable transaction table columns, **in-app onboarding tour**, a "schedule" button for future-dated transactions, a **Monte Carlo retirement report**, formula reports with `BALANCE_OF` and loan/investment functions.

  — [Actual release notes](https://actualbudget.org/docs/releases/)

**Lunch Money (commercial)**
- Rule conditions: payee (contains / starts with / exact), category, notes, amount ranges, day range (with wrapping) and account.
- Rule actions: rename payee, set category, add tags, link recurring items, split, mark reviewed, email notification.
- "A category rule is automatically created with a 'match exactly' rule whenever you change the category of a transaction". Edits prompt the user to create rules. Suggested rules are reviewed before retroactive application. Rules apply to synced, manual and CSV/PDF-imported transactions; they are off by default for API-added transactions.

  — [Lunch Money rules](https://support.lunchmoney.app/setup/rules)
- Recurring: Lunch Money "will automatically detect recurring transactions if there is a clear pattern of payee and amount that repeat over a regular cadence", creating a *suggested* recurring item. — [Lunch Money recurring items](https://support.lunchmoney.app/finances/recurring-items) (via search snippet)

**YNAB**
- Reconcile: "Your account registers should be exact mirrors of your accounts at your banks", verified via Reconcile. — [YNAB glossary](https://support.ynab.com/en_us/ynab-glossary-a-guide-BJd80SORq) (via search snippet)
- Age of Money: the average number of days between earning and spending money. — [YNAB: Age of Money](https://support.ynab.com/en_us/age-of-money-H1ZS84W1s)
- Target types: weekly, monthly and savings-by-date. Irregular bills get monthly targets so "by the time the bill arrives, the money is already there". — [Freenance YNAB review 2026](https://freenance.io/products/ynab-review-2026-budgeting-app-worth-it-european-investors/) (third-party)
- Pricing is ~$109/yr vs Monarch $99.99/yr (third-party comparison). — [Finny blog](https://getfinny.app/blog/ynab-vs-monarch-money-2026)

**Monarch Money**
- Aggregates banks, cards, loans, investments and real estate; tracks net worth, investments and cash flow; customizable dashboards; household sharing; "automatically detects recurring bills and subscriptions". The framing: "YNAB is a method… Monarch is a dashboard that shows you the whole picture". — [Monarch vs YNAB](https://www.monarch.com/compare/ynab-alternative); [envelopebudgeting.com comparison](https://envelopebudgeting.com/articles/monarch-vs-ynab) (third-party, via search snippets)

**Firefly III (self-hosted)**
- Features: "Rule based transaction handling", recurring transactions, budgets, piggy banks, reports, multi-currency, a near-complete REST API, double-entry, privacy ("will never contact external servers until you explicitly tell it to"). ~24.7k stars. — [firefly-iii/firefly-iii](https://github.com/firefly-iii/firefly-iii)

**Maybe Finance, now Sure**
- The Maybe repo was **archived on 2025-07-27**: "Maybe is pivoting to B2B financial forecasting… will no longer be actively maintaining this repository." — [maybe-finance/maybe](https://github.com/maybe-finance/maybe) (via search snippet)
- The community fork **Sure** (we-promise/sure) has ~10.0k stars. Features: net worth, investments, Plaid and SimpleFIN sync, transaction rules, an AI assistant (pgvector), budgets and CSV import. Stack: Rails, Postgres, Redis and Sidekiq. It is actively maintained. — [we-promise/sure](https://github.com/we-promise/sure)

**GnuCash**
- Bayesian import matcher that learns account assignments from token frequencies, with user override and duplicate matching against existing transactions. — [GnuCash manual](https://gnucash.org/docs/v5/C/gnucash-manual/trans-import.html)

### Inferences
- The single most transferable pattern is to **learn rules from corrections**. Actual, Lunch Money and GnuCash all turn a user's manual categorisation into a persistent rule or model. In PTA this could be a human-readable rules file (versioned alongside the journal) that the UI writes automatically, with staged, specificity-ordered evaluation (as in Actual).
- A **"to review" inbox** (Lunch Money's reviewed/unreviewed, the Actual and YNAB cleared/reconciled states) maps naturally onto PTA's `!` / `*` status flags and balance assertions. The missing piece is UI. A reconcile screen should take the statement balance, auto-generate a balance assertion and show the unmatched items.
- **Recurring detection** should be automatic (payee plus amount plus cadence) and produce *suggestions*. Paisa requires manual `Recurring:` tags and at least two occurrences.
- Bank sync is fragile and regional: GoCardless closed to new users in July 2025, and SimpleFIN covers only the US and Canada. A PTA core should treat sync as pluggable providers feeding the same import pipeline. For Brazil, Open Finance or OFX exports would be the relevant path; this is an inference, not researched here.
- Mainstream apps invest heavily in **onboarding** (Actual's 26.8 and 26.9 tours) and **forecasting and retirement** (Actual's Monte Carlo report, Monarch's dashboards, Paisa's retirement goal). Goal and forecast features are table stakes for a "pleasant" tool.
- Maybe's shutdown and archive shows a risk of VC-backed "open" apps. Plain-text data sidesteps lock-in, which remains a core PTA selling point (also cited on HN).

### Gaps
- Firefly III's rules docs returned 403, so there is no detail on its triggers, actions or "strict mode" here.
- lunchmoney.app/features returned 403, so the full feature list is not captured.
- No primary-source user-satisfaction data (surveys) was found for which features users "love". The evidence is vendor docs, third-party reviews and forum anecdotes.
- Monarch's AI categorisation or rules details were not verified from primary docs.

## 5. Top barriers to adoption of plain text accounting (user-cited)

### Takeaway
Across HN (2023, 2024), Lobsters (2026), blogs and the PTA forum, the recurring barriers are:
1. manual entry and time cost
2. import friction (inconsistent bank CSV/PDF, no OFX, no sync)
3. lack of mobile or real-time capture
4. learning curve (double entry, syntax)
5. investment and multi-currency tracking
6. sharing with non-technical partners

Reporting and visualisation is less of a complaint when Fava or Paisa are used. LLMs are increasingly cited (2024-2026) as a way to cut import and categorisation work.

### Cited Findings
- HN "Plain Text Accounting" thread (Sept 2024):
  - Banks resist standardization: "Banks are much less technological than the common stereotype."
  - Monthly import sessions of 30-60 minutes: "I planned to do it monthly but it's a bit of a chore" (RivieraKid).
  - Immediate strict categorisation fails in practice; the only workflow that stuck in one company was iPhone notes parsed into ledger.
  - One commenter called ledger's schema "absolutely atrocious" and wanted programmatic entry rather than editing in vim.
  - Liked: version control, scriptability, double-entry validation and no vendor lock-in (QuickBooks and Xero price hikes).

  — [HN 41550603](https://news.ycombinator.com/item?id=41550603)
- Show HN Paisa (Sept 2023, older): "I simply cannot use this system manually because I have so many accounts"; requests for mobile, Plaid, OFX, multi-currency, tags and cash-flow forecasting. — [HN 37613054](https://news.ycombinator.com/item?id=37613054)
- Lobsters "Plain Text Accounting is Pretty Cool" (submitted 2026-08-19):
  - "Do you really enter every transaction manually into hledger?"
  - Regional bank data quality: a bank that "just shows 'Spending' for every transaction".
  - Shared finances with non-technical partners and Splitwise; uncertainty about investments.
  - Workarounds: weekly CSV export, SMS-to-Tasker webhooks, Syncthing, LLMs (Claude) to write import scripts.
  - Double entry is "such a cheat-code once you understand how it works".

  — [Lobsters](https://lobste.rs/s/i4lxyt/plain_text_accounting_is_pretty_cool)
- Blog post being discussed (Sumner Evans, dated 2026-08-15 per fetch): "Of course, I'm back to manually entering every transaction into my ledger, but the amount of accounting rigour I'm able to apply to all my transactions makes it worth it for me." The author uses hledger CLI and hledger-web after leaving Mint, Rocket Money and Origin. — [sumnerevans.com](https://sumnerevans.com/posts/money/plain-text-accounting/)
- PTA forum, "Anyone else using LLMs with plain text accounting?" (2024-03-11, older):
  - GPT-4 was used to convert CSV/OFX to Beancount instead of writing parsers.
  - Constraint from the author: "my partner (non-technical background) must be able to understand every step".
  - simonmic noted the energy and cost concern.

  — [forum.plaintextaccounting.org](https://forum.plaintextaccounting.org/t/anyone-else-using-large-language-models-llms-with-plain-text-accounting-psa-it-works-well/108)
- Vendor blog (beancount.io, 2025-07-28; commercial, possibly marketing-driven): users paste messy CSVs into LLMs to convert to Beancount ("not always 100% perfect"). "Context window management is the main challenge with long statements". "PDF statement formats vary enormously between banks" and LLM extraction handles the variation better than regex. — [beancount.io blog](https://beancount.io/blog/2025/07/28/user-experience-and-feedback-on-llm-assisted-plain-text-accounting)
- Mobile capture is still weak: Cone was discontinued after data-corruption reports, MoLe needs a hledger-web server, NanoLedger is Android-only and entry-only, and Paisa has no mobile app. — [hledger mobile](https://hledger.org/mobile.html); [Paisa installation](https://paisa.fyi/getting-started/installation/)
- Investment tracking: hledger only in 2026 (1.99.x previews) added automatic lot tracking, capital gains and `holdings`. Paisa has open issues on average unit price (#312) and commodity valuation (#349, #352). — [hledger relnotes](https://hledger.org/relnotes.html); [Paisa #312](https://github.com/ananthakumaran/paisa/issues/312); [#349](https://github.com/ananthakumaran/paisa/issues/349); [#352](https://github.com/ananthakumaran/paisa/issues/352)

### Inferences
- The barrier ranking (manual entry and import first, then mobile, learning curve and investments) is qualitative and consistent across 2023-2026 threads. The problem has not been solved by existing frontends. Paisa's import preview was singled out as helpful, but it still needs per-bank templates.
- The quickest wins for a new core:
  - A zero-config first import: auto-detect CSV/OFX columns, preview, learned rules, dedup.
  - A mobile-friendly quick-entry path (PWA or append-only inbox file).
  - Guided onboarding (templates for common account trees, opening balances via balance assertions).
  - Friendly error messages and LSP diagnostics to reduce the syntax learning curve.
- LLM assistance is becoming a de facto import tool. The core should offer a structured, verifiable pipeline where the LLM proposes rows or categories and the core validates them (balancing, dedup, assertions), with local-model options for privacy. The partner-comprehensibility and "not 100% perfect" comments point to showing a diff or preview before writing.

### Gaps
- Reddit r/plaintextaccounting threads could not be retrieved directly (no Reddit fetch performed). Evidence comes from HN, Lobsters, the PTA forum and blogs instead.
- No quantitative survey of PTA users' pain points was found.
- No specific "why I stopped using ledger" post was found in 2025-2026 searches; the search returned only pro-PTA or comparison posts.

## 6. UX and feature ideas a new plain-text-accounting core should support (synthesis)

### Takeaway
To be easier than Ledger, hledger or Beancount while keeping plain text, a new core should be a headless, library-first engine with:
- a stable JSON/HTTP API (so frontends such as a Paisa-like UI never scrape CLI text)
- a built-in import pipeline (mapping, learned rules, dedup, preview, archive)
- native budgets, recurring and goals
- built-in price fetching and lot tracking
- an LSP and a tree-sitter grammar
- safe local-web defaults
- a mobile-friendly capture path

### Cited Findings
- CLI scraping breaks: Paisa's Beancount support broke when `bean-report` was removed (#331). Paisa builds custom `ledger csv --csv-format` strings and `bean-query` column lists. — [Paisa #331](https://github.com/ananthakumaran/paisa/issues/331); [ledger.go](https://github.com/ananthakumaran/paisa/blob/master/internal/ledger/ledger.go)
- Import automation that users ask for: YAML rules, a CLI, archiving and fingerprint dedup (Paisa #408). hledger's `.latest` dedup and `--dry-run` show a minimal working model. — [Paisa #408](https://github.com/ananthakumaran/paisa/issues/408); [hledger import-csv](https://hledger.org/import-csv.html)
- Learned, staged rules ordered by specificity (Actual). Automatic category rules on correction plus suggested rules (Lunch Money). Bayesian matcher (GnuCash). — [Actual rules](https://actualbudget.org/docs/budgeting/rules/); [Lunch Money rules](https://support.lunchmoney.app/setup/rules); [GnuCash import](https://gnucash.org/docs/v5/C/gnucash-manual/trans-import.html)
- Feature parity pain: Paisa budgets need ledger periodic transactions (not in Beancount), and recurring is partial on hledger and Beancount. — [Paisa ledger-cli reference](https://paisa.fyi/reference/ledger-cli/)
- Cores are adding price fetching and lots: hledger `get` (prices) and `holdings` (2026 previews); Paisa relies on external providers configured in YAML and updated only on demand. — [hledger relnotes](https://hledger.org/relnotes.html); [Paisa commodities](https://paisa.fyi/reference/commodities/)
- LSP features that already exist for other formats: completion, diagnostics, formatting, rename, references, inlay hints for balancing amounts (Beancount LSP); hover balances and go-to-definition (hledger-lsp). — [beancount-language-server](https://github.com/polarmutex/beancount-language-server); [hledger-lsp](https://github.com/juev/hledger-lsp)
- Security lesson: the Paisa RCE via auth bypass, file write and binary execution (≤0.7.0). hledger-web XSS fixes in Aug 2026. — [Paisa #294](https://github.com/ananthakumaran/paisa/issues/294); [hledger relnotes](https://hledger.org/relnotes.html)
- Mainstream UX investments in 2026: onboarding tours, mobile reconciliation, forecasting and Monte Carlo retirement, custom report formulas (Actual). — [Actual releases](https://actualbudget.org/docs/releases/)

### Inferences
Prioritised feature ideas, derived from the findings above:
1. **Library-first core plus a stable JSON schema and local API.** It should include source positions and transaction ids so that UIs, LSPs and mobile clients share one parser. The goal is to avoid the Paisa, Puffin and ledger-mode pattern of scraping CLI output.
2. **Import pipeline as a first-class subsystem**:
   - readers: CSV, XLSX, OFX/QFX, CAMT.053 and PDF plug-ins
   - a declarative mapping file (hledger-rules-like, human-editable)
   - a learned rules file written automatically from UI corrections, with pre/default/post stages and specificity ordering
   - fingerprint dedup with an import ledger
   - dry-run, diff and preview
   - archiving of processed files
   - CLI and UI parity
   - optional LLM or local-model suggestions that are always validated by the core
3. **Review and reconcile workflow**: an "inbox" of unreviewed or uncleared transactions; a reconcile screen that writes balance assertions; clear, human-readable error messages.
4. **Native planning primitives**: budgets or envelopes with rollover, recurring schedules (cron-like, as in Paisa's `Period:`), automatic recurring detection with suggestions, savings and retirement goals, and forecasts. All of these should work regardless of dialect.
5. **Investments**: lot tracking, cost basis (FIFO and average), unrealised and realised gains, and built-in pluggable price providers (Yahoo, Alpha Vantage, central-bank FX, local fund APIs), with a cached price DB written back as `P` directives.
6. **Editor experience**: an LSP (completion, diagnostics, inlay balancing amounts, hover balances, rename account, formatter) plus a tree-sitter grammar, shipped with the core.
7. **Capture anywhere**: an append-only "inbox" journal that is safe for Syncthing or mobile writes, plus a small PWA or HTTP endpoint for quick entry (the MoLe and NanoLedger patterns).
8. **Localization**: locale-aware number and date parsing and display (for example pt-BR comma decimals), configurable fiscal year and default currency. Paisa documents only en-US and en-IN locales.
9. **Secure-by-default web UI**: localhost binding, auth on, no arbitrary file writes, a CSP. Paisa's RCE and the hledger-web XSS are cautionary examples.
10. **Onboarding**: starter account trees, an opening-balances wizard and a sample journal/demo. Paisa's live demo and Actual's tours are good models.

### Gaps
- These recommendations are synthesis. No user-testing data was found that ranks them quantitatively.
- Brazil-specific aspects (Open Finance APIs, OFX prevalence among Brazilian banks, Tesouro Direto or B3 price sources) were not researched in this task.
