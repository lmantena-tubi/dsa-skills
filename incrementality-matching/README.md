# incrementality-matching

Build a matched treated/control cohort for any session type — organic, autoplay, search, deeplink, or a
custom predicate — using the content-incrementality matching framework.

## What it does

Content incrementality measures the causal effect of a title on downstream viewing by matching each
**treated** viewer (someone whose first qualifying session on the title was *organic discovery* — not search,
not deeplink) to a comparable **control** viewer who was shown the title by the ranker on the same day but
didn't watch it. The matching design — Variant C coarsened exact matching plus an ML-score caliper,
segment-conditional windows and bins, a deterministic tie-break, and a fixed set of balance gates — was tuned
across many pipeline versions and is treated as frozen.

This skill makes the **session type** the input instead of a hardcoded assumption. Ask for autoplay sessions,
search sessions, deeplink sessions, or define a new predicate, and the same framework gets applied to that
population: the same treated/control construction, the same blocks, the same caliper, the same verification.

The reason this is a skill and not a find-and-replace is that the session-type predicate appears in **five
places** in the pipeline, and treated and control must carry byte-identical text in all of them. Asymmetry
doesn't error — it silently biases the measured lift. The skill enumerates the five sites, verifies symmetry
by grep, and handles the two known collisions between a chosen type and the frozen block set.

## How to use

Invoke `/incrementality-matching` with the session type, or on its own to be prompted:

- `/incrementality-matching autoplay sessions, v12, Sep 1-7 2026, top 2000 US movies`
- `/incrementality-matching search sessions` — you'll be asked for the version and window
- `/incrementality-matching` — you'll be asked for the session type first, since it's the whole point

Arguments: **session_type** (required — `organic` / `autoplay` / `search` / `deeplink` / `custom`),
**version**, **event_window**, and optionally **base_version**, **universe**, **content_type**.

## Examples

1. `/incrementality-matching autoplay, v12, Sep 1–7 2026` — reports that an autoplay build is a *subset* of
   the current organic build rather than an alternative to it, asks whether you meant
   "organic excluding autoplay," then clones the newest pipeline, emits the autoplay predicate into all five
   sites, runs the three preflight checks, re-runs the variant sweep on the new population, and reports match
   rate and SMD against the gates.
2. `/incrementality-matching search sessions, v13` — flags up front that the `search_visit_7d` exact block
   degenerates under a search build (every treated viewer has a search visit by construction), offers three
   resolutions, and won't write SQL until one is chosen and recorded in the spec document.
3. `/incrementality-matching custom: sessions from the Kids tab` — pins down the predicate, its overlap with
   the existing types, the estimand, and which blocks it degenerates before touching any file.

## Output

A cloned, retargeted pipeline directory (`content_incrementality_{version}/`) with the session-type predicate
emitted symmetrically, a `MATCHING_SPEC.md` written *before* the SQL that records the predicate and every
deviation from frozen Variant C, and a verification report: preflight numbers, the sweep frontier and selected
variant, match rate, SMD tables, and pass/fail against each gate.

Plus a **matching diagnostics dashboard** — the diagnostics evaluation HTML at (title × segment) grain with
the two CI checks removed, leaving **9 checks and 2 gates**. CI half-width is the only part of the 11-check
build sourced from post-period tables, and it measures outcome variance rather than balance; dropping it
makes the report both *about the matching* and *available as soon as matching finishes*, before the
post-period rate steps run. That is the report that says whether the match on a new session type is sound
enough to proceed.

The skill does not run the pipeline for you — the SQL notebooks are run interactively in Databricks — and it
never commits, pushes, or uploads without being asked.

## Notes

- **No fabrication.** A missing table, absent column, or failing query stops the run with the error shown.
  Predicates, bin edges, match rates, and SMDs are never invented.
- The framework is frozen, but **Variant C is the sweep's answer for organic movies, not a constant**. A new
  session type gets the sweep re-run, not the cached answer.
- Reference material lives in `references/session-types.md` (the predicate library and how to define a new
  type), `references/matching-framework.md` (the frozen design, keys, blocks, caliper, and gates), and
  `references/matching-diagnostics.md` (the 9-check no-CI diagnostics report).

## Install

This copy is in the **dsa-skills layout** — `skill.md` (no frontmatter) + `skill.yaml` + `README.md` +
`references/`. It ports to `adRise/dsa-skills` as-is; open a `feature/` branch and a PR there, never commit
to `main`.

To run it in **local Claude Code** instead, copy the directory to `~/.claude/skills/incrementality-matching/`,
rename `skill.md` → `SKILL.md`, and prepend the frontmatter — `name` and `description` from `skill.yaml`,
plus the tool allow-list, which the dsa-skills format has nowhere to carry:

```yaml
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Bash(databricks api post /api/2.0/sql/statements:*)
  - Bash(databricks api get:*)
  - Bash(grep *)
  - Bash(diff *)
  - Bash(git show *)
  - Bash(python3 *)
  - Bash(open *)
```

Without that block the skill still works; it just prompts for each tool instead of pre-approving the
read-only Databricks and grep/diff calls it makes.
