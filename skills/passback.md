---
name: passback
description: Route any document into PassbackAI to collect structured feedback, and pull the answers back. ONE skill, three internal jobs — REVIEW (route existing prose for open-ended feedback), ASK (turn an ambiguous request into a questionnaire — and a prioritize when ordering a shortlist — wrapped in prose context, then route it), and PULL (read back what reviewers answered on a doc you sent). Infer PULL from the request; when the user hasn't said whether they want a clean feedback summary (REVIEW) or that summary with clarification questions on the undecided / missing parts (ASK), ask them to pick between those two before you build anything. Use whenever someone wants feedback on a draft, wants to turn messy input into structured questions, wants to know what came back on a routed doc, or says "passbackai" / "/passback". Triggers include "route this for review", "get feedback on this draft", "extract the open questions", "what's still unclear", "what decisions are missing", "turn this into a questionnaire", "what came back on the doc I sent", "did anyone answer", "תוציא שאלות פתוחות", "מה עדיין לא ברור", "מה צריך להחליט", "שלח לבדיקה", "passbackai".
version: 2.3
created: 05-05-2026
updated: 2026-07-02
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

**This skill has three internal jobs. Detect PULL from the request; between REVIEW and ASK, let the user choose whenever they haven't already said which.**

| Job | The user wants… | You do… |
|---|---|---|
| **REVIEW** | open-ended feedback on a draft that already exists | route the prose as-is for free-form annotation — **no questionnaire** |
| **ASK** | to surface and resolve what is undecided / unclear | author a questionnaire (and a prioritize when ordering is warranted) wrapped in prose context, then route it |
| **PULL** | to know what came back on a doc they already sent | call `list_responses` and synthesize — **never author a new questionnaire** |

## Intent routing — detect PULL, then let the user pick REVIEW vs ASK

Read the request and the document. First separate PULL (reading back existing answers) from the REVIEW/ASK pair (producing a document to route). Then, unless the user has already told you which of REVIEW or ASK they want, **ask them to choose** — don't silently pick for them.

