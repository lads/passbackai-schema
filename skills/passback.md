---
name: passback
description: Route any document into PassbackAI to collect structured feedback, and pull the answers back. ONE skill, three internal jobs — REVIEW (route existing prose for open-ended feedback), ASK (turn an ambiguous request into a questionnaire — and a prioritize when ordering a shortlist — wrapped in prose context, then route it), and PULL (read back what reviewers answered on a doc you sent). Infer which job from the request; ask the user only when it is genuinely unclear. Use whenever someone wants feedback on a draft, wants to turn messy input into structured questions, wants to know what came back on a routed doc, or says "passbackai" / "/passback". Triggers include "route this for review", "get feedback on this draft", "extract the open questions", "what's still unclear", "what decisions are missing", "turn this into a questionnaire", "what came back on the doc I sent", "did anyone answer", "תוציא שאלות פתוחות", "מה עדיין לא ברור", "מה צריך להחליט", "שלח לבדיקה", "passbackai".
version: 2.0
created: 05-05-2026
updated: 2026-06-27
created_by: adam-mason
owner: Adam Mason
triggers:
  - route this for review
  - get feedback on this draft
  - extract open questions
  - what's still unclear
  - turn this into a questionnaire
  - what came back on the doc I sent
  - did anyone answer
  - passbackai
  - תוציא שאלות
  - מה לא ברור
  - מה צריך להחליט
  - שלח לבדיקה
---

# /passback — route documents for feedback, pull the answers back

**What PassbackAI is.** One tool that closes a feedback loop: you **route** a document (prose, or prose with embedded decision widgets) to a reviewer, the reviewer answers in a beautiful interactive surface, and you **pull** their answers back as structured data. The reviewer is often the user themselves; sometimes it is someone they hand the link to.

**This skill has three internal jobs. Pick one by inferring intent — ask only when genuinely unclear.**

| Job | The user wants… | You do… |
|---|---|---|
| **REVIEW** | open-ended feedback on a draft that already exists | route the prose as-is for free-form annotation — **no questionnaire** |
| **ASK** | to surface and resolve what is undecided / unclear | author a questionnaire (and a prioritize when ordering is warranted) wrapped in prose context, then route it |
| **PULL** | to know what came back on a doc they already sent | call `list_responses` and synthesize — **never author a new questionnaire** |

## Intent routing — decide first, ask rarely

Read the request and the document. Route to exactly one job:

- **PULL** when the user asks about *answers that already exist*: "what came back", "did anyone respond", "what did they say on the doc I sent", "pull the feedback on <doc>". The doc is already routed; the user wants its responses. → **PULL.** Do **not** generate a questionnaire.
- **REVIEW** when there is an existing piece of prose and the user wants *someone's reaction* to it: "get feedback on this draft", "route this PRD for review", "send this to Dana for comments". The content is written; you are collecting open-ended notes, not forcing choices. → **REVIEW.** Do **not** invent questions the author did not ask for.
- **ASK** when the input is messy or under-decided and the user wants *the open questions surfaced*: "what's still unclear", "turn this into a questionnaire", "what decisions are missing". → **ASK.**
- **Ask the user ONE clarifying question only when the request is truly ambiguous** — e.g. a bare "help me with this doc" where you cannot tell whether they want open feedback (REVIEW) or the decisions extracted (ASK). Then:

  > "Do you want open-ended feedback on this as-is, or should I pull out the open questions for someone to decide?"

  If intent is clear from context, **skip the question** and act.

## Delivery — MCP route (primary) or paste (fallback)

All three jobs deliver the **same way**, by whether you are connected to PassbackAI over MCP:

- **Connected over MCP (PREFERRED):** call the **`route_document`** tool with a typed **`blocks[]`** array — prose as `{ "type": "markdown", "text": "…" }` blocks and each interactive component as its own typed block, in reading order. The server **validates each component as you generate it, writes the canonical fences for you, and returns a reviewer link** (`/r/<id>`) the user opens. On this path there is **no smart-quote risk** (you pass typed fields, not text), so the fence/straight-quote hardening below is a **paste-path concern only**. Read answers back with **`list_responses`** (a pull — no copy-paste, no webhook).
- **NOT connected (no MCP), or a route fails:** fall back to **paste** — emit the Markdown document (with any component as a fenced JSON block) and give the three-line paste instruction. Fully local; the document never leaves the browser.

> **The component fields are identical on both paths** — only delivery differs. This skill keeps the authoring rules lean; the **full, code-verified component schema (every field, both components) lives at <https://passbackai.com/ask>** (raw: `/ask.md`, JSON Schema: `/schema.json`). Read it there rather than expecting this file to restate it.

