# Trading Decision Maker — Design

**Status:** design only, no code yet. Review this before implementation starts.
**Date:** 2026-11-03
**Author:** design drafted with Claude Code, to be reviewed by Habibie Faried

---

## 0. What this system is (and is not)

**Goal.** Once a day, after the US close, answer one question per ticker:

> Given today's data, should I open a **LONG**, open a **SHORT**, or do **NOTHING** —
> and if I trade, where exactly do I put the take-profit and the stop-loss?

Output is a short table plus the reasoning, e.g.:

```
NVDA   LONG    conf 0.71   entry 183.40   TP 196.10 (+6.9%)   SL 177.05 (-3.5%)   R:R 2.0   hold <= 10d
       because: EMA21 slope +0.8σ, ADX 31 (trending), RSI 58, %B 0.74, RS-vs-SPY 94th pct
AMD    HOLD    conf 0.48   — no edge: ADX 14 (chop), earnings in 3 days
TSLA   SHORT   conf 0.66   entry 402.10   TP 377.90 (-6.0%)   SL 414.20 (+3.0%)   R:R 2.0   hold <= 10d
```

**This is a research and decision-support tool.** It is not a money printer and it is
not an auto-trader. It does not place orders. A realistic, honest expectation is laid
out in §11 — read that section before you judge the model's numbers, because the
single most common way a project like this fails is a beautiful backtest built on
leakage.

### Scope decisions (locked in)

| Decision | Choice | Why |
|---|---|---|
| Market | **US stocks & ETFs**, daily bars, via yfinance | Free, deep history, matches the previous repo's universe |
| Horizon | **1–10 trading days** (swing) | You check once a day; enough signal-to-noise for ML, unlike next-day |
| TP/SL definition | **ATR-multiples** (volatility-scaled) | Adapts per ticker and per regime; NVDA and KO need different absolute stops |
| Timeframes | **Daily primary, weekly as confirmation features** | Weekly trend as a filter is the cheapest real edge in swing trading |
| Python | **3.15** | Explicit requirement; verified viable in §1 |
| Model | **Decision tree as the readable baseline, gradient boosting as the workhorse** | See §9 |

---

## 1. Runtime & environment

### Python 3.15 — verified viable

I checked this rather than assuming it, because a brand-new Python usually means
missing wheels in the scientific stack. As of today:

| Package | Latest | `cp315` wheel? | Notes |
|---|---|---|---|
| numpy | 2.5.4 | ✅ | |
| pandas | 3.1.0 | ✅ | **Major version — breaking changes, see below** |
| scikit-learn | 1.9.1 | ✅ | |
| TA-Lib (`ta-lib`) | 0.8.3 | ✅ | Wheels are self-contained; no `brew install ta-lib` needed |
| matplotlib | 3.11.2 | ✅ | |
| pyarrow | 26.0.1 | ✅ | |
| xgboost | 3.4.1 | ✅ (pure-py wheel, `requires-python >=3.12`) | |
| shap | 0.54.1 | ✅ | optional, for explaining the boosted model |
| yfinance | 1.7.0 | ✅ (pure-py) | |
| exchange-calendars | 4.13.2 | ✅ (pure-py) | NYSE session calendar |

**Two things are ruled out by the 3.15 choice.** Both are worth knowing up front so
nobody wastes an afternoon on them:

- **`pandas-ta` cannot be used.** It hard-pins `numba==0.61.2`, and numba 0.61.2 only
  publishes wheels up to `cp313`. It will not install on 3.15 at all. → We use
  **TA-Lib** as the indicator engine instead, which is faster anyway (Cython, 2–4×
  the old SWIG bindings) and has cp315 wheels.
- **`vectorbt` cannot be used.** It declares `requires-python <3.15`. → We write our
  own small walk-forward evaluator (§10), which we want regardless, because the
  triple-barrier labeling in §8 already encodes the exit logic and a general-purpose
  backtester would just be a second, disagreeing implementation of it.

**pandas 3.x is a breaking release.** The changes that will actually bite this project:

1. **Copy-on-Write is now mandatory.** Chained assignment (`df['x'][0] = v`) silently
   does nothing or raises. All indicator code must assign via `df.loc[...]` or build
   new columns functionally. This is strictly good for us but it will break any code
   copy-pasted from the old repo.
2. **Default datetime resolution is microsecond, not nanosecond.** `pd.to_datetime([0], unit="s")`
   now yields `datetime64[s]`. Anything that assumed `ns` and did integer math on
   timestamps is now wrong by a factor of 1000. We sidestep this by storing dates as
   ISO `TEXT` in SQLite and parsing explicitly.
3. **Offset aliases `M`, `Q`, `Y` were removed** — must use `ME`, `QE`, `YE`. Weekly
   resampling (`W-FRI`) is unaffected, which is what we mainly need.
4. **`inplace=True` now returns `self`** instead of `None`.
5. **String columns infer to the new `str` dtype**, not `object`.

> **Risk to watch:** `yfinance` declares only `pandas>=1.3.0` with no upper bound, so
> pip will happily install it alongside pandas 3.1 — but that does not mean it has been
> tested there. **The ingest path is the first thing to smoke-test.** If it breaks,
> the fallback is pinning `pandas>=2.3,<3` (which still has cp315 wheels for 2.3.x? —
> verify at the time) or vendoring a thin Yahoo client. Do not discover this in week 3.

### venv — mandatory, enforced three ways

Every command in this repo runs inside `./venv`. This is enforced redundantly so that
a future agent (or a future you, six months from now) cannot get it wrong:

1. **`.python-version`** → `3.15` (pyenv/uv read this).
2. **`CLAUDE.md`** → an explicit, unmissable instruction block at the top of the file.
3. **A runtime guard in `src/tdm/__init__.py`.** If `sys.prefix == sys.base_prefix`,
   raise immediately with a message telling the caller to activate the venv. An agent
   that forgets gets a loud, self-explaining error on the first call rather than
   mysteriously polluting the system Python.

Bootstrap:

```bash
brew install python@3.15                  # verified available: 3.15.2
/opt/homebrew/bin/python3.15 -m venv venv
./venv/bin/python -m pip install -U pip
./venv/bin/pip install -r requirements.txt
```

`uv venv --python 3.15` also works, but note your local `uv` (0.11.7, built 2026-04)
only knows about `3.15.0a8`; run `uv self update` first if you want that path.

A `Makefile` wraps everything so no one has to remember the venv path:

```make
PY := ./venv/bin/python
venv:      ## create venv and install deps
fetch:     ## $(PY) -m tdm.cli fetch --universe config/universe.txt
features:  ## $(PY) -m tdm.cli features build
label:     ## $(PY) -m tdm.cli labels build
train:     ## $(PY) -m tdm.cli train
backtest:  ## $(PY) -m tdm.cli backtest
decide:    ## $(PY) -m tdm.cli decide --date today
daily:     ## fetch -> features -> decide   (the once-a-day command)
test lint
```

---

## 2. Repository layout

