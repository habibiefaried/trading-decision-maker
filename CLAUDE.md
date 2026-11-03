# CLAUDE.md — agent rules for this repo

## 1. ALWAYS use the project venv. Never the system Python.

This project targets **Python 3.15** and runs entirely inside `./venv`.

```bash
./venv/bin/python   # the ONLY interpreter you may use
./venv/bin/pip      # the ONLY pip you may use
```

**Do not** run `python`, `python3`, `pip`, or `pip3` bare — on this machine those
resolve to pyenv's 3.13.1 shim or Homebrew's 3.14, both of which are wrong, and
installing into them silently pollutes the user's global environment.

**Do not** use `source venv/bin/activate` in a `Bash` tool call. Shell state does not
persist between tool calls in Claude Code, so the activation is gone by the next
command and you end up on the system Python without noticing. Always call the
interpreter by its explicit path, or go through `make`.

Prefer `make` targets, which already point at `./venv/bin/python`:

```bash
make venv        # create venv with Python 3.15 + install requirements
make fetch       # incremental OHLCV download into data/market.db
make features    # compute indicators
make label       # build triple-barrier labels
make train       # train models
make backtest
make decide      # today's LONG / SHORT / HOLD call
make daily       # fetch -> features -> decide
make test
make lint
```

If `./venv` does not exist, create it — don't fall back to system Python:

```bash
brew install python@3.15                      # provides 3.15.2
/opt/homebrew/bin/python3.15 -m venv venv
./venv/bin/python -m pip install -U pip
./venv/bin/pip install -r requirements.txt
```

`src/tdm/__init__.py` contains a guard that raises if `sys.prefix == sys.base_prefix`.
If you see `RuntimeError: not running inside ./venv`, you used the wrong interpreter —
re-run with `./venv/bin/python`.

## 2. Read DESIGN.md before changing anything

`DESIGN.md` is the source of truth for architecture and, more importantly, for the
*reasoning*, and it assumes no finance background — **§2 is a glossary** defining every
domain term in software terms. Organized by section (§5 schema, §6 price adjustment,
§7 incremental fetch, §8 indicators, §9 labeling, §10 validation, §11 backtest + venue
+ costs, §13 decision rule, §15 resolved decisions, §16 still open). When you change
behaviour, update the relevant section in the same commit.

`README.md` is for script/command usage only — no design discussion there.

## 3. Hard rules that exist to stop specific, known bugs

Each of these has burned this class of project before. DESIGN.md explains why.

1. **There is no database.** The pipeline is stateless: re-download on every run,
   `auto_adjust=True`, recompute everything. Do not add SQLite, a schema, migrations,
   coverage tracking or gap bookkeeping — all of that was deliberately deleted. (§5)
2. **Snapshot the feature matrix, not the bars.** `data/features/<run_id>.parquet` where
   `run_id = hash(tickers, date range, feature config, code version)`, and append a line
   to `runs.jsonl`. This exists so two model runs are comparable; without it you cannot
   tell a worse model from a changed dataset. Training reads a snapshot by default;
   `--refresh` forces a new fetch. (§5)
3. **No lookahead, ever.** A feature on date *d* may only use bars up to and including
   *d*. In particular: ZigZag pivots must respect `confirmed_at`; z-scores and percentile
   ranks use trailing windows only (never `fit()` on the full series). (§8.5, §10.5)
4. **Never use plain `KFold` / `train_test_split`.** Use
   `model/cv.py::PurgedWalkForwardSplit`. Labels span up to H future bars, so ordinary
   CV leaks test prices into training labels and every metric comes out inflated. (§10.3)
5. **Don't touch the holdout window.** The final 12 months require an explicit
   `--unlock-holdout` flag. No hyperparameter search, feature selection, or threshold
   tuning against it. (§10.3)
6. **Resolve same-bar TP+SL touches pessimistically** (assume the stop hit first), in
   both the labeler and the backtest. The two must never disagree. (§9, §11)
7. **Report expectancy, not accuracy.** At 2:1 R:R break-even is a ~33% hit rate before
   costs and **~42% after** CFD spread, slippage and financing. Accuracy is close to
   meaningless here. (§10.4, §11)
