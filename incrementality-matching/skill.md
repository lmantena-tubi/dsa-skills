# Incrementality Matching Skill

Produce a matched treated↔control cohort for a **user-specified session type**, using the matching
framework developed for content incrementality. The framework itself is frozen and carries over verbatim;
the **session type is the input**.

Content incrementality today matches **organic discovery** sessions (everything except search and
deeplink). This skill generalizes that one axis. Other session types the user may ask for:

| `session_type` | Treated = a viewer's first qualifying session on the title that is… |
|---|---|
| `organic` (default) | not SearchPage **and** not a deeplink — the content-incrementality definition |
| `autoplay` | autoplay-driven (`autoplay_tvt_millisec > 0`) |
| `search` | started from SearchPage |
| `deeplink` | started from an external deeplink |
| `custom` | any predicate over `video_session` the user supplies |

Everything else — treated/control construction, CEM blocks, caliper, tie-break, verification — is the
frozen framework. **Do not redesign it because the session type changed.** Change the predicate, re-run
the sweep, re-verify balance.

## Arguments

- **session_type** (required): one of `organic`, `autoplay`, `search`, `deeplink`, or `custom`. If the
  user says "custom" or describes a new type in prose, get the predicate pinned down explicitly (see
  `references/session-types.md` → "Defining a new type") before writing any SQL.
- **version** (required): the new pipeline label, e.g. `v12`. Names every output table `…_{version}`.
- **base_version** (optional, default = newest `content_incrementality_v*` on disk or on a branch): the
  pipeline to clone. **Always clone a real pipeline — never author these queries from memory.**
- **event_window** (required): event start/end dates. Post window defaults to 7d.
- **universe** (optional, default: top-2000 US titles by TVT, re-ranked on the event window).
- **content_type** (optional, default `MOVIE`): `MOVIE` or `SERIES`. Series runs add the
  `prior_on_title` contamination screen — see the base pipeline's step1_2 trailer.

If `session_type` is missing, ask. It is the whole point of the skill and must not be guessed.

## The one thing that actually changes — and the five places it appears

The session-type predicate is **not a single filter**. In the pipeline it is emitted in five places, and
**treated and control must carry byte-identical text**. Any asymmetry silently biases matching: controls
get scored on a different notion of "session" than treated, the `sessions_bucket` exact block stops
comparing like with like, and the measured lift absorbs the difference.

| # | File | Site | Role |
|---|---|---|---|
| 1 | `v{N}_step1_2.sql` | `treated_base` | the discovery filter — defines who is treated |
| 2 | `v{N}_step1_2.sql` | treated `sessions_7d` counter | pre-event exposure covariate |
| 3 | `v{N}_step1_2.sql` | treated `sessions_1d` counter | same, 1-day window |
| 4 | `v{N}_control_pool.sql` | control `sessions_7d` / `sessions_1d` — **in every Step-3b chunk** | control-side mirror of 2 & 3 |
| 5 | both | the `search_visit_7d` exact block | reads `page_source = 'SearchPage'` from the identical source on both sides |

Site 4 is the trap. Step 3b is split into chunks (2 chunks on a 7-day window, 4 on a 14-day window) to
stay under the 3600s per-statement ceiling, and the chunks' SELECT bodies must stay **byte-for-byte
identical** — a divergence biases controls whose `event_date` lands in one chunk versus another. After
editing, diff the chunk bodies against each other and confirm the only difference is the
`event_date BETWEEN` bound.

Verification (run these after the edits, before running anything):

```bash
# the predicate must appear the same number of times on each side, and nowhere else
grep -c "<first distinctive line of the predicate>" sql/v{N}_step1_2.sql      # expect 3
grep -c "<first distinctive line of the predicate>" sql/v{N}_control_pool.sql # expect 2 × n_chunks
grep -rn "SearchPage\|attribution_campaign\|autoplay_tvt_millisec" sql/       # every hit accounted for
```

## Preflight — before you accept the frozen bins

The frozen Variant C bins and caliper were tuned on **organic MOVIE viewers**. They are a good starting
point for a new session type, not a guarantee. Three checks, all cheap, all read-only:

**P1 — treated volume.** Count treated rows per (title × segment) under the new predicate. Search and
deeplink populations are far smaller than organic; if a segment yields too few treated for `n_pairs >= 30`
cells, say so before spending a matching run.

