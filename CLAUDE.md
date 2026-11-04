# CLAUDE.md — agent rules for this repo

## 1. ALWAYS use the project venv. Never the system Python.

This project targets **Python 3.14** and runs entirely inside `./venv`.

```bash
./venv/bin/python   # the ONLY interpreter you may use
./venv/bin/pip      # the ONLY pip you may use
```

**Do not** run `python`, `python3`, `pip`, or `pip3` bare — on this machine those resolve
to pyenv's 3.13.1 shim, which is the wrong interpreter, and installing into it silently
pollutes the user's global environment.

**Do not** use `source venv/bin/activate` in a `Bash` tool call. Shell state does not
persist between tool calls, so the activation is gone by the next command and you end up on
the system Python without noticing. Always call the interpreter by its explicit path, or go
through `make`.

```bash
make venv        # create venv with Python 3.14 + install requirements
make fetch       # download 9 years of daily OHLCV
make features    # compute indicators
make charts      # render HTML charts
make dataset     # features + target columns
make train       # fit models
make backtest
make decide      # today's LONG / SHORT / HOLD calls
make daily       # fetch -> features -> dataset -> decide
make test
make lint
```

If `./venv` does not exist, create it — don't fall back to system Python:

```bash
brew install python@3.14
/opt/homebrew/bin/python3.14 -m venv venv
./venv/bin/python -m pip install -U pip
./venv/bin/pip install -r requirements.txt
```

`src/tdm/__init__.py` contains a guard that raises if `sys.prefix == sys.base_prefix`. If
you see `RuntimeError: not running inside ./venv`, you used the wrong interpreter.

## 2. Read DESIGN.md before changing anything

`DESIGN.md` is the source of truth for architecture and reasoning, and it assumes no
finance background — **§2 is a glossary** defining every domain term.

| Section | Content |
|---|---|
| §3 | environment, dependencies, pandas 3 gotchas |
| §5 | fetching, no-database rationale, the universe |
| §6 | features: the contract, the scale-free rule, the full list |
| §7 | charts |
| §8 | the dataset table, targets, the triple barrier |
| §9 | model choice, validation, metrics, **the leakage checklist** |
| §10 | backtest, venue, costs, position sizing |
| §11 | the daily decision |
| §14 | configuration reference |
| §15 | iteration speed — what to cut when a run feels slow |

When you change behaviour, update the relevant section in the same commit.

`README.md` is for script/command usage only — no design discussion there.

## 3. Chart before you train

`tdm chart` (§7) is build phase 3 — **before** the dataset and training. Two rules:

- **`report/` reads the feature matrix; it never computes an indicator.** A chart that
  draws its own indicators can look perfect while the model trains on garbage, and when a
  value looks wrong you cannot tell whether the bug is in the pipeline or the plot.
- Use **Plotly** (`go.Candlestick` + `make_subplots` + `write_html`). Self-contained
  interactive HTML, no server, no build step.

Visual checks catch what unit tests miss: off-by-one indicators, hindsight ZigZag pivots,
warm-up periods filled with 0 instead of NaN, features that are flat for 200 bars.

## 4. Hard rules that exist to stop specific, known bugs

1. **Nothing is stored. Everything runs in memory, in one process.** Re-download each run,
   `auto_adjust=True`, recompute everything. No database, no Parquet cache, no feature
   store, no saved model artifact. Do not add any. (§5)
2. **Model comparison must happen inside one invocation.** Two runs see different data (at
   minimum one extra bar), so numbers from separate runs aren't comparable.
   `tdm train --compare` fits all models on the same in-memory dataset. The only things
   written to disk are `reports/charts/*.html` (output) and `decisions.jsonl` (a logbook
   that cannot be recomputed — never delete it). (§5)
3. **Match the feature set to the universe size.** `minimal` (18) at ~15 tickers, `core`
   (40) at 20–40, `extended` beyond that. Keep ≥100 effective samples per feature; at 15
   tickers that is ~3,400 ÷ 18. Using all ~118 features on a small universe is ~29 samples
   per feature and will overfit silently. (§6.3, §5)
4. **Don't cut tickers to make a run faster.** Fetch is already negligible; training
   dominates. Cut CV folds, use `--fast` (Plain boosting, fewer iterations), or train one
   direction. Ticker count is statistical power and it is the scarce resource. (§15)
5. **Every feature must be scale-free** — a ratio, a bounded oscillator, or a trailing
   z-score/percentile. Never raw dollars (price, EMA, MACD, band levels) and never an
   unbounded cumulative sum (OBV). Two payoffs: tree thresholds transfer across tickers,
   and features become invariant to splits, which is what makes re-downloading safe.
   `tests/test_invariance.py` enforces it. (§6.2, §5)
6. **No lookahead, ever.** A feature on date *d* may only use bars up to and including *d*.
   In particular: ZigZag pivots must respect `confirmed_at`; z-scores and percentile ranks
   use trailing windows only, never `.fit()` on the full series. (§6.3, §9.4)
7. **Feature selection is an ALLOWLIST, never a denylist.** Targets and features live in
   the same table, so `X = df.drop(columns=["y_long"])` silently leaves `fwd_ret_10d` in
   `X` and hands the model the answer — ~99% accuracy, no error, a backtest that looks like
   a money printer. Always `X = df[registry.feature_names()]`. Non-feature columns carry a
   mandatory prefix: `y_`, `fwd_`, `meta_`. A bare column name means it is a feature. (§8.1)
