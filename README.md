# trading-decision-maker

Daily LONG / SHORT / HOLD decisions for US stocks, with explicit take-profit and
stop-loss prices.

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

### A note on `tdm`

`tdm` is **t**rading **d**ecision **m**aker — the package name and the CLI. After
`make venv` it exists as `./venv/bin/tdm`. These two are equivalent:

```bash
./venv/bin/tdm fetch
./venv/bin/python -m tdm.cli fetch
```

The commands below use the second form so it's unambiguous which Python is running. If
you'd rather type less, `alias tdm="./venv/bin/tdm"` from the repo root.

## Daily use

```bash
make daily        # fetch -> features -> dataset -> decide
```

```
NVDA   LONG   EV +$19 per $100 risked     P(TP) 0.51  P(SL) 0.32  P(TIMEOUT) 0.17
  entry  ~$183.40    (indicative; actual fill is tomorrow's open)
  TP      $189.12     +3.1%   →  +$149 if hit
  SL      $179.59     −2.1%   →  −$99 if hit
  size     26 units  = $4,768 notional, $477 margin
  hold    ≤ 10 trading days, then exit at market

AMD    HOLD   EV −$12      P(TP) 0.31  P(SL) 0.38  P(TIMEOUT) 0.31
                           — below the +$10 minimum; ADX 14 (choppy)

account: 3 of 10 slots used · $1,431 of $5,000 margin · $300 currently at risk
```

## Commands

### Data

```bash
./venv/bin/python -m tdm.cli universe validate   # check tickers BEFORE downloading
make fetch                                       # download 9 years of daily OHLCV
./venv/bin/python -m tdm.cli fetch --symbols MSFT,NVDA --years 9
```

**Nothing is stored** — each run downloads and recomputes everything in memory.
`config/universe.txt` is exactly the stocks you want to trade; the market index `^GSPC`
and `^VIX` are fetched separately as reference data for a few feature columns and are
never traded.

### Features

```bash
./venv/bin/python -m tdm.cli features --set minimal    # minimal | core | extended
./venv/bin/python -m tdm.cli features --dump-csv out.csv
```

### Charts

```bash
make charts                                      # -> reports/charts/<ticker>.html
./venv/bin/python -m tdm.cli chart --ticker MSFT
./venv/bin/python -m tdm.cli chart --ticker MSFT --with-labels     # TP/SL outcomes
./venv/bin/python -m tdm.cli chart --ticker MSFT --with-decisions  # model calls
```

Interactive Plotly HTML: candlesticks with every overlay, plus linked subpanels for RSI,
MACD, ADX, Stochastic and volume. **Look at these before training** — feature bugs are
obvious on a chart and nearly invisible in a metric.

### Dataset

```bash
make dataset                                     # features + target columns
./venv/bin/python -m tdm.cli labels sweep        # compare TP/SL/horizon profiles
./venv/bin/python -m tdm.cli dataset --dump-csv out.csv    # optional, on demand
./venv/bin/python -m tdm.cli dataset --schema    # column list, dtypes, tier
```

One row per `(symbol, date)`: identity, raw OHLCV, the feature columns, `y_long` /
`y_short` (3-class: `TP` / `SL` / `TIMEOUT`), and `fwd_*` diagnostics. See DESIGN.md §8.1.

### Train & evaluate

```bash
make train                                       # purged walk-forward CV
./venv/bin/python -m tdm.cli train --model decision_tree    # readable baseline
./venv/bin/python -m tdm.cli train --model catboost         # primary
./venv/bin/python -m tdm.cli train --model rf               # required control
./venv/bin/python -m tdm.cli train --compare                # all three, same folds
./venv/bin/python -m tdm.cli train --fast                   # quick; iteration only
./venv/bin/python -m tdm.cli train --side long              # halve the work
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
| `config/universe.txt` | **your** tickers, one per line, any number. Stocks only |
| `config/config.toml` | horizon, ATR multiples, entry mode, costs, sizing, `min_ev_r` |
| `config/feature_sets/*.toml` | `minimal` / `core` / `extended` |

Full reference: DESIGN.md §14.

## Layout

```
reports/         charts/, equity.html, daily decisions            (gitignored)
decisions.jsonl  every call the system issued — never delete      (gitignored)
src/tdm/         the package
tests/           see tests/test_leakage.py — the most important tests here
```

No `data/` or `models/` directory — nothing is cached or persisted (DESIGN.md §5).

## Development

```bash
make test
make lint
```

## Disclaimer

Research and decision-support tooling. It does not place orders and it is not financial
advice. Read [DESIGN.md §12](DESIGN.md) for what results to realistically expect before
trading on anything this produces.
