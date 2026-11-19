# APAS Project Context

## Project identity

| Item | Verified state |
|---|---|
| Project | APAS |
| Solution | APAS V01 — Adaptive Price Action System |
| Type | TradingView Pine Script strategy and technical-analysis system |
| Primary language / version | Pine Script v6 (`//@version=6`) |
| Platform | TradingView |
| Structure | Standalone Pine Script source under `PineScripts_Source/`, `.vscode/settings.json`, ChatGPT/Claude instructions under `AI-Governance/`, and Copilot/Phase 01 instructions under `.github/`. No conventional application, package, or Pine test-project structure was found. |
| Current source | [`PineScripts_Source/APAS_V01 - Adaptive Price Action System.pine`](<./PineScripts_Source/APAS_V01 - Adaptive Price Action System.pine>) |
| Source-declared version | APAS V01 (strategy title, short title, and filename). |

The source has no imports or external Pine libraries. No README or Pine test project was present. Git branch and working-tree state are task-specific and must be inspected before each change; this document does not record a current Git status.

## Strategy objective and approved configuration

APAS (Adaptive Price Action System) is a TradingView Pine Script v6 strategy intended to identify continuation and reversal opportunities. Its approved methodology combines multi-timeframe market structure, APAS support and resistance, RSI divergence, candlestick evidence, double tops/bottoms, neckline breakouts, Change of Character (CHoCH), pullback analysis, and separate weighted continuation and reversal scores.

The approved operating configuration is:

| Requirement | Approved value |
|---|---|
| Instruments | EURUSD and GBPUSD |
| Execution timeframe | M15 |
| Trend / setup context | H4 / H1 |
| Sessions | London and New York |
| Minimum setup score | 80 / 100 |
| Risk per trade / daily loss limit | 1% / 2% |
| News filter | Manually configured blackout windows |
| Overnight controls | No new entries from 21:00; close open APAS strategy positions at 22:00, `Africa/Johannesburg` (approved requirement; not implemented or validated) |
| Live autonomous trading | Not permitted |
| Deployment convention | One strategy instance per currency pair |

These are approved requirements, not evidence that the implementation is validated or complete. The methodology and configuration are requirements; only functionality explicitly described under **Current implementation** is implemented, and none of it should be treated as TradingView-validated unless validation is recorded below.

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

- APAS strategy S/R calculations use confirmed H4/H1 pivots with ATR-sized zones, and previous-day high/low levels. This is APAS's authoritative strategy S/R methodology.
- The APAS visual chart layer uses a native box-based support and resistance system. It does not replace or alter the H4/H1 strategy calculations, scores, entries, or trade decisions.
- A separate chart-timeframe confirmed-pivot/ATR zone layer is drawn on the active chart timeframe. It can show up to four historical zones per side by default; zones extend until invalidated or expired. This display layer is distinct from the H4/H1 strategy levels.
- The visual layer keeps each confirmed zone anchored to its original pivot candle and preserves its original price boundaries. The nearest active support and resistance zones are highlighted and pushed visually forward when price approaches them within the configured ATR buffer.
- The approved strength is a 0–100 evidence score, not a probability: `S = 100 * active * A * (0.40*C + 0.35*R + 0.25*T)`. `C = min(distinct confirmed retests / 5, 1)`. For each reaction, support rejection distance is `close - zoneTop`; resistance rejection distance is `zoneBottom - close`. `R` is the mean of `min(rejectionDistance / ATR_on_reaction_bar, 1)`. `T = exp(-ln(2) * elapsedSinceLastReaction / (maximumZoneAgeMs / 2))`, measured from confirmed bar close to current confirmed close, or zero without a reaction. `A = max(0, 1 - elapsedZoneAgeMs / maximumZoneAgeMs)`, with zone age measured from the original pivot time. `active` is 1 only while the APAS zone is active. Invalidated/expired zones score zero.
- Retests are confirmed post-pivot bars whose range intersects a zone and whose close is above the support top or below the resistance bottom. The first eligible bar is the first bar after pivot confirmation, and it can count as the first reaction without a preceding outside bar. Consecutive touch bars form one cluster; after a reaction, a new retest requires a fully outside confirmed candle on the safe/approach side. This applies symmetrically to support and resistance. For active zones, the scan runs from the first eligible bar through the current bar, subject to available chart history; there is no separate configurable reaction-lookback input. Zone age/expiry uses the configurable maximum zone age (30 days by default).
- Overlapping or closely spaced active zones retain their individual pivot boundaries, stop terms, and evidence scores. Overlap is handled by preserving the original zone boxes and distinguishing the nearest active zone visually; no APAS price levels are merged or recalculated to hide valid structure.