**P2 — block degeneracy.** For each of the 8 exact blocks, get the treated marginal distribution. Any
block sitting >95% on one value is no longer balancing anything, and it still costs control supply. This
is where a search or deeplink build bites: see "Collisions" below.

**P3 — bin saturation (`pct_at_cap`).** The base pipeline's step3_4 already has this verification cell —
the share of treated saturating the top TVT bin. Series viewers were what forced it into existence. If
`pct_at_cap` is materially above the base run's value, the cap is compressing the covariate and the bins
need re-sweeping, not reusing.

Report all three before proceeding. If any fails, the sweep (below) is mandatory, not optional.

## Collisions between the chosen type and the frozen blocks

The frozen block set was chosen with `organic` as the treated definition. Two collisions are known:

**`search_visit_7d` vs `session_type = search`.** The treated search-visit counter's window includes the
event session itself (`start_ts >= event_ts - INTERVAL 7 DAYS`, and the event session's `start_ts` equals
`event_ts`). So under a search build every treated viewer has `search_visit_7d = 1` by construction. The
block degenerates: it stops balancing and instead hard-restricts controls to prior searchers. Options, in
order of preference:
1. Exclude the event session from the counter on the treated side only — **no**, that breaks symmetry.
2. Move the window strictly before the event on **both** sides (`< event_ts` treated, `< midnight`
   control) so the block measures *prior* search affinity. Symmetric, informative, and a documented
   deviation.
3. Drop the block and replace it with the type-symmetric analogue (prior-7d exposure to the surface being
   studied, bucketed). Requires a sweep re-run.

Pick one with the user, write it into the spec doc, and never leave it implicit.

**`autoplay` overlaps `organic`.** The organic definition excludes search and deeplink but **not**
autoplay — autoplay-started sessions are already inside the current content-incrementality treated
population. An `autoplay` build is therefore a *subset* of the organic build, not a disjoint alternative.
If the user wants the disjoint complement ("organic excluding autoplay"), that is a different predicate
and must be stated. Also note the team-canonical source CASE resolves Autoplay **first**, before every
page-source bucket — so "Autoplay session" and "HomePage session" are not overlapping labels in that
taxonomy even though the raw columns overlap. Confirm which taxonomy the user means.

## The frozen framework (apply, do not reinvent)

Full detail in `references/matching-framework.md`. The load-bearing parts:

- **8 exact CEM blocks**: `title_id`, `event_date`, `user_segment`, `is_registered`, `tenure_bucket`,
  `search_visit_7d`, `sessions_bucket`, `platform_super_bucket`, `tvt_bucket`.
- **No hard IVT block.** That absence *is* Variant C. IVT balance is carried by the rank tie-break. Adding
  an IVT block is a different variant and must go through the sweep.
- **Stepwise ML-score caliper**: join ceiling `|Δlogit| ≤ 0.2`; the rank prefers the `≤ 0.1` tier.
- **5-key deterministic tie-break**: caliper tier → `ABS(Δw_ivt)` → `ABS(Δtvt_1d_hrs)` → `ABS(Δlogit)` →
  `ABS(HASH(t.viewer_id, c.viewer_id, t.title_id))`.
- **Segment-conditional windows**: Power Viewer and Retained match on **7d** TVT+IVT; New (M1) and
  Reactivated on **28d**.
- **Segment-conditional TVT bins** (`FLOOR(hrs / div)`, capped): Power 10/120 · Retained 4/40 ·
  New 30/90 · Reactivated 30/90. `sessions_bucket` edges also differ per segment.
- **Anchoring asymmetry is intentional**: treated windows anchor on `event_ts`; control windows anchor on
  **midnight of `event_date`**. This is deliberate — do not "fix" it.
- **`'UNKNOWN'` platform stays its own bucket.** Never fold it into an `ELSE`; a control with an
  unresolvable platform must not be able to match a genuine CTV treated.
- **Control eligibility**: ML-scored on that title/date · `NOT EXISTS` in treated for that (viewer, title)
  · US-active · passes the no-contamination screen (LEFT JOIN the wide-window title-TVT table, keep
  `IS NULL`).