8. **Never target an absolute price.** Predicting `close[t+1]` gives R²≈0.999 by learning
   `close[t+1]≈close[t]` — the identity function, zero tradeable information, and it looks
   like a triumph. Targets are always changes: returns, or barrier outcomes. (§8.1)
9. **Never use plain `KFold` / `train_test_split`.** Use
   `model/cv.py::PurgedWalkForwardSplit`. Labels span up to 10 future bars, so ordinary CV
   leaks test prices into training labels and every metric comes out inflated. (§9.2)
10. **Don't touch the holdout window.** The final 12 months require an explicit
    `--unlock-holdout` flag. No hyperparameter search, feature selection, or threshold
    tuning against it. (§9.2)
11. **Resolve same-bar TP+SL touches pessimistically** (assume the stop hit first), in both
    the simulator and the backtest. The two must never disagree. (§8.2, §10)
    **The target is 3-class: `TP` / `SL` / `TIMEOUT`.** TIMEOUT is a real outcome, not a
    degenerate case to fold into win/loss by sign — folding it destroys the information
    §11 needs to price the trade. `loss_function="MultiClass"`. (§8.2)
12. **Entry price must be knowable at decision time.** Never `(high[t]+low[t])/2` or
    anything else derived from bar `t`'s high/low — those aren't known until the day is
    over, and a mid-range fill is worth ~48% of the risk budget in phantom edge. Legal
    modes are `next_open` (default), `close`, and `limit_pullback` (a limit at
    `close[t] ± offset×ATR`, filled only if bar `t+1` trades through it). Horizon must
    stay bounded — unbounded breaks purging and collapses effective sample size to ~340.
    (§8.2)
13. **Filter training samples through the same gates `decide` uses.** `max_vol`,
    `min_liquidity` and the gap filter run at sample-construction time too — otherwise you
    train on regimes you will never trade in (e.g. March 2020 at 10× leverage). (§9.1)
14. **Decide on expected value, not a probability threshold.**
    `EV_R = 1.5·P(TP) − P(SL) + r_to·P(TIMEOUT) − 0.25`, and trade when it clears
    `min_ev_r` (default 0.10 R = +$10 per $100 risked). `min_ev_r` lives in `config.toml`,
    is never fitted, and must stay fixed across runs or the track record is
    uninterpretable. Rank candidates by EV when slots are scarce.
    **Calibration matters more than accuracy** — EV is computed from the three
    probabilities, so they must be numerically right, not merely well-ranked. Check
    reliability curves per class. At TP 1.5 / SL 1.0, break-even is a 40% hit rate before
    costs and **50% after**. (§11, §8.2, §9.3, §10.3)
15. **Leverage never multiplies position size.** Size from `risk_frac` only; leverage
    affects the margin a position ties up, nothing else. Getting this backwards is how
    leveraged accounts die. (§10.2)
16. **No ETFs anywhere — but `^GSPC` and `^VIX` are still required.** They are *indices*,
    not ETFs, fetched as **reference series**: inputs to the relative-strength and regime
    feature columns only. They produce no dataset rows, get no decisions, and appear
    nowhere in the output. Don't delete them — three of the 18 `minimal` features depend
    on a market series, including `rs_rank`, the strongest single feature in the set.
    Sector comparison uses the peer median of the user's own universe, not sector ETFs.
    (§5, §6.3)
17. **Long and short are one code path with a sign flip.** We trade CFDs — no share borrow,
    no uptick rule, no shortability check. Don't add that logic. (§10.1)
18. **Models: CatBoost is primary, RandomForest is a required control.** If RF matches or
    beats CatBoost on the holdout, CatBoost is overfit — report both, always. Keep the
    depth-4 tree permanently as the readable baseline. Shallow and heavily regularized:
    `depth=4`, `learning_rate=0.03`, `l2_leaf_reg≈10`, `od_type="Iter"`. Leave
    `boosting_type="Ordered"` on — it is the reason we chose CatBoost. (§9.1)

## 5. pandas 3.x gotchas

- Copy-on-Write is mandatory: chained assignment (`df['x'][0] = v`) no longer works. Use
  `df.loc[...]`, or build columns functionally.
- Default datetime resolution is **microsecond**, not nanosecond.
- Offset aliases `M` / `Q` / `Y` were removed — use `ME` / `QE` / `YE`.
- `inplace=True` now returns `self` instead of `None`.
- String columns infer to the new `str` dtype, not `object`.

Package constraints:

| Package | Status |
|---|---|
| `pandas-ta` | ❌ **cannot install** — pins `numba==0.61.2`, no cp314 wheel. Use **TA-Lib**. |
| `vectorbt` | ⚠️ installable, but **don't** — §10: one implementation of exit logic, shared. |
| `torch` | ⚠️ don't — the dataset is far too small for deep learning (§9.1). |
| `catboost` | ✅ available on 3.14 (this is why we're on 3.14, not 3.15). |

## 6. Don't commit data

`reports/` is gitignored and must stay that way — charts and decision reports are
re-derivable output. There is no `data/` or `models/` directory to worry about (§5).

**One exception worth knowing:** `decisions.jsonl` (§11) is the log of calls the system
actually issued, and it is the only genuinely out-of-sample record that will ever exist. It
is gitignored, but never delete or regenerate it.
