# Ghost Wick v5.8 — Test Plan

## Purpose

Isolate remaining bugs and validate fixes #25–#29 across diverse market conditions. Each instrument targets specific code paths, entry types, and edge cases. Results feed directly into future fix items.

---

## Instruments & Rationale

| # | Instrument | Exchange | Why Selected |
|---|-----------|----------|-------------|
| 1 | **PENGU/USDT** | Binance | Low-price coin (6-decimal mintick). Tests format.mintick precision (#27), thin liquidity mode, absorption mode on micro-cap. High wick-to-body ratio exposes stop/TP collision heuristic (#26). |
| 2 | **BTC/USDT** | Binance | High-price, high-liquidity benchmark. Tests HTF auto-scaling across timeframes, displacement entries (RVOL spikes), trend long/short. Validates that format.mintick shows 2 decimals (not 4). |
| 3 | **HYPER/USDT** | Binance | Mid-cap with strong trending phases. Tests Wyckoff Phase E scoring (#28), absorption mode probability accuracy, Phase D/E distribution equivalents. Known Bug #8/#9 validation target. |
| 4 | **ETH/USDT** | Binance | High-liquidity, moderate volatility. Tests range fade entries, continuation entries, structural invalidation paths. Good for EMA slope filter and MACD momentum validation. |
| 5 | **SOL/USDT** | Binance | High-volatility large-cap. Tests wide-range bar stop/TP collision (#26), same-bar stop guard (#25) on displacement candles, trend ride trailing mechanics. Fast regime transitions (range↔trend). |

---

## Timeframes Per Instrument

Each instrument requires **3 timeframes** to test HTF auto-scaling and entry type coverage.

| Instrument | TF 1 (Scalp) | HTF Auto | TF 2 (Swing) | HTF Auto | TF 3 (Position) | HTF Auto |
|-----------|--------------|----------|--------------|----------|-----------------|----------|
| PENGU/USDT | 3m | 15m | 30m | 2H | 2D | 3D |
| BTC/USDT | 5m | 15m | 1H | 3H | 4H | 7H |
| HYPER/USDT | 15m | 1H | 3H | 7H | 1D | 3D |
| ETH/USDT | 5m | 15m | 30m | 2H | 4H | 7H |
| SOL/USDT | 3m | 15m | 1H | 3H | 1D | 3D |

---

## Config Settings Per Test

Use **default settings** unless specified. Deviations listed per scenario.

### Global Defaults (All Tests)

| Setting | Value | Notes |
|---------|-------|-------|
| Mode | AUTO | Let classifier pick DEFAULT/THIN/ABSORPTION |
| HTF timeframe | 240 (default) | Auto-scaling active |
| HTF alignment | ON | Default behavior |
| HTF strict | OFF | Single HTF gate |
| Kill zone filter | ON, Auto-detect ON | Auto-disables on crypto |
| Thin liquidity mode | OFF | Enable only for PENGU thin test |
| Stalking mode | ON | Default |
| Prob threshold (range) | 0.55 | Default |
| Prob threshold (trend) | 0.45 | Default |
| Min structural R:R | 2.0 | Default |
| BB Squeeze filter | ON | Default |
| RSI Divergence | ON | Default |
| EMA slope filter | ON | Default |
| MACD filter | ON | Default |

### Instrument-Specific Overrides

| Instrument | Override | Value | Why |
|-----------|----------|-------|-----|
| PENGU/USDT (Test 1C) | Thin liquidity mode | ON | Test thin mode conviction spread bypass |
| PENGU/USDT (Test 1C) | Thin mode conviction spread | 0.0 | Validate zero-spread thin entries |
| BTC/USDT (Test 2B) | HTF strict | ON | Test dual-HTF gate (4H + Daily) |
| HYPER/USDT (Test 3A) | Mode override | ABSORPTION | Force absorption mode for Wyckoff testing |

---

## Test Scenarios

### Instrument 1: PENGU/USDT

| Test ID | Timeframe | Scenario | What to Verify | Entry Types Tested | Code Path / Fix |
|---------|-----------|----------|---------------|-------------------|----------------|
| 1A | 3m | **Range fade in micro-range** | Dashboard shows full 6-decimal precision (e.g., 0.006421 not 0.0064). Stop label shows S:/BE:/TR: correctly. NEXT row shows full precision with $ prefix. | FADE LONG, FADE SHORT | Fix #27 (format.mintick), Fix #27 (dynamic stop label) |
| 1B | 30m | **Stop/TP collision on wick bar** | Find a bar where low ≤ stop AND high ≥ TP1. Verify open-proximity heuristic: if open closer to TP1 → trade transitions to MANAGING at BE (0R), not -1R. Check R counter in dashboard. | TREND LONG, TREND SHORT | Fix #26 (open-proximity heuristic) |
| 1C | 30m | **Thin liquidity mode entries** | Enable thin mode. Verify conviction spread bypassed, entries fire with lower spread requirement. Check mode tag shows "THIN" in dashboard. Compare signal count vs default mode. | All types | Thin mode path, i_thin_conv |
| 1D | 2D | **Same-bar stop guard on entry bar** | Find a reversal entry where entry bar low < stop level. Verify trade survives to next bar (Fix #25 guard). Check entry_bar_idx set correctly. | Reversal entries | Fix #25 (bar_index > entry_bar_idx) |
| 1E | 2D | **Breakeven label after TP1 hit** | After TP1 hit, verify dashboard shows "BE:" prefix instead of "S:". If trail moves stop above entry, verify "TR:" prefix. | Any → MANAGING | Fix #27 (BE:/TR: label) |

**Screenshots needed (PENGU):**
1. 3m chart with active FADE signal — dashboard visible showing full precision
2. 30m chart with a wide-wick bar during active trade — dashboard R counter visible
3. 2D chart with active trade in MANAGING state — dashboard showing BE: or TR: label
4. 30m thin mode — side-by-side comparison: default mode vs thin mode signal count

---

### Instrument 2: BTC/USDT

| Test ID | Timeframe | Scenario | What to Verify | Entry Types Tested | Code Path / Fix |
|---------|-----------|----------|---------------|-------------------|----------------|
| 2A | 5m | **Displacement breakout with RVOL** | DISP LONG/SHORT fires with RVOL ≥ 1.2x, CVD alignment, body dominance. Verify HTF auto-scales to 15m. Dashboard shows 2-decimal precision (e.g., 97500.00). | DISP LONG, DISP SHORT | Fix #27 (format.mintick on BTC), HTF auto-scaling (#20) |
| 2B | 1H | **HTF strict mode (dual gate)** | Enable HTF strict. Verify both primary (3H) and secondary HTF must align. Count signal reduction vs non-strict. Dashboard HTF row shows both timeframes. | TREND LONG, TREND SHORT | HTF strict mode, i_htf_strict |
| 2C | 4H | **Structural invalidation (no TP collision)** | Find trade where struct_invalid fires WITHOUT TP hit. Verify exit at close price (not stop_price). Confirm R calculated from close, not stop. | Any → struct_invalid | Fix #26 (separated struct/range branch) |
| 2D | 1H | **Continuation entry after pullback** | CONT LONG/SHORT fires on pullback into structure after BOS. Verify EMA slope filter active, MACD momentum aligned. Check trade_state 4 → 1 → 2 transition. | CONT LONG, CONT SHORT | Continuation path, EMA slope (#NEW-6) |
| 2E | 5m | **OBV flow exit (highest priority)** | Active trade exits via OBV divergence against position. Verify this fires before other exit conditions. Check exit recorded as win (close > entry for long). | Any → OBV exit | Exit chain priority order |

**Screenshots needed (BTC):**
1. 5m chart with DISP entry — dashboard showing 2-decimal prices + RVOL indicator
2. 1H strict mode — dashboard showing dual HTF alignment status
3. 4H chart with struct_invalid exit — R counter showing close-based R (not stop-based)
4. 1H chart with CONT entry — EMA slope and MACD visible in dashboard

---

### Instrument 3: HYPER/USDT

| Test ID | Timeframe | Scenario | What to Verify | Entry Types Tested | Code Path / Fix |
|---------|-----------|----------|---------------|-------------------|----------------|
| 3A | 3H | **Wyckoff Phase E scoring (MARKUP)** | Force ABSORPTION mode. Find Phase E:MARKUP on chart. Verify bull_prob includes +0.12 from Phase E. Compare to v5.7 (should show higher prob). Dashboard Wyckoff row shows lime color. | ABSORB LONG | Fix #28 (Phase E scoring), Fix #29 (dashboard color) |
| 3B | 3H | **Wyckoff Phase E distribution (MARKDOWN)** | Find Phase E:MARKDOWN (bearish). Verify bear_prob includes +0.12 from Phase E_dist. Chart label shows "W:E↓" in red. Dashboard lime color active. | ABSORB SHORT | Fix #28 (Phase E_dist), Fix #29 |
| 3C | 15m | **Exclusive scoring chain validation** | Find bar where Phase D AND Phase C conditions could both be true. Verify only Phase D scores (0.10), not D+C (0.18). Check wyckoff_phase_str shows "D:BOS" not "C:SPRING". | Any absorption | Fix #28 (exclusive chain) |
| 3D | 1D | **Absorption spring entry at range bottom** | Confirmed range → spring (wick below range low, close back inside). Verify spring entry fires, Wyckoff shows "C:SPRING", prob scoring correct. | ABSORB LONG (spring) | Absorption spring path, i_abs_spring_enabled |
| 3E | 3H | **HTF bearish drag on bull probability** | Phase E:MARKUP active but HTF bearish. Verify bull_prob is below threshold (~29% per Fix #28 analysis). Document exact scoring breakdown for Bug #10 analysis. | No entry expected | Bug #10 investigation (symmetric scoring) |

**Screenshots needed (HYPER):**
1. 3H chart with E:MARKUP phase — dashboard showing probability breakdown + lime Wyckoff row
2. 3H chart with E:MARKDOWN phase — chart label "W:E↓" visible + dashboard
3. 15m chart with overlapping Wyckoff phase conditions — dashboard phase string
4. 3H chart with HTF bearish + bullish chart signals — full dashboard for Bug #10 documentation

---

### Instrument 4: ETH/USDT

| Test ID | Timeframe | Scenario | What to Verify | Entry Types Tested | Code Path / Fix |
|---------|-----------|----------|---------------|-------------------|----------------|
| 4A | 30m | **Range fade long at support** | Price at range low, FADE LONG fires. Verify range boundaries drawn (i_show_range), probability uses range threshold (0.40). Check conviction spread met. | FADE LONG | Range fade path, i_range_prob |
| 4B | 30m | **Range fade short at resistance** | Price at range high, FADE SHORT fires. Verify symmetric to 4A. Check ADX < 18 (range regime). | FADE SHORT | Range fade path, ADX range threshold |
| 4C | 5m | **Reversal exit in POSITIONED** | Active trend trade, reversal signal fires against position (BOS opposite direction + CVD flip). Verify rev_exit_pos fires, exit recorded as win. | TREND → rev_exit | Exit chain: rev_exit_pos path |
| 4D | 4H | **Range invalidation exit** | Active range fade trade, range dissolves (trend confirmed via ADX crossover). Verify range_invalid fires, exit at close. Verify separate from stopped branch. | FADE → range_invalid | Fix #26 (separated struct/range branch) |
| 4E | 5m | **Stalking mode → entry conversion** | trade_state 6 (STALKING) activates when conditions nearly met. Verify STALK label appears, then converts to full entry when micro catalyst fires. | Any (via STALK) | Stalking path, i_stalk_enabled |

**Screenshots needed (ETH):**
1. 30m chart with FADE LONG at range bottom — range boundaries visible + dashboard
2. 30m chart with FADE SHORT at range top — ADX row showing < 18
3. 5m chart with reversal exit — exit label + R counter
4. 4H chart with range_invalid — dissolved range visible + exit at close price
5. 5m chart with STALK label → entry conversion sequence

---

### Instrument 5: SOL/USDT

| Test ID | Timeframe | Scenario | What to Verify | Entry Types Tested | Code Path / Fix |
|---------|-----------|----------|---------------|-------------------|----------------|
| 5A | 3m | **Wide-range bar: stop wins heuristic** | Find bar where low ≤ stop AND high ≥ TP1, AND open is closer to stop. Verify eff_stopped = true, exit at stop_price (-1R). Heuristic correctly gives stop priority. | Any active trade | Fix #26 (stop wins collision) |
| 5B | 3m | **Wide-range bar: TP wins heuristic** | Same collision scenario but open closer to TP1. Verify eff_stopped = false, trade transitions to MANAGING at BE. Exit at 0R (breakeven), not -1R. | Any active trade | Fix #26 (TP wins collision) |
| 5C | 1H | **Trend ride trailing in MANAGING** | After TP1 hit, MANAGING state with trailing stop. Verify trail adjusts on swing structure. Stop label shows "TR:" when above entry. TP2 hit exits at full target. | TREND → MANAGING | MANAGING trail path, Fix #27 (TR: label) |
| 5D | 1D | **Regime transition: range → trend** | ADX crosses from < 18 to > 25. Verify mode transitions, active range trade handles transition correctly (range_invalid or continuation). Dashboard ADX row updates. | FADE → regime flip | ADX hysteresis, i_adx_range/i_adx_trend |
| 5E | 1H | **Re-entry watch after win** | Trade exits as win (TP2 or OBV exit). trade_state → 5 (REENTRY_WATCH). Verify cooldown respects i_cooldown bars. If structure holds, new entry fires. | Any → RE-ENTRY | Re-entry path, i_cooldown |

**Screenshots needed (SOL):**
1. 3m chart with stop/TP collision — annotate open position relative to stop/TP levels
2. 1H chart in MANAGING state — dashboard showing TR: label + trailing stop movement
3. 1D chart with regime transition — ADX row showing crossover + mode change
4. 1H chart showing RE-ENTRY signal after prior win — cooldown period visible

---

## Known Bugs to Watch For

During all tests, document any occurrence of these known issues:

| Bug # | Description | What to Capture | Priority |
|-------|-------------|----------------|----------|
| **#5** | `abs_loaded_bear` ignores conflicting `abs_higher_lows` | Screenshot when absorption bear loads despite higher lows forming. Note instrument/TF/bar. | Medium |
| **#10** | Symmetric scoring for directional signals | Screenshot probability breakdown when vol expansion adds equally to B% and S% during confirmed directional phase. Note exact B%/S% values. | High |
| **FIX-17** | EMA slope bypass for reversal signals (reverted) | Screenshot when reversal entry is blocked by EMA slope filter. Note if slope was against reversal direction. | Medium |
| **Dead code** | `i_cvd_lb` input unused after CVD refactor | Confirm input still appears in settings. Not a runtime bug — cleanup item. | Low |

---

## Test Execution Checklist

### Per Test Step

1. Load instrument + timeframe on TradingView
2. Apply Ghost Wick v5.8 indicator
3. Set config overrides (if any per test)
4. Scroll to find the target scenario (or wait for live signal)
5. Take screenshot with **full dashboard visible**
6. Annotate screenshot: circle the relevant dashboard row, label the entry/exit on chart
7. Record in results table below

### Screenshot Requirements

Every screenshot must include:
- Full chart with indicator overlays visible
- Dashboard table (all rows) — not cropped
- Bar timestamp visible (hover or crosshair)
- If comparing states: before/after or side-by-side

---

## Results Table (Template)

Copy this table for each completed test. Fill in after execution.

| Field | Value |
|-------|-------|
| **Test ID** | e.g., 1A |
| **Instrument** | e.g., PENGU/USDT |
| **Timeframe** | e.g., 3m |
| **Date/Time (UTC)** | |
| **Bar Timestamp** | |
| **Signal/Entry Type** | |
| **Dashboard Values** | E: / S:(or BE:/TR:) / T: |
| **Probability** | B:__% S:__% |
| **Wyckoff Phase** | |
| **ADX** | |
| **Trade State** | |
| **R Result** | |
| **Expected Behavior** | |
| **Actual Behavior** | |
| **Pass/Fail** | |
| **Bug # (if fail)** | |
| **Screenshot File** | |
| **Notes** | |

---

## Test Priority Order

Execute in this order to maximize early bug detection:

| Priority | Tests | Rationale |
|----------|-------|-----------|
| **P0 — Critical** | 1A, 1B, 3A, 5A, 5B | Validates core fixes #26, #27, #28. Failure here = regression. |
| **P1 — High** | 1D, 1E, 2A, 3B, 3E, 4D, 5C | Validates remaining fixes #25, #29 + key exit paths. Documents Bug #10. |
| **P2 — Medium** | 2B, 2C, 2D, 3C, 3D, 4A, 4B, 4C, 5D, 5E | Entry type coverage + edge cases. Important but less likely to reveal regressions. |
| **P3 — Low** | 1C, 2E, 4E | Thin mode, OBV exit, stalking — less common paths, lower regression risk. |

---

## Estimated Time Per Test

| Activity | Time |
|----------|------|
| Load chart + apply indicator + configure | 2 min |
| Scroll/find target scenario | 3–10 min (varies by signal frequency) |
| Screenshot + annotate | 2 min |
| Record results | 2 min |
| **Total per test** | **~10–15 min** |
| **Total all 25 tests** | **~4–6 hours** |

Recommend splitting across 2 sessions: P0+P1 (session 1, ~2.5h), P2+P3 (session 2, ~2.5h).

---

## Summary

- **5 instruments** covering low-price micro-cap, high-price benchmark, trending mid-cap, moderate-volatility large-cap, and high-volatility large-cap
- **3 timeframes each** = 15 chart configurations testing HTF auto-scaling
- **25 test scenarios** covering all 10 entry types, 7 exit paths, 4 recent fixes, and 4 known bugs
- **Standardized results table** for consistent documentation
- **Priority ordering** ensures critical fix validation first
