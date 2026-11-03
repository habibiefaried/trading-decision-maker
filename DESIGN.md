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

### As a system, it's five stages — a stateless pipeline

| Stage | What it is, in engineering terms | Cost | Idempotent? |
|---|---|---|---|
| **1. Fetch** | `yf.download()` 9 years of daily OHLCV for your ticker list | ~1–3 min for 30–60 tickers | Yes — just re-downloads |
| **2. Features** | `map` ~100 pure functions over each series → one wide numeric matrix | Seconds | Yes — pure |
| **3. Chart** | Render price + every indicator to interactive HTML so you can *read* it | Seconds | Yes |
| **4. Train** | Generate labels by simulation, then fit a classifier | Minutes | Yes, with a fixed seed |
| **5. Decide** | Run inference on today's row, write the output table | Milliseconds | Yes |

**There is no database.** The pipeline is a pure function from `(ticker list, date range)` to
`(charts, model, decisions)`. Every stage re-derives its input. §5 explains why that's
sound here and what the one real cost is.

### Three things make this harder than a normal ETL pipeline

Worth stating up front, because they drive most of the design decisions:

1. **The input data is not quite stable between runs.** Re-downloading tomorrow gives you
   a slightly different dataset: new bars at the tail, and occasionally a rescaled history
   when a stock splits. §5 shows why this is harmless for the *features* (they're
   invariant) but matters for *comparing two models*.
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
| History depth | **9 years** (~2,268 bars/ticker) | Reaches back to late 2017. Covers the Feb-2018 volatility spike, the 2020 COVID crash and the 2022 bear market — §10.1 |
| Style | **Swing trading** | Hold across days, not intraday, not months |
| Horizon | **10 trading days max** | This is the timeout barrier — explained in §9 |
| TP / SL | **2.0 × ATR / 1.0 × ATR** | |
| Both directions? | **Yes.** CFDs make shorting trivial | §11.1 |
| Weekly-timeframe features | **Dropped** | User's call; removes a whole class of leakage bug |
| Earnings-date data | **Optional, default off** | OHLCV-only is the v1 target. ⚠️ But see the leverage warning in §11.1 |
| Python | **3.14** | Changed from 3.15: CatBoost has no cp315 wheel — §3 |

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

## 3. Environment: Python 3.14 + venv

### Verified package availability

Checked against PyPI wheel tags rather than assumed. Every package below has a `cp314`
wheel or is pure-Python:

| Package | Version | Role |
|---|---|---|
| numpy | 2.5.4 | arrays |
| pandas | 3.1.0 | dataframes — **major version, breaking, see below** |
| scikit-learn | 1.9.1 | RandomForest, HistGradientBoosting, CV, calibration |
| **catboost** | **1.2.10** | **primary model (§10.1)** — macOS wheels are `universal2`, so Apple Silicon is covered |
| xgboost | 3.4.1 | alternative, for monotonic constraints |
| TA-Lib (`ta-lib`) | 0.8.3 | indicator engine (C, Cython-wrapped). Wheels self-contained — no `brew install ta-lib` |
| plotly | 7.1.0 | interactive HTML charts (§7) |
| pyarrow | 26.0.1 | Parquet snapshots |
| yfinance | 1.7.0 | data source |
| exchange-calendars | 4.13.2 | market session calendar |
| shap | 0.54.1 | model explanation (optional) |
| typer / rich / pydantic-settings / pytest | — | CLI, output, config, tests |

### Why 3.14 and not 3.15

The project originally targeted 3.15. **CatBoost ships wheels only up to `cp314`** — there
is an sdist, but CatBoost builds from source through its own toolchain and that is a
genuinely painful afternoon. Since CatBoost is the primary model (§10.1) and 3.14 is
stable with every other dependency available, 3.14 is the version.

Nothing in the design depended on 3.15; it was a preference, not a requirement.

**Still ruled out on 3.14:**

| Package | Why |
|---|---|
| `pandas-ta` | pins `numba==0.61.2`, which has no `cp314` wheel either. **TA-Lib** instead — faster anyway (Cython, 2–4× the old SWIG bindings). |
| `vectorbt` | *Technically installable* on 3.14 (`requires-python <3.15`), but still **not used** — §11 explains why: the §9 labeler already defines the exit logic, and a second framework-provided implementation of it would be a second thing to keep in sync. One implementation, shared. |

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
is pyenv 3.13.1 — the wrong interpreter:

