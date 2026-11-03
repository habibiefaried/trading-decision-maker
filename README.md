# trading-decision-maker

Daily LONG / SHORT / HOLD decisions for US stocks & ETFs, with explicit
take-profit and stop-loss levels.

> **Status: design phase.** The architecture and reasoning live in
> [DESIGN.md](DESIGN.md). No code has been written yet — the commands below are the
> planned surface, kept here so the CLI gets built to a fixed contract. This file
> documents *usage only*; all design discussion belongs in `DESIGN.md`.

## Setup

Requires **Python 3.14** and runs entirely inside `./venv`.

```bash
brew install python@3.14
make venv
```

Or manually:

```bash
/opt/homebrew/bin/python3.14 -m venv venv
./venv/bin/python -m pip install -U pip
./venv/bin/pip install -r requirements.txt
```

Always invoke the interpreter explicitly as `./venv/bin/python`, or use `make`.

## Daily use

```bash
make daily            # fetch -> features -> decide. The once-a-day command.
```

Writes `reports/decisions_YYYY-MM-DD.md` and prints:

```
NVDA   LONG    conf 0.71   entry 183.40   TP 196.10 (+6.9%)   SL 177.05 (-3.5%)   R:R 2.0   hold <= 10d
AMD    HOLD    conf 0.48   — no edge: ADX 14 (chop), earnings in 3 days
```

## Commands

### Data

```bash
./venv/bin/python -m tdm.cli universe validate # check tickers before downloading
make fetch                                    # incremental; skips what's already cached
./venv/bin/python -m tdm.cli fetch --symbols NVDA,AMD --since 2018-11-01
./venv/bin/python -m tdm.cli fetch --dry-run  # print the fetch plan, make no HTTP calls
./venv/bin/python -m tdm.cli fetch --force --symbols NVDA   # ignore cache, re-pull
./venv/bin/python -m tdm.cli db status        # coverage, gaps, staleness per symbol
./venv/bin/python -m tdm.cli db verify        # OHLC sanity checks
```

Prices are cached in `data/market.db` (SQLite). Re-running `make fetch` the same day
makes zero network calls.

### Features & labels

```bash
make features                                 # -> data/features/<run_id>.parquet
./venv/bin/python -m tdm.cli features build --set extended
./venv/bin/python -m tdm.cli features export --format csv --out data/exports/
./venv/bin/python -m tdm.cli features export --format sqlite --table feature_long

make label                                    # triple-barrier labels
./venv/bin/python -m tdm.cli labels sweep      # compare TP/SL/horizon profiles
```

### Charts

```bash
make charts                                   # -> reports/charts/<ticker>.html
./venv/bin/python -m tdm.cli chart --ticker NVDA
./venv/bin/python -m tdm.cli chart --ticker NVDA --with-labels     # TP/SL outcomes
./venv/bin/python -m tdm.cli chart --ticker NVDA --with-decisions  # model calls
```

Interactive Plotly HTML: candlesticks with every overlay, plus linked subpanels for RSI,
MACD, ADX, Stochastic and volume. **Look at these before training** — feature bugs are
obvious on a chart and nearly invisible in a metric.

### Train & evaluate

```bash
make train                                     # purged walk-forward CV
./venv/bin/python -m tdm.cli train --model decision_tree    # readable baseline
./venv/bin/python -m tdm.cli train --model catboost         # primary
./venv/bin/python -m tdm.cli train --model rf               # required control
./venv/bin/python -m tdm.cli train --compare                # all three, same folds
./venv/bin/python -m tdm.cli explain --run-id <id>          # rules + importances

make backtest
./venv/bin/python -m tdm.cli backtest --sensitivity         # 2x costs, shifted entry, ...
./venv/bin/python -m tdm.cli backtest --shuffle-control     # random-signal baseline
```

### Decide & self-score

```bash
make decide
./venv/bin/python -m tdm.cli decide --date today
./venv/bin/python -m tdm.cli score-decisions --since 2026-01-01   # your real track record
```

## Configuration

| File | Purpose |
|---|---|
| `config/universe.txt` | **your** tickers, one per line. Benchmarks (SPY etc.) added automatically |
| `config/config.toml` | horizon, ATR multiples, costs, thresholds, position sizing |
| `config/feature_sets/*.toml` | which indicators to compute |

## Layout

```
data/        Parquet feature/label snapshots, CSV exports   (gitignored)
models/      trained artifacts + frozen thresholds          (gitignored)
reports/     charts/, equity.html, daily decisions          (gitignored)
runs.jsonl   append-only provenance log                     (gitignored)
src/tdm/     the package
tests/       see tests/test_leakage.py — the most important tests here
```

## Development

```bash
make test
make lint
```

## Disclaimer

Research and decision-support tooling. It does not place orders and it is not
financial advice. Read [DESIGN.md §12](DESIGN.md) for what results to realistically
expect before trading on anything this produces.
