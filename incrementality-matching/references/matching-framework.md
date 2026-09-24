# The content-incrementality matching framework (Variant C)

The design below is **frozen**. It is what "apply the content-incrementality matching framework" means.
Copy the actual CASE expressions from the base pipeline's SQL — this document states the design and the
rules, not the canonical text. Where the two disagree, the SQL wins and this document is stale.

## Estimand

For each (viewer, title) whose **first** qualifying session on that title was of the chosen session type,
find a comparable viewer who was ML-scored on the same title on the same date but did **not** watch it, and
compare post-window viewing. Treated and control must be exchangeable on pre-event behaviour; the whole
apparatus below exists to make that true cell by cell.

## Keys and identities

```sql
-- viewer_id, used identically everywhere
CASE WHEN user_id IS NOT NULL AND user_id != 0
     THEN CAST(user_id AS STRING) ELSE device_id END AS viewer_id

-- event anchor
DATE_TRUNC('minute', start_ts) AS event_ts
DATE(start_ts)                 AS event_date
```

## Treated construction (step1_2)

1. `qualifying_sessions` — the type-independent gate (window, `ds` buffer, TVT ≥ 300s, US, universe,
   `device_hash_num` sample) with
   `ROW_NUMBER() OVER (PARTITION BY viewer_id, program_id ORDER BY start_ts ASC) AS session_rank`.
2. `treated_base` — `WHERE session_rank = 1` **then** the session-type predicate.

**Rank-then-filter is deliberate.** A viewer whose first touch was a different type is dropped, not
promoted to their second session. See SKILL.md for why this matters when switching types.

Covariates built on the treated side, all anchored on `event_ts`:

- `sessions_7d`, `sessions_1d` — `COUNT(DISTINCT video_session_id)` over sessions matching the **session-type
  predicate** in the window. These are sites 2 and 3 of the five.
- `tvt_7d_hrs`, `tvt_28d_hrs`, `tvt_1d_hrs`, and the IVT equivalents from `ivt_millisec`.
- `ivt_share_7d` / `ivt_share_28d` — `CAST(COALESCE(ivt,0) / NULLIF(tvt,0) AS DOUBLE)`. NULL (no pre-TVT)
  routes to the `<10%` bin downstream rather than dropping the row. Keep the `NULLIF`.
- `search_visit_7d` / `search_visit_1d` — `page_source = 'SearchPage'` presence. Site 5.
- `user_segment`, `tenure_bucket`, `is_registered`, `platform_device_bucket`, `is_fan_of_title_fandom`.

### Segment definition (identical expression on both sides, from `device_okr_daily`)

```
Power Viewer            visit_days_last28d_count >= 8
                        AND tvt_sec_28d / 3600 / visit_days >= 4
New (M1)                else, DATE(device_first_view_ts) >= DATE_SUB(event_date, 28)
Active non-power
  (Retained)            else, last_visit >= DATE_SUB(event_date, 28)
                              AND DATE(last_visit) != event_date
Non-active
  (Reactivated)         else
```

`seg_idx` mapping used downstream: `{0: Retained, 1: New (M1), 2: Reactivated, 3: Power Viewer}`.

### Tenure buckets

`unknown` · `lt_30d` · `30_90d` · `90_365d` · `365d_2y` · `gt_2y` (DATEDIFF thresholds 30 / 90 / 365 / 730).

## Control pool (control_pool)

Eligibility, all four required:

