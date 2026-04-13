# Skill Writing Guidelines

## Directory & File Structure

- Directory names: lowercase, digits, hyphens (e.g. `polardbx-sql`).
- Every skill must have a `SKILL.md` with YAML frontmatter (`name` + `description`).
- Detailed docs go in `references/`, executable scripts in `scripts/`, playbooks in `playbooks/`.
- Follow the [skills.sh](https://skills.sh) specification.

## SKILL.md Structure

Follow this section order (see `polardbx-sql/SKILL.md` as the reference example):

1. **Frontmatter**: `name` (kebab-case, matches directory), `description` (what it does + when to use + triggers, < 1024 chars), `metadata.version`.
2. **Title + one-line summary**: What the skill does.
3. **Scope**: What it applies to and what it does NOT apply to, with verification commands.
4. **Core Workflow**: Numbered steps the agent follows every time.
5. **Key Features / Differences Quick Reference**: Inline knowledge the agent can use without reading references — include key SQL commands, critical syntax, and common pitfalls.
6. **Best Practices** (if applicable): Numbered actionable recommendations.
7. **References**: Table format, relative paths. `| [references/xxx.md](references/xxx.md) | Description |`
8. **Playbooks** (if applicable): Table format, relative paths.

Key principle: **SKILL.md must contain enough inline knowledge for the agent to handle common questions without reading references.** References are for deep dives, not the primary source.

## Language

- Write `description`, `instructions`, and core logic in English — LLMs follow English instructions more reliably.
- Write `triggers` in both English and Chinese to cover different user input habits.
- Write output templates in the target user's language.
- Keep language consistent within each paragraph — do not mix languages mid-paragraph.

## Versioning

- Every `SKILL.md` has a `metadata.version` field in YAML frontmatter (semver).
- Bump the version once per commit, not per edit. Patch for fixes, minor for new content or restructuring.
