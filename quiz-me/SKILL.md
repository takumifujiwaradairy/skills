---
name: quiz-me
description: Active-recall quiz to verify the user's understanding of a topic they have just been taught (or one they explicitly request). Use when the user explicitly asks "quiz me", "test me", "クイズして", "理解度を確かめたい", "私が分かっているか確認して", or invokes `/quiz-me [topic]`. Generates 5 multiple-choice questions (mixed conceptual / applied / boundary), asks them one at a time via AskUserQuestion, gives instant feedback with the correct answer plus a one-line explanation, and ends with a gap analysis recommending what to review next. Source of the quiz: an explicit topic argument, a file path, or — by default — the recent conversation context. SKIP for codebase quizzes (use a codebase-specific quiz skill), course-specific phase quizzes, or anything that should run unattended without the user answering.
user-invocable: true
allowed-tools:
  - AskUserQuestion
  - Read
  - WebSearch
  - WebFetch
  - Bash(date *)
---

# quiz-me — Verify the user's understanding via active recall

## The Iron Law

```
QUIZ THE *USER*, NOT THE MODEL.
QUESTIONS MUST FORCE RECALL, NOT RECOGNITION.
ALWAYS USE AskUserQuestion. NEVER AUTO-ANSWER ON THE USER'S BEHALF.
```

If you draft a question and immediately see the answer in the way you wrote the choices, the question fails. Rewrite it.

If you find yourself answering the question yourself in narration ("the answer is C because…"), you are no longer running the skill. The skill *is* the user answering.

---

## Why this skill exists

Reading and being told something is not the same as being able to recall and apply it. The cognitive science term is **active recall**: retrieval is what consolidates memory, not re-reading. After the user has just been taught a topic (or studied something on their own), the cheapest way to find out what actually stuck is to ask them — in a forced-choice format that prevents the "yeah I get it" illusion.

This skill exists to make that ask **structured, fast (≤5 minutes), and honest about gaps**, so the next learning step is informed rather than guessed.

---

## When to invoke

Auto-trigger when the user says any of:

- "quiz me", "test me", "check my understanding"
- "クイズして", "理解度を確かめたい", "理解度チェックして"
- "私が分かっているか確認して"
- "今の話、私が分かっているか試して"
- explicit `/quiz-me [topic]` invocation

Trigger conditions where it is *also* appropriate but the user did not ask:

- The user has just been taught a non-trivial topic in this conversation and pauses with "なるほど" / "OK" / "got it" — **offer**: "Want me to quiz you on that to make sure it stuck? (`/quiz-me`)" Do not auto-run.

---

## When NOT to use

- Codebase comprehension quizzes — use a codebase-specific quiz skill (e.g. `willnordnet/quiz`).
- Course-specific phase quizzes with a fixed curriculum — use that course's skill.
- Anything that needs to run without the user being available to answer (this skill is interactive by design).
- Single-fact lookups ("what's the syntax of X?") — that is a Q&A, not a quiz.

---

## Inputs

The skill resolves its quiz source in this priority order:

1. **Explicit topic argument** — `/quiz-me Cloudflare Workers` → quiz on that topic.
2. **File path argument** — `/quiz-me ./path/to/note.md` → read the file, quiz on its content.
3. **No argument** — quiz on the **last meaningful explanation in this conversation**. Identify the topic by scanning back to the most recent block where the assistant taught a concept (definitions, comparisons, how-to). If the conversation has multiple topics, pick the most recent one and tell the user "quizzing on `<topic>` (the last thing we discussed) — switch with `/quiz-me <other-topic>` if you meant something else."

If a topic argument names something the assistant cannot teach with confidence, do a quick `WebSearch` to ground the quiz in real, current information — do not invent facts to ask about.

---

## The procedure

### Step 1 — Identify the topic + source material

State to the user, in one sentence, what topic you will quiz them on and where the material comes from. Examples:

- "Quizzing on **Cloudflare Workers** based on the explanation earlier in this conversation."
- "Quizzing on **GitHub Secrets の3種類** based on `~/Documents/my_notes/02_ナレッジ/02_ITスキル/GitHub_Secrets_3種類の使い分け.md`."

If the user did not explicitly ask, **wait for confirmation** before generating questions. If they did, proceed.

### Step 2 — Generate 5 questions

Generate exactly 5 questions, in this fixed mix:

| # | Type | What it tests |
|---|------|---------------|
| Q1 | **Conceptual (definition)** | "What *is* X?" — can the user state what X means? |
| Q2 | **Conceptual (why)** | "Why does X matter / why does Y happen?" — reasoning, not just naming |
| Q3 | **Applied (how)** | "How would you use X in situation S?" — recognition of correct application |
| Q4 | **Applied (when)** | "When is X the right choice vs alternative Y?" — judgment, not memorization |
| Q5 | **Boundary / negative** | "When does X *not* apply / what's a common mistake / what is X *not*?" — tests where the concept's limits are |

#### Question quality rules (non-negotiable)

- **4 choices each**, exactly 1 correct. Multi-correct or "all of the above" is forbidden — it lets the user dodge commitment.
- **Distractors must be plausible**, not obviously silly. The best distractors come from **common misconceptions** about the topic. If you cannot name a real misconception, the distractor is too easy.
- **No verbatim regurgitation of source phrasing**. Paraphrase. If the user can win by pattern-matching the question wording to a sentence they just read, the question tests reading speed, not understanding.
- **One concept per question**. If you find yourself writing "and / also / additionally" in the stem, split it.
- **No yes/no, no true/false** — those are 50/50 guesses.
- **Stay grounded in the source material**. Do not invent edge cases the user could not have encountered in the source.

