# Matching diagnostics report (9 checks, no CI)

This is the **matching-quality** report: the diagnostics evaluation dashboard restricted to the checks
that assess *how good the match is*, with the two confidence-interval checks removed. It answers "did the
matching work on this population," not "is the measured effect precise."

It is the same artifact `publish-diagnostics` produces (`{version}-diagnostics-evaluation.html` +
`scorecard-data.js`), built from the same `phase2_psm_diagnostics_{version}` /
`phase2_scorecard_metrics_{version}` tables, with a reduced check set and a reduced grade rule.

## Why CI comes out

Verified against `content_incrementality_v11/notebooks/v11_diagnostics.py`: the two CI checks are the
**only** checks in the 11-check build that read post-period tables.

```
ci_half_tvt_hrs  =  1.96 * se_7d_wz_hrs   FROM phase3_title_net_rate_{version}_wz        ← step 7 output
ci_half_ivt_hrs  =  1.96 * se_7d_wz_hrs   FROM phase3_title_net_rate_{version}_ivt_wz    ← step 7 output
```

Every other check comes from matching plus pre-period data:

| Source table | Feeds |
|---|---|
| `phase2_balance_diagnostics_{version}` | `smd_tvt`, `vr_tvt`, `smd_ivt`, `vr_ivt`, `n_pairs` |
| `phase2_matching_diagnostics_{version}` | `mean_logit_distance` |
| match-rate rollup (step3_4 §2) | `match_rate_pct` |
| out-of-sample placebo `[−56d, −28d)` | `placebo_out_cohens_d`, `placebo_out_ivt_cohens_d` |

Two consequences, both load-bearing:

1. **CI half-width is not a matching property.** It is driven by post-window outcome variance and cell
   size. A perfectly balanced cell with noisy post viewing fails CI; a badly balanced cell with tight
   post viewing passes it. Grading match quality on CI mixes two different questions.
2. **Dropping CI makes the report runnable immediately after `step3_4` + `balance_diagnostics`** — before
   `step5_8_tvt` / `step5_8_ivt` / step 7 exist. On a new session type that is exactly when you need it:
   it tells you whether to proceed into the post-period steps at all.

## The 9 checks

Dropping the two CI checks from the 11-check build leaves **9** — not 10. CI is two separate checks in
this build (`ci_tvt_pass` and `ci_ivt_pass`), one per metric.

| # | Check | Pass | Gate | Column |
|---|---|---|---|---|
| 1 | SMD Pre-TVT | \|SMD\| ≤ 0.10 | ★ | `smd_pre_tvt` / `smd_pass` |
| 2 | SMD Pre-IVT | \|SMD\| ≤ 0.10 | ★ | `smd_pre_ivt` / `smd_ivt_pass` |
| 3 | VR Pre-TVT | 0.80 ≤ VR ≤ 1.25 | | `vr_pre_tvt` / `vr_pre_pass` |
| 4 | VR Pre-IVT | 0.80 ≤ VR ≤ 1.25 | | `vr_pre_ivt` / `vr_ivt_pass` |
| 5 | A/A In-Sample TVT | \|d\| < 0.10 | | `placebo_in_cohens_d` / `aa_in_pass` |
| 6 | A/A In-Sample IVT | \|d\| < 0.10 | | `placebo_in_ivt_cohens_d` / `aa_in_ivt_pass` |
| 7 | A/A Out-of-Sample TVT | \|d\| < 0.10 | | `placebo_out_cohens_d` / `aa_oos_pass` |
| 8 | A/A Out-of-Sample IVT | \|d\| < 0.10 | | `placebo_out_ivt_cohens_d` / `aa_oos_ivt_pass` |
| 9 | Match Rate | ≥ 80% (title × segment) | | `match_rate_pct` / `match_rate_pass` |

**Checks 5 and 6 are algebraic restatements of the two gates, not independent evidence.** In
`v11_diagnostics.py`:

```python
smd_tvt["placebo_in_cohens_d"]     = smd_tvt["smd_pre_tvt"]
smd_ivt["placebo_in_ivt_cohens_d"] = smd_ivt["smd_pre_ivt"]
```

They differ from the gates only in the comparison operator (`< 0.10` strict vs `<= 0.10`), so they
diverge only at exactly \|d\| = 0.10. So `checks_passed / 9` double-counts pre-period balance: a cell
failing both SMD gates loses 4 of 9, not 2. **There are 7 distinct signals**: SMD-TVT, SMD-IVT, VR-TVT,
VR-IVT, A/A-OOS-TVT, A/A-OOS-IVT, match rate. Say this in the report's Criteria tab rather than letting a
reader treat 9/9 as nine independent confirmations.

