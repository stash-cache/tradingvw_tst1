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

### BUG-H — Asymmetric HTF scoring (carried forward as Bug #10)
- **Where**: `ghost_wick_v4.5_source.md:594-604` (bias resolution),
  `1141-1144` / `1191-1194` (probability weighting).
- **Symptom**: `htf_bull_ok` / `htf_bear_ok` collapse Daily + 4H into a
  single boolean via `i_htf_strict` (AND) or 4H-only (implicit OR with the
  Daily bias ignored). `eff_htf_bull_ok` then contributes a full `w_htf`
  (and a further `+0.05` off-kill-zone bonus) to `bp` independently of
  whether the bear side is also adding `w_htf` via the same mechanism.
  On charts where D / 4H / 1H are *all bullish*, any short-bias pocket at
  one TF can still lift `sp` — bear weight should not accrue at all when
  the majority of TFs align bull.
- **Observed effect from the shared chart**: B:52-55% would fall to 43%
  under a majority-alignment weighting.
- **Severity**: HIGH (system-wide mis-weighting; depresses conviction
  spread and pushes valid bull entries below `i_conv_spread`).
- **Proposed fix**: Score HTF as a single majority-alignment vote, not
  per-TF independently. Sketch:
  ```pinescript
  int htf_align = (htfd_bias_bull?1:0) + (htf4h_bias_bull?1:0) + (ema_is_bull?1:0)
                - (htfd_bias_bear?1:0) - (htf4h_bias_bear?1:0) - (ema_is_bear?1:0)
  if htf_align >= 2
      bp += w_htf
  else if htf_align <= -2
      sp += w_htf
  // |htf_align| < 2 stays neutral — no weight to either side
  ```
  Keep `eff_htf_bull_ok` / `eff_htf_bear_ok` as gates for LOADED, but
  replace the weight call-sites with the majority vote.
- **Status**: OPEN. System-wide — requires regression across the full
  instrument panel. Pair with BUG-B (absorption HTF overlap) since both
  touch the "which side gets HTF weight" question.

### BUG-I — Promotion bypass after N bars of blocked entries
- **Where**: `ghost_wick_v4.5_source.md:2057-2083` (L1 promotion requires
  `level_trades >= 5`).
- **Symptom**: Bootstrap trap — if every entry path is blocked (kill zone,
  conviction, RR, prob-threshold, OBV gate, etc.), `level_trades` never
  increments and the system is pinned at L1 indefinitely. Instruments that
  need L2+ to unlock the retest / direct-scan paths (see FIX-10, FIX-12)
  never reach them, so the probationary L2 behavior is unreachable on
  quiet charts.
- **Severity**: MED (time-based escape; no direct P&L corruption but
  silently disables features that could rescue the chart).
- **Proposed fix**: Add a time-based auto-promotion with revocation on
  first loss:
  ```pinescript
  var int blocked_since = 0
  blocked_since := (trade_state == 0 and cooldown_active == false and level_trades == 0)
      ? blocked_since + 1 : blocked_since
  bool bootstrap_escape = playbook_level == 1 and blocked_since > eff_cooldown * 2
      and every_path_blocked  // scanner that ORs the rejection reasons
  if bootstrap_escape
      playbook_level := 2
      level_trades := 0
      level_wins := 0
      level_r_start := total_r
      // probationary flag — revert on first loss
      var bool probation = true
  // in loss-handling branch:
  if probation and exit_loss
      playbook_level := 1
      probation := false
  ```
  Key detail: `every_path_blocked` must be a composite that proves
  the blockage is structural (not just cooldown).
- **Status**: OPEN. Wire `every_path_blocked` first — it needs visibility
  into *why* each entry path declined to fire this bar.

### BUG-J — Post-loss CONT re-entry drops continuation context
- **Where**: `ghost_wick_v4.5_source.md:1849-1870` (POSITIONED stop-out
  path clears `is_cont_trade` and drops `trade_state := 0`),
  `1891-1894` (only the *win* path promotes to CONT state 4).
