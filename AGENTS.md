# Skill Writing Guidelines

## Directory & File Structure

- Directory names: lowercase, digits, hyphens (e.g. `polardbx-sql`).
- Every skill must have a `SKILL.md` with YAML frontmatter (`name` + `description`).
- Keep `SKILL.md` concise: clear Workflow + quick reference for key differences.
- Detailed docs go in `references/`, executable scripts in `scripts/`.
- Follow the [skills.sh](https://skills.sh) specification.

## Language

- Write `description`, `instructions`, and core logic in English — LLMs follow English instructions more reliably.
- Write `triggers` in both English and Chinese to cover different user input habits.
- Write output templates in the target user's language.
- Keep language consistent within each paragraph — do not mix languages mid-paragraph.

## Versioning

- Every `SKILL.md` has a `metadata.version` field in YAML frontmatter (semver).
- Bump the version once per commit, not per edit. Patch for fixes, minor for new content or restructuring.
