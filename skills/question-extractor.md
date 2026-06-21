---
name: question-extractor
description: Extract open questions from any text (conversation, brief, PRD, email, spec) and output a JSON questionnaire that pastes directly into https://passbackai.com to render an interactive form. Use this whenever someone wants to identify unresolved decisions, find what's still unclear, or turn messy input into structured questions with answer options. Triggers include "extract the open questions", "what's still unclear", "what decisions are missing", "turn this into a questionnaire", "find the unknowns", "generate questions from this", "תוציא שאלות פתוחות", "מה עדיין לא ברור", "מה צריך להחליט", "passbackai".
version: 1.5
created: 05-05-2026
updated: 2026-06-21
created_by: adam-mason
owner: Adam Mason
triggers:
  - extract open questions
  - what's still unclear
  - turn this into a questionnaire
  - find unknowns
  - passbackai
  - תוציא שאלות
  - מה לא ברור
  - מה צריך להחליט
  - תהפוך לשאלון פתוח
---

# Question Extractor

**Job.** Read any input. Identify the open questions (decisions not made, ambiguities, missing info). Output a JSON questionnaire that pastes into https://passbackai.com.

> **Scope.** A questionnaire is **one** of passbackai's interactive components — the *choose* verb (pick from options or free-text). This skill authors that one. The site renders a growing family of them (e.g. a **prioritize** block — the *order* verb, ranking a shortlist), each embedded as its own fenced JSON block in a document. This skill stays focused on extracting open **questions** → a questionnaire; the full questionnaire schema (every field, including `recommended`) is the public reference at <https://passbackai.com/ask>.

**Output contract.** Two things, in this exact order, nothing else:
1. A single JSON object **inside a ```json fenced code block** — never as raw, unfenced text. A code fence is copied verbatim; unfenced JSON gets its straight quotes "smart-quoted" (`"` → `“ ”`) by many chat surfaces, which breaks `JSON.parse` and makes the app render raw text instead of the form. Use straight ASCII quotes (`"`) only — never `“` `”` `‚` `'` — for every key and string.
2. A short closing message in the user's language (see Closing Message Format below).

No commentary, no category breakdown, no schema explanation, no "here's what I built." The closing message is the ONLY chat text after the JSON.

## Closing Message Format (MANDATORY)