- **Control reuse cap**: a control may back at most 3 treated. Check it — v10 breached it (max 172×).

Copy every CASE expression **verbatim from the base pipeline's SQL**. Do not retype `platform_super_bucket`,
`user_segment`, `tenure_bucket`, or the bucket CASEs from memory or from this document — they are
duplicated in four places across step1_2 / control_pool / step3_4 and must stay identical.

## Ordering trap in `treated_base`

The treated build **ranks first and filters second**:

1. `ROW_NUMBER() OVER (PARTITION BY viewer_id, program_id ORDER BY start_ts ASC)` over **all** qualifying
   sessions (TVT ≥ 300s, in-window, in-universe, in-sample, US).
2. Take `session_rank = 1`.
3. **Then** apply the session-type predicate.

Consequence, and it is intentional: a viewer whose true first touch on the title was of a *different*
type is **dropped**, not promoted to their second session. The treated population is "viewers whose first
touch was `{session_type}`", not "viewers who ever had a `{session_type}` touch". This materially changes
who is treated when you switch types — a search-first viewer is excluded from the organic build and
included in the search build, and a viewer whose first touch was organic is excluded from the search build
even if they searched the title an hour later. State this in the spec doc for the new type; it is the
single most misread property of the design.

If the user wants "first `{session_type}` session" instead of "first session, if it was
`{session_type}`", that is a real design change: move the predicate inside the `ROW_NUMBER` source. It
changes the estimand. Confirm explicitly, and record which one was built.

## Workflow

### Step 0 — Establish the base pipeline

Locate the newest `content_incrementality_v*` directory. Some versions live only on a git branch
(`git show {branch}:{path}`), and late-added files (e.g. `step10_weights`) may be uncommitted in the
working tree — check both. Read the base `MATCHING_SPEC.md` and `step1_2.sql` in full before editing.

### Step 1 — Write the spec document first

Create `dsa-lab/content_incrementality_{version}/MATCHING_SPEC.md` **before** the SQL, stating:
the session type and its exact predicate; which of the five sites it lands in; the estimand
(first-touch-if-type vs first-type-touch); every deviation from frozen Variant C and why; and the
verification numbers marked **STALE / pending re-run on the {version} window** until the sweep confirms
them. A spec written after the SQL documents whatever happened; a spec written first is a decision.

### Step 2 — Clone and retarget

Clone the base pipeline to `content_incrementality_{version}/`, then apply, in this order:
1. version rename (`v{base}` → `v{version}`) across table names, column aliases, prose, the sweep
   frontier table, and the diagnostics notebook's DBFS path / grade function / metric-id literals;
2. the **session-type predicate** into all five sites;
3. date literals for the new window — **relative** offsets (`INTERVAL 7/28/56 DAYS`, `DATE_SUB(…)`,
   `SEQUENCE(-28, 6)`, tenure `DATEDIFF`) never move; only hardcoded literals do. Substitute in an order
   that can't collide (an old value that equals a new value elsewhere must be rewritten first);
4. Step-3b chunk count for the window length (7 days → 2 chunks, 14 days → 4).

Then diff every file against its base counterpart and confirm the only differences are: rename,
predicate, dates, chunking, prose. That diff is the proof of structural identity with a validated
pipeline — it is the main safety property of this workflow.

### Step 3 — Sweep

Run `v{version}_sweep.sql` to re-confirm the variant on the new population. The sweep writes a persistent
frontier table with a `variant` column and uses `{{VARIANT}}` template substitution over marked blocks.
**Do not skip this because Variant C "is the framework."** Variant C is the framework's current answer for
organic movies; the sweep is the framework's method for getting that answer. A new session type gets the
method, not the cached answer. Report the frontier and the selected variant.

### Step 4 — Match and verify

Run step3_4, then the balance diagnostics. Gates:

| Gate | Target |
|---|---|
| Match rate | ≥ 80% of (title × segment) cells |
| `|SMD|` on pre-period TVT | ≤ 0.10 |
| `|SMD|` on pre-period IVT | ≤ 0.10 |
| Cell inclusion | `n_pairs >= 30` |
| Control reuse | ≤ 3 treated per control |
| `pct_at_cap` | not materially worse than the base run |