```
trading-decision-maker/
├── .python-version                 # 3.15
├── CLAUDE.md                       # agent rules: ALWAYS ./venv, never system python
├── DESIGN.md                       # this file — the "why"
├── README.md                       # the "how to run" only
├── Makefile
├── pyproject.toml
├── requirements.txt                # loose constraints
├── requirements.lock.txt           # pip freeze, committed, for reproducibility
├── .gitignore                      # venv/, data/, models/, reports/, *.db
│
├── config/
│   ├── universe.txt                # one ticker per line
│   ├── config.toml                 # horizons, ATR multiples, costs, thresholds
│   └── feature_sets/
│       ├── baseline.toml           # the user-requested indicator list
│       └── extended.toml           # + the additions recommended in §7
│
├── src/tdm/
│   ├── __init__.py                 # venv guard
│   ├── cli.py                      # single entrypoint, subcommands
│   ├── config.py                   # pydantic-settings; config hashing
│   │
│   ├── store/
│   │   ├── schema.sql              # §4
│   │   ├── migrations/             # 001_init.sql, 002_... (forward-only)
│   │   ├── db.py                   # connection, pragmas, migration runner
│   │   └── bars.py                 # upsert_bars, read_bars, coverage queries
│   │
│   ├── ingest/
│   │   ├── provider.py             # Provider protocol (so ccxt/IDX can slot in later)
│   │   ├── yahoo.py                # yfinance adapter
│   │   ├── sessions.py             # exchange_calendars NYSE wrapper
│   │   └── sync.py                 # §5 incremental-fetch planner
│   │
│   ├── features/
│   │   ├── registry.py             # name -> fn, declared warm-up, declared deps
│   │   ├── trend.py                # EMA21 stack, ADX/DI, Aroon, Supertrend, Ichimoku
│   │   ├── momentum.py             # RSI, MACD, Stoch, ROC/MOM, CCI
│   │   ├── volatility.py           # ATR, Bollinger, Keltner, Donchian
│   │   ├── volume.py               # OBV, MFI, A/D, rolling & anchored VWAP
│   │   ├── structure.py            # ZigZag pivots -> Dow Theory, Elliott/Fib  (§7.4)
│   │   ├── market.py               # relative strength vs SPY, regime, breadth
│   │   ├── calendar_feats.py       # day-of-week, days-to-earnings, gap features
│   │   └── pipeline.py             # orchestrate, normalize, write Parquet
│   │
│   ├── labels/
│   │   └── triple_barrier.py       # §8
│   │
│   ├── model/
│   │   ├── dataset.py              # join features+labels, split, sample weights
│   │   ├── cv.py                   # PurgedWalkForwardSplit  (§9.3)
│   │   ├── train.py
│   │   ├── explain.py              # tree rules export, SHAP, permutation importance
│   │   └── registry.py             # model artifacts + provenance
│   │
│   ├── backtest/
│   │   └── engine.py               # §10
│   │
│   └── decide.py                   # §12 — the daily output
│
├── scripts/
│   └── bootstrap.sh
├── tests/
│   ├── test_indicators.py          # vs TA-Lib / hand-computed fixtures
│   ├── test_leakage.py             # §9.5 — the most important tests in the repo
│   ├── test_triple_barrier.py
│   └── test_incremental_fetch.py
│
└── data/                           # GITIGNORED
    ├── market.db                   # SQLite: bars, actions, metadata, runs
    ├── features/                   # Parquet, one file per (feature_set, run)
    ├── labels/                     # Parquet
    └── exports/                    # CSV, for eyeballing in Excel
└── models/                         # GITIGNORED — trained artifacts
└── reports/                        # GITIGNORED — daily decisions, backtest plots
```

### Why this split and not one big `train.py`

The previous repo (`stock-market-entry-signal-tpsl-specific`) had a 52 KB
`fetch_stock_data.py` and a 44 KB `train.py`, with per-day CSVs like
`AAPL_tpsl_data_20260610.csv` accumulating in `data/`. That layout has two specific
failure modes this design is built to avoid:

1. **No cache** — every run re-downloads, so you rate-limit yourself and can't iterate.
2. **No provenance** — given `models/xyz.pkl` you cannot reconstruct which features,
   which labeling parameters, or which date range produced it. So you can't trust a
   comparison between two models.

§4's `*_run` tables and §6's config hashing exist to fix (2). §5 fixes (1).

---

## 3. Data flow

```
                yfinance (rate-limited, retroactively restated)
                             │
                   ┌─────────▼──────────┐
                   │  ingest/sync.py    │  asks: what's actually missing?
                   │  (coverage-aware)  │  only fetches the gaps
                   └─────────┬──────────┘
                             │  raw, UNADJUSTED bars + corporate actions
                   ┌─────────▼──────────┐
                   │  data/market.db    │  ← the only expensive-to-rebuild thing
                   │  (SQLite, STRICT)  │     THIS is the cache that matters
                   └─────────┬──────────┘
                             │  read_bars(symbol, adjust=True)  ← adjust on READ
                   ┌─────────▼──────────┐
                   │ features/pipeline  │  TA-Lib + custom; all scale-free (§7.5)
                   └─────────┬──────────┘
                             │
          ┌──────────────────┴──────────────────┐
          │                                      │
 ┌────────▼─────────┐                  ┌────────▼──────────┐
 │ data/features/   │                  │ labels/triple_    │
 │   *.parquet      │                  │   barrier.py      │
 └────────┬─────────┘                  └────────┬──────────┘
          │                                      │ y, t1, ret, weight
          └──────────────────┬───────────────────┘
                             │
                   ┌─────────▼──────────┐
                   │ model/train.py     │  purged walk-forward CV
                   └─────────┬──────────┘
                             │
          ┌──────────────────┴──────────────────┐
 ┌────────▼─────────┐                  ┌────────▼──────────┐
 │ backtest/engine  │                  │ decide.py         │
 │ (honest metrics) │                  │ (today's call)    │
 └──────────────────┘                  └────────┬──────────┘
                                                 │
                                        reports/ + decision table in SQLite
                                        (so you can score your own track record)
```

---

## 4. SQLite schema

Design rules:

- **SQLite stores only what is expensive to re-acquire**: raw bars, corporate actions,
  instrument metadata, and run provenance. Derived matrices go to Parquet (§6).
- **`STRICT` tables** throughout (SQLite 3.51 is bundled with Python 3.15; `STRICT`
  needs only 3.37+). Without `STRICT`, SQLite's type affinity will happily store the
  string `"NaN"` in a `REAL` column and you will find out during training.
- **Dates are ISO-8601 `TEXT`** (`'2026-11-03'`). Avoids the pandas 3 datetime-resolution
  trap entirely, sorts lexicographically, and is readable in a SQL console.
- **Raw prices only.** See the boxed note below — this is the single most important
  schema decision in the document.

