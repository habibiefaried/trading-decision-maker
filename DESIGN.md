# Trading Decision Maker — Design

**Status:** design only, no code yet.
**Audience:** software engineers. No finance background assumed — §2 is a glossary that
defines every domain term used in this document, in programming terms.
**Date:** 2026-11-03

---

## 1. What the system does

A batch pipeline that runs once a day. Input: daily price history for a list of stock
tickers. Output: one row per ticker telling you to buy, sell-short, or do nothing, plus
the two price levels at which you'd exit.

```
$ make daily

NVDA   LONG    conf 0.71   entry 183.40   TP 196.10 (+6.9%)   SL 177.05 (-3.5%)   hold <= 10d
AMD    HOLD    conf 0.48   — no edge: ADX 14 (choppy), earnings in 3 days
TSLA   SHORT   conf 0.66   entry 402.10   TP 377.90 (-6.0%)   SL 414.20 (+3.0%)   hold <= 10d
```

Read that as: *"bet NVDA goes up. Exit with profit at 196.10, or cut the loss at 177.05,
whichever happens first. Give up after 10 days either way."*

### As a system, it's four stages

| Stage | What it is, in engineering terms | Cost to run | Idempotent? |
|---|---|---|---|
| **1. Ingest** | Pull rows from a flaky, rate-limited third-party HTTP API into local SQLite | **Expensive** (network, rate limits) | Yes — and must *skip* work already done |
| **2. Features** | `map` a few hundred pure functions over a time series to build a wide numeric matrix | Cheap (seconds) | Yes — pure, fully recomputable |
| **3. Train** | Generate labels by simulation, then fit a classifier | Minutes | Yes, given a fixed seed |
| **4. Decide** | Load model, run inference on today's row, write output | Milliseconds | Yes |

The architecture follows from that cost asymmetry: **stage 1 output is precious and gets
a durable transactional store; stages 2–4 are disposable and get recomputed freely.**

### Three things make this harder than a normal ETL pipeline

Worth stating up front, because they drive most of the design decisions:

1. **The upstream data is retroactively mutable.** Rows you fetched last year can change
   meaning this year (§6). This is a cache-invalidation problem, and it's why the schema
   looks the way it does.
2. **The correctness bug you cannot see.** The whole system predicts the future from the
   past. If any code path accidentally lets future data reach the model, everything still
   runs, all tests pass, metrics look *great*, and the system is worthless. There's no
   stack trace. §10 is entirely about this, and it's the most important section here.
3. **Two implementations of the same logic must agree.** The training labels are produced
   by a simulator, and production inference runs a separate code path. If they drift,
   you've trained on a world that doesn't match the one you deploy into (§9, §11).

### Scope (locked in with the user)

| Decision | Choice | Notes |
|---|---|---|
| **Where we actually trade** | **IQ Option, stock CFDs, 10× leverage** | User has traded there for years. This is a *derivative*, not share ownership — it changes the cost model and removes a regulatory constraint. **§11.1** |
| What we trade | **US stocks only — no ETFs** | SPY is still *read* as a benchmark for features, but never traded (§8.2) |
| Universe | **user-supplied ticker list** in `config/universe.txt` | You provide the symbols, the system downloads them. No hardcoded list, no intersection logic — §7 |
| Data source | `yfinance` (free Yahoo Finance API wrapper) | Prices come from Yahoo; execution happens at IQ Option. These differ — §11.1 |
| Granularity | One row per trading day | |
| History depth | **8 years** (~2,016 bars/ticker) | Covers the 2018 correction, the 2020 COVID crash, and the 2022 bear market — §10.1 |
| Style | **Swing trading** | Hold across days, not intraday, not months |
| Horizon | **10 trading days max** | This is the timeout barrier — explained in §9 |
| TP / SL | **2.0 × ATR / 1.0 × ATR** | |
| Both directions? | **Yes.** CFDs make shorting trivial | §11.1 |
| Weekly-timeframe features | **Dropped** | User's call; removes a whole class of leakage bug |
| Earnings-date data | **Optional, default off** | OHLCV-only is the v1 target. ⚠️ But see the leverage warning in §11.1 |
| Python | 3.15 | |

---

## 2. Glossary

Every domain term in this document, defined once. Skim it now, refer back as needed.

### The data

| Term | Meaning |
|---|---|
| **Ticker** / **symbol** | A string ID for a stock: `"NVDA"`, `"AAPL"`. The primary key of this whole system. |
| **Bar** | One row of data for one ticker for one time period. Our bars are daily. |
| **OHLCV** | The 5 numbers in a bar: **O**pen (first trade price of the day), **H**igh, **L**ow, **C**lose (last trade price), **V**olume (number of shares traded). That's the entire raw input. |
| **Session** | One trading day. The US market is open ~6.5h/day, weekdays, minus ~10 holidays/year. So "yesterday" and "the previous session" are different things, and you need a calendar library to know which. |
| **ETF** | A ticker that represents a basket of stocks. `SPY` tracks the 500 largest US companies, so it's a convenient one-number proxy for "the market". |
| **ADV** | Average Daily Volume. How much the stock trades. Low ADV = you can't buy or sell without moving the price yourself. |

### The actions

| Term | Meaning |
|---|---|
| **Long** | Buy now, hoping to sell higher later. The normal direction. You profit if price goes **up**. |
| **Short** | Borrow shares, sell them now, buy them back later to return them. You profit if price goes **down**. Requires a special account type and has extra costs/rules (§11.1). |
| **Flat** / **HOLD** | No position. Doing nothing. This is a valid and frequently correct output. |
| **Entry** | The price at which you open a position. |
| **TP** (take-profit) | A pre-set price where you close the position at a **profit**. In practice it's a standing order with the broker: "sell automatically if it reaches this." |
| **SL** (stop-loss) | A pre-set price where you close at a **loss**, to stop it getting worse. Also a standing order. |
| **Position sizing** | How many shares to buy. A separate decision from direction, and arguably more important. |

### Measuring things

| Term | Meaning |
|---|---|
| **Indicator** | A pure function over a window of past bars, returning one number per bar. `f: List[Bar] -> float`. That's all any of them are, including the ones with mystical names. §8 lists ~40. |
| **SMA / EMA** | Simple / Exponential Moving Average over N days. `EMA` weights recent days more. A low-pass filter on a noisy signal. |
| **ATR** (Average True Range) | **The most important one for us.** Roughly: the average distance between a day's high and low, over the last 14 days. It's a volatility measure in dollars. A $500 stock might have ATR=$12; a $20 stock might have ATR=$0.40. <br><br>We use ATR as a **unit of distance**, like using standard deviations instead of raw values. "Set the stop 1 ATR away" means "1 typical day's movement away", which is automatically the right size for both of those stocks. This is how the system stays comparable across tickers. |
| **Oscillator** | An indicator bounded to a fixed range (usually 0–100), measuring "how overextended is the recent move". RSI, Stochastic, MFI, CCI are all variations. Convenient because they're already normalized. |
| **Trend indicator** | Measures direction and/or strength of a sustained move. EMA slope, ADX, Aroon, Supertrend. |
| **Volatility indicator** | Measures how much the price is bouncing around. ATR, Bollinger Band width, Keltner. |
| **Volume indicator** | Combines price with trade volume. OBV, MFI, VWAP. The theory is that a price move backed by high volume is more meaningful. |

### Evaluating results

| Term | Meaning |
|---|---|
| **Return** | Percentage change. `(exit - entry) / entry`. |
| **R:R** (reward-to-risk) | Ratio of TP distance to SL distance. If TP is 2 ATR away and SL is 1 ATR away, R:R = 2.0. You win twice as much as you lose, per event. |
| **Hit rate** | Fraction of trades that hit TP instead of SL. **Analogous to precision, not accuracy.** |
| **Expectancy** | `p_win × avg_win − p_loss × avg_loss`. Expected value per trade. **This is the actual objective function.** Note the implication of R:R 2.0: you break even at a **33% hit rate**, so "67% of predictions wrong" is still profitable here. Accuracy is close to meaningless as a metric (§10.4). |
| **Base rate** | The unconditional fraction of positive labels in the dataset. Your baseline to beat. |
| **Drawdown** | Peak-to-trough decline in account value. Max drawdown is "the worst stretch you'd have had to sit through". |
| **Backtest** | A replay/simulation harness: run the strategy over historical data and see what would have happened. Trivially easy to get wrong in ways that flatter you (§12). |
| **Slippage** | The gap between the price you expected and the price you got. Real-world execution noise. |

### The hazards

| Term | Meaning |
|---|---|
| **Lookahead** / **leakage** | A bug where information from the future reaches a calculation that's supposed to only see the past. **The central correctness risk in this project.** Closest analogy: a test that passes because it reads the expected value out of the same fixture it's asserting against — green, meaningless. See §10. |
| **Survivorship bias** | Selection bias in the ticker list. If you pick today's famous companies, you picked them *because* they went up. Your backtest then "discovers" that buying stocks works. |
| **Overfitting** | Standard ML meaning. Acute here because the signal-to-noise ratio is genuinely terrible and the effective sample size is much smaller than the row count (§9). |
| **Corporate action** | An event that changes the share structure: a **split** (1 share becomes 10, price divides by 10, your holdings are worth the same) or a **dividend** (company pays out cash). These rewrite historical price series — see §6. |
| **Adjusted price** | A historical price retroactively modified to account for all splits/dividends since. The subject of the most important schema decision in this document (§6). |
| **Earnings** | A quarterly scheduled event where a company reports results. The price frequently jumps 5–15% overnight. Scheduled, known in advance, and the most common way a position blows through its stop-loss before you can react. So: a hard precondition check, not a feature to be learned (§13). |

### Things we deliberately avoid

| Term | Why it's here |
|---|---|
| **Elliott Wave**, **Dow Theory** | Two traditional chart-reading frameworks that the user asked for. They are **not formulas** — they're interpretive, and practitioners disagree on how to apply them to the same chart. §8.4 extracts measurable features from the underlying idea rather than pretending to implement them. |
| **VWAP** | Volume Weighted Average Price. The user asked for it, but the standard definition is *intraday* and mathematically degenerates to nothing useful on daily bars. §8.1 explains and substitutes a working variant. |

---

## 3. Environment: Python 3.15 + venv

### Verified package availability

A brand-new Python usually means missing binary wheels in the scientific stack, so I
checked PyPI wheel tags rather than assuming:

| Package | Version | `cp315` wheel | Role |
|---|---|---|---|
| numpy | 2.5.4 | ✅ | arrays |
| pandas | 3.1.0 | ✅ | dataframes — **major version, breaking, see below** |
| scikit-learn | 1.9.1 | ✅ | models, CV |
| TA-Lib (`ta-lib`) | 0.8.3 | ✅ | indicator library (C, Cython-wrapped). Wheels are self-contained — no `brew install ta-lib` |
| pyarrow | 26.0.1 | ✅ | Parquet I/O |
| matplotlib | 3.11.2 | ✅ | plots |
| yfinance | 1.7.0 | ✅ (pure-py) | data source |
| exchange-calendars | 4.13.2 | ✅ (pure-py) | which days the market is open |
| xgboost | 3.4.1 | ✅ | optional alternative model |
| shap | 0.54.1 | ✅ | optional model explanation |

### Two popular packages are ruled out by Python 3.15

Both would be natural choices; neither can be installed. Worth knowing before someone
spends an afternoon on it:

- **`pandas-ta`** (a popular pure-Python indicator library) **cannot install at all.** It
  hard-pins `numba==0.61.2`, and that numba version publishes wheels only up to `cp313`.
  → **Use TA-Lib**, which is faster anyway.
