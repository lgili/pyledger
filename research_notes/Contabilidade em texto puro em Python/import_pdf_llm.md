# Transaction import pipelines for plain-text accounting: CSV/OFX/QIF/XLSX/PDF, rule, ML and LLM categorization, dedup and reconciliation (state as of Sept 2026)

Research date: 2026-09-25. Versions and dates come from PyPI JSON metadata, RubyGems, GitHub repository metadata (via GitHub search) and the official docs pages cited inline. Star counts are from GitHub search on 2026-09-25. Anything older than 2025 is marked "(older)".

## Q1. Existing PTA import approaches (hledger CSV rules and `import`, beangulp, beanhub-import, Paisa, ledger-autosync, reckon, smart_importer, beancount-import, icsv2ledger, and others): what each does well and badly

### Takeaway
Every PTA importer does one of two jobs well and the other badly. Declarative converters (hledger CSV rules, beanhub-import, Paisa templates, double-entry-generator) are good at mapping columns and fixed rules. They have little or no learning or reconciliation, and dedup is weak: hledger only compares dates. Interactive or learning tools (reckon, smart_importer, beancount-import, icsv2ledger, beanborg) are good at suggesting accounts, but each needs code or a web app per institution. None of the mainstream tools treats PDF statements as a first-class, verified input. None checks a statement's closing balance by default. None handles credit-card installments.

### Cited Findings

