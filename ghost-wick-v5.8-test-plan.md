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

## Scenario Fallback Strategy

If the selected instrument does not have the target scenario available on the current chart (no active trade, no recent signal, or market conditions don't match), use these approaches **in order of preference**:

### Approach 1: Historical Scrollback

Scroll back on the **same instrument + timeframe** to find a past occurrence. Most scenarios have historical examples within 200–500 bars. Use TradingView's replay mode (bar replay button) to step through the scenario bar-by-bar if needed.

| Scenario Type | Typical Lookback | How to Spot It |
|--------------|-----------------|----------------|
| Stop/TP collision (1B, 5A, 5B) | 50–200 bars on 3m–30m | Wide-wick bars with both high > TP and low < stop visible. Look for long lower wicks on uptrends or long upper wicks on downtrends. |
| Range fade entries (4A, 4B) | 30–100 bars on 30m | Horizontal consolidation zones with "FADE LONG" / "FADE SHORT" plotshapes. Range boundaries drawn when i_show_range is ON. |
| Displacement entries (2A) | 20–80 bars on 5m | Large body candles with "DISP LONG" / "DISP SHORT" plotshapes. Cluster around news events or session opens. |
| Wyckoff Phase E (3A, 3B) | 100–300 bars on 3H | "W:E" (lime) or "W:E↓" (red) labels on chart. Occur after sustained accumulation/distribution. |
| Structural invalidation (2C, 4D) | 50–150 bars on 1H–4H | Trades that exit mid-bar without touching stop or TP levels. R counter shows fractional R (e.g., -0.4R). |
| Regime transition (5D) | 50–200 bars on 1D | ADX crossing from orange (< 18) to green (> 25) or vice versa in dashboard. Background color changes from ranging to trending. |
| Same-bar stop guard (1D) | 30–100 bars on 2D | Reversal entries where entry bar has a long wick past the stop level. Trade should survive (v5.8) — if it doesn't, that's a regression. |

### Approach 2: Substitute Instrument (Same Category)

If scrollback yields nothing within 500 bars, substitute with an instrument from the same category.

| Primary | Substitutes (in order) | Category Match |
|---------|----------------------|----------------|
| PENGU/USDT | FLOKI/USDT, BONK/USDT, SHIB/USDT | Low-price meme coin, 5-8 decimal mintick |
| BTC/USDT | BTC/USD (Coinbase), BTC/USDT.P (Bybit) | High-price, high-liquidity, 2-decimal |
| HYPER/USDT | ONDO/USDT, WIF/USDT, JUP/USDT | Mid-cap trending, absorption-prone |
| ETH/USDT | LINK/USDT, AVAX/USDT, AAVE/USDT | Large-cap moderate-volatility, range-prone |
| SOL/USDT | DOGE/USDT, NEAR/USDT, SUI/USDT | High-volatility large-cap, wide daily ranges |

### Approach 3: TradingView Screener — Find Asset That Fits

When both scrollback and substitutes fail, use TradingView's Crypto Screener to find an asset currently exhibiting the target conditions. Screener profiles below.

---

## TradingView Screener Profiles

Open TradingView → Screener → Crypto. Apply filters per scenario type. Sort by volume (descending) to prioritize liquid pairs.

### Profile S1: Low-Price Precision Testing (Tests 1A, 1E)

*Goal: Find coins where mintick has 5+ decimals to validate format.mintick.*

| Filter | Operator | Value |
|--------|----------|-------|
| Exchange | equals | BINANCE |
| Last Price | less than | 0.01 |
| Volume (24h, USD) | greater than | 5,000,000 |
| Change % | is not between | -1% to 1% *(needs movement)* |

**Sort by:** Volume 24h descending
**Pick:** Top 3 results. Apply Ghost Wick v5.8, verify dashboard shows 5+ decimal places.

### Profile S2: Wide-Range Bars / High Volatility (Tests 1B, 5A, 5B)

*Goal: Find assets with frequent wide-wick bars where stop+TP collision is likely.*

| Filter | Operator | Value |
|--------|----------|-------|
| Exchange | equals | BINANCE |
| Volatility (1W) | greater than | 8% |
| Average True Range (14) % | greater than | 4% |
| Volume (24h, USD) | greater than | 10,000,000 |

**Sort by:** ATR % descending
**Pick:** Top 3 results. Apply indicator on 3m–15m. Scroll for bars where high-low range > 2x typical body.

### Profile S3: Range-Bound / Low ADX (Tests 4A, 4B, 4D)

*Goal: Find assets in confirmed sideways consolidation for range fade testing.*

| Filter | Operator | Value |
|--------|----------|-------|
| Exchange | equals | BINANCE |
| ADX (14) | less than | 20 |
| Bollinger Bands Width (20) | less than | 5% |
| Volume (24h, USD) | greater than | 10,000,000 |
| Change % (1W) | is between | -5% to 5% |

**Sort by:** ADX ascending (lowest = most range-bound)
**Pick:** Top 3 results. Apply indicator on 30m. Look for range boundaries drawn + FADE signals at extremes.

### Profile S4: Strong Trend / Displacement (Tests 2A, 2D, 5C)

*Goal: Find assets in confirmed trend for displacement breakout and continuation testing.*

| Filter | Operator | Value |
|--------|----------|-------|
| Exchange | equals | BINANCE |
| ADX (14) | greater than | 25 |
| Relative Volume | greater than | 1.5 |
| Volume (24h, USD) | greater than | 20,000,000 |

**Sort by:** ADX descending (strongest trend first)
**Pick:** Top 3 results. Apply indicator on 5m–1H. Look for "DISP LONG/SHORT" or "CONT LONG/SHORT" plotshapes.

### Profile S5: Accumulation / BB Squeeze (Tests 3A, 3B, 3C, 3D)

*Goal: Find assets in Bollinger Band squeeze for Wyckoff phase and absorption testing.*

| Filter | Operator | Value |
|--------|----------|-------|
| Exchange | equals | BINANCE |
| Bollinger Bands Width (20) | less than | 4% |
| ADX (14) | less than | 22 |
| Volume (24h, USD) | greater than | 5,000,000 |
| Change % (1M) | is between | -15% to 15% |

**Sort by:** BB Width ascending (tightest squeeze first)
**Pick:** Top 3 results. Apply indicator with Mode Override = ABSORPTION. Look for "W:C", "W:D", "W:E" labels.

### Profile S6: Regime Transition (Test 5D)

*Goal: Find assets where ADX is near the range/trend boundary (transitioning).*

| Filter | Operator | Value |
|--------|----------|-------|
| Exchange | equals | BINANCE |
| ADX (14) | is between | 18 to 25 |
| Volume (24h, USD) | greater than | 10,000,000 |
| Change % (1D) | greater than | 3% OR less than -3% |

**Sort by:** Volume descending
**Pick:** Top 3 results. Apply indicator on 1D. The ADX is in the hysteresis zone — scroll back to find the crossover from < 18 to > 25.

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

**Fallbacks (PENGU):**

| Test ID | If Scenario Not Found | Alternative |
|---------|----------------------|-------------|
| 1A | No active FADE signal on 3m | **Scrollback** 200 bars on 3m — PENGU ranges frequently, FADE signals appear every 30–80 bars. **OR** any sub-$0.01 coin confirms precision — use Screener Profile S1 (FLOKI, BONK, SHIB). Precision test is instrument-agnostic; the key requirement is mintick ≥ 5 decimals. |
| 1B | No stop+TP collision in recent history | **Scrollback** 500 bars on 30m — collision requires extreme wick bar during active trade, rare (~1-2/week). **OR** switch to 3m (more bars, more wicks). **OR** use Screener Profile S2 for high-ATR% assets — SOL/USDT 3m is the best fallback (Test 5A/5B covers same path). |
| 1C | Thin mode doesn't change signal count | This is expected if market conditions already meet default spreads. **Reduce** `i_conv_spread` to 0.25 and `i_thin_conv` to 0.0 — this widens the gap. If still no difference, thin mode is working correctly (no conviction gate needed). Document the null result. |
| 1D | No reversal entry with wick past stop on 2D | **Scrollback** 300 bars — 2D bars have wide wicks. **OR** switch to 1D (more data points). **OR** use Screener Profile S2 and test on any high-volatility asset on daily TF. Reversal entries fire on sweep bars — look for "LONG" entries where the entry bar's low penetrates well below the subsequent stop level. |
| 1E | No trade reaching MANAGING state on 2D | **Scrollback** — TP1 hits are common. **OR** drop to 30m for faster signal cycle. Any instrument in MANAGING state works — the BE:/TR: label is purely display logic independent of asset. Use Screener Profile S4 for trending assets likely in active trades. |

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

**Fallbacks (BTC):**

| Test ID | If Scenario Not Found | Alternative |
|---------|----------------------|-------------|
| 2A | No DISP entry on 5m recently | **Scrollback** 300 bars — displacement fires around news/session opens (UTC 13:30 NY open, UTC 00:00 daily). **OR** switch to 3m (2x the data). **OR** use Screener Profile S4 — any asset with ADX > 25 and RVOL > 1.5 will produce DISP signals on 5m. ETH/USDT or SOL/USDT are reliable fallbacks. |
| 2B | Strict mode blocks all signals on 1H | This is a valid finding — strict mode is designed to reduce signals. **Document** the reduction count (signals with strict OFF vs ON over same 200-bar window). If zero signals, expand to 4H (larger bars = more HTF alignment opportunities). |
| 2C | No struct_invalid exit without TP collision | **Scrollback** 500 bars on 4H. Look for trades that exit mid-way (fractional R like -0.3R to -0.8R). **OR** test on 1H (faster signal cycle). **OR** use any large-cap asset — struct_invalid fires when BOS reverses against trade direction, common during ranging-to-trending transitions. Screener Profile S6 helps find transitioning assets. |
| 2D | No CONT entry on 1H | **Scrollback** 200 bars — CONT requires BOS + pullback + re-entry in trend direction. **OR** use Screener Profile S4 (strong trend assets). SOL/USDT and ETH/USDT in trending phases produce frequent CONT signals. Look for "CONT LONG" or "CONT SHORT" plotshapes. |
| 2E | No OBV exit during active trade | OBV exits require CVD/OBV divergence against position while trade is in profit. **Scrollback** 300 bars — OBV exits appear 1-3x/week on 5m. **OR** test on 15m. Any instrument works — the exit priority chain is identical across assets. |

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

**Fallbacks (HYPER):**

| Test ID | If Scenario Not Found | Alternative |
|---------|----------------------|-------------|
| 3A | No Phase E:MARKUP on 3H | Phase E requires `abs_price_breakout_long + abs_volume_expansion + adx_val > 20 + bb_expanding` — a rare high-conviction confluence. **Scrollback** 500 bars (3H = ~62 days). **OR** drop to 1H (4x more bars). **OR** use Screener Profile S5 to find a coin currently in BB squeeze that's breaking out — apply indicator with ABSORPTION mode, look for "W:E" label. ONDO/USDT and WIF/USDT are good candidates for trending mid-caps. **OR** relax to Phase D test (3C) which requires fewer conditions. |
| 3B | No Phase E:MARKDOWN on 3H | E_dist is the bearish mirror — requires `abs_price_breakout_short + abs_volume_expansion + adx_val > 20 + bb_expanding`. Same rarity as 3A but on sell-side. **Scrollback** 500 bars. **OR** use Screener Profile S4 filtered for **negative** Change% (trending down) — assets in confirmed downtrends are more likely to show E_dist phases. **OR** accept Phase D_dist (D:BREAKDOWN) as partial validation — it shares the scoring chain. |
| 3C | No overlapping Phase D + Phase C | Extremely rare edge case. **Acceptable to skip** if not found within 500 bars. The exclusive chain is validated structurally by code review — this test confirms the runtime behavior. **OR** test on 5m with ABSORPTION forced — faster phase cycling increases collision probability. |
| 3D | No spring entry on 1D | Springs require: confirmed range + wick below range low + close back inside. **Scrollback** 300 bars on 1D. **OR** drop to 4H. **OR** use Screener Profile S3 (range-bound, low ADX) — find an asset in confirmed range, wait for wick below support. LINK/USDT and AVAX/USDT range frequently on 4H–1D. |
| 3E | Phase E active but HTF is not bearish | This specific combination (bullish chart + bearish HTF) is the test target. **Scrollback** to find it — it occurred on HYPER 3H in the session where Bug #8 was identified. **OR** any mid-cap where chart TF shows markup but daily/weekly is still in downtrend. Common after local bottoms before HTF confirms. Document exact B%/S% values regardless of instrument. |

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

**Fallbacks (ETH):**

| Test ID | If Scenario Not Found | Alternative |
|---------|----------------------|-------------|
| 4A | No FADE LONG on 30m (ETH not ranging) | **Scrollback** 200 bars — ETH ranges frequently between support/resistance levels. **OR** use Screener Profile S3 (ADX < 20, tight BB) — any large-cap in consolidation will produce FADE signals. LINK/USDT, AVAX/USDT, DOT/USDT are reliable range-fade instruments. The key requirement is ADX < 18 so the indicator classifies as range regime. |
| 4B | No FADE SHORT on 30m | Same approach as 4A — range fades are symmetric. If 4A found a range, 4B exists on the opposite boundary. **Scrollback** to the top of the same range. If only one side fires (FADE LONG but no FADE SHORT), document the asymmetry — this may indicate a probability spread issue (Bug #10 adjacent). |
| 4C | No reversal exit on 5m | Reversal exits require BOS against the trade + CVD flip while in profit. **Scrollback** 500 bars on 5m. **OR** switch to 15m (reversal exits are more common on slightly larger TFs — more structural significance). **OR** use Screener Profile S6 (ADX 18–25, regime transition zone) — assets flipping regime produce frequent counter-BOS signals. |
| 4D | No range_invalid on 4H (no active range trade dissolving) | range_invalid fires when a range fade trade is active AND ADX crosses above trend threshold (range dissolves). **Scrollback** 300 bars. **OR** combine with Test 5D (regime transition) — same underlying mechanic viewed from different perspectives. Use Screener Profile S6 to find assets in the ADX transition zone. Any asset where range boundaries were drawn and then price broke out with ADX expansion qualifies. |
| 4E | No STALK signal on 5m | Stalking activates when conditions are *nearly* met — probability close to threshold but one micro catalyst missing. **Scrollback** 300 bars — STALK labels should appear near every entry signal (they precede entries by 1-5 bars). **OR** if stalking seems absent, verify `i_stalk_enabled = true`. **OR** any instrument on 5m–15m will show STALK labels — this path is instrument-agnostic. |

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

**Fallbacks (SOL):**

| Test ID | If Scenario Not Found | Alternative |
|---------|----------------------|-------------|
| 5A | No stop-wins collision on 3m | This requires: active trade + bar where open is closer to stop than TP + both levels breached. **Scrollback** 500 bars on 3m — SOL's volatility makes this the most likely asset. **OR** switch to 1m (even more wide-wick bars relative to trade ranges). **OR** use Screener Profile S2 (highest ATR%) — DOGE/USDT and WIF/USDT on 3m are high-wick-ratio alternatives. **OR** if no live collision found, test can be validated by comparing v5.7 vs v5.8 on the same historical bar — v5.7 always gives stop priority (-1R), v5.8 should show the heuristic result. |
| 5B | No TP-wins collision on 3m | Exact complement of 5A — same bar type but open closer to TP. **If 5A found a collision bar**, check bars in the same vicinity — both directions typically cluster near high-volatility events. **OR** use the same Screener Profile S2 approach. **OR** if only stop-wins collisions exist (open always near the stop), document that finding — it may indicate the stop level's proximity to typical opens is tighter than TP proximity, which is expected for tight stops. |
| 5C | No MANAGING trade with trailing stop on 1H | **Scrollback** 200 bars — TP1 hits transitioning to MANAGING are common on SOL 1H during trends. **OR** switch to 30m (faster cycle). **OR** use Screener Profile S4 (strong trend) — any trending asset in MANAGING state will show the TR: label. The trail mechanics are identical across instruments. Look for trades where the "LONG" entry was followed by upward price movement past TP1 level. |
| 5D | No regime transition on 1D recently | **Scrollback** 300 bars (1D = ~1 year). SOL transitions between range and trend 4-8x/year on daily. **OR** drop to 4H (transitions happen more frequently). **OR** use Screener Profile S6 (ADX 18–25) — assets currently in the transition zone are about to flip. Apply indicator and monitor for 2-5 bars. **OR** test on ETH/USDT 4H (Test 4D covers the same mechanic from the exit side). |
| 5E | No RE-ENTRY signal after win on 1H | RE-ENTRY requires: recent win (TP2 or OBV exit) + cooldown elapsed + structure still intact. **Scrollback** 300 bars — find any past WIN exit label, then check if RE-ENTRY appears 5+ bars later. **OR** switch to 30m (more trades = more re-entry opportunities). **OR** use Screener Profile S4 — strong trending assets produce sequential trades where re-entry is likely after each TP exit. |

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

### Fallback Decision Tree (Per Test)

```
1. Load primary instrument + TF
2. Is target scenario visible in last 50 bars?
   ├─ YES → Execute test, take screenshot
   └─ NO → Scrollback up to 500 bars
            ├─ FOUND → Use bar replay mode, execute test
            └─ NOT FOUND → Load substitute instrument (same category)
                           ├─ FOUND within 200 bars → Execute test, note substitute used
                           └─ NOT FOUND → Open TradingView Screener
                                          ├─ Apply matching Screener Profile (S1–S6)
                                          ├─ Pick top 3 results by volume
                                          ├─ Apply indicator to each
                                          └─ FOUND → Execute test, note screener-sourced asset
                                             NOT FOUND → Mark test as "DEFERRED — conditions unavailable"
                                                         Record: date, assets checked, screener results
                                                         Re-attempt in next session (market conditions change)
```

### Per Test Step

1. Load instrument + timeframe on TradingView
2. Apply Ghost Wick v5.8 indicator
3. Set config overrides (if any per test)
4. Check for target scenario in recent bars (last 50)
5. **If not found:** Follow fallback decision tree above
6. Execute test — take screenshot with **full dashboard visible**
7. Annotate screenshot: circle the relevant dashboard row, label the entry/exit on chart
8. Record in results table below (include substitute instrument if used)

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
| **Substitute Used?** | NO / YES → (asset name + reason) |
| **Screener Profile Used?** | NO / S1 / S2 / S3 / S4 / S5 / S6 |
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
| **Pass/Fail / DEFERRED** | |
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
