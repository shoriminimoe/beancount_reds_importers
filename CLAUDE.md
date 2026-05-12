# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

This project is mid-transition from Beancount v2 to Beangulp (Beancount v3's importer framework). The `main` branch / head targets v3/beangulp; the v2-compatible release is pinned at version 0.10.0. Assume new code targets v3 unless context says otherwise.

## Common Commands

This project uses `uv` for dependency management.

```bash
uv sync --all-extras --dev          # install deps incl. dev extras
uv run pytest                       # run full test suite
uv run pytest path/to/test_file.py  # run a single test file
uv run pytest -k <expr>             # filter tests by name
uv run pytest --generate            # (re)generate regression expected outputs (see below)
uv run ruff check . --statistics    # lint
uv run ruff format                  # format (and `--check` in CI)
uv run isort --profile black .      # import sort (and `--check` in CI)
```

CI (`.github/workflows/pythonpackage.yml`) runs ruff check, pytest, ruff format --check, and isort --check on Python 3.11 and 3.12. `pyproject.toml` excludes `beancount_reds_importers/importers/mercurycards` from pytest collection.

## Architecture

The codebase is a framework for writing Beancount importers, deliberately split into three layers so that institution-specific code stays minimal. Understanding this separation is essential before editing any importer.

### Three-layer design

1. **Readers** (`beancount_reds_importers/libreader/`) — file-format handling. One class per format: `ofxreader`, `csvreader`, `csv_multitable_reader`, `xlsxreader`, `xlsx_multitable_reader`, `xlsreader`, `pdfreader`, `jsonreader`, `xmlreader`, `tsvreader`. All inherit `reader.Reader`, which implements the `identify()` flow (extension check → `custom_init()` → `filename_pattern` regex → `initialize_reader()` → sets `self.reader_ready`).
2. **Transaction builders** (`beancount_reds_importers/libtransactionbuilder/`) — convert raw rows/records into Beancount directives. Three builders: `banking` (banks + credit cards), `investments` (brokerage incl. retirement; handles sweep funds, money market, all standard brokerage tx types), `paycheck` (single multi-posting entries). All inherit `transactionbuilder.TransactionBuilder`, which provides `set_config_variables` (`{currency}`/`{ticker}` substitution into account templates), `build_metadata`, and hooks like `skip_transaction`, `add_custom_postings`, `custom_entry_mods`, `get_tags`, `get_links` for importers to override.
3. **Importers** (`beancount_reds_importers/importers/<institution>/`) — institution-specific declarations. Each is typically tiny: a class that multiply-inherits one transaction builder + one reader, sets `IMPORTER_NAME` and `filename_pattern_def` in `custom_init()`, and tweaks small quirks. See `importers/citi/__init__.py` for the canonical ofx example and `importers/schwab/schwab_csv_brokerage.py` for the csv equivalent.

When adding an importer, choose an existing one of similar shape and keep yours about the same length — if it grows much larger, the quirks probably belong in a reader/builder instead.

### Configuration flow

Importers are instantiated by user-side `*_config.py` files (see `beancount_reds_importers/example/`) that pass a dict of account templates, identifiers, currency, etc. The reader's `identify()` and `account()` methods read this config. `account()` contains a deliberate hack to detect calls from `smart_importer`'s predictor and return `smart_importer_hack` in that case (see `libreader/reader.py` — referenced upstream issues are in the comment).

### Balance assertions

`balance_assertion_date_type` config key controls assertion date: `smart` (default; statement end date minus `balance_assertion_date_fudge` days, or last-tx date, whichever is later), `ofx_date`, `last_transaction`, or `today`. Override `get_balance_assertion_date()` for custom behavior.

### Currency precision

Excel readers quantize to 2 decimals by default; `.xlsx` honors cell number formats when present. Set `self.currency_precision` in `custom_init()` for JPY (0), crypto (8), etc.

### Utilities (installed as entry points)

Defined in `pyproject.toml` `[project.scripts]`:
- `ofx-summarize` → `util.ofx_summarize:summarize`
- `bean-download` → `util.bean_download:cli` (multi-threaded statement downloads; `bean-download needs-update` reports staleness from the latest balance assertion)
- `reds-ibkr-flexquery-download` → `importers.ibkr.flexquery_download:flexquery_download`

### Tests

Tests live next to importers (`importers/<inst>/tests/`) and next to readers/builders (`libreader/tests/`, `libtransactionbuilder/tests/`). The repo-root `conftest.py` registers `beancount_reds_importers.util.regression_pytest` as a pytest plugin, which adds the `--generate` flag used to (re)create expected-output golden files for the regression harness. Run `pytest --generate` once to materialize expected outputs, then `pytest` to verify.

## Conventions

- Conventional Commits for commit messages (enforced via `.github/workflows/conventionalcommits.yml`); new importers are typically `feat:`.
- Importer directory layout (from CONTRIBUTING.md):
  ```
  importers/<inst>/
    __init__.py                   # default importer
    <inst>_<variant>.py           # secondary importers for the same institution
    <inst>_<variant>_examples/    # sample inputs + .import config + run_test.bash
    tests/
  ```
- `payee`/`narration` may appear swapped depending on institution — swap in `custom_init()` if needed (see `libtransactionbuilder/banking.py`).
- Ruff line length 99, double quotes; isort uses the black profile.