### Step 3 — Ask one question at a time, with AskUserQuestion

For Q1 through Q5:

1. Present the question stem and 4 choices using **AskUserQuestion** (one question per call, four `options`). Always include `multiSelect: false`. Each option's `label` should be ≤12 words; put the longer answer text in `description` if needed.
2. Wait for the user's answer.
3. Respond immediately with:
   - ✅ if correct, or ❌ if incorrect (or if they typed "Other" with a free-text answer that does not match)
   - **The correct answer** (always, including on correct responses — reinforces)
   - **A one-line explanation** of *why* the correct answer is right, and (if the user picked a distractor) why the distractor is wrong
4. Do not pile on. No follow-up questions on the same item. Move on.

Track per-question results internally: `correct: bool`, `picked: <choice>`, `topic-area: <Q1-conceptual / Q2-why / Q3-how / Q4-when / Q5-boundary>`.

### Step 4 — Scoreboard + gap analysis

After Q5, present:

```markdown
## Quiz result

**Score: X / 5**

| Q | Type | Result | What it tested |
|---|------|--------|----------------|
| 1 | Definition | ✅/❌ | … |
| 2 | Why | ✅/❌ | … |
| 3 | How | ✅/❌ | … |
| 4 | When | ✅/❌ | … |
| 5 | Boundary | ✅/❌ | … |

### Strengths
- (one or two sentences naming what the user clearly understood)

### Gaps to revisit
- (specific, named — not "you should review more". Bad: "review Cloudflare Workers". Good: "the boundary between Workers and traditional servers — review the 'when NOT to use Workers' section.")

### Recommended next step
- Concrete action: a doc to re-read, a follow-up topic, or "quiz again in 2 days" (active recall benefits from spacing).
```

If 5/5: congratulate briefly, suggest a harder follow-up topic or a Feynman-style explanation as the next test ("explain it back to me without notes — that's the next level").

If 0/5 or 1/5: **do not pile on**. Acknowledge the topic was harder than the user expected, recommend a single concrete re-read, and offer to re-quiz after.

---

## Anti-patterns

### Anti-pattern 1 — Recognition disguised as recall
"Which of these is the definition of X?" with the textbook definition as one of four choices is **recognition**, not recall. The user just matches a phrase. Better: "If a colleague asked you what X does, which of these is the most accurate one-sentence answer?" — same surface, harder cognitive task because the choices are paraphrases.

### Anti-pattern 2 — Trick questions / pedantry
Questions that hinge on a specific number, exact API name, or case-sensitive detail test memorization, not understanding. Skip them unless the topic is itself a syntax topic. The user did not ask to be tricked; they asked to find out if they understand.

### Anti-pattern 3 — Auto-answering / narrating
"Question 1: …. The answer is B because …." — this is not a quiz, this is the assistant talking to itself. The skill *requires* `AskUserQuestion`; if the tool is unavailable, **say so and abort** rather than fake the interaction.

### Anti-pattern 4 — Source-phrasing parroting
If the source said "Cloudflare Workers run on V8 isolates", a question asking "What does Cloudflare Workers run on?" with "V8 isolates" as a choice tests scrolling, not understanding. Paraphrase: "What lets a Cloudflare Worker start in milliseconds rather than seconds, the way a typical Node container would?"

### Anti-pattern 5 — Skipping the gap analysis
A score with no diagnosis ("you got 3/5, good job") is junk. The user invoked the skill to find out *what to learn next*. Always end with a named gap and a concrete next step. If the user got 5/5, the next step is a harder challenge, not silence.

### Anti-pattern 6 — Asking too many questions
The skill is fixed at **5**. Not 8, not 10. More questions invite fatigue, lower-quality distractors, and lower the per-question signal. If the user wants more, they re-invoke `/quiz-me` on a related topic.

---

## Phrases that should make you stop

If you are about to write any of these inside a question stem, stop and rewrite:

- "Which of the following is true / correct / accurate"  — generic, forces choice-elimination instead of recall
- "All of the above" / "None of the above" — forbidden distractors
- "Which option **best** describes …"  — soft framing that lets the user defer judgment
- "What did the article / explanation say …" — tests reading, not understanding

Replace with concrete situational stems: "If you were doing X, which approach would you reach for first?"

---

## Output format example

```markdown
Quizzing on **Cloudflare Workers** based on the conversation just now. 5 questions, mixed conceptual + applied + boundary.

[AskUserQuestion #1]
Q1. A teammate asks "what's the simplest mental model for Cloudflare Workers?" Which is closest?
- A) A small Linux VM that runs your code on demand
- B) Code that runs on edge nodes only when a request hits, no idle billing
- C) A managed Docker container that auto-scales
- D) A scheduled cron job that polls for events

(... user answers ...)

✅ Correct — B.
"Code that runs on edge nodes only when called, no idle billing" captures the V8-isolate, scale-to-zero edge model. A and C describe container/VM platforms (long-running, billed even when idle). D is a polling worker, not the request-driven model.
```

---

## Interaction with other skills

- **`empirical-prompt-tuning`**: when this skill itself feels off (questions too easy, gap analysis too vague), use `empirical-prompt-tuning` to surface the ambiguity and tighten the rubric.
- **Feynman-style "explain it back"**: complementary. After 5/5 on `quiz-me`, the natural next step is "explain X to me without notes" — recall + production, the harder test.
- **Spaced repetition**: out of scope here. This skill produces a single-session score; pair it with a separate spacing tool if needed.

---

## The bottom line

Ask 5 forced-choice questions the user can answer in under 5 minutes. Use distractors that look like real misconceptions. Diagnose the gap. Recommend one concrete next step. Then stop.

That is the skill. Anything beyond that is bloat.
