# Strategy Validation Pipeline — Freqtrade, fail-closed gate, dry-run only

A research pipeline for crypto trading strategies, built on Freqtrade 2026.8. Its product is not returns but an **honest track record**: a strategy earns paper-trading time only by passing a nine-stage statistical gate, and "0 strategies passed" is a valid, recorded outcome. The code is private (`sherrybuilds-studio/sherry-trading`); the dated result file is [`evals/2026-10-03-trading-gate.json`](../../evals/2026-10-03-trading-gate.json).

## Rules the system enforces on itself

- **Spot, EUR, long-only, dry-run.** No leverage, shorting, DCA or martingale. The gate never places an order and never changes what a bot runs. Going live is a separate interactive script that a human runs and confirms by typing "yes".
- **Fail closed.** Any error, missing file or unparseable result is a FAIL with the reason recorded. Freqtrade exits 0 on some config errors, so the gate trusts result files, never exit codes.
- **Every number carries its data label**: `[BACKTEST <venue> <range>]`, `[PAPER <range>]` or `[SYNTHETIC]`. A repository guard runs before every commit, push and CI build; it rejects secret-shaped strings and a known fabricated accuracy figure.
- **Thresholds are never loosened to pass a strategy.** A change needs its reason in the decision registry and the owner's approval first. 17 such decisions are recorded to date.
- **Retired strategies stay retired.** A graveyard card holds the evidence; bringing an idea back means a new candidate under a new name, which counts as a new trial.
- **Isolation.** The repository lives outside every path the agent fleet can touch; the dispatcher refuses tasks that target it. Secrets are a 0600 file mounted as a Compose secret, never environment variables, and the agent harness blocks reading them.

## The gate, S1 to S9

| Stage | Question it answers | Fails when |
| --- | --- | --- |
| S1 static | Does the strategy code peek at the future? | An AST scan finds a negative `.shift(-n)` or a forward `.iloc[i + 1]` outside the FreqAI target function, before Freqtrade loads anything |
| S2 load | Does Freqtrade load it? | `list-strategies` reports anything but OK |
| S2b data | Does every feed it reads exist on the live venue and in the backtest data? | A pair or timeframe is missing on Kraken spot or too short for the warm-up |
| S2c signals | Are the signals real? | Entry fires on 95% or more of candles, or the exit signal never exited a trade |
| S2d FreqAI | Is a machine-learning model data-starved? | Fewer than 10 training rows per feature |
| S3 recursive | Do indicators depend on how much history was loaded? | A signal indicator drifts past tolerance at 199, 499 or 719 startup candles |
| S4 lookahead | Does it peek at future candles? | Freqtrade's lookahead analysis reports bias on the last 120 days |
| S5 walk-forward | Does it hold up out of sample? | Fewer than 3 of 4 rolling 91-day folds with profit factor above 1, or the aggregate misses trades ≥ 15, profit ≥ 2%, drawdown ≤ 20%, Sharpe ≥ 0.8, PF ≥ 1.2 |
| S6 cost | Does it survive realistic fees? | The fixed out-of-sample trade list, repriced at the stress fee, drops below PF 1.0 |
| S7 Monte Carlo | How bad can a run of the same trades get? | 95th-percentile drawdown over 2,000 resamples exceeds 25% |
| S8 deflated Sharpe | Is the result luck from trying many strategies? | Bailey and López de Prado's DSR below 0.95, with N = every strategy version ever gated, failures included |
| S9 champion | Is it better than what already runs? | Fewer than 28 paper days or 10 closed dry-run trades, drift, or a worse Sharpe than the current champion |

S5 reports two windows: the out-of-sample folds, and the **true out-of-sample** span for community strategies, measured from the upstream commit date their parameters last changed. For the strategies gated so far that span is the full 729 days.

## Fee realism

Kraken's published Tier 1 schedule (fetched 1 Oct 2026) is 0.40% maker / 0.80% taker. The pipeline sets the base fee explicitly at 0.004 per side and places post-only orders, because ccxt's built-in Kraken default (0.16 / 0.26%) would have made every backtest look better than the exchange ever pays. S6 then fails closed twice over: the recorded fees must equal what the bot settings declare, and they must reproduce every recorded profit with Freqtrade's own spot arithmetic.

## Result so far, 3 Oct 2026

`[BACKTEST binance 2024-10-02..2026-10-01]`, in-sample 365 days, four out-of-sample folds of 91 days.

- 12 strategy versions gated (N = 12 for the deflated-Sharpe bar), 8 distinct strategies, **0 passed**. All eight failed at S5.
- Best true out-of-sample profit factor over 729 days: 1.15 (C_AverageStrategy, 12 trades); the reference strategy EmaRsiBase: 0.34.
- Buy-and-hold over the same out-of-sample year, never gated and never ranked: equal weight −44.56%, BTC/EUR −27.21%.

Reading: the out-of-sample year was a bear year, every long-only strategy lost, and over the full span nothing shows an edge at real maker fees. The gate produced that answer instead of a flattering one.

## Data

- Binance history from July 2024 for backtests (5m, 1h, 4h and 1d candles, six EUR pairs), Kraken trade-derived candles from June 2026 for the live venue.
- A data-quality check compares the venues: daily-return correlation between 0.994 (BTC) and 0.998 (ADA). The one known gap (LINK/EUR, one candle) is recorded, and a live gap rule blocks new entries rather than filling it.

## Operations

- Docker Compose on the same server as the other systems, pinned image, credential-free healthcheck, log rotation. A read-only charts container renders the gate's JSON views.
- Offline tests run in CI on every push (actions pinned by SHA), next to the repository guard and a linter.
- Next milestones: dry-run challengers with a drift monitor that compares paper trades with their backtest, then the S9 champion rule.
