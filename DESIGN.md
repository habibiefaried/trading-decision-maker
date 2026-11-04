# Trading Decision Maker — Design

**Status:** design only, no code yet.
**Audience:** software engineers. No finance background assumed — §2 defines every domain
term used in this document.

---

## 1. What the system does

A batch pipeline you run once a day, after the US market closes. It reads 9 years of daily
price history for a list of stocks you choose and tells you, for each one: buy, sell-short,
or do nothing — and if you trade, the exact prices at which to exit.

```
$ make daily

NVDA   LONG   conf 0.71
  entry  ~$183.40    (indicative; actual fill is tomorrow's open)
  TP      $196.10     +$200 if hit
  SL      $177.05     −$100 if hit
  size     26 units  = $4,768 notional, $477 margin
  hold    ≤ 10 trading days, then exit at market

AMD    HOLD   conf 0.48   — no edge: ADX 14 (choppy), gap-risk window

account: 3 of 10 slots used · $1,431 of $5,000 margin · $300 currently at risk
```

Read the NVDA line as: *bet NVDA goes up. Take the profit at $196.10, or cut the loss at
$177.05, whichever comes first. Give up after 10 days either way.*

### Settings

| Setting | Value |
|---|---|
| Market | US stocks — liquid, well-known names (MSFT, NVDA, …). No ETFs |
| Universe | ~15 tickers you supply (any size you like — §5) |
| Data source | `yfinance`, daily bars, 9 years |
| Execution venue | IQ Option stock CFDs, 10× leverage |
| Style | Swing trading — hold across days |
| Take-profit | 2.0 × ATR from entry |
| Stop-loss | 1.0 × ATR from entry |
| Max holding period | 10 trading days |
| Entry | next day's open (two alternatives in §8.2) |
| Directions | Long and short |
| Account | $10,000, risk 1% per trade, max 10 concurrent positions |
| Primary model | CatBoost (gradient-boosted trees) |
| Python | 3.14 |

### The pipeline

| Stage | What it is, in engineering terms | Cost |
|---|---|---|
| **1. Fetch** | `yf.download()` 9 years of daily OHLCV | ~20–60 s |
| **2. Features** | `map` ~100 pure functions over each series → one wide numeric matrix | seconds |
| **3. Chart** | Render price + every indicator to interactive HTML so you can read it | seconds |
| **4. Dataset** | Add target columns by simulating each trade | seconds |
| **5. Train** | Fit a classifier with time-aware cross-validation | minutes |
| **6. Decide** | Run inference on today's row, write the output table | milliseconds |

**There is no database.** The pipeline is a pure function from `(ticker list, date range)`
to `(charts, model, decisions)`. Every stage re-derives its input. §5 explains why that
works.

### Three things that make this harder than ordinary ETL

1. **The correctness bug you cannot see.** The system predicts the future from the past.
   If any code path lets future data reach the model, everything still runs, every test
   passes, the metrics look *excellent*, and the system is worthless. There is no stack
   trace. §9.4 is entirely about preventing this, and it's the most important part of this
   document.
2. **Two implementations of the same logic must agree.** Training targets come from a
   simulator; the backtest replays it; production runs a third path. If they drift, you
   validated against a world you don't deploy into — and it surfaces as unexplained live
   underperformance, which is nearly impossible to debug after the fact.
3. **The signal is genuinely weak.** Daily price data is mostly noise. A good result here
   looks mediocre by normal ML standards (§12), which makes it very easy to mistake a bug
   for a breakthrough.

---

## 2. Glossary

### The data

| Term | Meaning |
|---|---|
| **Ticker** / **symbol** | String ID for a stock: `"NVDA"`. The primary key of the system. |
| **Bar** | One row of data for one ticker for one day. |
| **OHLCV** | The 5 numbers in a bar: **O**pen (first trade price of the day), **H**igh, **L**ow, **C**lose (last trade price), **V**olume (shares traded). That's the entire raw input. |
| **Session** | One trading day. The US market is open weekdays minus ~10 holidays, so "yesterday" and "the previous session" differ — you need a calendar library to know which. |
| **Index** | A calculated number tracking a group of stocks. `^GSPC` is the S&P 500 index — a one-number proxy for "the US market". `^VIX` tracks expected volatility. Indices aren't traded; we read them only as reference data (§6.3). |
| **ADV** | Average Daily Volume. Low ADV means you can't buy or sell without moving the price yourself. |

### The actions

| Term | Meaning |
|---|---|
| **Long** | Buy now, sell higher later. You profit if the price goes **up**. |
| **Short** | Sell now, buy back lower later. You profit if the price goes **down**. |
| **Flat** / **HOLD** | No position. A valid and frequently correct output. |
| **CFD** | *Contract for difference* — a derivative that pays you the price change without you owning the shares. This is what IQ Option sells. It makes shorting mechanically identical to going long, and adds a daily financing charge (§10.1). |
| **Leverage** | Trading a position larger than your cash by borrowing the rest. At 10×, $500 of your money controls a $5,000 position. |
| **Margin** | The portion of your own money a leveraged position ties up. |
| **TP** (take-profit) | A standing order with the broker: close automatically at this price, in **profit**. |
| **SL** (stop-loss) | The same, at a **loss**, to stop it getting worse. |
| **Position sizing** | How many units to buy. A separate decision from direction, and more important (§10.2). |

### Measuring things

| Term | Meaning |
|---|---|
| **Indicator** | A pure function over a window of past bars returning one number per bar: `f: List[Bar] -> float`. That's all any of them are, including the ones with mystical names. |
| **SMA / EMA** | Simple / Exponential Moving Average over N days. A low-pass filter on a noisy signal; `EMA` weights recent days more. |
| **ATR** (Average True Range) | **The most important one.** Roughly the average high-to-low distance over the last 14 days — a volatility measure, in dollars. A $500 stock might have ATR $12; a $20 stock, $0.40.<br><br>We use ATR as a **unit of distance**, like using standard deviations instead of raw values. "Put the stop 1 ATR away" means "one typical day's movement away", which is automatically the right size for both of those stocks. This is how one rule works across every ticker in the universe. |
| **Oscillator** | An indicator bounded to a fixed range (usually 0–100) measuring how overextended a recent move is. RSI, Stochastic, MFI, CCI are variations. Convenient because they're already normalized. |
| **Trend indicator** | Measures direction and/or strength of a sustained move. EMA slope, ADX, Aroon, Supertrend. |
| **Volume indicator** | Combines price with volume, on the theory that a move backed by heavy trading is more meaningful. OBV, MFI, VWAP. |

### Evaluating results

| Term | Meaning |
|---|---|
| **Return** | Percentage change: `(exit − entry) / entry`. |
| **R:R** | Ratio of TP distance to SL distance. TP 2 ATR / SL 1 ATR = 2.0 — you win twice what you lose, per event. |
| **Hit rate** | Fraction of trades that reach TP instead of SL. **Analogous to precision, not accuracy.** |
| **Expectancy** | `p_win × avg_win − p_loss × avg_loss`. Expected value per trade. **This is the real objective function.** At R:R 2.0 you break even at a 33% hit rate, so "67% of predictions wrong" is still profitable — which is why accuracy is a near-useless metric here (§9.3). |
| **Base rate** | The unconditional fraction of positive labels. Your baseline to beat. |
| **Drawdown** | Peak-to-trough decline in account value. Max drawdown is the worst stretch you'd have had to sit through. |
| **Backtest** | A replay harness: run the strategy over history and see what would have happened. Trivially easy to get wrong in ways that flatter you (§10). |
| **Slippage** | The gap between the price you expected and the price you got. |

### The hazards

| Term | Meaning |
|---|---|
| **Lookahead** / **leakage** | A bug where future information reaches a calculation that should only see the past. **The central correctness risk in this project.** Closest analogy: a test that passes because it reads the expected value out of the fixture it's asserting against — green, and meaningless. |
| **Survivorship bias** | Selection bias in the ticker list. Pick today's famous companies and you picked them *because* they went up; your backtest then "discovers" that buying stocks works. |
| **Overfitting** | Standard ML meaning. Acute here because the signal-to-noise ratio is terrible and the effective sample size is far smaller than the row count (§9.1). |
| **Corporate action** | An event that changes share structure: a **split** (1 share becomes 10, each worth 1/10th) or a **dividend** (cash payout). Both rewrite historical price series — §5. |
| **Earnings** | A quarterly scheduled report. The price frequently jumps 5–15% overnight, which can blow straight through a stop-loss. The most dangerous scheduled event for an open position. |

---

## 3. Environment

### Python 3.14, in a venv

Everything runs inside `./venv`. Enforced three ways, because bare `python3` on this
machine is the wrong interpreter:

1. **`.python-version`** → `3.14`
2. **`CLAUDE.md`** → instructions for AI agents working in the repo
3. **A runtime guard** in `src/tdm/__init__.py`, which is the one that actually works:

```python
import sys
if sys.prefix == sys.base_prefix:
    raise RuntimeError("not running inside ./venv — use ./venv/bin/python or `make`")
```

Setup:

```bash
brew install python@3.14
/opt/homebrew/bin/python3.14 -m venv venv
./venv/bin/python -m pip install -U pip
./venv/bin/pip install -r requirements.txt
```

> **For AI agents and shell scripts:** don't use `source venv/bin/activate` in a tool
> call. Shell state doesn't persist between calls, so activation is gone by the next
> command and you silently run on system Python. Always use the explicit path.

A `Makefile` wraps every command so the venv path is never typed by hand:

```make
PY := ./venv/bin/python

venv:       # create venv + install
fetch:      # download OHLCV
features:   # compute indicators
charts:     # render HTML charts
dataset:    # features + target columns
train:      # fit models
backtest:   # replay with costs
decide:     # today's calls
daily:      # fetch -> features -> dataset -> decide
test lint
```

### Dependencies

All verified to have `cp314` wheels or be pure-Python:

| Package | Version | Role |
|---|---|---|
| numpy | 2.5.4 | arrays |
| pandas | 3.1.0 | dataframes — **major version, breaking, see below** |
| scikit-learn | 1.9.1 | RandomForest, cross-validation, calibration |
| catboost | 1.2.10 | primary model (§9.1) |
| TA-Lib (`ta-lib`) | 0.8.3 | indicator engine. Wheels are self-contained — no `brew install ta-lib` |
| plotly | 7.1.0 | interactive HTML charts (§7) |
| pyarrow | 26.0.1 | Parquet |
| yfinance | 1.7.0 | data source |
| exchange-calendars | 4.13.2 | market session calendar |
| typer / rich / pydantic-settings / pytest | — | CLI, output, config, tests |

**Do not add these** — they will not install, or will cause problems:

| Package | Reason |
|---|---|
| `pandas-ta` | pins `numba==0.61.2`, which has no `cp314` wheel. Use TA-Lib. |
| `vectorbt` | installs, but don't — §10 explains why the backtest must share one implementation of the exit logic with the labeler. |
| `torch` | the dataset is far too small for deep learning (§9.1). |

### pandas 3.x gotchas

pandas 3 is a breaking release. The changes that affect this project:

| Change | Impact |
|---|---|
| **Copy-on-Write is mandatory** | Chained assignment `df['x'][0] = v` silently does nothing or raises. Use `df.loc[...]`, or build columns functionally. |
| **Default datetime resolution is microsecond**, not nanosecond | Integer arithmetic on timestamps is now off by 1000×. |
| **Offset aliases `M`/`Q`/`Y` removed** | Use `ME`/`QE`/`YE`. |
| **`inplace=True` returns `self`** | Previously `None`. |
| **String columns infer to `str` dtype** | Not `object`. |

> ⚠️ **Verify this first.** `yfinance` declares `pandas>=1.3.0` with no upper bound, so it
> *installs* next to pandas 3.1 — which is not the same as working there. Smoke-test the
> fetch path before building anything else (§13, phase 1). If it breaks, pin
> `pandas>=2.3,<3`.

---

## 4. Repository layout

> **`tdm` = **t**rading **d**ecision **m**aker.** It's the Python package name
> (`src/tdm/`) and the CLI command name. A `console_scripts` entry point in
> `pyproject.toml` maps it to `tdm.cli:app`, so after `make venv` the binary exists at
> `./venv/bin/tdm`.
>
> Throughout this document `tdm fetch` is shorthand for either of these, which are the
> same thing:
>
> ```bash
> ./venv/bin/tdm fetch                      # via the console script
> ./venv/bin/python -m tdm.cli fetch         # via the module, identical
> ```
>
> Nothing is installed globally — `tdm` only exists inside `./venv` (§3).