```sql
PRAGMA journal_mode = WAL;        -- concurrent read while a fetch writes
PRAGMA synchronous  = NORMAL;
PRAGMA foreign_keys = ON;

-- ── Instruments ──────────────────────────────────────────────────────────────
CREATE TABLE instrument (
    symbol        TEXT PRIMARY KEY,
    name          TEXT,
    exchange      TEXT,                     -- 'NMS', 'NYQ', ...
    currency      TEXT    NOT NULL DEFAULT 'USD',
    asset_type    TEXT    NOT NULL DEFAULT 'equity',   -- equity | etf | index
    shortable     INTEGER NOT NULL DEFAULT 1,          -- 0 => model may only go long
    active        INTEGER NOT NULL DEFAULT 1,
    first_bar     TEXT,
    last_bar      TEXT,
    meta_json     TEXT,                     -- raw provider payload, for later needs
    updated_at    TEXT    NOT NULL
) STRICT;

-- ── Raw daily bars — the core cache ──────────────────────────────────────────
CREATE TABLE bar_daily (
    symbol        TEXT    NOT NULL REFERENCES instrument(symbol),
    ts            TEXT    NOT NULL,         -- session date, exchange-local
    open          REAL    NOT NULL,
    high          REAL    NOT NULL,
    low           REAL    NOT NULL,
    close         REAL    NOT NULL,         -- UNADJUSTED close
    volume        INTEGER NOT NULL,
    source        TEXT    NOT NULL,         -- 'yahoo'
    ingested_at   TEXT    NOT NULL,
    revision      INTEGER NOT NULL DEFAULT 1,  -- bumped if provider restates the bar
    PRIMARY KEY (symbol, ts)
) STRICT, WITHOUT ROWID;

CREATE INDEX idx_bar_daily_ts ON bar_daily(ts);

-- ── Corporate actions — why we don't cache adjusted prices ───────────────────
CREATE TABLE corporate_action (
    symbol        TEXT NOT NULL REFERENCES instrument(symbol),
    ts            TEXT NOT NULL,
    kind          TEXT NOT NULL,            -- 'dividend' | 'split' | 'capital_gain'
    value         REAL NOT NULL,            -- cash amount, or split ratio (2.0 = 2:1)
    ingested_at   TEXT NOT NULL,
    PRIMARY KEY (symbol, ts, kind)
) STRICT;

-- ── Fetch bookkeeping — "don't fetch it twice" lives here ────────────────────
CREATE TABLE fetch_coverage (
    symbol        TEXT NOT NULL,
    interval      TEXT NOT NULL,            -- '1d'
    source        TEXT NOT NULL,
    start_ts      TEXT NOT NULL,            -- earliest session we have
    end_ts        TEXT NOT NULL,            -- latest session we have
    bar_count     INTEGER NOT NULL,
    last_fetch_at TEXT NOT NULL,
    PRIMARY KEY (symbol, interval, source)
) STRICT;

-- Sessions the calendar says exist but the provider returned nothing for.
-- Without this table you re-request the same holes every single run, forever.
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

-- ── Earnings dates — a swing-trade-specific hazard, see §7.3 ─────────────────
CREATE TABLE earnings_event (
    symbol        TEXT NOT NULL REFERENCES instrument(symbol),
    ts            TEXT NOT NULL,
    session       TEXT,                     -- 'bmo' | 'amc' | 'unknown'
    confirmed     INTEGER NOT NULL DEFAULT 0,
    ingested_at   TEXT NOT NULL,
    PRIMARY KEY (symbol, ts)
) STRICT;

-- ── Provenance: which code+config produced which artifact ────────────────────
CREATE TABLE feature_run (
    run_id        TEXT PRIMARY KEY,         -- hash(feature_set, code_ver, date_range)
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
    run_id            TEXT PRIMARY KEY,
    feature_run_id    TEXT NOT NULL REFERENCES feature_run(run_id),
    label_run_id      TEXT NOT NULL REFERENCES label_run(run_id),
    model_kind        TEXT NOT NULL,        -- 'decision_tree' | 'hgb' | 'xgboost'
    side              TEXT NOT NULL,        -- 'long' | 'short'
    params_json       TEXT NOT NULL,
    cv_metrics_json   TEXT NOT NULL,
    holdout_metrics_json TEXT,
    artifact_path     TEXT NOT NULL,
    created_at        TEXT NOT NULL
) STRICT;

-- ── Your own live track record ───────────────────────────────────────────────
CREATE TABLE decision (
    id            INTEGER PRIMARY KEY,
    model_run_id  TEXT NOT NULL REFERENCES model_run(run_id),
    symbol        TEXT NOT NULL,
    as_of         TEXT NOT NULL,            -- session whose close drove the decision
    action        TEXT NOT NULL,            -- 'LONG' | 'SHORT' | 'HOLD'
    confidence    REAL NOT NULL,
    entry_ref     REAL,                     -- expected entry (next open)
    tp_price      REAL, sl_price REAL,
    atr_at_signal REAL,
    max_hold_days INTEGER,
    rationale_json TEXT,                    -- top contributing features
    created_at    TEXT NOT NULL,
    UNIQUE (model_run_id, symbol, as_of)
) STRICT;

-- Filled in later by `tdm score-decisions`. This is how you find out whether the
-- thing actually works on data it has never seen, in the only test that counts.
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

> ### ⚠️ Store RAW prices, adjust on read
>
> It is tempting to let `yfinance` hand you `auto_adjust=True` (its default in 1.x) and
> save those numbers. **Don't.** Adjusted prices are a *function of all corporate actions
> that have happened since*, so they are retroactively rewritten every time a ticker
> splits or pays a dividend. If you cache adjusted closes, then NVDA announces a split,
> your cache is now silently inconsistent: bars fetched before the split disagree with
> bars fetched after, your 200-day EMA has a cliff in it, and nothing errors out.
>
> So: fetch with `auto_adjust=False, actions=True`, store the raw OHLCV in `bar_daily`
> and the actions in `corporate_action`, and compute the cumulative adjustment factor at
> **read** time in `store/bars.py::read_bars(..., adjust=True)`. Adjustment becomes a
> pure, re-derivable function of two immutable tables. A split arriving later changes
> nothing on disk and everything downstream recomputes correctly.
>
> Volume needs the inverse split adjustment (2:1 split ⇒ historical volume ×2). Easy to
> forget; `tests/test_indicators.py` should cover it with a known split (e.g. NVDA 10:1,
> 2024-06-10).

---

## 5. Incremental fetch: "never download the same bar twice"

This is the requirement you called out explicitly, so it gets a full algorithm.

### The planner

```
sync(symbol, requested_start, interval='1d'):

  1. sessions   := NYSE calendar sessions in [requested_start, today]   (exchange_calendars)
  2. last_final := last session whose close + settle_lag has passed
                   # a bar is not trustworthy until the session has actually closed;
                   # running at 14:00 ET must not cache a half-formed bar as final
  3. target     := sessions ∩ [requested_start, last_final]

  4. cov := fetch_coverage[symbol, interval, source]
     if cov is None:
         missing := target                                  # cold start
     else:
         head    := target ∩ [requested_start, cov.start_ts)   # extend history backwards
         tail    := target ∩ (cov.end_ts, last_final]          # the normal daily case
         interior:= target \ (bars_present ∪ known_gap)         # heal real holes only
         missing := head ∪ tail ∪ interior

  5. missing := missing \ known_gap        # don't re-ask about holidays/pre-IPO holes

  6. if missing is empty:  log 'ok, 0 rows', RETURN      # <-- the whole point
  7. coalesce `missing` into contiguous date ranges (one HTTP call per range,
     not one per day), and widen each range's tail by `RESTATE_LOOKBACK = 5`
     sessions to pick up provider restatements of recent bars

  8. for each range: provider.fetch(...) with exponential backoff + jitter on
     YfRateLimitError; UPSERT; bump `revision` where OHLCV changed

  9. any target session still absent after a successful fetch -> INSERT into
     known_gap(reason='no_data')

 10. recompute fetch_coverage; append fetch_log
```

### Why each piece exists

- **`known_gap`** — without it, every run asks Yahoo again for Thanksgiving 2019 and
  for the years before the ticker's IPO. Those requests always come back empty, so a
  naive "which dates am I missing?" check never converges and you burn your rate limit
  on holes that will never fill.
- **`last_final` / settle lag** — guards against caching a partially formed bar when you
  run the tool mid-session. Without this, an intraday run poisons the cache with a bar
  whose "close" is just the last trade so far.
- **`RESTATE_LOOKBACK = 5`** — Yahoo does revise recent bars (volume especially). Always
  re-pull the last week and UPSERT; it costs one request and keeps the tail honest.
  `revision` increments so you can see it happening.
- **Coalescing ranges** — one call for `2015-01-01..2026-11-03` instead of 2,700 calls.
- **Backoff on `YfRateLimitError`** — yfinance 0.2.52+ raises this explicitly; treat it
  as retryable, log it to `fetch_log`, and never let it look like a data gap.

### CLI surface

```bash
make fetch                                   # incremental, universe from config
tdm fetch --symbols NVDA,AMD --since 2015-01-01
tdm fetch --force --symbols NVDA             # ignore coverage, re-pull and UPSERT
tdm fetch --dry-run                          # print the fetch plan, make no calls
tdm db status                                # per-symbol coverage, gaps, staleness
tdm db verify                                # OHLC sanity: high>=low, no negatives,
                                             # no zero-volume non-holidays, dupes