- **PULL** when the user asks about *answers that already exist*: "what came back", "did anyone respond", "what did they say on the doc I sent", "pull the feedback on <doc>". The doc is already routed; the user wants its responses. → **PULL.** Do **not** generate a questionnaire, and do **not** ask the REVIEW-vs-ASK question — PULL is unambiguous.
- **The user already said which one → skip the question and act.** Explicit REVIEW intent ("get feedback on this draft", "route this PRD for review", "send this to Dana for comments" — they want *someone's reaction* to finished prose) → **REVIEW**, and don't invent questions they didn't ask for. Explicit ASK intent ("what's still unclear", "turn this into a questionnaire", "what decisions are missing", "pull out the open questions") → **ASK**.
- **Otherwise — the user hasn't specified which — ASK THEM before building anything**, and ask it the way a picker expects: a **short** question with **two tappable options**, not a long prose sentence. If your harness has a structured question/choice tool (e.g. **AskUserQuestion**), use it — one question, two options with terse labels and a one-line description each:
  - **Feedback summary** — a clean write-up to hand over for open feedback → **REVIEW**
  - **+ clarifying questions** — the same summary with questions built in on the undecided / missing parts → **ASK**

  Keep labels to ~2–4 words and write them in the user's language (Hebrew: **"סיכום לפידבק"** / **"סיכום + שאלות הבהרה"**). No structured tool available? Ask the same two-way choice in one short line. Then wait for the answer and build. Skip the question only when the user's own words already make the choice for you (per the bullet above).

## Delivery — MCP route (primary) or paste (fallback)

All three jobs deliver the **same way**, by whether you are connected to PassbackAI over MCP:

- **Connected over MCP (PREFERRED):** call the **`route_document`** tool with a typed **`blocks[]`** array — prose as `{ "type": "markdown", "text": "…" }` blocks and each interactive component as its own typed block, in reading order. The server **validates each component as you generate it, writes the canonical fences for you, and returns a reviewer link** (`/r/<id>`) the user opens. On this path there is **no smart-quote risk** (you pass typed fields, not text), so the fence/straight-quote hardening below is a **paste-path concern only**. Read answers back with **`list_responses`** (a pull — no copy-paste, no webhook).
- **NOT connected (no MCP), or a route fails:** fall back to **paste** — emit the **entire** Markdown document inside **one single outer code fence** (so the user copies it once and pastes it once — see the Output contract), then give the three-line paste instruction. Fully local; the document never leaves the browser.

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

Read the input. Identify the **open questions** — decisions not made, ambiguities, missing info, placeholders ("TBD", "ask the team"), conflicts between stated preferences, scope boundaries left undefined. Then author a **document with the questions embedded in it** — not a form (see the next rule), and route.

### Author a document, not a form — context as prose BEFORE each question

This is the rule that decides how the output *feels*. The reviewer should read **a short document with questions set into it**, not a stack of form fields with footnotes. Two failures produce the form feel — avoid both:

1. **Don't lump every question into one block.** A single `questionnaire` fence renders its questions back-to-back with nothing between them — that is the form. Instead, **give each question (or a tightly-coupled pair) its own embedded fence, and write a sentence or two of prose BEFORE it** that sets it up. The prose carries the explanation; the fence carries the choice. The reviewer reads context, *then* the question — the way a document reads.
2. **Don't bury the explanation in the per-question `context` field.** That field renders *below* the prompt, so the reviewer hits the question cold and reads the "why" only after. The explanation, the framing, the reassurance — put it in the **prose that comes before the question**. Leave `context` empty, or use it only for a terse one-line caption that genuinely belongs under the prompt (rare).

So the shape of an ASK output is: a short intro paragraph → prose → a question → prose → a question → … — interleaved. Open the document with 1–3 sentences of plain framing (who it's from, what it's for, that none of it is a test), then walk the reviewer question by question, each one introduced by its own lead-in.

**How that maps to delivery:**

- **Connected (MCP `route_document`):** emit an **interleaved `blocks[]` array** — a `{ "type": "markdown", "text": "…" }` lead-in block, then a `questionnaire` block holding just that one question, then the next markdown lead-in, then its question, and so on. **One question per `questionnaire` block**, never all of them in one. A `prioritize` block, when warranted, is just another block in the flow with its own prose lead-in.
- **Not connected (paste):** emit a **Markdown document** with prose paragraphs and **one ` ```questionnaire ` fence per question** set between them — and wrap that whole document in **one single outer code fence** (see the Output contract) so it's one copy button. Each inner fence is a complete JSON object (`version`, `questions` with that single question); prose lives between the inner fences, as ordinary Markdown, all inside the one outer fence.

A tightly-coupled pair of questions may share one fence when they truly have the same setup and no prose belongs between them — but the default is one question per fence with its own lead-in. When in doubt, split.

**Framing lives in prose, not in fields.** When questions are embedded in a document, the renderer shows only the question cards — it does **not** display the `title` or `routing.return_prompt`. So write the title as a Markdown heading, and put the "from <name> / send your answers back to <name>" instruction in the opening and closing prose. The `title`/`routing` fields may still be set (harmless), but never rely on them to be seen in document mode.

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
  "skill_version": "2.3",                  // REQUIRED — must match this skill's frontmatter version (a DIFFERENT field from "version")
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
      "context": "",                       // OPTIONAL — the explanation belongs in the prose BEFORE the question; renders below the prompt, so leave empty unless a terse one-line caption truly belongs there
      "section": "<chapter title>",        // optional — see Sections
      "options": ["a", "b", "c", "d"],     // 3 to 4 short labels — OR [] for a pure free-text question
      "multi": false,                      // optional — true to allow multiple picks; default false
      "recommended": "b",                  // the suggested option LABEL (string; array when multi). Set it whenever you have an honest lean — most questions do. Shows a badge; does NOT pre-select. Must exactly match an option label.
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
| Context | The explanation goes in the **prose before the question**, not here. The `context` field is OPTIONAL and renders *below* the prompt — leave it empty, or use it only for a terse one-line caption that genuinely belongs under the question. |
| Version | Always emit `"skill_version"` matching this file's frontmatter `version`. The app uses it to detect an outdated install and offer an upgrade. |

**Renderer features (use when they earn their weight):**

- **Default Other (prefer it).** By default **omit `open_field`** — that gives the localized "Other:" input *plus* the edit-into-Other gesture (pick a listed option, then tweak it: the desktop hover-pencil / mobile long-press / Shift+letter). That gesture only exists on the default Other, so omitting `open_field` is the richer choice and should be your norm. Reserve a custom `open_field.label` for the narrow case where the escape-hatch genuinely needs a directed phrase — blocked on a person → `"I need to check with:"`; external dependency → `"Depends on:"`; reasoning needed → `"The reason is:"`; constraint → `"The constraint is:"` — since a custom label intentionally hides the edit-into-Other pencil (injecting an option into a directed field reads as nonsense).
- **Free-text questions.** No preset choices? Emit `"options": []` **with** an explicit `open_field` whose `label` names the field ("Deadline:", "Owner:", "Describe it:") and an optional `placeholder`. `open_field` is REQUIRED here. Prefer real options when genuine distinct paths exist.
- **Sections.** Set `section` only when ≥2 chapters exist AND each has 2+ questions. Keep same-section questions consecutive (the renderer groups consecutive items). For <5 questions or no natural break, skip it.
- **Multi-select.** Set `multi: true` ONLY when the question genuinely allows more than one answer (features for v1, integrations, regions, stakeholders). Never for mutually-exclusive picks. Keep options short and parallel.
- **Recommended.** Set `recommended` to a single option label (or an array for `multi`) whenever one option is the safer, more common, or more defensible path — which is **most** option-bearing questions, so reach for it by default, not rarely. The badge labels *which*; the **prose lead-in before the question** explains *why* you lean that way. Skip it only when the options are genuinely equal and you'd have no honest lean. The value must exactly match an option label (graceful-ignored otherwise).
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
1. **The whole Markdown document, inside ONE single outer code fence** — nothing else in that fence, and never split across two. This is the copy-once/paste-once contract: the user reads your reply, clicks the code block's **one** copy button, and pastes the whole thing into PassbackAI in a single gesture. The inner *structure* is unchanged — for an ASK output it is still **prose paragraphs with one ` ```questionnaire ` fence per question (plus any ` ```prioritize ` fence) interleaved between them**, per the "document, not a form" rule — you are only wrapping that entire document in one outer fence.
   - **Use a FOUR-backtick outer fence** (` ```` `) so it can contain the inner three-backtick ` ```questionnaire ` / ` ```prioritize ` fences without the nesting breaking. If the document's own prose somehow contains a four-backtick fence, add more backticks to the outer fence than any fence inside it. The outer fence has no info-string (just the bare ` ```` ` line).
   - The chat surface renders this as a **single code block with one copy button**, and that button copies the *inner* content only (the outer fence markers are stripped) — so what lands on the clipboard is the exact Markdown document, inner ` ```questionnaire ` fences and all, ready to paste.
   - Wrapping everything in the outer fence also means the whole payload copies **verbatim**, which protects the JSON. **Use straight ASCII quotes (`"`) only** — never `“ ” ‚ '` — for every JSON key and string inside the fences; smart quotes break `JSON.parse` and render raw text instead of the form.
2. A short closing message in the user's language (below), **outside** the outer fence (so it isn't copied).

No commentary, no category breakdown, no schema explanation.

**Validate before you output (paste path):**
- It must `JSON.parse` cleanly — balanced `{ } [ ]`, no trailing commas, straight `"` quotes only.
- Exact key names, matched literally: root `version` (`"1"`), `questions` (non-empty); each question `id`, `question` (**never `q`**), `options`. Optional: `title`, `skill_version`, `source_summary`, `routing`, `section`, `multi`, `recommended`, `open_field`.
- `recommended`, when set, EXACTLY matches an option label (array for `multi`) — graceful-ignored otherwise, so a mismatch is a wasted nudge.
- `version` (schema, always `"1"`) vs `skill_version` (this skill's, `"2.3"`) are different fields. If unsure of your own skill version, **omit `skill_version`** rather than guess a wrong low value (which would trigger a false "update your skill" banner).
- Each ` ```questionnaire ` fence is its own complete object — every fence carries `version` (and the same `skill_version`); the `questions` array inside a per-question fence holds that one question. Multiple inner fences in one document is the norm, not an error.
- **Everything lives inside the one outer fence.** The whole document — heading, prose, every inner ` ```questionnaire ` / ` ```prioritize ` fence — is inside a single four-backtick outer fence, so the user copies it all in one click. The closing message stays outside it.

### Closing message (paste path)

After the document, output exactly this (translate to the user's language) — say **document** / **block**, not "JSON", because the output is a document with the questions set into it, and it sits in one code block:

```
X questions.

1. Copy the block above
2. Open https://passbackai.com
3. Click Paste
```

Hebrew variant:

```
X שאלות.

1. העתק את הבלוק למעלה
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

**Output (a document with the questions embedded — prose before each one, the WHOLE thing wrapped in one outer four-backtick fence so it's one copy button. Connected: the same content as an interleaved `blocks[]` array, one markdown block then one questionnaire block, repeated. Paste: the single fenced block below, verbatim):**

> ````
> # Guest check-in — a few open decisions
>
> Hi Elad — these are the three things we didn't lock on the mobile check-in brief, plus the launch order. None of this is a test; pick what fits or leave a note, and send your answers back to me when you're done.
>
> First, the front door. We never settled how a guest proves who they are at check-in — room number alone is the lightest, but it's also the weakest. I'd lean to room number + PIN: one extra field, and it closes the "anyone who sees a door number is in" hole. Where do you want to land?
>
> ```questionnaire
> {"version":"1","skill_version":"2.3","questions":[{"id":"q1","question":"What authentication method should guests use at check-in?","options":["Room number only","Room number + PIN","Last name + booking ref","Magic link"],"recommended":"Room number + PIN"}]}
> ```
>
> Next, biometrics. The team split on whether face/fingerprint ships in v1 or waits. Deferring keeps v1 tight and keeps a privacy review off the critical path — which is why I'd lean to deferring, but it's your call.
>
> ```questionnaire
> {"version":"1","skill_version":"2.3","questions":[{"id":"q2","question":"Should we offer biometric authentication in v1?","options":["No, defer to v2","Optional opt-in","Required"],"recommended":"No, defer to v2"}]}
> ```
>
> Last on the spec: retention. The brief flags legal sign-off as pending, so if this needs to go through someone, tell me who.
>
> ```questionnaire
> {"version":"1","skill_version":"2.3","questions":[{"id":"q3","question":"What is the data retention policy?","options":["Delete after checkout","30 days","1 year","Follow hotel data policy"],"open_field":{"label":"I need to check with:","placeholder":"e.g. legal team, DPO"}}]}
> ```
>
> Separately — we have four launch markets queued and no order. Drag them into the sequence you'd ship them in:
>
> ```prioritize
> {"version":"1","items":[{"id":"us","label":"United States"},{"id":"uk","label":"United Kingdom"},{"id":"de","label":"Germany"},{"id":"jp","label":"Japan"}]}
> ```
> ````

```
3 questions, plus a launch-order ranking.

1. Copy the block above
2. Open https://passbackai.com
3. Click Paste
```

Note the shape: **one outer four-backtick fence wraps the entire document** — heading, prose, and every inner three-backtick component fence — so the user copies it in a single click and pastes once. Inside it: a heading, then **prose that sets up each question before the question appears**, each question in its **own inner fence**, the four launch markets in a `prioritize` (the decision is **order**, ≥3 concrete mutually-rankable peers) — not crammed into one form block, and no per-question `context` field (the setup is the prose above it). Note also: q1 and q2 carry a `recommended` because there's an honest lean, and the *why* sits in the prose lead-in, not in `context`; q1 and q2 omit `open_field`, so they keep the **default Other** with its edit-into-Other pencil, while q3 opts into a directed `open_field.label` ("I need to check with:") and deliberately gives up that pencil. The "send it back to me" instruction is in the opening prose because the embedded renderer won't show `routing`. *(That leading/trailing `` ```` `` is the real four-backtick outer fence — it renders as one code block with one copy button, and the copy strips it, leaving the inner three-backtick fences intact on the clipboard. The blockquote `>` marks are only this doc's way of showing the block; your real output has no `>`.)*

## Changelog

### v2.3 (2026-07-02)
- **Copy once, paste once.** The paste output is now the whole document inside **one outer four-backtick fence** — a single code block with one copy button — instead of prose interleaved with separate fences the user had to select by hand. The inner "document, not a form" structure is unchanged; only the outer wrapper is new.
- **Ask REVIEW vs ASK when it isn't specified.** When the user hasn't said whether they want a clean feedback summary (REVIEW) or that summary with clarification questions on the undecided/missing parts (ASK), the skill now asks them to pick before building, instead of silently inferring. PULL is still detected without a question.

### v2.2 (2026-06-30)
- **Lean in with `recommended`.** Reach for `recommended` by default on option questions where there's an honest lean (most of them), not rarely — the *why* lives in the prose lead-in. Pairs with the renderer making the recommended star clearer at rest.
- **Prefer the default Other.** Omitting `open_field` is now the explicit norm, because only the default Other carries the edit-into-Other gesture (pick an option, then tweak it). A custom `open_field.label` is reserved for the narrow directed-escape-hatch case, which intentionally gives up that pencil.

### v2.1 (2026-06-30)
- **Document, not a form.** ASK now authors a document with the questions embedded in it: prose sets up each question *before* it, each question gets its own ` ```questionnaire ` fence (interleaved markdown + questionnaire `blocks[]` on the MCP path), and the explanation lives in that lead-in prose instead of the per-question `context` field (which renders below the prompt and made the output read like a form). One question per fence by default; framing (title, "send it back to <name>") goes in prose, since the embedded renderer shows only the question cards.

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
