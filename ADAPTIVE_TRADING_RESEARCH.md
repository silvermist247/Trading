# Adaptive Paper-Trading System: Research and Build Plan

**Status:** design research only; no change to entry/exit behavior has been made in this step.  
**Mode selected:** simulated long/short exposure that reverses when the opposite regime is sufficiently strong.  
**Important:** this is not a promise of profitability, and the script remains a paper simulator with no live orders.

## Executive recommendation

Build this in stages. First make the simulator measure every decision and trade correctly; then evaluate a small, bounded set of adaptive policies offline; only after repeated out-of-sample improvement should any learned policy be used in the indicator.

For “always have a trade,” use a **two-sided exposure state machine** after the MTF warm-up: LONG or SHORT, with hysteresis so it does not reverse on every noisy signal. Keep safety exceptions for missing/stale data, MTF warm-up, a hard drawdown limit, and a stop-out cooldown. Those exceptions mean exposure cannot honestly be guaranteed at every instant. Forcing re-entry after every stop without a cooldown can create a rapid loss loop.

## What the current script already does

The current script is a **rule-based paper simulator**, not an AI learner. It already has useful pieces for adaptive exits:

- It forms a structural stop from recent support/resistance, a pullback, or a breakout/retest area, then adds an ATR buffer.
- It starts the target from an ATR multiple and may bring it closer to nearby prior support/resistance.
- It rejects plans whose stop is too large, whose estimated net reward/risk is too small after configured costs, or which lack room to the next obstacle.
- It sizes simulated positions from the configured risk budget when equity is supplied.
- The chosen stop and target are **fixed after entry**. There is not yet a learned policy, a regime-specific exit selector, or a per-trade training log.
- The chart script keeps aggregate counts/P&L in runtime state. Recompiling or reloading reinitializes that state, so it is not a durable model store.

Earlier proxy backtests in this project did **not** establish a robust edge: tested baselines/filters had full-period zero-cost profit factors below 1 and inconsistent results across costs and splits. That makes a carefully controlled evaluation more important than adding complexity or maximizing trade count.

## Recommended “learning” approach

### Start with a contextual bandit, not deep reinforcement learning

A deep RL agent would have many ways to overfit the historical path and is difficult to validate with a small, non-stationary trade sample. Start with a small set of pre-approved exit policies and let a simple contextual selector compare them by market regime. For example, candidate policies can vary only within safe bounds for:

- structural/ATR stop buffer;
- ATR target multiple or “near structure” target;
- fixed target versus a conservative trailing rule;
- reversal confirmation/cooldown length.

Do not allow the learner to bypass the hard maximum-risk, minimum-net-reward, or data-quality gates. Start with a frozen baseline and a shadow learner that records what it *would* choose before allowing it to affect simulated positions.

### What to log for every simulated decision/trade

The existing aggregate trade counters are not enough to train or diagnose a learner. Record a bounded sample (or an exportable stream) containing:

- timestamp, symbol/timeframe, position side, entry/exit prices, and exit reason;
- MTF scores/trends, EMA spread divided by ATR, ATR as a fraction of price, distance to nearby structure in ATR units, volume ratio, and available tick-direction imbalance;
- selected stop/target policy and the exact stop distance/target distance at entry;
- gross P&L, estimated fees/slippage, **net P&L**, initial risk, net R-multiple, holding time, and maximum favorable/adverse excursion;
- whether the position was reversed, stopped, time-exited, or closed by a safety rule.

All features used for a decision must be available at that decision time. Continue using completed higher-timeframe candles; never train on a future candle value.

### Reward definition

Reward realized, risk-normalized results—not raw quote-currency P&L and not win rate alone. A starting reward can be:

`reward = net_R - λ_drawdown × drawdown_increase - λ_turnover × reversal_cost_in_R`

where `net_R` is realized P&L **after** costs divided by the trade’s initial risk. The penalties discourage a policy from appearing successful by taking excessive risk or flipping constantly. Tune the penalty weights only on training/validation periods, never on the final holdout.