SMD = `(AVG(t) - AVG(c)) / SQRT((VAR(t) + VAR(c)) / 2)`, with controls deduped to pair grain first.
Variance ratio is computed but not enforced. Known structural limit: the Reactivated segment tops out
around 73% match rate — that is control supply, not bins, and widening bins will not fix it.

If a gate fails, go back to Step 3 — do not loosen the gate.

### Step 5 — Publish the matching diagnostics report (9 checks, no CI)

Full spec in `references/matching-diagnostics.md`. Build the diagnostics evaluation dashboard
(`{version}-diagnostics-evaluation.html` + `scorecard-data.js`, per the `publish-diagnostics` layout) with
the **two CI checks removed**, leaving **9 checks and 2 gates**. This is the report that answers "how did
the matching go" at (title × segment) grain.

| | 11-check build | this report |
|---|---|---|
| Checks | 11 | **9** — drop `ci_tvt_pass`, `ci_ivt_pass` |
| Gates | 4 (SMD-TVT, SMD-IVT, CI-TVT, CI-IVT) | **2** (SMD-TVT, SMD-IVT) |
| HIGH | all 4 gates + ≤2 of 7 non-gate fail | both gates + ≤2 of 7 non-gate fail |
| MEDIUM | `checks_passed >= 6` (of 11) | `checks_passed >= 5` (of 9) |
| Runnable | after step 7 | **after `step3_4` + `balance_diagnostics`** |

Why CI comes out: the two CI half-widths are the only checks sourced from post-period tables
(`1.96 * se_7d_wz_hrs` from `phase3_title_net_rate_{version}_wz` / `_ivt_wz`). CI half-width measures
post-window outcome variance and cell size, not balance — a well-matched cell with noisy post viewing
fails it. Excluding it makes the report both *about matching* and *available before the post steps run*,
which is when a new session type needs it.

Three things to get right, all detailed in the reference:

- **Verdict weight.** The coverage-weighted PASS / CONDITIONAL PASS / NEEDS IMPROVEMENT rule (≥80% / ≥60%
  of HIGH-grade weight) normally weights by `{month}_segment_tvt_hrs` — a step5_8 output that does not
  exist yet in matching-only mode. Weight by `n_pairs` and **label it "pair-weighted"**; never print a
  pair count under a TVT label.
- **Checks 5 and 6 are aliases of the gates**, not independent evidence — `placebo_in_cohens_d` is
  assigned `smd_pre_tvt` verbatim. There are 7 distinct signals in the 9. State this in the Criteria tab.
- **The grade is not the 11-check grade.** Label the header "9 checks · 2 gates · CI excluded", recompute
  grades client-side from the 9 flags rather than reusing the stored `grade` column, and keep the
  scorecard at a distinct path if an 11-check dashboard is already published against the same one.

### Step 6 — Hand off

Report: the predicate, the five sites verified symmetric, the preflight numbers, the selected variant, the
match rate and SMD tables, the gates that passed or failed, and the 9-check diagnostics report with its
grade distribution and coverage verdict. Then the pipeline continues into the post-period rate steps and
the CI checks, which are out of scope here.

## Notes

- **No fabrication.** If a table is missing, a column absent, or a query errors — show the error and stop.
  Never invent a predicate, a bin edge, a match rate, or an SMD.
- **Never push to `main`.** Feature branch + PR only, in both `dsa-lab` and `dsa-skills`.
- All SQL goes through the team's `query-runner`; the legacy CLI name is banned by CI.
- Ask before uploading notebooks to Databricks, and check whether the target notebook is currently
  running before overwriting it. Use `tubi-dev.cloud.databricks.com` links.
- `dsa.dsac_program_info` — not `core_dev.dsa.dsac_program_info`. `content_type` is **lowercase** there and
  **uppercase** in `video_session`; a missing `LOWER()`/`UPPER()` silently empties the table.
- Always add a static `vs.ds BETWEEN` filter, buffering the `start_ts` window by ~1 day (`ds` is derived
  from `end_ts`). Use `ivt_millisec` for IVT. Pre-aggregate engagement at (viewer, event_date) before
  joining to the title-level table. Never `ROW_NUMBER() + RAND()` to sample — use `device_hash_num`.
- CAST decimal aggregates to DOUBLE before `toPandas()`.
