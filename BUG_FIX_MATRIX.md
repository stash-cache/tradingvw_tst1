# Ghost Wick — Bug / Fix Matrix

Continuing from the "Fix ghost wick trading quality issues" session. This is the
single source of truth for all identified quality issues in
`ghost_wick_v4.5_source.md`, their remediation status, performance evidence, and
follow-up work. New entries get appended below; existing entries are updated in
place.

Legend
- **Status**: `APPLIED` (merged into shipping code) · `REVERTED` (rolled back
  after regression) · `REDESIGN` (queued, requires different approach) ·
  `PENDING` (identified, not yet implemented) · `OPEN` (new candidate under
  review) · `WONTFIX` (intentional trade-off).
- **Severity**: `CRIT` (silently corrupts signals / P&L) · `HIGH` (skews
  probability or state transitions) · `MED` (affects edge cases, readability,
  or tooling) · `LOW` (cosmetic / non-trivial but bounded).
- **Source-of-truth column** names the file + line anchor in
  `ghost_wick_v4.5_source.md` so the matrix stays greppable.

---

## 1. Applied fixes — v4.0 foundational batch

| ID | Area | Fix | Status | Severity | Evidence / Source |
|----|------|-----|--------|----------|-------------------|
| FIX-1 | CVD engine | Clamped body/range ratio + `ta.rma(_, 3)` smoothing — removes doji/gap outliers from delta | APPLIED v4.0 | HIGH | `ghost_wick_v4.5_source.md:488-492` |
| FIX-2 | ADX regime | `ta.ema(adx, 3)` smoothing + 18/25 hysteresis band + `adx_regime_state` latch | APPLIED v4.0 | HIGH | `ghost_wick_v4.5_source.md:359-371` |
| FIX-3 | HTF bias | 2-bar close confirmation + neutral state; removes single-spike flips | APPLIED v4.0 | HIGH | `ghost_wick_v4.5_source.md:565-592`, weight penalty at `1078-1079` |
| FIX-4 | Probability | ADX-regime-interpolated weight matrix (`w_htf_base`, `w_cvd_base`, `w_struct`, `w_liq` scale with `_adx_norm`) | APPLIED v4.0 | HIGH | `ghost_wick_v4.5_source.md:1069-1076` |
| FIX-5 | OBV gate | ROC(3)+EMA(5) replaces SMA(14); 64% lag reduction; adds divergence guard | APPLIED v4.0 | HIGH | `ghost_wick_v4.5_source.md:472-478`, gate at `1248-1250` |
| FIX-6 | EQH/EQL | NATR-scaled dynamic tolerance; correct sensitivity across vol environments | APPLIED v4.0 | MED | `ghost_wick_v4.5_source.md:797-802` |
| FIX-7 | Absorption range | Confirmed pivot anchors replace rolling highest/lowest; anti-repaint | APPLIED v4.0 | HIGH | `ghost_wick_v4.5_source.md:842-853` |
| FIX-8 | Spring precision | Range-proximity filter (< 0.5 ATR of range low); false springs ~40% → <8% | APPLIED v4.0 | HIGH | `ghost_wick_v4.5_source.md:906-911` |
| FIX-9 | FVG | 3-bar + 2-bar body imbalance detection; stale reset on full fill | APPLIED v4.0 | MED | `ghost_wick_v4.5_source.md:713-745` |

## 2. Applied fixes — incremental releases

| ID | Ver | Area | Fix | Status | Severity | Evidence |
|----|-----|------|-----|--------|----------|----------|
| FIX-10 | v4.1 | Thin-mode execution | Retest gate as secondary path at `playbook_level >= 2` (micro stays primary) | APPLIED | MED | `ghost_wick_v4.5_source.md:1652-1709` |
| FIX-11 | v4.2 | Probability threshold | Dynamic `regime_thresh = i_trend_prob + (1 - _adx_norm) * (i_prob_thresh - i_trend_prob)` — matches FIX-4 scaling | APPLIED | HIGH | `ghost_wick_v4.5_source.md:1295-1308` |
| FIX-12 | v4.3 | State machine | Direct displacement retest from SCANNING(0) → POSITIONED(2), gated by BOS invalidation, prob, RR | APPLIED | HIGH | `ghost_wick_v4.5_source.md:1523-1582` |
| FIX-14 | v4.4 | Reversal detection | `rev_bull`/`rev_bear` composite (near level + RVOL + sweep/wick reject) feeding micro-trigger, WATCH re-entry, stalking, scoring | APPLIED | HIGH | `ghost_wick_v4.5_source.md:826-836`, micro `1311-1313`, re-entry `1997-2005`, scoring `1187-1189 / 1237-1239` |

## 3. Reverted / redesign queue

| ID | Ver | Area | Attempted fix | Status | Severity | Evidence |
|----|-----|------|---------------|--------|----------|----------|
| FIX-17 | v4.5 | LOADED reversal gate | EMA slope bypass for `rev_bull`/`rev_bear` in LOADED | REVERTED (needs redesign) | CRIT | PENGU 30m: 62W/26L +8.5R (v4.4) → 99W/101L -45.4R (v4.5 broken). `ghost_wick_v4.5_source.md:5-15`, `33-41` |

