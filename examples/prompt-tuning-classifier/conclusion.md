# Conclusion — prompt-tuning-classifier (synthetic demo)

> *Synthetic demo. Shape and per-slice gotcha modeled after a real
> classifier-prompt decision; numbers are illustrative.*

## Question (verbatim from Phase 0)

> Does a cross-source prompt outperform the baseline classifier on a B2B SaaS
> transaction-categorization task?

## Arms (each changes one thing vs baseline)

| Arm | Change | Cost / 1k rows |
|---|---|---|
| `prompt-v1` (baseline) | Production classifier as deployed (`classify/processor.ts:42`). | **$1.20** |
| `prompt-v2` | Baseline + cross-source signal block in system prompt. | **$1.80** |
| `prompt-v3` | prompt-v2 + amount-direction hint for transfer disambiguation. | **$1.90** |

## Metric + threshold (pre-registered Phase 2)

- Primary: classification accuracy vs human-labeled ground truth (judge: Claude Sonnet 4.6, rubric locked).
- Ship-rule: ALL of (a) aggregate effect >= **+5pp**, (b) no per-slice CI lower bound below **-2pp**, (c) no regression on the highest-severity failure category `transfer_as_revenue`.
- Failure-mode categories (locked with rubric): `wrong_code` (weight 1.0), `wrong_sign` (2.0), `transfer_as_revenue` (3.0). Each failed row tagged by the judge with one or more.
- Severity weighting: `abs(transaction_amount_usd)` capped at $50k. Both unweighted and severity-weighted aggregates reported below.

## Phase 3 — dev-set scores

![Arm comparison: mean accuracy with variance error bars across 3 trials](./charts/arm-bar.png)

All three arms beat baseline on aggregate by ~8.9pp (std across 3 trials: ~1.4pp). Both `v2` and `v3` are flagged as candidates for the held-out test pass.

## Phase 4 — diagnostic

Per-row diff inspection on dev surfaced one signal worth flagging into Phase 0 of any follow-up: `v3`'s amount-direction hint helped Enterprise rows (which carry rich amount context) but appears to **over-correct** SMB rows where the amount column is sparse or zero-filled. The iter-1 hypothesis ("amount hints universally help") was **falsified by iter-3**. The sprint completed in 3 iterations; the generalization gate ran (amount-direction hint qualified as a "universal mechanism" rather than a hardcode, so the change stayed in the arm); dev signal saturated at the slice level — arms were locked and we proceeded to test rather than start a second sprint.

## Phase 5 — verdict (one pass on held-out test set)

![Effect size forest plot, aggregate and per-slice with 95% CIs](./charts/forest-plot.png)

| Arm | Aggregate Δ | CI | Enterprise Δ | SMB Δ | Verdict |
|---|---|---|---|---|---|
| `prompt-v2` | **+8.9pp** | [+5.0, +12.8] | +13.4pp | +6.7pp | **ship** ✓ |
| `prompt-v3` | +8.9pp | [+5.0, +12.8] | +20.0pp | **-3.3pp** | **kill** ✗ |

### Failure-mode delta (per category — count of mis-tagged rows on N=60 test set)

| Arm | `wrong_code` (w=1.0) | `wrong_sign` (w=2.0) | `transfer_as_revenue` (w=3.0) | Total |
|---|---|---|---|---|
| `prompt-v1` (baseline) | 14 | 5 | 4 | 23 |
| `prompt-v2`            | 12 (Δ -2) | 4 (Δ -1) | **2 (Δ -2)** ✓ | 18 |
| `prompt-v3`            |  8 (Δ -6) | 4 (Δ -1) | **6 (Δ +2)** ✗ | 18 |

`v2` and `v3` tie on aggregate failure count (18 vs 18), but the **distribution is qualitatively different**: `v3` trades 6 fewer `wrong_code` errors (weight 1.0) for 2 MORE `transfer_as_revenue` errors (weight 3.0) — the highest-severity category. By the per-failure-mode rule pre-registered in Phase 0, this is a regression dressed as a tie.