- **`vectorbt`** (a backtesting framework) declares `requires-python <3.15`.
  → **Write our own** (§12). We'd want to regardless — see the "two implementations must
  agree" problem in §1, which a third-party framework would make worse, not better.

### pandas 3.x is a breaking release

Code copied from the user's previous project (which used pandas 2.x) will break:

| Change | Impact |
|---|---|
| **Copy-on-Write is mandatory** | Chained assignment `df['x'][0] = v` silently does nothing or raises. Use `df.loc[...]`, or build columns functionally. Strictly an improvement, but it *will* break old code. |
| **Default datetime resolution is microsecond**, not nanosecond | Any integer arithmetic on timestamps is now off by 1000×. We avoid this entirely by storing dates as ISO `TEXT` (§5). |
| **Offset aliases `M`/`Q`/`Y` removed** | Use `ME`/`QE`/`YE`. (`W-FRI`, which we need for weekly resampling, is unaffected.) |
| **`inplace=True` returns `self`** | Previously `None`. |
| **String columns infer to `str` dtype**, not `object` | |

> ⚠️ **Unverified risk — test this first.** `yfinance` declares only `pandas>=1.3.0` with
> no upper bound, so pip will happily install it next to pandas 3.1. That is not the same
> as it *working* there. **Smoke-test the ingest path in phase 1** (§14). If it breaks,
> fall back to pinning `pandas>=2.3,<3`. Do not discover this in week three.

### venv, enforced three ways

Everything runs in `./venv`. Enforced redundantly, because on this machine bare `python3`
is pyenv 3.13.1 and `/opt/homebrew/bin/python3` is 3.14 — both wrong:

1. **`.python-version`** → `3.15`
2. **`CLAUDE.md`** → explicit instructions for AI agents working in the repo
3. **A runtime guard** in `src/tdm/__init__.py`:

```python
import sys
if sys.prefix == sys.base_prefix:
    raise RuntimeError(
        "not running inside ./venv — use ./venv/bin/python or `make`"
    )
```

That guard is the one that actually works. The other two are documentation; this one
fails loudly at import time.

> **Note for AI agents and shell scripts:** do *not* use `source venv/bin/activate` in a
> tool call. Shell state doesn't persist between calls, so activation is gone by the next
> command and you silently run on system Python. Always use the explicit path.

Setup:

```bash
brew install python@3.15                  # provides 3.15.2
/opt/homebrew/bin/python3.15 -m venv venv
./venv/bin/python -m pip install -U pip
./venv/bin/pip install -r requirements.txt
```

A `Makefile` wraps every command so the venv path is never typed by hand:

```make
PY := ./venv/bin/python

venv:       # create venv + install
fetch:      # $(PY) -m tdm.cli fetch
features:   # $(PY) -m tdm.cli features build
label:      # $(PY) -m tdm.cli labels build
train:      # $(PY) -m tdm.cli train
backtest:   # $(PY) -m tdm.cli backtest
decide:     # $(PY) -m tdm.cli decide --date today
daily:      # fetch -> features -> decide   (the once-a-day command)
test lint
```

---

## 4. Repository layout

```
trading-decision-maker/
├── .python-version              # 3.15
├── CLAUDE.md                    # rules for AI agents working here
├── DESIGN.md                    # this file — the "why"
├── README.md                    # the "how to run" only
├── Makefile
├── pyproject.toml
├── requirements.txt             # loose constraints
├── requirements.lock.txt        # pip freeze, committed, for reproducibility
│
├── config/
│   ├── universe.txt             # tickers, one per line
│   ├── config.toml              # horizons, ATR multiples, costs, thresholds
│   └── feature_sets/
│       ├── baseline.toml
│       └── extended.toml
│
├── src/tdm/
│   ├── __init__.py              # venv guard
│   ├── cli.py                   # single entrypoint, subcommands
│   ├── config.py                # pydantic-settings + config hashing
│   │
│   ├── store/                   # ── persistence layer
│   │   ├── schema.sql           #    §5
│   │   ├── migrations/          #    001_init.sql, ... forward-only
│   │   ├── db.py                #    connection, pragmas, migration runner
│   │   └── bars.py              #    upsert_bars / read_bars / coverage queries
│   │
│   ├── ingest/                  # ── the only module that touches the network
│   │   ├── provider.py          #    Provider protocol (§7)
│   │   ├── yahoo.py             #    yfinance implementation
│   │   ├── sessions.py          #    market-calendar wrapper
│   │   └── sync.py              #    incremental fetch planner (§7)
│   │
│   ├── features/                # ── pure functions, no I/O
│   │   ├── registry.py          #    name -> fn + declared warmup + deps
│   │   ├── trend.py             #    EMA stack, ADX, Aroon, Supertrend, Ichimoku
│   │   ├── momentum.py          #    RSI, MACD, Stochastic, ROC, CCI
│   │   ├── volatility.py        #    ATR, Bollinger, Keltner, Donchian
│   │   ├── volume.py            #    OBV, MFI, VWAP variants
│   │   ├── structure.py         #    ZigZag pivots -> Dow/Elliott features (§8.4)
│   │   ├── market.py            #    relative strength vs SPY, market regime
│   │   ├── calendar_feats.py    #    day-of-week, days-to-earnings, gaps
│   │   └── pipeline.py          #    orchestrate, normalize, write Parquet
│   │
│   ├── labels/
│   │   └── triple_barrier.py    #    the simulator that generates y (§9)
│   │
│   ├── model/
│   │   ├── dataset.py           #    join features+labels, sample weights
│   │   ├── cv.py                #    PurgedWalkForwardSplit (§10.3)
│   │   ├── train.py
│   │   ├── explain.py           #    rule export, permutation importance, SHAP
│   │   └── registry.py          #    artifacts + provenance
│   │
│   ├── backtest/
│   │   └── engine.py            #    §12
│   │
│   └── decide.py                #    §13 — production inference path
│
├── scripts/bootstrap.sh
├── tests/
│   ├── test_indicators.py       # vs TA-Lib / hand-computed fixtures
│   ├── test_leakage.py          # ← the most important tests in the repo (§10.5)
│   ├── test_triple_barrier.py
│   └── test_incremental_fetch.py
│
├── data/                        # GITIGNORED
│   ├── market.db                #   SQLite: bars, actions, metadata, provenance
│   ├── features/                #   Parquet
│   ├── labels/                  #   Parquet
│   └── exports/                 #   CSV, for eyeballing in Excel
├── models/                      # GITIGNORED
└── reports/                     # GITIGNORED
```

### Layering rule

```
cli  ->  decide / train / backtest  ->  features / labels  ->  store  ->  ingest
```

Dependencies point one way only. Two consequences worth enforcing in review:

- **`features/` and `labels/` do no I/O and import nothing from `ingest/`.** They're pure
  functions over DataFrames. That's what makes them unit-testable without a network or a
  database, which in turn is what makes §10.5's leakage tests possible.
- **`ingest/` is the only module that makes HTTP calls.** All network flakiness,
  retries, and rate-limit handling is contained there.

### What went wrong in the previous attempt

The user's earlier project (`~/code/stock-market-entry-signal-tpsl-specific`) had a 52 KB
`fetch_stock_data.py`, a 44 KB `train.py`, and a `data/` directory accumulating files like
`AAPL_tpsl_data_20260610.csv`, one per ticker per day. Two concrete failure modes that
this layout exists to fix:

1. **No cache.** Every run re-downloaded everything, so it self-inflicted API rate limits
   and iteration was slow. → §7.
2. **No provenance.** Given `models/xyz.pkl` there was no way to recover which features,
   which parameters, or which date range produced it. **So you couldn't meaningfully
   compare two models** — the central activity of the project. → the `*_run` tables in §5.

---

## 5. Storage layer

### What goes where, and why

| Data | Store | Reason |
|---|---|---|
| Raw daily bars | **SQLite** | Expensive to re-acquire (network, rate limits). Needs transactions and durability. |
| Corporate actions | **SQLite** | Small, critical, same reason |
| Provenance / run metadata | **SQLite** | Relational, queried ad hoc |
| Issued decisions + outcomes | **SQLite** | Append-only audit log |
| Feature matrix | **Parquet** | Wide, numeric, recomputable in seconds. See the trade-off table below. |
| Labels | **Parquet** | Same |
| Exports for Excel | **CSV** | On demand only |

The user asked for "CSV or SQLite" for features. Having looked at the access pattern,
here's the honest comparison:

| Option | Shape | Read speed | Schema churn | Verdict |
|---|---|---|---|---|
| SQLite **wide** (1 column per feature) | 120+ columns | fast | `ALTER TABLE` for every new indicator | ✗ |
| SQLite **long** (`symbol, ts, name, value`) | 3 cols, ~5× the rows | needs a pivot on read | none | ○ available as export |
| **Parquet** | wide, columnar, typed | fastest; column pruning | none | ✓ **default** |
| CSV | wide, untyped | slow; loses dtypes and NaN fidelity | none | ○ available as export |

**Reasoning:** features are *cheap and disposable* — a full rebuild over 10 tickers × 15
years is seconds of TA-Lib work. What's expensive is the network fetch. So the store that
needs to be durable and transactional is `bar_daily`, which is exactly SQLite's job. The
feature matrix is a wide block of float64 read whole into pandas — Parquet's sweet spot —
and it stays schema-free, so adding your 40th indicator is a one-line registry entry
rather than a migration. Both other formats remain available:

```bash
tdm features export --format csv    --out data/exports/
tdm features export --format sqlite --table feature_long
```

### Schema conventions

- **`STRICT` tables** throughout. SQLite's default type affinity will cheerfully store the
  string `"NaN"` in a `REAL` column, and you'd find out during training. `STRICT` needs
  SQLite 3.37+; Python 3.15 bundles 3.51. Use it.
- **Dates are ISO-8601 `TEXT`** (`'2026-11-03'`). Sidesteps the pandas 3 datetime
  resolution change, sorts lexicographically, readable in a SQL console.
- **Raw prices only.** See §6 — this is the schema decision that matters most.

