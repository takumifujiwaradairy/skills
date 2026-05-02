---
name: research-verify
description: Verifies the trustworthiness of research notes produced by research-search. Use when the user asks to verify / cross-check / audit a research-raw.md file, when research-search has just finished, when the user asks "is this research reliable?", or when explicitly invoked via /research-verify. Cross-checks facts across multiple sources, flags contradictions, validates publication dates against today's date, and confirms every source URL actually resolves via WebFetch. Writes the result to research-verified.md with per-fact metadata (primary/secondary, date freshness, cross-check status, URL liveness) and a flag summary at the top.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - WebFetch
  - Bash(date *)
---

# research-verify — Verification of Research Notes

Reads `research-raw.md`, checks every fact and URL, and writes `research-verified.md` with explicit reliability metadata. **This skill only verifies.** It does not produce final reports or recommendations.

## When to run

- After `research-search` has produced `research-raw.md`
- User asks to verify, cross-check, or audit collected research
- User asks "are these facts still current?" about an existing research file
- User explicitly invokes `/research-verify`

## Inputs

- `./research-raw.md` (default) or a path supplied by the user
- If the file does not exist, stop and tell the user to run `/research-search` first

## Output

`./research-verified.md` (same directory as the input). Overwrite confirmation rule applies: if the file exists, ask before replacing.

## Workflow

### 1. Read the raw file and today's date

```bash
date +%Y-%m-%d
```

Parse `research-raw.md` into a list of `Source` blocks. For each block extract: URL, source type, publication date, related options, related axes, summarized facts.

### 2. URL liveness check (every URL, no exceptions)

For each `## Source: <URL>` in the raw file:

1. Run `WebFetch` on the URL.
2. Classify the result:
   - **Live** — fetch succeeded and content matches the original topic
   - **Redirected** — fetch succeeded but final URL or content is unrelated → flag `URL drift`
   - **404 / dead** — flag `URL dead`
   - **Paywall / login wall** — flag `URL gated` (cannot be cross-checked)
3. For shortener / tracking URLs, follow until the final destination and record both.

A URL that fails this check causes every fact attached to it to drop to `未検証`.

### 3. Date freshness check

For each source's publication / last-updated date, compute age relative to today:

| Age | Flag | Treatment |
|-----|------|-----------|
| ≤ 12 months | `fresh` | Use as-is |
| 12–24 months | `aging` | Note "may be outdated, re-verify if pricing/feature-related" |
| > 24 months | `stale` | Demote to historical context; do not use as current fact |
| Unknown | `date-unknown` | Treat as `aging` and note explicitly |

Pricing, free-tier limits, and feature-flag states are **especially volatile**. Apply one tier stricter to those (e.g. an `aging` pricing fact gets the `stale` treatment).

### 4. Cross-check facts across sources

Group facts by (option, axis). For each group:

- **≥ 2 sources agree, at least one is primary** → `cross-checked, primary-backed`
- **≥ 2 sources agree, all secondary** → `cross-checked, secondary-only`
- **Only 1 source** → `single-source`
- **Sources disagree** → `contradiction`. Record both claims verbatim with their source URLs. Do not pick a winner — except note "primary source disagrees with secondary" if applicable, since primary supersedes secondary by policy.

### 5. Primary vs secondary classification

Re-classify each source from the raw file:

- **Primary** — official docs, official blog, official GitHub README/release notes, official changelog
- **Secondary** — third-party blogs, Qiita / Zenn / Medium, aggregator sites, AI-summarized content
- **Benchmark** — independently published benchmarks; classify primary if the methodology is published and reproducible, otherwise secondary

A fact is `primary-backed` if at least one of its supporting sources is primary.

### 6. Write `research-verified.md`

Structure:

```markdown
# research-verified: <topic>

- 検証日時: YYYY-MM-DD HH:MM
- 入力ファイル: research-raw.md
- 総ソース数: N (live: X, dead/drift: Y, gated: Z)

## ⚠ Flag summary (read first)

### URL 無効
- <fact summary> — source: <URL> (404)
- ...

### 矛盾あり
- <axis>: <option> — Source A says X (<URL>), Source B says Y (<URL>). Primary: <which side, or "neither">
- ...

### 古い情報 (> 24ヶ月)
- <fact summary> — source dated YYYY-MM-DD
- ...

### やや古い (12–24ヶ月)
- <fact summary> — source dated YYYY-MM-DD
- ...

### 単一ソースのみ
- <fact summary> — only source: <URL>
- ...

### 日付不明
- <fact summary> — source: <URL>
- ...

---

## Verified facts

### <Option A> × <Axis 1>
- **Fact**: <paraphrased fact>
- **Sources**:
  - <URL> — primary, fresh (2026-02-10), live
  - <URL> — secondary, aging (2025-04-05), live
- **Status**: cross-checked, primary-backed

### <Option A> × <Axis 2>
...
```

The flag summary at the top is mandatory. The downstream `research-document` skill reads it to decide which cells to mark "情報なし" or "要再確認".

### 7. Stop here

Do not write a comparison table or recommendation. Tell the user the output path and suggest `/research-document` next.

## Anti-Patterns

| Pattern | Why it fails | Correct behavior |
|---------|--------------|------------------|
| Skipping URL fetches because the URL "looks fine" | LLMs fabricate URLs and the raw file may contain hallucinations | Fetch every URL with WebFetch, no exceptions |
| Marking single-source facts as `cross-checked` | Defeats the purpose of verification | Single-source stays `single-source` even if the source is the official vendor |
| Treating date-unknown as fresh | Stale facts slip through | Treat date-unknown as at least `aging` |
| Resolving a contradiction by picking the more popular source | Popularity ≠ correctness | Record both claims; let the human decide. Note primary-vs-secondary if applicable |
| Trusting "official docs" without checking publication date | Vendor docs sometimes lag behind actual product state | Apply the same date check to primary sources |
| Silently dropping facts whose URL 404s | Hides verification gaps | Promote them to the `URL 無効` flag list explicitly |
| Editorializing in fact summaries | This skill does not judge | Paraphrase neutrally; flags carry the judgment |

## Test case

Input: a `research-raw.md` for "Neon vs Supabase 2026" with ~8 `## Source:` blocks (mix of official + personal blogs, one deliberately dead URL, one 2023 blog).

Expected output:
- The dead URL surfaces under `URL 無効`
- The 2023 blog surfaces under `古い情報`
- Pricing facts that only appear in one personal blog surface under `単一ソースのみ`
- Free-tier limits that match between official docs and a recent benchmark show as `cross-checked, primary-backed`