**hledger (manual 1.52, current on hledger.org in Sept 2026)**
- The CSV conversion is built in and driven by a `.rules` file. The rules in the cheatsheet are `source`, `archive`, `encoding`, `separator`, `decimal-mark`, `date-format`, `timezone`, `newest-first`, `intra-day-reversed`, `skip`, `fields`, field assignments, `if` blocks, `if` tables, `end`, `balance-type` and `include` — [hledger manual 1.52, CSV rules cheatsheet](https://hledger.org/hledger.html)
- The `source` rule takes a file name or glob. A bare file name is looked up in `~/Downloads`, and if a glob matches several files the newest one is read. The manual says this "enables a convenient workflow where can you just download CSV files, then run hledger import rules/*" — [hledger manual](https://hledger.org/hledger.html)
- Since 1.50 (marked experimental), `source` can pipe into a data-cleaning command (`source ./paypal.json | paypalcsv`) or run a data-generating command (`source | simplefincsv data/simplefin.json 'chase.*card'`). This lets JSON or API sources feed the CSV rules — [hledger manual](https://hledger.org/hledger.html)
- `archive` (1.50, experimental) moves each imported file into `data/` next to the rules file, with a dated name. With archiving on, a glob picks the oldest match first — [hledger manual](https://hledger.org/hledger.html)
- `if` tables are a compact matcher-to-field-values syntax. `MATCHERB && MATCHERC` has been allowed since 1.42 — [hledger manual](https://hledger.org/hledger.html)
- How `import` handles overlap: "We don't call this 'deduplication', as it's generally not possible to reliably detect duplicates in bank CSV." It keeps a `.latest.FILE` file holding the latest imported date, repeated N times when N records share that date, and skips earlier records. This relies on four assumptions: stable file names, stable dates, stable order within a day, and the newest items having the newest dates. The flags are `--catchup` and `--dry-run`, and `print --new` uses the same mechanism — [hledger manual, import](https://hledger.org/hledger.html)
- `import` explicitly does not catch two kinds of duplicate. The first is identical new records. The second is a new record identical to an entry already in the journal — [hledger manual](https://hledger.org/hledger.html)
- hledger-flow is a layered workflow on top of hledger for automated statement import and classification (Haskell, 219 stars) — [apauley/hledger-flow](https://github.com/apauley/hledger-flow)

**Beancount ecosystem**
- beangulp (0.2.0, 2025-01-20) is "the evolution of `beancount.ingest` in Beancount 2.3 and replaces it in the Beancount 3.0 release". Its documentation is "code docstrings and examples" — [beangulp README](https://github.com/beancount/beangulp), [PyPI](https://pypi.org/project/beangulp/). Beancount itself is at 3.2.3 (2026-05-05) — [PyPI beancount](https://pypi.org/project/beancount/)
- beangulp's duplicate detector is `find_similar_entries(..., window_days=2)` with a `heuristic_comparator`. Two entries count as similar when the dates are close, posting amounts on the same account are within 5% (default `epsilon`), and one entry's account set equals or contains the other's — [beangulp/similar.py](https://github.com/beancount/beangulp/blob/master/beangulp/similar.py)
- smart_importer (1.2, 2025-10-17, 308 stars) adds `PredictPostings` and `PredictPayees` to importers through beangulp hooks. It trains scikit-learn's SVC on the fly from `existing_entries` in the ledger. Custom tokenizers are supported (the README has a jieba example for Chinese). Status is "Working protoype, development status: beta". The README warns that "small or homogeneous datasets may result in poor predictions" — [beancount/smart_importer](https://github.com/beancount/smart_importer), [PyPI](https://pypi.org/project/smart-importer/)
- beancount-import (jbms; 1.4.0, 2024-04-19 (older); 471 stars; still active) is a web UI for "semi-automatically importing … as well as merging and reconciling imported transactions". It has sources for OFX, Mint, Venmo, Amazon, PayPal, HealthEquity, Schwab and others. It links entries to source data through metadata (`source_desc`, date), merges transfers seen from two sources, keeps an "Uncleared" view, and predicts unknown legs with "a learned classifier (currently decision tree-based)" — [jbms/beancount-import](https://github.com/jbms/beancount-import)
- beanhub-import (1.4.1, 2026-06-20; 27 stars) keeps declarative YAML rules in `.beanhub/imports.yaml`, split into `context`, `inputs` and `imports`. Matching can use `equals`, `one_of`, regex (including on file paths, with named groups) and date comparisons. Actions add transactions, delete them, or `ignore`. It is idempotent through `import-id` metadata: "As long as the input data and rules are the same, the Beancount files will be the same". When rules change it will "automatically remove the old ones and add the new ones for you". Extraction lives in a separate library, beanhub-extract (0.1.7, 2025-12-20), with extractors such as mercury, chase_credit_card and plaid. Merging data from several files is listed as "coming soon" — [beanhub-import docs](https://beanhub-import-docs.beanhub.io/), [PyPI beanhub-import](https://pypi.org/project/beanhub-import/), [PyPI beanhub-extract](https://pypi.org/project/beanhub-extract/)
- beancount_reds_importers (0.10.0, 2025-02-24; 166 stars) is a framework for writing importers (OFX, CSV and others) — [redstreet/beancount_reds_importers](https://github.com/redstreet/beancount_reds_importers), [PyPI](https://pypi.org/project/beancount-reds-importers/)
- beanborg (70 stars) runs a 3-stage pipeline: `bb_mover`, then `bb_import`, then `bb_archive`. Categorization tries YAML lookup rules first (equals, contains, startsWith and so on), then "an ML model trained on historical data", then optionally ChatGPT (`OPENAI_API_KEY`, `rules.use_llm: true`). Dedup combines an MD5 of the CSV row stored in metadata with a fallback check on same date and amount in the same account — [luciano-fiandesio/beanborg](https://github.com/luciano-fiandesio/beanborg)
- double-entry-generator (Go, 720 stars) is a rule-based converter for Alipay, WeChat, Huobi and similar sources that writes Beancount or Ledger. It is the most-starred importer in GitHub search results for "beancount importer" — [deb-sig/double-entry-generator](https://github.com/deb-sig/double-entry-generator)

**Ledger-family tools**
- ledger-autosync (1.2.0, 2024-08-22 (older); 322 stars) downloads OFX through Direct Connect and converts it. It writes an `ofxid` (bank FITID) or `csvid` metadata tag on each transaction, then asks ledger or hledger whether that ID already exists. Its payee matching "only does exact matching on the payee" and "is also not currently working with CSV files". `--assertions` emits balance assertions from the OFX balances. For Mint CSVs without IDs it hashes row contents, which "is likely to generate false negatives … It will not generate false positives" — [ledger-autosync README](https://gitlab.com/egh/ledger-autosync/-/raw/master/README.rst), [PyPI](https://pypi.org/project/ledger-autosync/)
- reckon (Ruby; 0.11.1, 2025-03-05; 437 stars) uses "Bayesian machine learning" to suggest accounts. It learns from an existing ledger (`-l`) and from an account-tokens YAML file with a `similarity_threshold`. It auto-detects the money, date and header columns, has interactive and `--unattended` modes, and outputs Ledger or Beancount. The README documents no dedup — [cantino/reckon](https://github.com/cantino/reckon), [RubyGems](https://rubygems.org/gems/reckon)
- icsv2ledger (Python; 200 stars; not on PyPI) is interactive with tab completion. Its mapping file of literal or `/regex/` entries grows from past decisions ("Later matching entries overwrite earlier ones"). `--skip-dupes` compares MD5s of formatted CSV values stored as comments. It needs the `ledger` binary — [quentinsf/icsv2ledger](https://github.com/quentinsf/icsv2ledger)
- Paisa (TypeScript/Go, 3,218 stars) imports CSV, TXT, XLS, XLSX and PDF through Handlebars templates, one row per transaction. Helpers include `predictAccount()` ("find a transaction that is similar … pick the accounts from the top match"), `match()` (regex to account), `amount()` and `isDate()`. The docs say "PDF support is in an experimental stage and may not accurately detect rows" and "Prediction will only work if you have similar transactions in ledger file" — [Paisa import docs](https://paisa.fyi/reference/import/). Its package.json pulls in `pdfjs-dist ^3.10.111`, SheetJS `xlsx 0.19.3` and `handlebars` — [paisa package.json](https://github.com/ananthakumaran/paisa/blob/master/package.json)

**Newer or niche tools listed on plaintextaccounting.org (2026)**
- ledgerbridge reads OFX 1.x/2.x, QFX, QIF, CAMT.053 and CSV and writes Beancount, hledger or Firefly III CSV. It "verifies opening + transactions == closing balance and refuses to write output when it doesn't" (Decimal, zero tolerance, exit 1). IDs are the bank ID (`b:` prefix) or a SHA-256 (`h:`) over date, amount, normalized description (accents folded, punctuation, case and whitespace ignored) and the occurrence count within the day. It deliberately does no categorization: every counter-account is `Expenses:Unknown` or `Income:Unknown`. It is stdlib-only, MIT, 2 stars and not on PyPI — [MugenLab/ledgerbridge](https://github.com/MugenLab/ledgerbridge), [plaintextaccounting.org](https://plaintextaccounting.org/)
- limabean-harvest (Rust/Clojure) is a Beancount importer configured "as data not code". It infers accounts from payee and narration and pairs "transactions between accounts where both accounts are imported in the same group" (transfers). It describes itself as "very new" — [tesujimath/limabean-harvest](https://github.com/tesujimath/limabean-harvest)
- Also listed: buchhaltung (Haskell, CSV/FinTS/HBCI/OFX conversion with dedup), ledger-guesser (a neural net that works with ledger-autosync), finfetch (Plaid download with categorization and dedup), hledger-import-dsl, fints2ledger and plaid2text — [plaintextaccounting.org, Data conversion/import](https://plaintextaccounting.org/)

### Inferences
- **Strengths worth copying.** From hledger: zero-code declarative column mapping, `if` tables for bulk rules, the `source` glob over `~/Downloads`, archiving, and preprocess pipes. From beanhub-import: idempotent regeneration keyed by `import-id`, so changing a rule rewrites earlier generated entries. From ledger-autosync: storing bank IDs as metadata and checking the ledger itself for them. From beangulp: fuzzy similarity within a date window. From ledgerbridge: stable hashes that include the occurrence index, and a hard balance gate. From limabean-harvest and beancount-import: pairing transfers seen from both accounts. From reckon, smart_importer and Paisa: learning from the ledger's own history.
- **Weaknesses to avoid.**
  - hledger's `.latest` overlap detection breaks when banks reorder records or change dates.
  - beangulp importers are Python code per institution, with no declarative option in core.
  - smart_importer needs a large, varied history and gives no confidence feedback in the UI.
  - beancount-import is powerful but heavy: a web app, custom sources and an older release.
  - ledger-autosync depends on OFX Direct Connect, which is disappearing at banks (an inference; see Gaps).
  - Paisa's PDF import is experimental.
- The clear gap is that no mainstream PTA tool combines all of these: (a) declarative templates for CSV, XLSX and PDF, (b) a verified closing balance, (c) persistent IDs plus fuzzy dedup, (d) learned or LLM suggestions that can be turned into rules, and (e) transfer and card-payment matching.

### Gaps
- I did not fetch hledger 1.50, 1.51 and 1.52 release dates. The manual only confirms which features were added in which version.
- Paisa's `predictAccount` algorithm (TF-IDF, BM25 or something else) is not documented on the import page. I did not verify it in the source.
- Whether beancount-import 1.4.0 fully supports Beancount v3 was not confirmed from primary sources.
- No published accuracy figures were found for smart_importer, reckon, Paisa `predictAccount` or beancount-import's classifier.

## Q2. Python libraries for OFX/QFX, QIF, CSV sniffing and XLSX

### Takeaway
Use ofxtools for OFX/QFX: it is actively maintained, has no dependencies and targets OFX 1.6 and 2.03. ofxparse is stale (2021). Use quiffen for QIF, CleverCSV for dialect detection on messy bank CSVs, and calamine (through python-calamine, fastexcel, or Polars' default `read_excel` engine) for fast XLSX/XLS/ODS reading. openpyxl is only needed for writing or styling.

### Cited Findings
- **ofxtools** 1.1.1 (2026-06-12), GPL-3.0-only, no dependencies, 345 stars. It "requests, consumes and produces both OFXv1 (SGML) and OFXv2 (XML)". It targets OFX 1.6 and 2.03 and "handles Quicken's QFX format, although it ignores Intuit's proprietary extension tags". It covers banking, investments, bill pay and the 1099 tax extensions, and includes an OFX client for downloading — [PyPI ofxtools](https://pypi.org/project/ofxtools/), [csingley/ofxtools](https://github.com/csingley/ofxtools)
- **ofxparse** 0.21 was last released 2021-05-31 (older). It is MIT, has 219 stars and 43 open issues — [PyPI ofxparse](https://pypi.org/project/ofxparse/), [jseutter/ofxparse](https://github.com/jseutter/ofxparse)
- **ofxstatement** 0.9.3 (2025-09-10) is a "tool to convert proprietary bank statement to OFX format". Bank-specific support comes through a plugin ecosystem — [PyPI ofxstatement](https://pypi.org/project/ofxstatement/)
- **QIF.** quiffen 4.0.1 (2025-12-31) is MIT. The alternative qifparse 0.5 dates from 2013 (older, GPL) — [PyPI quiffen](https://pypi.org/project/quiffen/), [PyPI qifparse](https://pypi.org/project/qifparse/)
- **CleverCSV** 0.8.5 (2026-05-11, MIT, Alan Turing Institute) is a drop-in `Sniffer` replacement. Its authors report "97% accuracy for dialect detection, with a 21% improvement on non-standard (messy) CSV files compared to" Python's `csv.Sniffer`. This comes from their 2019 Data Mining and Knowledge Discovery paper (older). Detection got much faster in v0.8.0 — [PyPI clevercsv](https://pypi.org/project/clevercsv/), [paper PDF](https://gertjanvandenburg.com/papers/VandenBurg_Nazabal_Sutton_-_Wrangling_Messy_CSV_Files_by_Detecting_Row_and_Type_Patterns_2019.pdf)
- **XLSX.**
  - Polars `read_excel` now defaults to `engine="calamine"` (previously `xlsx2csv`). It can also use `openpyxl` or `xlsx2csv` — [Polars read_excel docs](https://docs.pola.rs/api/python/stable/reference/api/polars.read_excel.html)
  - polars is at 1.44.2 (2026-09-09), python-calamine at 0.8.2 (2026-07-13, MIT) and fastexcel at 0.21.0 (2026-08-19) — [PyPI polars](https://pypi.org/project/polars/), [PyPI python-calamine](https://pypi.org/project/python-calamine/), [PyPI fastexcel](https://pypi.org/project/fastexcel/)
  - openpyxl's last release is 3.1.5 (2024-06-28, older) — [PyPI openpyxl](https://pypi.org/project/openpyxl/)
- **Beyond these formats.** bankstatementparser (Python, Apache-2.0/MIT) also parses CAMT.053 (ISO 20022), PAIN.001 and MT940 as well as CSV and OFX/QFX. It reports "27,000+ tx/s" on CAMT — [sebastienrousseau/bankstatementparser](https://github.com/sebastienrousseau/bankstatementparser)

### Inferences
- A new system can cover the structured formats deterministically at no cost:
  - ofxtools for OFX/QFX; take care with ofxtools' GPL-3.0 if the new project wants a permissive license.
  - quiffen for QIF.
  - CleverCSV, then the stdlib `csv` module, for CSV.
  - python-calamine or fastexcel for XLSX/ODS.
  - A CAMT or MT940 parser for European banks.
- Automatic CSV detection should do three things: detect the dialect, detect the encoding (`encoding` is a common need, as hledger's `encoding` rule shows), and guess the date format, decimal mark and money columns (the reckon-style heuristics). The result should be saved as a reusable declarative profile, like an hledger `.rules` file.

### Gaps
- I found no recent independent benchmark comparing python-calamine, openpyxl and fastexcel on bank-sized files. They are all small, so speed probably doesn't matter here.
- I found no survey of how many banks still expose OFX Direct Connect in 2026.

## Q3. PDF extraction for bank statements and credit-card bills

### Takeaway
PDF statements are the hardest input and the biggest opportunity. For born-digital statements, the text layer (pdfplumber, pdftotext/Poppler, PyMuPDF, pypdf) plus per-bank templates or regexes still gives the most reliable results. monopoly and the FinNLP 2026 auditing pipeline both take this approach. Heavy layout and VLM parsers (Docling, Marker, MinerU) are strong on generic tables but are built for RAG and markdown output, not ledger rows. Common failure modes include wrapped descriptions, page-break continuations, debit and credit sign conventions, and column swaps. The practical safeguard every serious tool converges on is arithmetic verification: opening + credits − debits = closing, or checking the running balance row by row.

### Cited Findings

**Library landscape (versions and licenses from PyPI, 2026-09-25)**
- **pdfplumber** 0.11.10 (2026-06-15, MIT, 10,777 stars). Table finding uses `vertical_strategy` and `horizontal_strategy` set to `lines`, `lines_strict`, `text` or `explicit`, and `explicit_vertical_lines` can pin column x-coordinates. It also has `debug_tablefinder()` visual debugging, `dedupe_chars()`, and `extract_words` with bounding boxes — [pdfplumber README](https://github.com/jsvine/pdfplumber), [PyPI](https://pypi.org/project/pdfplumber/)
- **PyMuPDF** 1.28.2 (2026-08-06) is dual-licensed under AGPL-3.0 or an Artifex commercial license, with 10,770 stars — [PyPI pymupdf](https://pypi.org/project/pymupdf/)
- **pypdf** 6.19.0 (2026-09-16) is BSD-3-Clause and pure Python. **pypdfium2** 5.13.0 is BSD-3/Apache-2.0. **pdftotext** (Poppler binding) 4.0.0 (2026-06-26) is MIT — [PyPI pypdf](https://pypi.org/project/pypdf/), [PyPI pypdfium2](https://pypi.org/project/pypdfium2/), [PyPI pdftotext](https://pypi.org/project/pdftotext/)
- **camelot-py** 2.0.0 (2026-06-04, MIT, 3,823 stars) offers lattice mode (ruled lines) and stream mode (whitespace). **tabula-py** 2.10.0 (2024-10-17, older) wraps tabula-java, so it needs a JRE — [PyPI camelot-py](https://pypi.org/project/camelot-py/), [camelot docs](https://camelot-py.readthedocs.io/), [PyPI tabula-py](https://pypi.org/project/tabula-py/)
- **Docling** 2.130.0 (2026-09-22) is MIT and hosted by the LF AI & Data Foundation, with 67,912 stars. It does layout, reading order, table structure and OCR, and reads PDF, DOCX, XLSX, HTML and more. It can also run VLMs such as GraniteDocling-258M (`--pipeline vlm`) — [PyPI docling](https://pypi.org/project/docling/), [docling-project/docling](https://github.com/docling-project/docling)
- **Marker** 2.0.0 (2026-07-20, 39,963 stars)
  - Licensing: the code is Apache-2.0. The model weights use a modified OpenRAIL-M license that is "free for research, personal use, and startups under $5M"; beyond that, commercial use needs a paid license.
  - Benchmark (vendor self-report): Marker claims 76.0% overall on olmocr-bench (1,403 PDFs) and 83.5% on born-digital PDFs, "ahead of MinerU and docling".
  - LLM mode: `--use_llm` can "merge tables across pages … and extract values from forms". It works with Gemini (default `gemini-3.5-flash`), Claude, OpenAI-compatible APIs and Ollama.
  - Table output: `TableConverter` outputs only tables, as HTML or JSON with bounding boxes. Layout, OCR and table recognition run through a Surya VLM server (vLLM or llama.cpp).
  - Sources: [PyPI marker-pdf](https://pypi.org/project/marker-pdf/), [datalab-to/marker](https://github.com/datalab-to/marker)
- **MinerU** 4.0.7 (2026-09-23, 80,631 stars) uses the "MinerU Open Source License … based on Apache 2.0 with additional conditions". Version 4.0 presents itself as a document reader for agents. Its `basic` tier runs ONNX on CPU (~0.8 GB) and its `standard` tier runs ONNX plus a llama.cpp VLM (8 GB RAM, CPU works) — [PyPI mineru](https://pypi.org/project/mineru/), [opendatalab/MinerU](https://github.com/opendatalab/MinerU)
- **unstructured** 0.27.8 (2026-09-22) is Apache-2.0 — [PyPI unstructured](https://pypi.org/project/unstructured/)
- **OCR.** OCRmyPDF 17.12.1 (2026-09-16, MPL-2.0, 34,872 stars) adds a Tesseract text layer to scanned PDFs. pytesseract 0.3.13 was released 2024-08-16 (older) — [PyPI ocrmypdf](https://pypi.org/project/ocrmypdf/), [ocrmypdf/OCRmyPDF](https://github.com/ocrmypdf/OCRmyPDF), [PyPI pytesseract](https://pypi.org/project/pytesseract/)

**Generic document-parsing benchmarks (not bank-specific)**
- OmniDocBench (CVPR 2025) covers text edit distance, table TEDS, formulas and reading order. A search summary reported MinerU 2.5 at table TEDS ≈ 88.2. I did not verify this against the leaderboard itself — [OmniDocBench](https://github.com/opendatalab/OmniDocBench)
- ParseBench (arXiv 2604.08538, 2026, LlamaIndex-associated, so vendor bias is possible) has ~2,000 human-verified enterprise pages including finance and evaluates 14 methods. LlamaParse Agentic scored highest at 84.9%. The authors conclude "no method is consistently strong across all five dimensions" — [arXiv 2604.08538](https://arxiv.org/abs/2604.08538)
- I found no single public leaderboard that ranks Docling, Marker and MinerU on the same suite (per search summaries). The Medium comparison "Docling vs Marker vs MinerU (2026)" returned HTTP 403 and was not read — [Medium (not accessible)](https://adityamangal98.medium.com/docling-vs-marker-vs-mineru-the-ultimate-open-source-pdf-parser-benchmark-2026-which-is-best-a36ecbb6c6b1)

**Bank-statement-specific tools and evidence**
- **monopoly** (monopoly-core 0.23.1, 2026-09-05; AGPL-3.0; 426 stars) converts statements to CSV using `pdftotext`/Poppler and "predefined configuration classes per bank" for 17+ banks, including Chase, BofA, HSBC, DBS, RBC and TD. It has a "safety check" that "validates totals for debit or credit statements". There is also a generic parser for unsupported banks (to be used "with appropriate caution"), optional OCR through ocrmypdf, encrypted PDF support via `PDF_PASSWORDS`, and JSON output with stable per-transaction `id`s — [benjamin-awd/monopoly](https://github.com/benjamin-awd/monopoly), [PyPI monopoly-core](https://pypi.org/project/monopoly-core/)
- **"Auditing Bank Statement PDFs: A Multi-Tier Parser …"** (accepted at FinNLP @ EMNLP 2026)
  - Pipeline: Tier 1 is a PyMuPDF grid extractor for digital pages "with clean ruling-line grids matching the expected column count". Tier 2 is LlamaParse for scanned or borderless pages. Gemini 2.5-flash handles metadata.
  - Benchmark: 23 statements from 20 banks, 75 pages and 1,672 transactions (a private auditing partner's data; only 2 mutated samples are public).
  - Repairs: page-spanning narrations are merged back into the previous transaction. A balance identity `Balance_t = Balance_{t-1} + Deposit_t − Withdrawal_t` "detects and repairs OCR artifacts (sign inversions, decimal shifts, credit/debit column swaps)".
  - I found no per-tier accuracy numbers in the repo.
  - Source: [ahemtiaz/auditing-bank-statement-pdfs](https://github.com/ahemtiaz/auditing-bank-statement-pdfs)
- **"Parsing Bank Statement PDFs: 5 Tools Compared (2026)"** (DEV, 2026-04-21) is by the author of pdftoxlsx, a disclosed vendor bias.
  - Reported accuracy: Tabula/Camelot 81%, pdftoxlsx 99.1%, Nanonets 88.7%, DocParser 85.3%, PDFTables 83.6%. The methodology is not independently verifiable.
  - Failure modes named: multi-page table continuation without repeated headers, merged header cells, mixed lattice and stream regions, debit/credit column ambiguity, and scans. "Multi-page tables are hardest", and bank-specific templates beat general parsers.
  - Source: [DEV article](https://dev.to/urios/parsing-bank-statement-pdfs-5-tools-compared-for-developers-2026-4b70)
- **Brazilian credit-card bills (faturas)**, only briefly since another researcher covers Brazil:
  - importaFatura (Inter, Mercado Pago) extracts "transações, valores, datas, parcelas", keeps separate summaries per card (primary and additional), and validates "o total declarado e a soma das transações" (the stated total against the sum of transactions). It blocks export when they disagree unless `--allow-warnings` is passed — [cranzss/importaFatura](https://github.com/cranzss/importaFatura)
  - turicas/nubank-to-csv converts the Nubank PDF to HTML and parses it, "already summing IOF for each item" (from a search snippet) — [turicas/nubank-to-csv](https://github.com/turicas/nubank-to-csv)
- **Paisa**'s PDF import (pdf.js-based) is "experimental … may not accurately detect rows" — [Paisa docs](https://paisa.fyi/reference/import/)

### Inferences
- **Recommended PDF pipeline:**
  1. Detect whether the page has a text layer. If it doesn't, run OCRmyPDF/Tesseract or a VLM.
  2. Extract words with coordinates (pdfplumber `extract_words` or PyMuPDF). Keep pypdf/pypdfium2 as permissive-license options, since PyMuPDF is AGPL.
  3. Apply a declarative per-issuer template: header and footer anchors, column x-ranges (pdfplumber `explicit_vertical_lines`), a row-start regex (a line starts with a date), a continuation-line merge rule (lines without a date or amount join the previous row, including across pages), a sign rule (a `C`/`CR`/`-` suffix, parentheses, or separate columns), and an installment regex such as `(\d{1,2})/(\d{1,2})` giving installment n of N.
  4. Validate against the statement totals or the running balance, and refuse (or flag) on mismatch.
  5. Fall back to an LLM or VLM only when no template matches, then persist a generated template for next time.
- Docling, Marker and MinerU are best used as a fallback "layout to table" step. They add GPU or model weight, and Marker's weights and MinerU carry license conditions. They are not ledger-aware: they don't merge wrapped rows semantically or apply sign conventions, although Marker's `--use_llm` merges tables across pages.
- Multi-currency lines on card bills (original currency amount, exchange rate, amount in local currency, IOF) need a template that can capture several amount fields per row. None of the general PTA importers handles this out of the box.

### Gaps
- I found no independent, open benchmark of open-source tools (pdfplumber, camelot, Docling, Marker, MinerU, VLMs) on real bank and credit-card statements with row-level accuracy. The only numbers are vendor posts, or research with private data and no per-tier numbers.
- I did not fetch Camelot 2.0.0's changelog; the HISTORY.md URL returned 404, so the major-version changes are unknown.
- Installment ("parcela 3/10") handling is documented only in small Brazilian projects. I found no PTA tool that models future installments as liabilities.

## Q4. LLM-based extraction and categorization for PTA: projects, accuracy, privacy, patterns

### Takeaway
Most LLM use in PTA as of 2026 is either (a) an optional fallback categorizer after rules and ML (beanborg, actual-ai for Actual Budget, hledger-tools), or (b) agentic use of Claude Code, Codex or MCP servers over the ledger. Published accuracy is sobering:
- GPT-4o zero-shot reached 60.4% on UK SME transaction categories, against 73.4% for a fine-tuned, calibrated model.
- A vendor reports that LLM statement extraction reconciles on the first pass only about 85% of the time, and that the failures "look just as clean as the passing ones".

The patterns that work are to constrain the output with a schema, verify arithmetic, use the user's own ledger as few-shot or kNN context, mark guesses for review, and turn accepted guesses into deterministic rules.

### Cited Findings

**Projects**
- beanborg: rules first, then an ML model, then optional ChatGPT (`rules.use_llm: true`) — [beanborg](https://github.com/luciano-fiandesio/beanborg)
- hledger-tools (dgzlopes; 1 star) has an `import` command that sends "your account names and source file (with raw transactions)" to OpenAI and gets hledger journal entries back. It supports `--show-prompt` and `--context`. `--include-journal` is possible "but you probably shouldn't" — [dgzlopes/hledger-tools](https://github.com/dgzlopes/hledger-tools)
- actual-ai (for Actual Budget, not PTA; 525 stars, MIT) supports OpenAI, Anthropic, Google, Ollama, Groq, OpenRouter and any OpenAI-compatible API. Its prompts are Handlebars templates. It marks every guessed transaction "as guessed in notes, so you can review", can "suggest and create new categories", and can do optional web search on unknown merchants (ValueSerp). Unclassifiable transactions are marked "not guessed" and skipped on later runs — [sakowicz/actual-ai](https://github.com/sakowicz/actual-ai)
- firefly-iii-ai-categorize (OpenAI, 220 stars) is now archived — [bahuma20/firefly-iii-ai-categorize](https://github.com/bahuma20/firefly-iii-ai-categorize)
- bankstatementparser routes by input type:
  - Structured formats go to deterministic parsers ("$0, fastest").
  - Digital PDFs get pypdf text (or pdfplumber) plus an LLM through litellm (default `ollama/llama3`).
  - Scans go to a vision model (default `ollama/minicpm-v`).
  - It then runs a "Golden Rule" check ("Opening + credits − debits == closing"), grouped per currency since v0.0.8, and a continuity check across statements. PII is masked in console output. It includes a Plaid-13-category categorizer. No LLM accuracy figures are published.
  - Source: [bankstatementparser](https://github.com/sebastienrousseau/bankstatementparser)
- statement-extractor (a personal demo on synthetic data) sends digital PDFs through the pdfplumber text layer and only scans to Claude vision, capped at 8 pages. It validates with Pydantic, which rejects rather than coerces. It checks "opening + credits + debits must equal the reported closing, within one cent" and exits with code 1 on failure: "Extraction you cannot verify is not automation, it is a new source of errors" — [solijon-ai/statement-extractor](https://github.com/solijon-ai/statement-extractor)
- The AI section of plaintextaccounting.org (2026) lists accountant24 (a local-first multi-model agent on hledger), a Claude Code + hledger + Obsidian setup, Countbean (a Claude Code plugin and MCP server for Beancount), Beanhand (imports receipt scans or photos as transactions and links them to existing ones) and Finzytrack (a GUI with optional local or cloud AI) — [plaintextaccounting.org](https://plaintextaccounting.org/)
- Awesome Beancount lists a Beancount Telegram Bot "powered with llm", beancount-gs (a self-hosted web service with an AI assistant and MCP), beanquery-mcp, and Cocono (an iOS client with on-device AI) — [awesome-beancount.com](https://awesome-beancount.com/)
- GitHub search results from 2026 include:
  - dhr2333/Beancount-Trans (79 stars): upload bills and get auditable Beancount entries.
  - finkit: a Beancount v3 MCP server and CLI with CSV/PDF import.
  - geofftang/programmatic-finance-tracker: SimpleFIN sync, "deterministic categorization", and a ledger an LLM can query.
  - alerque/acceptarium: receipt scanning into PTA.
  - Source: GitHub repository search, 2026-09-25 ([Beancount-Trans](https://github.com/dhr2333/Beancount-Trans), [finkit](https://github.com/ankurawl/finkit), [programmatic-finance-tracker](https://github.com/geofftang/programmatic-finance-tracker), [acceptarium](https://github.com/alerque/acceptarium))
- A practitioner report (2026-01-10) used Claude Code v2.1.3 with Opus 4.5 on Mercury and Stripe CSVs. The model categorized, reconciled and produced reports, and the ending cash matched independently computed balances. The author had abandoned building a custom double-entry CLI and concluded "I should have allowed Claude to be Claude!" — [Sean Moriarity, "Claude is my accountant now"](https://seanmoriarity.com/2026/01/10/claude-is-my-accountant/)

**Accuracy evidence**
- UK SME bank transactions (arXiv 2508.05425, Aug 2025): 11 categories and 2,984–9,884 transactions per SME. Results: GPT-4o zero-shot 60.4%, TF-IDF + Random Forest 50.0%, fine-tuned FinBERT 68.0%, and fine-tuned plus calibrated FinBERT 73.4% ± 8.1%. "High-confidence predictions reached 90.36% accuracy", which the authors say makes semi-automated review workflows viable — [arXiv 2508.05425](https://arxiv.org/html/2508.05425v1)
- QuickBooks Rel-Cat (arXiv 2506.09234): few-shot top-1 68.67% and top-5 88.04% for transaction categorization. This came from a search snippet; I did not read the full paper — [arXiv 2506.09234](https://arxiv.org/pdf/2506.09234)
- LLM statement extraction: across Gemini 2.5 Flash, Claude Haiku 4.5 and Claude Sonnet 4.6 on real anonymized statements, first-pass reconciliation was "approximately 85%", and "the remaining 15% look just as clean as the passing ones". The failure modes were sign errors (especially "Payments and Credits" sections on card bills), rows dropped or duplicated at page breaks, and models "correcting" genuine rounding to force reconciliation. The author sells BankPDFtoXLS (vendor bias), and the sample size is not stated — [DEV, "Why ChatGPT will silently lie about your bank statement"](https://dev.to/poskono_99425db14ac67774/why-chatgpt-will-silently-lie-about-your-bank-statement-and-how-to-catch-it-13mb)

**Recommended patterns and privacy**
- Beancount.io's guidance is for the LLM to suggest categories within explicit account limits, "with room for uncertainty". A human then reviews against source documents. Then "Turn repeated, confirmed mappings into explicit import rules" and validate with bean-check. On privacy: "A local ledger can still send data to a cloud model through an editor or MCP client" — [beancount.io AI guide](https://beancount.io/docs/Solutions/using-llms-to-automate-and-enhance-bookkeeping-with-beancount)
- Local options:
  - bankstatementparser defaults to Ollama models — [bankstatementparser](https://github.com/sebastienrousseau/bankstatementparser)
  - Marker `--use_llm` supports Ollama — [marker](https://github.com/datalab-to/marker)
  - MinerU's VLM runs on llama.cpp on CPU — [MinerU](https://github.com/opendatalab/MinerU)
  - Docling can use the 258M-parameter GraniteDocling VLM — [docling](https://github.com/docling-project/docling)

### Inferences
- Recommended design, "LLM proposes, deterministic rules persist":
  1. Run exact-ID and rule matches first.
  2. Then run kNN or classifier suggestions trained on the user's own ledger (as smart_importer, reckon and Paisa `predictAccount` do).
  3. Only then call an LLM, with the closed list of accounts, a few similar past transactions as few-shot context, JSON-schema output (account, payee, confidence, rationale) and a "don't know" option.
  4. Mark the result as a guess (metadata such as `guessed: llm`).
  5. When the user accepts, write a deterministic rule (regex to account) so the same merchant never reaches the LLM again.
  - This keeps costs and privacy exposure small and shrinking over time.
- For extraction, an LLM should never be the source of truth for amounts without an arithmetic gate. A good hybrid pulls amounts and dates deterministically from the text layer and uses the LLM for layout inference or template generation, row-to-column mapping, and merchant normalization. Row-count and sum checks go against the statement.
- The ~60% zero-shot figure versus ~90% on high-confidence predictions suggests showing confidence and auto-accepting only above a calibrated threshold. This is an inference from the SME paper's results.

### Gaps
- I found no rigorous public benchmark of LLM categorization for personal transactions that uses the user's own ledger history as few-shot context, which is the case that matters for PTA. The available numbers are SME, business or vendor figures.
- I found no GitHub project named "beancount-gpt" (it is not on PyPI either). The widely known LLM PTA projects are small (≤100 stars), apart from Actual/Firefly add-ons outside PTA.
- I did not verify accuracy of local models (e.g. Llama or Qwen through Ollama) on transaction categorization against cloud models.

## Q5. Deduplication and reconciliation patterns

### Takeaway
The robust approach has three layers:
1. Persistent IDs stored in the ledger: the bank FITID or transaction ID, else a content hash that includes the occurrence index within the day.
2. Fuzzy similarity within a date window, to catch manual entries and pending-to-posted changes.
3. Arithmetic gates: statement opening/closing balances, running balances and bill totals, emitted as balance assertions.

Transfer pairing across one's own accounts and matching card payments to bills are handled only by beancount-import, limabean-harvest and custom scripts.

### Cited Findings
- **ID-based dedup.**
  - ledger-autosync stores `ofxid` (FITID) or `csvid` and asks ledger or hledger whether it exists — [ledger-autosync](https://gitlab.com/egh/ledger-autosync/-/raw/master/README.rst)
  - beanhub-import uses `import-id` metadata for idempotency — [beanhub-import docs](https://beanhub-import-docs.beanhub.io/)
  - monopoly emits stable per-transaction `id`s — [monopoly](https://github.com/benjamin-awd/monopoly)
- **Content hashes.**
  - ledgerbridge prefers the bank ID (`b:`) and falls back to `h:` SHA-256 over date, amount, normalized description and occurrence count within the day, which "survives re-exports" — [ledgerbridge](https://github.com/MugenLab/ledgerbridge)
  - bankstatementparser uses an MD5 of `date | normalized_description | amount` plus "suspected matches" scored by similarity thresholds with audit reasons — [bankstatementparser](https://github.com/sebastienrousseau/bankstatementparser)
  - beanborg uses an MD5 of the CSV row plus a same-date, same-amount, same-account fallback — [beanborg](https://github.com/luciano-fiandesio/beanborg)
  - icsv2ledger uses `--skip-dupes` with an MD5 of formatted values — [icsv2ledger](https://github.com/quentinsf/icsv2ledger)
- **Fuzzy matching.** beangulp's `find_similar_entries` uses a ±2-day window, a 5% amount tolerance on the same accounts, and an account-set subset test — [beangulp similar.py](https://github.com/beancount/beangulp/blob/master/beangulp/similar.py). beancount-import fuzzily associates imported rows with existing entries and "robustly associates imported transactions with the source data, to automatically avoid duplicates" — [beancount-import](https://github.com/jbms/beancount-import)
- **Date-watermark overlap.** hledger's `.latest.FILE` approach is intentionally not called dedup and has explicit assumptions (see Q1) — [hledger manual](https://hledger.org/hledger.html)
- **Pitfalls of hashing without IDs.** ledger-autosync's own caveat is that row-hash dedup produces false negatives, where transactions look new but are old — [ledger-autosync](https://gitlab.com/egh/ledger-autosync/-/raw/master/README.rst)
- **Transfers between own accounts.**
  - beancount-import merges correlated postings from independent sources — [beancount-import](https://github.com/jbms/beancount-import)
  - limabean-harvest pairs transactions "between accounts where both accounts are imported in the same group" — [limabean-harvest](https://github.com/tesujimath/limabean-harvest)
- **Statement balance checks.**
  - ledger-autosync `--assertions` emits balance assertions from OFX balances — [ledger-autosync](https://gitlab.com/egh/ledger-autosync/-/raw/master/README.rst)
  - hledger CSV rules can generate balance assertions from a balance column (`balance-type` rule) — [hledger manual](https://hledger.org/hledger.html)
  - ledgerbridge, bankstatementparser (per currency), monopoly ("safety check"), statement-extractor and importaFatura all refuse or flag output when totals don't reconcile — [ledgerbridge](https://github.com/MugenLab/ledgerbridge), [bankstatementparser](https://github.com/sebastienrousseau/bankstatementparser), [monopoly](https://github.com/benjamin-awd/monopoly), [statement-extractor](https://github.com/solijon-ai/statement-extractor), [importaFatura](https://github.com/cranzss/importaFatura)
  - The FinNLP 2026 pipeline uses the running-balance identity to repair sign inversions, decimal shifts and column swaps — [auditing-bank-statement-pdfs](https://github.com/ahemtiaz/auditing-bank-statement-pdfs)

### Inferences
- Suggested dedup key design:
  1. Use the institution's ID when present (OFX FITID, API transaction_id).
  2. Otherwise use `hash(account, posted_date, amount, normalized_description, seq_within_day)`, stored as metadata on the posting (not the transaction), so that a transfer imported from both sides keeps both source IDs.
  3. Always run a fuzzy pass (±N days, exact amount, description similarity) against un-sourced manual entries, and ask the user rather than silently skipping.
- **Card payment matching.** A payment out of checking (Assets → Liabilities:Card) should be matched to the card statement's "payment received" line by amount and a ±5-day window, then merged into one transaction. The bill's "total due" at closing becomes a balance assertion on Liabilities:Card at the closing date. Previous balance + purchases + fees/interest − payments/credits must equal the total. This is an inference combining the patterns above; no mainstream PTA tool does it specifically for card bills.
- **Pending versus posted and changed descriptions.** Keep the provisional ID and let a later import upgrade the entry (beancount-import's "cleared" concept) rather than duplicating it.

### Gaps
- I found no quantitative comparison of dedup strategies (false positive and false negative rates) in PTA contexts.
- beancount-import's exact matching tolerance (the fetch summary mentioned a 5-day window) was not verified in its source code.

## Q6. Automatic bank data access (Plaid, GoCardless/Nordigen, SimpleFIN, Teller) and PTA integration

### Takeaway
The free EU option (GoCardless/Nordigen Bank Account Data) closed to new signups in 2025. In the US, individuals mostly use SimpleFIN Bridge (about $15/year), Teller (a free developer tier) or Plaid through community scripts. PTA integration is script-level: hledger `source |` pipes with simplefin scripts, plaid2text, finfetch, beancount-gocardless and beancount-openbanking-io. There is no built-in first-class sync in any PTA tool. Brazil's Open Finance is covered by another researcher.

### Cited Findings
- GoCardless says: "New signups for Bank Account Data are currently disabled" — [GoCardless](https://bankaccountdata.gocardless.com/new-signups-disabled). A search summary of openbankingtracker says this has been the case "from July 2025 onwards" and that existing accounts keep working. I did not verify the date on a primary page; the openbankingtracker page returned HTTP 429 — [openbankingtracker (search snippet)](https://www.openbankingtracker.com/guides/free-open-banking-apis)
- A July 2026 DEV article by an author affiliated with open-banking.io (disclosed) lists alternatives: open-banking.io, Tink, Plaid, Yapily and Enable Banking. It notes that an eIDAS qualified certificate "costs €2,000–10,000 per year", which most providers require — [DEV, GoCardless alternatives](https://dev.to/johnfrandsen/gocardless-bank-account-data-alternatives-what-to-use-when-signups-are-disabled-326d)
- Per the openbankingtracker search snippets: SimpleFIN Bridge costs about $15/year and suits read-only personal finance with daily refresh limits, and Teller's developer tier gives "100 free live connections". Enable Banking and Yapily are suggested for new EU/UK builds — [openbankingtracker (search snippet)](https://www.openbankingtracker.com/guides/free-open-banking-apis)
- hledger describes SimpleFIN as "a small-developer-friendly aggregator of US bank data - like Plaid or Yodlee, but simpler and cheaper". It is read-only. Scripts `simplefinsetup`, `simplefinjson` and `simplefincsv` feed rules through `source | simplefincsv data/simplefin.json 'my bank name.*checking'`, refreshed "every day or as needed" — [hledger.org/simplefin](https://hledger.org/simplefin.html)
- Community integrations:
  - beancount-gocardless 0.1.14 (2026-02-04) — [PyPI](https://pypi.org/project/beancount-gocardless/)
  - beancount-openbanking-io, a beangulp importer over PSD2 "decrypted client-side; no eIDAS certificate" — [plaintextaccounting.org](https://plaintextaccounting.org/)
  - finfetch, which uses Plaid, and plaid2text (Plaid to ledger/beancount) — [plaintextaccounting.org](https://plaintextaccounting.org/)
  - programmatic-finance-tracker (SimpleFIN plus deterministic categorization) — [GitHub](https://github.com/geofftang/programmatic-finance-tracker)
- beanhub-extract ships a `plaid` extractor, and beanhub-import can consume it — [beanhub-import docs](https://beanhub-import-docs.beanhub.io/)

### Inferences
- A new system should treat aggregator or API data as one more extractor producing the same canonical records, with bank-supplied transaction IDs, so that the same rules, dedup and balance-assertion machinery applies. hledger's `source | command` design already shows how to plug in fetchers without coupling them to the core.
- Because aggregator availability changes (the GoCardless closure being the example), file-based import (CSV, OFX and especially PDF, which every bank provides) remains the universal fallback. That makes making PDF import easy and verified the highest-leverage feature. The same holds for Brazilian card bills, where PDF is the norm; see the Brazil researcher's notes for Open Finance.

### Gaps
- I did not verify current Plaid pricing or free-tier terms for individuals in 2026 from primary Plaid pages.
- I did not fetch SimpleFIN's or Teller's official pricing pages; those figures come from a third-party tracker's search snippets.
- I did not fetch the Actual Budget GoCardless/SimpleFIN docs, which show how non-PTA open-source apps integrate.