### Risk, execution, and exits

- Strategy defaults: initial capital 10,000, no pyramiding, zero commission and zero slippage. These are TradingView strategy-emulator settings, not verified broker conditions.
- Stop candidates use the latest M15 swing with a configurable ATR buffer (default 0.10 ATR). Quantity is estimated from equity risk, stop distance, and `syminfo.pointvalue`, rounded down to a configurable quantity step (default 1,000) and capped (default 1,000,000). Risk defaults to 1% but is configurable. The source warns that point value and contract sizing require broker validation; account-currency conversion accuracy is not established.
- Daily loss protection compares strategy equity, including unrealized P/L, with an equity baseline reset at UTC midnight. The limit defaults to 2% but is configurable; lockout is enabled by default and can close an open position at the limit. No 21:00 entry cutoff or scheduled 22:00 close is currently implemented; the Owner-approved overnight requirement is a separate work package.
- Take-profit choices are labeled to reflect configurability: `TP-A: Fixed reward/risk` uses the configurable fixed reward/risk setting (default 1:2); `TP-B: Next opposing zone` uses the nearest opposing H4/H1/previous-day level, falling back to the fixed target if none is found; `TP-C: 50% at 1R / 50% at fixed R:R` places two 50% exits at 1R and the configured fixed reward/risk multiple (default 2R). TP-A is not unconditionally fixed at 1:2, and TP-C's second target can differ from 2R. For TP-C, behavior when the configured fixed reward/risk is at or below 1 remains an outstanding Owner decision; no guard or behavior change has been made.
- `strategy.entry`/`strategy.exit` submit orders to TradingView's strategy engine. Alerts are notifications; no broker API or live-order integration is present in the source.

### Chart presentation

**Approved presentation decisions:** display APAS chart-timeframe pivot/ATR S/R as one integrated native visual system; always highlight the nearest active support and resistance; emphasize them more strongly on ATR proximity/touch; retain the original pivot-candle anchor while making its confirmation delay clear; keep overlaps as separate APAS zones; retain the compact analysis table; and keep candle labels optional and limited to relevant S/R areas. There is no separate profile panel, no profile histogram, and no volume-distribution presentation.

**Implemented, not yet TradingView-validated:** each APAS chart-timeframe pivot zone remains the single price-zone box, starting at its original pivot candle and extending right only while active. Strength percentages are rendered directly on or beside the relevant support/resistance box using the approved evidence score. Inactive zones stop at their APAS termination point and show subdued visual treatment. Nearest active support/resistance are highlighted; the existing ATR proximity logic drives the strongest emphasis. Overlap is handled by keeping the original zone boundaries and visually prioritising the nearest active zones instead of merging unrelated price levels.

**Configurable through inputs:** H4/H1/M15 and other chart pivot sensitivities; zone width, proximity and expiry; maximum displayed chart zones; support/resistance styling, nearest-zone emphasis, and confirmation-related label display; candle-label visibility; RSI and divergence parameters; session windows/enables; four manual news blackout windows; risk, stop buffer, quantity rounding/cap and take-profit mode/ratio.

**Hidden by default or not drawn:** candle labels at current chart S/R are off by default. H4/H1 strategy zones and previous-day levels used for scoring/targets; M15 structure, pullback, divergence, double-pattern, and CHoCH calculations remain internal. RSI divergence has no chart markers or lines; the native APAS S/R visual layer does not determine entries. The analysis table and chart-timeframe S/R zones do not have a visibility toggle in the inspected code. The script can display analysis off M15, where entries are blocked.

## Requirement differences and limitations

| Area | Finding |
|---|---|
| Naming/version | Strategy title, short title, and source filename now consistently use APAS V01. |
| Timeframes and score | H4/H1 and score 80 are defaults rather than locked requirements: timeframe inputs can be changed (subject to trend > setup), and the minimum score input can be changed from 1 to 100. |
| Double-pattern validity | 30-bar expiry, minimum separation of 5 bars, and 0.25-ATR similarity are fixed in code rather than configurable. |
| Take profit | The approved TP-A 1:2 and TP-C 2R values are defaults, not fixed, because both use a configurable fixed reward/risk input. TP-B falls back to that configurable fixed target if no opposing level exists. |
| Overnight control | Owner-approved requirement: block all new APAS entries from 21:00 and close all open APAS strategy positions at 22:00 `Africa/Johannesburg`, every trading day. This is not implemented or TradingView-validated and must be handled as a separate work package. |
| Zones | Implementation uses pivot-centered ATR zones and latest HTF zones; advanced refinement/clustering is not implemented. Chart-timeframe zones are visual and do not replace internal H4/H1 scoring zones. |
| Proximity emphasis | Nearest active chart zones use the existing chart-zone proximity logic (`f_zoneTouch` and configured ATR buffer) for stronger visual emphasis. TradingView behavior remains unvalidated. |
| Sizing | `syminfo.pointvalue` sizing is explicitly an approximation pending broker/contract and account-currency validation. |
| Portfolio controls | One-instance-per-pair is an operating rule, not enforced across strategy instances. No portfolio-level exposure or aggregate risk mechanism was found. |
| Live trading | Source contains TradingView strategy-emulator orders and alert notifications, not a broker integration. Live autonomous trading remains prohibited. |
| Backtest assumptions | Commission and slippage default to zero; test results would need realistic instrument/broker settings before conclusions. Risk and daily-loss defaults match approval but can be changed through inputs. |