```sql
PRAGMA journal_mode = WAL;        -- read while a fetch writes
PRAGMA synchronous  = NORMAL;
PRAGMA foreign_keys = ON;

-- ── Instruments ──────────────────────────────────────────────────────────────
CREATE TABLE instrument (
    symbol        TEXT PRIMARY KEY,
    name          TEXT,
    exchange      TEXT,
    currency      TEXT    NOT NULL DEFAULT 'USD',
    asset_type    TEXT    NOT NULL DEFAULT 'equity',   -- equity | etf | index
    -- Two INDEPENDENT flags. ETFs (SPY, sector ETFs) are tradeable=0 but still
    -- fetched and read, because features depend on them as benchmarks (§8.2).
    -- "Don't trade ETFs" must not become "don't download ETFs".
    tradeable     INTEGER NOT NULL DEFAULT 1,          -- 0 => data-only, never in a decision
    active        INTEGER NOT NULL DEFAULT 1,          -- 0 => delisted; KEEP the rows (§10.5)
    first_bar     TEXT,
    last_bar      TEXT,
    meta_json     TEXT,                                -- raw provider payload
    updated_at    TEXT    NOT NULL
) STRICT;

-- ── Raw daily bars: the cache that matters ───────────────────────────────────
CREATE TABLE bar_daily (
    symbol        TEXT    NOT NULL REFERENCES instrument(symbol),
    ts            TEXT    NOT NULL,         -- session date, exchange-local
    open          REAL    NOT NULL,
    high          REAL    NOT NULL,
    low           REAL    NOT NULL,
    close         REAL    NOT NULL,         -- UNADJUSTED (§6)
    volume        INTEGER NOT NULL,
    source        TEXT    NOT NULL,         -- 'yahoo'
    ingested_at   TEXT    NOT NULL,
    revision      INTEGER NOT NULL DEFAULT 1,  -- bumped if the provider restates the bar
    PRIMARY KEY (symbol, ts)
) STRICT, WITHOUT ROWID;

CREATE INDEX idx_bar_daily_ts ON bar_daily(ts);

-- ── Corporate actions: why we don't store adjusted prices ────────────────────
CREATE TABLE corporate_action (
    symbol        TEXT NOT NULL REFERENCES instrument(symbol),
    ts            TEXT NOT NULL,
    kind          TEXT NOT NULL,            -- 'dividend' | 'split' | 'capital_gain'
    value         REAL NOT NULL,            -- cash amount, or split ratio (10.0 = 10:1)
    ingested_at   TEXT NOT NULL,
    PRIMARY KEY (symbol, ts, kind)
) STRICT;

-- ── Fetch bookkeeping: "don't download it twice" lives here ───────────────────
CREATE TABLE fetch_coverage (
    symbol        TEXT NOT NULL,
    interval      TEXT NOT NULL,            -- '1d'
    source        TEXT NOT NULL,
    start_ts      TEXT NOT NULL,            -- earliest session held
    end_ts        TEXT NOT NULL,            -- latest session held
    bar_count     INTEGER NOT NULL,
    last_fetch_at TEXT NOT NULL,
    PRIMARY KEY (symbol, interval, source)
) STRICT;

-- Sessions the calendar says exist but the provider has no data for.
-- Without this table, every run re-requests the same permanent holes forever.
CREATE TABLE known_gap (
    symbol        TEXT NOT NULL,
    ts            TEXT NOT NULL,
    reason        TEXT NOT NULL,            -- 'no_data' | 'halted' | 'pre_ipo'
    checked_at    TEXT NOT NULL,
    PRIMARY KEY (symbol, ts)
) STRICT;

CREATE TABLE fetch_log (
    id            INTEGER PRIMARY KEY,
    symbol        TEXT, interval TEXT, source TEXT,
    req_start     TEXT, req_end TEXT,
    status        TEXT NOT NULL,            -- 'ok' | 'empty' | 'rate_limited' | 'error'
    rows_written  INTEGER,
    error         TEXT,
    duration_ms   INTEGER,
    at            TEXT NOT NULL
) STRICT;

-- ── Scheduled events we must not trade through (§13) ─────────────────────────
CREATE TABLE earnings_event (
    symbol        TEXT NOT NULL REFERENCES instrument(symbol),
    ts            TEXT NOT NULL,
    session       TEXT,                     -- 'bmo' (before open) | 'amc' (after close)
    confirmed     INTEGER NOT NULL DEFAULT 0,
    ingested_at   TEXT NOT NULL,            -- when WE learned it (§10.5 item 6)
    PRIMARY KEY (symbol, ts)
) STRICT;

-- ── Provenance: which code + config produced which artifact ──────────────────
-- Content-addressed: run_id = hash(config, code_version, date_range, symbols).
-- This is what the previous project lacked.
CREATE TABLE feature_run (
    run_id        TEXT PRIMARY KEY,
    feature_set   TEXT NOT NULL,
    config_json   TEXT NOT NULL,
    code_version  TEXT NOT NULL,            -- git describe --always --dirty
    symbols       TEXT NOT NULL,
    start_ts      TEXT NOT NULL, end_ts TEXT NOT NULL,
    n_rows        INTEGER, n_cols INTEGER,
    artifact_path TEXT NOT NULL,            -- data/features/<run_id>.parquet
    created_at    TEXT NOT NULL
) STRICT;

CREATE TABLE label_run (
    run_id        TEXT PRIMARY KEY,
    config_json   TEXT NOT NULL,            -- horizon, tp_mult, sl_mult, cost model
    code_version  TEXT NOT NULL,
    artifact_path TEXT NOT NULL,
    created_at    TEXT NOT NULL
) STRICT;

CREATE TABLE model_run (
    run_id               TEXT PRIMARY KEY,
    feature_run_id       TEXT NOT NULL REFERENCES feature_run(run_id),
    label_run_id         TEXT NOT NULL REFERENCES label_run(run_id),
    model_kind           TEXT NOT NULL,     -- 'decision_tree' | 'hgb' | 'xgboost'
    side                 TEXT NOT NULL,     -- 'long' | 'short'
    params_json          TEXT NOT NULL,
    cv_metrics_json      TEXT NOT NULL,
    holdout_metrics_json TEXT,
    artifact_path        TEXT NOT NULL,
    created_at           TEXT NOT NULL
) STRICT;

-- ── Audit log of what the system actually told you to do ─────────────────────
CREATE TABLE decision (
    id             INTEGER PRIMARY KEY,
    model_run_id   TEXT NOT NULL REFERENCES model_run(run_id),
    symbol         TEXT NOT NULL,
    as_of          TEXT NOT NULL,           -- the session whose close drove it
    action         TEXT NOT NULL,           -- 'LONG' | 'SHORT' | 'HOLD'
    confidence     REAL NOT NULL,
    entry_ref      REAL,
    tp_price       REAL, sl_price REAL,
    atr_at_signal  REAL,
    max_hold_days  INTEGER,
    rationale_json TEXT,                    -- top contributing features
    created_at     TEXT NOT NULL,
    UNIQUE (model_run_id, symbol, as_of)
) STRICT;

-- Filled in later by `tdm score-decisions` (§13). This is the real scoreboard.
CREATE TABLE decision_outcome (
    decision_id   INTEGER PRIMARY KEY REFERENCES decision(id),
    resolved_ts   TEXT NOT NULL,
    exit_reason   TEXT NOT NULL,            -- 'tp' | 'sl' | 'timeout' | 'ambiguous'
    exit_price    REAL NOT NULL,
    gross_return  REAL NOT NULL,
    net_return    REAL NOT NULL,
    holding_days  INTEGER NOT NULL
) STRICT;
```

---

## 6. The cache-invalidation problem: store raw, adjust on read

This is the single most important schema decision, and it's a pure engineering problem —
no finance knowledge needed to see why.

**The domain fact:** when a company does a 10-for-1 **split**, every shareholder's 1 share
becomes 10 shares, each worth 1/10th as much. Nothing of value changed. But the price
series now has a 90% cliff in it at that date. So data providers offer *adjusted* prices:
all history before the split is divided by 10, making the series continuous again.

**The engineering problem:** an adjusted price is **a derived value computed from every
corporate action that has happened since.** It is not a fact about the past; it's a
function of the present. When a new split happens, *every historical adjusted price
changes.*

`yfinance` defaults to `auto_adjust=True`, so the obvious implementation stores exactly
those adjusted numbers. Then:

```
Jan 2026:  fetch NVDA 2015-2025, store adjusted closes.      ✓ consistent
Jun 2026:  NVDA does a 4:1 split.
Jul 2026:  fetch NVDA 2026 (incremental). Provider now returns
           everything on the NEW adjustment basis.
           -> rows written before June and rows written after June
              are on DIFFERENT SCALES, in the same table.
```

Your 200-day moving average now has a 75% discontinuity in the middle of it. **Nothing
raises an error.** No constraint is violated. The data is just quietly wrong, and every
model trained on it is wrong, and you will not find out from a stack trace.

**The fix** — treat it like any derived-value caching problem: don't cache the derived
value, cache the inputs and compute on read.

```python
# ingest: fetch and store the immutable facts
df = yf.download(symbol, auto_adjust=False, actions=True)   # RAW prices
store.upsert_bars(symbol, df[["Open","High","Low","Close","Volume"]])
store.upsert_actions(symbol, df[["Dividends","Stock Splits"]])

# read: derive adjustment at query time
def read_bars(symbol: str, *, adjust: bool = True,
              as_of: str | None = None) -> pd.DataFrame:
    bars    = _raw_bars(symbol)
    actions = _actions(symbol, before=as_of)      # as_of enables point-in-time reads
    if adjust:
        factor = _cumulative_adjustment_factor(actions, bars.index)
        bars[["open","high","low","close"]] *= factor
        bars["volume"] /= factor                  # ← inverse! see below
    return bars
```

Now `bar_daily` and `corporate_action` hold only immutable facts, adjustment is a pure
function of the two, and a split arriving in June recomputes all history correctly with
zero writes. The `as_of` parameter additionally gives you point-in-time reads, which §10.5
needs.

> **Easy bug to miss:** volume adjusts in the *opposite* direction. A 10:1 split means
> historical share counts must be **multiplied** by 10 while prices are divided by 10.
> `tests/test_indicators.py` should pin this against a real split — NVDA's 10:1 on
> 2024-06-10 is a good fixture.

---

## 7. Incremental ingest: never download the same bar twice

The user's explicit requirement. This is a cache-coherency problem with three wrinkles:
the upstream is rate-limited, it has permanent holes that look identical to cache misses,
and it revises recent rows.

### The provider interface

