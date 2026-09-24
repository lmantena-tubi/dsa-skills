# dsa-skills (personal)

Skills I author before they go to [adRise/dsa-skills](https://github.com/adRise/dsa-skills). Same layout, so
porting is a directory copy plus a PR.

**Private on purpose.** These skills name internal warehouse tables (`core_dev.dsa.*`, `core_prod.tubidw.*`),
ranker-log sources, and the frozen content-incrementality matching design. Keep it that way unless a skill is
scrubbed of them first.

## Structure

Each top-level directory is one skill:

- `skill.md` — the skill prompt (no YAML frontmatter; that's the dsa-skills convention)
- `skill.yaml` — metadata: `name`, `description`, `trigger`, `version`, `category`
- `README.md` — human-facing docs: what it does, how to invoke it, how to install it locally
- `references/` — material the skill loads on demand rather than carrying inline

## Skills

### Content

| Skill | Description |
|---|---|
| [incrementality-matching](incrementality-matching/) | Build a matched treated/control cohort for any session type — organic, autoplay, search, deeplink, or custom — using the content-incrementality matching framework (Variant C CEM + ML-score caliper), then publish the 9-check matching-diagnostics report |

## Porting to adRise/dsa-skills

Copy the skill directory into a clone of the team repo, branch with a `feature/` prefix, and open a PR.
Never commit or push to that repo's `main`. Required `skill.yaml` fields there are `name`, `description`,
`trigger`, and `version`.