### Severity-weighted aggregate

| Arm | Unweighted accuracy | Severity-weighted accuracy | Disagreement? |
|---|---|---|---|
| `prompt-v1` (baseline) | 0.617 | 0.581 | — |
| `prompt-v2`            | **0.706** | **0.724** | no — v2 wins both |
| `prompt-v3`            | 0.706 | **0.572** | **yes — v3 ties v2 unweighted but is BELOW BASELINE weighted** |

The two rankings disagree on `v3`. Under the pre-registered tie-break (severity-weighted winner ships only if unweighted score also clears threshold; otherwise kill), `v3` is killed regardless of its per-slice result — the severity-weighted score is a fail on its own.

**Verdict: ship `prompt-v2`, kill `prompt-v3`.**

`v2` passes all three pre-registered rules cleanly: aggregate +8.9pp (CI lower bound lands exactly on the registered +5pp floor); per-slice — both CIs positive; per-failure-mode — `transfer_as_revenue` drops 4→2 and severity-weighted accuracy rises 0.581→0.724. `v3` passes aggregate just as cleanly, but fails TWO of the three rules: SMB slice CI `[-8.8pp, +2.2pp]` crosses both zero and the -2pp loss floor; `transfer_as_revenue` rises 4→6 in the highest-severity category, and severity-weighted accuracy at 0.572 is BELOW baseline 0.581. **Aggregate winners are not winners when a major slice regresses or when the failure mix shifts into the highest-severity category.** Without the per-slice + per-failure-mode + severity-weighted rules pre-registered in Phase 0, we would have shipped `v3` and quietly hurt every SMB customer AND inflated the top-line P&L on every founder dashboard.

## Cost view

![Cost vs accuracy Pareto across all three arms](./charts/cost-vs-accuracy.png)

`v2` is the Pareto-frontier move: +8.9pp aggregate at +$0.60 / 1k rows. `v3` adds +$0.10 / 1k for zero aggregate gain and a per-slice regression.

## What to test next

Investigate whether amount-direction hints can be made **slice-conditional** (apply only to Enterprise rows where the amount field is reliably populated). That is a separate Phase 0, not a v3 retry.

## Discipline self-audit

- [x] Test set sealed until Phase 5; opened ONCE
- [x] Pre-registered metric + threshold; no drift
- [x] Pilot N=1 validated all metric fields populated before full dev run
- [x] Distribution audit: per-slice counts match production traffic (50/50 Enterprise/SMB on this run; matches production)
- [x] Variance baseline measured: 3 same-prompt trials per arm, std ~1.4pp
- [x] Effect >= 2× variance (8.9pp vs ~2.8pp)
- [x] Cross-judge sanity: 5 rows checked with Gemini-2.5-Pro; agreed on ranking direction on 5/5
- [x] Blind judge (arm IDs anonymized per row)
- [x] Each iter changed ONE thing
- [x] Iter hypotheses written in advance; iter-3 falsification accepted as a finished iteration
- [x] Sprint cap (3 iter / sprint) enforced
- [x] Generalization gate run at sprint end; amount-direction hint classified as universal mechanism (kept), not hardcode (would have been deleted)
- [x] Single sprint sufficed (dev signal saturated; second sprint would have been metric-chasing)
- [x] Verdict locks LATEST gate-passed hypothesis-driven iter (`prompt-v3` was locked for test even though its dev score wasn't higher than `prompt-v2`'s)
- [x] Per-slice scores reported; aggregate winner `v3` overturned by per-slice rule
- [x] Failure-mode categories pre-registered in Phase 2; per-category delta surfaced `v3`'s severity-3.0 regression
- [x] Severity-weighted aggregate reported alongside unweighted; the two rankings disagreed on `v3`, resolved per pre-registered tie-break