```

`--dry-run` matters more than it sounds: it's how you confirm the cache logic works
without waiting on the network, and it's what you check when a run is unexpectedly slow.

---

## 6. Where derived data lives, and why not all in SQLite

You asked for "csv or sqlite" for the computed indicators. Having thought about the
access pattern, here's the recommendation and the honest tradeoff.

| Store | Shape | Read speed | Schema churn | Verdict |
|---|---|---|---|---|
| SQLite **wide** (1 col/feature) | 120+ columns | fast | `ALTER TABLE` on every new indicator — painful | ✗ |
| SQLite **long** (`symbol, ts, name, value`) | 3 cols, ~5× rows | needs a pivot on read | none | ○ optional |
| **Parquet** | wide, columnar, typed | fastest; column pruning | none | ✓ **default** |
| CSV | wide, untyped | slow; loses dtypes/NaN fidelity | none | ○ export only |

**Recommendation: raw bars in SQLite, feature matrix in Parquet, CSV on demand.**

The reasoning is that features are *cheap and disposable* — a full rebuild across 10
tickers × 15 years of daily bars is seconds of TA-Lib work. What's expensive is the
network fetch. So the cache that needs to be durable and transactional is `bar_daily`,
and that's exactly what SQLite is good at. The feature matrix is a wide block of
float64 that gets read whole into pandas — that's Parquet's sweet spot, and it stays
schema-free so adding your 40th indicator is a one-line registry entry, not a migration.

Both other formats stay available because you asked for them:

```bash
tdm features build --set extended            # -> data/features/<run_id>.parquet
tdm features export --format csv    --out data/exports/   # for Excel
tdm features export --format sqlite --table feature_long  # long-narrow, if you want SQL
```

The `feature_run` / `label_run` / `model_run` tables in §4 are the glue: the Parquet
files are content-addressed by a hash of (feature-set config + code version + date
range + symbol list), and that same `run_id` is recorded against every model trained
from it. So `models/abc123.pkl` is always traceable back to the exact features,
labels, and parameters that produced it. That's what the previous repo was missing,
and it's what makes two models actually comparable.

---

## 7. Indicators — research and design

You listed eleven. All eleven are in, but **four need real modification** to be usable,
and two of them are not indicators at all. I'd rather flag that now than have you
discover it when the features turn out to be useless.

### 7.1 Your list, as designed

| # | You asked for | Standard params | What we actually compute, and why |
|---|---|---|---|
| 1 | **Bollinger Bands** | 20, 2σ | **`%B`** = (C − lower)/(upper − lower), and **bandwidth** = (upper−lower)/mid. Never the raw band prices — those are in dollars (see §7.5). Bandwidth is a genuinely useful volatility-regime feature; low bandwidth = "squeeze" = breakout pending. |
| 2 | **RSI** | Wilder 14 | Raw RSI(14) is already 0–100 and scale-free ✓. Add RSI(2) (short-term mean-reversion, well documented for swing) and an **RSI-divergence** flag from §7.4 pivots. |
| 3 | **MACD** | 12, 26, 9 | ⚠️ **MACD is in price units** — AAPL's MACD and BRK.A's MACD are incomparable, and a tree splitting at "MACD > 1.8" learns nothing transferable. We store **MACD/ATR** and **histogram/ATR**, plus the signed bars-since-cross. |
| 4 | **ADX** | 14 | ADX is 0–100 ✓, measures trend *strength* without direction. Excellent **regime gate**: >25 trending, <20 chop. Pair with `(+DI − −DI)` for direction. |
| 5 | **Momentum Oscillator** | — | Use **ROC** at several lookbacks (5, 10, 21, 63) — already a percentage ✓. Prefer ROC over TA-Lib `MOM`, which returns a raw price difference. |
| 6 | **OBV** | — | ⚠️ **OBV is a running cumulative sum**, so its level is an arbitrary number that depends on how far back your data starts. Unusable raw. We use **OBV's 21-day slope, z-scored**, and OBV/price divergence. |
| 7 | **Stochastic** | 14, 3, 3 | %K and %D, both 0–100 ✓. Add `%K − %D` and bars-in-overbought/oversold. |
| 8 | **VWAP** | — | ⚠️ **This is the big one — see the box below.** True VWAP is intraday and resets daily; on daily bars it degenerates to roughly the typical price of that one bar. Replaced by rolling VWAP + **anchored VWAP**. |
| 9 | **Elliott Wave** | — | ⚠️ Not a computable indicator. Operationalized as Fibonacci-retracement features over ZigZag pivots — §7.4. |
| 10 | **Dow Theory** | — | ⚠️ Not a computable indicator. Operationalized as HH/HL/LH/LL swing structure + volume + multi-timeframe confirmation — §7.4. |
| 11 | **21 EMA trend** | 21 | `(C/EMA21 − 1)`, EMA21 slope normalized by ATR, and the **stack state** EMA8/21/50/200 (an ordinal 0–4 "how aligned is the trend" feature). |

> #### ⚠️ On VWAP specifically
>
> VWAP = Σ(price×volume)/Σ(volume) **accumulated from the session open, reset every day**.
> It is an intraday execution benchmark. If you compute it on a daily bar you get
> `(H+L+C)/3 × V / V` — the typical price of that bar, which carries no information
> beyond the OHLC you already have. Many tutorials get this wrong.
>
> For daily/weekly swing trading the two meaningful variants are:
>
> - **Rolling VWAP(n)** — volume-weighted average over the trailing *n* days (20, 50).
>   Feature: `C/RVWAP(n) − 1`. A volume-weighted cousin of distance-from-MA.
> - **Anchored VWAP (AVWAP)** — accumulated from a *chosen event*, not a fixed window:
>   the last major swing low/high (from §7.4), the year-to-date open, or the last
>   earnings date. This is the version swing traders actually use, because it answers
>   "what's the average price everyone who bought since the bottom paid?" — which is a
>   real support/resistance level. Features: `C/AVWAP_swinglow − 1`,
>   `C/AVWAP_ytd − 1`, `C/AVWAP_earnings − 1`.
>
> AVWAP-from-swing-low is, in my judgment, the most promising single item on your
> original list once corrected — it's genuinely informative and it is not a linear
> function of the moving averages you already have.

### 7.2 Additional indicators worth adding

Researched specifically for **daily/weekly swing** horizons. The first two are the ones
I'd push hardest for.

**Tier 1 — add these, they carry information nothing on your list does**

- **Relative strength vs SPY** (ratio ROC at 21/63/126 d, plus cross-sectional
  percentile rank). Across published equity work, relative strength is repeatedly the
  strongest single daily-horizon feature. Your whole list is *absolute* — it can't tell
  "NVDA is up 5%" from "NVDA is up 5% while the market is up 9%", which are opposite
  signals.
- **Market-regime features** (computed on SPY, broadcast to every row): SPY vs its 200
  EMA, SPY 21-day realized vol percentile, VIX level + percentile, VIX term structure
  (VIX3M/VIX), % of universe above its own 50 EMA. **Long setups and short setups behave
  completely differently by regime**, and a tree given these can split on regime first
  — which is exactly the structure you want for a long/short/hold decision.
- **ATR(14) and normalized ATR** (`ATR/C`). You need ATR anyway for TP/SL (§8); as a
  feature, normalized ATR is your volatility-regime variable.
- **Days-to-next-earnings** and **is-earnings-within-horizon**. An earnings gap can jump
  straight through a stop, which makes it the most common way a backtested swing
  strategy dies in live trading. It must be a feature *and* a hard gate (§12). ⚠️ Note
  the subtlety: future earnings dates must come from an as-of-that-date snapshot, not
  today's calendar — see §9.5.

**Tier 2 — cheap, complementary, standard for this horizon**

- **Keltner Channels** (EMA20 ± 2×ATR10) — ATR-based rather than σ-based; the
  BB-inside-Keltner condition is the canonical "squeeze".
- **Donchian / 52-week-high proximity** (`C/max(H,252)`) — breakout and momentum, and a
  well-documented anomaly in its own right.
- **MFI(14)** — volume-weighted RSI; bounded 0–100, so unlike OBV it's directly usable
  and comparable across tickers.
- **Aroon(25)** up/down/oscillator — measures *time since* extremes rather than price
  distance; captures something the price-distance indicators don't.
- **Supertrend(10, 3×ATR)** — binary trend state + distance-to-flip in ATRs. Popular for
  swing and a natural tree feature.
- **Ichimoku** (Tenkan/Kijun/cloud position, cloud thickness) — explicitly designed for
  daily/weekly timeframes.
- **CCI(20)**, **Williams %R(14)** — bounded oscillators, nearly free given TA-Lib.
- **Candle/microstructure**: close-location-value `(C−L)/(H−L)`, range/ATR, gap size in
  ATRs, consecutive up/down days, volume vs its 20-day median.

**Tier 3 — weekly-timeframe mirrors (the "weekly" half of your request)**

Resample daily → weekly (`W-FRI`) and recompute a **small** subset: weekly RSI(14),
weekly MACD histogram/ATR, weekly ADX, weekly EMA21 position, weekly trend state.
Then forward-fill onto daily rows.

> ⚠️ **Leakage hazard, and the reason to keep this subset small.** On Wednesday, that
> week's weekly bar is *not finished*. You must use the **last completed** weekly bar,
> i.e. shift by one week before the forward-fill. Getting this wrong hands the model
> partial knowledge of the current week — including some of the move you're trying to
> predict. `tests/test_leakage.py` must assert that every weekly feature on date *d*
> is computable from bars strictly before the Monday of *d*'s week.

**Explicitly not recommended**

- Candlestick-pattern recognizers (TA-Lib's `CDL*`, 60+ of them). Very weak standalone
  edge on daily bars, and dumping 60 sparse binary columns into a tree is an invitation
  to overfit. The CLV + range/ATR features above capture the same information
  continuously and in ~3 columns.
- Deep-learning architectures on this data volume. See §9.1.

### 7.3 Indicator redundancy — a real risk

RSI, Stochastic, CCI, Williams %R and MFI are all bounded momentum oscillators and are
**heavily mutually correlated** (|ρ| often > 0.8). Trees handle correlated inputs
without breaking, but it has two costs: feature importances get diluted and become
uninterpretable (importance splits arbitrarily among near-duplicates), and effective
model complexity rises without new information.

Mitigation, as a pipeline step:

1. Compute the full correlation matrix at `tdm features build` time and write it to
   `reports/feature_corr.html`.
2. Greedily drop one of any pair with |ρ| > 0.95, keeping the more interpretable member;
   log every drop so it's visible and reviewable, not silent.
3. Report **permutation importance on held-out folds**, not tree `feature_importances_`
   (which is impurity-based, biased toward high-cardinality features, and computed
   in-sample — it will lie to you).

### 7.4 Operationalizing Elliott Wave and Dow Theory

Both are **interpretive frameworks applied by humans**, not formulas. Fully automated
Elliott wave counting is famously ambiguous — competent analysts disagree on the count
for the same chart, and any count is revised as new bars arrive. So I will not pretend
to compute "we are in wave 3". Instead both reduce to a shared primitive, and we extract
*features* from it.

**The shared primitive: ZigZag swing pivots.**

```
pivots(bars, threshold = 3 × ATR(14)  OR  5%):
    walk forward, alternating between seeking a swing high and a swing low;
    confirm a pivot only once price has reversed by >= threshold from the extreme.