**Redesign brief for FIX-17** — require higher-confluence confirmation before
bypassing `ema8_rising`/`ema8_falling`:

- Option A: `rev_bull AND cvd_lean_bull AND rsi_reg_bull_ctx` — triple-stack
  confluence (level + volume + lagging momentum flip).
- Option B: `rev_bull AND bos_bull` — structural confirmation the turn is real.
- Option C: Keep EMA slope gate but soften to `ema8_slope >= 0` (flat or
  rising) when `rev_bull_ctx` is active within the last 3 bars.

Preferred path: Option A (keeps the lagging-indicator lesson intact while
recognising that the reversal fingerprint plus CVD plus a pivot-anchored
RSI div is a very different beast from a bare wick-rejection bounce).

## 4. Numbering gaps — candidates carried over from session 016dNFW5

The FIX-13, FIX-15, FIX-16 slots are empty in the shipped source. Assigning
the candidates identified in section 5 to fill them (subject to user confirm):

| Slot | Candidate | Severity | Notes |
|------|-----------|----------|-------|
| FIX-13 | Double-counted fresh S/D zone weight in probability | MED | Easy one-line fix; see BUG-A. |
| FIX-15 | Overlapping `abs_htf_bull` / `abs_htf_bear` thresholds | HIGH | Absorption mode can show simultaneous bull+bear HTF bias; see BUG-B. |
| FIX-16 | `adx_confirmed_recntly` typo and dead `adx_confirmed_recently` | MED | Silent miswire of `trend_ride_eligible`; see BUG-C. |

## 5. New candidates identified this pass

### BUG-A — Fresh S/D zone weight double-counted
- **Where**: `ghost_wick_v4.5_source.md:1167-1168` and `1183-1184` (bull),
  `1217-1218` and `1233-1234` (bear).
- **Symptom**: `demand_zone_fresh` adds `+0.06` once and `+0.04` again within
  the same bull scoring block (same for `supply_zone_fresh` on bear side). A
  fresh zone therefore contributes `0.10` instead of the intended `0.06`
  (or `0.04`), pushing prob over `perf_thresh` on otherwise borderline bars.
- **Severity**: MED (skews entries toward fresh-zone retests in trending
  regimes; effect is directional so it tilts performance rather than
  cancelling).
- **Proposed fix**: Delete the second occurrence in each branch — the first
  one is closer to the other S/D related weights and reads consistently.
- **Status**: OPEN.

### BUG-B — Absorption HTF bias thresholds overlap
- **Where**: `ghost_wick_v4.5_source.md:612-613`.
  ```pinescript
  bool abs_htf_bull = htf_range_pos > 0.45
  bool abs_htf_bear = htf_range_pos < 0.55
  ```
- **Symptom**: For `htf_range_pos ∈ (0.45, 0.55)` both booleans are true.
  `eff_htf_bull_ok` and `eff_htf_bear_ok` can then both be `true` in
  absorption mode, letting the probability scorer add `w_htf` to both sides
  and enabling both `abs_loaded_bull` and `abs_loaded_bear` to fire near the
  mid-range — wasted LOADED states and indecisive direction.
- **Severity**: HIGH (two-way bias in a mode that already suffers from
  whipsaw).
- **Proposed fix**: Use a dead-band split matching the comment at line 611:
  ```pinescript
  bool abs_htf_bull = htf_range_pos > 0.55
  bool abs_htf_bear = htf_range_pos < 0.45
  // 0.45-0.55 stays neutral (neither side loaded)
  ```
- **Status**: OPEN.

### BUG-C — `adx_confirmed_recntly` typo + dead code
- **Where**: Declared correctly at `ghost_wick_v4.5_source.md:814` as
  `adx_confirmed_recently`, then shadowed by the typo
  `adx_confirmed_recntly` at `1016` which is the one actually consumed by
  `trend_ride_eligible` on line `1017`.
- **Symptom**: The `:=` on line 1016 redeclares a new local variable with a
  name that silently passes Pine's compiler; the original
  `adx_confirmed_recently` becomes dead code. Nothing is broken at runtime
  because the typo'd version is computed from the same expression, but
  any future edit to `adx_confirmed_recently` will have no effect and will
  mislead debuggers.
- **Severity**: MED (latent foot-gun; currently benign).
- **Proposed fix**: Delete the typo'd reassignment at line 1016 and consume
  `adx_confirmed_recently` directly on line 1017.
- **Status**: OPEN.

### BUG-D — Absorption scoring adds `abs_supply_depleting` to both sides
- **Where**: `ghost_wick_v4.5_source.md:1097-1102`.
  ```pinescript
  if abs_supply_depleting
      bp += 0.15
      sp += 0.15
  if abs_supply_depleting and abs_range_tightening
      bp += 0.10
      sp += 0.10
  ```
