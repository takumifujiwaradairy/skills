---
name: architecture-premise-verify
description: Use this skill BEFORE recommending any architecture, design, or implementation plan that depends on a feature, capability, step, or integration of an external service (Slack, GitHub, AWS, GCP, Cloudflare, Vercel, Stripe, OpenAI, Notion, Zapier, third-party SDKs, SaaS platforms, etc.). It forces external-service feature claims to be web-verified before they are presented to the user as part of a recommendation. TRIGGER when about to say "you can do X with service Y", "service Y has a step/feature called Z", "use Y's built-in Z", or otherwise propose an architecture whose feasibility depends on a specific external feature existing. Skip only for pure in-codebase work, refactors, internal logic, and language/framework primitives that the assistant has direct first-hand knowledge of.
user-invocable: true
allowed-tools:
  - WebSearch
  - WebFetch
  - Read
  - Write
  - Bash(date *)
---

# architecture-premise-verify — Verify external-service features BEFORE recommending architectures

## The Iron Law

```
NO ARCHITECTURE RECOMMENDATION WITHOUT FRESH EXTERNAL EVIDENCE
FOR EVERY EXTERNAL-SERVICE FEATURE IT DEPENDS ON.
```

If a recommendation depends on "service X has feature Y", and you have not produced fresh, in-this-session web evidence that feature Y exists, in the form the recommendation assumes, on the user's plan tier, in the current year — **you may not present that recommendation as viable**.

This is a hard rule. Skipping it is hallucination, not optimism.

---

## Why this skill exists

The expensive failure mode this skill prevents:

1. Assistant proposes an architecture (e.g., "Slack Workflow Builder has a Send Webhook step → use that to call GitHub").
2. User trusts the recommendation, builds the rest of the system around it (e.g., GitHub Actions side, secrets, CI workflow).
3. **At the last integration step**, the assumed feature turns out not to exist (or to require a paid tier the user doesn't have, or to behave differently than assumed).
4. The whole architecture has to be redesigned. Hours wasted, trust eroded.

The root cause is almost never lack of effort downstream — it is **deferred verification of an upstream premise**. This skill moves the verification to the moment the premise is introduced.

---

## When to invoke this skill (auto-trigger conditions)

Invoke this skill **before completing your response** whenever any of the following are true:

- You are about to recommend an architecture, integration, or design that uses an external service.
- You are listing options for the user to choose between, and one or more options claim a specific feature of an external service exists.
- You are answering "can we do X with service Y?" affirmatively.
- You are about to write code, configuration, or a plan that depends on a named feature of an external service.
- The user has shared a constraint ("no external hosting", "free tier only", "must be on AWS") and you are presenting a path that satisfies it via an external service's specific capability.
- You catch yourself writing phrases like:
  - "Y has a built-in step/action/feature called Z"
  - "you can configure Y to send Z"
  - "Y supports Z out of the box"
  - "use Y's Z to ..."
  - "natively / native / out of the box / built-in"

If any of the above applies, **run this skill before sending your response.** Do not rely on memory of "I'm pretty sure it has that".

---

## When NOT to use this skill

Do not invoke this skill for:

- Pure in-codebase work (refactor, bug fix, code review of existing files).
- Language and standard-library primitives the assistant has direct first-hand knowledge of (`Array.map`, `for` loops, basic SQL).
- Stable framework features that have not changed in years and where misuse would cause an obvious local error (`useState`, `useEffect`).
- Internal Yourtory / project-specific logic where authority lives in the local CLAUDE.md, README, or code.
- Cases where the user has already verified the premise and stated so explicitly in this conversation.

When in doubt about whether to skip, **do not skip**. The cost of running this skill is a few seconds of search; the cost of skipping is a redesign.

---

## The process

For every recommendation that triggered this skill, do the following, in order. Do not collapse steps.

### Step 1 — Enumerate (service, feature) pairs

Before writing the recommendation, list every (external service, specific feature) pair the recommendation depends on. Be granular. "GitHub" is not a feature; "GitHub repository_dispatch event with custom client_payload" is.

Example for a Slack→GitHub bot:

| # | Service | Feature claim | Critical? |
|---|---------|---------------|-----------|
| 1 | Slack Workflow Builder | Has a "Send Webhook" / outgoing HTTP step | Critical |
| 2 | Slack Workflow Builder | Emoji reaction can be a workflow trigger | Critical |
| 3 | GitHub Actions | `repository_dispatch` accepts `client_payload` | Critical |
| 4 | GitHub Actions | Workflows on private repos can create PRs with `GITHUB_TOKEN` | Critical |
| 5 | Slack | `conversations.history` returns message by exact `ts` | Supportive |

**Critical** = the recommendation collapses if false. **Supportive** = workaround exists if false.

If you cannot list at least one pair, the recommendation probably does not depend on an external service — re-check whether this skill should fire at all.

#### Step 1 granularity rules

**Decompose absence and "no-X" claims as their own pairs.** Phrases like "no middleware needed", "no external infrastructure", "no auth required", "no polling", "zero ops", "out of the box" are themselves premises — and historically the most likely to be hallucinated. Convert each into an explicit pair:

- "no middleware needed" → `(Service, can deliver payload directly to target without payload transformation)` — Critical
- "no external infrastructure" → `(Service, hosts the trigger, the action, AND the connection between them)` — Critical
- "no polling" → `(Service, supports push / event subscription on this resource)` — Critical
- "no auth setup" → `(Service, accepts requests from target without explicit credential exchange)` — Critical

Treat these "absence" pairs as **the most suspicious** and Heavy-tier them by default.

**Decompose implicit pairs hidden in user constraints.** When the user said "I want X without Y", and your recommendation satisfies it, list at least one pair that explicitly captures *why* this recommendation satisfies the constraint. If you cannot articulate that pair, you have not actually thought about whether the constraint is met.

**Granularity smell test.** Each pair should be specific enough that a `WebSearch` query can confirm or deny it in one round. "GitHub Actions integration" is too vague; "GitHub Actions `repository_dispatch` event accepts arbitrary `client_payload` JSON" is right.

#### Step 1.5 — Disambiguate ambiguous claims into sub-pairs

For every Critical pair, ask: **"Are there multiple plausible implementations of this claim, with different feasibility?"** If yes, you must split it into sub-pairs (1a, 1b, ...) and verify each independently.

Common ambiguity patterns:

| Original claim | Hidden interpretations |
|----------------|------------------------|
| "Service X integrates with Service Y" | (a) direct API call, (b) native action/connector, (c) third-party connector, (d) only via shared SSO — all "integration" but different infra |
| "X sends a webhook to Y" | (a) raw POST to a URL, (b) typed integration where payload format is matched, (c) mediated through a connector that transforms |
| "X supports webhooks" | (a) X *receives* webhooks (incoming), (b) X *sends* webhooks (outgoing) — opposite directions, often confused |
| "X has SSO" | (a) X is the IdP, (b) X is the SP, (c) only OIDC, (d) only SAML, (e) only specific IdPs |
| "X has native Slack integration" | (a) X posts to Slack, (b) Slack posts to X, (c) bidirectional, (d) only OAuth login, (e) only via Slack App Marketplace |
| "no infrastructure needed" | (a) literally zero servers (and payloads/formats happen to align), (b) one platform handles both ends, (c) "no infrastructure *that I have to maintain*" but a hosted intermediary is involved |

**The rule**: if a claim has 2+ plausible interpretations and the recommendation does not explicitly pick one, **split the pair, verify each, and present the live ones to the user** so they pick — or revise the recommendation to disambiguate. Do not silently pick one interpretation and run with it.

The Notion→Slack dry-run made this concrete: "Notion's outgoing webhook to notify Slack — no infrastructure" splits into (a) literal POST to a Slack incoming webhook URL — fails on payload format mismatch — and (b) Notion's native "Send to Slack" automation action — works but is a different feature with different tradeoffs. Without splitting, the recommendation is ambiguous and one of the interpretations is broken.

### Step 2 — Verify each pair (tier appropriate to criticality)

For every Critical pair, do at least the **Medium** tier below. For every Supportive pair, do at least the **Light** tier.

**Light tier** (≈ 30 seconds, 1 search):
- One `WebSearch` query of the form `"{service} {feature} {current year from Bash(date)}"`.
- Read the top 3 results' titles + snippets.
- Pass criterion: at least one result from official docs, vendor blog, or a credible engineering blog explicitly confirms the feature in the form claimed.

**Medium tier** (≈ 1–2 minutes, official docs check):
- Light tier, plus:
- One `WebFetch` against the official documentation page that should describe the feature.
- Read what the page actually says about the feature, not just confirm a URL exists.
- Pass criterion: official docs explicitly describe the feature in the form claimed, with current-year examples or no deprecation notice.

**Heavy tier** (≈ 3–5 minutes, plus implementation evidence):
- Medium tier, plus:
- Find at least one real-world implementation of the **specific combination** you are proposing. The combination must match on:
  - Source service + feature
  - Target service + feature
  - The connecting mechanism (direct, intermediary, transform, etc.)
- A reference implementation counts only if it satisfies **all three** of the following:
  1. **Code or concrete configuration is shown** (not just a marketing claim that "X integrates with Y").
  2. **Dated within the last 24 months** (or the docs are explicitly current — services change).
  3. **From an independent source**: a public GitHub repo with the integration code, an official sample/recipe from one of the two services, or an engineering blog post with full setup steps. Vendor marketing pages and "we add this missing feature" tool ads do **not** count — they are negative evidence (see Anti-pattern #5).

**Search budget:** Spend at most ~5 minutes / 3 distinct search queries trying to find a reference implementation.

**Decision rule on the result of the search:**

| Found | Verdict |
|-------|---------|
| ≥1 reference implementation matching all 3 criteria | ✅ Verified |
| 0 reference implementations, but feature is documented as "supported" in vendor docs | ❓ Unverified — present as conditional, recommend a small spike before committing |
| 0 reference implementations AND ≥1 source explicitly says it does not work / requires workaround | ❌ Falsified |
| 0 reference implementations AND the proposed combination would obviously be popular if it worked (e.g., "free real-time GitHub→Slack with no infra") | ❌ Treat as falsified — silence in the wild for an obviously useful pattern is a strong negative signal, not absence of evidence |

The last row matters most. The Slack-WfB-webhook hallucination would have been caught by it: a free real-time Slack-reaction-to-anything bridge with no hosting would be on every "automation tips" blog if it worked. None of them describe it.

Use Heavy tier whenever the recommendation perfectly satisfies a strong user constraint that "feels too clean" — see Anti-pattern #2 below.

### Step 3 — Confirm plan / pricing constraints

For each verified feature, also confirm:
- The feature is available on the user's plan tier (Free / Pro / Business+ / Enterprise / per-seat / per-project).
- The feature is generally available, not in private beta or experimental.
- The feature is not deprecated or scheduled for removal.
- Region / locale restrictions, if any, do not apply to the user.

If you do not know the user's plan, **ask** rather than assume.

### Step 4 — Annotate

For every (service, feature) pair, attach a status flag in the response you give the user:

- ✅ **VERIFIED** — Web evidence collected this session. Include the source URL inline.
- ❓ **UNVERIFIED** — Could not confirm in this session. Mark the recommendation as conditional.
- ❌ **DOES NOT EXIST** / **NOT AVAILABLE** — Confirmed missing or unavailable on user's plan. Recommendation must be revised before being presented.

Do not fold these flags into a footnote. Put them inline next to the claim, so the user reads "Slack Workflow Builder Send Webhook step ✅" or "Slack Workflow Builder Send Webhook step ❌ (does not exist as native step in 2025; requires third-party Slack app or own webhook receiver)".

### Step 5 — Block on any Critical ❌

If any **Critical** pair is ❌, you may not present the recommendation as a viable path. Either:
- Remove the option from the comparison table entirely, with a one-line note on why.
- Replace it with a redesigned option that does not depend on the falsified premise.
- Surface the contradiction to the user explicitly and propose alternatives.

If any **Critical** pair is ❓, present the recommendation as **conditional**, explicitly stating that "this option assumes feature X exists; we should verify before committing."

If only **Supportive** pairs are ❓ or ❌, you may proceed but must list the workaround you would take when the supportive feature is absent.

---

## Anti-patterns and red flags

### Anti-pattern 1 — "Other tools have this, so this one must too"

The single most common cause of architecture-premise hallucinations. "Send a webhook" exists in Zapier, Make, n8n, IFTTT, etc., so it must exist in Slack Workflow Builder. **It does not.**

Whenever you find yourself reasoning "tool family X has feature Y, so this specific tool also has Y", treat it as a red flag and verify, not as a green light.

### Anti-pattern 2 — Too-good-to-be-true match against a strong user constraint

When the user has said "I want X but without Y" (e.g., "real-time but no external hosting"), and you produce a recommendation that perfectly satisfies the constraint via a single neat trick — **search for who else is doing that trick**. If the answer is "nobody", the trick almost certainly does not work; you have invented it.

The honest framing for the user is: "I cannot find anyone doing this combination, which usually means it isn't supported. The realistic options are X, Y, Z."

### Anti-pattern 3 — Verification deferred to implementation time

Saying "we'll figure out the Slack side when we get there" is the failure mode this skill exists to prevent. The cheapest moment to discover a missing feature is **before the surrounding code is written**. The most expensive moment is after.

If you find yourself wanting to defer verification, **do the verification now instead**. It is faster than you think.

### Anti-pattern 4 — Trusting your training data on third-party features

Your training data is a snapshot. External services add and remove features constantly. "I'm pretty sure Slack added that step in 2024" is not evidence; **the docs page, fetched in this session, is evidence**.

This applies most strongly to:
- Newer SaaS platforms (anything where features change quarterly)
- Enterprise tier features (often gated and undocumented in public training data)
- Recently deprecated features (your training data may still describe them as current)

### Anti-pattern 5 — Confirmation bias from a single search hit

One blog post saying "you can do this" is not enough if it is unofficial, undated, or written by someone selling a workaround. Prefer official docs. If the only source is a third-party tool's marketing page advertising "we add this missing feature to Slack", that is **negative evidence** — it confirms the feature is missing from the platform itself.

### Anti-pattern 6 — Skipping when "it's just a small thing"

The cheapest features to misremember are the ones that look standard. The most expensive bugs are the ones that look obvious. Run the skill anyway.

---

## Phrases that should make you stop and verify

If any of these are about to appear in your response and the surrounding subject is an external service, you must run this skill first:

- "natively supports"
- "out of the box"
- "built-in step / action / feature"
- "you can just"
- "all you have to do is"
- "Y has a webhook / API / integration for that"
- "this is a standard pattern"
- "obviously you can"
- "simply configure"

These phrases are linguistic signatures of unverified premises.

---

## Output format

When the skill has run, the response to the user should include, near the recommended architecture:

```markdown
**Premise verification (architecture-premise-verify):**

| # | Premise | Source | Status |
|---|---------|--------|--------|
| 1 | Slack Workflow Builder has native "Send Webhook" step | https://… (fetched) | ❌ Does not exist (2025) |
| 2 | Slack `reaction_added` event API delivers channel + message ts | https://api.slack.com/events/reaction_added | ✅ Verified |
| 3 | GitHub Actions `repository_dispatch` accepts `client_payload` | https://docs.github.com/… | ✅ Verified |

Critical-status check: 1 ❌, recommendation revised below.
```

Then present the actual revised recommendation, with the falsified premise removed or replaced.

If everything verifies green, the table is still included so the user can see what was checked.

---

## Worked example — what this skill would have caught

**Original (hallucinated) recommendation:**
> Use Slack Workflow Builder's emoji-reaction trigger and its built-in Send Webhook step to call GitHub `repository_dispatch`. No external hosting needed.

**With this skill applied:**

Step 1 — Pairs:
1. Slack Workflow Builder: emoji-reaction trigger → Critical
2. Slack Workflow Builder: built-in Send Webhook step → **Critical**
3. GitHub Actions: `repository_dispatch` → Critical

Step 2 — Verification (Heavy tier, since matches "no external hosting" constraint perfectly):
- Pair 1: ✅ Verified, official docs.
- Pair 2: ❌ **No native Send Webhook step exists in Slack Workflow Builder** (Building on Slack docs explicitly say so; only third-party apps like Workflow Buddy provide it). 0 official sources, 0 reference implementations, multiple sources confirming absence.
- Pair 3: ✅ Verified.

Step 5 — Block: Pair 2 is Critical and ❌. Recommendation cannot be presented as "no external hosting needed". Revise to options that include a webhook receiver (Cloudflare Worker, GAS, Lambda) or polling, and present those honestly with their tradeoffs.

This is what should have happened. It is what this skill exists to make happen automatically next time.

---

## Interaction with other skills and modes

- **Plan mode** (`EnterPlanMode`): if plan mode is active and the plan touches external services, this skill must run before the plan is presented to the user.
- **dig review / Plan agent**: dig is for finding edge cases in a plan whose premises are accepted. This skill is upstream of dig; it audits the premises themselves. Run this skill first; then run dig on the survivors.
- **research-search / research-verify**: those are for technology selection. This skill is for architecture validation. Different layer; use both when both apply.

---

## The bottom line

Before you say "use service Y's feature Z", search for Z, in the current year, on the user's plan. Then say it.

Run the search. Read the docs. Then claim the feature exists. Non-negotiable.