```

⚠️ **Critical: ZigZag is a lookahead trap.** Every charting-platform ZigZag is drawn
with hindsight — the pivot at bar *t* is only *knowable* at bar *t+k*, once the reversal
has actually happened. If you label pivots on the full series and feed them as features,
you leak the future and your backtest will look spectacular and be worthless. So:

> **Every pivot carries a `confirmed_at` timestamp, and features on date *d* may only
> reference pivots with `confirmed_at <= d`.** This is enforced in code (the pivot
> detector emits `(pivot_ts, confirmed_at, price, kind)` and the feature builder filters
> on `confirmed_at`) and asserted in `tests/test_leakage.py`. I consider this the
> highest-risk item in the whole feature set.

**Dow Theory features** (from confirmed pivots):

| Feature | Definition |
|---|---|
| `dow_trend` | +1 primary up (last 2 swing highs rising **and** last 2 swing lows rising), −1 primary down (both falling), 0 indeterminate |
| `dow_trend_age` | bars since `dow_trend` last changed |
| `swing_hh`, `swing_hl`, `swing_lh`, `swing_ll` | booleans on the most recent completed leg |
| `dow_volume_confirm` | +1 if volume expands in the trend direction and contracts on counter-moves (Dow's volume-confirmation principle), else −1 |
| `dow_mtf_agree` | daily `dow_trend` == weekly `dow_trend` (Dow's primary/secondary trend distinction, done properly as two timeframes) |
| `dow_sector_confirm` | ticker's `dow_trend` == its sector ETF's `dow_trend` (the modern stand-in for Dow's rails-confirm-industrials idea) |

**Elliott Wave features** (deliberately modest — ratios, not counts):

| Feature | Definition |
|---|---|
| `leg_retrace_ratio` | current leg's retracement of the previous leg, as a fraction |
| `fib_zone` | which Fibonacci band that lands in: <0.236, 0.236–0.382, 0.382–0.5, 0.5–0.618, 0.618–0.786, >0.786 (ordinal) |
| `leg_extension_ratio` | current leg length ÷ previous leg length (1.618 and 2.618 being the watched values) |
| `legs_since_major` | count of alternating legs since the last pivot exceeding 2× threshold — a weak proxy for "where in a 5-wave sequence are we" |
| `impulse_vs_corrective` | is the current leg longer and steeper than the prior counter-leg (impulse-like) or shorter and shallower (corrective-like) |
| `leg_slope_atr` | current leg's price change per bar, in ATR units |

**My honest expectation:** the Dow-Theory structure features (especially `dow_trend`,
`dow_mtf_agree`, `dow_trend_age`) should earn their place — they encode trend persistence
in a form trees split on cleanly. The Elliott/Fibonacci features I expect to be weak.
`leg_retrace_ratio` plausibly carries something real (retracement depth does relate to
continuation odds); the Fibonacci-specific zone boundaries I'd treat as a hypothesis to
be tested, not a prior to be trusted. We'll know from permutation importance on held-out
folds, which is precisely why §7.3 insists on measuring it that way. If they come out
flat, drop them — that's a successful experiment, not a failure.

### 7.5 The feature-engineering rule that matters most

> **Every feature must be scale-free and approximately stationary.**

A decision tree learns literal threshold rules: `if MACD > 1.84 then ...`. If a feature
is measured in dollars, that rule is specific to one ticker at one price level and
transfers to nothing. Train across 10 tickers over 15 years and these features
contribute noise at best. This single issue is, in my experience, the most common
reason a cross-sectional technical-indicator model fails to generalize, and it's
invisible — the model trains fine and scores fine in-sample.

Transform table:

| Raw quantity | Problem | Stored form |
|---|---|---|
| Price, EMA, BB bands, VWAP, Ichimoku lines | dollars | ratio to close (`C/x − 1`) or distance in ATRs |
| MACD, MACD histogram, MOM | dollars | ÷ ATR |
| OBV, A/D line | unbounded cumulative sum | 21-d slope, then z-score over 252 d |
| Volume | varies 1000× across tickers | ÷ 20-d median volume (log) |
| ATR | dollars | ÷ close |
| RSI, Stoch, ADX, MFI, CCI, Aroon, %B | already bounded ✓ | as-is |
| Anything with a long tail | outliers dominate splits | winsorize at 1st/99th pct, or 252-d rolling percentile rank |

Rolling z-scores and percentile ranks use **trailing windows only** (`min_periods` set,
no centering) — a centered or full-sample z-score reaches into the future. Each registry
entry declares its own warm-up length so the pipeline can drop exactly the right number
of leading rows rather than guessing.

---

## 8. Labeling: the triple-barrier method

This is where "should I go long/short so I take profit before I cut loss" becomes a
supervised learning problem. The approach is the **triple-barrier method** (López de
Prado, *Advances in Financial Machine Learning*), which fits your question almost
exactly: instead of predicting a return, it asks *which exit happens first*.

For each candidate entry bar *t*, on each side independently:

```
atr   := ATR(14)[t]                      # known at t's close — no leakage
entry := open[t+1]                       # decide at t's close, fill at next open

LONG:   TP = entry + k_tp × atr          # k_tp = 2.0
        SL = entry − k_sl × atr          # k_sl = 1.0   -> R:R 2.0
SHORT:  TP = entry − k_tp × atr
        SL = entry + k_sl × atr

scan bars t+1 .. t+H  (H = 10 sessions):
    TP touched first              -> y = 1   (win)
    SL touched first              -> y = 0   (loss)
    both touched in the same bar  -> y = 0   (see below)
    neither by t+H                -> exit at close[t+H]; y = 1 if net return > 0 else 0