## Claude review findings — current status

| Finding | Status and treatment |
|---|---|
| F01 — S/R retest initial state | `armed = true` is present; first eligible bar remains `confirmBar + 1`. Source inspection and a separate transition model were performed; TradingView runtime remains unverified. |
| F02 — HTF repainting/lookahead | Outstanding validation only; no logic changes authorized. |
| F03 — Position sizing and margin | Broker/emulator validation outstanding; no sizing changes authorized. |
| F04 — Signal close versus fill | Actual signal-to-fill timing must be measured and reported; no execution redesign authorized. |
| F05 — Overnight closure | Owner decision resolved: 21:00 entry cutoff and 22:00 close in `Africa/Johannesburg`. Implementation and validation are outstanding in a separate work package. |
| F06 — Commission and slippage | Existing settings retained; realistic simulation validation remains outstanding. |
| F07 — CHoCH methodology | Historical evidence required before any methodology change; no changes authorized. |
| F08 — Pullback methodology | Historical evidence required before any methodology change; no changes authorized. |
| F09 — Correlated scoring | Historical evidence required before any score or weight change; no changes authorized. |
| F10 — TP descriptions | Neutral configurable descriptions and corresponding comparisons are present; target calculations are unchanged. TP-C behavior at fixed R:R values at or below 1 remains an Owner decision. |

## Outstanding decisions

- Validate the overnight control's strategy-emulator order timing in a separate work package. If exact timing cannot be guaranteed, present the smallest safe alternative for Owner review.
- Validate quantity step, maximum quantity, contract size, point value, and account-currency conversion for the intended broker and account.
- Decide whether to refine supply/demand zones beyond the current pivot/ATR method.
- Decide whether portfolio-level risk limits are required across currency pairs/instances.
- Compare TP-A, TP-B, and TP-C on approved test data and select/retain the take-profit approach.
- Confirm whether the approved H4/H1 timeframes, 80-point threshold, 1% trade risk, 2% daily loss cap, TP-A 1:2 and TP-C 2R should be enforced rather than user-adjustable.
- Confirm deployment controls that ensure only one strategy instance per pair.
- Decide TP-C behavior when `fixedRewardRisk` is at or below 1 before making any behavioral change.

## Phase 01 — Approved Decisions and Implementation Status

The Project Owner's approval exchange and answers are recorded below:

1. **Question — Native chart S/R presentation: approve or request changes?** **Answer — Approve as proposed.** Remove the profile architecture and keep a native APAS support/resistance box system directly on the chart. The visual layer uses the confirmed chart-timeframe pivot/ATR zone arrays, not a separate profile calculation or histogram. Do not merge overlapping APAS zones; retain their exact prices and preserve their individual evidence.
2. **Question — Strength formula: approve or request changes?** **Answer — Approve as proposed.** The exact approved formula is `S = 100 * active * A * (0.40*C + 0.35*R + 0.25*T)`, where `C = min(distinct confirmed retests / 5, 1)`, `R = mean(min(rejectionDistance / reactionBarATR, 1))`, `T = exp(-ln(2) * elapsedSinceLastReaction / (maximumZoneAgeMs / 2))` or zero without a reaction, and `A = max(0, 1 - elapsedZoneAgeMs / maximumZoneAgeMs)`. Support rejection distance is close minus zone top; resistance rejection distance is zone bottom minus close. `active` is 1 only while the APAS zone is active, otherwise 0. A reaction is a confirmed post-pivot candle intersecting the zone and closing on the safe side; another retest requires a fully outside confirmed candle before the next touch. The score is a normalized S/R evidence score, not a probability or guarantee.
3. **Question — Zone visual anchor: original pivot candle, confirmation candle, or another option?** **Answer — Approve original pivot candle.** Zones remain anchored at the pivot candle and are created only after the existing right-side confirmation delay; this is historical placement, not a claim that the pivot was known at that earlier time.
4. **Question — Changes or conditions?** **Answer — “make sure you document all these question and answers”.** This section records all four questions and answers.

