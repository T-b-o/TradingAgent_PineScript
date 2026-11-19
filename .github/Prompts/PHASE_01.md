# Phase 01 — APAS V01 S/R Review and Status

## Priority and scope

Phase 01 focuses on the existing APAS chart-timeframe Support and Resistance zones. Preserve the pivot-centred ATR method, original pivot anchor and confirmation delay, exact separate zone boundaries, strength formula, native boxes, inline strength labels, nearest-zone highlighting, and analysis-table integration. Do not combine unrelated methodology or execution changes with this work.

The approved strength formula remains:

`S = 100 * active * A * (0.40*C + 0.35*R + 0.25*T)`

Where `C = min(distinct confirmed retests / 5, 1)`, `R` is the mean of `min(rejectionDistance / reactionBarATR, 1)`, `T = exp(-ln(2) * elapsedSinceLastReaction / (maximumZoneAgeMs / 2))` or zero without a reaction, and `A = max(0, 1 - elapsedZoneAgeMs / maximumZoneAgeMs)`. Support rejection distance is `close - zoneTop`; resistance rejection distance is `zoneBottom - close`. `active` is 1 only while the zone is active. The score is evidence, not probability.

The approved visual system uses native chart-timeframe boxes and inline strength labels, keeps overlapping zones separate, highlights the nearest active support/resistance, and retains the analysis-table integration.

No S/R formula, weight, zone architecture, lifecycle, visual, or scoring change was made during this review.

## S/R source review

The source inspection confirmed:

- Confirmed chart pivots create ATR-sized support/resistance boxes. Boxes are left-anchored at `time[chartPivotRight]`, while the confirmation bar is recorded separately.
- Active boxes extend right. Support zones become inactive on a close below their lower boundary; resistance zones become inactive on a close above their upper boundary. Both also expire at the configured maximum zone age.
- Overlapping zones remain separate. The closest active support/resistance gets the existing stronger emphasis.
- Strength labels are positioned at the zone midpoint and box-right time. `maxZonesPerSide` defaults to 4 (input range 1–10); strategy limits are 200 boxes and 100 labels.
- Source inspection cannot establish visual readability or runtime/performance limits.

## F01 — Retest initial state

**Status: correction present in source; no Pine code change was required in this review.**

`f_zoneStrength()` starts scanning at `confirmBar + 1` and initializes `armed = true`. Thus neither the pivot-formation nor confirmation candle is eligible. A confirmed safe-side zone intersection can count immediately; a counted reaction disarms the state, consecutive touch bars remain one cluster, and a fully outside safe-side bar re-arms it. The safe-side test is symmetric.

The listed sequences were run through a local transition model for support and resistance, not through Pine or TradingView:

| Sequence | Expected count | Model result |
|---|---:|---:|
| First eligible candle touches and closes safe | 1 | 1 |
| Outside candle, then three consecutive safe touches | 1 | 1 |
| Outside candle, unsafe inside-zone close, then safe touch | 1 | 1 |
| Safe touch, unsafe touch, safe touch without an outside candle | 1 | 1 |
| Touch, outside, touch, outside, touch | 3 | 3 |
| Inactive zone | 0 | 0 |

Source inspection confirms an inactive/expired zone skips the scan and has an active factor of zero. The model's inactive case does not itself execute the Pine expiry logic.

The reaction scan does not have a separate configurable lookback input. It scans from the first eligible bar through the current bar for active zones, within loaded chart history. Zone expiry and age use `Maximum zone age (days)`, default 30 and configurable from 1 to 365 days.

### Dynamic history-reference risk

`chartAtr[offset]` and related OHLC/time series are indexed with an offset derived from `barsToScan`. `f_zoneStrength()` is called in the confirmed last-bar strength-label path. The source establishes a history-dependent offset, but no TradingView runtime error has been observed or reproduced. A lower chart timeframe combined with a high maximum-age setting (up to 365 days) is a relevant stress case for a future TradingView test, not a confirmed failure. Reproduction conditions remain unknown. Do not redesign the scan without evidence and a scope review.

## F10 — Take-profit descriptions

**Status: labels and comparisons are present; calculations were not changed.**

- `TP-A: Fixed reward/risk`
- `TP-B: Next opposing zone`
- `TP-C: 50% at 1R / 50% at fixed R:R`

Source inspection found consistent TP-C references in long and short entry and protective-order paths. TP-A and the TP-B fallback use `fixedRewardRisk`; TP-B retains its opposing-zone calculation; TP-C retains 50% exits at 1R and the configured multiple. The input remains default 2.0, range 0.5–10. TP-C behavior at fixed R:R values at or below 1 remains an Owner decision; no guard was added.

## Approved overnight requirement — separate work package

The Owner-approved rule is no new entry orders from 21:00 and close all open APAS strategy positions at 22:00 every trading day, using `Africa/Johannesburg`.

| Status | Result |
|---|---|
| Approved | Yes |
| Implemented | No |
| TradingView validated | No |

The current source has no Johannesburg entry cutoff or scheduled close-all. The two strategy entry calls are gated by the shared entry filter, which currently checks M15, pair, session, news, daily lockout, confirmed bar, and flat position. `strategy.close_all()` currently appears only for daily-loss protection. Do not add overnight logic to this S/R work package.

The strategy has `calc_on_every_tick = false` and `process_orders_on_close = false`; entries require a confirmed M15 bar. A future implementation must test the relationship between bar `time`/`time_close`, order generation, and the next available emulator fill. On a regular M15 sequence the bar ending at 22:00 could generate a close order whose next available tick is the 22:00 open, but this has not been verified. A missing bar/update can delay evaluation or filling. Emulator orders do not guarantee that a live broker account is flattened.

## Other Claude findings

- F02 HTF repainting/lookahead: validation only; no logic changes.
- F03 sizing/margin: broker/emulator validation; no changes.
- F04 signal-close versus actual fill: measure and report; no execution redesign.
- F05 overnight: Owner decision is resolved as above; implementation remains separate.
- F06 commission/slippage: retain current settings; validate realistic assumptions later.
- F07 CHoCH and F08 pullback: historical evidence before any methodology changes.
- F09 correlated scoring: historical evidence before any score/weight changes.

## Validation status

The source assertions and transition model for F01 were executed in the previous focused correction work; they are source/model checks, not Pine tests. Current source review confirms F01/F10 remain present. No Pine v6 compiler result, TradingView runtime, visual review, overlap-readability check, drawing-limit test, performance test, or Strategy Tester result is available. The current worktree diff check reports trailing whitespace at line 242 of the Pine source; it was not changed during this review because it is outside the S/R scope.

**Phase 01 remains in progress.** Do not mark it complete until TradingView compilation and the required runtime/visual checks have evidence.