> **Privacy — keep it accurate.** The local **paste** loop never transmits the document. **Routing over MCP is an opt-in, server-side egress** — `route_document` stores the routed doc on the PassbackAI backend (server-managed keys, **not** zero-knowledge). Never describe a routed document as end-to-end-encrypted.

---

## JOB: REVIEW — route existing prose for open feedback

The author already wrote the document; they want a reviewer's reaction. Route it **as prose**, unchanged in meaning.

- **Connected:** one or more `{ "type": "markdown", "text": "…" }` blocks carrying the document, in order. No questionnaire, no prioritize unless the author's own content contains a genuine open decision you'd surface (see ASK).
- **Not connected:** emit the Markdown as-is and give the paste instruction.

Do **not** rewrite the author's prose into questions. REVIEW collects free-form annotations on what they wrote; the structured-question path is ASK.

---

## JOB: ASK — turn ambiguity into a questionnaire (+ prioritize when warranted), then route

Read the input. Identify the **open questions** — decisions not made, ambiguities, missing info, placeholders ("TBD", "ask the team"), conflicts between stated preferences, scope boundaries left undefined. Wrap them in just enough prose context, then route.

### Choose the right component — the sharpened gate

PassbackAI renders a **growing family** of interactive components. Two are shipped today:

| Component | Verb | Use it to… |
|---|---|---|
| **questionnaire** | **choose** | ask the user to pick from options (single/multi) or type a free-text answer |
| **prioritize** | **order** | ask the user to rank/reorder a short list by priority |

**Default to a questionnaire.** Reach for **prioritize ONLY when ALL THREE hold:**
1. the decision is **order / rank**, not **choose** one;
2. there are **≥3 concrete, comparable alternatives that already exist** (not hypothetical, not yet-to-be-invented);
3. they are **mutually-rankable peers** — the same kind of thing, sortable against each other.

> **Heuristic: Picking → questionnaire. Ordering a shortlist of ≥3 peers → prioritize.**

A two-way choice is a **questionnaire**, never a prioritize ("ranking" two items is just picking one to go first). "Which database?" is a questionnaire. "What order should we ship these four features?" is a prioritize. **A questionnaire and a prioritize can co-exist in one routed document** — extract the decisions, give each the component its verb demands.

### Questionnaire authoring — the rules

```jsonc
{
  "version": "1",                          // required, always "1" — the SCHEMA version, not the skill's
  "skill_version": "2.0",                  // REQUIRED — must match this skill's frontmatter version (a DIFFERENT field from "version")
  "title": "<short title>",                // recommended
  "source_summary": "<one sentence>",      // optional
  "routing": {                             // omit when self-answering
    "from": "<sender name>",
    "return_prompt": "When done, copy your answers and send them back to <sender name>."
  },
  "questions": [
    {
      "id": "q1",                          // required, unique
      "question": "<the open question>",   // required — NEVER the key `q`
      "context": "<why this is open>",     // required, one sentence, reference the source
      "section": "<chapter title>",        // optional — see Sections
      "options": ["a", "b", "c", "d"],     // 3 to 4 short labels — OR [] for a pure free-text question
      "multi": false,                      // optional — true to allow multiple picks; default false
      "recommended": "b",                  // optional — the suggested option LABEL (string; array of labels when multi). Shows a badge; does NOT pre-select. Must exactly match an option label.
      "open_field": {                      // OMIT for the default "Other"; REQUIRED when options is []
        "label": "I need to check with:",
        "placeholder": "e.g. legal team"
      }
    }
  ]
}
```

| Rule | Detail |
|---|---|
| Cap | **Maximum 12 questions.** If more found, pick the most blocking. |
| Options | 3 to 4 per question, short labels (under 8 words), genuinely distinct paths — not synonyms. **Exception:** no genuine preset choices (a date, a name, a free-form description) → emit `"options": []` with an explicit `open_field` (see Free-text). Don't manufacture filler options. |
| Never | Don't put "Other" inside `options` — the renderer adds it. |
| Skip | If the answer is already in the text, don't ask. |
| One per | One decision per question. Don't bundle. |
| Context | Required on every question. One sentence. Reference the source. |
| Version | Always emit `"skill_version"` matching this file's frontmatter `version`. The app uses it to detect an outdated install and offer an upgrade. |

**Renderer features (use when they earn their weight):**

