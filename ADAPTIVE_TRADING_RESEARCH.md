# Adaptive Paper-Trading System: Research and Build Plan

**Status:** the paper simulator is in `trading` and `fighter_jet.indie`. Its bounded research layer includes a UTC-day aggregator, online NN/KNN studies, horizontal S/R retest/break scoring, a MACD shadow vote, SFX-style consensus, staggered SIM TP ladders, and SIM performance metrics. The latest user screenshot showed the MACD/HUD revision loaded with a blank Exness error checker; the persistent arrow-redraw change still needs Exness replay and visual confirmation.
**Mode selected:** simulated long/short exposure that reverses only after a confirmed opposing regime, with stop cooldowns and risk locks.
**Chat panel:** top-right rule-based chart readout with a selectable prompt in indicator settings; it is not a free-typing or external-LLM chat service.
**Important:** this is not a promise of profitability, and the script remains a paper simulator with no live orders.

## New bounded shadow research layer

- A fifth, UTC-bucketed daily aggregator feeds a small 6-input/5-hidden/1-output online neural regressor. Its six normalized inputs are EMA spread/ATR, RSI, Bollinger-mid deviation/ATR, efficiency ratio, relative ATR, and candle/volume pressure. It learns from the following completed daily close and reports its rolling classification error beside a daily SuperTrend baseline. It resets with the indicator/runtime settings and is not durable.
- A bounded 100-sample KNN compares those normalized daily features. Neighbors are labeled by whether the next daily move continued or opposed the SuperTrend direction; the displayed neighbor share is a historical continuation share, **not a win probability**.
- The existing horizontal rolling S/R levels now collect touch → bounce/break outcomes over the next three closed 1m bars. A small online bounded score is trained from the touch context; its displayed score is explicitly uncalibrated. No vertical S/R drawings or manual-arrow inputs are used.
- The SFX-style readout is a weighted rule/model vote with a sensitivity threshold that rises with volatility/stress and relaxes modestly when local and MTF components agree. `Scalp`, `Day`, and `Swing` adjust that shadow sensitivity and the TP1/TP2 fractions of distance to TP3; the original TP3 and stop/RR entry gate remain unchanged.
- MACD uses a manually seeded 12/26 EMA difference and 9-period signal EMA on completed 1m bars. The live histogram/ATR is shown in the analyst readout and contributes a bounded 0.10-weight vote to SFX only; it does not change entry confluence/gates or the existing 12 plot outputs.
- The bottom-right analyst readout remains at its prior chart position, with a smaller font, seven concise left-aligned rows, and shorter position/risk text to reduce clipping.
- The top-left progress panel adds a separate warning label (red for avoid-buy/downtrend, green for avoid-sell/uptrend) and a signed SFX directional-vote meter. These are advisory confluence diagnostics, not profitability probabilities and not new trade gates.
- Swing-proxy arrows are queued once from each cached closed-bar candidate on the next candle. A 50-event queue is retained, while at most 11 arrows (33 line segments) are re-rendered per update; with 13 HUD labels and up to two SIM exit lines, the update stays at or below 48 drawing changes. Each arrow still requires a qualifying OHLCV proxy.
- A confirmed daily stress state can veto a fresh entry at planning and next-open checks. It does not force-close an open SIM position or loosen/replace any stop, sizing, cooldown, MTF, drawdown, or cost/RR control.
- TP1/TP2/TP3 are simulated as 33%/33%/34% staggered exits inside the original TP3 target. The entry stop/RR gate and initial size calculation still use the unchanged structural stop and full TP3 plan; if a candle touches the stop and any target, the stop is assumed first. These are paper fills, not broker orders. `Scalp`, `Day`, and `Swing` select TP1/TP2 distances inside TP3. SIM closed-trade count, win rate, net profit factor, average net R, and configured-equity drawdown are displayed separately.
- Model and consensus scores are diagnostics, not calibrated probabilities or proof of an edge. The daily aggregator is UTC-based, not the broker's native daily session. The NN/KNN/S/R/MACD/SFX outputs are shadow-only except for the restrictive daily-stress entry veto; the TP ladder changes only paper-simulator exits.

## Executive recommendation

The first in-indicator prototype is implemented with bounded, rule-based exit templates and in-memory outcome scoring. Treat it as experimental: next add durable/exportable trade logs, then evaluate the policies in a separate walk-forward harness. Do not trust the selector or change to live execution without repeated out-of-sample improvement.

## Implemented prototype and remaining gaps