```
trading-decision-maker/
├── .python-version              # 3.14
├── CLAUDE.md                    # rules for AI agents working here
├── DESIGN.md                    # this file — the "why"
├── README.md                    # the "how to run" only
├── Makefile
├── pyproject.toml               # packaging + the `tdm` console script
├── requirements.txt
│
├── config/
│   ├── universe.txt             # your tickers, one per line (MSFT, NVDA, …)
│   ├── config.toml              # horizons, ATR multiples, costs, account (§14)
│   └── feature_sets/
│       ├── minimal.toml         # 18 features — the default at ~15 tickers (§6.3)
│       ├── core.toml            # 40 features
│       └── extended.toml        # ~117 features
│
├── src/tdm/
│   ├── __init__.py              # venv guard
│   ├── cli.py                   # single entrypoint, subcommands
│   ├── config.py                # settings + config hashing
│   │
│   ├── ingest/                  # ── the only module that touches the network
│   │   ├── fetch.py             #    yf.download wrapper + retries (§5)
│   │   ├── validate.py          #    universe validation, OHLCV sanity checks
│   │   └── sessions.py          #    market-calendar wrapper
│   │
│   ├── features/                # ── pure functions, no I/O
│   │   ├── registry.py          #    name -> fn + warmup + tier (§6.1)
│   │   ├── trend.py             #    EMA stack, ADX, Aroon, Supertrend, Ichimoku
│   │   ├── momentum.py          #    RSI, MACD, Stochastic, ROC, CCI
│   │   ├── volatility.py        #    ATR, Bollinger, Keltner, Donchian
│   │   ├── volume.py            #    OBV, MFI, VWAP variants
│   │   ├── structure.py         #    ZigZag pivots -> Dow/Elliott features (§6.4)
│   │   ├── market.py            #    relative strength vs ^GSPC, market regime
│   │   └── pipeline.py          #    orchestrate, normalize, write Parquet
│   │
│   ├── dataset/
│   │   ├── barriers.py          #    the trade simulator that produces y (§8.2)
│   │   └── build.py             #    join features + targets into one table (§8.1)
│   │
│   ├── model/
│   │   ├── cv.py                #    PurgedWalkForwardSplit (§9.2)
│   │   ├── train.py
│   │   ├── explain.py           #    rule export, permutation importance
│   │   └── registry.py          #    artifacts + frozen thresholds
│   │
│   ├── report/                  # ── §7 — visual verification
│   │   ├── charts.py            #    Plotly multi-panel price + indicators
│   │   ├── overlays.py          #    label brackets, decision markers, pivots
│   │   └── diagnostics.py       #    correlation heatmap, feature distributions
│   │
│   ├── backtest/engine.py       #    §10
│   ├── provenance.py            #    append to runs.jsonl
│   └── decide.py                #    §11 — production inference
│
├── tests/
│   ├── test_indicators.py       # vs TA-Lib / hand-computed fixtures
│   ├── test_leakage.py          # ← the most important tests here (§9.4)
│   ├── test_invariance.py       # inject a split, assert past features unchanged (§5)
│   └── test_barriers.py
│
├── data/                        # GITIGNORED
│   ├── features/<run_id>.parquet
│   ├── datasets/<run_id>.parquet
│   └── exports/*.csv
├── models/<run_id>/             # GITIGNORED
├── reports/                     # GITIGNORED — charts/, equity.html, decisions
├── runs.jsonl                   # GITIGNORED — provenance log
└── decisions.jsonl              # GITIGNORED — never delete (§11)
```

### Layering rule

```
cli  ->  decide / train / backtest / report  ->  dataset  ->  features  ->  ingest
```

Dependencies point one way only. Two consequences worth enforcing in review:

- **`features/` and `dataset/` do no I/O and import nothing from `ingest/`.** They're pure
  functions over DataFrames. That's what makes the leakage tests in §9.4 possible — you
  can call any feature on a synthetic 50-row frame.
- **`report/` reads the feature matrix; it never computes an indicator.** §7 explains why.

---

## 5. Fetching data

### No database

Each run downloads fresh. `yfinance` is fast and reliable enough that re-downloading is
cheaper than the bookkeeping needed to avoid it — and it means no schema, no migrations,
no cache invalidation, no staleness logic.

```python
df = yf.download(
    tickers,                 # all 50 in one call
    start=start, end=end,
    interval="1d",
    auto_adjust=True,        # ← see below
    group_by="ticker",
    multi_level_index=False, # flatten at the boundary
    threads=True,
    progress=False,
)
```

### Why `auto_adjust=True` is safe

When a company does a 10-for-1 **split**, every share becomes 10 shares worth 1/10th each.
Nothing of value changed, but the raw price series now has a 90% cliff in it. So providers
offer *adjusted* prices, dividing all pre-split history by 10 to make the series
continuous.

The catch is that an adjusted price is **a function of every corporate action since**, not
a fact about the past. When a new split happens, every historical adjusted price changes.
If you *stored* adjusted prices you'd have a cache-coherency problem: rows written before
the split and after it would be on different scales, in the same table, with no error
raised.

Since nothing is stored, that can't happen. Every run gets one internally consistent
series.

### Why re-downloading doesn't change your features

The remaining worry: if a stock splits between two runs, the second run's prices are all
rescaled. Doesn't that change the training data?

**No.** A split rescales every pre-split price by the same constant `k`, and every feature
(§6.2) is a *ratio*, so `k` cancels:

| Feature | Under a rescale by `k` | Result |
|---|---|---|
| `close/EMA200 − 1` | `(k·C)/(k·EMA200) − 1` | **unchanged** |
| `ATR/close` | both scale by `k` | **unchanged** |
| `%B` = `(C−lo)/(hi−lo)` | all three scale by `k` | **unchanged** |
| `MACD/ATR` | both scale by `k` | **unchanged** |
| RSI, Stochastic, ADX | built from price *ratios* | **unchanged** |
| `volume/median_volume` | both scale by `1/k` | **unchanged** |

The scale-free rule in §6.2 exists so a decision tree's thresholds transfer across
tickers. It also makes the feature matrix immune to re-downloading. **Scale-free and
corporate-action-invariant are the same property.**

`tests/test_invariance.py` enforces this: inject a synthetic 10:1 split into a series,
recompute all features, assert every value before that date is bit-identical. Any feature
that fails is not scale-free, which means it was also broken for §6.2's reason. One
assertion, two bug classes.

### Snapshots, for comparing models

What *does* differ between runs is the tail — new bars arrive daily. Harmless for daily
use, corrosive for one activity:

```
Monday:  train model A, holdout expectancy 0.31
Friday:  train model B, holdout expectancy 0.28
         -> is B worse, or did the dataset change underneath you?
```

Comparing models is the central activity of this project, so:

> `tdm dataset build` writes `data/datasets/<run_id>.parquet` where
> `run_id = hash(tickers, date range, feature config, code version)`, and appends a line
> to `runs.jsonl`. Training reads a snapshot by default; `--refresh` forces a new fetch.

A file and a hash. No schema, no invalidation — per the invariance result above, a snapshot
is correct forever.

### The universe is your input

`config/universe.txt` is a plain list of tickers, one per line. Any length — the pipeline
doesn't care, but the *statistics* do, and the table below is the honest version.

```
NVDA
AMD
JPM
XOM
UNH
...
```

**Reference series are fetched separately and never traded.** The relative-strength and
regime features (§6.3) need a market series to compare against, so `fetch` also downloads
`^GSPC` (the S&P 500 **index** — not an ETF) and `^VIX`. These are not part of your
universe: they produce no rows in the dataset, get no decisions, and appear nowhere in the
output. They exist purely as inputs to a handful of feature columns.

`universe.txt` is therefore exactly and only the list of things you want to trade.

### How universe size affects what you can conclude