## Grade rule (2 gates, documented deviation)

```
HIGH    both gates pass (SMD-TVT, SMD-IVT)  AND  <= 2 of the remaining 7 fail
MEDIUM  not HIGH, but checks_passed >= 5     (majority of 9)
LOW     otherwise
```

This mirrors the 11-check rule structurally — all gates + ≤2 non-gate fails for HIGH, simple majority for
MEDIUM — with the gate set reduced from 4 to 2 and the MEDIUM bar from ≥6/11 to ≥5/9. It is a **different
grade** from the 11-check grade and must not be compared against a published 11-check dashboard
cell-for-cell: with two gates removed, cells that were MEDIUM-on-CI-failure move up. Label the report
header "9 checks · 2 gates · CI excluded" so the two never get confused, and never reuse the stored
`grade` column from `phase2_psm_diagnostics_{version}` — recompute from the 9 flags client-side.

## Overall verdict badge

Same coverage-weighted rule as `publish-diagnostics` — HIGH-grade share of total weight, not row count,
so thin cells can't sink a strong catalog:

```
>= 80%  PASS      |  >= 60%  CONDITIONAL PASS  |  else  NEEDS IMPROVEMENT
```

**The weight depends on when you run it.** The canonical weight is `{month}_segment_tvt_hrs` from
`phase3_title_segment_summary_{version}_combined` — a **step5_8 output**, which does not exist yet in the
matching-only mode this report is designed for.

- **Mode A — matching-only (before the post steps).** Weight by `n_pairs`. Label the column and the
  header "pair-weighted", not "TVT coverage". Do not silently substitute pairs for hours under a TVT
  label. If you want a TVT-like weight at matching time, add `SUM(t_tvt) AS treated_pre_tvt_hrs` to the
  per-cell aggregate in `v{N}_balance_diagnostics.sql` (alongside `COUNT(*) AS n_pairs`) and weight by
  that — it is pre-event treated TVT, so label it that way too.
- **Mode B — after `step5_8_tvt` has run.** Weight by `{month}_segment_tvt_hrs` exactly as
  `publish-diagnostics` does, and say so in the header.

## Building it

Follow `publish-diagnostics` (it owns the file layout, the tab structure, and the client-side
recomputation), with these deltas:

1. **Query A** — drop `ci_half_tvt_hrs`, `ci_half_ivt_hrs`, `ci_tvt_pass`, `ci_ivt_pass` from the SELECT.
   Keep the `n_pairs >= 30` filter. If the pipeline has not reached step 7, those columns will be NULL or
   the whole `phase2_psm_diagnostics_{version}` table may not exist yet — in that case build the report
   directly from `phase2_balance_diagnostics_{version}` + the match-rate rollout + the OOS placebo cells
   and say in the header that it was assembled pre-step-7.
2. **Query B** — pipeline metrics: skip `{version}_ci_tvt_pass_rate_seg`, `{version}_ci_ivt_pass_rate_seg`
   and `{version}_ci_pass_rate_seg`. Keep the SMD / VR / A/A / match-rate pass rates.
3. **Query C** — the segment-TVT weight query only applies in Mode B. In Mode A skip it and set the
   weight from `n_pairs`.
4. **Data file** — `window.SCORECARD_DATA` with `ci_half_*` / `ci_*_pass` omitted from every
   `segment_results` row, `checks_passed` recomputed over the 9 flags, and `meta.description` naming the
   reduced set, e.g. `"CEM + ML score caliper (Variant C, segment-conditional), 9 matching diagnostics
   (TVT + IVT) — CI excluded"`.
5. **HTML** — 4 tabs unchanged (Summary / Diagnostics / Title × Segment / Criteria); 9 pass-rate cards
   instead of 11, 2 flagged gates instead of 4, CI columns removed from the Title × Segment table, and
   the Criteria tab carrying the 2-gate rule plus the checks-5-and-6-are-aliases note.
6. **Grades do not port.** If a grade-coloured results dashboard is also being published, it reads
   whichever `scorecard-data.js` is on disk. Writing a 9-check scorecard to the same path changes the
   colouring of an existing results dashboard. Either keep it at a distinct path
   (`scorecard/{version}-matching/`) or state plainly that the colouring is now 9-check.

## What this report does not tell you

It does not tell you the effect is measurable. A run can pass all 9 checks and still produce CIs too wide
to act on — that is the CI checks' job, and they only become computable after the post-period steps.
Report the 9-check verdict as "the matching is sound," and keep the precision question separate.