- Continuous exposure is enabled by default, while setup-only mode remains available for comparison. New directions still wait for MTF warm-up and valid structural/ATR stop, target, cost, room, and reward/risk gates. Reversals require the stronger opposite regime for the configured number of 1m bars. TP/SL exits start a cooldown.
- A bounded three-template selector adapts target distance and trailing behavior by range/trend/high-volatility context. It rotates templates during warm-up and only promotes a non-baseline template after at least five outcomes per template in that context. Rewards use net R after configured costs, with modest drawdown and reversal penalties.
- The trend template can trail only after at least +1 initial R, and the stop can only tighten. A realized-equity drawdown cap can lock further entries when quote-currency equity is configured. Without equity, the simulator uses one unit and cannot calculate percentage risk or the drawdown cap.
- The selector keeps only per-context reward sums/counts in runtime memory. It does **not** yet persist a model or export a complete feature/outcome log; reloading/reinitializing resets these scores. It is a simple rule-based paper selector, not an adaptive AI service.
- The chart's “JET CHAT” box is a read-only rule-based explanation panel. Users choose one of four prompts in the indicator settings; Indie's drawing label does not accept free-form chart input or call an LLM.

For “always have a trade,” use a **two-sided exposure state machine** after the MTF warm-up: LONG or SHORT, with hysteresis so it does not reverse on every noisy signal. Keep safety exceptions for missing/stale data, MTF warm-up, a hard drawdown limit, and a stop-out cooldown. Those exceptions mean exposure cannot honestly be guaranteed at every instant. Forcing re-entry after every stop without a cooldown can create a rapid loss loop.

## What the current script already does

The **trade engine** remains a rule-based paper simulator with no live orders. A small in-session neural/KNN research layer now runs in shadow mode beside it, while the existing bounded outcome selector still adapts only among pre-approved exit templates:

- It forms structural stops from recent support/resistance, a pullback, or a breakout/retest area, then adds an ATR buffer.
- It selects one of three bounded exit templates—SAFE, BASE, or TRAIL—within the existing structural, max-ATR-stop, room, cost, and net reward/risk gates. The target scale responds to range/trend/high-volatility context; the trend template can trail after +1 initial R, and a stop can only tighten.
- It rotates templates while gathering outcomes, then selects a non-baseline template only after at least five outcomes per template in that context and a 0.10 net-R advantage over baseline. Reward is net R after estimated costs, less drawdown-increase and reversal penalties. These thresholds are safeguards, not proof of edge.
- Continuous long/short paper exposure is on by default, with a stricter opposite-regime threshold, 2-bar confirmation by default, stop/target cooldown, a maximum initial stop distance, and a realized-drawdown lock when equity is supplied. Setup-only mode remains available for comparison.
- When quote-currency equity is supplied, simulated size is based on the risk budget. Without it, the script simulates one unit; percentage risk and drawdown cannot be computed.
- The selector stores only reward sums/counts by regime/template in runtime memory. It does not yet export complete per-trade features, and reload/reinitialization clears its state. A durable, exportable training log remains a key next step.

Earlier proxy backtests in this project did **not** establish a robust edge: tested baselines/filters had full-period zero-cost profit factors below 1 and inconsistent results across costs and splits. That makes a carefully controlled evaluation more important than adding complexity or maximizing trade count.

## Recommended “learning” approach

### Start with a contextual bandit, not deep reinforcement learning

A deep RL agent would have many ways to overfit the historical path and is difficult to validate with a small, non-stationary trade sample. Start with a small set of pre-approved exit policies and let a simple contextual selector compare them by market regime. For example, candidate policies can vary only within safe bounds for:

- structural/ATR stop buffer;
- ATR target multiple or “near structure” target;
- fixed target versus a conservative trailing rule;
- reversal confirmation/cooldown length.

The current prototype explores these bounded templates only in paper simulation and cannot bypass the hard maximum-risk, minimum-net-reward, or data-quality gates. For any future version beyond paper-only use, keep a frozen baseline and shadow selector that records what it *would* choose until walk-forward validation passes.

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

## Next steps before trusting the prototype

1. Add a bounded, exportable per-trade/decision log with timestamp, symbol/timeframe, feature snapshot, selected template, entry/exit reason, net P&L, net R, and stop/target distances. Current runtime state keeps only aggregate rewards/counts.
2. Build a separate chronological walk-forward harness comparing setup-only mode, continuous exposure, fixed exits, and the three adaptive templates on identical data.
3. Validate costs, gap fills, stop/target order, drawdown, turnover, exposure, and results by volatility/trend regime. Keep untuned data for one final holdout.
4. Re-run the selector in shadow mode against the frozen baseline; use multiple out-of-sample windows and a minimum sample threshold before trusting a policy choice.
5. Keep model persistence/versioning outside the chart script unless a durable storage mechanism is confirmed. In the indicator, treat scores as session/replay-local; reload/reinitialization resets them.
6. Compile and visually check in Exness, including chart replay and live updates, especially the top-right readout position and the trailing-stop behavior.

## Further reading

These are implementation references for research environments—not evidence that their agents are profitable:

- [FinRL](https://github.com/AI4Finance-Foundation/FinRL) — financial RL research framework and environment patterns.
- [gym-anytrading](https://github.com/AminHP/gym-anytrading) — a compact trading-environment example useful for understanding action/reward design.
- [vectorbt](https://github.com/polakowo/vectorbt) — high-throughput strategy research/backtesting patterns.