```

Per sample we persist: `y`, `t1` (the bar where it resolved — **required** for purging),
`exit_reason`, `gross_return`, `net_return`, `holding_days`, and `weight`.

### Design decisions, with reasoning

**The same-bar ambiguity, resolved pessimistically.** When a daily bar's high reaches TP
*and* its low reaches SL, the OHLC does not tell you which came first — the intrabar path
is not in the data. The honest options are to resolve pessimistically (assume SL), fetch
intraday data to disambiguate, or drop the sample. **We assume SL hit first.** Resolving
these optimistically is a classic silent backtest inflator, and it's worst exactly where
it hurts most: in high-volatility bars, which is where your wide-TP trades live. Dropping
the samples instead biases the training set toward calm regimes. The count of ambiguous
bars is logged every run — if it exceeds ~5% of samples, the barriers are too tight
relative to daily range and `k_sl` should widen.

**Fill at the next open, not today's close.** Deciding on bar *t*'s close and claiming a
fill at that same close assumes you traded the closing auction with information from it.
Entering at `open[t+1]` is what you can actually do and costs a little realism in
exchange for a lot of honesty. Overnight gap risk then shows up in the labels where it
belongs.

**Costs are inside the label, not bolted on later.** `net_return` subtracts commission,
spread/slippage (modeled as a fraction of ATR, not a fixed bps — slippage scales with
volatility), and for shorts a daily **borrow cost**. Because the timeout branch labels on
`net_return > 0`, the model learns *post-cost* expectancy directly. Shorting is genuinely
more expensive and more dangerous than going long, and the labels should say so rather
than leaving you to remember it.

**Two binary models, not one three-class model.** This is the design choice I'd most
want you to scrutinize, so here is the argument. The obvious framing is a 3-class label
(LONG / SHORT / HOLD). The problem: with asymmetric barriers (TP 2×ATR, SL 1×ATR), the
long and short setups are *not* mirror images, so a single label can't represent both,
and "HOLD" has no natural definition — you'd have to invent a dead-zone threshold and
bake it into the labels, where you can never tune it again. Instead:

```
M_long  : P(long  TP before SL | features)      binary
M_short : P(short TP before SL | features)      binary
```

and HOLD emerges at decision time as *"neither probability clears its threshold"* (§12).
The thresholds then become knobs you tune against the equity curve after training, which
is exactly where you want them. It also lets each side learn its own feature
relationships — short setups really do key off different variables (volatility
expansion, breadth deterioration) than long setups.

**Sample weights for overlapping labels.** Consecutive samples share future bars (bar
*t* and *t+1* both look 10 days ahead), so they are not independent — and standard CV
treats them as if they were, which inflates every metric. Two corrections, both from the
same source as the labeling:

- **Uniqueness weight**: for each sample, average over `[t, t1]` of `1/concurrency(bar)`,
  where `concurrency(bar)` counts how many live labels span it. Passed as
  `sample_weight` so crowded periods don't dominate.
- **Purging + embargo** in cross-validation — §9.3.

**Configurable barriers.** `k_tp`, `k_sl`, `H`, and the ATR period all live in
`config.toml` as a named *label profile*, hashed into `label_run.run_id`. So
"TP 2 / SL 1 / 10 d" and "TP 3 / SL 1.5 / 20 d" are two comparable, reproducible
experiments rather than an uncommitted edit. A `tdm labels sweep` command grids over
profiles and reports base rate, mean holding period, and ambiguity rate for each — run
this *before* training, because a profile with a 15% base rate is telling you the setup
is rare and you'll be learning from very few positives.

---

## 9. The model

### 9.1 Recommendation

You said "simple decision tree model (you can suggest this as well)". My suggestion is
**both**, for two different jobs:

| Role | Model | Why |
|---|---|---|
| **Interpretability baseline** | `DecisionTreeClassifier(max_depth=3..5, class_weight='balanced', min_samples_leaf≈2%)` | You can print the entire rule set and read it. Invaluable: it tells you whether the model found something sensible ("ADX>25 and EMA-stack aligned and RSI<70 → long") or something absurd. Keep this permanently as a sanity check, not just a first step. |
| **Workhorse** | `HistGradientBoostingClassifier` (sklearn 1.9, no extra dependency) | ✅ **Handles NaN natively** — a big practical win, since every indicator has a warm-up period and 200-EMA features are NaN for the first 200 bars per ticker. Fast, strong on tabular data, built-in early stopping. |
| Alternative workhorse | `xgboost 3.4.1` | Comparable; worth it if you want `scale_pos_weight`, monotonic constraints, or GPU. Adds a dependency for a marginal gain — start with HGB. |
| Also worth trying | `RandomForest` / `ExtraTrees` | Cheap variance baseline; if it beats HGB, your HGB is overfit. |

**Why not a neural network.** 10 tickers × ~3,800 daily bars ≈ 38k rows, and after
uniqueness weighting the *effective independent* sample count is more like 2–4k. On
tabular data at that scale, gradient-boosted trees beat deep nets consistently in the
published benchmarks. The previous repo carried a `torch` dependency; I'd drop it. If
you later want sequence models, that's an argument for more tickers first, not a deeper
architecture.

**A single global model, with ticker identity excluded.** Train one model across all
tickers rather than one per ticker — per-ticker models get ~3,800 rows each and overfit.
Deliberately **omit ticker ID as a feature**: it invites the model to memorize "NVDA went
up 2015–2025", which is survivorship-flavored and won't repeat. Sector/industry as a
categorical is acceptable; HGB supports native categoricals.

### 9.2 Class balance

With TP 2×ATR / SL 1×ATR, the base win rate will be well under 50% (roughly 33% is
break-even at 2:1 before costs). Don't fight this with resampling — SMOTE on
time-series features generates samples that violate the temporal structure. Use
`class_weight='balanced'` / `sample_weight`, and **tune the decision threshold**, which
is what §12 does anyway.

### 9.3 Validation — the part that determines whether any of this is real

> **Naive K-fold cross-validation on this dataset is guaranteed to lie to you.** Two
> independent leaks: (a) shuffled folds put future bars in training and past bars in
> test; (b) even with contiguous folds, a label at the end of fold *k* resolves up to
> *H* bars into fold *k+1*, so the training label literally contains test-period prices.

`model/cv.py` implements **`PurgedWalkForwardSplit`**:

```
|== train ==|  purge  | embargo |== test ==|          fold 1
|========= train =====|  purge  | embargo |== test ==|  fold 2
...
```

- **Expanding-window walk-forward** — train on the past, test on the future, always.
- **Purge**: drop any training sample whose `[t, t1]` interval overlaps the test window.
  This is what `t1` from §8 is for.
- **Embargo**: additionally drop training samples within *E* bars before the test start
  (`E ≈ H`), because serial correlation leaks across the seam even without interval
  overlap.
- **A true final holdout**: the most recent ~18 months are *never* touched — not for
  hyperparameters, not for threshold tuning, not for feature selection — until you have
  one final model. Each peek costs you a bit of its honesty, so `tdm train` should
  refuse to read it without an explicit `--unlock-holdout` flag. The flag is there to
  make the peek a conscious decision you'll remember making.

### 9.4 Metrics — accuracy is the wrong objective

Accuracy on an imbalanced, asymmetric-payoff problem is close to meaningless; 70%
accuracy achieved by predicting HOLD always is worth nothing. Report, per fold and on
the holdout:

**Classification:** precision at the operating threshold (← the one that matters: of the
trades you actually take, how many win), recall/coverage (how many opportunities you
catch), PR-AUC (better than ROC-AUC under imbalance), and **calibration** — a reliability
curve, because §12 uses the probability as a confidence number and you need "0.7" to
actually mean 70%. Calibrate with `CalibratedClassifierCV` on a purged inner split.

**Economic — these are the real scoreboard:**

| Metric | Why it's here |
|---|---|
| **Expectancy per trade** = `p_win × avg_win − p_loss × avg_loss`, net of costs | The single number that decides whether to trade this |
| Profit factor | gross wins ÷ gross losses |
| Hit rate + avg R multiple | the two factors behind expectancy |
| Max drawdown, Calmar | whether you could actually sit through it |
| Sharpe / Sortino (annualized) | risk-adjusted, comparable to buy-and-hold |
| Trade count, avg holding days | <100 holdout trades ⇒ your estimate is noise |
| **vs. buy-and-hold SPY over the same window** | the benchmark that matters. Beating a coin flip is not the bar; beating "do nothing and own the index" is |

**Report the full distribution across folds, not the mean.** A strategy that was
excellent 2017–2019 and flat since has a fine average and is not tradeable. Per-fold
equity curves go in `reports/`.

### 9.5 Leakage checklist — `tests/test_leakage.py`

Every item here is a leak I expect this project to hit at least once. Each gets a test.

1. **No feature on date *d* may use any bar after *d*.** Property test: recompute every
   feature on a series truncated at *d*; assert the value at *d* is bit-identical to the
   full-series computation. This one test catches most of the rest.
2. **ZigZag pivots respect `confirmed_at`** (§7.4). The highest-risk item.
3. **Weekly features use the last *completed* weekly bar** (§7.2, Tier 3).
4. **Scalers/z-scores/percentiles fit on trailing windows only** — never
   `StandardScaler().fit(X_all)`.
5. **Adjustment-factor lookahead** — splits rewrite history. Since we store raw prices
   and adjust on read (§4), the test is that features computed as-of date *d* use only
   corporate actions with `ts <= d`. ⚠️ Strictly, a prospective backtest should use the
   adjustment factors *known at d*, not today's. Note this as a known, bounded
   approximation rather than pretending it's solved.
6. **Earnings dates** — today's earnings calendar is not what you knew six months ago.
   Use `confirmed` + `ingested_at` to reconstruct as-of knowledge, and treat
   `days_to_earnings` for historical rows as approximate. Document the limitation.
7. **Survivorship bias** — a universe of today's winners (NVDA, META, TSLA...) is a
   universe selected *because* they went up. **This is the one bias on the list that
   good code cannot fix**, and it will make the backtest look better than reality.
   Partial mitigations: include some has-beens (INTC is already in your list — good),
   add ETFs, add a few tickers that did badly, and keep delisted names with
   `active = 0` rather than deleting them. Mostly: know that the number is optimistic.
8. **Threshold/hyperparameter tuning on the holdout** — §9.3's `--unlock-holdout`.
9. **`t1`-based purging actually applied** — assert no training `[t, t1]` overlaps any
   test fold.

---

## 10. Backtest

A small event-driven walk-forward evaluator in `backtest/engine.py` (recall from §1 that
`vectorbt` is unavailable on 3.15 — but we'd write this anyway, because the triple-barrier
labeler already defines the exit logic and a second implementation of it would just be a
second thing to keep in sync).

```
for each session d in the test window:
    1. features as-of d's close  (from the Parquet matrix — never recomputed live)
    2. p_long, p_short := models.predict_proba
    3. apply decision rule + gates (§12) -> candidate orders
    4. position sizing: risk a fixed fraction f of equity per trade
           shares = (f × equity) / (k_sl × ATR)          # fixed fractional risk
       cap: max N concurrent positions, max exposure per sector
    5. fill at open[d+1], applying slippage = s × ATR
    6. manage open positions: TP / SL / timeout, same pessimistic
       same-bar rule as §8 (the backtest must not be more optimistic than the labels)
    7. accrue borrow cost on shorts; mark to market