- **Symptom**: When a continuation trade stops out, the next bar has to
  re-qualify from SCANNING(0). Until ADX rises back above `i_adx_trend`
  and the next pullback forms, the system cannot re-attack the trend, so
  mid-trend shakeouts cost more than their nominal 1R — they also cost
  the *next* entry that would have ridden the continuation.
- **Severity**: HIGH (hands back trend equity on every stop, which is the
  dominant loss mode in trend regimes).
- **Proposed fix** (same shape as the 2H proposal from the prior session):
  on an `exit_loss` within trend context, route to `trade_state := 4`
  (CONT watch) instead of `0`, with the same timeout/invalidation guards
  that already exist for the TP2→4 promotion. Sketch:
  ```pinescript
  bool cont_eligible_after_loss = not is_range_trade and not is_abs_trade
      and trend_confirmed and playbook_level >= 3
      and ((trade_dir == 1 and eff_htf_bull_ok) or (trade_dir == -1 and eff_htf_bear_ok))
  // inside the exit_loss branch:
  if cont_eligible_after_loss
      trade_state := 4
      trade_dir := saved_dir_for_cont
  else
      trade_state := 0
  ```
- **Severity**: HIGH — already proposed for the 2H case in the prior
  session; the fix generalises once `adx_val` crosses `i_adx_trend` on
  any TF, so the same patch benefits this chart too.
- **Status**: OPEN. Ship together with FIX-17 redesign and BUG-F, since
  all three govern how reversal / continuation signals translate into
  re-entry after a loss.

### BUG-K — CONT LONG re-entry from state 0 after recent stop-out
- **Where**: `ghost_wick_v4.5_source.md:1484-1521` (CONT direct-entry
  from SCANNING requires a fresh `pb_pullback_bull` / `pb_pullback_bear`),
  `1849-1870` (stop-out clears all continuation context to state 0).
- **Symptom**: A CONT LONG that stops out within the last N bars while
  OBV + MACD + EMA are still bullish and price is back above the
  original entry's invalidation is treated as a full reset — the system
  waits for a *new* pullback rather than recognising the stop-out as a
  "false break" that confirms direction. On the reference chart this
  missed an 11% rally because the next pullback never formed.
- **Severity**: HIGH — directly responsible for a specific missed move;
  narrower than BUG-J but higher-confidence fix.
- **Proposed fix**: Add a dedicated state-0 path that fires when a
  recent CONT stop is "rejected" by the next bar(s):
  ```pinescript
  var int last_cont_stop_bar = -1
  var int last_cont_stop_dir = 0
  var float last_cont_invalid_lvl = na
  // in the CONT exit_loss branch, remember the context:
  if exit_loss and is_cont_trade
      last_cont_stop_bar := bar_index
      last_cont_stop_dir := saved_cont_dir
      last_cont_invalid_lvl := entry_price_at_stop

  // in state 0:
  bool cont_false_break_bull = last_cont_stop_dir == 1
      and bar_index - last_cont_stop_bar <= N
      and close > last_cont_invalid_lvl
      and obv_bull_robust and macd_bull_momentum and ema8_rising
  if cont_false_break_bull and trend_confirmed and eff_htf_bull_ok
      // direct state 0 → 2 entry without requiring pb_pullback_bull
  ```
- **Status**: OPEN. Pair with BUG-J (same anchor point: the `exit_loss`
  branch at line 1849). BUG-K handles the state-0 recovery;
  BUG-J handles the state-4 (CONT watch) recovery. They are complementary,
  not alternatives.

### BUG-L — Relax `tpb_rr >= 1.5` for CONT continuations after recent loss
- **Where**: `ghost_wick_v4.5_source.md:1490` (bull) and `1509` (bear).
- **Symptom**: The fixed `tpb_rr >= 1.5` check prices the target off
  `bsl` / `ssl` (current liquidity level). After a CONT stop-out in a
  continuing trend, the next pullback often sits so close to `bsl` that
  `tpb_rr` fails even though CVD + OBV confirm continuation. Entries
  that would clear `i_min_rr` against a further-out target are rejected
  because the target is pinned to BSL.