Only update a policy after a minimum sample count for that regime/action. A single lucky TP should not cause the next stop or target to change. A conservative Bayesian/shrinkage estimate or simple decayed average is safer than rewarding a policy immediately after one winner.

## Continuous-exposure design

Use a state machine rather than opening a new position on every calculation:

1. **WARM-UP / SAFETY LOCK:** no forced position until the 1m chart and all completed 5m/15m/1h/4h contexts are ready. Also lock out on invalid data or a configured hard loss limit.
2. **INITIAL DIRECTION:** once ready, select LONG or SHORT from the MTF regime/confluence score.
3. **HOLD:** retain the current side through neutral/noisy readings; use separate enter and reverse thresholds (hysteresis).
4. **REVERSE:** change sides only after an opposite regime has crossed its stronger threshold, optionally for more than one completed 1m candle. Charge the reversal cost in the simulator.
5. **STOP/TP EXIT:** apply the position’s protective exit. Do not instantly re-enter on the same tick; use a bounded cooldown/fresh confirmation to reduce repeated stop-outs.

Benchmark this mode against the existing “allow FLAT” mode. Report exposure percentage and reversal/turnover alongside P&L; a strategy that is almost always invested can still be worse after costs and drawdown.

## Market-dependent stop and target policy

Keep the hard risk controls and let market regime select among bounded choices:

- **Range / weak trend:** use the nearer structural target and a conservative stop; reject trades with inadequate room. Do not stretch targets just to make a reward/risk threshold pass.
- **Strong aligned trend:** retain the structural stop, size down if its distance is larger, and consider a modest target plus a trailing exit rather than an arbitrarily distant fixed TP.
- **High volatility:** reduce unit size to keep quote-risk bounded; if the structural stop exceeds the maximum allowed ATR distance, skip or remain in the prior state rather than widening risk.
- **After entry:** a trailing stop may move only toward reducing risk (never farther away). Any target/stop update must use data available at that time and be tested with realistic intrabar fill assumptions.

A candidate policy should still satisfy the configured cost-adjusted minimum reward/risk at entry. The learner may choose among safe templates, but it must not override that gate.

## Validation gate before trusting adaptation

1. Compare static exits, market-regime exits, and the continuous-exposure state machine on identical chronological data.
2. Use walk-forward splits (train on earlier data, tune on a following validation window, then evaluate once on a later untouched holdout). Do not randomly shuffle candles.
3. Include realistic spread/commission/slippage assumptions and conservative stop/target fill rules. The current simulator already assumes the stop first when one candle touches both levels; preserve that conservatism.
4. Track net expectancy in R, profit factor, maximum drawdown and drawdown duration, exposure, turnover/reversal count, stop-outs, and results by volatility/trend regime. Treat hit rate as secondary.
5. Keep the learner in **shadow mode** until it beats the frozen baseline across multiple walk-forward windows without unacceptable drawdown. Then paper-test the frozen version in replay/live updates before considering any further change.

## Suggested implementation order

1. Add bounded per-trade/decision logging and a simple equity/drawdown series to the paper simulator.
2. Add a continuous-exposure paper mode with hysteresis, cooldown, and explicit safety overrides; leave the current flat-when-no-setup mode available for comparison.
3. Add regime labels and a small fixed grid of safe stop/target templates; compare them without learning.
4. Add a shadow contextual selector using the reward above and minimum sample requirements.
5. Only after walk-forward evidence, let the selector choose the simulated exit template. Keep learning/versioning outside the chart script unless a durable storage mechanism is confirmed; the current `self` state is reset on reload.

## Further reading

These are implementation references for research environments—not evidence that their agents are profitable:

- [FinRL](https://github.com/AI4Finance-Foundation/FinRL) — financial RL research framework and environment patterns.
- [gym-anytrading](https://github.com/AminHP/gym-anytrading) — a compact trading-environment example useful for understanding action/reward design.
- [vectorbt](https://github.com/polakowo/vectorbt) — high-throughput strategy research/backtesting patterns.