```

**Cost model** (defaults in `config.toml`, all adjustable):

| Item | Default | Note |
|---|---|---|
| Commission | $0 + $0.005/share | typical US retail |
| Slippage | 0.05 × ATR per side | volatility-scaled, not fixed bps |
| Short borrow | 0.3%/yr on easy-to-borrow | accrued daily on open shorts; see §10.1 |
| Short availability | assume unavailable if `instrument.shortable = 0` | |
| Short entry blocked by SSR | skip the entry (don't fill at a worse price) | **§10.1** — this one materially changes short-side results |

**Required sensitivity runs.** A single backtest number is not evidence. Report the grid:
2× slippage, 2× commission, entry delayed one more bar, TP/SL multiples ±25%, horizon
±5 days. **A strategy whose edge vanishes when slippage doubles did not have an edge** —
it had a cost assumption. Also run the **randomized-signal control**: shuffle the
predictions and re-backtest; if random does comparably, the model contributes nothing and
the equity curve is measuring your position sizing.

### 10.1 Short-side mechanics (confirmed in scope)

You've confirmed you can short, so `M_short` is in. Shorting is not a sign flip on
longs, and the asymmetries below are modeled explicitly rather than left for you to
remember at the terminal.

> #### ⚠️ Reg SHO Rule 201 — the short-side constraint that actually bites
>
> The SEC's **alternative uptick rule** triggers when a stock declines **10% or more
> from the prior day's close**. Once triggered, short sales may only be effected at a
> price **above the current national best bid**, for the remainder of that day **and
> the entire following day**.
>
> **Why this is a real problem for this specific system, not a footnote:** a
> mean-reversion or breakdown-continuation short signal fires precisely *after* a sharp
> down day. That is exactly when SSR is active. So the setups the model most wants to
> short are disproportionately the ones you cannot get filled on at market — and since
> we enter at `open[t+1]`, a restricted open means a short-at-market order simply does
> not execute at the price the backtest assumed.
>
> If you ignore this, the backtest cheerfully books the best short entries in the
> dataset and live trading silently skips them. That single discrepancy could account
> for most of the gap between a good short backtest and a bad short track record.
>
> **Handling:** derive `ssr_active[t]` from the stored bars — true if
> `low[t] <= 0.90 × close[t-1]` (triggered today, restricted today + tomorrow) or if it
> triggered on `t-1`. Then:
> - **As a hard gate (§12):** if `ssr_active`, do not issue a market short. Report
>   `HOLD — SSR active` so you can place a limit order above the bid manually if you
>   want the trade.
> - **As a backtest rule:** skip the entry entirely rather than filling it at a worse
>   price. Skipping is the honest model of "my market order didn't fill"; a fudged worse
>   fill invents a trade that wouldn't have happened.
> - **As a feature:** `ssr_active` and `days_since_ssr` go into the matrix, since
>   "recently fell 10% in a day" is genuine regime information for both sides.
>
> Note this is derived from daily OHLC, so it's an approximation — the true trigger is
> an intraday print against the prior official close, and halts complicate it. It will
> be close enough to matter and not exact. Document it as such.

Other short-side asymmetries:

| Item | Treatment |
|---|---|
| **Borrow cost** | Accrued daily on open shorts, inside `net_return` (§8), not bolted on afterward. Default 0.3%/yr; fine for the mega-caps in your universe, wildly optimistic for small caps. If the universe expands (§14 #1), this assumption needs revisiting. |
| **Hard-to-borrow / recall** | Can't be reconstructed historically without a vendor borrow-rate feed. Treated as a known unmodeled risk: the backtest assumes every short is borrowable, which is optimistic. Mitigate by restricting shorts to high-ADV names via the liquidity gate. |
| **Unbounded loss tail** | A stop caps your loss only if price passes *through* it. A short that gaps up 30% overnight on a buyout rumor fills far above the stop. The pessimistic same-bar rule (§8) already handles intrabar ambiguity, but **gap-through-stop is a separate, real tail** — report short-side max single-trade loss separately from the aggregate drawdown, so one outlier can't hide in the average. |
| **Earnings gates matter more** | The earnings gate (§12) is doubly important for shorts: an upside earnings gap is the canonical way a short blows through its stop. |
| **Margin** | Shorting requires a margin account and ties up maintenance margin, so a short position consumes more buying power than a long of the same notional. Position sizing (§10) uses the same fixed-fractional risk for both; if margin ever becomes the binding constraint, that's a sizing change, not a model change. |

**Evaluate the two sides separately.** All of §9.4's metrics are reported per side, never
pooled. Pooling hides the most likely outcome — that longs carry a modest edge and shorts
carry none — behind a mediocre blended number. §11's warning stands: if `M_short` shows
no out-of-sample edge, shipping long/flat is the correct response, and now it's a
measured decision rather than an assumption.

---

## 11. What "good" realistically looks like

Worth fixing expectations before you see the first number, so you can tell success from
self-deception.

- At **2:1 R:R**, break-even is a **~33% hit rate** before costs, maybe ~36% after. So a
  45% hit rate is a genuinely good result here, not a mediocre one. Judge expectancy, not
  accuracy.
- Out-of-sample PR-AUC meaningfully above base rate, and precision at threshold around
  **0.50–0.60**, is a solid outcome for daily-bar technical features.
- ⚠️ **If you see >65% out-of-sample accuracy or a smooth upward equity curve on the
  first try, you have a bug, not an edge.** Go back to §9.5. This is the expected
  failure mode, not a remote one — and the previous repo's structure (no purging, no
  holdout discipline, per-day CSVs with no provenance) wouldn't have surfaced it.
- Expect the honest outcome to be **"a modest edge in some regimes, nothing in others"**.
  That's still useful: a model that reliably says HOLD during chop and sizes up in
  trending regimes has value even with an unremarkable hit rate.
- **Position sizing and risk management will affect your results more than model
  accuracy.** Fixed-fractional risk, a cap on concurrent positions, and not trading
  through earnings are worth more than several points of AUC.
- Be prepared for the result that **the shorts don't work**. US equity indices drift up;
  short setups on daily bars have a worse base rate, higher costs, and unbounded loss
  tails. If `M_short` has no edge out-of-sample, the correct response is to ship
  long/flat only. That's a finding, not a defeat.

---

## 12. The daily decision

`tdm decide --date today` is the command you'll actually run.

### Decision rule

```python
p_long  = M_long.predict_proba(x)[1]
p_short = M_short.predict_proba(x)[1]

