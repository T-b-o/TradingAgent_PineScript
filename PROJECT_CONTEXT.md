# APAS Project Context

## Project identity

| Item | Verified state |
|---|---|
| Project | APAS |
| Solution | APAS V01 — Adaptive Price Action System |
| Type | TradingView automated strategy and technical-analysis system |
| Primary language / version | Pine Script v6 (`//@version=6`) |
| Platform | TradingView |
| Structure | Standalone Pine Script source under `PineScripts_Source/`, plus `.vscode/settings.json` and AI governance instructions under `AI-Governance/`. No conventional application, package, or test-project structure was found. |
| Current source | [`PineScripts_Source/APAS_V01 - Adaptive Price Action System.pine`](<./PineScripts_Source/APAS_V01 - Adaptive Price Action System.pine>) |
| Source-declared version | APAS V01 (strategy title, short title, and filename). |

The source has no imports or external Pine libraries. No README or Pine test project was present. The existing context draft and AI governance files were reviewed. The current uncommitted Pine change renames the strategy title, short title, table header, and alert text from V01.2 to V01; the inspected diff contains no strategy-logic changes.

## Strategy objective and approved configuration

APAS is intended to identify continuation and reversal opportunities using confirmed multi-timeframe market structure, support/resistance, RSI divergence, candlestick evidence, double tops/bottoms, neckline breakouts, CHoCH, and separate weighted scores. The approved operating configuration is:

| Requirement | Approved value |
|---|---|
| Instruments | EURUSD and GBPUSD |
| Execution timeframe | M15 |
| Trend / setup context | H4 / H1 |
| Sessions | London and New York |
| Minimum setup score | 80 / 100 |
| Risk per trade / daily loss limit | 1% / 2% |
| News filter | Manually configured blackout windows |
| Overnight positions | Must be closed before overnight holding |
| Live autonomous trading | Not permitted |
| Deployment convention | One strategy instance per currency pair |

These are approved requirements, not evidence that the implementation is validated or complete.

## Current implementation

### Structure, setups, and scoring

- Entry logic is restricted to M15 and to forex symbols identified as EURUSD or GBPUSD. Other chart timeframes can display analysis but do not pass the entry filter.
- Trend and setup timeframes default to H4 and H1 and are configurable; the only ordering validation is that the trend timeframe be higher than the setup timeframe. H4 and H1 structure use separate configurable pivot sensitivity. M15 entry structure has its own pivots. H4/H1 structure requests use `lookahead_on` while returning prior requested-timeframe-bar values from the structure function; previous-day high/low use the prior completed daily candle. This is intended to avoid developing HTF values influencing decisions, but has not been independently tested for repainting.
- Continuation scores are separate for long and short: H4 direction 20, H1 direction 15, S/R 20, pullback 15, RSI divergence 10, and candle evidence 20.
- Reversal scores are separate from continuation scores and from each other by direction: S/R 20, RSI divergence 20, double-top/bottom breakout 15, CHoCH 20, candle evidence 20, and session 5. The minimum score defaults to 80 but is configurable from 1 to 100. Setup conditions also impose required structure/zone conditions; a score alone does not trigger an entry.
- Pullbacks use a 30–70% retracement of the latest M15 swing range. Double tops/bottoms require intervening opposite pivots, highs/lows within 0.25 ATR, at least five bars of separation, and a confirmed close crossing the neckline. Untriggered candidates expire after 30 M15 bars; similarity, separation, and expiry values are hard-coded rather than configurable.
- RSI period defaults to 14 and is configurable. Confirmed price pivots are compared with RSI values at those pivots; pivot separation defaults to 5–30 bars and divergence remains active for 20 M15 bars. These settings are configurable.
- Implemented bullish candle patterns: Bullish Engulfing, Morning Star, Piercing Line, Hammer, Inverted Hammer, Bull Marubozu. Bearish: Bearish Engulfing, Evening Star, Dark Cloud Cover, Shooting Star, Hanging Man, Bear Marubozu. Candle evidence contributes to scoring only at strategy S/R; it is not an independent entry.
- CHoCH is represented by price crossing the latest M15 pivot high/low. There is no separate drawn CHoCH annotation.
- London and New York session filters default to weekdays 08:00–17:00 in `Europe/London` and `America/New_York`, respectively, and each can be disabled or reconfigured. Four optional manual news-blackout intervals are supported; there is no automatic news feed.

### Support/resistance

- Strategy calculations maintain the latest H4 and H1 demand/supply zones, centered on confirmed pivots and sized using ATR. Zones are invalidated by a close through the far edge or by age; maximum age defaults to 30 days. Previous-day high/low are included in strategy zone proximity and opposing-target selection.
- These H4/H1 and previous-day levels are internal calculations and are not drawn as strategy zones.
- A separate chart-timeframe pivot-zone layer is drawn on the active chart timeframe. It can show up to four historical zones per side by default; zones extend until invalidated or expired. This display layer is distinct from the H4/H1 strategy levels.
- The optional visual profile is also separate from strategy scoring. It bins volume over a fixed or pivot-anchored dynamic lookback and draws low-volume areas, a right-side profile, and an optional lookback background. Forex volume depends on the TradingView data feed and may be tick volume.

### Risk, execution, and exits