**Implementation status:** the Pine source uses the APAS chart-timeframe pivot/ATR zone arrays as the single source for native S/R boxes and inline strength labels. The visual system is intentionally not a profile panel or histogram, and no H4/H1 or trade-strategy logic was changed.

**Phase 01 priority:** Phase 01 focuses on the existing S/R zones and approved evidence formula. Do not combine unrelated methodology or execution changes with this work. F01 and F10 corrections are present in source; their status does not mean Phase 01 is complete.

**F01/F10 source status:** F01 starts its scan at `confirmBar + 1` with `armed = true`, preserving the existing subsequent cluster/re-arm transitions. F10 uses the configurable TP-A/TP-C descriptions in its input options and corresponding long/short comparisons; the fixed R:R input and target calculations are unchanged.

**Overlap presentation:** exact APAS boundaries and individual scores are retained. Overlapping price ranges are kept as separate zone boxes and visually prioritized by proximity to the nearest active zone. They are not merged, re-bucketed, or hidden behind a profile presentation. These are illustrative values only; no APAS calculation is merged or adjusted.

**Verification status:** source inspection confirms the native APAS chart-timeframe zone arrays drive the boxes and inline strength labels. Zones use confirmed chart pivots, ATR-sized boundaries, original pivot timestamps for their left anchors, right extension while active, and close-break/maximum-age invalidation; overlapping zones remain separate and nearest-zone emphasis is retained. The approved strength formula and reaction transitions are present. A local source assertion and a transition model for the listed reaction sequences were executed for support and resistance; these are not Pine tests. No TradingView compilation, runtime, visual, overlap-readability, drawing-limit, or performance validation has been performed. The dynamic `chartAtr[offset]` history reference runs in the last-bar strength-label path with an offset derived from zone age/bar history; a possible history-buffer/runtime problem remains unverified.

## Phase 01 — Visual and Compilation Validation

**Status:** Phase 01 remains in progress; source inspection and reasoned/modelled state-transition checks do not complete the phase. TradingView validation is outstanding.

Scope and acceptance criteria:

1. Validate APAS chart-timeframe S/R presentation and nearest active support/resistance highlighting.
2. Confirm native S/R box styling, inline strength labels, and analysis-table S/R boundaries match the same APAS chart-timeframe zones.
3. Validate original-pivot placement, zone extension, and invalidation behavior.
4. Validate the compact analysis table.
5. Check chart readability across timeframes.
6. Check EURUSD and GBPUSD.
7. Compile in TradingView using Pine Script v6.
8. Check runtime errors, drawing limits, and performance.

Do not include strategy logic, S/R calculation methodology, scoring, indicators, dependencies, or architectural changes in this phase. Record actual TradingView evidence and any unvalidated items before marking Phase 01 complete.

**Remaining validation:** TradingView Pine v6 compilation, runtime behavior, M15/H1/H4 and EURUSD/GBPUSD visual checks, overlap readability, score display, candle labels, drawing limits, and performance at maximum zone age remain unverified. The `chartAtr[offset]` concern must be tested at relevant chart timeframes and maximum-age settings before determining whether it reproduces.

## Development and verification status

**Status: in development; Phase 01 is not complete.** F01/F10 corrections are present and have source/model checks only. There is no Pine test project or recorded Pine v6 compiler result. TradingView runtime, visual acceptance, overlap readability, drawing limits/performance, repainting/lookahead, broker sizing, execution timing, and Strategy Tester validation remain outstanding. The approved overnight control is recorded but not implemented or validated; it is a separate work package. No broker integration or live-account closure is provided by this TradingView strategy.

## Workspace documentation and governance

No README or separate Claude handoff was found. The workspace contains `.github/COPILOT_INSTRUCTIONS.md`, `.github/Prompts/PHASE_01.md`, `AI-Governance/CLAUDE_INSTRUCTIONS.md`, and `AI-Governance/CHATGPT_INSTRUCTIONS.md`; they establish AI roles and documentation/verification boundaries. This file is the authoritative shared description of APAS requirements, decisions, and verified implementation status.

Approved AI responsibilities:

- ChatGPT: project orchestration, technical planning, requirements, Copilot prompts, and implementation review.
- Claude: independent architecture and technical review; no implementation or project-documentation changes.
- GitHub Copilot: sole authorized implementation and project-documentation agent.
- Project Owner: final decision-making authority.

These role boundaries and authorities are unchanged.