1. **`.python-version`** → `3.14`
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
brew install python@3.14                  # already installed on this machine
/opt/homebrew/bin/python3.14 -m venv venv
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
├── .python-version              # 3.14
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
│   ├── ingest/                  # ── the only module that touches the network
│   │   ├── fetch.py             #    yf.download wrapper + retries (§6)
│   │   ├── validate.py          #    `universe validate` + OHLCV sanity checks
│   │   └── sessions.py          #    market-calendar wrapper
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
│   │   └── engine.py            #    §11
│   │
│   ├── report/                  # ── §7 — visual verification
│   │   ├── charts.py            #    Plotly multi-panel price + indicators
│   │   ├── overlays.py          #    label brackets, decision markers, pivots
│   │   └── diagnostics.py       #    correlation heatmap, feature distributions
│   │
│   ├── provenance.py            #    append to runs.jsonl (replaces the *_run tables)
│   └── decide.py                #    §13 — production inference path
│
├── scripts/bootstrap.sh
├── tests/
│   ├── test_indicators.py       # vs TA-Lib / hand-computed fixtures
│   ├── test_leakage.py          # ← the most important tests in the repo (§10.5)
│   ├── test_invariance.py       # inject a split, assert past features unchanged (§6.1)
│   ├── test_triple_barrier.py
│   └── test_incremental_fetch.py
│
├── data/                        # GITIGNORED
│   ├── raw/                     #   optional OHLCV snapshot, for offline work
│   ├── features/                #   Parquet — the snapshot that matters (§5)
│   ├── labels/                  #   Parquet
│   └── exports/                 #   CSV, for eyeballing in Excel
├── models/                      # GITIGNORED
├── reports/                     # GITIGNORED — charts/, equity.html, decisions
└── runs.jsonl                   # GITIGNORED — append-only provenance log
```

### Layering rule

```
cli  ->  decide / train / backtest / report  ->  features / labels  ->  ingest
```

Dependencies point one way only. Two consequences worth enforcing in review:

- **`features/` and `labels/` do no I/O and import nothing from `ingest/`.** They're pure
  functions over DataFrames. That's what makes them unit-testable without a network or a
  database, which in turn is what makes §10.5's leakage tests possible.
- **`ingest/` is the only module that makes HTTP calls.** All retries and validation of
  provider output are contained there. Everything downstream sees clean DataFrames.
- **`report/` reads the feature matrix; it never computes indicators.** §7 explains why
  a chart that draws its own indicators is worse than no chart.

### What went wrong in the previous attempt

The user's earlier project (`~/code/stock-market-entry-signal-tpsl-specific`) had a 52 KB
`fetch_stock_data.py`, a 44 KB `train.py`, and a `data/` directory accumulating files like
`AAPL_tpsl_data_20260610.csv`, one per ticker per day. Two concrete failure modes that
this layout exists to fix:

1. **No cache.** Every run re-downloaded everything, so iteration was slow and any run
   could fail outright whenever yfinance happened to be down. → §7.
2. **No provenance.** Given `models/xyz.pkl` there was no way to recover which features,
   which parameters, or which date range produced it. **So you couldn't meaningfully
   compare two models** — the central activity of the project. → the `*_run` tables in §5.

---

## 5. No database — and why that's sound here

The original plan had SQLite with a schema, migrations, coverage tracking, and gap
bookkeeping, so that a bar was never downloaded twice. **That's deleted.** yfinance is
reliable enough and fast enough that re-downloading is cheaper than the machinery needed
to avoid it. This section records what that buys, what it costs, and the one mitigation
worth keeping.

### What gets deleted

| Removed | Why it existed | Why it's unnecessary now |
|---|---|---|
| `bar_daily`, `corporate_action` tables | durable cache of expensive fetches | fetch is cheap and reliable |
| `fetch_coverage`, `known_gap` | work out which dates are missing, never re-request | we always request everything |
| the sync planner (range coalescing, gap healing, restatement lookback) | incremental fetching | no increments — it's a full refresh |
| migrations, `STRICT` schema, WAL pragmas | schema evolution | no schema |
| **raw-price storage + adjust-on-read** | a cached *adjusted* price goes stale when a split rewrites history | ⬇ see below — this one is the interesting deletion |

### The adjustment problem deletes itself

The most intricate part of the old design existed purely *because* we cached. Recall the
failure it prevented: adjusted prices are a function of all corporate actions *since*, so
a split silently rewrites them — and rows written before the split end up on a different
scale from rows written after, in the same table, with no error raised.

**With no cache, that failure is impossible.** Every run downloads one internally
consistent series. So we can simply use `auto_adjust=True` and let Yahoo do the
adjustment:

```python
df = yf.download(
    tickers,                 # the whole list in one call
    start=start, end=end,
    interval="1d",
    auto_adjust=True,        # ← now safe, because nothing persists
    actions=False,
    group_by="ticker",
    multi_level_index=False,
    threads=True,
    progress=False,
)
```

That removes `corporate_action`, the cumulative-adjustment-factor code, the inverse volume
adjustment, and the `as_of` point-in-time read path. A genuinely large simplification.

### Why re-downloading doesn't change your features

The obvious worry: if a stock splits between two runs, the second run's prices are all
rescaled. Doesn't that change the training data?

**No — and this is the result that makes the whole no-database approach sound.** A split
rescales every pre-split price by the same constant `k`, and every feature in §8.5 is a
*ratio*, so `k` cancels:

| Feature | Under a rescale by `k` | Result |
|---|---|---|
| `close/EMA200 − 1` | `(k·C)/(k·EMA200) − 1` | **invariant** |
| `ATR/close` | both scale by `k` | **invariant** |
| `%B` = `(C−lo)/(hi−lo)` | all three scale by `k` | **invariant** |
| `MACD/ATR` | both scale by `k` | **invariant** |
| RSI, Stochastic, ADX | built from price *ratios* | **invariant** |
| `volume/median_volume` | both scale by `1/k` | **invariant** |
| OBV slope, z-scored | scales by `1/k`, z-score normalizes | **invariant** |

> **§8.5's normalization rule was adopted so a decision tree's thresholds would transfer
> across tickers. It turns out to also make the feature matrix immune to re-downloading.**
> Scale-free and corporate-action-invariant are the same property. That convergence is
> what lets us throw away the storage layer without losing anything.

**This must be a test, not an assumption:** `tests/test_invariance.py` injects a synthetic
10:1 split into a series, recomputes all features, and asserts every value before that
date is bit-identical. Any feature that fails is not scale-free — which means it was also
broken for §8.5's original reason. One assertion, two bug classes.

### The one real cost: run-to-run reproducibility

What *does* differ between two runs is the tail (new bars arrive daily) and the rare
provider restatement of a recent bar. That's harmless for daily use and corrosive for one
specific activity: **comparing two models.**

```
Monday:    train model A, holdout expectancy 0.31
Friday:    train model B, holdout expectancy 0.28
           -> is B worse, or did the dataset change underneath you?