8. **Leverage never multiplies position size.** Size from `risk_frac` only; leverage
   affects the margin a position ties up, nothing else. Getting this backwards is how
   leveraged accounts die. (§11.1)
9. **No ETFs in decisions, but always ingest them.** SPY and sector ETFs are benchmark
   inputs for relative-strength and regime features. They carry `tradeable = 0` and are
   filtered at the decision step, never at ingest. (§8.2)
10. **Don't add Reg SHO / share-borrow / shortability logic.** We trade CFDs, which have
    no share borrow and no uptick rule. Long and short are one code path with a sign
    flip. An earlier draft modeled all of this; it was deleted on purpose. (§11.1)
11. **No weekly resampling.** Dropped by decision. For a longer-timeframe trend signal,
    use a wider ZigZag threshold on daily bars (`dow_trend_major`), which avoids the
    partial-bar leakage hazard entirely. (§8.2, §8.4)
12. **Every feature must be scale-free** — a ratio, a bounded oscillator, or a trailing
    z-score/percentile. Never raw dollars (price, EMA, MACD, band levels) and never an
    unbounded cumulative sum (OBV). Two independent payoffs: tree thresholds transfer
    across tickers, and features become invariant to splits — which is what makes
    re-downloading safe. `tests/test_invariance.py` enforces it by injecting a synthetic
    split and asserting past feature values are unchanged. (§8.5, §5)
14. **Models: HistGradientBoosting is primary, RandomForest is a required control.** If
    RF matches or beats HGB on the holdout, HGB is overfit — report both, always. Keep
    the depth-4 tree permanently as the readable baseline. Shallow and heavily
    regularized: `max_depth=3`, `learning_rate=0.03`, `min_samples_leaf≈100`, early
    stopping on. CatBoost has no cp315 wheel; do not add it. (§10.1)
13. **Filter training samples through the same gates `decide` uses.** `max_vol`,
    `min_liquidity` and the gap filter run at sample-construction time too — otherwise
    you train on regimes you will never trade in (e.g. March 2020 at 10x leverage).
    (§10.1)

## 3a. Chart before you train

`tdm chart` (§7) is build phase 3 — **before** labels and training. Two rules:

- **`report/` reads the feature matrix; it never computes an indicator.** A chart that
  draws its own indicators can look perfect while the model trains on garbage, and when
  a value looks wrong you cannot tell whether the bug is in the pipeline or the plot.
- Use **Plotly** (`go.Candlestick` + `make_subplots` + `write_html`). Self-contained
  interactive HTML, no server, no build step.

Visual checks catch what unit tests miss: off-by-one indicators, hindsight ZigZag pivots,
warm-up periods filled with 0 instead of NaN, features that are flat for 200 bars.

## 4. pandas 3.x gotchas

`pandas` is on 3.x in this project — a breaking release. Code copied from older
projects (including `~/code/stock-market-entry-signal-tpsl-specific`) will likely break:

- Copy-on-Write is mandatory: chained assignment (`df['x'][0] = v`) no longer works.
  Use `df.loc[...]`, or build columns functionally.
- Default datetime resolution is **microsecond**, not nanosecond.
- Offset aliases `M` / `Q` / `Y` were removed — use `ME` / `QE` / `YE`. (`W-FRI` is fine.)
- `inplace=True` now returns `self` instead of `None`.
- String columns infer to the new `str` dtype, not `object`.

Packages that **cannot** be installed on Python 3.15 — don't try:

| Package | Why |
|---|---|
| `pandas-ta` | pins `numba==0.61.2`, no cp315 wheel. Use **TA-Lib**. |
| `vectorbt` | declares `requires-python <3.15`. Use the in-repo `backtest/engine.py`. |
| `catboost` | wheels stop at cp314. Use `HistGradientBoostingClassifier`. |

## 5. Don't commit data

`data/`, `models/`, `reports/`, `*.parquet`, `runs.jsonl` are gitignored and must stay
that way. All of it is re-derivable from `make fetch` onward.

**One exception worth knowing:** `decisions.jsonl` (§13) is the log of calls the system
actually issued, and it is the only genuinely out-of-sample record that will ever exist.
It is still gitignored, but never delete or regenerate it.
