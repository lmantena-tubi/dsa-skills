# Session-type predicate library

Every predicate below is a `WHERE`-clause fragment over `core_prod.tubidw.video_session` (aliased `vs` in
the counters, unaliased in `treated_base`). Whichever one is chosen gets emitted into the **five sites**
listed in SKILL.md, byte-identically on the treated and control sides.

These are predicates on a **session**, applied on top of the qualifying gate that is independent of type
and never changes:

```sql
vs.ds BETWEEN DATE('{event_start}') AND DATE('{event_end_plus_1}')   -- ds is end_ts-derived; buffer +1d
AND DATE(vs.start_ts) BETWEEN '{event_start}' AND '{event_end}'
AND vs.tvt_millisec >= 300000                                        -- 5-min qualifying view
AND vs.country = 'US'
AND vs.device_hash_num < 500                                         -- 50% sample
AND vs.program_id IN (SELECT title_id FROM {universe_table})
```

---

## `organic` — the content-incrementality default

Organic discovery = not search, not deeplink. **Autoplay is included.**

```sql
(page_source IS NULL OR page_source != 'SearchPage')
AND NOT (
    attribution_type IS NOT NULL AND attribution_type != ''
    AND attribution_campaign IS NOT NULL AND attribution_campaign != ''
    AND attribution_medium IS NOT NULL AND attribution_medium != ''
)
```

The `page_source IS NULL OR …` form is deliberate: a bare `page_source != 'SearchPage'` drops NULL
`page_source` rows, because `NULL != 'SearchPage'` is NULL, not TRUE. Keep the explicit NULL arm.

The deeplink arm is the same attribution triple the team-canonical ICR-70% intentional filter uses. The
canonical **source CASE** uses a `TRIM`-hardened version — `NULLIF(TRIM(attribution_type), '') IS NOT NULL`
and so on — which additionally handles whitespace-only values. The pipeline's frozen text does not TRIM.
If you switch to the TRIM form, switch it in all five sites at once and note the deviation; a
whitespace-only attribution value will flip a handful of sessions between organic and deeplink.

## `autoplay`

```sql
autoplay_tvt_millisec > 0
```

This is the same test the canonical source CASE resolves **first**, ahead of every page-source bucket. Two
consequences worth stating to the user before building:

- An `autoplay` build is a **subset** of the `organic` build, not an alternative to it — autoplay sessions
  that aren't search/deeplink are already treated in the organic pipeline.
- If the user means "the complement of autoplay within organic," the predicate is the organic pair above
  plus `AND (autoplay_tvt_millisec IS NULL OR autoplay_tvt_millisec = 0)`. Ask which one.

Related columns, if a finer autoplay cut is wanted: `non_autoplay_tvt_millisec`, and on
`core_dev.dsa.dsac_appres_vsesh_sample` the chain fields (`video_session_sequence_id` = 1 for a manual
start, 2+ for autoplayed; `current_program_id` vs `program_id` for AP-same vs AP-diff).

## `search`

```sql
page_source = 'SearchPage'
```

**Collides with the `search_visit_7d` exact block** — see SKILL.md → "Collisions". Resolve before writing
SQL. Expect a much smaller treated population than organic; run preflight P1 first.

## `deeplink`

Two defensible definitions. Pick one explicitly.

**Raw attribution triple** (the complement of the organic deeplink arm):

```sql
attribution_type IS NOT NULL AND attribution_type != ''
AND attribution_campaign IS NOT NULL AND attribution_campaign != ''
AND attribution_medium IS NOT NULL AND attribution_medium != ''
```

**Canonical `source = 'Deeplink'` bucket** — the attribution triple *and* not already claimed by an
earlier arm of the team source CASE (autoplay wins first, then the page-source buckets):

```sql
(autoplay_tvt_millisec IS NULL OR autoplay_tvt_millisec = 0)
AND (page_source IS NULL OR page_source NOT IN (
      'VideoPlayerPage', 'HomePage', 'MovieBrowsePage', 'SeriesBrowsePage',
      'VideoPage', 'SeriesDetailPage'))
AND NULLIF(TRIM(attribution_type), '')     IS NOT NULL
AND NULLIF(TRIM(attribution_campaign), '') IS NOT NULL
AND NULLIF(TRIM(attribution_medium), '')   IS NOT NULL
```

Prefer the canonical bucket when the result will be compared against anything else labelled "Deeplink" on
the team — dashboards, `dsac_vs_first_watch.source_{0,1,5}`, and the STP deeplink-viewer counts all use the
CASE, so the raw-triple version will not reconcile with them. Also note that the raw triple **overlaps
autoplay and HomePage**, so a raw-triple deeplink build and an autoplay build are not disjoint.

Verify the `page_source` value list against the current data before trusting it — the CASE arms are the
source of truth, and page-source enums do change.

---

## Defining a new type

Pin down four things with the user, in writing, before any SQL:

1. **The predicate** — exact columns and values over `video_session`, NULL-handling included. If it needs a
   column the pipeline doesn't currently select, add it to the `qualifying_sessions` projection in
   step1_2 **and** to the control-side session scan.
2. **Disjointness** — does it overlap `organic` / `autoplay`? Overlap is allowed, but the user should know
   whether they're measuring a subset of an existing result or something new.
3. **The estimand** — "first session on the title, if it was this type" (frozen behaviour, rank-then-filter)
   or "first session of this type on the title" (predicate inside `ROW_NUMBER`). Different populations,
   different answer.
4. **Block interactions** — which of the 8 exact blocks the predicate makes degenerate. Run preflight P2
   and look at the marginals rather than reasoning about it.

Then write it into the version's `MATCHING_SPEC.md` and emit it into the five sites.

### Predicates that need more than a filter

Some plausible "types" are not session predicates at all and need extra pipeline work — flag this rather
than approximating:

- **Container / row-position types** ("row-1 HomePage", "above-the-fold 2×8"): `row_num` / `col_num` /
  `container` live on `core_prod.dsa.viewpres_vidsession_sample`, a **3% device sample**, not on
  `video_session`. Matching off a 3% sample cuts both treated and control supply ~33×.
- **Audience / fan types** ("horror fans"): that is a viewer attribute, not a session type. It belongs as a
  9th exact block (from `device_audiences_daily`, `running_points >= 1`) or as a population filter — not in
  the session predicate. Say which.
- **Presentation-conditional types** ("saw it but didn't click"): needs the presentation join, which the
  matching pipeline does not currently carry. That is a pipeline extension, not an argument change.
