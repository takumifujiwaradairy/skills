---
name: research-search
description: Collects up-to-date information from the web for technology-selection research. Use when the user asks to compare libraries/SaaS/infrastructure (e.g. "Neon vs Supabase", "ORM 比較", "技術選定の調査", "調査して"), needs current pricing/feature/version facts that may post-date training data, or explicitly invokes /research-search. Performs web search, fetches official docs and GitHub repositories, and writes findings with source URLs to research-raw.md. Always includes the current year in queries to avoid stale results and prefers primary sources (official docs, official blog, official GitHub) over personal blogs.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - WebSearch
  - WebFetch
  - Bash(date *)
---

# research-search — Information Collection for Technology Selection

Collects facts from the web for a technology comparison and writes them to `research-raw.md` in the current working directory. **This skill only collects raw material.** It does not verify, judge, or recommend — that is the job of `research-verify` and `research-document`.

## When to run

- User asks to compare libraries / SaaS / infrastructure (Neon vs Supabase, Drizzle vs Prisma, etc.)
- User asks for current pricing, feature support, or version status of a technology
- User explicitly invokes `/research-search`
- User asks any question whose correct answer depends on facts that change over time (pricing, deprecations, new features, latest stable version)

## Inputs

Ask the user (or accept from arguments) before starting:

1. **Topic** — what is being compared. Example: "PostgreSQL hosting for a Vercel-deployed Next.js app".
2. **Options** — concrete candidates. Example: Neon, Supabase, Cloud SQL.
3. **Comparison axes** — what dimensions matter. Example: pricing, latency, pgvector support, region availability, connection pooling behavior.

If any of these are missing, ask before searching. Vague topics produce useless searches.

## Output

A single file at `./research-raw.md` (current working directory).

If the file already exists, **stop and ask** the user whether to overwrite, append, or write to a different filename. Do not silently overwrite.

## Workflow

### 1. Confirm today's date

```bash
date +%Y-%m-%d
```

Use the result for query construction and for `取得日時` stamps. Do not rely on memorized dates.

### 2. Construct queries

Every search query must include the current year (e.g. `2026`). Examples:

- `Neon Postgres pricing 2026`
- `Supabase free tier limits 2026`
- `Drizzle ORM vs Prisma performance benchmark 2026`

Run **at least one query per option × axis combination**, plus one broad comparison query per pair of options. Prefer English queries — English documentation is usually fresher than translated versions.

### 3. Prioritize sources

Process results in this order:

1. **Official documentation** — `docs.<vendor>.com`, `<vendor>.com/docs`
2. **Official blog / changelog** — for recent feature changes and deprecations
3. **Official GitHub repository** — README, releases, recent issues
4. **Reputable benchmarks** — vendor-neutral benchmarks with reproducible methodology
5. **Personal blogs / Qiita / Zenn / Medium** — only as supplementary context. If a personal blog contradicts official docs, official docs win.

### 4. For each candidate technology, fetch the GitHub repository

Use WebFetch on `https://github.com/<owner>/<repo>` and capture:

- Star count, fork count
- Date of last commit on default branch (proxy for maintenance health)
- Number of open issues / pull requests
- Date and tag of the most recent release (from `/releases`)
- Any obvious red flags pinned in README or top issues (security advisories, "this project is unmaintained" notices)

If a candidate has no GitHub repo (proprietary SaaS), skip this step and note `GitHub: N/A (proprietary)` in the source section.

### 5. For each official docs page, extract

- Feature support and explicit limitations
- Pricing tiers and quotas (with the page's own date if shown)
- Deprecation notices and breaking-change announcements
- Supported runtime / language / region constraints

### 6. Write `research-raw.md`

Use this exact structure:

```markdown
# research-raw: <topic>

- 取得日時: YYYY-MM-DD HH:MM (timezone)
- 調査対象: <topic>
- 候補: <option A>, <option B>, <option C>
- 比較観点: <axis 1>, <axis 2>, ...

---

## Source: <full URL>
- 種別: official docs | official blog | official GitHub | benchmark | personal blog | other
- 公開日 / 最終更新日: YYYY-MM-DD (or "不明")
- 取得日時: YYYY-MM-DD HH:MM
- 関連候補: <option name(s)>
- 関連観点: <axis name(s)>

### 要約
- (paraphrased fact 1)
- (paraphrased fact 2)
- ...

### 直接引用 (最小限)
> (only if a specific phrase matters — keep under ~25 words)

---

## Source: <next URL>
...
```

**Every fact must live under a `## Source:` block.** A claim without a source block is forbidden. If the same fact is supported by three sources, it gets three blocks.

### 7. Stop here

Do not produce a comparison table, recommendation, or summary. The next skill (`research-verify`) consumes this raw file. Tell the user the file path and suggest running `/research-verify` next.

## Anti-Patterns

| Pattern | Why it fails | Correct behavior |
|---------|--------------|------------------|
| Answering from training data without searching | Knowledge cutoff makes pricing / version facts wrong | Always run WebSearch + WebFetch. Even "well-known" facts go stale. |
| Inventing plausible URLs | LLMs fabricate URLs that look real | Only record URLs that came back from actual search results or were fetched successfully |
| Concluding from a single personal blog | Conflicts with official docs are common | Always also fetch the matching official docs page; let primary win on conflict |
| Omitting the year from queries | Returns ancient blog posts | Every query includes the current year |
| Long verbatim quotes | Copyright risk, citation rot | Paraphrase. Direct quotes only for short, decisive phrases |
| Editorializing in summaries ("Neon is better at...") | This skill does not compare or judge | Record facts only. Comparison is `research-document`'s job |
| Filling unknowns with educated guesses | Pollutes the verified output downstream | Write `情報なし (要追加調査)` literally |

## Test case

Run this to sanity-check the skill end-to-end:

> Topic: "Neon vs Supabase for a Vercel-hosted Next.js 16 app, 2026"
> Options: Neon, Supabase
> Axes: free-tier limits, connection pooler behavior, pgvector support, region availability in Asia, pricing for ~10GB DB

Expected: a `research-raw.md` with at least 6–10 `## Source:` blocks, mixing official docs, official blogs, and at most a couple of secondary sources, every block dated and URL-tagged.