- **Default Other.** Omit `open_field` for the localized "Other:" input plus the edit-into-Other gesture (hover-pencil desktop, long-press mobile). Set a custom `open_field.label` only when the escape-hatch needs a directed phrase: blocked on a person → `"I need to check with:"`; external dependency → `"Depends on:"`; reasoning needed → `"The reason is:"`; constraint → `"The constraint is:"`.
- **Free-text questions.** No preset choices? Emit `"options": []` **with** an explicit `open_field` whose `label` names the field ("Deadline:", "Owner:", "Describe it:") and an optional `placeholder`. `open_field` is REQUIRED here. Prefer real options when genuine distinct paths exist.
- **Sections.** Set `section` only when ≥2 chapters exist AND each has 2+ questions. Keep same-section questions consecutive (the renderer groups consecutive items). For <5 questions or no natural break, skip it.
- **Multi-select.** Set `multi: true` ONLY when the question genuinely allows more than one answer (features for v1, integrations, regions, stakeholders). Never for mutually-exclusive picks. Keep options short and parallel.
- **Recommended.** Optional. Set `recommended` to a single option label (or an array for `multi`) to nudge toward the suggested pick — the badge labels *which*, the `context` must explain *why*. Use sparingly; the value must exactly match an option label.
- **Routing.** When set, `routing.return_prompt` MUST contain the literal value of `routing.from` so the renderer can auto-bold the name.
- **RTL.** Auto-detected from content. Don't set a language field. Hebrew/Arabic input → Hebrew/Arabic chrome.
- **Notes.** Every option-bearing question has a `+ add a note` toggle; a note alone is a valid answer. Don't engineer "TBD" options.

### Prioritize authoring — the shape

When the gate above is satisfied, add a prioritize block. The full field list is at <https://passbackai.com/ask>; the minimal shape is:

```jsonc
{
  "version": "1",
  "items": [
    { "id": "a", "label": "<peer 1>" },
    { "id": "b", "label": "<peer 2>" },
    { "id": "c", "label": "<peer 3>" }
  ]
}
```

The reviewer drags the items into priority order; you read the order back via `list_responses` (or the pasted export).

### Self-answer vs routing to someone else

If the conversation does NOT make clear whether the user answers themselves or sends to someone else, ask one question first:

> "Are you answering these yourself, or sending them to someone else?"

If clear from context ("for me", "for David"), skip it.