After the JSON block, output exactly this structure (translate to user's language):

```
X questions in Y categories.

1. Copy the JSON above
2. Open https://passbackai.com
3. Click Paste
```

If there are no sections, say `X questions.` (drop the categories phrase). That's the entire post-JSON output. No additional sentences. No listing the categories. No explaining the JSON. Done.

Hebrew variant:
```
X שאלות ב-Y קטגוריות.

1. העתק את ה-JSON למעלה
2. פתח את https://passbackai.com
3. לחץ Paste
```

## Optional pre-step (only when ambiguous)

If the conversation does NOT make it clear whether the user is answering themselves or sending to someone else, ask one question before generating:

> "Are you answering these yourself, or sending them to someone else?"

If clear from context (user said "for me", "for David", etc.), skip the question. Just produce the output.

- **Self-answer** → omit `routing` from the JSON.
- **Sending to someone else** → include `routing.from` (sender's name from context) and `routing.return_prompt` ("When done, copy your answers and send them back to <name>.").

## JSON schema

```jsonc
{
  "version": "1",                          // required, always "1" — the SCHEMA version, not the skill's
  "skill_version": "1.5",                  // REQUIRED — must match this skill's frontmatter version (a DIFFERENT field from "version")
  "title": "<short title>",                // recommended
  "source_summary": "<one sentence>",      // optional
  "routing": {                             // omit when self-answering
    "from": "<sender name>",
    "return_prompt": "When done, copy your answers and send them back to <sender name>."
  },
  "questions": [
    {
      "id": "q1",                          // required, unique
      "question": "<the open question>",   // required
      "context": "<why this is open>",     // required, one sentence, reference source
      "section": "<chapter title>",        // optional, see Sections below
      "options": ["a", "b", "c", "d"],     // 3 to 4 short labels — OR [] for a pure free-text question
      "multi": false,                      // OPTIONAL — true to allow multiple picks; default false
      "recommended": "b",                  // OPTIONAL — the suggested option LABEL (a string; an ARRAY of labels when multi). Shows a "Recommended" badge; does NOT pre-select. Must exactly match an option label.
      "open_field": {                      // OMIT to use default Other; REQUIRED when options is []
        "label": "I need to check with:",
        "placeholder": "e.g. legal team"
      }
    }
  ]
}
```

## Hard rules

| Rule | Detail |
|---|---|
| Cap | **Maximum 12 questions.** If more found, pick the most important / most blocking. |
| Options | 3 to 4 per question. Short labels (under 8 words). Genuinely distinct paths, not synonyms. **Exception:** when there are no genuine preset choices (a date, a name, a free-form description), emit `"options": []` and an explicit `open_field` — see Free-text questions below. Don't manufacture filler options just to fill the array. |
| Never | Don't include "Other" inside the `options` array. The renderer adds it automatically. |
| Skip | If the answer is already in the text, don't ask the question. |
| One per | One decision per question. Don't bundle. |
| Context | Required on every question. One sentence. Reference the source. |
| Version | Always emit `"skill_version": "<this skill's frontmatter version>"` at the JSON root. The app uses it to detect when the user's installed skill is older than the deployed version and offer an in-app upgrade prompt. The string MUST match this file's `version:` frontmatter. |

## Validate before sending (MANDATORY)

Before you output the JSON, verify every item below. A single miss makes the app render raw text instead of the interactive form:

- **It must `JSON.parse` cleanly.** Balanced `{ }` and `[ ]`, no trailing comma before a `}` or `]`, every key and string wrapped in straight `"` quotes (no `“ ”`). If mentally running the block through `JSON.parse` would throw, fix it first.
- **Exact key names — the renderer matches them literally, no aliases:**
  - root: `version` (always `"1"`), `questions` (non-empty array). Optional: `title`, `skill_version`, `source_summary`, `routing`.
  - each question: `id`, `question` (the question text — **never `q`**), `options` (array of strings). Optional: `context`, `section`, `multi`, `recommended`, `open_field`.
- **`recommended`, when set, must EXACTLY match an option label** (an array of labels for `multi`). The app graceful-ignores any value that doesn't match — the badge just won't show — but a non-matching value is a wasted nudge, so copy the option label verbatim.
- **`version` vs `skill_version` are different fields.** `version` is the schema version, always `"1"`. `skill_version` is THIS skill's version (`"1.5"`). If you are not certain of your own skill version, **omit `skill_version`** rather than guessing — a missing value is treated as "unknown" (no nag), but a wrong low value triggers a false "update your skill" banner.

## Renderer features (use when they earn their weight)

**Default Other.** Omit `open_field` for the default. The renderer renders a localized "Other:" input AND unlocks the edit-into-Other gesture (hover-pencil desktop, long-press mobile). Setting a custom `open_field.label` disables that gesture, so only set it when the escape-hatch needs a directed phrase:

| Situation | Label |
|---|---|
| Default | omit `open_field` |
| Blocked on a person/team | `"I need to check with:"` |
| Depends on external condition | `"Depends on:"` |
| Reasoning needed | `"The reason is:"` |
| Constraint/deadline | `"The constraint is:"` |
| Tool/resource not listed | `"The tool/resource is:"` |

**Free-text questions.** Some decisions have no preset choices — a launch date, a person's name, a free-form description ("Describe the target persona"). For those, emit `"options": []` **with** an explicit `open_field` whose `label` names the field ("Deadline:", "Owner:", "Describe it:") and an optional `placeholder`. The renderer shows that field as the question's sole input (no radios, no "Other:"), and the export emits `Answer: <label> <text>`. `open_field` is REQUIRED here — empty options with no `open_field` is rejected, because the only row would be a meaningless default "Other:". Prefer real options when genuine distinct paths exist; reach for free-text only when forcing 3–4 choices would invent fake ones.

**Sections.** Set `section` only when ≥2 chapters exist AND each has 2+ questions. Use short titles ("Authentication", "Compliance"). Keep same-section questions consecutive in the array (the renderer groups CONSECUTIVE items). For <5 questions or no natural break, skip `section` entirely.

**Multi-select.** Default is single-select (radio). Set `multi: true` ONLY when the question genuinely allows more than one answer — features to include in v1, integrations to support, channels to launch on, regions to roll out to, stakeholders to consult. Do NOT set `multi` for questions where the answers are mutually exclusive ("which auth method", "which deadline"). When `multi` is true, the renderer shows a "Select all that apply" badge, switches to checkboxes, and the export emits an `Answers:` bullet list instead of a single `Answer:` line. Keep options short and parallel — they read as a set, not a hierarchy.

**Recommended option.** OPTIONAL. Set `recommended` to nudge the reader toward the option you'd suggest: a single option **label** (a string) for a single-select question, or an **array of labels** for a `multi` question. The renderer shows a small "Recommended" badge on the matching option(s) — it labels which, it does NOT pre-select it, and the reader still actively picks. The value must **exactly match** an option label (graceful-ignored otherwise, so the badge silently won't show). When you set `recommended`, make sure that question's `context` explains WHY it's the recommendation — the badge shows *which*, the context shows *why*. Use it **sparingly** — only when you have a defensible recommendation, not on every question. Never put rationale text in the option label; keep options short and let `context` carry the reasoning. (The recommendation is NOT echoed into the copied-back answers — it's an in-app answering aid, and the reader's actual pick is the signal you get back.)

**Routing.** When set, `routing.return_prompt` MUST contain the literal value of `routing.from` so the renderer can auto-bold the name.

**RTL.** Auto-detected from content. Don't set a language field. Hebrew/Arabic input → Hebrew/Arabic chrome strings.

**Notes.** Every option-bearing question has a `+ add a note` toggle (a pure free-text question doesn't — there the field itself IS the answer, so a separate note would be redundant). A note alone is a valid answer. Don't engineer "TBD" options.

## What counts as an open question

Explicit "we haven't decided X" / placeholder ("TBD", "ask the team") / conflict between stated preferences / missing info the rest assumes / implicit decision the author made without flagging / scope boundary not defined.

**Skip:** style preferences with no consequence, questions answered later in the same doc, pure implementation details (unless the input IS a tech spec).

## Worked example

**Input:** "Mobile guest check-in feature — I'm sending the open decisions to Elad to call. Haven't decided PIN vs room number, biometric debate (v1 or defer), legal needs to confirm retention, storage location TBD: region-local or centralized."

**Output:**

```json
{
  "version": "1",
  "skill_version": "1.5",
  "title": "Guest Check-In, Open Decisions",
  "routing": {
    "from": "Elad",
    "return_prompt": "When done, copy your answers and send them back to Elad."
  },
  "questions": [
    {
      "id": "q1",
      "section": "Authentication",
      "question": "What authentication method should guests use at check-in?",
      "context": "Brief: PIN vs room number not yet decided.",
      "options": ["Room number only", "Room number + PIN", "Last name + booking ref", "Magic link"]
    },
    {
      "id": "q2",
      "section": "Authentication",
      "question": "Should we offer biometric authentication in v1?",
      "context": "Team split between v1 and defer-to-v2; deferring keeps v1 scope tight and avoids a privacy review on the critical path.",
      "options": ["No, defer to v2", "Optional opt-in", "Required"],
      "recommended": "No, defer to v2"
    },
    {
      "id": "q3",
      "section": "Compliance",
      "question": "What is the data retention policy?",
      "context": "Brief notes legal sign-off pending.",
      "options": ["Delete after checkout", "30 days", "1 year", "Follow hotel data policy"],
      "open_field": {"label": "I need to check with:", "placeholder": "e.g. legal team, DPO"}
    },
    {
      "id": "q4",
      "section": "Compliance",
      "question": "Where is guest check-in data stored?",
      "context": "Brief lists region-local vs centralized as TBD.",
      "options": ["Region-local", "Centralized", "Hybrid (PII local, telemetry central)"]
    },
    {
      "id": "q5",
      "section": "Compliance",
      "question": "Which guest data fields do we collect at check-in?",
      "context": "Brief is silent on the field list — depends on policy below.",
      "multi": true,
      "options": ["Full name", "ID/passport number", "Phone", "Email", "License plate", "Estimated arrival time"]
    },
    {
      "id": "q6",
      "section": "Compliance",
      "question": "What is the target launch date?",
      "context": "Brief never states a date — a free-form answer, not a choice.",
      "options": [],
      "open_field": {"label": "Launch date:", "placeholder": "e.g. Q3, or a specific date"}
    }
  ]
}
```

```
6 questions in 2 categories.

1. Copy the JSON above
2. Open https://passbackai.com
3. Click Paste
```

## Changelog

### v1.5 (2026-06-21)
- Recommended option: set `recommended` (an option label, or an array of labels for `multi`) to nudge the reader toward the suggested pick. The renderer shows a "Recommended" badge on the matching option without pre-selecting it; pair it with a `context` that explains why, and use it sparingly.

### v1.4 (2026-06-18)
- Output hardening: the JSON MUST ship inside a ```json fenced block with straight ASCII quotes (prevents smart-quote corruption), plus a mandatory pre-send validation checklist that pins the exact key names (`question`, never `q`) and the `version` / `skill_version` distinction.

### v1.3 (2026-06-17)
- Free-text questions: emit `"options": []` with an explicit `open_field` to ask an open question with no preset choices.

### v1.2 (2026-05-06)
- Multi-select questions (set `multi: true`), checkboxes + bullet-list export.

### v1.1 (2026-05-05)
- Sections, routing, default Other field with edit-into-Other gesture.

### v1.0 (2026-05-05)
- Initial release.