```

With a moving dataset you can't tell, and model comparison is the central activity of this
project. So one thing survives from the old design:

> **Snapshot the feature matrix, not the bars.** `tdm features build` writes
> `data/features/<run_id>.parquet` where `run_id = hash(tickers, date range, feature-set
> config, code version)`, and `tdm train` records which `run_id` it consumed. Training
> against a snapshot is the default; `--refresh` forces a new fetch.

That's a file and a hash — no schema, no migrations, no invalidation (per the invariance
result above, a snapshot is correct forever). It costs ~20 lines and makes two model runs
actually comparable, which is the difference between measuring and guessing.

### What's on disk

```
data/
├── raw/<run_id>.parquet          # optional: the downloaded OHLCV, for offline work
├── features/<run_id>.parquet     # the feature matrix  ← the snapshot that matters
├── labels/<run_id>.parquet       # y, t1, weights
└── exports/*.csv                 # for eyeballing in Excel
models/<run_id>/                  # fitted model + frozen thresholds + metrics.json
reports/
├── charts/<ticker>.html          # §7
└── decisions_YYYY-MM-DD.md
runs.jsonl                        # append-only provenance log (replaces the *_run tables)
```

`runs.jsonl` is one JSON object per pipeline run — tickers, date range, config hash, code
version, artifact paths, metrics. Append-only, greppable, diffable, no schema. It's the
provenance the previous project lacked, at about 1% of the complexity of the SQLite
version.

---

## 6. Fetching

One module, `ingest/fetch.py`, and it stays small.

```python
def fetch(tickers: list[str], years: int = 9) -> dict[str, pd.DataFrame]:
    """Download daily OHLCV. Returns {ticker: frame}, adjusted, indexed by date."""
```

### Benchmarks are appended automatically

You supply only the tickers you want to **trade** in `config/universe.txt`. The
relative-strength and market-regime features (§8.2) need SPY and the relevant sector ETFs,
so `fetch` adds them itself and tags them `tradeable=False`. They flow through features
exactly like any other series and are filtered out at the decision step (§13).

So "no ETFs" (§1) stays true of your decisions, the features still work, and you never
have to remember to add SPY by hand.

### Validate before downloading

```bash
tdm universe validate
```

9 years × 60 tickers is a few minutes; you don't want to discover at ticker 48 that three
were typos. Checks each symbol and reports:

| Check | Why |
|---|---|
| Symbol resolves | Catches typos and wrong suffixes — Yahoo wants `BRK-B`, not `BRK.B` |
| Has ≥ 9 years of history | ⚠️ A 2024 IPO can't supply 9 years. Flagged, not rejected — but a ticker with 300 bars has NaN for every 200-day feature and contributes almost nothing |
| Not delisted | |
| Median volume above the liquidity floor | A thin stock just gets gated out at decision time (§13), so it's dead weight |
| Duplicates, blank lines, whitespace | |

### Practical details that still matter

- **One batched call.** `yf.download()` accepts the whole ticker list and parallelises
  internally (`threads=True`). Don't loop one ticker at a time.
- **`multi_level_index=False`.** yfinance returns a MultiIndex column frame by default for
  multiple tickers; flattening at the boundary keeps the rest of the code simple.
- **Retries.** Even a reliable service has transient failures. Wrap the call in 3 attempts
  with exponential backoff. If a ticker still fails, **drop it with a loud warning and
  continue** — one bad symbol shouldn't kill a 60-ticker run.
- **Validate what comes back** before computing anything on it: `high >= low`, `high >=
  open/close`, no negative prices, no duplicate dates, index is sorted, no all-zero-volume
  stretches. Cheap, and it catches provider glitches before they become confusing
  indicator values.
- ⚠️ **Smoke-test yfinance against pandas 3.1 first** (§3). It declares `pandas>=1.3.0`
  with no upper bound, so it *installs* next to pandas 3.1, which is not the same as
  working there. This is the first thing to verify in phase 1.

---

## 7. The HTML chart

A visual read of every ticker with all of its indicators, written to self-contained
interactive HTML.

```bash
tdm chart --ticker NVDA                  # -> reports/charts/NVDA.html
tdm chart --all                          # every ticker in the universe
tdm chart --ticker NVDA --with-labels    # overlay the triple-barrier outcomes
tdm chart --ticker NVDA --with-decisions # overlay the model's calls
```

**Library: Plotly** (7.1.0, pure-Python). `go.Candlestick` +
`make_subplots` + `write_html` gives one self-contained file with zoom, pan and hover
tooltips, no server and no build step. Bokeh is comparable; `lightweight-charts` looks
more like TradingView but needs JS plumbing; matplotlib/mplfinance give static PNGs, which
are much less useful for actually *reading* a series.

### Layout

```
┌────────────────────────────────────────────────────────────┐
│ NVDA   2017-11-03 → 2026-11-03   9y daily                  │
├────────────────────────────────────────────────────────────┤
│ PRICE  candlesticks                                   60%  │
│        + EMA 8/21/50/200        + Bollinger(20,2)          │
│        + Keltner  + Supertrend  + Ichimoku cloud           │
│        + anchored VWAP          + ZigZag pivots ◆          │
│        + Donchian / 52w high                               │
│        [--with-labels]    TP/SL brackets, green=win red=loss│
│        [--with-decisions] ▲ LONG  ▼ SHORT  · HOLD          │
├────────────────────────────────────────────────────────────┤
│ VOLUME bars, coloured by up/down + 20d median line     8%  │
├────────────────────────────────────────────────────────────┤
│ RSI(14) + RSI(2), bands at 30/70                       8%  │
├────────────────────────────────────────────────────────────┤
│ MACD/ATR line + signal + histogram                     8%  │
├────────────────────────────────────────────────────────────┤
│ ADX(14) + (+DI − −DI), bands at 20/25                  8%  │
├────────────────────────────────────────────────────────────┤
│ Stochastic %K/%D  ·  MFI  ·  OBV slope z-score          8%  │
└────────────────────────────────────────────────────────────┘
      shared, linked x-axis — zoom one panel, all follow
```

### Three design rules

**1. Render from the feature matrix, never recompute.**

```python
# RIGHT: chart the exact numbers the model will see
features = pd.read_parquet(f"data/features/{run_id}.parquet")
plot(features.loc[ticker])

# WRONG: a second implementation that can silently disagree
plot(talib.RSI(bars.close, 14))
```

> This is the whole point of the chart. If it draws its own indicators, then when a value
> looks wrong you don't know whether the bug is in the feature pipeline or the plotting
> code — and worse, the chart can look perfect while the model trains on garbage. Charting
> the actual feature columns makes the picture **evidence** about the model's input. Same
> principle as §11's "one implementation, shared" rule.

**2. Visual verification catches what unit tests don't.**

Unit tests confirm RSI(14) matches a reference value. They do not catch: an indicator
off by one bar, a ZigZag pivot drawn with hindsight (§8.4 — *this is visible instantly on
a chart and nearly invisible in a test*), a warm-up period filled with zeros instead of
NaN, a feature that's flat for 200 bars, or an anchored VWAP anchored to the wrong pivot.
Thirty seconds of looking at `NVDA.html` is one of the highest-value debugging steps in
the project, and it's why this is stage 3 — **before** training, not after.

**3. Overlay the labels, because the labeler is the hardest part to verify.**

`--with-labels` draws each labeled entry as a TP/SL bracket, coloured by outcome. This is
how you sanity-check §9 by eye:

- Are TP/SL distances sane relative to the candles, or absurdly wide/narrow?
- Do the winners look like they were genuinely going somewhere?
- Is the ambiguous-bar count (TP and SL in the same bar) visibly high?
- During March 2020, are the brackets enormous? **They should be** — ATR exploded, so
  ATR-scaled barriers widened with it. Seeing that confirms §10.1's `max_vol` gate is
  needed and that the ATR scaling works as intended.

### Also worth generating

| Report | Content |
|---|---|
| `reports/feature_corr.html` | correlation heatmap — drives the dedup step in §8.3 |
| `reports/feature_dist.html` | per-feature histogram + NaN count + trailing-z sanity. Catches a feature that's constant, or 90% NaN, in one glance |
| `reports/equity.html` | backtest equity curve per fold (§11), plus the shuffled-signal control |
| `reports/tree.txt` | the depth-4 baseline's rule set as plain text (§10.1) — readable in a terminal |

---

## 8. Feature engineering

### 8.0 The complete feature list

Everything the system computes, in one place. Scattered across §8.1–§8.4 below is the
*reasoning*; this is the build spec.

#### The swing-trading logic these features serve

The list isn't a grab bag — it exists to answer six questions in sequence, which is what
a swing trade actually is:

```
1. Is this stock trending?                → TREND        (is there anything to ride?)
2. Is it stronger than the market?        → RELATIVE      (is it the best horse?)
3. Has it pulled back enough to enter?    → MOMENTUM      (entry timing)
4. Is volume confirming the move?         → VOLUME        (is the move real?)
5. Is volatility sane enough to size it?  → VOLATILITY    (can I risk-manage it?)
6. Is the market regime favourable?       → MARKET        (should I be trading at all?)
```

Each family maps to one question. If a candidate feature doesn't help answer one of them,
it probably shouldn't be in the list.

> #### ⚠️ Start with the CORE set, not all of it
>
> The full list is **~115 features**. Against ~11,350 effective samples (§10.1) that's
> ~100 samples per feature — thin, and a direct invitation to overfit.
>
> So the list is split into **CORE (38)** and **EXTENDED (~77)**. Build and validate CORE
> first (phase 2), and only add EXTENDED features once CORE shows signal above base rate.
> If CORE shows nothing, more features will not rescue it — they'll just make the
> overfitting harder to detect.
>
> `config/feature_sets/core.toml` and `extended.toml` select between them.

#### 1 · TREND — "is there anything to ride?"

| Feature | Params | Stored as | Warm-up | Tier |
|---|---|---|---|---|
| `close_vs_ema8/21/50/200` | 8, 21, 50, 200 | `close/EMA − 1` | 200 | **CORE** (21, 50, 200) |
| `ema_stack` | 8/21/50/200 | ordinal 0–4: how many are in ascending order | 200 | **CORE** |
| `ema21_slope_atr` | 21, 5-bar slope | slope ÷ ATR | 26 | **CORE** |
| `ema_cross_age` | 8×21 | signed bars since last cross | 21 | EXT |
| `adx_14` | 14 | raw 0–100 ✓ | 28 | **CORE** |
| `di_diff` | 14 | `(+DI − −DI)` | 28 | **CORE** |
| `aroon_up/down/osc` | 25 | raw 0–100 ✓ | 25 | EXT |
| `supertrend_state` | 10, 3×ATR | −1 / +1 | 24 | **CORE** |
| `supertrend_dist_atr` | 10, 3×ATR | distance to flip ÷ ATR | 24 | EXT |
| `ichimoku_tenkan/kijun_pos` | 9, 26 | `close/line − 1` | 52 | EXT |
| `ichimoku_cloud_pos` | 9,26,52 | above / inside / below (ordinal) | 78 | EXT |
| `ichimoku_cloud_thick_atr` | 9,26,52 | thickness ÷ ATR | 78 | EXT |
| `linreg_slope_atr_21/63` | 21, 63 | regression slope ÷ ATR | 63 | EXT |
| `linreg_r2_21/63` | 21, 63 | R² 0–1 ✓ — *trend quality*, not direction | 63 | EXT |

#### 2 · RELATIVE STRENGTH & MARKET REGIME — "best horse? should I trade at all?"

Computed against SPY and sector ETFs, which are fetched but never traded (§6).

| Feature | Params | Stored as | Warm-up | Tier |
|---|---|---|---|---|
| `rs_spy_21/63/126` | 21, 63, 126 | ROC of (close/SPY) | 126 | **CORE** (21, 63) |
| `rs_rank` | 63 | **cross-sectional percentile** of `rs_spy_63` across the universe, per day | 63 | **CORE** |
| `rs_sector_63` | 63 | ROC of (close/sector ETF) | 63 | EXT |
| `beta_spy_63` | 63 | regression beta | 63 | EXT |
| `corr_spy_63` | 63 | correlation −1..1 ✓ | 63 | EXT |
| `spy_above_ema200` | 200 | 0/1 | 200 | **CORE** |
| `spy_vol_rank` | 21, 252 | percentile of SPY realized vol | 252 | **CORE** |
| `vix_level`, `vix_rank` | 252 | raw, and percentile | 252 | **CORE** (`vix_rank`) |
| `vix_term_structure` | — | VIX3M / VIX — <1 signals stress | 1 | EXT |
| `breadth_above_ema50` | 50 | fraction of universe above own EMA50 | 50 | EXT |

> `rs_rank` is the one feature here that needs the **whole universe at once** rather than
> one ticker at a time — it's a cross-sectional rank. Worth noting because it makes the
> feature pipeline not purely per-ticker, and it's easy to get the groupby wrong.

#### 3 · MOMENTUM / MEAN REVERSION — "entry timing"

| Feature | Params | Stored as | Warm-up | Tier |
|---|---|---|---|---|
| `rsi_14` | 14 | raw 0–100 ✓ | 14 | **CORE** |
| `rsi_2` | 2 | raw 0–100 ✓ — short-term pullback depth | 2 | **CORE** |
| `rsi_divergence` | from pivots | −1 / 0 / +1 (§8.4) | ~60 | EXT |
| `stoch_k`, `stoch_d` | 14,3,3 | raw 0–100 ✓ | 17 | **CORE** |
| `stoch_kd_diff` | 14,3,3 | `%K − %D` | 17 | EXT |
| `macd_hist_atr` | 12,26,9 | histogram ÷ ATR | 35 | **CORE** |
| `macd_line_atr`, `macd_signal_atr` | 12,26,9 | ÷ ATR | 35 | EXT |
| `macd_cross_age` | 12,26,9 | signed bars since cross | 35 | EXT |
| `ppo` | 12,26 | `(EMA12−EMA26)/EMA26 × 100` — **inherently scale-free**, a cleaner MACD | 26 | EXT |
| `roc_5/10/21/63` | 5,10,21,63 | percentage ✓ | 63 | **CORE** (21, 63) |
| `cci_20` | 20 | raw ✓ | 20 | EXT |
| `williams_r_14` | 14 | raw −100..0 ✓ | 14 | EXT |

#### 4 · VOLATILITY — "can I size this safely?"

| Feature | Params | Stored as | Warm-up | Tier |
|---|---|---|---|---|
| `atr_pct` | 14 | `ATR/close` | 14 | **CORE** |
| `atr_pct_rank` | 14, 252 | trailing percentile | 252 | **CORE** |
| `bb_pctb` | 20, 2σ | `(C−lower)/(upper−lower)` | 20 | **CORE** |
| `bb_bandwidth` | 20, 2σ | `(upper−lower)/mid` | 20 | **CORE** |
| `bb_bandwidth_rank` | 20, 252 | percentile — low = squeeze | 252 | **CORE** |
| `keltner_pos` | 20, 2×ATR10 | position in channel | 20 | EXT |
| `squeeze_on` | BB vs Keltner | 0/1 — BB inside Keltner | 20 | EXT |
| `donchian_pos_20/55` | 20, 55 | position in channel 0–1 ✓ | 55 | **CORE** (20) |
| `pct_from_52w_high` | 252 | `close/max(high,252) − 1` | 252 | **CORE** |
| `pct_from_52w_low` | 252 | `close/min(low,252) − 1` | 252 | EXT |
| `realized_vol_21` | 21 | stdev of log returns, annualized | 21 | EXT |
| `parkinson_vol_21` | 21 | high/low range estimator — **more efficient than close-to-close**, which matters when you only have 9 years | 21 | EXT |
| `vol_of_vol` | 21, 63 | stdev of `atr_pct` | 63 | EXT |

#### 5 · VOLUME — "is the move real?"

| Feature | Params | Stored as | Warm-up | Tier |
|---|---|---|---|---|
| `volume_rel_median` | 20 | `log(volume / median20)` | 20 | **CORE** |
| `volume_z` | 63 | trailing z-score | 63 | EXT |
| `obv_slope_z` | 21 slope, 252 z | ⚠️ never raw OBV (§8.1 #6) | 252 | **CORE** |
| `obv_price_divergence` | 21 | sign disagreement between OBV slope and price slope | 252 | EXT |
| `mfi_14` | 14 | raw 0–100 ✓ — volume-weighted RSI | 14 | **CORE** |
| `ad_slope_z` | 21, 252 | Chaikin A/D line slope, z-scored | 252 | EXT |
| `cmf_20` | 20 | Chaikin Money Flow, bounded ✓ | 20 | EXT |
| `close_vs_rvwap_20/50` | 20, 50 | `close/rolling VWAP − 1` | 50 | **CORE** (20) |
| `close_vs_avwap_swinglow` | anchored | `close/AVWAP − 1` from last confirmed swing low | ~60 | **CORE** |
| `close_vs_avwap_swinghigh` | anchored | same, from last swing high | ~60 | EXT |
| `close_vs_avwap_ytd` | anchored | from 1 Jan | ≤252 | EXT |

> ⚠️ **No plain daily VWAP.** It's an intraday metric and degenerates to the bar's typical
> price on daily data — §8.1 #8 has the algebra. Rolling and anchored variants replace it.

#### 6 · PRICE STRUCTURE — Dow Theory & Elliott, from ZigZag pivots (§8.4)

All of these depend on the pivot detector, and **all must respect `confirmed_at`** — this
is the highest-leakage-risk group in the whole list.

| Feature | Stored as | Tier |
|---|---|---|
| `dow_trend` | −1 / 0 / +1 from swing-high and swing-low sequences | **CORE** |
| `dow_trend_age` | bars since it last changed | **CORE** |
| `dow_trend_major` | same, from a wider ZigZag (8×ATR vs 3×ATR) | EXT |
| `dow_mtf_agree` | `dow_trend == dow_trend_major` | EXT |
| `swing_hh/hl/lh/ll` | 4 booleans on the latest completed leg | EXT |
| `dow_volume_confirm` | −1 / +1 | EXT |
| `dow_sector_confirm` | ticker trend == sector ETF trend | EXT |
| `dist_to_swing_high_atr` | distance to last confirmed swing high ÷ ATR | **CORE** |
| `dist_to_swing_low_atr` | same, swing low | EXT |
| `leg_retrace_ratio` | current leg's retracement of the previous, as a fraction | EXT |
| `fib_zone` | ordinal band: <.236 / .236–.382 / .382–.5 / .5–.618 / .618–.786 / >.786 | EXT |
| `leg_extension_ratio` | current leg ÷ previous leg | EXT |
| `legs_since_major` | alternating legs since the last major pivot | EXT |
| `impulse_vs_corrective` | −1 / +1 | EXT |
| `leg_slope_atr` | leg price change per bar ÷ ATR | EXT |

#### 7 · BAR SHAPE, GAPS & CALENDAR

| Feature | Stored as | Tier |
|---|---|---|
| `clv` | `(close−low)/(high−low)` — where in the day's range it closed | **CORE** |
| `range_atr` | `(high−low)/ATR` | EXT |
| `gap_atr` | `(open − prev close)/ATR` | **CORE** |
| `consec_updown` | signed run length of up/down closes | EXT |
| `days_since_large_gap` | bars since a gap exceeded 3×ATR | EXT |
| `quarterly_cycle_pos` | position in the ~63-bar cycle since the last large gap | EXT |
| `large_gap_due` | 0/1 — the OHLCV-only earnings proxy (§8.2.1) | **CORE** |
| `day_of_week` | 0–4 categorical | EXT |
| `sector` | categorical (~11 values) — CatBoost handles natively | EXT |

> **`day_of_week` and `sector` are the only categoricals**, and both are EXTENDED on
> purpose. Calendar features in particular are easy to overfit on 9 years of data — there
> are only ~470 Mondays in the sample.

#### Totals

| Family | CORE | EXTENDED |
|---|---|---|
| 1 · Trend | 8 | 17 |
| 2 · Relative & market | 6 | 8 |
| 3 · Momentum | 7 | 12 |
| 4 · Volatility | 7 | 11 |
| 5 · Volume | 6 | 10 |
| 6 · Structure | 4 | 11 |
| 7 · Bar shape & calendar | 3 | 8 |
| **Total** | **41** | **77** |

Dedup (§8.3) will cut some of EXTENDED automatically — RSI, Stochastic, CCI, Williams %R
and MFI are heavily mutually correlated, and the pipeline drops one of any pair above
|ρ| > 0.95. Expect the usable extended set to land nearer 60.

> #### If you only implement five features
>
> Asked to bet on which carry real signal at this horizon, in order:
>
> 1. **`rs_rank`** — relative strength vs the universe. In published equity work this is
>    repeatedly the strongest single daily-horizon feature, and it's the one thing your
>    original indicator list had no way to express.
> 2. **`adx_14`** — separates "trending" from "chop", which is the single most useful
>    regime split for a tree to make first.
> 3. **`ema_stack`** — cheap, robust summary of multi-timeframe trend agreement.
> 4. **`atr_pct_rank`** — volatility regime; gates sizing and tells the model when the
>    barriers are unusually wide.
> 5. **`spy_above_ema200`** — long and short setups behave completely differently either
>    side of this line.
>
> Notice that **only one of those five is on your original list** (the EMA one). That's
> the main substantive thing §8.2 adds: the original list is entirely *absolute and
> single-ticker*, with no notion of "compared to what?" or "under what conditions?".

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
> *to*. That's what the `tradeable=False` tag is for (§6): they flow through fetch and
> features exactly like any other ticker, and are filtered out only at the decision step
> (§13). Don't let "exclude ETFs" collapse into "don't download SPY."

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

## 9. The training table and label generation

### 9.0 What the table actually looks like

You asked what columns get added once OHLCV + indicators exist. Here is the whole schema.
One row per `(symbol, date)`, written to Parquet and exportable to CSV.

```
┌─ IDENTITY ────────────────────────────── never features ─┐
  symbol                  str        "NVDA"
  date                    date       2026-11-03

┌─ RAW ───────── kept for charting (§7) and labeling; NOT features ─┐
  open high low close volume

┌─ FEATURES (X) ──────── the allowlist from registry.feature_names() ─┐
  close_vs_ema21          float      -0.012
  adx_14                  float      31.4
  rsi_14                  float      58.2
  rs_rank                 float      0.94
  atr_14                  float      3.81      ← used by labeler, also a feature as atr_pct
  ... 41 CORE columns (§8.0), ~115 with EXTENDED

┌─ TARGETS (y) ───────────────── one set per direction, from §9.1 ─┐
  y_long                  int8       1 = TP hit before SL, 0 = not
  y_short                 int8
  y_long_t1               date       ⚠️ when the label RESOLVED — §10.3 needs this
  y_short_t1              date
  y_long_reason           str        tp | sl | timeout | ambiguous
  y_short_reason          str
  y_long_ret              float      realized return, NET of costs (§11)
  y_short_ret             float
  y_long_hold_days        int8
  y_short_hold_days       int8
  y_long_weight           float      label-uniqueness weight (§9.1)
  y_short_weight          float

┌─ FORWARD RETURNS ─────── diagnostics + optional alternative target ─┐
  fwd_ret_1d              float      close[t+1]/close[t] − 1
  fwd_ret_2d              float
  fwd_ret_5d              float
  fwd_ret_10d             float
  fwd_mfe_10d             float      max FAVOURABLE excursion ÷ ATR  ← see below
  fwd_mae_10d             float      max ADVERSE excursion ÷ ATR     ← see below

┌─ META ─────────────────────────────────── bookkeeping, never features ─┐
  meta_tradeable          bool       False for SPY and sector ETFs (§6)
  meta_gate_pass          bool       passed the §13 hard gates (§10.1 sample filter)
  meta_run_id             str        provenance (§5)
```

### Yes, add forward-looking columns — with two corrections

Your instinct is right: the table needs columns computed from the future, or there is
nothing to learn from. Two things to get right about *which* future, though.

> #### ⚠️ Correction 1: predict the **return**, never the **price**
>
> This is the single most common way a financial ML project fools itself, and it's worth
> being blunt about because the failure looks like spectacular success.
>
> If you add `next_day_close` as the target and regress on it, you will get **R² ≈ 0.999**.
> You will think you have built something extraordinary. You have built this:
>
> ```
> close[t+1] ≈ close[t]
> ```
>
> Tomorrow's price is almost entirely explained by today's price. The model learns the
> identity function, scores brilliantly on every regression metric, and contains **zero**
> tradeable information. Worse, it's hard to notice: the predictions look right, the
> scatter plot looks like a perfect diagonal, and the errors are tiny.
>
> So the target must always be a **change**: `close[t+1]/close[t] − 1`. Do that and R²
> drops to roughly **0.00–0.01**, which is disappointing and *true*. Next-day returns are
> ~95% noise. The honest number is the useful one.
>
> **Hard rule: no absolute price level is ever a target or a feature.** This is the same
> scale-free rule as §8.5, applied to the `y` side of the table.

> #### ⚠️ Correction 2: next-day return doesn't answer your question
>
> Suppose you predict `fwd_ret_10d = +4%` perfectly. Should you take the trade, and where
> does the stop go?
>
> You can't tell — because **the endpoint doesn't describe the path.** Consider two stocks
> that both finish +4% after 10 days:
>
> ```
> A:  +1% +1% +1% +1% ... steady            → your 1-ATR stop never triggers.  WIN
> B:  −9% −3% +2% ... recovers to +4%       → your stop triggered on day 1.    LOSS
> ```
>
> **Identical target value, opposite trade outcomes.** A forward-return model cannot
> distinguish them, which means it cannot tell you whether TP gets hit before SL — and
> "take profit before I cut loss" was the original question (§1).
>
> That's what the triple barrier in §9.1 is for: it simulates the *path* and reports which
> exit fired first. `y_long` already encodes "profit before loss" directly, so no
> post-processing is needed to turn a number into a decision.

### So why keep `fwd_*` columns at all?

Three genuinely useful jobs, none of them "be the primary target":

**1. Sanity check on whether any signal exists.** Correlate each feature against
`fwd_ret_5d` and `fwd_ret_10d`. If *nothing* in ~115 features correlates with *any*
forward horizon, stop and reconsider the features — don't go train a model on it. This is
a five-minute check that can save weeks, and it goes in `reports/feature_dist.html`.

**2. `fwd_mfe_10d` and `fwd_mae_10d` tell you if your TP/SL multiples are sane.** These
are the maximum favourable and adverse excursions over the horizon, in ATR units — i.e.
*how far did price actually travel in each direction before the window closed.* They make
the barrier choice empirical instead of guessed:

| Observation | What it means | Action |
|---|---|---|
| `fwd_mfe_10d` rarely reaches 2.0 | your TP is further than price usually travels | lower `k_tp`, or raise `H` |
| `fwd_mae_10d` usually exceeds 1.0 | your SL sits inside normal noise — you'll be stopped out constantly | raise `k_sl` |
| both frequently exceeded | barriers are too tight for daily bars | widen both; expect many `ambiguous` labels (§9.1) |

Feed these into `tdm labels sweep` (§9.1). They turn "TP 2.0 / SL 1.0" from a reasonable
guess into a measured choice.

**3. An alternative target you can actually test.** The design's claim that
classification-on-barriers beats regression-on-returns is an argument, not a proof. So
training supports both and you can compare on the same folds:

```bash
tdm train --target barrier          # default: y_long / y_short (§9.1)
tdm train --target fwd_ret_10d      # regression on net 10-day return
tdm train --target fwd_ret_sign     # classification on direction only
```

If forward-return regression wins on *expectancy after costs* (§10.4 — not on R², which
is the wrong metric), use it. The infrastructure cost is one flag.

> #### ⚠️ The leakage bug this table invites, and how the design prevents it
>
> Features and targets now live in **the same table**. That is convenient, and it is one
> line away from catastrophe:
>
> ```python
> X = df.drop(columns=["y_long"])        # ← LEAK. fwd_ret_10d is still in X.
> ```
>
> You just handed the model the answer. It will score ~99% accuracy, the metrics will look
> incredible, and the backtest will look like a money printer. There is no error message.
> This is §10.5's central bug class arriving through a one-word mistake.
>
> **Two mechanisms, both required:**
>
> 1. **Column naming is load-bearing.** Every non-feature column carries a mandatory
>    prefix: `y_` (targets), `fwd_` (forward-looking), `meta_` (bookkeeping). A bare name
>    is a feature. This makes a leak *visible* when reading code.
> 2. **Feature selection is an allowlist, never a denylist:**
>
>    ```python
>    X = df[registry.feature_names()]       # ✅ allowlist — explicit, from §8.0
>    X = df.drop(columns=TARGETS)           # ❌ denylist — silently leaks any new column
>    ```
>
>    With an allowlist, adding `fwd_ret_20d` tomorrow is inert: it isn't registered, so it
>    can't reach the model. With a denylist, the same addition silently poisons every
>    model you train afterwards.
>
> `tests/test_leakage.py` asserts that no column matching `^(y_|fwd_|meta_)` ever appears
> in `X`, and that `set(X.columns) == set(registry.feature_names())` exactly.

### Export

```bash
tdm dataset build --set core           # -> data/datasets/<run_id>.parquet
tdm dataset export --format csv        # -> data/exports/<run_id>.csv   (one flat table)
tdm dataset schema                     # print the column list with dtypes and tiers
```

The CSV export is the table above, flat, one row per `(symbol, date)` — readable in Excel,
and exactly what the model consumes.

---

### 9.1 Labeling: the triple barrier

The method is the **triple-barrier method** (López de Prado, *Advances in Financial
Machine Learning*), and it's what turns "should I trade?" into a supervised learning
problem that matches the question.

Rather than predicting a price (hard, and — per §9.0 — not what you need), simulate the
trade and ask **which exit fires first**. It's a race between three conditions — close to a `select` over
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

### 9.2 Configuration and the parameter sweep

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
                     #   median fwd_mfe / fwd_mae in ATR  (§9.0) — are the
                     #     barriers placed where price actually travels?
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

### 10.1 Model choice — XGBoost or Random Forest?

You asked about XGBoost vs Random Forest, then suggested CatBoost. **CatBoost is the
right call**, and for a better reason than "it's usually good" — see below. Random Forest
stays as a **required control**, not an alternative.

But the more useful answer is the caveat first.

> #### ⚠️ Model choice is not the bottleneck here
>
> Swapping Random Forest for XGBoost will move your hit rate by **one or two points**.
> For comparison:
>
> | Issue | Impact on results |
> |---|---|
> | One undetected leak (§10.5) | the entire result — a worthless model that looks excellent |
> | Getting the cost model wrong (§11) | ~8 points of break-even hit rate |
> | Features containing no real signal | the entire result |
> | Universe too small to measure anything (§16 #1) | you can't tell if it worked |
> | **CatBoost vs RF vs XGBoost** | **1–2 points** |
>
> Spend your effort in that order. Model choice is the last ~5% of this problem, and it's
> the part most people spend 80% of their time on.

### The ladder, in order

| # | Model | Role |
|---|---|---|
| 1 | `DecisionTreeClassifier(max_depth=3..4)` | **Readable baseline.** Print the whole rule set (`reports/tree.txt`) and read it. It tells you whether the model found something sensible (`ADX>25 and EMA stack aligned and RSI<70 → long`) or something absurd. **Keep this permanently**, not just as a first step — it's your sanity check on everything above it. |
| 2 | `RandomForestClassifier` | **The control.** Few knobs, hard to misuse, robust to noise. Its job is diagnostic: **if RF beats your boosted model out-of-sample, the boosted model is overfit.** Always report it. |
| 3 | **`CatBoostClassifier`** | **The primary model.** Reasons below. |
| 4 | `HistGradientBoostingClassifier` | Second opinion, zero extra dependency. Worth running once to confirm CatBoost isn't doing something odd. |
| 5 | `xgboost` | Refinement, for monotonic constraints. |

### Why CatBoost

You suggested it, and it holds up on merits — **this problem is close to CatBoost's best
use case**, which is not something I'd say about most datasets.

- ✅ **Ordered boosting** (`boosting_type="Ordered"`) is the real reason. Standard
  gradient boosting computes each tree's gradients using a model fitted on the *same*
  rows, which biases the estimates — CatBoost's authors call this *prediction shift*, and
  it's a subtle form of overfitting that gets worse as the dataset gets smaller and
  noisier. Ordered boosting estimates gradients from models trained on data *excluding*
  the row being evaluated. **On ~11,000 effective samples of very low signal-to-noise
  data, that is exactly the right correction.** It's the one genuinely differentiating
  feature for us.
- ✅ **Oblivious (symmetric) trees** by default — every node at a given depth splits on
  the same feature. That's a strong structural regularizer, which is what you want when
  the noise dominates. It also makes inference very fast, irrelevant here but harmless.
- ✅ **Strong defaults and a built-in overfitting detector** (`od_type="Iter"`). This
  matters more than it sounds: per the box above, *every hyperparameter you tune is
  another chance to overfit the validation folds*, so a library that needs less tuning is
  a genuine safety feature, not just a convenience.
- ✅ **Native NaN handling.** Necessary, not optional — every indicator has a warm-up
  period, so a 200-day feature is NaN for the first 200 bars of *every* ticker.
- ✅ Native categorical support for `sector`.

**What CatBoost is famous for that does *not* help us:** its headline feature is
target-statistic encoding of high-cardinality categoricals. Our features are almost
entirely numeric — `sector` is the only categorical, and it has ~11 values. So ignore
that part of the marketing; the reasons to use it here are ordered boosting and the
regularization defaults.

**The cost:** CatBoost has no `cp315` wheel, which is why the project is on Python 3.14
(§3). Nothing else in the design cared, so this was cheap.

> #### Suggested starting configuration
>
> ```python
> CatBoostClassifier(
>     loss_function="Logloss",
>     boosting_type="Ordered",   # ← the reason we're here. Don't switch to "Plain".
>     depth=4,                   # SHALLOW. not 8, not 10.
>     learning_rate=0.03,        # slow
>     iterations=3000,           # high, but...
>     od_type="Iter",            # ...stopped by the overfitting detector
>     od_wait=100,
>     l2_leaf_reg=10.0,          # strong
>     rsm=0.7,                   # feature subsampling, decorrelates trees
>     random_seed=42,
>     auto_class_weights="Balanced",
>     verbose=False,
> )
> ```
>
> ⚠️ **`boosting_type="Ordered"` is slower** — noticeably so on larger datasets. At our
> size (~11k effective samples) that's fine and worth it. If training becomes painful,
> reduce `iterations` or `depth` before you reach for `"Plain"`, because `"Plain"` throws
> away the main reason for choosing this library.

### Why Random Forest is not the primary choice (but must be reported)

RF reduces **variance** by averaging deep trees. Boosting reduces **bias** by sequentially
fitting what the previous trees got wrong, which makes it more statistically efficient at
extracting a weak signal — and a weak signal is precisely what we have.

The counter-argument is real though: **boosting overfits noisy data more readily, and
financial data is the textbook case.** Ordered boosting mitigates it but does not remove
it. That's exactly why RF is a required control rather than an afterthought. The
comparison is informative in both directions:

| Outcome | Reading |
|---|---|
| CatBoost > RF on the holdout | boosting is extracting real signal. Good. |
| **RF ≥ CatBoost on the holdout** | **CatBoost is fitting noise.** Raise `l2_leaf_reg`, drop `depth` to 3, or ship RF. |
| Both ≈ the depth-4 tree | there are only a few real effects here. Ship the tree — it's readable and you'll trust it. |
| All ≈ base rate | the features don't contain signal. No model fixes that. |

That last row is a real possible outcome, and finding it out in week 6 rather than month 6
is worth the 20 minutes it costs to run all three.

> #### Regularization matters more than which library you pick
>
> On ~11,000 effective samples (below) with very low signal-to-noise, the settings matter
> more than which library you pick. Shallow, slow, strongly regularized — concrete values
> for CatBoost are given below.
>
> The instinct to set `depth=8` and `iterations=5000` is how you get a beautiful training
> curve and a worthless model. **Every hyperparameter you tune is another chance to
> overfit the validation folds**, so keep the search small and coarse — a handful of
> values for `depth` and `l2_leaf_reg`, and let the overfitting detector handle the rest.

### What `xgboost` adds, and when to reach for it

One thing worth the dependency, at the refinement stage: **monotonic constraints.** You
can declare that a feature's effect must be non-decreasing — e.g. "all else equal, higher
ADX must not *decrease* `P(trend continues)`". This encodes domain knowledge directly into
the hypothesis space, which **reduces overfitting by making implausible shapes
unlearnable.** On a small, noisy dataset that's a genuine edge. CatBoost supports it too
(`monotone_constraints`), so try it there first before adding XGBoost.

### What not to use

| Model | Why not |
|---|---|
| **LightGBM** | Installs cleanly, but it's a fourth boosting library with no advantage over CatBoost here. Leaf-wise growth also overfits small data more readily unless carefully capped. |
| **Neural networks / LSTMs** | See the sample-size box below. On tabular data at this scale, boosted trees beat deep nets consistently in published benchmarks. The previous project carried a `torch` dependency; drop it. |
| **Stacking / ensembling the above** | Tempting and almost always a way to overfit the validation folds on a dataset this size. Revisit only if a single model shows a solid holdout edge first. |

> #### Sample size: the consequence of choosing 9 years
>
> This governs how complex a model you can justify, so it's worth the arithmetic.
>
> | | |
> |---|---|
> | 9 years × ~252 sessions | **~2,268 bars per ticker** |
> | ÷ horizon 10 (labels overlap — §9) | **~227 *effective independent* samples per ticker** |
> | **× 50 tickers (chosen)** | **~11,350 effective samples** |
>
> The row count will *look* like ~113,000. The information content is ~11,350.
> **That smaller number is what should govern model complexity** — it's why §10.1 insists
> on `depth=4` and strong `l2_leaf_reg` rather than the defaults.
>
> **The key relationship: `effective samples ≈ tickers × years × 25`.** Years and tickers
> are interchangeable for statistical power. At 50 × 9 you have enough to justify a
> regularized boosted model and to get a holdout verdict that means something.
>
> What 9 years puts in the window:
>
> | Period | What it contributes |
> |---|---|
> | Feb 2018 | a sharp volatility spike |
> | late 2018 | a ~20% correction |
> | 2019 | steady bull |
> | **Feb–Mar 2020** | **the COVID crash — a ~34% drop in five weeks** |
> | 2020–21 | recovery and melt-up |
> | 2022 | a ~25% bear market |
> | 2023–26 | bull, then the current regime |
>
> Two bear markets plus two volatility shocks — about as much regime diversity as any
> 9-year window could offer. You'd argued COVID wasn't essential because technical setups
> recur; at 9 years you get it anyway, which is a free hedge against that assumption
> being wrong.
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
> One caveat on the 50: they must be **spread across sectors**. Fifty names that all move
> together deliver far less than fifty tickers' worth of independent information, so the
> effective count above is an upper bound that assumes reasonable diversification.

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

**On sample size and model capacity.** Effective sample size is ~11,350 (above). On tabular data at that scale, gradient-boosted trees consistently beat deep nets
in published benchmarks. The previous project carried a `torch` dependency; drop it. If you
later want sequence models, that's an argument for *more tickers first*, not a deeper
architecture.

**One global model across all tickers**, not one per ticker (per-ticker gets ~3,800 rows
and overfits). **Deliberately exclude ticker identity as a feature** — it invites the model
to memorize "NVDA went up 2015–2025", which is survivorship-flavored and won't repeat.
Sector as a categorical is fine — CatBoost handles it natively.

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

> **Why 12 months.** With 9 years of history (§10.1) a 12-month holdout is ~11% of the
> data — a comfortable fraction, leaving 8 years for walk-forward CV.
>
> **The holdout governs how much your final verdict is worth.** 12 months is ~25 effective
> samples per ticker, so at 50 tickers it carries ~1,250 — comfortably past the ~100-trade
> minimum from §10.4 even after the §13 threshold filters most bars out. The go/no-go will
> mean something, which was not true at 9 tickers (~225).

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
`vectorbt` is deliberately unused — we'd write this anyway, because §9's labeler
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
- **Expect wide confidence intervals, and check them.** 50 tickers × 9 years gives
  ~11,350 effective independent samples (§10.1). Distinguishing a 40% hit rate from a 45% one needs on the order of a
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
    if not universe.is_tradeable(symbol):
        return SKIP("data-only instrument (ETF/benchmark)")   # §6, §8.2
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

**Output:** a `rich` table on stdout, `reports/decisions_YYYY-MM-DD.{md,csv}`, and one
appended line in `decisions.jsonl` — the log that §13's self-scoring reads back. Keep that
file forever; it is the only truly out-of-sample record you will ever have.

### Self-scoring — build this early, not last

```bash
tdm score-decisions --since 2026-01-01
```

Walks every past row in `decision`, resolves it against bars that have since arrived,
Reads the appended decision log, resolves each past decision against bars that have since
arrived, and reports realized hit rate / expectancy / drawdown **of the decisions the
system actually issued**.

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
| **0 · Scaffold** | venv 3.14, `CLAUDE.md`, Makefile, test runner | `make venv && make test` green; the venv guard fires outside the venv |
| **1 · Fetch** | `yf.download` wrapper, `universe validate`, OHLCV sanity checks, Parquet snapshot | 9 years × your tickers lands clean. ⚠️ **Smoke-test yfinance against pandas 3.1 here** (§3) — it is the one unverified dependency |
| **2 · Indicators** | registry + Tier-1 + your requested list, Parquet output, CSV export | `test_indicators.py` green; `test_invariance.py` green (§5) |
| **3 · Chart** | Plotly multi-panel HTML per ticker (§7) | **you open `NVDA.html` and the indicators look right.** This is the gate on phases 1–2 — do not proceed past a chart that looks wrong |
| **4 · Structure features** | ZigZag + Dow/Elliott with `confirmed_at` | `test_leakage.py` pivot tests green, **and the pivots look right on the chart** — hindsight pivots are obvious visually and nearly invisible in a test |
| **5 · Labels** | triple barrier, `t1`, weights, `labels sweep` | base rate / timeout fraction / ambiguity rate sane; **TP/SL brackets look sane on the chart** (`--with-labels`) |
| **6 · Baseline model** | depth-4 tree + purged walk-forward CV | **the printed rule set is readable and not absurd** |
| **7 · Model comparison** | CatBoost + RandomForest control + calibration + permutation importance + threshold tuning | all three compared on the same folds (§10.1); you know which row of that table you are in |
| **8 · Backtest** | engine, costs, sensitivity grid, shuffled control | survives 2× slippage; beats the shuffled control |
| **9 · Decide** | `tdm decide`, gates, reports | one command, end to end |
| **10 · Self-scoring** | `tdm score-decisions` | reproduces phase-8 numbers on the overlapping window |
| **11 · Holdout** | one single evaluation on the untouched 12 months | **go / no-go** |

**Phase 3 is deliberately early.** Charting before labeling or training means you catch
feature bugs while they are still cheap — an off-by-one indicator or a lookahead pivot is
obvious on a chart and nearly undetectable in a metric. A model trained on quietly broken
features still produces a plausible-looking number, which is the worst outcome available.

Phases 3 and 10 — the chart and the self-scoring — are what make this different from
the previous attempt. **If you only reach
phase 3**, you still have a reproducible feature pipeline and readable charts of every
ticker — already a useful analysis tool on its own, independent of any model.

---

## 15. Decisions resolved on 2026-11-03

| # | Question | Resolution |
|---|---|---|
| 1 | Asset class | **US stocks only, no ETFs** as trading candidates. ETFs still fetched as benchmarks, tagged `tradeable=False` (§6) |
| 1b | Universe source | **User-supplied ticker list** in `config/universe.txt`; benchmarks appended automatically — §6 |
| 2 | History depth | **9 years** (~2,268 bars/ticker). Covers Feb 2018, the 2020 COVID crash and the 2022 bear market — §10.1 |
| 3 | Label profile | **TP 2.0 / SL 1.0 ATR.** Confirmed. `labels sweep` will still test wider profiles, because §11's cost arithmetic favours them |
| 4 | Horizon | **10 trading days.** Explained in §9 — it's the timeout barrier |
| 5 | Can you short? | **Yes**, and via CFDs it's mechanically symmetric with going long (§11.1) |
| 6 | Execution venue | **IQ Option stock CFDs, 10× leverage.** Drove the §11.1 rewrite: Reg SHO and share-borrow modeling deleted; financing and margin added |
| 7 | Earnings data | **Not required.** OHLCV-only, with a derived quarterly-gap proxy (§8.2.1). Real calendar stays behind a config flag |
| 8 | Weekly features | **Dropped.** Removes a leakage class (§8.2) |
| 9 | Storage | **No database.** Re-download each run; snapshot only the feature matrix for reproducibility — §5 |
| 10 | Visualization | **Plotly HTML chart per ticker, generated before training** — §7, build phase 3 |
| 11 | Model | **CatBoost primary** (ordered boosting suits small noisy data), **RandomForest as a required control**, depth-4 tree as the readable baseline — §10.1 |
| 12 | Python | **3.14**, not 3.15 — CatBoost has no cp315 wheel — §3 |
| 13 | Universe size | **50 tickers**, user-supplied — ~11,350 effective samples, ~1,250 in the holdout — §10.1 |

---

## 16. Still open

1. ~~**How many tickers?**~~ **Resolved: 50.** That puts you in good shape —
   ~11,350 effective samples for training and ~1,250 in the 12-month holdout, which is
   comfortably past the ~100-trade minimum from §10.4, so the final go/no-go verdict will
   actually mean something. Two asks on *which* 50:

   - **Spread them across sectors.** 50 correlated mega-cap tech names deliver far less
     than 50 tickers' worth of independent information, because they mostly move together.
     Aim for coverage: tech, financials, healthcare, energy, industrials, consumer,
     utilities, materials.
   - **Include some losers.** Deliberately add names that did badly over the window
     (INTC is a good start). This is the only practical lever against the survivorship
     bias in §10.5 #7 — a list of today's winners was selected *because* they went up, and
     a model trained only on those learns that buying always works.

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