- **Symptom**: "Supply depleting" is a bullish accumulation signal. Adding
  `+0.25` to `sp` (short prob) as well as `bp` is either a naming bug
  (should be a generic "compression" concept) or a directional bug
  (bearish entries should not benefit). In absorption mode the scorer then
  commonly produces near-equal `bp`/`sp` and fails `conviction_ok`,
  suppressing valid long entries.
- **Severity**: HIGH (symmetric weight nullifies directional edge in
  the mode it is designed for).
- **Proposed fix**: Either (a) rename to `compression_ok` and keep the
  symmetry, or (b) restrict the demand-side weight to bull and add an
  opposing `abs_supply_building` for bears. (a) is lower-risk and matches
  the Wyckoff Phase B/C interpretation the rest of the scorer already uses.
- **Status**: OPEN. Needs a regression run on instruments where absorption
  mode auto-engages (BTC daily, ETH daily) before shipping.

### BUG-E — `dormant_rec` blind spot for non-thin L1/phase 1
- **Where**: `ghost_wick_v4.5_source.md:2107`.
  ```pinescript
  dormant_rec := playbook_level == 1 and total_r < -5.0 and (thin_asset or l1_phase == 2)
  ```
- **Symptom**: A non-thin asset sitting in L1 phase 1 with `total_r < -5R`
  never flips to DORMANT, so the "No edge — consider switching instrument"
  NEXT hint at line 2630 never appears. The user only sees the STREAK label
  (line 2595) which resets with wins.
- **Severity**: MED (UX, not P&L — but the purpose of `dormant_rec` is to
  surface dead instruments, and this branch leaves them invisible).
- **Proposed fix**: Drop the phase guard or make it progressive:
  ```pinescript
  dormant_rec := playbook_level == 1 and total_r < -5.0 and (level_trades + level_wins) >= 8
  ```
- **Status**: OPEN.

### BUG-F — `rev_bull_ctx` fixed weight ignores the FIX-17 lesson
- **Where**: `ghost_wick_v4.5_source.md:1187-1189`, `1237-1239`.
- **Symptom**: The FIX-17 revert proved that the reversal fingerprint alone
  is too noisy to be trusted in active downtrends. The probability scorer
  still grants `rev_bull_ctx` a flat `+0.08` regardless of slope or
  counter-trend context. In an active downtrend with `ema8_falling` the
  scorer effectively rewards the same pattern the revert was designed to
  reject.
- **Severity**: HIGH (directionally correlated with the exact failure
  mode FIX-17 exposed).
- **Proposed fix**: Make the weight conditional on slope / BOS alignment:
  ```pinescript
  if rev_bull_ctx and (ema8_rising or bos_bull or cvd_lean_bull)
      bp += 0.08
  else if rev_bull_ctx
      bp += 0.03
  ```
  Same shape on the bear side with opposite signals.
- **Status**: OPEN — pair this with the FIX-17 redesign; both are about
  trusting the reversal fingerprint only with confluence.

### BUG-G — CVD sigma reset uses updated-bar stats
- **Where**: `ghost_wick_v4.5_source.md:512-519`.
- **Symptom**: `cvd_mean` and `cvd_stdev` are computed from `session_cvd`
  before the current-bar update (Pine evaluates the SMA using values up to
  and including the prior bar, since `session_cvd` is `var`). In
  practice this means the 3-sigma test compares the *new* `session_cvd`
  against stats that already include it, biasing toward "looks normal".
- **Severity**: MED (the reset rarely fires; but when it should — on a
  genuine regime break — the bias pushes it past the threshold bar
  instead of reacting promptly).
- **Proposed fix**: Snapshot into a prior value before updating:
  ```pinescript
  float scvd_prev = session_cvd[1]
  float cvd_mean = ta.sma(scvd_prev, 20)
  float cvd_stdev = ta.stdev(scvd_prev, 20)
  ```
- **Status**: OPEN. Low priority unless regression testing flags it.

## 6. Cross-cutting tech debt (tracked, not scheduled)

| ID | Area | Note |
|----|------|-----|
| TD-1 | `entry_class` / `l1_phase` | Two-axis state machine is under-documented; lines `2056-2105` mix playbook gating with phase progression. Extract into a dedicated section comment. |
| TD-2 | `htf_bull_cnt` reset | Resets to 0 on a single bearish close (`htf_bull_cnt := ... ? ... + 1 : 0`). Consider adding a 1-bar tolerance to survive wick rejections at the HTF EMA. |
| TD-3 | Kill-zone auto-off | `kz_auto_off` silently mutates `effective_kz` at line 246-248; dashboard does not reflect that the user-set filter was ignored. |

## 7. Next actions

1. User to confirm FIX-13/15/16 slot assignments in section 4.
2. Ship BUG-B (HTF threshold overlap) and BUG-C (typo) first — both are
   low-risk, high-clarity wins with no expected perf regression.
3. Backtest BUG-A and BUG-D together on PENGU/TRX/BTC 30m and daily
   before merging — they both touch probability weights and should be
   measured jointly to isolate side effects.
4. Revisit FIX-17 and BUG-F as one work-item: the reversal fingerprint
   needs a unified confluence policy, not two disjoint weight paths.