- **Severity**: MED — addresses the BSL target invalidation edge case;
  compounds with BUG-K on the same post-loss bars.
- **Proposed fix**: When CVD/OBV confirm continuation, extend the target
  past BSL/SSL using an adaptive multiplier, mirroring the approach
  proposed as FIX-19 for displacement:
  ```pinescript
  float tpb_tgt_adaptive = trade_dir == 1
      ? (cvd_lean_bull and obv_bull_robust ? bsl + adaptive_atr * 1.2 : bsl)
      : (cvd_lean_bear and obv_bear_robust ? ssl - adaptive_atr * 1.2 : ssl)
  ```
  Keep the `>= 1.5` floor; the adaptive target lifts the numerator so
  genuine continuations clear it. Use a slightly lower floor
  (e.g. `>= 1.3`) only when `(cvd_lean_* and obv_*_robust)` both hold.
- **Status**: OPEN. Depends on FIX-19 being defined first (the
  "adaptive target like FIX-19 does for displacement" reference —
  FIX-19 is not yet in this matrix or source; add it as a PENDING slot
  once the displacement adaptive-target spec lands).

### BUG-M — RE-ENTRY watch (state 5) never fires post-loss
- **Where**: `ghost_wick_v4.5_source.md:1842-1847` and `1926-1930`
  (only the `obv_flow_exit` / `exit_win` branches route into state 5;
  the `exit_loss` branch at `1849-1870` goes to state 0).
- **Symptom**: State 5 is the system's re-entry watch, but the only
  transitions into it come from winning exits. After a loss with the
  bull signal stack still intact (OBV ✓, MACD↑, EMA↑), the re-entry
  infrastructure — `disp_retest_bull`, `micro_bull_gated`, `rev_bull`
  at lines 1984-2005 — is never armed, so legitimate re-entries go
  uncaught.
- **Severity**: HIGH — directly responsible for missed re-entries on
  charts where the first entry stops but the trend resumes.
- **Proposed fix**: Add a post-loss arming path to state 5 with a
  relaxed pullback definition:
  ```pinescript
  bool post_loss_stack_bull = trade_dir == 1 and exit_loss
      and obv_bull_robust and macd_bull_momentum and ema8_rising
      and eff_htf_bull_ok
  bool post_loss_stack_bear = trade_dir == -1 and exit_loss
      and obv_bear_robust and macd_bear_momentum and ema8_falling
      and eff_htf_bear_ok
  if post_loss_stack_bull or post_loss_stack_bear
      reentry_dir := post_loss_stack_bull ? 1 : -1
      reentry_bar := bar_index
      trade_state := 5
  else
      trade_state := 0
  ```
  Inside state 5, relax the pullback requirement: allow any of
  `disp_retest_*`, `micro_*_gated`, `rev_*`, or a simple
  `close > entry_price_before_stop` confirmation bar to arm the entry.
- **Status**: OPEN. BUG-K, BUG-J, and BUG-M are three recovery-path
  variants:
  - BUG-K: state-0 direct entry on "false break" confirmation.
  - BUG-J: state-4 (CONT watch) promotion on trend-context stop.
  - BUG-M: state-5 (re-entry watch) arming on stack-intact stop.
  They should be designed as a unified post-loss router rather than
  three independent patches — pick the target state from the context
  at exit time (range vs trend vs reversal) instead of layering three
  parallel branches into the `exit_loss` block.

### BUG-N — Thin-asset cooldown is too long on higher intraday TFs
- **Where**: `ghost_wick_v4.5_source.md:253` (`eff_cooldown` derivation),
  `187` (`i_cooldown` default of 5), `1293` (`cooldown_active` gate),
  `2584` (dashboard).
- **Symptom**: `i_cooldown` defaults to 5 bars. On 2H that is 10 hours
  between trades; on thin/volatile assets 2-3 bars is usually enough
  for the book to reset. The current scaling only halves the cooldown
  on daily-plus TFs (`is_daily_plus`), not for thin-asset mode on
  intraday — so thin 2H charts inherit the default.