Effective sample count is `tickers × 227` (§9.1 explains why it isn't the row count). That
number governs how many features you can use and how much the final verdict is worth:

| Tickers | Effective samples | Holdout | Feature set | What you get |
|---|---|---|---|---|
| 1–3 | 230–680 | ~50 | — | ⚠️ **Charts only.** Too few samples to fit anything that generalizes. The pipeline runs; the model output is noise. |
| 5–10 | 1,100–2,300 | 125–250 | `minimal` (18) | A depth-3 tree. Treat any result as a hypothesis, not a finding. |
| **15** | **~3,400** | **~375** | **`minimal` (18)** | **Workable.** Enough to train and to see whether there's signal. Holdout gives a *direction*, not a verdict. |
| 30 | ~6,800 | ~750 | `core` (40) | Holdout starts to mean something. |
| 50+ | ~11,400+ | ~1,250+ | `core`, then `extended` | A holdout verdict you can act on. |

> **At 15 tickers, two things follow and both are in the design:**
>
> 1. **Use the `minimal` feature set, not `core`.** 3,400 effective samples ÷ 40 features
>    is ~85 samples per feature, which is thin; ÷ 18 features is ~190, which is defensible.
>    §6.3 lists which 18.
> 2. **The 12-month holdout carries ~375 effective samples, and once §11's threshold filters
>    most bars out the surviving *trade* count will be under the ~100-trade minimum from
>    §9.3.** So the holdout tells you the sign of the edge, not its size. **This makes
>    `decisions.jsonl` (§11) relatively more important** — it's how you accumulate real
>    out-of-sample evidence over the following months that the holdout can't give you
>    up front.
>
> Neither is a blocker. They just mean "promising" is the best verdict available at this
> size, and the self-scoring log is what eventually upgrades it.

> ⚠️ **Below ~10 tickers, the cross-sectional features degenerate.** `rs_rank` is a
> percentile *across the universe* on each day — with 2 tickers it can only ever be 0 or 1,
> and with 1 it's a constant. Same for `breadth_above_ema50`. The feature registry drops
> both automatically when `len(universe) < 10` and logs that it did. `rs_mkt_63` still
> works at any size, since it's measured against `^GSPC` rather than against peers — so you
> don't lose relative strength entirely, just the cross-sectional ranking of it.

If you want to add tickers later, do it — more is strictly better for the statistics, and
the only cost is a slightly longer fetch. There's no schema to migrate (§5).

### Validate before downloading

```bash
tdm universe validate
```

| Check | Why |
|---|---|
| Symbol resolves | Catches typos and wrong suffixes — Yahoo wants `BRK-B`, not `BRK.B` |
| Has ≥ 9 years of history | A 2024 IPO can't supply 9 years; it'll be NaN for every 200-day feature |
| Not delisted | |
| Median volume above the liquidity floor | A thin stock gets gated out at decision time anyway (§11) |
| Duplicates, blank lines, whitespace | |

### Other practical details

- **Retries.** 3 attempts with exponential backoff. If a ticker still fails, drop it with a
  loud warning and continue — one bad symbol shouldn't kill the whole run.
- **Validate what comes back** before computing on it: `high >= low`, `high >= open/close`,
  no negative prices, no duplicate dates, sorted index, no all-zero-volume stretches.
  Cheap, and it catches provider glitches before they become confusing indicator values.

```bash
tdm fetch                              # everything in universe.txt, 9 years
tdm fetch --symbols NVDA,AMD --years 9
```

---
## 6. Features

### 6.1 The contract

Every feature is a pure function from a price series to a numeric series, registered with
its metadata:

```python
@dataclass(frozen=True)
class FeatureSpec:
    name: str
    fn: Callable[[pd.DataFrame], pd.Series]
    warmup: int                   # leading rows that will be NaN
    tier: Literal["core", "extended"]

registry.register(FeatureSpec("rsi_14", lambda df: talib.RSI(df.close, 14),
                              warmup=14, tier="core"))
```

`warmup` is declared rather than inferred, so the pipeline drops exactly the right number
of leading rows instead of guessing. A 200-day EMA is NaN for its first 200 bars, and
silently filling those with zeros would quietly poison the training set.

**No I/O in this layer.** That's what makes §9.4's leakage tests possible — you can call
any feature on a synthetic 50-row DataFrame.

### 6.2 The rule that matters most

> **Every feature must be scale-free: a ratio, a bounded oscillator, or a trailing
> z-score. Never raw dollars, never an unbounded cumulative sum.**

A decision tree learns literal threshold comparisons: `if macd > 1.84`. If a feature is
measured in dollars, that rule is specific to one ticker at one price level and transfers
to nothing. Train across a multi-ticker universe and such features contribute pure noise.

The everyday analogy: alerting on raw bytes-free instead of percent-free across a fleet of
different disk sizes. "Alert below 10 GB" is meaningless when the disks range from 50 GB to
50 TB. Same bug, same fix — divide by something that sets the scale.

| Raw quantity | Problem | Stored as |
|---|---|---|
| price, EMA, Bollinger bands, VWAP, Ichimoku lines | dollars | `x/close − 1`, or distance in ATRs |
| MACD, histogram, momentum | dollars | ÷ ATR |
| OBV, A/D line | unbounded cumulative sum | 21-day slope, z-scored over 252 days |
| volume | varies 1000× across tickers | `log(volume / median20)` |
| ATR | dollars | ÷ close |
| RSI, Stochastic, ADX, MFI, CCI, Aroon, %B | already bounded ✓ | as-is |
| anything long-tailed | outliers dominate the splits | winsorize at 1/99th pct, or 252-day rolling percentile |

> ⚠️ Rolling z-scores and percentiles use **trailing windows only** — `min_periods` set,
> never centered, and never `.fit()` on the full series. A centered window or a global
> `StandardScaler` reaches into the future (§9.4 #3).

This is the most common reason a cross-sectional indicator model fails to generalize, and
it's invisible: the model trains fine and scores fine in-sample.

### 6.3 The complete feature list

The list answers six questions in sequence, which is what a swing trade actually is:

```
1. Is this stock trending?                → TREND       (is there anything to ride?)
2. Is it stronger than the market?        → RELATIVE    (is it the best horse?)
3. Has it pulled back enough to enter?    → MOMENTUM    (entry timing)
4. Is volume confirming the move?         → VOLUME      (is the move real?)
5. Is volatility sane enough to size it?  → VOLATILITY  (can I risk-manage it?)
6. Is the market regime favourable?       → MARKET      (should I trade at all?)
```

If a candidate feature doesn't help answer one of those, it shouldn't be in the list.

> #### ⚠️ Match the feature count to the universe size
>
> The full list is **~118 features**. At 15 tickers you have ~3,400 effective samples
> (§9.1), so using all of them would be ~29 samples per feature — hopeless. Even the
> 40-feature CORE set is ~85 per feature, which is thin.
>
> So there are three tiers, and the right one depends on how many tickers you're running:
>
> | Tickers | Set | Features | Samples per feature |
> |---|---|---|---|
> | 5–20 | **`minimal`** | **18** | ~190 at 15 tickers ✓ |
> | 20–40 | `core` | 40 | ~170 at 30 tickers ✓ |
> | 40+ | `core`, then `extended` | 40 → ~117 | ~285 / ~97 at 50 tickers |
>
> **At the planned 15 tickers, use `minimal`.** Add features only after the smaller set
> shows signal above base rate — **if `minimal` shows nothing, more features won't rescue
> it**, they'll just make the overfitting harder to detect. The `minimal` 18 are marked
> **MIN** in the tables below.
>
> Rule of thumb: keep **≥100 effective samples per feature**. `tdm dataset build` warns
> when the selected set violates it for your universe.

#### 1 · TREND

| Feature | Params | Stored as | Warm-up | Tier |
|---|---|---|---|---|
| `close_vs_ema21/50/200` | 21, 50, 200 | `close/EMA − 1` | 200 | **MIN** |
| `close_vs_ema8` | 8 | `close/EMA − 1` | 8 | EXT |
| `ema_stack` | 8/21/50/200 | ordinal 0–4: how many are in ascending order | 200 | **MIN** |
| `ema21_slope_atr` | 21, 5-bar slope | slope ÷ ATR | 26 | **CORE** |
| `ema_cross_age` | 8×21 | signed bars since last cross | 21 | EXT |
| `adx_14` | 14 | raw 0–100 ✓ — trend *strength*, no direction | 28 | **MIN** |
| `di_diff` | 14 | `(+DI − −DI)` — the direction half | 28 | **MIN** |
| `aroon_up/down/osc` | 25 | raw 0–100 ✓ — measures *time since* extremes | 25 | EXT |
| `supertrend_state` | 10, 3×ATR | −1 / +1 | 24 | **CORE** |
| `supertrend_dist_atr` | 10, 3×ATR | distance to flip ÷ ATR | 24 | EXT |
| `ichimoku_tenkan/kijun_pos` | 9, 26 | `close/line − 1` | 52 | EXT |
| `ichimoku_cloud_pos` | 9,26,52 | above / inside / below (ordinal) | 78 | EXT |
| `ichimoku_cloud_thick_atr` | 9,26,52 | thickness ÷ ATR | 78 | EXT |
| `linreg_slope_atr_21/63` | 21, 63 | regression slope ÷ ATR | 63 | EXT |
| `linreg_r2_21/63` | 21, 63 | R² 0–1 ✓ — trend *quality*, not direction | 63 | EXT |

#### 2 · RELATIVE STRENGTH & MARKET REGIME

Computed against the **market index** (`^GSPC`) and the **peer median** of your own
universe. No ETFs involved — see the box below.

| Feature | Params | Stored as | Warm-up | Tier |
|---|---|---|---|---|
| `rs_mkt_21/63` | 21, 63 | ROC of `(close / ^GSPC)` | 63 | **MIN** |
| `rs_mkt_126` | 126 | same | 126 | EXT |
| `rs_rank` | 63 | **cross-sectional percentile** of `rs_mkt_63` across the universe, per day | 63 | **MIN** |
| `rs_peer_63` | 63 | ROC of `(close / median of your universe)` | 63 | EXT |
| `beta_mkt_63` | 63 | regression beta | 63 | EXT |
| `corr_mkt_63` | 63 | correlation −1..1 ✓ | 63 | EXT |
| `mkt_above_ema200` | 200 | 0/1 | 200 | **MIN** |
| `mkt_vol_rank` | 21, 252 | percentile of `^GSPC` realized vol | 252 | **CORE** |
| `vix_rank` | 252 | percentile of VIX | 252 | **CORE** |
| `vix_level` | — | raw | 1 | EXT |
| `vix_term_structure` | — | VIX3M / VIX — below 1 signals stress | 1 | EXT |
| `breadth_above_ema50` | 50 | fraction of universe above own EMA50 | 50 | EXT |

> #### Reference series — not part of your universe
>
> Two of these features compare a stock against "the market", which needs a market series
> to compare against. That series is the **S&P 500 index, `^GSPC`** — an index, not an ETF.
> `^VIX` likewise. They are **reference data only**: downloaded, used to compute columns,
> and never traded, never scored, never rows in the dataset (§8.1).
>
> This matters because without a market series you lose the entire RELATIVE family —
> including `rs_rank`, which is the single strongest daily-horizon feature in published
> equity work, and `mkt_above_ema200`, which is the regime split long and short setups
> behave most differently across. Three of the 18 `minimal` features are in this family.
>
> **For sector-style comparison the design uses the peer median of your own universe** —
> the equal-weighted median return of the tickers you chose. No extra downloads, and it
> adapts to whatever list you run. ⚠️ With ~15 tickers that median is noisy, which is why
> `rs_peer_63` and `dow_peer_confirm` are both EXTENDED rather than MIN.
>
> Two practical notes: `^VIX` has **no volume** (it's a calculated index, not a traded
> instrument), so only its price is usable — fine, since that's all these features need.
> And `rs_rank` is the only feature needing the **whole universe at once** rather than one
> ticker at a time, since it's a cross-sectional rank per day; that breaks the otherwise
> per-ticker shape of the pipeline, and the groupby is easy to get wrong.

#### 3 · MOMENTUM / MEAN REVERSION

| Feature | Params | Stored as | Warm-up | Tier |
|---|---|---|---|---|
| `rsi_14` | 14 | raw 0–100 ✓ | 14 | **MIN** |
| `rsi_2` | 2 | raw 0–100 ✓ — short-term pullback depth | 2 | **MIN** |
| `rsi_divergence` | from pivots | −1 / 0 / +1 (§6.4) | ~60 | EXT |
| `stoch_k`, `stoch_d` | 14,3,3 | raw 0–100 ✓ | 17 | **CORE** |
| `stoch_kd_diff` | 14,3,3 | `%K − %D` | 17 | EXT |
| `macd_hist_atr` | 12,26,9 | histogram ÷ ATR | 35 | **MIN** |
| `macd_line_atr`, `macd_signal_atr` | 12,26,9 | ÷ ATR | 35 | EXT |
| `macd_cross_age` | 12,26,9 | signed bars since cross | 35 | EXT |
| `ppo` | 12,26 | `(EMA12−EMA26)/EMA26 × 100` — inherently scale-free, a cleaner MACD | 26 | EXT |
| `roc_21/63` | 21, 63 | percentage ✓ | 63 | **CORE** |
| `roc_5/10` | 5, 10 | percentage ✓ | 10 | EXT |
| `cci_20` | 20 | raw ✓ | 20 | EXT |
| `williams_r_14` | 14 | raw −100..0 ✓ | 14 | EXT |

> **MACD is measured in dollars**, so AAPL's MACD and a $20 stock's MACD aren't
> comparable. Always stored ÷ ATR. `ppo` is the same idea with the normalization built in.

#### 4 · VOLATILITY

| Feature | Params | Stored as | Warm-up | Tier |
|---|---|---|---|---|
| `atr_pct` | 14 | `ATR/close` | 14 | **MIN** |
| `atr_pct_rank` | 14, 252 | trailing percentile | 252 | **MIN** |
| `bb_pctb` | 20, 2σ | `(C−lower)/(upper−lower)` | 20 | **MIN** |
| `bb_bandwidth` | 20, 2σ | `(upper−lower)/mid` | 20 | **CORE** |
| `bb_bandwidth_rank` | 20, 252 | percentile — low = squeeze | 252 | **MIN** |
| `donchian_pos_20` | 20 | position in channel 0–1 ✓ | 20 | **CORE** |
| `donchian_pos_55` | 55 | same | 55 | EXT |
| `pct_from_52w_high` | 252 | `close/max(high,252) − 1` | 252 | **CORE** |
| `pct_from_52w_low` | 252 | `close/min(low,252) − 1` | 252 | EXT |
| `keltner_pos` | 20, 2×ATR10 | position in channel | 20 | EXT |
| `squeeze_on` | BB vs Keltner | 0/1 — BB inside Keltner | 20 | EXT |
| `realized_vol_21` | 21 | stdev of log returns, annualized | 21 | EXT |
| `parkinson_vol_21` | 21 | high/low range estimator — more efficient than close-to-close, which matters with only 9 years | 21 | EXT |
| `vol_of_vol` | 21, 63 | stdev of `atr_pct` | 63 | EXT |

> Bollinger Bands are stored as `%B` and `bandwidth`, never as raw band *prices* — those
> are in dollars (§6.2).

#### 5 · VOLUME

| Feature | Params | Stored as | Warm-up | Tier |
|---|---|---|---|---|
| `volume_rel_median` | 20 | `log(volume / median20)` | 20 | **MIN** |
| `volume_z` | 63 | trailing z-score | 63 | EXT |
| `obv_slope_z` | 21 slope, 252 z | see note below | 252 | **CORE** |
| `obv_price_divergence` | 21 | sign disagreement between OBV slope and price slope | 252 | EXT |
| `mfi_14` | 14 | raw 0–100 ✓ — volume-weighted RSI | 14 | **MIN** |
| `ad_slope_z` | 21, 252 | Chaikin A/D line slope, z-scored | 252 | EXT |
| `cmf_20` | 20 | Chaikin Money Flow, bounded ✓ | 20 | EXT |
| `close_vs_rvwap_20` | 20 | `close/rolling VWAP − 1` | 20 | **CORE** |
| `close_vs_rvwap_50` | 50 | same | 50 | EXT |
| `close_vs_avwap_swinglow` | anchored | `close/AVWAP − 1` from the last confirmed swing low | ~60 | **CORE** |
| `close_vs_avwap_swinghigh` | anchored | same, from the last swing high | ~60 | EXT |
| `close_vs_avwap_ytd` | anchored | from 1 Jan | ≤252 | EXT |

> **OBV is a running cumulative sum**, so its absolute level depends on how far back your
> data happens to start — unusable raw. Stored as a z-scored 21-day slope.

> **There is no plain daily VWAP, deliberately.** VWAP accumulates from the market open and
> **resets every day** — it's an intraday metric. Computed on a single daily bar it reduces
> to `typical_price × volume / volume` = the typical price, which you already have from the
> OHLC. Zero new information. (Many online tutorials get this wrong.) The two variants that
> *do* carry information at daily granularity are a **rolling** VWAP over trailing N days,
> and an **anchored** VWAP accumulating from a chosen event — the last swing low, or the
> start of the year. Anchored-VWAP-from-swing-low answers "what did everyone who bought
> since the bottom pay?", which is a real support level and not a near-linear function of
> the moving averages you already have.

#### 6 · PRICE STRUCTURE

Dow Theory and Elliott Wave are **interpretive human frameworks, not formulas**. Automated
Elliott wave counting is famously ambiguous — competent practitioners disagree on the same
chart, and any count gets revised as new bars arrive. So rather than claiming to compute
"we are in wave 3", both reduce to one shared primitive and we extract measurable features
from it.

**The primitive: ZigZag swing pivots.** Find local maxima and minima, ignoring wiggles
below a threshold:

```python
def pivots(bars, threshold) -> list[Pivot]:
    """Alternate between seeking a high and a low. Confirm a pivot only once
    price has reversed by >= threshold from the running extreme.
    threshold = 3 * ATR(14) for swings, 8 * ATR(14) for major structure."""
```

> #### ⚠️ ZigZag is a lookahead trap — the highest-risk item in the feature set
>
> Every charting tool draws ZigZag *with hindsight*. The pivot sitting at bar `t` only
> becomes **knowable** at bar `t+k`, once the reversal has actually happened. Label pivots
> over the full series, feed them as features, and you've handed the model the future. The
> backtest will look spectacular and mean nothing.
>
> So every pivot carries two timestamps:
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
> the feature builder, asserted in `tests/test_leakage.py`, and **visible instantly on a
> chart** (§7) — which is the main reason charts come before training.

| Feature | Stored as | Tier |
|---|---|---|
| `dow_trend` | −1 / 0 / +1: `+1` if the last two swing highs **and** last two swing lows are both rising, `−1` if both falling | **MIN** |
| `dow_trend_age` | bars since it last changed | **CORE** |
| `dist_to_swing_high_atr` | distance to the last confirmed swing high ÷ ATR | **CORE** |
| `dist_to_swing_low_atr` | same, swing low | EXT |
| `dow_trend_major` | `dow_trend` from the wider 8×ATR ZigZag | EXT |
| `dow_mtf_agree` | `dow_trend == dow_trend_major` — Dow's primary-vs-secondary trend idea | EXT |
| `swing_hh/hl/lh/ll` | 4 booleans on the latest completed leg | EXT |
| `dow_volume_confirm` | `+1` if volume expands with the trend and contracts against it | EXT |
| `dow_peer_confirm` | ticker `dow_trend` == peer-median `dow_trend` | EXT |
| `leg_retrace_ratio` | current leg's retracement of the previous, as a fraction | EXT |
| `fib_zone` | ordinal band: <.236 / .236–.382 / .382–.5 / .5–.618 / .618–.786 / >.786 | EXT |
| `leg_extension_ratio` | current leg ÷ previous leg | EXT |
| `legs_since_major` | alternating legs since the last major pivot | EXT |
| `impulse_vs_corrective` | −1 / +1: is the current leg longer and steeper than the prior counter-leg | EXT |
| `leg_slope_atr` | leg price change per bar ÷ ATR | EXT |

> **Honest expectation:** the Dow structure features (`dow_trend`, `dow_trend_age`,
> `dow_mtf_agree`) should earn their place — they encode trend persistence in a form trees
> split on cleanly. **The Elliott/Fibonacci features I expect to be weak.**
> `leg_retrace_ratio` plausibly carries something, since retracement depth does relate to
> continuation odds; the specific Fibonacci boundaries are a hypothesis to test, not a
> prior to trust. §6.4's held-out permutation importance will tell us. If they come out
> flat, delete them — that's a successful experiment, not a failure.

#### 7 · BAR SHAPE, GAPS & CALENDAR

| Feature | Stored as | Tier |
|---|---|---|
| `clv` | `(close−low)/(high−low)` — where in the day's range it closed | **CORE** |
| `gap_atr` | `(open − prev close)/ATR` | **CORE** |
| `large_gap_due` | 0/1 — the earnings proxy, see below | **MIN** |
| `range_atr` | `(high−low)/ATR` | EXT |
| `consec_updown` | signed run length of up/down closes | EXT |
| `days_since_large_gap` | bars since a gap exceeded 3×ATR | EXT |
| `quarterly_cycle_pos` | position in the ~63-bar cycle since the last large gap | EXT |
| `day_of_week` | 0–4 categorical | EXT |
| `sector` | categorical (~11 values) — CatBoost handles natively | EXT |

> **The earnings proxy.** Earnings reports are the most dangerous scheduled event for an
> open position: the price can gap 10% overnight, straight through a stop-loss, and at 10×
> leverage that wipes the position's entire margin (§10.1). It's also the only tail risk
> here that's knowable in advance.
>
> A real earnings calendar would be better, but `yfinance`'s dates are unreliable and carry
> no as-of history, which creates its own leakage problem. So the design derives a proxy
> from price data alone: earnings land on a quarterly cadence, so **past large gaps predict
> future ones.** `days_since_large_gap` + `quarterly_cycle_pos` → `large_gap_due`.
>
> Weaker than a real calendar — it will mis-date events and confuse earnings gaps with
> news gaps — but it needs no extra data source and captures most of the "don't open a new
> position right now" value. A real calendar stays available behind a config flag.

#### Totals

| Family | MIN | CORE | EXTENDED |
|---|---|---|---|
| Trend | 4 | 8 | 17 |
| Relative & market | 3 | 6 | 8 |
| Momentum | 3 | 7 | 12 |
| Volatility | 4 | 7 | 11 |
| Volume | 2 | 6 | 10 |
| Structure | 1 | 3 | 12 |
| Bar shape & calendar | 1 | 3 | 7 |
| **Total** | **18** | **40** | **77** |

`MIN ⊂ CORE ⊂ CORE+EXTENDED`. The 18 MIN features are the highest-conviction member of
each family — enough to answer all six questions above with the fewest columns.

> #### If you only implement five
>
> Ranked by my confidence they carry real signal at this horizon:
>
> 1. **`rs_rank`** — relative strength vs the universe. Repeatedly the strongest single
>    daily-horizon feature in published equity work, and the only thing here that can
>    distinguish "NVDA rose 5%" from "NVDA rose 5% while the market rose 9%" — which are
>    opposite signals.
> 2. **`adx_14`** — separates trending from choppy, the most useful first split a tree can
>    make.
> 3. **`ema_stack`** — cheap, robust summary of multi-timeframe trend agreement.
> 4. **`atr_pct_rank`** — volatility regime; gates sizing and flags when barriers are
>    unusually wide.
> 5. **`mkt_above_ema200`** — long and short setups behave completely differently either
>    side of this line.

### 6.4 Redundancy

RSI, Stochastic, CCI, Williams %R and MFI are all bounded momentum oscillators and are
heavily mutually correlated (|ρ| often > 0.8). Trees tolerate correlated inputs, but there
are two real costs: **feature importances become meaningless** (importance splits
arbitrarily among near-duplicates, so you can't tell what the model is using), and
effective complexity rises with no new information.

1. Compute the correlation matrix at build time → `reports/feature_corr.html`.
2. Greedily drop one of any pair with |ρ| > 0.95, keeping the more interpretable one.
   **Log every drop** — visible and reviewable, never silent.
3. Report **permutation importance on held-out folds**, not sklearn's
   `feature_importances_`. The built-in attribute is impurity-based, biased toward
   high-cardinality features, and computed in-sample. It will mislead you.

Expect the usable extended set to land nearer 60 after dedup.

---

## 7. Charts

Interactive HTML per ticker: price with every overlay, plus linked indicator panels.

```bash
tdm chart --ticker NVDA                  # -> reports/charts/NVDA.html
tdm chart --all
tdm chart --ticker NVDA --with-labels    # overlay the TP/SL outcomes
tdm chart --ticker NVDA --with-decisions # overlay the model's calls
```

**Library: Plotly.** `go.Candlestick` + `make_subplots` + `write_html` gives one
self-contained file with zoom, pan and hover tooltips — no server, no build step.

```
┌────────────────────────────────────────────────────────────┐
│ NVDA   2017-11-03 → 2026-11-03   9y daily                  │
├────────────────────────────────────────────────────────────┤
│ PRICE  candlesticks                                   60%  │
│        + EMA 8/21/50/200        + Bollinger(20,2)          │
│        + Keltner  + Supertrend  + Ichimoku cloud           │
│        + anchored VWAP          + ZigZag pivots ◆          │
│        [--with-labels]    TP/SL brackets, green=win red=loss│
│        [--with-decisions] ▲ LONG  ▼ SHORT  · HOLD          │
├────────────────────────────────────────────────────────────┤
│ VOLUME bars coloured by up/down + 20d median line      8%  │
├────────────────────────────────────────────────────────────┤
│ RSI(14) + RSI(2), bands at 30/70                       8%  │
├────────────────────────────────────────────────────────────┤
│ MACD/ATR line + signal + histogram                     8%  │
├────────────────────────────────────────────────────────────┤
│ ADX(14) + (+DI − −DI), bands at 20/25                  8%  │
├────────────────────────────────────────────────────────────┤
│ Stochastic %K/%D  ·  MFI  ·  OBV slope z-score         8%  │
└────────────────────────────────────────────────────────────┘
      shared, linked x-axis — zoom one panel, all follow
```

### Three rules

**1. Render from the feature matrix, never recompute.**

```python
# RIGHT: chart the exact numbers the model will see
features = pd.read_parquet(f"data/features/{run_id}.parquet")
plot(features.loc[ticker])

# WRONG: a second implementation that can silently disagree
plot(talib.RSI(bars.close, 14))
```

If the chart draws its own indicators, then when a value looks wrong you don't know
whether the bug is in the pipeline or the plot — and worse, **the chart can look perfect
while the model trains on garbage.** Charting the actual feature columns makes the picture
*evidence* about the model's input.

**2. Charts come before training, because visual checks catch what tests don't.**

Unit tests confirm RSI(14) matches a reference value. They do not catch: an indicator off
by one bar, a ZigZag pivot drawn with hindsight, a warm-up period filled with 0 instead of
NaN, a feature that's flat for 200 bars, or an anchored VWAP anchored to the wrong pivot.
Thirty seconds looking at `NVDA.html` is one of the highest-value debugging steps in the
project.

**3. Overlay the labels, because the simulator is the hardest part to verify.**

`--with-labels` draws each labeled entry as a TP/SL bracket coloured by outcome. This is
how you sanity-check §8.2 by eye: are the distances sane relative to the candles? Do the
winners look like they were going somewhere? During March 2020, are the brackets enormous?
**They should be** — ATR exploded, so ATR-scaled barriers widened with it. Seeing that
confirms the ATR scaling works.

### Also generated

| Report | Content |
|---|---|
| `reports/feature_corr.html` | correlation heatmap — drives the dedup in §6.4 |
| `reports/feature_dist.html` | per-feature histogram + NaN count. Catches a feature that's constant or 90% NaN in one glance |
| `reports/equity.html` | backtest equity curve per fold (§10), plus the shuffled-signal control |
| `reports/tree.txt` | the depth-4 baseline's rule set as plain text (§9.1) |

---
## 8. The dataset

### 8.1 The table

One row per `(symbol, date)`, written to Parquet, exportable to CSV.

```
┌─ IDENTITY ────────────────────────────── never features ─┐
  symbol                  str        "NVDA"
  date                    date       2026-11-03

┌─ RAW ──────── kept for charting (§7) and simulation; NOT features ─┐
  open high low close volume

┌─ FEATURES (X) ──────── the allowlist from registry.feature_names() ─┐
  close_vs_ema21          float      -0.012
  adx_14                  float      31.4
  rsi_14                  float      58.2
  rs_rank                 float      0.94
  ... 40 CORE columns (§6.3), ~117 with EXTENDED

┌─ TARGETS (y) ───────────────── one set per direction, from §8.2 ─┐
  y_long                  int8       1 = TP hit before SL, 0 = not
  y_short                 int8
  y_long_t1               date       ⚠️ when the label RESOLVED — §9.2 needs this
  y_short_t1              date
  y_long_reason           str        tp | sl | timeout | ambiguous
  y_short_reason          str
  y_long_ret              float      realized return, NET of costs (§10)
  y_short_ret             float
  y_long_hold_days        int8
  y_short_hold_days       int8
  y_long_weight           float      label-uniqueness weight (§8.2)
  y_short_weight          float

┌─ FORWARD RETURNS ─────── diagnostics + optional alternative target ─┐
  fwd_ret_1d              float      close[t+1]/close[t] − 1
  fwd_ret_2d              float
  fwd_ret_5d              float
  fwd_ret_10d             float
  fwd_mfe_10d             float      max FAVOURABLE excursion ÷ ATR
  fwd_mae_10d             float      max ADVERSE excursion ÷ ATR

┌─ META ───────────────────────────────── bookkeeping, never features ─┐
  meta_gate_pass          bool       passed the §11 hard gates (§9.1)
  meta_run_id             str        provenance (§5)
```

```bash
tdm dataset build --set core
tdm dataset export --format csv      # -> data/exports/<run_id>.csv
tdm dataset schema                   # column list, dtypes, tier
```

### Why the target is a barrier outcome, not a price

Two traps that the column layout above steers around.

> #### ⚠️ Never target an absolute price
>
> This is the most common way a financial ML project fools itself, and the failure looks
> like spectacular success.
>
> Target `close[t+1]` as a price and you'll get **R² ≈ 0.999**. You'll think you've built
> something extraordinary. You've built this:
>
> ```
> close[t+1] ≈ close[t]
> ```
>
> Tomorrow's price is almost entirely explained by today's. The model learns the identity
> function, scores brilliantly on every regression metric, and contains **zero** tradeable
> information — and it's hard to notice, because the scatter plot is a perfect diagonal and
> the errors are tiny.
>
> Targets are always **changes**. Do that and R² drops to ~0.00–0.01, which is
> disappointing and true: next-day returns are ~95% noise.

> #### ⚠️ A forward return can't tell you where to put a stop
>
> Suppose you predict `fwd_ret_10d = +4%` perfectly. Should you take the trade, and where
> does the stop go?
>
> You can't tell, because **the endpoint doesn't describe the path.** Two stocks both
> finishing +4% after 10 days:
>
> ```
> A:  +1% +1% +1% +1% ... steady            → your 1-ATR stop never triggers.  WIN
> B:  −9% −3% +2% ... recovers to +4%       → your stop triggered on day 1.    LOSS
> ```
>
> **Identical target value, opposite trade outcomes.** A forward-return model cannot
> distinguish them, which means it cannot answer "will I take profit before I cut loss" —
> the actual question (§1).
>
> That's what §8.2 is for: it simulates the *path* and reports which exit fired first, so
> `y_long` encodes "profit before loss" directly with no post-processing.

### So why keep the `fwd_*` columns?

Three jobs, none of them "be the primary target":

**1. A five-minute check on whether any signal exists.** Correlate every feature against
`fwd_ret_5d` and `fwd_ret_10d`. If *nothing* in ~118 features correlates with *any*
forward horizon, stop and reconsider the features rather than training on them. Goes in
`reports/feature_dist.html`.

**2. `fwd_mfe_10d` / `fwd_mae_10d` make the barrier multiples empirical.** These are the
maximum favourable and adverse excursions over the horizon, in ATR units — how far price
actually travelled in each direction before the window closed:

| Observation | Meaning | Action |
|---|---|---|
| `fwd_mfe_10d` rarely reaches 2.0 | your TP is further than price usually travels | lower `k_tp`, or raise `H` |
| `fwd_mae_10d` routinely exceeds 1.0 | your SL sits inside normal noise — you'll be stopped out constantly | raise `k_sl` |
| both frequently exceeded | barriers are too tight for daily bars | widen both |

**3. An alternative target you can actually test.** The claim that barrier-classification
beats return-regression is an argument, not a proof:

```bash
tdm train --target barrier          # default (§8.2)
tdm train --target fwd_ret_10d      # regression on net 10-day return
tdm train --target fwd_ret_sign     # classification on direction only
```

If regression wins on **expectancy after costs** (§9.3 — not on R², which is the wrong
metric), use it. Cost: one flag.

> #### ⚠️ The leakage bug this table invites
>
> Features and targets live in the same table. Convenient, and one line from catastrophe:
>
> ```python
> X = df.drop(columns=["y_long"])        # ← LEAK. fwd_ret_10d is still in X.
> ```
>
> You just handed the model the answer. ~99% accuracy, metrics that look incredible, a
> backtest that looks like a money printer, and **no error message.**
>
> **Two mechanisms, both required:**
>
> 1. **Column prefixes are load-bearing.** Every non-feature column carries a mandatory
>    prefix: `y_`, `fwd_`, `meta_`. A bare name means it's a feature. This makes a leak
>    visible when *reading* code.
> 2. **Feature selection is an allowlist, never a denylist:**
>
>    ```python
>    X = df[registry.feature_names()]       # ✅ explicit, from §6.3
>    X = df.drop(columns=TARGETS)           # ❌ silently leaks any new column
>    ```
>
>    With an allowlist, adding `fwd_ret_20d` tomorrow is inert — it isn't registered, so it
>    can't reach the model. With a denylist, the same addition silently poisons every model
>    you train afterwards.
>
> `tests/test_leakage.py` asserts no column matching `^(y_|fwd_|meta_)` appears in `X`, and
> that `set(X.columns) == set(registry.feature_names())` exactly.

### 8.2 Simulating the trade: the triple barrier

The **triple-barrier method** (López de Prado, *Advances in Financial Machine Learning*).
Rather than predicting a price, simulate the trade and ask **which exit fires first.** It's
a race between three conditions — close to `asyncio.wait(..., FIRST_COMPLETED)`:

```
     price
       │
  TP ──┼─────────────────────────────  ← profit barrier    → y = 1
       │        ╱╲
 entry─┼──────╱────╲──╱──────────────
       │    ╱        ╲╱
  SL ──┼──────────────────────────────  ← loss barrier      → y = 0
       │
       └────────────────────────┤
        t+1                    t+10     ← timeout barrier   → y = sign(net return)
```

```python
def label_one(bars, t: int, side: Side, cfg) -> Label:
    atr   = bars.atr_14.iloc[t]        # known at t's close — no future data
    entry = bars.open.iloc[t + 1]      # decide at t's close, fill at next open

    if side is Side.LONG:
        tp, sl = entry + cfg.k_tp * atr, entry - cfg.k_sl * atr
    else:
        tp, sl = entry - cfg.k_tp * atr, entry + cfg.k_sl * atr

    for i in range(t + 1, min(t + 1 + cfg.horizon, len(bars))):
        hi, lo = bars.high.iloc[i], bars.low.iloc[i]
        hit_tp = (hi >= tp) if side is Side.LONG else (lo <= tp)
        hit_sl = (lo <= sl) if side is Side.LONG else (hi >= sl)

        if hit_tp and hit_sl:          # same bar touched BOTH
            return Label(y=0, t1=bars.index[i], reason="ambiguous")
        if hit_tp:
            return Label(y=1, t1=bars.index[i], reason="tp")
        if hit_sl:
            return Label(y=0, t1=bars.index[i], reason="sl")

    exit_px = bars.close.iloc[t + cfg.horizon]
    net = net_return(entry, exit_px, side, cfg.costs)
    return Label(y=int(net > 0), t1=..., reason="timeout", net_return=net)
```

### What "horizon" means

**The horizon is the third barrier: the maximum trading days you hold before giving up and
closing at whatever the price happens to be.** `H = 10` is about two calendar weeks. It's a
timeout, in the ordinary programming sense.

Three reasons it exists, in order of importance:

1. **Without it, labels may never resolve.** A position that drifts sideways forever has no
   outcome, so there's no `y` to train on. The horizon guarantees every sample resolves and
   bounds how far into the future a label reaches — which is also what makes the purging in
   §9.2 possible.
2. **Capital has an opportunity cost.** A position going nowhere occupies margin and a
   position slot that a better setup could use.
3. **Financing accrues daily** (§10.1). On a leveraged CFD a stagnant position isn't
   neutral — it bleeds a little every day.

**Why 10?** It should match the TP distance. With TP at 2 × ATR you're asking for roughly
two average days of movement *in one direction*, and random walks take considerably longer
than two days to travel two ATRs. Empirically ~5–15 days is where a 2-ATR move either
happens or the setup has gone stale.

It's a config value, not a law. `tdm labels sweep` reports the timeout fraction: **if most
trades are timing out, `H` is too short** or the TP is too far; if almost none are, `H`
isn't doing any work and you're paying financing for nothing.

> #### ⚠️ Why `H` can't be unbounded ("hold until it hits one or the other")
>
> Scanning forward indefinitely until a barrier fires is the natural reading of "will it
> hit TP or SL *eventually*", and it fails for four separate reasons — any one of which
> would be disqualifying.
>
> **1. It destroys cross-validation.** §9.2 purges every training sample whose `[t, t1]`
> interval overlaps the test fold. That works because `t1 ≤ t + 10`. With unbounded `t1`,
> a sample from 2019 can resolve in 2023, so purging a 2023 test fold would remove most of
> the training set. There is no valid split left.
>
> **2. Effective sample size collapses.** Overlapping labels are not independent (decision
> 5 below), so effective samples ≈ `rows ÷ H`:
>
> | Horizon | Effective samples (15 tickers) |
> |---|---|
> | **10** | **~3,400** ✓ |
> | 20 | ~1,700 |
> | 30 | ~1,100 |
> | 60 | ~570 |
> | unbounded (~100-day average) | **~340** ✗ |
>
> At ~340 you cannot fit anything that generalizes — even the 18-feature `minimal` set
> would be ~19 samples per feature.
>
> **3. Some labels never resolve.** A position that drifts sideways until the data runs out
> has no outcome. Those rows get dropped, and they're concentrated in the **most recent**
> months — exactly the period you care most about.
>
> **4. Financing eats the win.** At 0.02%/day on notional (§10.1), a 200-day hold costs 4%
> of notional — which at 10× leverage is **40% of the margin posted.** The trade can hit
> its TP on price and still lose money on carry.
>
> There's also a statistical curiosity worth knowing: with infinite time and barriers at
> +2 / −1 ATR, a driftless random walk hits the nearer barrier first about two-thirds of
> the time, giving a base rate near 33% — which is almost exactly break-even before costs
> (§10.3). So removing the timeout doesn't just break the machinery; it converges the label
> toward "no edge by construction".
>
> **What the design does instead** is the practical version of "eventually": scan forward
> up to `H`, and if neither barrier has fired, close at market and label by whether the
> result was profitable net of costs. Every row resolves, `t1` stays bounded, and nothing
> is discarded.
>
> If 10 days feels short, sweep it — `tdm labels sweep --horizon 10,20,30` prices the
> trade-off in effective samples against timeout fraction, and you can pick with the
> numbers in front of you. Just don't set it to infinity.

### Five design decisions

**1. Same-bar ambiguity resolves pessimistically.** If a daily bar's high reaches TP *and*
its low reaches SL, the OHLC **cannot tell you which came first** — the intrabar path isn't
in the data. This is genuine information loss, not a coding problem. **We assume the loss.**
Resolving these optimistically is a classic silent backtest inflator, and it's worst exactly
where it hurts most: in high-volatility bars, which is where wide-TP trades live. Dropping
them instead would bias the training set toward calm periods. The ambiguous count is logged
every run; above ~5% of samples, the barriers are too tight for daily bars.

**2. Entry price must be knowable when you decide.** This sounds obvious and is the
easiest place in the whole design to introduce a lookahead bug, so it gets its own
treatment below.

**3. Costs go inside the label, not bolted on afterward.** `net_return` subtracts the CFD
spread, slippage (a fraction of ATR — execution noise scales with volatility, so fixed
basis points would be wrong), and daily financing on the notional for both directions
(§10.1). Because the timeout branch labels on `net_return > 0`, the model learns *post-cost*
profitability directly. Given that costs move break-even from 33% to ~42% (§10), having
them inside the label is what keeps the training target honest.

**4. Two binary models, not one three-class model.** The obvious framing is a 3-class label
(LONG / SHORT / HOLD). Two problems: with asymmetric barriers (TP 2 ATR, SL 1 ATR) the
up-bet and down-bet are **not mirror images**, so one label can't encode both; and "HOLD"
would need an invented dead-zone threshold **baked into the labels, where you can never
re-tune it.** Instead:

```python
M_long  : P(long  hits TP before SL | features)      # binary
M_short : P(short hits TP before SL | features)      # binary
```

HOLD *emerges at inference time* as "neither probability clears its threshold" (§11), so
the thresholds become runtime knobs you tune against results. It also lets each direction
learn its own feature relationships.

**5. Sample weights, because the labels overlap.** Bar `t` and bar `t+1` both look 10 days
ahead, so **consecutive samples share most of their future** and are not independent.
Standard CV treats them as if they were, which inflates every metric. Two corrections:

- **Uniqueness weight** — for each sample, average `1/concurrency(bar)` over `[t, t1]`,
  where `concurrency(bar)` counts how many live labels span that bar. Passed as
  `sample_weight` so crowded periods don't dominate.
- **Purging and embargo** in cross-validation — §9.2.

**Practical consequence:** 15 tickers × ~2,268 bars ≈ 34,000 rows, but the **effective
independent sample count is ~3,400.** That smaller number is what governs model
complexity, and it's why §6.3 recommends 18 features rather than 118.

### Entry price: what you can and can't use

The simulator needs an entry price for every row. The constraint is that it must be a price
**you could actually have transacted at, using only information available when the decision
was made** (bar `t`'s close).

> #### ⚠️ Why the mid-range price `(high + low) / 2` cannot be used
>
> Entering at the midpoint of the day's range is an appealing idea — it looks like a fair,
> average fill rather than an optimistic one. It is unfortunately **not implementable**, and
> it inflates results badly.
>
> The problem: `high[t]` and `low[t]` are not known until day `t` has closed. By the time
> you can compute the midpoint, the day is over and that price is gone. Using it means the
> simulator transacts at a price that was only identifiable in hindsight.
>
> **How much it's worth.** The midpoint is, by construction, guaranteed to sit inside the
> day's range — so a midpoint entry can *never* have bought the high or sold the low. A
> real entry has no such guarantee. That structural advantage is worth roughly half the
> daily range:
>
> | | |
> |---|---|
> | Position notional (the §10.2 example) | $4,768 |
> | Daily range ≈ ATR | ~2% of price |
> | Half the range | 1.0% = **$48** |
> | Risk budget per trade | $100 |
> | **Phantom edge from a mid-range fill** | **$48 — 48% of the risk budget, 24% of the TP target** |
>
> A free $48 per trade against a $100 risk budget turns a losing strategy into a winning
> one on paper. The backtest would look excellent and live results would not resemble it,
> with no error anywhere to explain the gap.
>
> This is the §9.4 bug class in its most tempting form: a reasonable-sounding entry
> assumption that quietly reads the future.

### The implementable options

`entry_mode` in `config.toml` selects between these. All three use only information
available at bar `t`'s close.

| `entry_mode` | Entry price | Notes |
|---|---|---|
| **`next_open`** (default) | `open[t+1]` | Decide on tonight's close, order fills at tomorrow's opening print. Simplest, always fills, and the most conservative of the three — you accept whatever the overnight gap did. |
| `close` | `close[t]` | You place the order into the closing minutes. Realistic on a CFD platform that trades continuously, and it removes overnight gap risk *from the entry* (not from the position). Slightly more aggressive, since you're deciding on a price you then transact at. |
| `limit_pullback` | `close[t] − offset × ATR` for a long, `+` for a short | ⬇ see below |

> #### `limit_pullback` — the honest version of "enter somewhere during the day"
>
> If the intent behind a mid-range entry is *"I want a better price than the open, by
> entering on an intraday dip"*, this is how to express that without lookahead:
>
> 1. At bar `t`'s close, compute the limit price from `close[t]` and `ATR[t]` — both known
>    now. Default `offset = 0.25`.
> 2. Place a resting limit order for bar `t+1`.
> 3. **It fills only if `low[t+1] <= limit`** (for a long). Otherwise there is no trade,
>    and the row is recorded as `y_long_reason = "no_fill"` and excluded from training.
>
> This captures the real benefit of a patient entry — a better average price — while also
> capturing its real cost, which the mid-range version hides: **you miss the trades that
> run away from you immediately.** Those are disproportionately the *good* ones, so the
> better fills come at the price of lower coverage. That trade-off is genuine and
> measurable; the mid-range version gives you the upside with none of it.
>
> Worth testing against `next_open` once the baseline works — `tdm labels sweep` reports
> fill rate alongside base rate for each mode. Not the place to start, though: the
> no-fill handling adds a branch to the simulator, the backtest and `decide`, and all
> three have to agree (§10).

> **Leakage test.** `tests/test_leakage.py` asserts that the entry price for bar `t` is a
> function only of columns at index `≤ t`, plus `open[t+1]`/`low[t+1]`/`high[t+1]` where
> the mode legitimately needs the *next* bar for fill determination. Any reference to
> `high[t]` or `low[t]` in entry-price computation fails the test.

### 8.3 The parameter sweep

`k_tp`, `k_sl`, `horizon` and the ATR period form a named *label profile* in `config.toml`,
hashed into the run id. Two settings are two reproducible, comparable experiments rather
than an uncommitted edit.

```bash
tdm labels sweep     # per profile, report:
                     #   base rate            (fraction of positive labels)
                     #   break-even hit rate  (from the §10 cost model)
                     #   margin above break-even   <- the number that matters
                     #   timeout fraction, ambiguity rate
                     #   effective sample count
                     #   median fwd_mfe / fwd_mae in ATR (§8.1)
```

**Run this before training.** In five minutes it tells you whether a profile is worth
modeling at all:

- A **15% base rate** means the setup is rare and you'll learn from very few positives.
- A base rate **below the break-even hit rate** means the profile is unprofitable *even
  with a perfect model* — no tuning saves it.
- Since §10's cost arithmetic puts break-even near 42%, this is where you find out whether
  a wider profile (TP 3 / SL 1.5) amortizes the fixed costs better. **Don't assume
  TP 2 / SL 1 is optimal just because it's the starting point.**

---

## 9. Model and validation

### 9.1 Model choice

| # | Model | Role |
|---|---|---|
| 1 | `DecisionTreeClassifier(max_depth=3..4)` | **Readable baseline.** Print the whole rule set (`reports/tree.txt`) and read it. It tells you whether the model found something sensible (`ADX>25 and EMA stack aligned and RSI<70 → long`) or something absurd. **Keep this permanently** — it's your sanity check on everything above it. |
| 2 | `RandomForestClassifier` | **The control.** Few knobs, hard to misuse, robust to noise. Its job is diagnostic: **if RF beats the boosted model out-of-sample, the boosted model is overfit.** Always report it. |
| 3 | **`CatBoostClassifier`** | **The primary model.** |
| 4 | `xgboost` | Optional refinement, for monotonic constraints. |

> #### ⚠️ Model choice is not the bottleneck
>
> Swapping one tree library for another moves your hit rate by **one or two points.** For
> comparison:
>
> | Issue | Impact on results |
> |---|---|
> | One undetected leak (§9.4) | the entire result — a worthless model that looks excellent |
> | Getting the cost model wrong (§10) | ~8 points of break-even hit rate |
> | Features containing no real signal | the entire result |
> | **Which boosting library** | **1–2 points** |
>
> Spend effort in that order. Model choice is the last ~5% of this problem, and the part
> most people spend 80% of their time on.

### Why CatBoost

- **Ordered boosting** (`boosting_type="Ordered"`) is the real reason. Standard gradient
  boosting computes each tree's gradients using a model fitted on the *same* rows, which
  biases the estimates — CatBoost's authors call this *prediction shift*, and it's a subtle
  form of overfitting that worsens as data gets smaller and noisier. Ordered boosting
  estimates gradients from models trained on data *excluding* the row being evaluated. On
  ~3,400 effective samples of very low signal-to-noise data, that's exactly the right
  correction.
- **Oblivious (symmetric) trees** by default — every node at a given depth splits on the
  same feature. A strong structural regularizer, which is what you want when noise
  dominates.
- **Strong defaults and a built-in overfitting detector.** This matters more than it
  sounds: *every hyperparameter you tune is another chance to overfit the validation
  folds*, so a library needing less tuning is a safety feature, not a convenience.
- **Native NaN handling.** Necessary, not optional — every indicator has a warm-up period,
  so a 200-day feature is NaN for the first 200 bars of *every* ticker.
- Native categorical support for `sector`.

Its famous feature — target-statistic encoding of high-cardinality categoricals — is
nearly irrelevant here, since the features are almost all numeric. The reasons to use it
are ordered boosting and the regularization defaults.

```python
CatBoostClassifier(
    loss_function="Logloss",
    boosting_type="Ordered",   # ← the reason we're here. Don't switch to "Plain".
    depth=4,                   # SHALLOW. not 8, not 10.
    learning_rate=0.03,        # slow
    iterations=3000,           # high, but...
    od_type="Iter", od_wait=100,   # ...stopped by the overfitting detector
    l2_leaf_reg=10.0,          # strong
    rsm=0.7,                   # feature subsampling, decorrelates trees
    auto_class_weights="Balanced",
    random_seed=42,
)
```

> ⚠️ `boosting_type="Ordered"` is slower. At this size that's fine. If training becomes
> painful, reduce `iterations` or `depth` before reaching for `"Plain"` — `"Plain"` throws
> away the main reason for choosing this library.
>
> The instinct to set `depth=8, iterations=5000` is how you get a beautiful training curve
> and a worthless model. Keep the hyperparameter search small and coarse — a few values
> for `depth` and `l2_leaf_reg`, and let the overfitting detector handle the rest.

### Reading the comparison

| Outcome | What it means |
|---|---|
| CatBoost > RF on the holdout | boosting is extracting real signal. Good. |
| **RF ≥ CatBoost** | **CatBoost is fitting noise.** Raise `l2_leaf_reg`, drop `depth` to 3, or ship RF. |
| Both ≈ the depth-4 tree | there are only a few real effects here. Ship the tree — it's readable and you'll trust it. |
| All ≈ base rate | the features contain no signal. No model fixes that. |

That last row is a real possible outcome, and finding it out in week 6 rather than month 6
is worth the 20 minutes it costs to run all three.

### Sample size and capacity

| | |
|---|---|
| 9 years × ~252 sessions | ~2,268 bars per ticker |
| ÷ horizon 10 (labels overlap — §8.2) | ~227 **effective independent** samples per ticker |
| × 15 tickers | **~3,400 effective samples** |

The row count will *look* like ~34,000. The information content is ~3,400, and that's what
should govern model complexity — hence `depth=4`, strong `l2_leaf_reg`, and the 18-feature
`minimal` set (§6.3) rather than the defaults and the full list.

**`effective samples ≈ tickers × years × 25`.** Years and tickers trade off directly, so
adding tickers is the cheapest way to buy statistical power — see §5 for what each universe
size gets you. Note also that tickers which all move together deliver less independent
information than the count suggests, so a spread across sectors is worth more than the same
number of names from one industry.

### One global model, no ticker identity

Train one model across all tickers rather than one per ticker — per-ticker gets ~2,268 rows
and overfits. **Deliberately exclude ticker ID as a feature:** it invites the model to
memorize "NVDA went up 2017–2026", which is survivorship-flavoured and won't repeat. Sector
as a categorical is fine.

### Class imbalance

With TP 2 ATR / SL 1 ATR the positive rate will be well under 50%. **Don't fight it with
resampling** — SMOTE on time-series features synthesizes rows that violate the temporal
structure. Use `auto_class_weights="Balanced"` and tune the decision threshold, which §11
does anyway.

### Train on the distribution you'll trade

> In March 2020 `ATR/close` reached ~8–10% on volatile names. At 10× leverage a 1-ATR stop
> is then an **80–100% loss of that position's margin** — a margin wipe. So §11's `max_vol`
> gate correctly refuses to trade in those conditions.
>
> But if those bars stay in the *training* set, the model learns from a regime it will
> **never be asked to act in** — and five weeks of violent moves can carry outsized
> influence on the fitted thresholds.
>
> **Rule: build the training set through the same hard gates §11 applies at inference
> time.** `max_vol`, `min_liquidity` and the gap filter run at sample-construction time,
> recorded in `meta_gate_pass`. Report the count excluded by each gate so the filtering is
> visible.
>
> This doesn't mean deleting those bars — they're needed to compute indicators and regime
> features for *later* dates, and the market-regime features still let the model learn
> "conditions like these are dangerous". Only the tradeable *samples* are filtered.

### 9.2 Validation

> ⚠️ **`train_test_split` and `KFold` are both guaranteed to lie to you here.** Two
> independent leaks:
>
> 1. **Shuffled folds** put future rows in train and past rows in test. You're predicting
>    2019 using 2024 data.
> 2. **Even with contiguous folds**, a label near the end of fold *k* resolves up to 10
>    bars into fold *k+1* (that's what `y_long_t1` records). So the training label literally
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
- **Embargo** — additionally drop training samples within ~10 bars *before* the test start,
  because serial correlation leaks across the seam even without interval overlap.
- **A final holdout** — the most recent **12 months** are never touched: not for
  hyperparameters, not for threshold tuning, not for feature selection. `tdm train` refuses
  to read it without an explicit `--unlock-holdout` flag, which exists to make the peek a
  deliberate act you'll remember.

> ⚠️ **The holdout governs what your final verdict is worth, and at 15 tickers it is thin.**
> 12 months is ~25 effective samples per ticker, so 15 tickers carries ~375 — and once
> §11's threshold filters most bars out, the surviving *trade* count lands below the
> ~100-trade minimum in §9.3.
>
> So the holdout will tell you the **sign** of the edge, not its size. "Promising" is the
> best verdict available at this universe size, and that's fine — it just means
> `decisions.jsonl` (§11) is where the real evidence accumulates afterwards. If you later
> want a verdict you can size up on, add tickers (§5), not years.

### 9.3 Metrics

Accuracy is near-meaningless on an imbalanced problem with asymmetric payoffs. A model that
always says HOLD can score 70% accuracy and be worth nothing. Report per fold **and per
direction**:

**Classification:** precision at the operating threshold (the one that matters: *of the
trades you actually take, how many win*), recall/coverage, PR-AUC (better than ROC-AUC
under imbalance), and **calibration** — a reliability curve, because §11 prints the
probability as a confidence number and "0.71" needs to actually mean 71%. Calibrate with
`CalibratedClassifierCV` on a purged inner split.

**Economic — the real scoreboard:**

| Metric | Why |
|---|---|
| **Expectancy per trade**, net of costs | the single number that decides whether to use this |
| Profit factor | gross wins ÷ gross losses |
| Hit rate + average R multiple | the two components of expectancy |
| Max drawdown, Calmar | whether you could actually tolerate it |
| Sharpe / Sortino | risk-adjusted, comparable to doing nothing |
| Trade count | **fewer than ~100 holdout trades ⇒ your estimate is noise** |
| **vs. buy-and-hold `^GSPC` over the same window** | the benchmark that counts. Beating a coin flip isn't the bar; beating "do nothing and own the index" is |

**Report the distribution across folds, never just the mean.** A strategy that was
excellent 2019–2021 and flat since has a fine average and is untradeable.

### 9.4 Leakage checklist — `tests/test_leakage.py`

Every item is a bug I expect this project to hit at least once. **Treat this as the most
important file in the repo.**

| # | Invariant | Test |
|---|---|---|
| 1 | **No feature at date `d` uses any bar after `d`** | Property test: recompute every feature on the series truncated at `d`; assert the value at `d` is identical to the full-series computation. **This one test catches most of the rest.** |
| 2 | ZigZag pivots respect `confirmed_at` (§6.3) | The highest-risk item |
| 2b | **Entry price is knowable at decision time** | Entry for bar `t` may use columns at index `≤ t` plus `open/high/low[t+1]` for fill determination. Any use of `high[t]` or `low[t]` — e.g. a mid-range entry — fails (§8.2) |
| 3 | Scalers / z-scores / percentiles use trailing windows only | Ban `.fit(X_all)`; assert window params |
| 4 | No target column reaches `X` | `set(X.columns) == set(registry.feature_names())`, and nothing matching `^(y_|fwd_|meta_)` (§8.1) |
| 5 | Purging actually applied | Assert no training `[t, t1]` overlaps any test fold |
| 6 | **Survivorship bias** | ⚠️ **The one item good code cannot fix.** A universe of today's winners was selected *because* they went up. Mitigations are in §5: spread across sectors, include some losers. Mostly: know the number is optimistic. |
| 7 | Holdout untouched | `--unlock-holdout` required (§9.2) |

---
## 10. Backtest

A small event-driven replay harness in `backtest/engine.py`.

```python
for d in test_sessions:
    x = features.loc[d]                  # from Parquet — never recomputed live
    p_long, p_short = m_long.predict_proba(x), m_short.predict_proba(x)

    orders = decision_rule(p_long, p_short, gates(d))   # SAME function as §11

    units  = (risk_frac * equity) / (cfg.k_sl * atr)    # leverage absent — §10.2
    margin = units * price / cfg.leverage

    fills = fill_at(open_price[d + 1], slippage=s * atr)
    manage_open_positions(d)      # TP / SL / timeout — SAME pessimistic
                                  # same-bar rule as §8.2. Non-negotiable.
    accrue_financing(d)           # daily, on notional, both directions
    mark_to_market(d)
```

> **The consistency requirement.** `decision_rule` and the exit logic are **imported** by
> the backtest and by production (§11), never reimplemented. If the simulator is more
> optimistic than the labeler, or production more permissive than the backtest, you trained
> and validated against a world you don't deploy into — and the discrepancy surfaces as
> unexplained live underperformance, which is almost impossible to debug after the fact.
> This is also why no third-party backtesting framework is used: it would be a second,
> independently-drifting implementation of logic §8.2 already defines.

### 10.1 The venue: IQ Option CFDs at 10× leverage

A **CFD** is a derivative — you don't own shares, you hold a contract that pays the price
difference. Two consequences.

**Shorting is symmetric with going long.** With real shares, going short means borrowing
them, which brings availability limits, borrow fees, recall risk and regulatory
restrictions on short sales after a sharp drop. None of that applies to CFDs: there's no
share borrow, so long and short are the same code path with a sign flip, differing only in
the financing rate.

**Financing is charged daily on the notional, in both directions.** Unlike owning shares, a
leveraged CFD accrues a daily charge on the **full position value**, not on your margin —
and on **long** positions too, not just shorts. At 10× that's 10× larger relative to your
own capital than the headline rate suggests. Over a 10-day hold it's a real cost.

### Cost model

Defaults are industry-typical retail CFD figures. **Replace them with your real numbers**
from the platform — §10.3 shows why they're load-bearing.

```toml
[costs]
spread_pct        = 0.10   # round-trip spread, % of notional. The dominant cost.
                           # Typical retail: 0.05% on the most liquid names,
                           # 0.1–0.2% mid-cap, wider on thin ones.
slippage_atr_frac = 0.05   # execution noise per side, volatility-scaled
financing_pct_day = 0.02   # % per DAY on notional, both directions.
                           # Typical: benchmark rate + ~3%/yr markup, /365.
commission        = 0.0    # normally zero on CFDs — the cost is the spread
```

### 10.2 Position sizing, in plain dollars

This is the decision that affects your results more than the model does (§12), and the
phrase "risk 1% per trade" is easy to read the wrong way.

> **It means: if the stop-loss hits, you lose $100 of your $10,000. Nothing else.**
>
> It does **not** mean "put $100 into the trade."

The position will be worth about $4,800; the amount you *lose when wrong* is $100. Sizing
works backwards from the loss you're willing to take.

Using the NVDA example from §1 — entry $183.40, ATR $3.81, stop 1 ATR away:

| Step | Calculation | Result |
|---|---|---|
| 1. Risk budget | 1% × $10,000 | **$100** |
| 2. Loss per unit if stopped | `k_sl × ATR` = 1.0 × $3.81 | **$3.81** |
| 3. Units to buy | $100 ÷ $3.81 | **26 units** |
| 4. Position value (notional) | 26 × $183.40 | **$4,768** |
| 5. Margin tied up at 10× | $4,768 ÷ 10 | **$477** |
| 6. Loss if the stop hits | 26 × $3.81 | **−$100** ✓ |
| 7. Gain if the TP hits | 26 × 2 × $3.81 | **+$200** |

Step 6 is the check — it returns to $100 by construction. **You decide the loss first, and
the position size falls out of it.**

### Why this is the safe way to use leverage

Leverage appears in exactly one row — step 5 — and changes only how much capital the
position *occupies*, not how much you can lose:

| | No leverage | 10× leverage |
|---|---|---|
| Units | 26 | 26 |
| Loss if stopped | $100 | $100 |
| Capital tied up | $4,768 (48% of equity) | **$477 (4.8%)** |
| Positions you can hold | 2 | **~10** |

**Same risk, same trade, ten times the capital efficiency.** What leverage buys is the
ability to hold ten positions instead of two — which *reduces* the variance of your
results, because ten partly-independent bets are smoother than two.

The intuitive-but-wrong reading of "10× leverage" is *"I can take positions 10× bigger"*,
which multiplies your risk by ten and is how leveraged accounts die. Leverage never enters
the sizing formula.

### Account settings

| Setting | Value | Reasoning |
|---|---|---|
| `equity` | $10,000 | starting figure |
| `risk_frac` | 1% = $100/trade | standard, and survivable |
| `max_concurrent` | 10 | leverage makes it affordable; diversification makes it desirable |
| `max_margin_frac` | 50% | 10 positions × ~5% margin ≈ 50%, so this binds at about the same point — a second, independent brake |

> ⚠️ **The number to be comfortable with is 10%, not 1%.**
>
> Ten concurrent positions at 1% each means **$1,000 at risk simultaneously.** If all ten
> hit their stops on the same day you'd lose 10% of the account.
>
> That's not hypothetical. **In a market-wide selloff, correlations go to 1** — the
> diversification you were counting on evaporates exactly when you need it, and a basket of
> ten longs behaves like one big long. March 2020 would have done this. Treat "−10% in a
> day" as a real possible outcome.
>
> If that's too much, the lever is `max_concurrent`, not `risk_frac`: five positions caps
> simultaneous risk at 5%. Halving `risk_frac` would do the same, but it makes each trade's
> edge smaller relative to the fixed per-trade costs (§10.3), so cutting position count is
> the better of the two.

### 10.3 What costs do to the target

Costs are not a rounding error here, and it's worth seeing the number before building
anything. Same example — $10,000 equity, 1% risk, $100 stock, ATR $2, 50 units, $5,000
notional, TP 2 ATR / SL 1 ATR, 10-day hold:

| | |
|---|---|
| Gross win (TP hit) | 2 × $2 × 50 = **+$200** |
| Gross loss (SL hit) | 1 × $2 × 50 = **−$100** |
| Spread, round trip | 0.10% × $5,000 = −$5 |
| Slippage, both sides | 0.1 × ATR × 50 = −$10 |
| Financing, 10 days | 0.02%/day × 10 × $5,000 = −$10 |
| **Total costs** | **−$25** |

Break-even hit rate `p` solves `p × (200 − 25) = (1 − p) × (100 + 25)`:

| | Break-even hit rate |
|---|---|
| Ignoring costs | **33.3%** |
| **With costs** | **41.7%** |

**Costs move the bar by 8.4 percentage points.** Since §12 puts a realistically good hit
rate at ~45%, costs consume most of the available margin. Three consequences:

1. **§12's expectations are calibrated against 41.7%, not 33%.** A 40% hit rate is a
   *losing* strategy here, not a near-miss.
2. **Spread and slippage are fixed per trade; financing scales with holding time.** So
   *wider* barriers amortize the fixed part better — a TP 3 / SL 1.5 profile pays the same
   spread to chase a 50% bigger move. A real argument for testing wider profiles, which
   `tdm labels sweep` (§8.3) reports on.
3. **Fewer, better trades beat more trades.** Every extra trade pays the fixed cost again —
   an independent argument for a high decision threshold in §11.

### 10.4 Gap risk, amplified 10×

A stop-loss only caps your loss if the price passes *through* it. An overnight or weekend
gap fills you well past it. At 10× leverage this is the one tail that matters:

> **A 10% adverse overnight gap, at 10× leverage, wipes out 100% of that position's
> margin** — regardless of where your stop was.

A 10% single-day move in a large US stock is uncommon but routine on earnings days. That's
the argument for the earnings-proxy gate in §6.3.

Backtest handling:
- **Report max single-trade loss per direction**, separately from aggregate drawdown, so
  one catastrophic gap can't hide inside an average.
- **Report the margin-wipe count**: how many simulated positions lost ≥100% of posted
  margin. If that's above zero, the sizing or the gates need changing before this goes near
  real money.
- Model weekend gaps explicitly — `open[Monday]` vs `close[Friday]`.

### 10.5 Required sensitivity runs

A single backtest number is not evidence. Report the grid: 2× slippage, 2× commission,
entry delayed one more bar, TP/SL multiples ±25%, horizon ±5 days. **If the edge vanishes
when slippage doubles, there was no edge — there was a cost assumption.**

Also run the **shuffled-signal control**: randomly permute the predictions and re-run. If
random does comparably well, the model contributes nothing and the equity curve is just
measuring your position sizing.

**Report all §9.3 metrics per direction, never pooled.** Pooling hides the most likely
outcome — longs carry a modest edge, shorts carry none — behind a mediocre blended number.

> One mismatch to keep in mind: prices come from Yahoo (the real exchange); execution
> happens at IQ Option, whose quotes are its own and carry a wider spread. `spread_pct` is
> what bridges the two. If live fills consistently differ from the backtest, that parameter
> is the first place to look — not the model.

---

## 11. The daily decision

```python
def decide(symbol: str, as_of: date) -> Decision:
    x = features.load(symbol, as_of)      # from Parquet, not recomputed

    # ── Hard gates. Preconditions the model cannot see. Checked FIRST.
    if data_staleness(symbol) > 1:
        return ABSTAIN("stale data")          # ← fail loud, see below
    if adv_20d(symbol) < cfg.min_liquidity:
        return HOLD("illiquid")
    if atr_pct(symbol) > cfg.max_vol:
        return HOLD("volatility too extreme to size safely")
    if open_positions() >= cfg.max_concurrent:
        return HOLD("position limit")
    if margin_used_frac() >= cfg.max_margin_frac:
        return HOLD("margin ceiling reached")
    if x["large_gap_due"]:
        return HOLD("gap-risk window")        # earnings proxy, §6.3

    # ── Model
    p_long  = m_long.predict_proba(x)[1]
    p_short = m_short.predict_proba(x)[1]

    if p_long >= cfg.theta_long and p_long > p_short:
        side = Side.LONG
    elif p_short >= cfg.theta_short and p_short > p_long:
        side = Side.SHORT
    else:
        return HOLD(f"no edge: p_long={p_long:.2f} p_short={p_short:.2f}")

    # ── Levels: same arithmetic as the simulator, so what you see is
    #    exactly what the model was trained on
    atr   = x["atr_14"]
    entry = last_close(symbol)                 # indicative; real fill is next open
    sign  = +1 if side is Side.LONG else -1
    units = (cfg.risk_frac * equity) / (cfg.k_sl * atr)
    return Decision(
        side=side, confidence=max(p_long, p_short),
        entry_ref=entry,
        tp=entry + sign * cfg.k_tp * atr,
        sl=entry - sign * cfg.k_sl * atr,
        units=int(units),
        margin=units * entry / cfg.leverage,
        rationale=top_contributing_features(x, n=5),
    )
```

`theta_long` / `theta_short` are tuned on **validation folds only**, to maximize expectancy
(not accuracy), then **frozen and stored with the model artifact.** A threshold that drifts
between runs makes your track record uninterpretable.

> **The `ABSTAIN` on stale data is deliberate.** If the fetch failed and the newest bar is
> three days old, the system must **refuse to answer** rather than quietly deciding on stale
> prices. Silent degradation is the worst failure mode for a tool you check once a day and
> trust. Fail loud.

### Output

Everything is denominated in dollars and units — you shouldn't have to compute anything to
place the order:

```
NVDA   LONG   conf 0.71
  entry  ~$183.40    (indicative; actual fill is tomorrow's open)
  TP      $196.10     +$200 if hit
  SL      $177.05     −$100 if hit
  size     26 units  = $4,768 notional, $477 margin
  hold    ≤ 10 trading days, then exit at market
  because  EMA21 slope +0.8σ · ADX 31 · RSI 58 · %B 0.74 · RS rank 94th

account: 3 of 10 slots used · $1,431 of $5,000 margin · $300 currently at risk
```

The account line shows the aggregate before you add to it — "$300 currently at risk" is the
sum across open trades, and it's what tells you whether to take the next signal.

> **TP/SL are displayed as dollar prices but computed as ATR multiples** (§8.2). A fixed
> dollar stop can't work across the universe — $3.81 is a normal day's move for NVDA and a
> catastrophic one for a $20 stock, so a fixed figure would be far too tight on some
> tickers and far too loose on others. The ATR scaling is what makes one rule work for all
> 50. You just never have to see it.

Written to stdout as a `rich` table, to `reports/decisions_YYYY-MM-DD.{md,csv}`, and as one
appended line in `decisions.jsonl`.

### Scoring yourself

```bash
tdm score-decisions --since 2026-01-01
```

Reads `decisions.jsonl`, resolves each past decision against bars that have since arrived,
and reports the realized hit rate / expectancy / drawdown **of the decisions the system
actually issued.**

**This is the only genuinely out-of-sample measurement you will ever have.** No purging
subtleties, no leakage, no survivorship bias — just a prediction log and what happened
next. It's also what will tell you if a backtest was fooling you. Build it early and let it
accumulate: after six months it's worth more than any amount of backtesting. Never delete
`decisions.jsonl`.

---

## 12. Realistic expectations

Worth calibrating before you see a number, so you can tell success from self-deception.

- At R:R 2.0, break-even is a 33% hit rate ignoring costs — but **~42% once spread,
  slippage and 10 days of financing are included** (§10.3). Judge expectancy, not accuracy.
- ⚠️ **That 42% is uncomfortably close to the ~45% that counts as a genuinely good result.**
  The realistic gap between "works" and "doesn't" is a few percentage points of hit rate.
  Two implications: the cost parameters need to be *your real ones*, and wider TP/SL
  profiles deserve a serious look since they amortize fixed costs better.
- Out-of-sample precision at threshold around **0.50–0.60** is a solid outcome for daily-bar
  technical features — and that comfortably clears 42%, which is where the opportunity is.
- ⚠️ **If your first run shows >65% out-of-sample accuracy, or a smooth upward equity curve,
  you have a bug — not an edge.** Go back to §9.4. This is the *expected* failure mode, not
  a remote one.
- Expect the honest outcome to be **"a modest edge in some market conditions, nothing in
  others."** Still useful: a model that reliably says HOLD during choppy periods and sizes
  up in trending ones has value even with an unremarkable hit rate.
- **Position sizing and risk management will affect your results more than model accuracy**
  — doubly so at 10× leverage. Risk-based sizing (§10.2), a cap on concurrent positions, and
  avoiding earnings gaps are worth more than several points of AUC.
- **Be prepared for the shorts not to work.** US stock indices drift upward over time, so
  short setups have a worse base rate. CFDs remove the mechanical obstacles but not the
  drift. If `M_short` shows no out-of-sample edge, **ship long-only.** That's a finding, not
  a defeat.
- ⚠️ **Report confidence intervals on the hit rate, not just the point estimate — and at
  15 tickers expect them to be wide.** Distinguishing a 40% hit rate from 45% needs on the
  order of a thousand independent trades; the 12-month holdout will have well under a
  hundred (§9.2). **If the interval straddles 42%, the honest readout is "inconclusive",
  not "profitable".** That is the most likely outcome of the first full run, and it is not
  a failure — it's the point at which `decisions.jsonl` starts earning its keep.

---

## 13. Build order

Each phase ends somewhere useful, so you can stop or re-scope without a half-built system.

| Phase | Deliverable | Done when |
|---|---|---|
| **0 · Scaffold** | venv 3.14, `CLAUDE.md`, Makefile, test runner | `make venv && make test` green; the venv guard fires outside the venv |
| **1 · Fetch** | `yf.download` wrapper, `universe validate`, OHLCV sanity checks, Parquet snapshot | 9 years × your tickers lands clean. ⚠️ **Smoke-test yfinance against pandas 3.1 here** (§3) — the one unverified dependency |
| **2 · Indicators** | registry + the 18 `minimal` features, Parquet output, CSV export | `test_indicators.py` green; `test_invariance.py` green (§5) |
| **3 · Charts** | Plotly multi-panel HTML per ticker (§7) | **you open `NVDA.html` and the indicators look right.** This gates phases 1–2 — do not proceed past a chart that looks wrong |
| **4 · Structure features** | ZigZag + Dow/Elliott with `confirmed_at` | `test_leakage.py` pivot tests green, **and the pivots look right on the chart** |
| **5 · Dataset** | triple barrier, `t1`, weights, `fwd_*` columns, `labels sweep` | base rate / timeout fraction / ambiguity rate sane; **TP/SL brackets look sane on the chart** (`--with-labels`) |
| **6 · Baseline model** | depth-4 tree + purged walk-forward CV | **the printed rule set is readable and not absurd** |
| **7 · Model comparison** | CatBoost + RandomForest control + calibration + permutation importance + threshold tuning | all three compared on the same folds; you know which row of §9.1's table you're in |
| **8 · Backtest** | engine, costs, sensitivity grid, shuffled control | survives 2× slippage; beats the shuffled control |
| **9 · Decide** | `tdm decide`, gates, reports | one command, end to end |
| **10 · Self-scoring** | `tdm score-decisions` | reproduces phase-8 numbers on the overlapping window |
| **11 · Holdout** | one single evaluation on the untouched 12 months | **go / no-go** |

**Phase 3 is deliberately early.** Charting before labeling or training catches feature
bugs while they're still cheap — an off-by-one indicator or a lookahead pivot is obvious on
a chart and nearly undetectable in a metric. A model trained on quietly broken features
still produces a plausible-looking number, which is the worst outcome available.

**If you only reach phase 3**, you still have a reproducible feature pipeline and readable
charts of every ticker — a useful analysis tool on its own, independent of any model.

---

## 14. Configuration reference

```toml
# config/config.toml

[universe]
file        = "config/universe.txt"   # your tickers — any number (§5)
years       = 9

[reference]                           # fetched for features only, never traded (§6.3)
market      = "^GSPC"                 # S&P 500 INDEX, not an ETF
volatility  = "^VIX"                  # no volume; price only
peer_proxy  = "universe_median"       # sector comparison, from your own tickers

[labels]                              # the "label profile" — hashed into run_id
k_tp        = 2.0                     # take-profit, in ATR
k_sl        = 1.0                     # stop-loss, in ATR
horizon     = 10                      # max holding period, trading days. NOT unbounded —
                                      # see §8.2 for the four reasons why
atr_period  = 14
entry_mode  = "next_open"             # next_open | close | limit_pullback (§8.2)
entry_offset_atr = 0.25               # only used by limit_pullback

[costs]                               # ⚠️ replace with your real IQ Option numbers
spread_pct        = 0.10              # round-trip, % of notional
slippage_atr_frac = 0.05              # per side
financing_pct_day = 0.02              # per day, on notional, both directions
commission        = 0.0

[account]
equity          = 10_000
leverage        = 10                  # sets margin used, NOT position size (§10.2)
risk_frac       = 0.01                # 1% = $100 per trade
max_concurrent  = 10
max_margin_frac = 0.50

[gates]                               # applied at decision time AND sample construction
min_liquidity   = 1_000_000           # min 20-day ADV, shares
max_vol         = 0.08                # max ATR/close — above this, don't trade
earnings_proxy  = true                # use the OHLCV-derived gap-risk window (§6.3)
earnings_calendar = false             # real calendar, if you later add a data source

[model]
primary         = "catboost"
features        = "minimal"           # minimal | core | extended — match to
                                      # universe size (§6.3)
holdout_months  = 12
embargo_bars    = 10
```

### Settings you may want to revisit

| Setting | Current | When to change it |
|---|---|---|
| `max_concurrent` | 10 | Lower to 5 if "−10% in a day" (§10.2) is more than you want. Note it can't bind above your ticker count |
| `costs.*` | estimates | As soon as you can read your real spread and overnight fee off the platform — they move break-even by ~8 points (§10.3) |
| `k_tp` / `k_sl` | 2.0 / 1.0 | After `labels sweep` — wider profiles amortize fixed costs better |
| `features` | `minimal` | Move to `core` once `minimal` shows signal, or if you grow the universe past ~20 tickers (§6.3) |
| `universe` | ~15 tickers | Adding tickers is the cheapest way to buy statistical power and the only way to get a holdout verdict you can size up on (§5) |
| `earnings_calendar` | `false` | Before sizing up — a real calendar beats the proxy, and earnings is the only knowable tail risk (§10.4) |

---

## 15. Iteration speed

If a run feels slow, it is almost certainly **not** the download. 15 tickers × 9 years of
daily bars is ~34,000 rows — a small HTTP payload that lands in well under a minute. Where
the time actually goes:

| Stage | Cost | Why |
|---|---|---|
| Fetch | ~20–60 s | small payload, batched and threaded |
| Features | seconds | TA-Lib over 34k rows |
| Dataset / barriers | seconds | one pass per row |
| **Train** | **minutes** | ← **the real cost** |

Training dominates because one experiment is **12 model fits**: 6 walk-forward folds ×
2 directions (long and short), and `boosting_type="Ordered"` is 2–4× slower than plain
boosting.

### The cheap levers, in order

**1. The dataset snapshot already removes fetch from the loop.** `tdm dataset build` writes
Parquet keyed by config hash (§5); every subsequent `tdm train` reads that file. You pay
the download once, then iterate offline. This is the single biggest win and it's automatic.

**2. Use a fast profile while exploring, the real one to conclude:**

```bash
tdm train --fast      # 3 folds, boosting_type="Plain", iterations=500
tdm train             # 6 folds, Ordered, iterations=3000
```

`--fast` is for "does this feature help at all?" — roughly 10× quicker and directionally
honest. Never report a `--fast` number as a result, and never use one to make the
phase-11 go/no-go call.

**3. Train one direction while iterating.** `--side long` halves the work, and per §12 the
long side is where the edge is more likely to be anyway.

> ⚠️ **Don't cut tickers to save time.** It's the most expensive lever available: fetch
> time scales with ticker count but is already negligible, while *statistical power* scales
> with it and is the scarce resource (§5). Dropping from 50 to 15 saves perhaps 40 seconds
> of download and costs you the ability to draw a conclusion from the holdout. Cut folds,
> cut iterations, cut one direction — not tickers.
