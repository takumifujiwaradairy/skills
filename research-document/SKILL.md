---
name: research-document
description: Renders a verified research file (research-verified.md) into a neutral, decision-ready Markdown report for technology selection. Use when the user asks for a final research report, a comparison document, asks to "まとめて" / "ドキュメント化して" after research-verify has run, or explicitly invokes /research-document. Produces docs/research/research-report-<topic>-<YYYYMMDD>.md following a fixed template (overview, comparison table, per-option detail, verification status, sources). Strictly never recommends a winner, never assigns scores or rankings, and never invents missing cells — the human reader makes the decision.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash(mkdir *)
  - Bash(date *)
  - Bash(ls *)
---

# research-document — Final Report Generation

Reads `research-verified.md` and produces a neutral comparison report at `docs/research/research-report-<topic>-<YYYYMMDD>.md`. **This skill provides comparison material only.** It does not recommend, rank, score, or otherwise make decisions on behalf of the user.

## When to run

- After `research-verify` has produced `research-verified.md`
- User asks for a final research report or comparison document
- User asks to "ドキュメントにまとめて" / "レポート化して"
- User explicitly invokes `/research-document`

## Inputs

- `./research-verified.md` (default) or a path supplied by the user
- Topic slug for the filename (derive from the verified file's title; if ambiguous, ask the user for a short kebab-case slug like `neon-vs-supabase`)

## Output

```
docs/research/research-report-<topic-slug>-<YYYYMMDD>.md
```

Setup:

```bash
mkdir -p docs/research
date +%Y%m%d
```

If `docs/` does not fit the project structure (e.g. the cwd is not a project root), fall back to `./research-report-<topic-slug>-<YYYYMMDD>.md` and tell the user where it landed.

If the target file already exists, ask before overwriting.

## IRON RULES (non-negotiable)

These rules override stylistic instincts. Violating them defeats the purpose of the skill.

1. **No recommendations.** Never write "X is better", "X を採用すべき", "おすすめは X", or any equivalent. Strengths and weaknesses are listed in parallel; the reader chooses.
2. **No scores or rankings.** No 5-point ratings, no "総合評価", no "コスパが良い順". Facts only.
3. **No invented cells.** A comparison-table cell with no verified fact must contain literally `情報なし` (and optionally a parenthetical reason like `(URL無効)`, `(矛盾あり)`, `(単一ソースのみ)`).
4. **Every strength / weakness bullet cites a source number.** Format: `- 強み: <fact> [S3]`. The `[Sn]` references the numbered list in section 5.
5. **Verification status is mandatory.** Section 4 must list every flagged item from the verified input. Skipping it hides reliability and is forbidden.
6. **Tone is neutral.** No marketing language, no superlatives, no "shines at", "powerful", "elegant".

If the user explicitly asks for a recommendation, refuse with: "This skill produces comparison material only. The decision belongs to you — happy to highlight specific axes if that helps."

## Workflow

### 1. Read the verified file

Parse `research-verified.md`. Capture:

- The flag summary block (URL 無効 / 矛盾あり / 古い情報 / やや古い / 単一ソースのみ / 日付不明)
- The verified facts grouped by (option, axis)
- The full source list with URLs, dates, primary/secondary classification

### 2. Resolve filename

Topic slug: lowercase, hyphenated, ASCII. Examples: `neon-vs-supabase`, `drizzle-vs-prisma-vs-kysely`. Date: `date +%Y%m%d`.

### 3. Build the report using this exact template

```markdown
# 調査レポート: <調査テーマ>

- 調査日: YYYY-MM-DD
- 調査者: Claude Code (research-search → research-verify → research-document)
- 比較観点: <axis 1>, <axis 2>, ...

## 1. 概要

### <選択肢 A>
<2–3 line factual summary. No adjectives like "powerful" / "lightweight". State what the product is and what it provides.>

### <選択肢 B>
...

## 2. 比較表

| 観点 | <選択肢 A> | <選択肢 B> | <選択肢 C> |
|------|-----------|-----------|-----------|
| 料金 (free tier) | ... [S1] | ... [S4] | 情報なし |
| 料金 (paid 最小プラン) | ... [S1] | ... [S4] | ... [S7] |
| <axis> | ... [Sn] | 情報なし (URL無効) | ... [Sn] |
| ... | ... | ... | ... |

セルに事実が無い場合は必ず「情報なし」と明示する。推測で埋めない。

## 3. 各選択肢の詳細

### <選択肢 A>
- 強み:
  - <fact> [S1]
  - <fact> [S2]
- 弱み・制約:
  - <fact> [S3]
- 注意事項:
  - <fact> [S2]

### <選択肢 B>
...

## 4. 検証ステータス

### 裏付け取得済 (primary-backed, cross-checked)
- <axis>: <option> — <fact> [S1, S5]
- ...

### 単一ソースのみ
- <axis>: <option> — <fact> [S3]
- ...

### 矛盾あり (要再確認)
- <axis>: <option> — Source A says X [S2], Source B says Y [S6]
- ...

### 古い情報 (> 24ヶ月) / やや古い (12–24ヶ月)
- <fact> — source dated YYYY-MM-DD [S4]
- ...

### URL 無効により未検証
- <fact> — original URL [S8]
- ...

## 5. 参照ソース

- [S1] <title or short label> — <URL> (公開日: YYYY-MM-DD, 種別: 一次, URL検証: 済)
- [S2] ... — <URL> (公開日: YYYY-MM-DD, 種別: 二次, URL検証: 済)
- [S3] ... — <URL> (公開日: 不明, 種別: 二次, URL検証: 無効)
- ...
```

### 4. Cross-check the IRON RULES before writing

Before calling Write, scan the drafted document for forbidden phrases:

- 「おすすめ」「推奨」「採用すべき」「ベスト」「優れている」「劣っている」
- 「総合評価」「総合スコア」「ランキング」「順位」
- Adjectives that imply judgment: 「素晴らしい」「強力」「貧弱」

If any are found, rewrite the offending sentence as a neutral fact statement before saving.

### 5. Confirm verification-status section is non-empty

Even if every fact passes verification, write the section with `(該当なし)` under empty subsections — never delete the subsections. The reader needs to see that all categories were considered.

### 6. Save and report

Write the file. Tell the user:

- The full output path
- A 2–3 line summary of what the report contains (number of options, axes, flag counts) — **without** any judgment
- A reminder that the decision is theirs

## Anti-Patterns

| Pattern | Why it fails | Correct behavior |
|---------|--------------|------------------|
| Writing "X is recommended for use case Y" | Decision belongs to the human | List facts; do not conclude |
| Empty cells in the comparison table | Looks like an oversight; misleads the reader | Write `情報なし` literally, optionally with reason |
| Strength bullets without source numbers | Untraceable claims | Every bullet ends with `[Sn]` |
| Skipping section 4 because "everything was verified" | Hides what was checked | Keep all subsections; mark empty ones `(該当なし)` |
| Subjective adjectives ("elegant", "powerful", "lightweight") | Smuggles judgment back in | Use measurable facts ("bundle size: 12KB" instead of "lightweight") |
| Inventing a "総合評価" score because the comparison table looks bare | Violates IRON RULE #2 | Leave the table sparse; "情報なし" is honest |
| Combining sources into a single citation `[S1,S2,...]` for an unsupported claim | Citation-laundering | Each `[Sn]` must independently support that bullet |

## Test case

Input: `research-verified.md` for "Neon vs Supabase 2026" containing ~12 verified facts, 1 contradiction (region availability), 1 dead URL, and 2 single-source pricing facts.

Expected output: `docs/research/research-report-neon-vs-supabase-20260430.md` containing:
- Two `### 概要` subsections, both adjective-free
- A 6–8 row comparison table with explicit `情報なし` cells where appropriate
- Per-option strengths/weaknesses, every bullet ending in `[Sn]`
- A populated section 4 with the contradiction, dead URL, and two single-source items
- A section 5 listing every source with date, type (一次/二次), and URL-validation status
- Zero occurrences of: 「おすすめ」「採用すべき」「総合評価」「ランキング」
