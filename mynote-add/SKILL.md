---
name: mynote-add
description: Personal-use skill for the maintainer (@takumifujiwaradairy). Adds a new note to the local Obsidian vault at `~/Documents/my_notes/` following the maintainer's existing conventions (frontmatter with `created`/`updated`, `#tag`-style tags, underscore-separated filenames, automatic folder routing, wikilinks to related notes, index registration when applicable). Triggers when the user (= the maintainer) explicitly says "メモして", "ノート追加して", "学んだから残して", "save to my notes", "今日学んだことを残して", or invokes `/mynote-add [topic]`. Source of the note: an explicit topic argument, a file path, or — by default — the recent conversation context. NOT generally useful for other people; this skill hardcodes the maintainer's vault path and folder taxonomy.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash(date *)
  - Bash(ls *)
  - AskUserQuestion
---

# mynote-add — Capture knowledge into the maintainer's local Obsidian vault

> **This skill is personal-use. The vault path, folder taxonomy, naming convention, and index files are all hardcoded for `@takumifujiwaradairy`'s setup.** If you are someone else reading this, fork it and replace the constants in [§ Vault constants](#vault-constants) below.

## The Iron Law

```
RESPECT THE EXISTING VAULT. DO NOT INVENT NEW STRUCTURE.
ALWAYS CONFIRM THE TARGET FOLDER + FILENAME WITH THE USER BEFORE WRITING.
PRESERVE CONVENTIONS — frontmatter, tags, naming, wikilinks — even when sloppy.
```

The vault has accumulated several years of conventions. The skill's job is to **append into that pattern**, not to reorganize it. If a folder name has a typo, the skill keeps the typo. If two folders both look right, the skill **asks** the user instead of inventing a third.

---

## Why this skill exists

Capturing what was just learned is the highest-leverage use of an LLM session. Without a skill:

1. The user types "メモして" and the assistant invents a path / filename / format on the fly.
2. The new note doesn't follow existing conventions → vault inconsistency grows.
3. The note isn't linked from any index → orphan, never re-discovered.
4. Frontmatter dates are wrong / missing → Obsidian queries break.
5. Tags don't match the rest of the vault → search noise.

This skill encodes the maintainer's vault conventions so capture is **instant, consistent, and discoverable**.

---

## Vault constants

These are hardcoded for the maintainer. Fork the skill to change them.

```
VAULT_ROOT       = ~/Documents/my_notes
WORKLOG_DIR      = 01_ログ/作業ログ
KNOWLEDGE_DIR    = 02_ナレッジ
STASH_DIR        = 06_スタッシュ
INBOX_FALLBACK   = 06_スタッシュ        # if folder cannot be confidently chosen
```

### Folder taxonomy (existing — do not invent new categories)

```
~/Documents/my_notes/
├── 00_INDEX(目次)/
├── 01_ログ/
│   └── 作業ログ/                    # session work logs (YYYY-MM-DD_*.md)
├── 02_ナレッジ/
│   ├── 00_ソフトスキル/
│   ├── 01_業務メモ/
│   ├── 02_ITスキル/                  # technical knowledge (largest folder)
│   ├── 03_読んだ本/
│   ├── 04_人物/
│   ├── 05_心理学/
│   ├── 07_記憶術/
│   ├── 08_AI活用/                    # AI-related knowledge (skills, prompts, hallucination patterns)
│   ├── 12_新しく何かを始めた時/
│   ├── 13_英語/, 14_English_Learning/
│   └── 15_起業/
├── 03_知識の体系/
├── 04_計画・振り返り/
├── 05_目標/
├── 06_スタッシュ/                    # temporary / unsorted
├── 11_個人開発/, 011_個人開発/
├── 12_Anki暗記カード/
├── 13_契約書&APIキー/
├── 14_成功体験/, 15_失敗体験/
├── 18_LeetCode/
└── 19_Yourtory/                      # Yourtory-specific notes
```

### Folder routing rules (priority order)

When deciding where to place a new note, evaluate in this order and pick the first match:

1. **Explicit user instruction** ("Yourtoryのノートに追加して" → `19_Yourtory/`)
2. **Session work log** (today's hands-on session, "今日やったこと") → `01_ログ/作業ログ/YYYY-MM-DD_<title>.md`
3. **Topic match against `02_ナレッジ/` subfolders**:
   - Tech (programming, infra, DB, CI/CD, framework) → `02_ナレッジ/02_ITスキル/`
   - AI / Claude / prompt / skill / LLM → `02_ナレッジ/08_AI活用/`
   - Soft skill, comm, mgmt → `02_ナレッジ/00_ソフトスキル/`
   - Book summary → `02_ナレッジ/03_読んだ本/`
   - English / language learning → `02_ナレッジ/14_English_Learning/`
   - Startup / business → `02_ナレッジ/15_起業/`
4. **Failure / success retrospective** → `15_失敗体験/` or `14_成功体験/`
5. **Yourtory-specific business knowledge** → `19_Yourtory/`
6. **Cannot decide confidently** → ask user via `AskUserQuestion` with the top 2-3 folder candidates.

**Never silently dump to `INBOX_FALLBACK` without asking.** The fallback is for when the user explicitly says "とりあえずどこかに置いて".

---

## Naming conventions (observed from existing files — do not deviate)

### Filename pattern

```
<TopicPrefix>_<具体>_<より具体>.md
```

- Underscore (`_`) as separator, no spaces, no kebab-case
- Mix EN + JP allowed and common
- TopicPrefix examples observed: `GitHub`, `Cloudflare`, `DB設計`, `ER`, `BaaS`, `ClaudeCode`, `AI設計ハルシネーション`, `Devin`
- For session work logs: `YYYY-MM-DD_<内容>.md`

### Examples (good — match existing style)

```
GitHub_Secrets_3種類の使い分け.md
Cloudflare_wrangler_secret管理_理解の盲点.md
ClaudeCode_セッション管理の運用ベストプラクティス.md
DB設計_PostgreSQLスキーマ設計チェックリスト.md
2026-05-02_Slack_GitHub_保存Bot構築とSkill運用整備.md
```

### Examples (bad — do NOT use)

```
github-secrets.md            # kebab-case, English-only — not the vault style
cloudflare workers notes.md  # spaces in filename
note_2026-05-03.md           # too generic, no topic
01_introduction.md           # numbered prefix collides with folder convention
```

---

## Frontmatter format (required, exactly this)

```markdown
---
created: YYYY-MM-DDTHH:MM
updated: YYYY-MM-DDTHH:MM
---
#Tag1 #Tag2 #Tag3

# <Title in Japanese or English, matching the topic>
```

- `created` and `updated` are **identical on first write** (set both to current local time)
- Tags use `#PascalCase` or `#kebab-case` or `#日本語` — observed mix; pick what matches sibling files in the same folder
- Title (`# `) is on a separate line from the tag line, with one blank line between

Get the timestamp via `Bash(date "+%Y-%m-%dT%H:%M")`. Do not hardcode.

---

## The procedure

### Step 1 — Resolve source material

Determine what to capture, in this priority:

1. **File path argument** (`/mynote-add ./path/to/transcript.md`) → read that file as source.
2. **Topic argument** (`/mynote-add Cloudflare Worker secret管理`) → use the recent conversation about that topic, plus optional brief web search if conversation lacks depth.
3. **No argument** → use the most recent meaningful explanation in this conversation (last block where the assistant taught a concept, debugged a problem, or surfaced a non-obvious insight).

State to the user in one sentence what will be captured. Example:
> "Capturing today's understanding of `Cloudflare wrangler secrets vs vars` from the recent quiz-me session."

### Step 2 — Decide target folder

Apply the routing rules above. If you can pick confidently (one rule clearly fires), state the folder and proceed. If two or more candidates are plausible, **ask via `AskUserQuestion`** with the top 2-3 candidates, and let the user pick (or pick "Other" to type a custom path).

Never write the file before the folder is confirmed.

### Step 3 — Generate filename

Generate one filename following the naming convention. Show it to the user and **confirm before writing** (one short `AskUserQuestion` is fine, or just present and ask "OK?").

If the file already exists at the target path, do **not** overwrite. Append a numeric suffix (`_2`, `_3`) and re-confirm — or ask the user whether to merge into the existing file instead.

### Step 4 — Generate the note body

Use this skeleton, fill it from the source material:

```markdown
---
created: <NOW>
updated: <NOW>
---
#<Tag1> #<Tag2> #<Tag3>

# <Title>

## なぜこのノートを書いたか / Why this note exists

<2-4 sentences. The trigger that made this worth capturing. Not "I learned X" — the *moment* of understanding or failure.>

## <Main content sections>

<Use H2/H3 headings. Tables for comparisons. Code blocks for any concrete syntax.>

## 学び / Takeaways

<3-5 bullet points. The rules / lessons that survive into future work.>

## 関連ノート

- [[<existing related note 1>]]
- [[<existing related note 2>]]

## 参考

- <URLs, sources>
```

Adapt sections to the topic. Mandatory sections: frontmatter, title, **学び / Takeaways**, **関連ノート** (even if empty after a search). Everything else is optional.

### Step 5 — Find related notes (wikilinks)

Before writing, run `Glob` + `Grep` against the vault to find 2-4 related notes:

- Glob: `~/Documents/my_notes/**/*.md` filtered by main topic keywords
- Grep: search for the most distinctive 1-2 keywords inside `*.md` to find notes that mention the topic

Prefer recent (`02_ナレッジ/` and `01_ログ/作業ログ/` over `06_スタッシュ/` and `100_アーカイブ/`).

Add them to the `## 関連ノート` section as `[[<filename without .md>]]` Obsidian wikilinks. If nothing matches, leave the section with a single line `- (none yet)` rather than omitting it.

### Step 6 — Write the file

Use `Write` tool with the full path. Confirm the path one more time in the final reply (so the user can `Cmd+click` to open it).

### Step 7 — Update index, if applicable

If the chosen folder has a co-located index file (e.g., `02_ナレッジ/08_AI活用/AI活用_目次.md`), append a one-line entry to the appropriate category table in the index. Same Underscore wikilink format. If unsure where in the index, ask before editing.

If the folder has no index, skip — do not create one.

### Step 8 — Confirm to user

Reply with:
- ✅ filename + full path
- the chosen folder + reason
- which sibling notes you wikilinked to
- whether the index was updated
- offer: "再 quiz したい? `/quiz-me <topic>` で" — if the captured note is a learning gap (i.e., it includes a "理解の盲点" or "失敗" theme)

---

## Anti-patterns

### Anti-pattern 1 — Inventing new folders or top-level categories
The vault has 30+ years of accumulated structure. Adding `02_ナレッジ/16_Cloudflare/` because Cloudflare doesn't have a dedicated folder is **wrong**. The right answer is `02_ナレッジ/02_ITスキル/Cloudflare_<...>.md`. Tag for the topic, don't fragment the folder structure.

### Anti-pattern 2 — Writing without confirming folder + filename
Even if confident, present the destination once before writing. The user catches mistakes that the routing rules miss.

### Anti-pattern 3 — Generic / dateless filenames
`memo.md`, `note_2026-05-03.md`, `claude_session.md` — useless in 6 months. Filename must contain the **topic** specifically enough that searching the vault for that topic finds it.

### Anti-pattern 4 — Skipping the frontmatter timestamp
`created` / `updated` drive Obsidian's sorting and any future scripted queries. Always populate via `Bash(date)`, never guess or omit.

### Anti-pattern 5 — Empty 関連ノート when the vault has matches
The wikilink graph is the entire point of an Obsidian vault. Spend 10 seconds on `Glob` + `Grep` before claiming there's nothing related. Only write `- (none yet)` after an actual search returned nothing.

### Anti-pattern 6 — Capturing the assistant's words verbatim
The user's understanding is the source, not the assistant's explanation. If the captured note is going to be re-read by the user later, it should reflect *their* framing of what stuck — not a transcript of the assistant's perfect prose. When the user articulated a wrong-then-right journey (as in `quiz-me` failures), capture that journey explicitly. The "wrong belief → corrected" structure is the most valuable kind of note.

### Anti-pattern 7 — Overwriting existing files
Never `Write` over an existing path without explicit confirmation. Suffix or merge — do not silently replace.

---

## Output format example

```markdown
Capturing your understanding of **Cloudflare wrangler secrets vs vars** based on the quiz-me session just now.

**Target**: `~/Documents/my_notes/02_ナレッジ/02_ITスキル/Cloudflare_wrangler_secret管理_理解の盲点.md`
**Reason**: technical knowledge → 02_ITスキル. Topic prefix `Cloudflare_` already used by sibling files; suffix `_理解の盲点` reflects this is a gap-correction note.

[AskUserQuestion: confirm path?]
✓ User confirms.

[Glob + Grep for related notes]
Found 3 wikilink candidates:
- [[CloudflareWorkers_概念と具体例とGCP対応]]
- [[Cloudflare_IaC_wrangler_セットアップ]]
- [[GitHub_Secrets_3種類の使い分け]]

[Write file]
✅ Written to `02_ナレッジ/02_ITスキル/Cloudflare_wrangler_secret管理_理解の盲点.md`

The folder has no index file → no index update needed.

Want to re-quiz on this topic in a few days to confirm it sticks? `/quiz-me Cloudflare wrangler secret`
```

---

## Interaction with other skills

- **`quiz-me`**: this skill is the natural follow-up after a quiz reveals a gap. quiz-me identifies what wasn't understood; mynote-add captures the corrected understanding.
- **`empirical-prompt-tuning`**: when this skill itself feels off (wrong folder choices, ugly filenames, missed wikilinks), use empirical-prompt-tuning to tighten the rubric.
- **Future Phase 2**: integrate with Anki via AnkiConnect so each captured note can optionally produce 1-3 spaced-repetition cards. Inspired by [`ishiko732/anki-skills`](https://github.com/ishiko732/anki-skills) and [`ianhi/anki-defs`](https://github.com/ianhi/anki-defs). Not implemented yet.

---

## Attribution

The vault-aware capture pattern, frontmatter discipline, and PARA-style routing are inspired by:

- [`heyitsnoah/claudesidian`](https://github.com/heyitsnoah/claudesidian) — especially the `inbox-processor` and `add-frontmatter` skills (PARA-method routing, frontmatter as a first-class concern)
- [`kepano/obsidian-skills`](https://github.com/kepano/obsidian-skills) — Obsidian-flavored Markdown discipline (wikilinks, callouts, frontmatter properties); maintained by Stephan Ango (Obsidian CEO)
- The "Iron Law" framing pattern is borrowed from [`Asher-/claude-skills-verification-before-completion`](https://github.com/Asher-/claude-skills-verification-before-completion)

This skill is original code, but the conceptual debt is real and acknowledged.

---

## The bottom line

Capture into the existing vault, never invent. Confirm folder + filename before writing. Frontmatter and wikilinks are not optional. The user's framing beats the assistant's prose. That is the skill.