`ingest/` is the only package that touches the network, behind one protocol — so a
different data source (crypto via `ccxt`, another broker's API) slots in without touching
anything downstream:

```python
class Provider(Protocol):
    name: str

    def fetch_bars(self, symbol: str, start: date, end: date,
                   interval: str = "1d") -> pd.DataFrame:
        """Raw, UNADJUSTED OHLCV. Raises RateLimited (retryable)
        or ProviderError (fatal). Returns empty frame for no data."""

    def fetch_actions(self, symbol: str) -> pd.DataFrame: ...
    def fetch_metadata(self, symbol: str) -> dict: ...
```

### The sync planner

```python
def plan(symbol: str, requested_start: date) -> list[DateRange]:
    # 1. What days SHOULD exist? (market calendar, not just weekdays)
    sessions = calendar.sessions_in_range(requested_start, today)

    # 2. Which of those are final? A bar isn't trustworthy until the
    #    session has actually closed. Running at 14:00 ET must not
    #    cache a half-formed bar as if it were done.
    last_final = calendar.last_closed_session(now, settle_lag)
    target = [s for s in sessions if s <= last_final]

    # 3. Subtract what we already have
    cov = store.coverage(symbol, "1d", provider.name)
    if cov is None:
        missing = target                                        # cold start
    else:
        head     = [s for s in target if s <  cov.start_ts]     # extend backwards
        tail     = [s for s in target if s >  cov.end_ts]       # the normal daily case
        interior = [s for s in target if not store.has_bar(symbol, s)]   # heal holes
        missing  = head + tail + interior

    # 4. Subtract the permanent holes. Without this step the set never
    #    empties and you re-request Thanksgiving 2019 forever.
    missing = [s for s in missing if not store.is_known_gap(symbol, s)]

    if not missing:
        return []            # <-- the whole point: zero HTTP calls

    # 5. One request per contiguous run, not one per day.
    ranges = coalesce_contiguous(missing)

    # 6. Always re-pull the last few sessions: the provider revises
    #    recent bars (volume especially). Costs one request, keeps the
    #    tail honest, and `revision` makes it visible when it happens.
    ranges = widen_tail(ranges, RESTATE_LOOKBACK_SESSIONS)

    return ranges
```

Then execute: `fetch` each range with exponential backoff + jitter on `YfRateLimitError`
(yfinance raises this explicitly since 0.2.52), `UPSERT` the rows, bump `revision` where
OHLCV actually changed, record any still-absent target session as a `known_gap`, recompute
`fetch_coverage`, and append to `fetch_log`.

### Why each wrinkle needs its own mechanism

| Wrinkle | Mechanism | What breaks without it |
|---|---|---|
| Market isn't open every day | `exchange_calendars` NYSE schedule | You treat ~10 holidays/year as missing data and re-request them every run |
| Permanent holes (pre-IPO, halts) look like cache misses | `known_gap` table | The missing-set never converges; you burn the rate limit on holes that can never fill |
| Running mid-session | `last_final` + settle lag | You cache a partial bar whose "close" is just the last trade so far, as if final |
| Provider revises recent rows | `RESTATE_LOOKBACK = 5` + `UPSERT` + `revision` | Your recent tail silently diverges from the source |
| Rate limiting | backoff + `fetch_log` status | A 429 gets misread as "no data" and written as a permanent `known_gap` |

### Acceptance test

The one that matters, and it's mechanically checkable:

```bash
make fetch     # first run:  N HTTP calls
make fetch     # second run: ZERO HTTP calls   <- assert this in CI
```

### CLI

```bash
tdm universe validate                          # check the ticker list BEFORE downloading
tdm fetch                                      # incremental, universe from config
tdm fetch --symbols NVDA,AMD --since 2018-11-01
tdm fetch --dry-run                            # print the plan, make no calls
tdm fetch --force --symbols NVDA               # ignore coverage, re-pull and UPSERT
tdm db status                                  # coverage, gaps, staleness per symbol
tdm db verify                                  # integrity: high>=low, no negatives,
                                               # no dupes, no zero-volume non-holidays
```

`--dry-run` is how you test the planner without the network, and the first thing to reach
for when a run is unexpectedly slow.

### The universe is your input

`config/universe.txt` is a plain list of tickers you provide, one per line. Nothing is
hardcoded. Two things the ingest layer does with it:

**1. `tdm universe validate` runs first and fails fast.** Downloading 8 years × N tickers
takes a while, and you don't want to find out at ticker 48 that three of them were typos.
It resolves each symbol against the provider and reports:

| Check | Why |
|---|---|
| Symbol resolves at all | Catches typos and wrong suffixes (`BRK.B` vs `BRK-B` — Yahoo uses the dash) |
| Has ≥ 8 years of history | ⚠️ A 2024 IPO can't give you 8 years. Flagged, not rejected — but a ticker with 300 bars contributes almost nothing and will have NaN for the 200-day features |
| Currently active, not delisted | |
| Median volume above the liquidity floor | A thin stock will just be gated out at decision time (§13), so it's dead weight in the universe |
| Duplicates / blank lines | |

**2. Benchmark tickers are appended automatically.** The relative-strength and regime
features (§8.2) need SPY and the relevant sector ETFs. You list only what you want to
*trade*; `sync` adds the benchmarks it needs with `tradeable = 0`. So "no ETFs" (§1) stays
true of your decisions without breaking the features — and you never have to remember to
add SPY by hand.

---

## 8. Feature engineering

### The contract

Every feature is a pure function from a price series to a numeric series, registered with
its metadata:

```python
@dataclass(frozen=True)
class FeatureSpec:
    name: str
    fn: Callable[[pd.DataFrame], pd.Series]
    warmup: int                  # leading rows that will be NaN
    deps: tuple[str, ...] = ()   # other features this needs
    normalize: Normalizer | None = None   # see §8.5

registry.register(FeatureSpec(
    name="rsi_14",
    fn=lambda df: talib.RSI(df["close"], timeperiod=14),
    warmup=14,
))
```

`warmup` is declared rather than inferred so the pipeline drops exactly the right number
of leading rows instead of guessing — a 200-day EMA is NaN for its first 200 bars, and
silently filling those with zeros would be a subtle poisoning of the training set.

**No I/O in this layer**, which is what makes the leakage tests in §10.5 possible: you can
call any feature on a synthetic 50-row DataFrame in a unit test.

### 8.1 The user's requested indicators

All eleven are implemented. **Four need modification to be usable at all** — explained
below, because these are data-modelling bugs, not finance opinions.

| # | Requested | What we actually compute, and why |
|---|---|---|
| 1 | **Bollinger Bands** (20, 2σ) | Store `%B` = `(close − lower)/(upper − lower)` and `bandwidth` = `(upper − lower)/mid`. Not the raw band *prices* — those are in dollars (§8.5). |
| 2 | **RSI** (14) | Already 0–100 and dimensionless ✓. Store as-is. Plus RSI(2) and a divergence flag from §8.4. |
| 3 | **MACD** (12,26,9) | ⚠️ **Measured in dollars.** Store `MACD/ATR` and `histogram/ATR`, plus bars-since-sign-change. |
| 4 | **ADX** (14) | Already 0–100 ✓. Measures trend *strength* without direction — a natural first split for a tree. Pair with `(+DI − −DI)` for direction. |
| 5 | **Momentum Oscillator** | Use **ROC** (rate of change, a percentage ✓) at lookbacks 5/10/21/63. Avoid TA-Lib's `MOM`, which returns a raw dollar difference. |
| 6 | **OBV** | ⚠️ **A running cumulative sum** — its absolute value depends on how far back your data happens to start. Store the 21-day slope, z-scored. |
| 7 | **Stochastic** (14,3,3) | `%K`, `%D` already 0–100 ✓. Plus `%K − %D`. |
| 8 | **VWAP** | ⚠️ **Degenerate on daily bars** — see below. |
| 9 | **Elliott Wave** | ⚠️ Not a formula. → §8.4 |
| 10 | **Dow Theory** | ⚠️ Not a formula. → §8.4 |
| 11 | **21 EMA trend** | `close/EMA21 − 1`, EMA21 slope in ATR units, and an ordinal 0–4 "how aligned are EMA 8/21/50/200". |

> #### ⚠️ Why VWAP as specified doesn't work
>
> VWAP = `Σ(price × volume) / Σ(volume)`, **accumulated from the market open and reset
> every day**. It's an intraday metric.
>
> Compute it on a single daily bar and you get `typical_price × volume / volume` =
> `typical_price`. It algebraically reduces to a number you already have from the OHLC.
> Zero new information. (Many online tutorials get this wrong.)
>
> Two variants that do carry information at daily granularity:
>
> - **Rolling VWAP(n)** — volume-weighted mean over the trailing n days (20, 50).
>   Feature: `close/rvwap(n) − 1`.
> - **Anchored VWAP** — accumulate from a *chosen event* rather than a fixed window: the
>   last major swing low/high (from §8.4), the start of the year, or the last earnings
>   date. Features: `close/avwap_swinglow − 1`, `close/avwap_ytd − 1`.
>
> Anchored-VWAP-from-swing-low is, in my judgment, the most promising single item on the
> original list once corrected — it's genuinely informative and, unlike most of the list,
> it is not a near-linear function of the moving averages you already have.

### 8.2 Additional features worth adding

**Tier 1 — these carry information nothing on the original list does**

- **Relative strength vs SPY** (ratio ROC at 21/63/126 days, plus cross-sectional
  percentile rank). The requested list is entirely *absolute*: it cannot distinguish "NVDA
  rose 5%" from "NVDA rose 5% while the whole market rose 9%" — which are opposite
  signals. In published equity work, relative strength is repeatedly the strongest single
  daily-horizon feature.
- **Market-regime features** — computed once on SPY and broadcast to every row: SPY vs its
  own 200-day EMA, SPY realized-volatility percentile, VIX level and percentile, fraction
  of the universe above its own 50-day EMA. Up-markets and down-markets behave completely
  differently, and giving a tree these lets it split on regime *first*, which is exactly
  the structure you want for a 3-way decision.
- **ATR(14) and ATR/close.** Needed for TP/SL anyway (§9); as a feature it's your
  volatility-regime variable.
- **Gap features** — overnight gap in ATRs, and `days_since_large_gap`. Cheap, OHLCV-only,
  and they double as the earnings proxy described below.

> **ETFs are read but never traded.** The universe excludes ETFs *as trading candidates*
> (§1), but SPY and the sector ETFs are still fetched and used as **benchmark inputs** to
> the features above — relative strength is meaningless without something to be relative
> *to*. That's what `instrument.tradeable = 0` is for (§5): these rows flow through ingest
> and features exactly like any other ticker, and are filtered out only at the decision
> step (§13). Don't let "exclude ETFs" collapse into "don't download SPY."

**Tier 2 — cheap and complementary, given TA-Lib is already a dependency**

Keltner Channels; Donchian / 52-week-high proximity (`close/max(high, 252)`); MFI(14)
(volume-weighted RSI — bounded 0–100, so unlike OBV it's directly usable); Aroon(25)
(measures *time since* extremes rather than price distance); Supertrend(10, 3×ATR) (binary
state + distance-to-flip in ATRs — a natural tree feature); Ichimoku; CCI(20);
Williams %R(14); and simple bar-shape features — `(close−low)/(high−low)`, `range/ATR`,
consecutive up/down days, volume vs its 20-day median.

**~~Tier 3 — weekly timeframe~~ — dropped.**

Per the user's decision, no weekly resampling. Worth noting what this buys: the weekly
path carried a nasty leakage hazard (on a Wednesday the current week's bar isn't finished,
so using it leaks part of the move you're predicting), and dropping it **removes that
entire bug class** along with one item from §10.5's checklist. Daily bars with a 1–10 day
horizon is already swing trading; the weekly features were confirmation, not the signal.

**Explicitly rejected:** TA-Lib's 60+ candlestick pattern recognizers (`CDL*`). Dumping 60
sparse binary columns into a tree is an invitation to overfit, and the bar-shape features
above capture the same information continuously in ~3 columns.

### 8.2.1 Earnings: optional, and an OHLCV-only substitute

You asked whether earnings data is really needed, and whether OHLCV alone is enough.
**Short answer: OHLCV is enough to build the system, so earnings is `optional = false` by
default** and the `earnings_event` table stays empty until you turn it on. The build does
not block on it.

The honest caveat, stated once: at 10× leverage an earnings gap is the single most
dangerous event for an open position (§11.1 — a 10% gap wipes that position's margin
regardless of your stop). Earnings is also the *one* risk here that is **scheduled and
knowable in advance**, which makes skipping it the most avoidable of the tail risks.

So, a middle path that respects the OHLCV-only constraint — **derive an earnings proxy
from price data alone:**

| Feature | From OHLCV only |
|---|---|
| `days_since_large_gap` | bars since an overnight gap exceeded ~3 × ATR |
| `quarterly_cycle_pos` | position in the ~63-trading-day cycle since the last large gap — earnings land on a quarterly cadence, so past gaps predict future ones |
| `large_gap_due` | `quarterly_cycle_pos` in the window where the next one is statistically likely |

This is a weaker signal than a real calendar — it will mis-date events and confuse
earnings gaps with news gaps — but it needs no extra data source, no vendor, and no as-of
bookkeeping (§10.5 #6), and it captures most of the "don't open a new position right now"
value. **Recommendation:** ship v1 with the proxy, keep the real-calendar gate behind the
config flag, and turn it on before sizing up.

### 8.3 Redundancy is a real problem

RSI, Stochastic, CCI, Williams %R and MFI are all bounded momentum oscillators and are
heavily mutually correlated (|ρ| often > 0.8). Trees tolerate correlated inputs, but there
are two real costs: **feature importances become meaningless** (importance splits
arbitrarily among near-duplicates, so you can't tell what the model is using), and
effective complexity rises with no new information.

Pipeline steps:

1. Compute the correlation matrix at build time → `reports/feature_corr.html`.
2. Greedily drop one of any pair with |ρ| > 0.95, keeping the more interpretable one.
   **Log every drop** — visible and reviewable, never silent.
3. Report **permutation importance on held-out folds**, not sklearn's
   `feature_importances_`. The built-in attribute is impurity-based, biased toward
   high-cardinality features, and computed in-sample. It will mislead you.

### 8.4 Elliott Wave and Dow Theory

Both are **interpretive human frameworks, not algorithms.** Automated Elliott wave
counting is famously ambiguous — competent practitioners disagree on the same chart, and
any count gets revised as new data arrives. I'm not going to pretend to compute "we are in
wave 3". Instead both reduce to one shared primitive, and we extract measurable features
from it.

**The primitive: ZigZag swing pivots.** Find the local maxima and minima of the price
series, ignoring wiggles below a threshold:

```python
def pivots(bars, threshold) -> list[Pivot]:
    """Alternate between seeking a high and a low. Confirm a pivot only
    once price has reversed by >= threshold from the running extreme.
    threshold = 3 * ATR(14), or 5%."""
```

> ⚠️ **ZigZag is a lookahead trap, and the highest-risk item in this feature set.**
>
> Every charting tool draws ZigZag *with hindsight*. The pivot sitting at bar `t` is only
> **knowable** at bar `t+k`, once the reversal has actually occurred. Label pivots over the
> full series, feed them as features, and you've handed the model the future. The backtest
> will look spectacular and mean nothing.
>
> **So every pivot carries two timestamps:**
>
> ```python
> @dataclass(frozen=True)
> class Pivot:
>     ts: date            # when the extreme occurred
>     confirmed_at: date  # when it became KNOWABLE  <- filter on this
>     price: float
>     kind: Literal["high", "low"]
> ```
>
> Features for date `d` may only reference pivots where `confirmed_at <= d`. Enforced in
> the feature builder, asserted in `tests/test_leakage.py`.

**Dow Theory features** (the idea: trends persist, and are confirmed by volume and by
agreement across timeframes):

| Feature | Definition |
|---|---|
| `dow_trend` | `+1` if the last two swing highs *and* last two swing lows are both rising; `−1` if both falling; `0` otherwise |
| `dow_trend_age` | bars since `dow_trend` last changed |
| `swing_hh/hl/lh/ll` | booleans on the most recent completed leg |
| `dow_volume_confirm` | `+1` if volume expands with the trend and contracts against it, else `−1` |
| `dow_trend_major` | the same `dow_trend` computed from a **wider ZigZag threshold** (8 × ATR instead of 3 × ATR) |
| `dow_mtf_agree` | does `dow_trend` equal `dow_trend_major` |
| `dow_sector_confirm` | does the ticker's `dow_trend` equal its sector ETF's |

> **Note on `dow_trend_major` / `dow_mtf_agree`.** Dow Theory's "primary vs secondary
> trend" idea needs a longer timeframe to compare against. The obvious implementation is
> weekly bars — but weekly resampling is dropped (§8.2). The cleaner substitute is **two
> ZigZag thresholds over the same daily series**: a 3 × ATR threshold gives short-term
> swings, 8 × ATR gives the major ones. Same information, one data path, and **no
> partial-bar leakage hazard at all** — which makes this strictly better than the weekly
> version, not just a workaround.

**Elliott Wave features** — deliberately modest: ratios, not wave counts.

| Feature | Definition |
|---|---|
| `leg_retrace_ratio` | current leg's retracement of the previous leg, as a fraction |
| `fib_zone` | which band that falls in: `<0.236`, `0.236–0.382`, `0.382–0.5`, `0.5–0.618`, `0.618–0.786`, `>0.786` (ordinal) |
| `leg_extension_ratio` | current leg length ÷ previous leg length |
| `legs_since_major` | alternating legs since the last pivot exceeding 2× threshold |
| `impulse_vs_corrective` | is the current leg longer and steeper than the prior counter-leg |
| `leg_slope_atr` | leg price change per bar, in ATR units |

**My honest expectation:** the Dow structure features (`dow_trend`, `dow_mtf_agree`,
`dow_trend_age`) should earn their place — they encode trend persistence in a form trees
split on cleanly. **The Elliott/Fibonacci features I expect to be weak.**
`leg_retrace_ratio` plausibly carries something; the specific Fibonacci boundaries I'd
treat as a hypothesis to test, not a prior to trust. §8.3's held-out permutation
importance will tell us. If they come out flat, delete them — that's a successful
experiment, not a failure.

### 8.5 The normalization rule — read this one

> **Every feature must be dimensionless and roughly stationary.**

A decision tree learns literal threshold comparisons: `if macd > 1.84`. If a feature is
measured in dollars, that rule is specific to one ticker at one price level and transfers
to nothing. Train across 10 tickers over 15 years and such features contribute pure noise.

The closest everyday analogy: comparing servers by raw bytes-free rather than
percent-free. The threshold "alert below 10 GB" is meaningless across a fleet of different
disk sizes. Same bug class, same fix — divide by something that sets the scale.

| Raw quantity | Problem | Stored as |
|---|---|---|
| price, EMA, Bollinger bands, VWAP, Ichimoku lines | dollars | ratio to close (`x/close − 1`) or distance in ATRs |
| MACD, histogram, MOM | dollars | ÷ ATR |
| OBV, A/D line | unbounded cumulative sum | 21-day slope, then z-score over 252 days |
| volume | varies 1000× across tickers | ÷ 20-day median (log) |
| ATR | dollars | ÷ close |
| RSI, Stochastic, ADX, MFI, CCI, Aroon, %B | already bounded ✓ | as-is |
| anything long-tailed | outliers dominate the splits | winsorize at 1/99th pct, or 252-day rolling percentile rank |

> ⚠️ All rolling z-scores and percentile ranks use **trailing windows only** —
> `min_periods` set, **never centered**, and never `.fit()` on the full series. A centered
> window or a global `StandardScaler` reaches into the future. This is §10.5 item 4.

This normalization issue is, in my experience, the most common reason a cross-sectional
technical-indicator model fails to generalize — and it's invisible, because the model
trains fine and scores fine in-sample.

---

## 9. Label generation: a race between three exit conditions

This is where the vague question "should I trade?" becomes a concrete supervised learning
problem. The method is the **triple-barrier method** (from López de Prado, *Advances in
Financial Machine Learning*).

### The idea

Rather than predicting a price (hard, and not what you need), simulate the trade and ask
**which exit fires first**. It's a race between three conditions — close to a `select` over
three channels, or `asyncio.wait(..., FIRST_COMPLETED)`:

```
     price
       │
  TP ──┼─────────────────────────────  ← "profit" barrier    → y = 1
       │        ╱╲
 entry─┼──────╱────╲──╱──────────────
       │    ╱        ╲╱
  SL ──┼──────────────────────────────  ← "loss" barrier      → y = 0
       │
       └────────────────────────┤
        t+1                    t+10     ← timeout barrier     → y = sign(return)
```

### What "horizon" means

**Horizon (`H`) is the third barrier: the maximum number of trading days you hold before
giving up and closing at whatever the price happens to be.** `H = 10` means ~2 calendar
weeks. It's a timeout, in the ordinary programming sense.

So a trade has exactly three possible endings, and the horizon supplies the last one:

| Ending | Condition | Label |
|---|---|---|
| Win | TP touched first | `y = 1` |
| Loss | SL touched first | `y = 0` |
| **Timeout** | **neither touched within `H` days** | `y = 1` if the net return is positive, else `0` |

**Why have a timeout at all?** Three reasons, in order of importance:

1. **Without it, labels may never resolve.** A position that drifts sideways forever has
   no outcome, so there's no `y` to train on. The horizon guarantees every sample resolves
   and bounds how far into the future a label reaches — which is also what makes the
   purging in §10.3 possible.
2. **Capital has an opportunity cost.** A position going nowhere for a month is occupying
   margin and a concurrent-position slot that a better setup could use.
3. **Financing accrues daily** (§11.1). On a leveraged CFD, a stagnant position isn't
   neutral — it bleeds a little every day. The timeout caps that bleed.

**Why 10 and not something else?** It should match the TP distance: with TP at 2 × ATR,
you're asking for roughly two average days' worth of movement *in one direction*. Random
walks take a lot longer than two days to travel two ATRs, and empirically ~5–15 days is
the range where a 2-ATR move either happens or the setup has gone stale. 10 is the
midpoint, and it matches "swing trading" — holding across days, not minutes and not
months.

It's a tunable config value, not a law. `tdm labels sweep` reports what fraction of
samples end in timeout for each horizon: **if most of your trades are timing out, `H` is
too short** (or the TP is too far away); if almost none are, `H` isn't doing any work and
you're carrying financing cost for nothing.

### The generator

```python
def label_one(bars: pd.DataFrame, t: int, side: Side, cfg: LabelConfig) -> Label:
    atr   = bars.atr_14.iloc[t]        # known at t's close — no future data
    entry = bars.open.iloc[t + 1]      # we decide at t's close, fill at next open

    if side is Side.LONG:
        tp, sl = entry + cfg.k_tp * atr, entry - cfg.k_sl * atr
    else:
        tp, sl = entry - cfg.k_tp * atr, entry + cfg.k_sl * atr

    for i in range(t + 1, min(t + 1 + cfg.horizon, len(bars))):
        hi, lo = bars.high.iloc[i], bars.low.iloc[i]
        hit_tp = (hi >= tp) if side is Side.LONG else (lo <= tp)
        hit_sl = (lo <= sl) if side is Side.LONG else (hi >= sl)

        if hit_tp and hit_sl:          # same bar touched BOTH — see below
            return Label(y=0, t1=bars.index[i], reason="ambiguous", ...)
        if hit_tp:
            return Label(y=1, t1=bars.index[i], reason="tp", ...)
        if hit_sl:
            return Label(y=0, t1=bars.index[i], reason="sl", ...)

    # timeout: exit at the last bar's close, label on return net of costs
    exit_px = bars.close.iloc[t + cfg.horizon]
    net = net_return(entry, exit_px, side, cfg.costs)
    return Label(y=int(net > 0), t1=..., reason="timeout", net_return=net, ...)
```

Per sample we persist `y`, **`t1`** (when it resolved — §10.3 needs this), `exit_reason`,
`gross_return`, `net_return`, `holding_days`, and `weight`.

### Five design decisions, with reasoning

**1. The same-bar ambiguity, resolved pessimistically.** If a daily bar's high reaches TP
*and* its low reaches SL, the OHLC **does not tell you which happened first** — the
intrabar path isn't in the data. This is a genuine information loss, not a coding problem.
Options: assume the loss, fetch intraday data, or drop the sample. **We assume the loss.**
Resolving these optimistically is a classic silent backtest inflator, and it's worst
exactly where it hurts most — in high-volatility bars, which is where wide-TP trades live.
Dropping them instead biases the training set toward calm periods. The ambiguous count is
logged every run; above ~5% of samples, the barriers are too tight for daily data and
`k_sl` should widen.

**2. Fill at the next open, not today's close.** Deciding at bar `t`'s close and claiming a
fill at that same close assumes you traded using information from a price you then got.
`open[t+1]` is what you can actually do. Costs a little realism, buys a lot of honesty.

**3. Costs go *inside* the label, not bolted on afterward.** `net_return` subtracts the
CFD spread, slippage (a fraction of ATR — execution noise scales with volatility, so fixed
basis points would be wrong), and **daily financing on the notional, for both directions**
(§11.1). Because the timeout branch labels on `net_return > 0`, the model learns *post-cost*
profitability directly rather than learning a gross signal you then have to remember to
discount. Given that costs move break-even from 33% to ~42% (§11), having them inside the
label rather than applied later is what keeps the training target honest.

**4. Two binary models, not one 3-class model.** The design choice most worth scrutinizing.
The obvious framing is a 3-class label (LONG/SHORT/HOLD). Two problems: with asymmetric
barriers (TP 2 ATR, SL 1 ATR) the up-bet and down-bet are **not mirror images**, so one
label can't encode both; and "HOLD" would need an invented dead-zone threshold **baked
into the labels, where you can never re-tune it.** Instead:

```python
M_long  : P(long  hits TP before SL | features)      # binary
M_short : P(short hits TP before SL | features)      # binary
```

and HOLD *emerges at inference time* as "neither probability clears its threshold" (§13).
The thresholds become runtime knobs you tune against results, which is where they belong.
It also lets each direction learn its own feature relationships.

**5. Sample weights, because the labels overlap.** Bar `t` and bar `t+1` both look 10 days
ahead, so **consecutive samples share most of their future** and are not independent.
Standard CV treats them as if they were, which inflates every metric. Two corrections:

- **Uniqueness weight** — for each sample, average `1/concurrency(bar)` over `[t, t1]`,
  where `concurrency(bar)` counts how many live labels span that bar. Passed as
  `sample_weight` so crowded periods don't dominate.
- **Purging and embargo** in cross-validation — §10.3.

**Practical consequence:** 10 tickers × ~3,800 daily bars ≈ 38,000 rows, but the
**effective independent sample count is more like 2,000–4,000.** That number, not the row
count, is what should govern model complexity.

### Configuration and the parameter sweep

`k_tp`, `k_sl`, `horizon` and the ATR period form a named *label profile* in
`config.toml`, hashed into `label_run.run_id`. So two settings are two reproducible,
comparable experiments rather than an uncommitted edit.

```bash
tdm labels sweep     # grid over profiles. Per profile, report:
                     #   base rate            (fraction of positive labels)
                     #   break-even hit rate  (from the §11 cost model)
                     #   margin above break-even   <- the number that matters
                     #   mean holding period, timeout fraction, ambiguity rate
                     #   effective sample count (after uniqueness weighting)
```

**Run this before training.** It answers, in five minutes rather than after a day of model
tuning, whether a profile is even worth modeling:

- A **15% base rate** means the setup is rare and you'll learn from very few positives.
- A base rate **below the break-even hit rate** means the profile is unprofitable *even
  with a perfect model* — no amount of tuning saves it.
- **Most samples timing out** means `H` is too short, or the TP too far (§9).
- Since §11's cost arithmetic puts break-even near 42%, this sweep is where you find out
  whether a wider profile (TP 3 / SL 1.5) amortizes the fixed costs better. Don't assume
  TP 2 / SL 1 is optimal just because it's the starting point.

---

## 10. Model and validation

### 10.1 Model choice

The user asked for "a simple decision tree (you can suggest as well)". My suggestion is
**both**, doing two different jobs:

| Role | Model | Why |
|---|---|---|
| **Readable baseline** | `DecisionTreeClassifier(max_depth=3..5, class_weight="balanced", min_samples_leaf≈2%)` | You can print the entire rule set and read it. This tells you whether the model found something sensible or something absurd. **Keep it permanently as a sanity check**, not just as a first step. |
| **Workhorse** | `HistGradientBoostingClassifier` (sklearn, no extra dependency) | ✅ **Handles NaN natively** — a significant practical win, since every indicator has a warm-up period and a 200-EMA feature is NaN for the first 200 bars of each ticker. Fast, strong on tabular data, built-in early stopping. |
| Alternative | `xgboost` 3.4.1 | Comparable. Worth it for monotonic constraints or GPU; otherwise it's a dependency for marginal gain. Start with HGB. |
| Variance check | `RandomForest` / `ExtraTrees` | Cheap baseline. If it beats HGB, your HGB is overfit. |

> #### Sample size: the consequence of choosing 8 years
>
> This governs how complex a model you can justify, so it's worth the arithmetic.
>
> | | |
> |---|---|
> | 8 years × ~252 sessions | **~2,016 bars per ticker** |
> | ÷ horizon 10 (labels overlap — §9) | **~200 *effective independent* samples per ticker** |
> | × 9 tickers | ~1,800 effective samples |
> | × 30 tickers | ~6,000 effective samples |
> | × 60 tickers | ~12,000 effective samples |
>
> With 9 tickers the row count *looks* like 18,000. The information content is ~1,800.
> **That smaller number is what should govern model complexity.**
>
> **The key relationship: `effective samples ≈ tickers × years × 25`.** Years and tickers
> are interchangeable for statistical power, so the two knobs trade off directly.
>
> **8 years instead of 5 buys ~1.6× the power** — a real gain, and it costs nothing but
> download time. It also changes what's *in* the window:
>
> | Period | What it contributes |
> |---|---|
> | late 2018 | a ~20% correction |
> | 2019 | steady bull |
> | **Feb–Mar 2020** | **the COVID crash — a ~34% drop in five weeks** |
> | 2020–21 | recovery and melt-up |
> | 2022 | a ~25% bear market |
> | 2023–26 | bull, then the current regime |
>
> That's two bear markets plus a volatility shock — about as much regime diversity as any
> 8-year window could offer. You'd argued COVID wasn't essential because technical setups
> recur; at 8 years you get it included anyway, which is a free hedge against that
> assumption being wrong.
>
> **Two notes on including the COVID period:**
>
> 1. **ATR-scaled barriers partly self-normalize**, which is a nice property of the §9
>    design. When volatility tripled, ATR tripled, so TP/SL distances widened
>    proportionally rather than being blown through instantly. A fixed-percentage barrier
>    would have behaved very differently in Mar 2020 than in Jul 2021; an ATR barrier
>    adapts. This is a concrete payoff from the ATR choice.
> 2. ⚠️ **But it interacts badly with 10× leverage, and that needs handling** — see the
>    box below.
>
> The recommendation remains **"more tickers"** (§16 #1): 8 years is now a decent base, but
> tickers are the knob that also attacks survivorship bias (§10.5 #7), which adding years
> does not.

> #### ⚠️ Train on the same distribution you'll trade: filter training samples by the inference gates
>
> A consequence of including extreme-volatility periods, and a bug worth designing out
> rather than discovering.
>
> In March 2020, `ATR/close` for a volatile stock reached ~8–10%. At 10× leverage, a 1-ATR
> stop is then an **80–100% loss of that position's margin** — a margin wipe. So §13's
> `max_vol` gate will correctly refuse to trade at all in those conditions.
>
> But if those bars stay in the *training* set, the model is learning from a regime it will
> **never be asked to act in**, and in a 5-week stretch of violent moves those samples can
> carry outsized influence on the fitted thresholds.
>
> **Rule: build the training set through the same hard gates that §13 applies at inference
> time.** `max_vol`, `min_liquidity` and the earnings/gap filter all run at sample-
> construction time, not just at decision time. This is the "two implementations must
> agree" principle from §1 applied to *sample selection* rather than exit logic.
>
> Report the count of samples excluded by each gate, so the filtering is visible and you
> can tell whether a gate is quietly discarding half your data.
>
> Note this does **not** mean deleting the COVID period from the database. The bars stay —
> they're needed to compute indicators and regime features for the dates *after* them, and
> the market-regime features (§8.2) still let the model learn "conditions like these are
> dangerous." Only the *tradeable samples* are filtered.

**No neural networks.** Effective sample size is ~1,800–12,000 depending on universe size
(above). On tabular data at that scale, gradient-boosted trees consistently beat deep nets
in published benchmarks. The previous project carried a `torch` dependency; drop it. If you
later want sequence models, that's an argument for *more tickers first*, not a deeper
architecture.

**One global model across all tickers**, not one per ticker (per-ticker gets ~3,800 rows
and overfits). **Deliberately exclude ticker identity as a feature** — it invites the model
to memorize "NVDA went up 2015–2025", which is survivorship-flavored and won't repeat.
Sector as a categorical is fine; HGB supports native categoricals.

### 10.2 Class imbalance

With TP 2 ATR / SL 1 ATR, the positive rate will be well under 50% (≈33% is break-even).
**Don't fight it with resampling** — SMOTE on time-series features synthesizes rows that
violate the temporal structure. Use `class_weight="balanced"` / `sample_weight`, and tune
the decision threshold, which §13 does anyway.

### 10.3 Validation — the section that determines whether any of this is real

> ⚠️ **`train_test_split` and `KFold` are both guaranteed to lie to you here.** Two
> independent leaks:
>
> 1. **Shuffled folds** put future rows in train and past rows in test. You're predicting
>    2019 using 2024 data.
> 2. **Even with contiguous folds**, a label near the end of fold *k* resolves up to 10
>    bars into fold *k+1* (that's what `t1` records). So the training label literally
>    contains test-period prices.
>
> Leak (2) is the subtle one, and it's why `t1` exists.

`model/cv.py` implements `PurgedWalkForwardSplit`:

```
fold 1:  |== train ==| purge | embargo |== test ==|
fold 2:  |======= train =====| purge | embargo |== test ==|
fold 3:  |=========== train =======| purge | embargo |== test ==|
```

- **Expanding-window walk-forward** — always train on the past, test on the future.
- **Purge** — drop any training sample whose `[t, t1]` interval overlaps the test window.
- **Embargo** — additionally drop training samples within `E ≈ horizon` bars *before* the
  test start, because serial correlation leaks across the seam even without interval
  overlap.
- **A true final holdout** — the most recent **12 months** are **never touched**: not for
  hyperparameters, not for threshold tuning, not for feature selection. `tdm train` must
  refuse to read it without an explicit `--unlock-holdout` flag. The flag exists to make
  the peek a deliberate act you'll remember, because each peek spends a little of its
  honesty.

> **Why 12 months.** With 8 years of history (§10.1) a 12-month holdout is ~12.5% of the
> data — a comfortable fraction, leaving 7 years for walk-forward CV. (At the earlier
> 5-year plan an 18-month holdout would have been 30% and starved training; 8 years makes
> this a non-issue.)
>
> ⚠️ **The holdout is still the tightest constraint on your conclusion.** 12 months is ~25
> effective samples per ticker, so with 9 tickers the holdout carries ~225 — and once the
> §13 threshold filters most bars out, the surviving *trade* count can fall below the
> ~100-trade minimum from §10.4. **That would make the final go/no-go verdict itself
> statistically weak**, however good the training looked. With 30+ tickers it's ~750 and
> the verdict means something.
>
> This is §10.1's conclusion arrived at from a different direction, and it's why §16 #1 is
> the one open question that matters: a wider universe isn't a nice-to-have, it's what
> makes the final evaluation interpretable at all.

### 10.4 Metrics — accuracy is the wrong objective

Accuracy is near-meaningless on an imbalanced problem with asymmetric payoffs. A model
that always says HOLD can score 70% accuracy and be worth exactly nothing. Report, per
fold **and per direction**:

**Classification:** precision at the operating threshold (the one that matters: *of the
trades you actually take, how many win*), recall/coverage, PR-AUC (better than ROC-AUC
under imbalance), and **calibration** — a reliability curve, because §13 prints the
probability as a confidence number and "0.71" needs to actually mean 71%. Calibrate with
`CalibratedClassifierCV` on a purged inner split.

**Economic — the real scoreboard:**

| Metric | Why |
|---|---|
| **Expectancy per trade**, net of costs | The single number that decides whether to use this at all |
| Profit factor | gross wins ÷ gross losses |
| Hit rate + average R multiple | the two components of expectancy |
| Max drawdown, Calmar | whether you could actually tolerate it |
| Sharpe / Sortino | risk-adjusted, comparable to doing nothing |
| Trade count, avg holding days | **fewer than ~100 holdout trades ⇒ your estimate is noise** |
| **vs. buy-and-hold SPY over the same window** | the benchmark that counts. Beating a coin flip isn't the bar; beating "do nothing and own the index" is |

**Report the distribution across folds, never just the mean.** A strategy that was
excellent 2017–2019 and flat since has a fine average and is untradeable. Per-fold equity
curves go to `reports/`.

### 10.5 Leakage checklist — `tests/test_leakage.py`

Every item here is a bug I expect this project to hit at least once. Each gets a test.
**Treat this file as the most important one in the repo.**

| # | Invariant | Test |
|---|---|---|
| 1 | **No feature at date `d` uses any bar after `d`** | Property test: recompute every feature on the series truncated at `d`; assert the value at `d` is identical to the full-series computation. **This single test catches most of the rest.** |
| 2 | ZigZag pivots respect `confirmed_at` (§8.4) | Highest-risk item |
| 3 | ~~Weekly features~~ | **N/A — weekly features dropped (§8.2).** One fewer leakage class to defend against |
| 4 | Scalers / z-scores / percentiles use trailing windows only | Grep-level ban on `.fit(X_all)`; assert window params |
| 5 | Corporate-action lookahead | Features as-of `d` use only actions with `ts <= d` (the `as_of` param in §6). ⚠️ Strictly, a prospective backtest should use the adjustment factors *known at d*, not today's — this remains a **known bounded approximation**, documented rather than pretended-away |
| 6 | Earnings dates are as-of | Only applies if the optional earnings calendar is enabled (§8.2.1). Today's calendar isn't what you knew six months ago — use `confirmed` + `ingested_at`. The OHLCV-only proxy has no such problem, which is a point in its favour |
| 7 | **Survivorship bias** | ⚠️ **The one item on this list that good code cannot fix.** A universe of today's winners was selected *because* they went up. Partial mitigations: include has-beens (INTC already qualifies), add ETFs, add some losers, and keep delisted tickers with `active = 0` **rather than deleting the rows**. Mostly: know the number is optimistic |
| 8 | Holdout untouched | `--unlock-holdout` required (§10.3) |
| 9 | Purging actually applied | Assert no training `[t, t1]` overlaps any test fold |

---

## 11. Backtest

A small event-driven replay harness in `backtest/engine.py`. Recall from §3 that
`vectorbt` can't be installed on 3.15 — but we'd write this anyway, because §9's labeler
already defines the exit logic, and a third-party framework would be a *second,
independently-drifting* implementation of it. Here, one is the authority and the other
replays it.

```python
for d in test_sessions:
    x = features.loc[d]                        # from Parquet — never recomputed live
    p_long, p_short = m_long.predict_proba(x), m_short.predict_proba(x)

    orders = decision_rule(p_long, p_short, gates(d))     # SAME function as §13

    # position sizing: risk a fixed fraction of equity per trade.
    # NOTE: leverage does NOT appear here, by design — see §11.1
    units = (risk_frac * equity) / (cfg.k_sl * atr)
    margin = units * price / cfg.leverage        # what the position ties up
    # caps: max N concurrent positions, total margin used, max exposure per sector

    fills = fill_at(open_price[d + 1], slippage=s * atr)
    manage_open_positions(d)        # TP / SL / timeout — SAME pessimistic
                                    # same-bar rule as §9. Non-negotiable.
    accrue_financing(d)             # daily, on NOTIONAL, both directions (§11.1)
    mark_to_market(d)
```

> **The consistency requirement.** `decision_rule` and the exit logic are **imported** by
> both the backtest and production (§13), never reimplemented. If the simulator is more
> optimistic than the labeler, or production more permissive than the backtest, you've
> trained and validated against a world you don't deploy into — and the discrepancy shows
> up as unexplained live underperformance, which is almost impossible to debug after the
> fact. Share the code.

**Cost model.** Defaults below are **industry-typical retail stock-CFD figures, not
IQ Option's published numbers** — their site is JavaScript-rendered and I couldn't read
the actual terms. Treat them as order-of-magnitude starting points and replace them with
your real values from the platform (§16 #5).

```toml
[costs]
spread_pct        = 0.10   # round-trip spread, % of notional. Quoted the way brokers
                           # quote it; the engine divides by ATR internally.
                           # Typical retail: 0.05% on the most liquid (AAPL/MSFT),
                           # 0.1-0.2% mid-cap, wider on thin names.
slippage_atr_frac = 0.05   # execution noise per side, volatility-scaled
financing_pct_day = 0.02   # % per DAY on NOTIONAL, both directions.
                           # Typical: benchmark rate + ~3%/yr markup, /365.
commission        = 0.0    # normally zero on CFDs — the cost is the spread

[account]
equity         = 10_000
leverage       = 10        # sets margin used, NOT position size (§11.1)
risk_frac      = 0.01      # risk 1% of equity per trade
max_concurrent = 5
max_margin_frac = 0.50     # never tie up more than half of equity as margin
```

> #### ⚠️ Running the arithmetic changes the target
>
> Costs are not a rounding error here, and it's worth seeing the number before you build
> anything. Same example as §11.1 — $10,000 equity, 1% risk, $100 stock, ATR $2, 50 units,
> $5,000 notional, TP 2 ATR / SL 1 ATR, 10-day hold:
>
> | | |
> |---|---|
> | Gross win (TP hit) | 2 × $2 × 50 = **+$200** |
> | Gross loss (SL hit) | 1 × $2 × 50 = **−$100** |
> | Spread, round trip | 0.10% × $5,000 = −$5 |
> | Slippage, both sides | 0.1 × ATR × 50 = −$10 |
> | Financing, 10 days | 0.02%/day × 10 × $5,000 = −$10 |
> | **Total costs** | **−$25** |
>
> Break-even hit rate `p` solves `p × (200 − 25) = (1 − p) × (100 + 25)`:
>
> | | Break-even hit rate |
> |---|---|
> | Ignoring costs | **33.3%** |
> | **With costs** | **41.7%** |
>
> **Costs move the bar by 8.4 percentage points.** Given that §12 puts a realistically
> good hit rate at ~45%, costs consume most of the available margin. Three consequences:
>
> 1. **§12's expectations are recalibrated against 41.7%, not 33%.** A 40% hit rate is a
>    *losing* strategy here, not a near-miss.
> 2. **Spread and slippage are fixed per trade; financing scales with holding time.** So
>    *wider* barriers amortize the fixed part better — a TP 3 / SL 1.5 profile pays the
>    same spread to chase a 50% bigger move. This is a real argument for testing wider
>    profiles, which `tdm labels sweep` (§9) should now report **break-even hit rate per
>    profile** alongside base rate.
> 3. **Fewer, better trades beat more trades.** Every additional trade pays the fixed cost
>    again, which is an independent argument for a high decision threshold in §13.

**Required sensitivity runs.** A single backtest number is not evidence. Report the grid:
2× slippage, 2× commission, entry delayed one more bar, TP/SL multiples ±25%, horizon ±5
days. **If the edge vanishes when slippage doubles, there was no edge — there was a cost
assumption.**

Also run the **shuffled-signal control**: randomly permute the predictions and re-run. If
random does comparably well, the model contributes nothing and your equity curve is just
measuring the position sizing.

### 11.1 Execution venue: IQ Option CFDs at 10× leverage

Only the parts that change the code. A **CFD** (contract for difference) is a derivative:
you don't own shares, you hold a contract that pays the price difference. Three
consequences for this design.

**1. Shorting is symmetric, and one whole section of complexity disappears.**

With real shares, going short means borrowing them, which brings share-availability
limits, borrow fees, recall risk, and — significantly — **SEC Reg SHO Rule 201**, which
restricts short sales after a stock drops 10% in a day. An earlier draft of this document
had a long section on modeling that rule, because it would have fired on exactly the
setups the model most wants to short.

**None of it applies to CFDs.** There's no share borrow, so no availability constraint and
no uptick rule. Deleted:

- the `ssr_active` gate and feature
- the `shortable` availability logic
- borrow-cost and hard-to-borrow modeling

This is a genuine simplification: short and long become the same code path with a sign
flip, differing only in the financing rate.

**2. Financing is charged daily on notional, for both directions.**

Unlike owning shares, a leveraged CFD accrues a daily financing charge on the **full
position value**, not on your margin — and on **long positions too**, not just shorts. At
10× that charge is 10× larger relative to your own capital than the headline rate
suggests. Over a 10-day hold it's a real cost, not a rounding error.

So `accrue_financing` runs daily on open positions in both the labeler (§9) and the
backtest, and the rate is a single config value you fill in from your account.

**3. Position sizing: `risk_frac` sets the size, leverage only sets the margin.**

> #### ⚠️ The one piece of math worth being careful about
>
> The intuitive-but-wrong reading of "10× leverage" is *"I can take positions 10× bigger."*
> That multiplies your risk by 10 and is how leveraged accounts die.
>
> The design does the opposite: **size from risk, then check that margin fits.**
>
> ```
> units  = (risk_frac × equity) / (k_sl × ATR)     # leverage absent
> margin = units × price / leverage                # leverage only here
> ```
>
> Worked example — $10,000 equity, 1% risk, a $100 stock with ATR $2, SL at 1 ATR:
>
> | | Value |
> |---|---|
> | Risk budget | $100 (1% of equity) |
> | Loss per unit if SL hits | $2 |
> | **Units** | **50** |
> | Position value (notional) | $5,000 |
> | Margin tied up at 10× | **$500 (5% of equity)** |
>
> Your loss if the stop hits is $100 — 1% — **exactly the same as it would be with no
> leverage at all.** What leverage bought you is that the position consumes $500 of
> capital instead of $5,000, so you can hold ~10 uncorrelated positions instead of 2.
>
> **That's the correct use of leverage here: diversification, not amplification.** More
> concurrent positions lowers the variance of your results. Sizing 10× bigger instead
> would raise it tenfold, and a single gap could end the account.
>
> Because sizing is risk-based, `leverage` affects only the margin cap. The backtest
> enforces both `max_concurrent` and a total-margin ceiling so a run can't quietly assume
> more buying power than you have.

**4. What remains genuinely dangerous: gap risk, amplified 10×.**

A stop-loss only caps your loss if the price passes *through* it. An overnight or weekend
gap fills you well past it. At 10× leverage this is the one tail that matters:

> **A 10% adverse overnight gap, at 10× leverage, wipes out 100% of the margin on that
> position** — regardless of where your stop was.

A 10% single-day move in a large US stock is uncommon but entirely routine on earnings
days. That's the argument for the earnings gate, and it's why I'd keep it even though
you'd rather skip the earnings data (§16 #4 explains the OHLCV-only compromise).

Handling in the backtest:
- **Report max single-trade loss per direction**, separately from aggregate drawdown, so
  one catastrophic gap can't hide inside an average.
- **Report the margin-wipe count**: how many simulated positions lost ≥100% of their
  posted margin. If that number is above zero, the sizing or the gates need changing
  before this goes anywhere near real money.
- Model weekend gaps explicitly — CFD markets close, and `open[Monday]` vs
  `close[Friday]` is where they show up.

**Also report all §10.4 metrics per direction, never pooled.** Pooling hides the most
likely outcome — longs carry a modest edge, shorts carry none — behind a mediocre blended
number.

> **One mismatch to keep in mind.** Prices come from Yahoo (the real exchange); execution
> happens at IQ Option, whose quotes are its own and carry a wider spread. Features and
> labels are built from exchange data, so `spread_pct` is what bridges the two. If your
> live fills consistently differ from the backtest, that parameter is the first place to
> look — not the model.

---

## 12. Realistic expectations

Worth calibrating before you see a number, so you can tell success from self-deception.

- At R:R 2.0, break-even is a 33% hit rate *ignoring costs* — but **~42% once CFD spread,
  slippage and 10 days of financing are included** (the arithmetic is in §11). Judge
  expectancy (§2), not accuracy.
- ⚠️ **That 42% is the number that matters, and it's uncomfortably close to the ~45% that
  counts as a genuinely good result.** The realistic gap between "works" and "doesn't" is
  only a few percentage points of hit rate. Two implications: the cost parameters need to
  be *your real ones* (§16 #5), and wider TP/SL profiles deserve a serious look since they
  amortize the fixed costs better.
- Out-of-sample precision at threshold around **0.50–0.60** is a solid outcome for
  daily-bar technical features — and note that comfortably clears the 42% bar, which is
  where the opportunity is.
- ⚠️ **If your first run shows >65% out-of-sample accuracy, or a smooth upward equity
  curve, you have a bug — not an edge.** Go back to §10.5. This is the *expected* failure
  mode, not a remote one. The previous project's structure (no purging, no holdout
  discipline, no provenance) would not have surfaced it.
- Expect the honest outcome to be **"a modest edge in some market conditions, nothing in
  others."** Still useful: a model that reliably says HOLD during choppy periods and sizes
  up in trending ones has value even with an unremarkable hit rate.
- **Position sizing and risk management will affect your results more than model
  accuracy** — and at 10× leverage that's doubly true. Risk-based sizing (§11.1), a cap on
  concurrent positions, and avoiding earnings gaps are worth more than several points of
  AUC.
- **Be prepared for the shorts not to work.** US stock indices drift upward over time, so
  short setups have a worse base rate. CFDs remove the mechanical obstacles (§11.1) but
  not that drift. If `M_short` shows no out-of-sample edge, **ship long-only.** That's a
  finding, not a defeat.
- **Expect wide confidence intervals, and check them.** With 8 years the effective
  independent sample count is roughly `tickers × 200` (§10.1) — ~1,800 at 9 tickers,
  ~6,000 at 30. Distinguishing a 40% hit rate from a 45% one needs on the order of a
  thousand independent trades, so **report confidence intervals on the hit rate, not just
  the point estimate.** If the interval straddles the ~42% break-even, the honest readout
  is "inconclusive", not "profitable". **This is the strongest argument for a wider
  universe** (§16 #1).

---

## 13. Production inference: `tdm decide`

The command actually run day to day.

```python
def decide(symbol: str, as_of: date) -> Decision:
    x = features.load(symbol, as_of)          # from Parquet, not recomputed

    # ── Hard gates. Preconditions the model cannot see. Checked FIRST.
    inst = instrument(symbol)
    if not inst.tradeable:
        return SKIP("data-only instrument (ETF/benchmark)")   # §8.2
    if data_staleness(symbol) > 1:
        return ABSTAIN("stale data")          # ← fail loud, see below
    if adv_20d(symbol) < cfg.min_liquidity:
        return HOLD("illiquid")
    if atr_pct(symbol) > cfg.max_vol:
        return HOLD("volatility too extreme to size safely")
    if open_positions() >= cfg.max_concurrent:
        return HOLD("position limit")
    if margin_used_frac() >= cfg.max_margin_frac:
        return HOLD("margin ceiling reached")                 # §11.1

    # Earnings gate: real calendar if enabled, else the OHLCV-derived
    # proxy. Matters more at 10x leverage — §11.1, §8.2.1
    if cfg.earnings.enabled and earnings_within(symbol, cfg.horizon):
        return HOLD("earnings within horizon")
    if x["large_gap_due"] and cfg.earnings.use_proxy:
        return HOLD("gap-risk window (quarterly cycle proxy)")

    # ── Model
    p_long  = m_long.predict_proba(x)[1]
    p_short = m_short.predict_proba(x)[1]

    if p_long >= cfg.theta_long and p_long > p_short:
        side = Side.LONG
    elif p_short >= cfg.theta_short and p_short > p_long:
        side = Side.SHORT
    else:
        return HOLD(f"no edge: p_long={p_long:.2f} p_short={p_short:.2f}")

    # NOTE: no short-specific gates. CFDs have no share borrow and no
    # uptick rule, so both directions are the same path (§11.1).

    # ── Levels: same arithmetic as the labeler, so what you see is
    #    exactly what the model was trained on
    atr   = x["atr_14"]
    entry = last_close(symbol)                     # indicative; real fill is next open
    sign  = +1 if side is Side.LONG else -1
    units = (cfg.risk_frac * equity) / (cfg.k_sl * atr)   # leverage absent — §11.1
    return Decision(
        side=side, confidence=max(p_long, p_short),
        entry_ref=entry,
        tp=entry + sign * cfg.k_tp * atr,
        sl=entry - sign * cfg.k_sl * atr,
        units=int(units),
        margin=units * entry / cfg.leverage,       # what it ties up
        rationale=top_contributing_features(x, n=5),
    )
```

`theta_long` / `theta_short` are tuned on **validation folds only**, to maximize
expectancy (not accuracy), then **frozen and stored with the model artifact**. A threshold
that drifts between runs makes your track record uninterpretable.

> **The `ABSTAIN` on stale data is deliberate.** If the fetch failed and the newest bar is
> three days old, the system must **refuse to answer** rather than quietly deciding on
> stale prices. Silent degradation is the worst failure mode for a tool you check once a
> day and trust. Fail loud.

**Output:** a `rich` table on stdout, `reports/decisions_YYYY-MM-DD.{md,csv}`, and a row
in the `decision` table.

### Self-scoring — build this early, not last

```bash
tdm score-decisions --since 2026-01-01
```

Walks every past row in `decision`, resolves it against bars that have since arrived,
fills `decision_outcome`, and reports realized hit rate / expectancy / drawdown **of the
decisions the system actually issued**.

**This is the only genuinely out-of-sample measurement you will ever have.** No purging
subtleties, no leakage, no survivorship bias — just a prediction log and what happened
next. It's also what will tell you if a backtest was fooling you. Build it in phase 9, not
at the end, and let it accumulate: after six months it's worth more than any amount of
backtesting.

---

## 14. Build order

Each phase ends somewhere useful, so you can stop or re-scope without a half-built system.

| Phase | Deliverable | Done when |
|---|---|---|
| **0 · Scaffold** | venv 3.15, `CLAUDE.md`, Makefile, test runner | `make venv && make test` green; the venv guard fires outside the venv |
| **1 · Store + ingest** | schema, migrations, yfinance adapter, sync planner | `make fetch` twice ⇒ **second run makes zero HTTP calls**; `tdm db verify` clean. **Smoke-test yfinance against pandas 3.1 here** (§3) |
| **2 · Indicators** | registry + Tier-1 + the requested list, Parquet output, CSV export | spot-checks match a charting tool; `test_indicators.py` green; correlation report produced |
| **3 · Structure features** | ZigZag + Dow/Elliott with `confirmed_at` | `test_leakage.py` pivot tests green |
| **4 · Labels** | triple barrier, `t1`, weights, `labels sweep` | base rate / holding period / ambiguity rate sane across profiles |
| **5 · Baseline model** | depth-4 tree + purged walk-forward CV | **the printed rule set is readable and not absurd** |
| **6 · Workhorse** | HGB + calibration + permutation importance + threshold tuning | beats the depth-4 tree on expectancy, on validation folds |
| **7 · Backtest** | engine, costs, sensitivity grid, shuffled control | survives 2× slippage; beats the shuffled control |
| **8 · Decide** | `tdm decide`, gates, reports, `decision` table | one command, end to end |
| **9 · Self-scoring** | `tdm score-decisions` | reproduces phase-7 numbers on the overlapping window |
| **10 · Holdout** | one single evaluation on the untouched 12 months | **go / no-go** |

Phases 1 and 9 are what make this different from the previous attempt. **If you only reach
phase 5**, you still have a clean cached dataset, a reproducible feature pipeline, and a
readable baseline — already enough to iterate on.

---

## 15. Decisions resolved on 2026-11-03

| # | Question | Resolution |
|---|---|---|
| 1 | Asset class | **US stocks only, no ETFs** as trading candidates. ETFs still ingested as benchmarks with `tradeable = 0` (§8.2) |
| 1b | Universe source | **User-supplied ticker list** in `config/universe.txt`. No hardcoded universe, no IQ Option intersection logic — §7 |
| 2 | History depth | **8 years** (~2,016 bars/ticker). Covers the 2018 correction, the 2020 COVID crash and the 2022 bear market — §10.1 |
| 3 | Label profile | **TP 2.0 / SL 1.0 ATR.** Confirmed. `labels sweep` will still test wider profiles, because §11's cost arithmetic favours them |
| 4 | Horizon | **10 trading days.** Explained in §9 — it's the timeout barrier |
| 5 | Can you short? | **Yes**, and via CFDs it's mechanically symmetric with going long (§11.1) |
| 6 | Execution venue | **IQ Option stock CFDs, 10× leverage.** Drove the §11.1 rewrite: Reg SHO and share-borrow modeling deleted; financing and margin added |
| 7 | Earnings data | **Not required.** OHLCV-only, with a derived quarterly-gap proxy (§8.2.1). Real calendar stays behind a config flag |
| 8 | Weekly features | **Dropped.** Removes a leakage class (§8.2) |

---

## 16. Still open

1. **How many tickers?** ⚠️ **Still the most consequential open item**, and the only one
   left that changes whether the results are interpretable. 8 years gives
   `tickers × 200` effective samples (§10.1):

   | Tickers | Effective samples | 12-month holdout | Verdict quality |
   |---|---|---|---|
   | 9 | ~1,800 | ~225 | ⚠️ too thin for a confident go/no-go |
   | 30 | ~6,000 | ~750 | workable |
   | 60 | ~12,000 | ~1,500 | good |

   **Recommendation: 30 minimum, 50–60 preferred, spread across sectors** rather than
   concentrated in tech. Nine mega-cap tech names are also highly correlated with each
   other, so 9 tickers delivers appreciably less than 9× the information of one. More
   tickers is the only knob that buys statistical power *and* attacks survivorship bias
   (§10.5 #7) at once — and since you're supplying the list, it costs you nothing but
   typing.
2. **Risk per trade.** Default is 1% of equity, max 5 concurrent, margin ceiling 50%
   (§11). At 10× leverage, 1% risk uses only ~5% of equity as margin per position, so
   you could comfortably run 10–15 concurrent positions — which **lowers** variance
   through diversification. Want to raise `max_concurrent`?
3. **Earnings — final call.** Design ships OHLCV-only per your preference (§8.2.1). My
   one reservation, stated and then dropped: at 10× leverage a 10% earnings gap wipes a
   position's full margin (§11.1), and it's the only tail risk here that's knowable in
   advance. The proxy covers most of it. Your call, and it's reversible later.
4. **Your actual IQ Option costs.** The §11 defaults (0.10% round-trip spread, 0.02%/day
   financing) are industry-typical guesses, not their published numbers — I couldn't read
   their site. **Since costs move break-even from 33% to ~42%, these are load-bearing.**
   If you can read your spread and overnight fee off the platform for one or two stocks,
   that's the single highest-value number you can give me.
5. **Starting equity** for backtest sizing. Default $10,000.
