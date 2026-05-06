---
name: check_english
description: Check the user's English text against IELTS Band 7 standards (C1, "good user"). Use when the user invokes /check_english, pastes English and asks for a review, or explicitly asks to "check my English". Targets the user's stated IELTS 7.0 goal — flags Japanese-influenced patterns, suggests more precise / idiomatic alternatives, and avoids over-correction of stylistic choices.
user-invocable: true
---

# check_english — IELTS 7.0 (C1) targeted English review

## The user's goal

The user is a Japanese native speaker preparing for **IELTS Band 7.0** (CEFR C1, "Good user"). At Band 7 the IELTS descriptors expect:

- **Lexical Resource**: sufficient flexibility to discuss a variety of topics; some less common vocabulary; awareness of style and collocation; occasional inaccuracies in word choice.
- **Grammatical Range and Accuracy**: a variety of complex structures with frequent error-free sentences; good control of grammar despite occasional errors.
- **Coherence and Cohesion** (Writing) / **Fluency and Coherence** (Speaking): logical organisation, clear progression, range of cohesive devices used appropriately, though sometimes overused or underused.

The skill calibrates to that level: not pedantic about every micro-error, but firm about patterns that would keep a writer at Band 6 or below.

## When to invoke

- User explicitly types `/check_english`.
- User pastes a chunk of English (Slack message, PR comment, email draft, IELTS practice essay, etc.) and asks "check this" / "is this natural" / "英語チェックして".
- User asks "does this make sense?" about an English sentence they wrote.

Do NOT invoke when the user is asking about *English-language documentation* of a tool, or quoting English from a third-party source — only when the text is *the user's own production*.

## Output structure

Always produce in this order:

### 1. Verdict (one line)

Pick the closest:

- **Band 7+** — natural, flexible, minor polish only.
- **Band 6.5** — communicates clearly, but specific patterns are pulling it below 7. Listed below.
- **Band 6 or below** — meaning is preserved but multiple Band-7 criteria are missed; suggested rewrite included.
- **Unclear** — meaning ambiguous; ask one clarifying question before reviewing.

### 2. Issues (bulleted)

Each bullet:

> `original phrase` → `suggested phrase` — *one-line why* (which IELTS criterion it touches).

Group by criterion if there are 5+ issues:
- **Lexical Resource** (word choice, collocation, register, idiom)
- **Grammatical Range and Accuracy** (tense, articles, agreement, sentence structure)
- **Coherence and Cohesion** (linking words, paragraph flow — only if text is 3+ sentences)
- **Japanese-influenced pattern** (literal translation artefacts; flag separately so the user can build awareness)

### 3. Suggested rewrite (only if Band 6.5 or below)

Provide a single revised version that hits Band 7. Keep the user's voice and intent — don't replace their content with your own.

### 4. One-line "what to practise" takeaway

A pointer for next time, e.g. *"Practise article use with abstract nouns (e.g., 'consider impact' → 'consider the impact')"* or *"Vary sentence openers — three sentences in a row started with 'I think'"*. Maximum one item per check, so it's actionable.

## Patterns to flag specifically

These are the **high-leverage** issues for Japanese learners aiming at Band 7. Watch for:

### Japanese-influenced patterns (flag explicitly)

- **Topic-marker leakage**: "About this issue, I think..." (← 〜について); prefer "On this issue," / "As for X," / make it the subject.
- **Article omission with abstract nouns**: "consider impact" → "consider the impact"; "in same way" → "in the same way".
- **Overuse of "I think"**: every paragraph starting with "I think" / "I feel" — mix in "It seems / It appears / Arguably / In my view / Based on X".
- **Literal translation of 〜と思う / 〜と考える**: "I think that the system is good" sounds B1-ish; Band 7 prefers "The system appears effective" / "Arguably, the system is well designed".
- **〜することができる → "can do"**: "I can do the implementation" → "I can implement it" (avoid noun-heavy "do" phrases).
- **Double subject from は/が**: "This system, it works well" → "This system works well".
- **Wrong preposition collocation**: "discuss about" → "discuss" (no preposition); "consist with" → "consist of" / "be consistent with"; "different than" (US) vs "different from" (UK/IELTS).
- **Overly literal "very"**: at Band 7, prefer specific intensifiers — "extremely / particularly / remarkably / strikingly" — depending on register.

### General Band-7 lift opportunities

- **Sentence-structure variety**: if 5+ sentences are all simple SVO, suggest combining with relative clauses, participles, or subordinators.
- **Cohesive device variety**: if every paragraph starts with "Also" or "And", suggest "Furthermore / In addition / Moreover / What's more" depending on register.
- **Word-form precision**: "make a decision" is fine, but "decide" is tighter; "give an explanation" → "explain". IELTS Band 7 rewards precision over circumlocution.
- **Collocation fixing**: "do a mistake" → "make a mistake"; "strong rain" → "heavy rain"; "high price" → "high price" ✓ but "expensive price" ✗.

## What NOT to flag

- **British vs American spelling/phrasing** — IELTS accepts both consistently. Match what the user is writing.
- **Stylistic choices in informal Slack/chat** — don't formalise "Cheers" or "Got it" if the context is casual.
- **The user's voice** — if they prefer short punchy sentences, don't add Latinate connectives just to inflate complexity.
- **Already-correct phrasing** — don't suggest changes for the sake of producing output. Saying "this is already Band 7" is a valid response.
- **Punctuation in chat contexts** unless it changes meaning.

## Tone

- Direct, specific, brief. The user is an engineer — they want signal, not English-class essays.
- Use the criteria names (Lexical Resource, etc.) so the user builds the IELTS framework mental model alongside the corrections.
- If the text is already Band 7+, say so — don't manufacture issues.
- Never lecture. One short "why" per item, then move on.

## Calibration examples

**Input**: "About the new feature, I think it is very useful for users."

**Output**:
- Verdict: **Band 6.5** — communicates clearly, but three patterns pull it below 7.
- Issues:
  - `About the new feature, I think it is` → `The new feature is` — *Lexical Resource: Japanese-influenced topic-marker ("〜について"); Band 7 prefers direct subjects.*
  - `very useful` → `particularly useful` / `genuinely valuable` — *Lexical Resource: "very" + adjective is a Band 6 marker; vary intensifiers.*
- Rewrite: "The new feature is particularly useful for users."
- Practise: Drop "About X, I think..." openers — make X the grammatical subject.

---

**Input**: "We deployed the fix, but it didn't actually solve the issue — turns out Vercel reads commit metadata even on token deploys."

**Output**:
- Verdict: **Band 7+** — natural, idiomatic, varied. No changes.
- Practise: Keep using em-dashes for parenthetical observations; this is a Band 7 cohesion device used well here.

## Reference

When asked to explain a band descriptor in detail, refer the user to the official IELTS Writing Band Descriptors:
- Public version: <https://ielts.org/-/media/pdfs/writing-band-descriptors-task-2.ashx>

(Don't fetch it on every check — just cite when asked.)
