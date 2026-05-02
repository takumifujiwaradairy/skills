# CLAUDE.md — takumifujiwaradairy/skills

Repo conventions for Claude Code agents working on this repository.

## Purpose

This repository hosts personal Claude Code skills owned by `@takumifujiwaradairy`. Each skill is a single self-contained directory at the repo root.

## Layout

```
<repo-root>/
├── README.md
├── CLAUDE.md       (this file)
├── LICENSE
├── .gitignore
└── <skill-name>/
    ├── SKILL.md         (required, frontmatter + body)
    ├── references/      (optional, supporting docs)
    ├── assets/          (optional, images / templates)
    └── agents/          (optional, sub-agent prompts)
```

A skill directory at the repo root is automatically discoverable when the repo is cloned as `~/.claude/skills/`.

## SKILL.md frontmatter

Every `SKILL.md` must start with YAML frontmatter:

```yaml
---
name: <kebab-case-skill-name>            # must equal the directory name
description: <one-paragraph trigger description for Claude to decide invocation>
user-invocable: true                     # whether `/<name>` activates the skill
allowed-tools:                           # restrict tool access if appropriate
  - WebSearch
  - WebFetch
  - Read
  - Write
---
```

The `description` is the most important field — it is what Claude reads to decide whether to invoke the skill. Make it concrete: list specific trigger phrases, example user requests, and a brief "skip when ..." clause.

## Skill body conventions

- Open with the **single most important rule** in a fenced block ("the iron law", "the gate", etc.) so Claude internalizes it before reading details.
- Use numbered steps for the procedure; do not collapse them.
- Include a **worked example** (a real or representative case) showing the skill applied end-to-end.
- Tag every external claim with a **status flag** (✅ / ❓ / ❌) so the user reads verification status inline, not in a footnote.
- Keep skills under ~500 lines unless the domain genuinely requires more. Long skills should be split into a short `SKILL.md` + supporting files in `references/`.

## Workflow

- All changes go through pull requests against `main`.
- Branch naming: `feat/<skill-name>` for new skills, `fix/<skill-name>-<summary>` for bug fixes, `docs/<scope>` for README / repo docs.
- Commit style: Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`).
- A new skill PR must include:
  1. The `<skill-name>/SKILL.md` and any supporting files.
  2. A README update adding the skill to the appropriate category table.
  3. A short PR body that explains the trigger, the gap it fills, and at least one worked example.

## Reviewing a skill PR

When reviewing a skill PR, check:

- Does the `description` clearly enumerate trigger conditions?
- Is the iron law / hard rule visible above the fold?
- Are anti-patterns / red flags enumerated, not implied?
- Does the worked example actually demonstrate the rule firing?
- Does the skill duplicate something that already exists in the public ecosystem? (Search GitHub for similar skills first.)
- Is `allowed-tools` minimal?

## Out of scope

- Project-specific skills go in the project's `.claude/skills/` directory, not here.
- Skills that depend on private credentials or company-proprietary information should not be in this public repo.