1. ML-scored on that title on that date (the ranker log — verify ID grain; step1_2 carries an explicit
   ranker ID-grain gate as Verification #1b);
2. `NOT EXISTS` in treated for that (viewer, title);
3. US-active on the date;
4. **No-contamination screen** — LEFT JOIN a wide-window title-TVT table (`ds` and `start_ts` spanning well
   before and through the post window) and keep only `vt.viewer_id IS NULL`. A viewer who watched the title
   anywhere in that span is not a valid control for it.

Control covariates mirror the treated ones exactly, with two intentional differences:

- **Anchor**: control windows anchor on **midnight of `event_date`** (`TIMESTAMP(DATE_SUB(event_date, 7))`),
  not on `event_ts`. This asymmetry is intentional and previously litigated — do not "fix" it.
- **Platform**: controls have no session, so platform is the deterministic primary-platform pick
  `ORDER BY SUM(tvt_millisec) DESC NULLS LAST, platform LIMIT 1`, wrapped
  `COALESCE(…, 'UNKNOWN')`. `'UNKNOWN'` must remain its own bucket — never an `ELSE` arm. A treated viewer
  always carries a real session platform, so unresolvable controls simply drop rather than matching a
  genuine CTV treated.

Step 3b (control engagement) is **chunked by `event_date`** to stay under the 3600s per-statement ceiling:
one `CREATE … AS SELECT` plus N−1 `INSERT INTO`. 7-day window → 2 chunks; 14-day → 4. The SELECT bodies must
be byte-for-byte identical apart from the `event_date BETWEEN` bound.

Use a `LEFT JOIN` + zero-fill for control engagement, not an `INNER JOIN` — an inner join drops zero-session
controls and biases the pool toward active viewers.

## Matching (step3_4)

### The 8 exact CEM blocks

```sql
ON  c.title_id             = t.title_id
AND c.event_date           = t.event_date
AND c.user_segment         = t.user_segment
AND c.is_registered        = t.is_registered
AND c.tenure_bucket        = t.tenure_bucket
AND c.search_visit_7d      = t.search_visit_7d
AND c.sessions_bucket      = t.sessions_bucket
AND c.platform_super_bucket= t.platform_super_bucket
AND c.tvt_bucket           = t.tvt_bucket
AND c.viewer_id           != t.viewer_id
WHERE ABS(t.logit_treated - c.logit_control) <= 0.2
```

**There is no hard IVT block. That absence is Variant C.** IVT balance is carried by the tie-break's second
key instead. Adding an IVT block makes it a different variant — it must go through the sweep and be
recorded, not slipped in.

### Stepwise ML-score caliper

Join ceiling `|Δlogit| ≤ 0.2`; the rank *prefers* the `≤ 0.1` tier. Both numbers matter: the ceiling
determines who is eligible, the tier determines who wins.

### Tie-break (5 keys, deterministic)

```sql
ROW_NUMBER() OVER (
  PARTITION BY t.viewer_id, t.title_id
  ORDER BY
    CASE WHEN ABS(t.logit_treated - c.logit_control) <= 0.1 THEN 0 ELSE 1 END,
    ABS(t.w_ivt        - c.w_ivt)        ASC,
    ABS(t.tvt_1d_hrs   - c.tvt_1d_hrs)   ASC,
    ABS(t.logit_treated- c.logit_control) ASC,
    ABS(HASH(t.viewer_id, c.viewer_id, t.title_id))
) AS treated_rank
```

The hash key makes ties resolve identically across re-runs. Keep it last.

Note that the bucket CASEs are **duplicated verbatim** in the treated and control CTEs of step3_4
(`treated_b`/`treated_x`, `control_b`/`control_x`). Edit all copies together.

### Segment-conditional windows and bins

| Segment | Match window | TVT bin (`FLOOR(hrs / div)`, capped) |
|---|---|---|
| Power Viewer | 7d | div 10, cap 120 |
| Active non-power (Retained) | 7d | div 4, cap 40 |
| New (M1) | 28d | div 30, cap 90 |
| Non-active (Reactivated) | 28d | div 30, cap 90 |

New and Reactivated were widened from 8/60 to 30/90 by a coverage sweep: New-segment match rate went
77.9% → 83.2% and cells at ≥80% went 13% → 42.5%. `sessions_bucket` edges also differ per segment — copy
them from the SQL.

### Series-only addition

For `content_type = SERIES`, a segment-conditional `prior_on_title` contamination screen (7d for
Power/Retained, 28d for New/Reactivated) with a `COALESCE`-to-0 guard so never-active viewers aren't
wrongly dropped, plus an explicit no-over-drop verification cell.

## Sweep

`v{N}_sweep.sql` maintains a persistent frontier table (`v{N}_sweep_frontier`) with a `variant` column, and
uses `{{VARIANT}}` template substitution over marked blocks (e.g. `>>> IVT BUCKET (variant A) <<<`). This is
the mechanism for re-optimising on a new population. Variant C is the sweep's answer for organic movies —
not a constant.

## Verification gates

| Gate | Target | Where |
|---|---|---|
| Match rate | ≥ 80% of (title × segment) cells | step3_4 §2 |
| `|SMD|` pre-period TVT | ≤ 0.10 | balance diagnostics |
| `|SMD|` pre-period IVT | ≤ 0.10 | balance diagnostics |
| Caliper quality | reported | step3_4 §4 |
| `pct_at_cap` | not materially worse than base | step3_4 §4b |
| Cell inclusion | `n_pairs >= 30` | all |
| Control reuse | ≤ 3 treated per control | audit explicitly |

```sql
SMD = (AVG(treated) - AVG(control)) / SQRT((VAR(treated) + VAR(control)) / 2)
```

Dedup controls to pair grain before computing SMD, and match the `DISTINCT` grain to the `GROUP BY` grain —
control reuse otherwise multi-counts control TVT/IVT and flatters balance.

Variance ratio is computed but **not** enforced. Known structural limit: Reactivated tops out near 73% match
rate — control supply, not bin width. Widening bins will not fix it and will degrade balance.
