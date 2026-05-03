# takumifujiwaradairy/skills

Personal collection of Claude Code skills maintained by [@takumifujiwaradairy](https://github.com/takumifujiwaradairy).

Each top-level directory is a standalone skill. The directory layout deliberately mirrors `~/.claude/skills/<skill-name>/SKILL.md` so the repo can be cloned directly into the Claude Code skill location.

## Install

### Option A — clone into your Claude Code skill directory

```sh
# Back up any existing skills first
mv ~/.claude/skills ~/.claude/skills.bak  # if you already have skills

git clone git@github.com:takumifujiwaradairy/skills.git ~/.claude/skills
```

After this, every skill directory inside the repo is automatically discoverable by Claude Code as `~/.claude/skills/<skill-name>/`.

### Option B — install a single skill

```sh
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone --depth 1 --filter=blob:none --sparse \
  git@github.com:takumifujiwaradairy/skills.git takumifujiwaradairy-skills
cd takumifujiwaradairy-skills
git sparse-checkout set <skill-name>
mv <skill-name> ../
cd .. && rm -rf takumifujiwaradairy-skills
```

## Skills

### Planning & architecture

| Skill | Description |
| --- | --- |
| [architecture-premise-verify](architecture-premise-verify/) | Force web verification of every external-service feature claim BEFORE recommending an architecture. Prevents the "I assumed service X had feature Y" failure mode that wastes hours of downstream implementation. |

### Skill maintenance

| Skill | Description |
| --- | --- |
| [empirical-prompt-tuning](empirical-prompt-tuning/) | Iteratively improve agent-facing instructions (skills / slash commands / CLAUDE.md / code-gen prompts) by having a bias-free executor run them and evaluating two-sidedly until improvements plateau. **Mirrored from [mizchi/skills](https://github.com/mizchi/skills/tree/main/empirical-prompt-tuning) — see `empirical-prompt-tuning/NOTICE.md` for attribution.** |

### Personal use (maintainer-only)

| Skill | Description |
| --- | --- |
| [mynote-add](mynote-add/) | Capture knowledge into [@takumifujiwaradairy](https://github.com/takumifujiwaradairy)'s local Obsidian vault at `~/Documents/my_notes/`, following the existing folder taxonomy / naming convention / frontmatter format / wikilink discipline. **Personal-use only — vault path and folder structure are hardcoded for the maintainer.** Inspired by [claudesidian](https://github.com/heyitsnoah/claudesidian) and [kepano/obsidian-skills](https://github.com/kepano/obsidian-skills). |

### Research workflow

| Skill | Description |
| --- | --- |
| [research-search](research-search/) | Collect up-to-date information from the web for technology-selection research; writes `research-raw.md`. |
| [research-verify](research-verify/) | Cross-check facts in `research-raw.md` against multiple sources, validate dates and URLs; writes `research-verified.md`. |
| [research-document](research-document/) | Render `research-verified.md` into a neutral, decision-ready Markdown comparison report. |

## Contributing / maintaining

Changes are made via pull requests against `main`. See [`CLAUDE.md`](CLAUDE.md) for skill conventions and review guidelines.

## License

[MIT](LICENSE).