# Hard gates — these override the model, because the model cannot see them
if earnings_within(horizon_days):      return HOLD, "earnings in N days"
if adv_20d < min_liquidity:            return HOLD, "illiquid"
if atr/close > max_vol:                return HOLD, "volatility too extreme to size"
if open_positions >= max_concurrent:   return HOLD, "position limit"
if data_staleness > 1 session:         return ABSTAIN, "stale data"   # fail loud

# Short-specific gates (§10.1) — shorting is confirmed in scope, but these
# three are the difference between a signal and a fillable order
if side == SHORT:
    if not instrument.shortable:       return HOLD, "not shortable"
    if ssr_active:                     return HOLD, "SSR active — needs a limit above the bid"
    if adv_20d < min_short_liquidity:  return HOLD, "borrow risk: ADV too low to assume easy-to-borrow"

# Model
if p_long >= θ_long and p_long > p_short:    action = LONG
elif p_short >= θ_short and p_short > p_long: action = SHORT
else:                                         action = HOLD
```

`θ_long` and `θ_short` are tuned on **validation folds only** to maximize expectancy
(not accuracy), then frozen. They are stored with the model artifact — a threshold that
drifts between runs makes your track record uninterpretable.

The **data-staleness ABSTAIN** is deliberate: if the fetch failed and the newest bar is
three days old, the system must refuse to answer rather than quietly decide on stale
prices. Silent degradation is the worst failure mode for a tool you check once a day and
trust.

### Output

Levels come from the same ATR arithmetic as the labels, so what you see is exactly what
the model was trained on:

```
entry_ref = last_close           (indicative; actual fill = next open)
TP        = entry ± k_tp × ATR14
SL        = entry ∓ k_sl × ATR14
shares    = (risk_frac × equity) / (k_sl × ATR14)
```

Written to: `reports/decisions_YYYY-MM-DD.{md,csv}`, a `rich` table on stdout, and the
`decision` table in SQLite.

### Scoring yourself — the feature that makes this trustworthy over time

```bash
tdm score-decisions --since 2026-01-01
```

Walks every past row in `decision`, resolves it against bars that have since arrived,
fills `decision_outcome`, and prints realized hit rate / expectancy / drawdown **of the
decisions the system actually issued**.

This is the only genuinely out-of-sample measurement you will ever have — no purging
subtleties, no leakage, no survivorship. It's also the thing that will tell you if a
backtest was fooling you. Build it early, not last, and let it accumulate: after six
months it is worth more than any amount of backtesting.

---

## 13. Build order

Each phase ends somewhere useful, so you can stop or re-scope without a half-finished
system.

| Phase | Deliverable | Done when |
|---|---|---|
| **0 · Scaffold** | venv 3.15, `CLAUDE.md`, Makefile, `.gitignore`, CI-less test runner | `make venv && make test` green; venv guard fires outside the venv |
| **1 · Store + ingest** | schema, migrations, yfinance adapter, incremental sync | `make fetch` twice ⇒ second run makes **zero** HTTP calls; `tdm db verify` clean. **Smoke-test yfinance against pandas 3.1 here** (§1) |
| **2 · Indicators** | registry + Tier-1/your-list features, Parquet output, CSV export | spot-checks match TradingView; `test_indicators.py` green; correlation report produced |
| **3 · Structure features** | ZigZag + Dow/Elliott features with `confirmed_at` | `test_leakage.py` pivot tests green |
| **4 · Labels** | triple barrier, `t1`, weights, `tdm labels sweep` | base rate / holding / ambiguity rate sane across profiles |
| **5 · Baseline model** | depth-4 decision tree + purged walk-forward CV | the printed rule set is *readable and not absurd* |
| **6 · Workhorse** | HGB + calibration + permutation importance + threshold tuning | beats the depth-4 tree on expectancy, on validation folds |
| **7 · Backtest** | engine, costs, sensitivity grid, shuffled-signal control | survives 2× slippage; beats shuffled control |
| **8 · Decide** | `tdm decide`, gates, reports, `decision` table | one command, end to end |
| **9 · Self-scoring** | `tdm score-decisions` | reproduces phase-7 numbers on the overlap window |
| **10 · Holdout** | one single evaluation on the untouched 18 months | **go / no-go** |

Phases 1 and 9 are the two that make this different from the previous repo. If you only
get through phase 5, you'll still have a clean cached dataset and a readable baseline —
which is already more than enough to iterate on.

---

## 14. Open questions for your review

1. **Universe.** Keep the previous nine (AMD, AMZN, GOOGL, AAPL, INTC, META, NFLX, NVDA,
   TSLA)? I'd argue for **expanding to 50–100 liquid names plus sector ETFs** — more
   independent samples is the cheapest available improvement to statistical power, and
   it directly dilutes the survivorship problem in §9.5(7). Nine mega-cap tech names are
   also highly correlated, so your effective sample size is far below 9× one ticker.
2. **History depth.** 15 years (includes 2015/2018 corrections, the 2020 crash, 2022 bear
   market — valuable regime diversity) or 10 (more similar to the present regime)? I lean
   15, with regime features so the model can condition on it.
3. **TP/SL profile.** Start at TP 2.0 / SL 1.0 / H=10? `tdm labels sweep` will tell us if
   a different profile has a healthier base rate before we commit.
4. ~~**Are you actually able to short?**~~ **RESOLVED 2026-11-03: yes.** `M_short` is in
   scope, and the short-side mechanics — Reg SHO Rule 201, borrow cost, gap-through-stop
   tail risk — are specified in **§10.1**. Remaining sub-question: do you have a
   **margin account with easy-to-borrow access** at your broker, and do you know your
   actual borrow rate? The 0.3%/yr default is reasonable for mega-caps and badly wrong
   for anything small — it matters once the universe expands (#1).
5. **Earnings data source.** `yfinance` earnings dates are unreliable and have no
   as-of history. Options: accept the approximation (documented in §9.5(6)), scrape and
   snapshot forward from today so the data becomes trustworthy going forward, or pay for
   a vendor. Given that earnings gaps are the top killer of swing trades, I'd at least
   start snapshotting from day one — it costs little and compounds.
6. **Equity / risk-per-trade** for position sizing in the backtest. Defaults: $10,000,
   1% risk per trade, max 5 concurrent positions.
7. **Weekly as a separate model?** Current design uses weekly features inside the daily
   model. The alternative — a dedicated weekly-bar model — gets ~1/5 the samples. I
   recommend features-only, as designed.