- Strategy defaults: initial capital 10,000, no pyramiding, zero commission and zero slippage. These are TradingView strategy-emulator settings, not verified broker conditions.
- Stop candidates use the latest M15 swing with a configurable ATR buffer (default 0.10 ATR). Quantity is estimated from equity risk, stop distance, and `syminfo.pointvalue`, rounded down to a configurable quantity step (default 1,000) and capped (default 1,000,000). Risk defaults to 1% but is configurable. The source warns that point value and contract sizing require broker validation; account-currency conversion accuracy is not established.
- Daily loss protection compares strategy equity, including unrealized P/L, with an equity baseline reset at UTC midnight. The limit defaults to 2% but is configurable; lockout is enabled by default and can close an open position at the limit. It does not implement a session-end or overnight close.
- Take-profit choices: TP-A uses the fixed reward/risk setting (default 1:2); TP-B uses the nearest opposing H4/H1/previous-day level, falling back to the fixed target if none is found; TP-C places two 50% exits at 1R and the configured fixed reward/risk multiple (default 2R). The fixed reward/risk input is configurable, so TP-A is not unconditionally fixed at 1:2 and TP-C's second target can also differ from 2R.
- `strategy.entry`/`strategy.exit` submit orders to TradingView's strategy engine. Alerts are notifications; no broker API or live-order integration is present in the source.

### Chart presentation

**Approved principle:** keep the chart clean and readable; internal calculations do not imply that a value must be drawn.

**Displayed:** active-chart-timeframe pivot S/R boxes and a compact top-right analysis table with current chart S/R, H4/H1 context, scores, filters, and signal status. The profile and lookback background are also displayed by default.

**Configurable through inputs:** H4/H1/M15 and other chart pivot sensitivities; zone width, proximity and expiry; maximum displayed chart zones; profile visibility, lookback, resolution, width, offset and colors; candle-label visibility; RSI and divergence parameters; session windows/enables; four manual news blackout windows; risk, stop buffer, quantity rounding/cap and take-profit mode/ratio.

**Hidden by default or not drawn:** candle labels at current chart S/R are off by default. H4/H1 strategy zones and previous-day levels used for scoring/targets; M15 structure, pullback, divergence, double-pattern, and CHoCH calculations remain internal. RSI divergence has no chart markers or lines; the volume profile does not determine entries. The analysis table and chart-timeframe S/R zones do not have a visibility toggle in the inspected code. The script can display analysis off M15, where entries are blocked.

## Requirement differences and limitations

| Area | Finding |
|---|---|
| Naming/version | Strategy title, short title, and source filename now consistently use APAS V01. |
| Timeframes and score | H4/H1 and score 80 are defaults rather than locked requirements: timeframe inputs can be changed (subject to trend > setup), and the minimum score input can be changed from 1 to 100. |
| Double-pattern validity | 30-bar expiry, minimum separation of 5 bars, and 0.25-ATR similarity are fixed in code rather than configurable. |
| Take profit | The approved TP-A 1:2 and TP-C 2R values are defaults, not fixed, because both use a configurable fixed reward/risk input. TP-B falls back to that configurable fixed target if no opposing level exists. |
| Overnight control | No close-before-overnight behavior exists yet. The required cutoff time and timezone are undecided. |
| Zones | Implementation uses pivot-centered ATR zones and latest HTF zones; advanced refinement/clustering is not implemented. Chart-timeframe zones are visual and do not replace internal H4/H1 scoring zones. |
| Sizing | `syminfo.pointvalue` sizing is explicitly an approximation pending broker/contract and account-currency validation. |
| Portfolio controls | One-instance-per-pair is an operating rule, not enforced across strategy instances. No portfolio-level exposure or aggregate risk mechanism was found. |
| Live trading | Source contains TradingView strategy-emulator orders and alert notifications, not a broker integration. Live autonomous trading remains prohibited. |
| Backtest assumptions | Commission and slippage default to zero; test results would need realistic instrument/broker settings before conclusions. Risk and daily-loss defaults match approval but can be changed through inputs. |

## Outstanding decisions

- Define the end-of-day cutoff and timezone, then approve how and when overnight closure should execute.
- Validate quantity step, maximum quantity, contract size, point value, and account-currency conversion for the intended broker and account.
- Decide whether to refine supply/demand zones beyond the current pivot/ATR method.
- Decide whether portfolio-level risk limits are required across currency pairs/instances.
- Compare TP-A, TP-B, and TP-C on approved test data and select/retain the take-profit approach.
- Confirm whether the approved H4/H1 timeframes, 80-point threshold, 1% trade risk, 2% daily loss cap, TP-A 1:2 and TP-C 2R should be enforced rather than user-adjustable.
- Confirm deployment controls that ensure only one strategy instance per pair.

## Development and verification status

**Status: in development; not approved as complete.** This inspection verified workspace contents, the current Pine source, and its diff against the available Git baseline. The diff is limited to V01.2-to-V01 naming in the strategy title, short title, table header and alerts. No Pine v6 compiler result, automated tests, independent strategy-logic review, repainting/lookahead test, broker sizing validation, execution validation, visual acceptance review, or TradingView Strategy Tester report was available. Compilation and strategy behavior therefore remain unverified, including entry/exit behavior, daily loss handling, TP calculations, chart behavior across symbols/timeframes, and performance. No independent live-trading or broker validation has been performed.

## Workspace documentation and governance

No README was found. The workspace contains `AI-Governance/copilot-instructions.md`, `AI-Governance/CLAUDE_INSTRUCTIONS.md`, and `AI-Governance/CHATGPT_INSTRUCTIONS.md`; they establish AI roles and documentation/verification boundaries. This file is the shared source of project context for ChatGPT, Claude, and Copilot and should be updated when verified implementation or approved requirements change.