- **Severity**: LOW — settings tuning, not a correctness bug. Didn't
  block in the referenced case because enough time had passed, but
  generally relevant and will block on shorter gaps between stops.
- **Proposed change**: Add a thin-mode branch to the `eff_cooldown`
  derivation (still at line 253):
  ```pinescript
  int eff_cooldown = is_daily_plus
      ? math.max(math.round(i_cooldown / 2), 2)
      : thin_asset
          ? math.max(math.round(i_cooldown / 2), 2)  // 2-3 bars
          : i_cooldown
  ```
  Alternative: expose a dedicated `i_thin_cooldown` input so the user
  can tune it without affecting default-mode instruments.
- **Status**: OPEN. Lowest priority — ship with the next routine
  settings-tuning pass rather than as its own patch.

### BUG-O — DISP LONG gate blocks L3 + recent-loss displacement entries
- **Where**: `ghost_wick_v4.5_source.md:1529` (FIX-12 direct displacement
  retest from SCANNING requires `playbook_level >= 2`).
- **Symptom**: At `playbook_level == 3` with a recent stop-out, a clean
  displacement breakout still has to pass the `playbook_level >= 2`
  gate. The gate was originally meant to hold L1 instruments behind
  the retest infrastructure until they prove out — but at L3 the
  instrument has already proven out, and on a recent-loss bar the
  system should *prefer* the displacement path (higher conviction,
  clearer invalidation) over waiting for another pullback.
- **Severity**: MED — opens the displacement path during post-loss
  scenarios where it is most valuable. Compounds with BUG-K / BUG-M
  on the same bars.
- **Proposed fix**: Split the gate so L3 + recent-loss bypasses the
  `playbook_level >= 2` requirement:
  ```pinescript
  bool disp_gate_ok = playbook_level >= 2
      or (playbook_level >= 3
          and bar_index - last_exit_bar <= eff_cooldown * 2
          and last_trade_r < 0)
  if trade_state == 0 and bar_confirmed and not cooldown_active
      and not stop_too_tight and disp_gate_ok and not absorption_mode
      // FIX-12 direct displacement retest block
  ```
  Note: this interacts with BUG-I (bootstrap promotion) — if BUG-I
  auto-promotes to L2 on a blocked L1, the L3-recent-loss bypass here
  remains separate because it is a per-signal relaxation at L3, not a
  level promotion.
- **Status**: OPEN. Pair with the post-loss router (BUG-J/K/M) so the
  displacement bypass and the state-0/4/5 routing are reasoned about
  together — both are about "what the system does on the bar after a
  stop when context is still good."

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
5. BUG-H (HTF majority alignment) is system-wide — stage it behind a
   feature flag (`i_htf_majority = input.bool(false, ...)`) so a full
   instrument-panel regression can be run before flipping the default.
6. BUG-I (promotion bypass) depends on a new `every_path_blocked`
   scanner — build that first; it is also diagnostically useful on its
   own and should surface in the dashboard row 9 "Guards" string.
7. BUG-J (post-loss CONT re-entry) lands together with FIX-17 redesign
   and BUG-F — all three are about how the system recovers directional
   context after a counter-trend hit.
8. BUG-K / BUG-L / BUG-M are the three post-loss recovery levers
   (state-0 false-break, relaxed `tpb_rr` with adaptive target, state-5
   re-entry arming). Design them together as a single post-loss router
   inside the `exit_loss` branch rather than three parallel patches —
   the router picks state-0 / state-4 / state-5 from the context at
   exit time (trend regime, stack intactness, invalidation reclaim).
9. FIX-19 (adaptive displacement target) is referenced by BUG-L but
   not yet specified in this matrix or source. Add a PENDING FIX-19
   entry as soon as the spec lands — BUG-L depends on it.
10. BUG-O (L3 + recent-loss displacement bypass) lands with the
    post-loss router work from action 8. The bypass is a per-signal
    relaxation at L3, not a level promotion, so it sits alongside the
    BUG-I bootstrap escape without conflicting.
11. BUG-N (thin-asset cooldown) defers to the next settings-tuning
    pass — low priority, no correctness impact.