- **Self-answer** → omit `routing`.
- **Sending to someone else** → include `routing.from` (sender's name) and `routing.return_prompt`.

---

## JOB: PULL — read back the answers, then synthesize

The user asks what came back on a doc they already routed. **Do not author a new questionnaire.**

- **Connected:** call **`list_responses`** with the document id. It returns each reviewer's `verdict`, leaf-anchored `annotations[]`, and leaf-free `componentInputs[]` (their questionnaire/prioritize answers) — a pull, no webhook. Then **synthesize for the user**: summarize the verdict, the structured answers, and the free-form notes in their own language; surface conflicts and anything still unanswered.
- **Not connected:** the answers come back as the reviewer's **pasted export bundle**. Read it and synthesize the same way.

If you don't know the document id, ask the user which routed document they mean (or list recent ones if the tool surface allows).

---

## Output contract & validation — PASTE path only

When you are **NOT** routing over MCP, deliver as paste. On the `route_document` path the platform validates the typed `blocks[]` for you, so the fence/quote hardening below does **not** apply — but the field shapes (key names, the `version` vs `skill_version` distinction) are still the correct shapes you pass as typed arguments.

**Output two things, in this exact order, nothing else:**
1. The Markdown document **inside a fenced code block** when it's a sole component (so it copies verbatim), or as plain Markdown when it's prose with embedded ` ```questionnaire ` / ` ```prioritize ` fences. **Use straight ASCII quotes (`"`) only** — never `“ ” ‚ '` — for every JSON key and string. A code fence is copied verbatim; unfenced JSON gets its quotes "smart-quoted" by many chat surfaces, which breaks `JSON.parse` and renders raw text instead of the form.
2. A short closing message in the user's language (below).

No commentary, no category breakdown, no schema explanation.

**Validate before you output (paste path):**
- It must `JSON.parse` cleanly — balanced `{ } [ ]`, no trailing commas, straight `"` quotes only.
- Exact key names, matched literally: root `version` (`"1"`), `questions` (non-empty); each question `id`, `question` (**never `q`**), `options`. Optional: `title`, `skill_version`, `source_summary`, `routing`, `section`, `multi`, `recommended`, `open_field`.
- `recommended`, when set, EXACTLY matches an option label (array for `multi`) — graceful-ignored otherwise, so a mismatch is a wasted nudge.
- `version` (schema, always `"1"`) vs `skill_version` (this skill's, `"2.0"`) are different fields. If unsure of your own skill version, **omit `skill_version`** rather than guess a wrong low value (which would trigger a false "update your skill" banner).

### Closing message (paste path)

For a questionnaire, after the block output exactly this (translate to the user's language):

```
X questions in Y categories.

1. Copy the JSON above
2. Open https://passbackai.com
3. Click Paste
```

Drop the categories phrase (`X questions.`) when there are no sections. Hebrew variant:

```
X שאלות ב-Y קטגוריות.

1. העתק את ה-JSON למעלה
2. פתח את https://passbackai.com
3. לחץ Paste
```

That's the entire post-output text.

---

## What counts as an open question (ASK)

Explicit "we haven't decided X" / placeholder ("TBD", "ask the team") / conflict between stated preferences / missing info the rest assumes / implicit decision made without flagging / scope boundary not defined.

**Skip:** style preferences with no consequence, questions answered later in the same doc, pure implementation details (unless the input IS a tech spec).

## Worked example (ASK — a questionnaire + a co-existing prioritize)

**Input:** "Mobile guest check-in feature — sending the open decisions to Elad. Haven't decided PIN vs room number; biometric debate (v1 or defer); legal must confirm retention. Also: we have four launch markets queued — US, UK, Germany, Japan — and need Elad to set the rollout order."

**Output (connected → typed `blocks[]`; shown here as the equivalent document):**

```json
{
  "version": "1",
  "skill_version": "2.0",
  "title": "Guest Check-In, Open Decisions",
  "routing": {
    "from": "Elad",
    "return_prompt": "When done, copy your answers and send them back to Elad."
  },
  "questions": [
    {
      "id": "q1",
      "question": "What authentication method should guests use at check-in?",
      "context": "Brief: PIN vs room number not yet decided.",
      "options": ["Room number only", "Room number + PIN", "Last name + booking ref", "Magic link"]
    },
    {
      "id": "q2",
      "question": "Should we offer biometric authentication in v1?",
      "context": "Team split between v1 and defer; deferring keeps v1 scope tight and avoids a privacy review on the critical path.",
      "options": ["No, defer to v2", "Optional opt-in", "Required"],
      "recommended": "No, defer to v2"
    },
    {
      "id": "q3",
      "question": "What is the data retention policy?",
      "context": "Brief notes legal sign-off pending.",
      "options": ["Delete after checkout", "30 days", "1 year", "Follow hotel data policy"],
      "open_field": {"label": "I need to check with:", "placeholder": "e.g. legal team, DPO"}
    }
  ]
}
```

```prioritize
{"version":"1","items":[{"id":"us","label":"United States"},{"id":"uk","label":"United Kingdom"},{"id":"de","label":"Germany"},{"id":"jp","label":"Japan"}]}
```

```
3 questions, plus a launch-order ranking.

1. Copy the document above
2. Open https://passbackai.com
3. Click Paste
```

The four launch markets are a clean prioritize: the decision is **order**, there are **≥3 concrete peers** that already exist, and they're **mutually rankable**. The three open decisions stay a questionnaire — they're **picks**, not an ordering.

## Changelog

### v2.0 (2026-06-27)
- **One skill, three jobs.** `/passback` consolidates the loop into REVIEW (route existing prose for open feedback), ASK (turn ambiguity into a questionnaire — plus a prioritize when ordering a ≥3-peer shortlist — wrapped in prose context, then route), and PULL (`list_responses` + synthesize). Intent is inferred; the model asks only when genuinely unclear. Supersedes and renames the old `question-extractor` skill.
- **Sharpened prioritize gate.** Default to a questionnaire; use prioritize ONLY when the decision is order-not-choose, ≥3 concrete comparable alternatives already exist, and they're mutually-rankable peers. A questionnaire and a prioritize can co-exist in one routed document.
- Absorbs question-extractor's questionnaire authoring rules, hard rules, paste fallback, and validation checklist unchanged; the full component schema now lives at <https://passbackai.com/ask> rather than restated here.

### v1.6 (2026-06-26)
- MCP delivery (`route_document`): route the questionnaire as typed `blocks[]` (server-validated, returns a reviewer link) instead of pasting JSON; paste is the fallback.

### v1.5 (2026-06-21)
- Recommended option (`recommended`): flag the suggested pick with a badge that doesn't pre-select.

### v1.4 (2026-06-18)
- Output hardening: fenced block, straight quotes, pre-send validation of exact key names.

### v1.3 (2026-06-17)
- Free-text questions (`open_field` + empty `options`).

### v1.2 (2026-05-06)
- Multi-select questions (`multi: true`).

### v1.1 (2026-05-05)
- Sections, routing, default Other field with edit-into-Other gesture.

### v1.0 (2026-05-05)
- Initial release (as `question-extractor`).
