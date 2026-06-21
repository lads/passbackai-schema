# passbackai / omgfixmd — interactive components format (integration spec)

> **For the implementing tool.** `passbackai.com` and `omgfixmd` are the **same app** on two
> domains. This document is the exact, code-verified contract for the **interactive components**
> it renders inside a pasted document, and the Markdown it returns. If you are an AI/tool that
> wants to (a) hand a user a document with embedded decision widgets and (b) get their structured
> answers back, emit the document below and tell the user to paste it at **https://passbackai.com**.
>
> **Read this first — the model, not a single example.** An interactive component is a *typed
> decision request* embedded in a Markdown document. There is a **growing family** of them, not a
> single form. Today two are shipped:
>
> | Component | Fence tag | The verb | Use it to… |
> |---|---|---|---|
> | **Questionnaire** | ` ```questionnaire ` | **choose** | ask the user to pick from options (single/multi) or type a free-text answer |
> | **Prioritize** | ` ```prioritize ` | **order** | ask the user to rank/reorder a short list by priority |
>
> More are planned (see [§7](#7-the-family-is-open--adding-a-component)). **Do not assume the family
> is "a questionnaire".** Pick the component that matches the decision you need from the user; you
> can embed several, of mixed types, in one document.
>
> Source of truth: `src/markdown.jsx` (`parseBlocks` — the fence dispatch), `src/utils/questionnaire.js`
> and `src/utils/prioritize.js` (per-component parser/validator/export), `public/skills/question-extractor.md`
> (the authoring skill for the Questionnaire component), `src/domain.js` (types).

---

## 0. CANONICAL SPEC — status, version, and where to read it

**This file (`public/ask.md`) is the single canonical, machine-readable spec for the
interactive-components format. There is exactly one renderer (the PassbackAI app, served on both
`passbackai.com` and `omgfixmd.com`) and exactly one schema. Every authoring side — our
`question-extractor` skill, VideoStudio AI's Elements/Story prompts, the studio tooling, and anything
added later — must agree with this file.**

### Where to read the spec (three faces, one source)

| Audience | Link | What it is |
|---|---|---|
| **Humans** | `https://passbackai.com/ask` | The rendered HTML page (`public/ask.html`). |
| **Agents / tools** | `https://passbackai.com/ask.md` | **This file**, raw Markdown — plain `curl`, `text/markdown`, `Access-Control-Allow-Origin: *`, no JS/cookies/UA gating. |
| **Programmatic validation** | `https://passbackai.com/schema.json` | JSON Schema (2020-12) for every component, for validating a payload before you send it. |

> **Sandboxed agents — reachability vs format are two problems.** `/ask.md` fixes *format* (clean
> Markdown, curl-able, CORS), but a sandbox whose egress allowlist doesn't include `passbackai.com`
> still can't *reach* it. So CI mirrors this exact file (and `schema.json`) to a **public GitHub repo**;
> `raw.githubusercontent.com` is allowlisted by default in most agent sandboxes, so the mirror is
> readable live with zero per-environment config. The mirror destination is configured by the
> `SCHEMA_MIRROR_REPO` repo variable (see `.github/workflows/mirror-schema.yml`); the public raw URL is
> the same content, same commit. `passbackai.com/ask.md` stays the canonical home.

| | |
|---|---|
| **Spec status** | Canonical. Authoritative over any skill README, prompt, or tribal note. |
| **Wire `version`** | `"1"` for **every** component block. The only value any validator accepts (a number `1` is rejected). It has never moved; new capabilities ship as additive optional fields, never a `version` bump. |
| **Current `skill_version`** | `1.5` (`LATEST_SKILL_VERSION` in `src/data/skill-changelog.js`). Authoring-side build tag — see the version model below. |
| **Spec revision** | `r6` · 2026-06-21 · hardened the **fence contract** ([§2.1](#21-not-block-types--do-not-invent-fence-tags)/[§2.2](#22-common-mistakes)): the only two block types, a "NOT block types" list of hallucinated tags, common mistakes, and a combined starter — the Markdown fence contract is now documented as strongly as the JSON one. r5: served at `/ask.md` + mirrored to a public repo; added `/schema.json`. r4: `/ask` reframed to the full family. r3: named the public link; documented `recommended`. |

### Governance — why this exists, and the one rule that keeps it true

This document is the **structural fix for renderer/authoring drift**. The drift it prevents: a team
authored JSON against one team's mental model of the schema (e.g. "there is no `recommended` field")
while the feature already shipped — because there was no single versioned link everyone pointed at.

> **The rule:** any change to the renderer or a validator (`src/utils/questionnaire.js`,
> `src/utils/prioritize.js`, `src/markdown.jsx` fence dispatch, `src/domain.js` types) **updates, in the
> same commit, all three published faces** — this file (`public/ask.md`), the HTML page
> (`public/ask.html`), and the JSON Schema (`public/schema.json`) — and bumps the **Spec revision** line
> above. CI then mirrors `ask.md` + `schema.json` to the public repo automatically. A schema change that
> lands while any face is stale is the bug — a stale page is exactly how the `recommended` drift stayed
> invisible. Treat it like the extension contract: these three files are the contract, not the code's
> private knowledge.

### The version model — read this to avoid the exact drift above

Two numbers travel with a questionnaire, and conflating them is what caused the "v1.2 vs 1.5" split.
**They are different axes:**

| Field | What it versions | Values | Gates rendering? |
|---|---|---|---|
| `version` | **The wire schema** — the shape this doc describes. | Always the literal string `"1"`. | **YES.** A block whose `version` isn't `"1"` for its tag is rejected. |
| `skill_version` | **The authoring skill build** that emitted the JSON (questionnaire only). | `"1.5"` today; `"1.2"`, `"1.4"`, … are older builds of the *same* skill. | **NO.** Optional metadata. Drives only the in-app "your skill is outdated" banner + the export footer. Safe to bump or omit. |

So an "omgfixmd at 1.2" payload and a "passbackai at 1.5" payload are **the same schema** authored by
**two builds of the same skill**. Both render correctly on **both** domains, because `recommended`
(the 1.5 addition) is an additive, **graceful-ignored** optional field — an older payload that omits
it loses nothing; a payload that includes it renders the badge. There is no second schema and no
second renderer to reconcile; sync the authoring side by matching this doc, not by matching a domain.

---

## TL;DR — the whole loop

1. **You emit** a **Markdown document** containing one or more fenced interactive-component blocks
   (` ```questionnaire `, ` ```prioritize `), each holding bare JSON.
2. **User pastes it** into the textarea at **https://passbackai.com** → each block renders as an
   interactive widget inline in the document. No account, no login, nothing leaves the browser.
3. **User fills/orders it, clicks "Copy" (answers / their edits)** → gets a **Markdown** block.
4. **User pastes that Markdown back to you.** You parse it as their decisions. The loop is
   **copy-paste, manual** — there is no API.

Minimal valid document (one questionnaire, one prioritize):

````markdown
Pick the auth method and rank the rollout regions.

```questionnaire
{ "version": "1", "questions": [
  { "id": "q1", "question": "Which auth method?", "options": ["PIN", "Room number", "Magic link"] }
] }
```

```prioritize
{ "version": "1", "items": [
  { "id": "eu", "label": "EU" },
  { "id": "us", "label": "US" },
  { "id": "apac", "label": "APAC" }
] }
```
````

> **Back-compat shortcut.** A document that is **nothing but a single bare questionnaire JSON object**
> (no surrounding Markdown, optionally inside a ` ```json ` fence) is also accepted and renders as a
> standalone questionnaire form — this is the legacy single-form paste. It still works unchanged. The
> Prioritize component has **no** bare-JSON shortcut; it is only recognized inside a ` ```prioritize `
> fence within a document. See the [migration note](#8-migration--from-the-legacy-select-only-questionnaire).

---

## 1. HOW TO EMBED A COMPONENT (shared rules)

These rules are identical for every component type. The per-type schema follows in §3–4.

**1. The unit you paste is a Markdown document.** Write normal Markdown — headings, prose, lists —
and drop interactive components into it as fenced code blocks. They render inline, in document order.

**2. Embed a component as a fenced block whose info-string is the component's tag.** The opening
fence is the tag with nothing else on the line (` ```questionnaire ` or ` ```prioritize `); the body
is **bare JSON**; the closing fence is ` ``` `.

````markdown
```questionnaire
{ "version": "1", "questions": [ … ] }
```
````

If the JSON inside a tagged fence fails to validate, the app falls back to rendering it as a plain
code block (so a typo shows up as visible JSON, not a broken page).

**3. Bare JSON inside the fence — no nested ` ```json `.** Put the raw JSON object directly between
the tag fence and its closing fence. (A ` ```json ` fence *inside* the tag fence is tolerated by the
parser, but it's redundant — emit bare JSON.)

**4. Straight ASCII quotes only.** Use `"` for every key and string — **never** `"` `"` `‚` `'`.
Many chat surfaces "smart-quote" straight quotes, which breaks `JSON.parse` and makes the block
render as raw text instead of the widget. The parser attempts a smart-quote rescue on the structural
double-quotes, but don't rely on it — emit straight quotes. Before sending, mentally run each block
through `JSON.parse`: balanced `{ }` / `[ ]`, no trailing commas.

**5. Mix and repeat freely.** A document may contain any number of component blocks, of mixed types,
interleaved with prose. Each renders independently and each contributes its own section to the
copied-back Markdown, in document order.

**6. `version` is the per-block SCHEMA version.** Every component's JSON carries `version: "1"`
(the literal string; a number `1` is rejected). It versions that component's wire format, not your
generator. See [§9 stability](#9-stability--versioning).

**Render flow.** Open **https://passbackai.com** → paste the document into the textarea → each tagged
block becomes its widget inline. **No account, anonymous, fully client-side — the document never
leaves the browser.** There is no hosted-form URL, embeddable widget, or preload param: it's the
paste surface.

---

## 2. COMPONENT INDEX

**There are exactly TWO renderable block types. This is the whole set:**

| § | Component | Fence tag (the ONLY valid ones) | Choose it when you need the user to… |
|---|---|---|---|
| [§3](#3-component-questionnaire--choose) | **Questionnaire** | ` ```questionnaire ` | pick from options, multi-select, or type a free-text answer |
| [§4](#4-component-prioritize--order) | **Prioritize** | ` ```prioritize ` | rank / reorder a short list by priority |

### 2.1 NOT block types — do not invent fence tags

A fenced block whose info-string is anything other than the two above renders as a **plain code block,
silently, with no error** — so a wrong tag looks like it "did nothing". None of these are real (a
non-exhaustive list of tempting hallucinations):

` ```multiple-choice ` · ` ```open-question ` · ` ```match ` · ` ```quiz ` · ` ```poll ` ·
` ```survey ` · ` ```form ` · ` ```rank ` · ` ```question ` · ` ```answer ` — **and anything else.**
The renderer knows only `questionnaire` and `prioritize`.

### 2.2 Common mistakes

- **Question *types* are MODES inside one ` ```questionnaire ` block — not separate blocks.**
  Single-select, multi-select (`multi: true`), and free-text (`options: []`) are the same question shape,
  listed together in the one `questions` array. **Do not** create separate fenced blocks for "multiple
  choice", "open question", or "matching" — every question, of any mode, goes inside the single
  `questionnaire` block. (See [§3.2 Question modes](#32-question-modes).)
- **Ranking is the separate ` ```prioritize ` component**, not a question mode. Everything a user
  *chooses* is a questionnaire question; the one thing they *order* is prioritize.
- **One info-string, lowercase, nothing after it** — the opening fence is exactly ` ```questionnaire ` or
  ` ```prioritize ` on its own line.

### 2.3 Copy-paste starter — prose + one prioritize + one questionnaire

A single document mixing prose with both block types. Renders top-to-bottom: a line of prose, a draggable
priority list, then a three-question form (single-select with a recommended badge, a multi-select, and a
free-text field).

````markdown
Here's the Q3 plan. Rank the regions, then answer the two open calls.

```prioritize
{ "version": "1", "items": [
  { "id": "eu", "label": "EU" },
  { "id": "us", "label": "US" },
  { "id": "apac", "label": "APAC" }
] }
```

```questionnaire
{ "version": "1", "questions": [
  { "id": "q1", "question": "Which auth method?", "options": ["PIN", "Room number", "Magic link"], "recommended": "Magic link" },
  { "id": "q2", "question": "Which regions need localization first?", "multi": true, "options": ["EU", "US", "APAC"] },
  { "id": "q3", "question": "Target launch date?", "options": [], "open_field": { "label": "Target date:" } }
] }
```
````

---

## 3. COMPONENT: Questionnaire — *choose*

A list of questions. Each question is a pick-from-options control (single- or multi-select) with an
always-present free-text escape hatch; a question with **no** options is a pure free-text field.

**Source of truth:** `isValidQuestionnaire` / `isValidQuestion` / `parseQuestionnaire` /
`buildAnswersExport` in `src/utils/questionnaire.js`; type `Questionnaire` in `src/domain.js`.

### 3.1 Schema

**Top-level envelope** (validated in `isValidQuestionnaire`):

| Key | Type | Required | Notes |
|---|---|---|---|
| `version` | string | **YES** | Must be the literal string `"1"`. A number `1` is rejected. |
| `questions` | array | **YES** | Non-empty array of question objects (below). |
| `title` | string | no | Shown as the form header. |
| `source_summary` | string | no | One-line subtitle under the title. |
| `skill_version` | string | no | Build tag of your generator (e.g. `"1.5"`); echoed into the export footer. Powers an in-app "update available" banner. Strings only. |
| `routing` | object | no | Include only when the user is sending the form to **someone else** (see §5). |

**`routing`** (when present, both sub-fields validated):

| Key | Type | Required | Notes |
|---|---|---|---|
| `from` | string | **YES**, non-empty | Sender's name. Auto-bolded in the UI and used in the answers header. |
| `return_prompt` | string | **YES** | Instruction text. **Must contain the literal value of `from`** so the renderer can bold it. |

**Per-question object** (`isValidQuestion`):

| Key | Type | Required | Notes |
|---|---|---|---|
| `id` | string | **YES**, non-empty | Stable unique id. Keys the answer state. |
| `question` | string | **YES**, non-empty | The prompt shown to the user. (The key is `question` — **never** `q`.) |
| `options` | string[] | **YES** | Array of plain strings — see §3.3. Use `[]` for a pure free-text question (then `open_field` is required). |
| `context` | string | no | One-line subtitle explaining why the question is open. |
| `section` | string | no | Groups **consecutive** same-`section` questions under a heading. |
| `multi` | boolean | no | `true` → multi-select (checkboxes). Default/absent → single-select (radio). See §3.2. |
| `recommended` | string \| string[] | no | The option **label** you'd nudge toward (an array of labels when `multi`). Shows a "Recommended" badge; does **not** pre-select. Must match an option label exactly (graceful-ignored otherwise). The *why* belongs in `context`. |
| `open_field` | object | no | Renames the always-present "Other" row; **required** when `options` is `[]`. See §3.3. |
| `allow_note` | boolean | no | Cosmetic only — every option-bearing question always has a note toggle. |

### 3.2 Question modes

There is **one question shape**; the `multi` boolean and an empty `options` array select the mode.
There is **no `type` field.**

| Mode | How to encode | Renders as | Answer in export |
|---|---|---|---|
| **Single-select** | `multi` absent/`false`, non-empty `options` | Radio buttons | one `Answer:` line |
| **Multi-select** | `multi: true`, non-empty `options` | Checkboxes + "Select all that apply" badge | `Answers:` + bullet list |
| **Free text** | `options: []` + an `open_field` | A single labeled text field (no radios) | `Answer: {label} {text}` |

Model a rating/scale or yes/no as option labels (e.g. `["1","2","3","4","5"]`, `["Yes","No"]`).
Note: a **note** field is also on every option-bearing question, and a note alone counts as a valid
answer. (Ranking/ordering is a **separate component** — see [§4](#4-component-prioritize--order) — not a question mode.)

### 3.3 Options and the "Other" row

**Options are bare strings, not objects.** `options: ["A", "B", "C"]`. Each string is both the
displayed label and the returned value. There is **no** `value` or `description` sub-field — an
option *object* is rejected.

**"Other" / free text is automatic — never put "Other" inside `options`.** Every option-bearing
question renders an Other row below the options:

- **Default:** omit `open_field`. The row shows a localized label (`Other:` / `אחר:` / `أخر:`) and
  unlocks the "edit-into-Other" pencil gesture.
- **Custom label:** set `open_field` to redirect the escape hatch (this also disables the pencil gesture):

```json
"open_field": { "label": "I need to check with:", "placeholder": "e.g. legal team" }
```

`open_field.label` is required when `open_field` is present; `placeholder` is optional. For a pure
free-text question (`options: []`), `open_field` is **required** and its field becomes the question's
sole input.

### 3.4 Worked example

````markdown
```questionnaire
{
  "version": "1",
  "skill_version": "1.5",
  "title": "Guest Check-In, Open Decisions",
  "source_summary": "Open decisions pulled from the mobile check-in brief.",
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
      "section": "Compliance",
      "question": "What is the data retention policy?",
      "context": "Brief notes legal sign-off pending.",
      "options": ["Delete after checkout", "30 days", "1 year", "Follow hotel data policy"],
      "recommended": "Follow hotel data policy",
      "open_field": { "label": "I need to check with:", "placeholder": "e.g. legal team, DPO" }
    },
    {
      "id": "q3",
      "section": "Compliance",
      "question": "Which guest data fields do we collect at check-in?",
      "context": "Brief is silent on the field list.",
      "multi": true,
      "options": ["Full name", "ID/passport number", "Phone", "Email", "License plate", "Estimated arrival time"]
    },
    {
      "id": "q4",
      "question": "What is the target launch date?",
      "context": "Brief never states a date — a free-form answer, not a choice.",
      "options": [],
      "open_field": { "label": "Launch date:", "placeholder": "e.g. Q3, or a specific date" }
    }
  ]
}
```
````

### 3.5 Answers output

After filling the form the user clicks **"Copy answers"** (or Ctrl/Cmd-Enter), which copies a
**Markdown** block (`buildAnswersExport`). One entry per **answered** question (unanswered ones are
omitted); entries separated by `---`.

- Single-select: `Answer: {label}`.
- Multi-select with one pick: `Answer: {label}`; with 2+ picks: `Answers:` then `- {label}` bullets.
- "Other" / free-text pick: `Answer: {open_field.label} {typed text}`.
- A non-empty note adds a `Note: {text}` line; a note-only answer emits just the `Note:` line.
- Header: `Answers → {routing.from}` if routing is set, else `## {title}` if a title exists, else nothing.
- Footer (if `skill_version` was present): `<!-- question-extractor:v{skill_version} -->`.

```markdown
Answers → Elad

**Q: What authentication method should guests use at check-in?**
Answer: Room number + PIN

---

**Q: What is the data retention policy?**
Answer: I need to check with: legal team
Note: confirm with DPO before launch

---

**Q: Which guest data fields do we collect at check-in?**
Answers:
- Full name
- Email
- Estimated arrival time

<!-- question-extractor:v1.5 -->
```

> **For your parser:** the entry header is `**Q: <exact question text>**`. There's no machine `id` in
> the output, so match answers back by the question text you sent (keep your `question` strings
> stable), or just feed the whole Markdown block to your model as the user's decisions.

### 3.6 Shape guards — the silent breakers

These are the validation rules in `isValidQuestionnaire` / `isValidQuestion` that, if violated, make a
block **fail validation and fall through to the plain Markdown reviewer** (the form silently doesn't
render) — except the last row, which validates but mis-keys answer state. Author against these exactly.

| Guard | Rule | What happens if you break it |
|---|---|---|
| **`version`** | Must be the literal **string** `"1"`. A number `1` fails. | Whole block rejected → renders as JSON, not a form. |
| **`questions`** | Non-empty array. | Rejected. |
| **`id`** | Every question needs a **non-empty string** `id`. | Rejected. |
| **`question` key** | The prompt key must be **`question`** — never `q`, `text`, `prompt`, `title`. (`q` alone is auto-recovered as a single forgiven alias; nothing else is.) | Rejected (banner pinpoints the question). |
| **`options` are plain strings** | `options` must be an array and **every entry a string**. An option *object* (`{ "label": …, "recommended": true }`) fails. There is no per-option `value`/`description`/`recommended`. | Rejected — this is the classic "object-form options" miss. |
| **No `"Other"` inside `options`** | The Other / free-text row is added automatically. Don't author it. | Not a hard reject, but produces a duplicate, dead "Other" entry; use `open_field` to rename the real one. |
| **Empty `options`** | `options: []` is valid **only** with an explicit `open_field` (pure free-text question). | `[]` with no `open_field` is rejected. |
| **`recommended`** | Optional. A **string** (single-select) or **string[]** (`multi`). Must **verbatim-match** an existing option **label** — value(s) that don't match are silently dropped (no badge). At most one effective pick for single-select. Mark-only: it shows a "Recommended" badge and **never** pre-selects. Never echoed into the copied-back answers. | Never rejects the block (graceful-ignore); a typo just shows no badge. |
| **Duplicate `id`s** | Ids should be **unique** across questions. *Not* enforced by the questionnaire validator (unlike Prioritize, which hard-rejects dupes), but answer state is keyed by `id`. | Two questions with the same `id` **collide** — they share one answer slot. Silent data corruption; keep ids unique. |

---

## 4. COMPONENT: Prioritize — *order*

A short list the user **ranks** by dragging or with ▲/▼ controls, top = highest priority, plus one
optional note for the whole list. Unlike a questionnaire, a prioritize block carries a **default**
(the order you authored), so it **always** reports back what the user did — including "kept the order
unchanged" (an affirmative endorsement).

**Source of truth:** `isValidPrioritize` / `parsePrioritize` / `buildPrioritizeExport` in
`src/utils/prioritize.js`; type `Prioritize` in `src/domain.js`.

### 4.1 Schema

**Top-level envelope** (validated in `isValidPrioritize`):

| Key | Type | Required | Notes |
|---|---|---|---|
| `version` | string | **YES** | Must be the literal string `"1"`. |
| `items` | array | **YES** | Non-empty array of item objects with **unique** `id`s (below). The array order is your **suggested** order. |
| `title` | string | no | Shown as the header / export header. |
| `instruction` | string | no | Helper line under the title (e.g. "Order by priority — top = first."). |
| `note_field` | object | no | Config for the one optional block-level note: `{ "enabled": boolean, "label": string }`. Defaults to enabled with label "Add a note (optional)". Set `"enabled": false` to hide it. |
| `routing` | object | no | Same shape as the questionnaire's `routing` (`from` + `return_prompt`). Include only when sending to someone else. |

**Per-item object** (`isValidItem`):

| Key | Type | Required | Notes |
|---|---|---|---|
| `id` | string | **YES**, non-empty | Stable, **unique** across items. Keys the answer order. |
| `label` | string | **YES**, non-empty | The item text the user sees and reorders. |
| `context` | string | no | A quiet one-line caption; also appended to the export line as `— {context}`. |

### 4.2 Worked example

````markdown
```prioritize
{
  "version": "1",
  "title": "Q3 rollout order",
  "instruction": "Order by priority — top ships first.",
  "items": [
    { "id": "eu", "label": "EU", "context": "largest existing waitlist" },
    { "id": "us", "label": "US", "context": "highest revenue per seat" },
    { "id": "apac", "label": "APAC", "context": "needs localization first" }
  ],
  "note_field": { "label": "Anything blocking this order?" },
  "routing": {
    "from": "Elad",
    "return_prompt": "Act in this order."
  }
}
```
````

### 4.3 Answers output

A prioritize block **always** serializes (`buildPrioritizeExport`), tagged with what the user did:

- Header carries the tag — `reader reordered` or `reader kept the order unchanged`:
  - routing present → `Priority order ({tag}) → {routing.from}`
  - title only → `## {title} ({tag})`
  - neither → `({tag})` on its own first line.
- A numbered list (`1.`, `2.`, …) in the user's **final** order; each line appends `— {context}` when
  the item carries one.
- A single trailing `Note: {text}` line **only** when the user wrote a note.
- Footer: `routing.return_prompt` (e.g. "Act in this order.") when routing is present.

```markdown
Priority order (reader reordered) → Elad

1. US — highest revenue per seat
2. EU — largest existing waitlist
3. APAC — needs localization first

Note: APAC slips if localization isn't staffed by July.

Act in this order.
```

> **For your parser:** the tag in the header (`reader reordered` vs `reader kept the order unchanged`)
> tells you whether the user actively changed your suggested order or endorsed it. The numbered list
> is always the user's resolved order — act on it either way.

---

## 5. ROUTING (shared)

Both components accept the same optional `routing` object — use it **only** when the document is being
sent to **someone else** (not the user answering for themselves):

```json
"routing": { "from": "Elad", "return_prompt": "When done, send your answers back to Elad." }
```

`routing.return_prompt` **must contain the literal value of `routing.from`** so the renderer can bold
the name. Omit `routing` entirely for self-answering.

---

## 6. CONSTRAINTS (shared)

| Topic | Answer |
|---|---|
| **Conditional / branching logic** | **Not supported.** Flat, linear; every block and question is always shown. |
| **Markdown inside a component's question/option/label text** | **Not rendered** — shown literally (no bold, links, code). (Prose *outside* the fenced blocks is normal Markdown.) |
| **RTL / Hebrew / Arabic** | **Fully supported and automatic.** Direction + chrome strings (`en`/`he`/`ar`) are auto-detected from the content. **Do not** set any language/`dir` field. |
| **Questionnaire — max questions** | No hard limit in the app. The authoring skill caps at **12** (pick the most blocking). |
| **Questionnaire — options per question** | Validator requires the `options` key (may be `[]` for free-text); skill guidance is **3–4** short, distinct labels. |
| **Prioritize — item ids** | Must be **unique** within the block (they key the order). |

---

## 7. THE FAMILY IS OPEN — adding a component

This format is designed to **grow**. New component types ship as new fence tags with their own JSON
schema and their own copy-back serialization — additively, without changing the ones above. Planned
components (design notes in `docs/ideas/interactive-blocks.md`) include **suggested-change**
(accept/reject/edit a rewrite), **verify** (confirm/correct claims), and **tradeoff/scale** (a slider
on a spectrum).

**When you integrate, code to the family, not to one example:** check this section's component index
(§2) for the tags that exist *today*, and pick the component that matches the decision you need —
don't assume "questionnaire" is the only option, and don't hard-code a single block type. When a new
component lands, a new subsection appears here between §4 and this one; your integration picks it up
by adding the new tag, nothing else changes.

---

## 8. MIGRATION — from the legacy select-only "questionnaire"

Earlier versions of this guide described a single thing: a **questionnaire** = a select-from-list
form, emitted as a bare JSON object. That framing is now one *member* of the interactive-components
family. Nothing you already emit breaks; the model just got wider.

- **Your existing questionnaire JSON still works, unchanged.** A bare questionnaire JSON object (or a
  ` ```json ` fence) pasted on its own still renders as a standalone form — the [back-compat
  shortcut](#tldr--the-whole-loop). The questionnaire schema in §3 is byte-for-byte the same.
- **Prefer the fenced form for anything new.** Wrap the questionnaire JSON in a ` ```questionnaire `
  fence inside a Markdown document. That's the general embedding mechanism (§1), it lets you add
  prose around it, and it's how *every* component (including the ones that have no bare-JSON
  shortcut) is embedded.
- **Ranking is no longer "not supported."** If you previously simulated ordering as a multi-select or
  a list of "1st / 2nd / 3rd" options, switch to the [Prioritize](#4-component-prioritize--order)
  component — it's a first-class drag-to-rank widget that reports the resolved order.
- **You can now combine.** A single document can carry a questionnaire *and* a prioritize block (and
  prose), each contributing its own section to the copied-back Markdown.

---

## 9. STABILITY — versioning & breaking changes

- Each component's payload is **explicitly versioned** by its own `version: "1"`. A parser accepts
  only `"1"` for its tag; a future incompatible format ships as `"2"` and is handled separately, so a
  document authored against `"1"` stays valid.
- `skill_version` (questionnaire only) is **independent** generator-build metadata; it does **not**
  gate rendering — it drives the optional "your generator is outdated" banner and the export footer.
  Safe to bump or omit.
- New capabilities have so far been **additive optional fields** (`section`, `routing`, `multi`,
  `recommended`) and **new component types** (`prioritize`), never breaking changes to a shipped tag.

---

## 10. API vs MANUAL — is there a programmatic loop?

**No API. The round-trip is strictly copy-paste and stays fully manual:**

- **In:** the user pastes your document into https://passbackai.com. There is no endpoint to POST a
  document or a URL param to preload one.
- **Out:** the user copies the Markdown answers and pastes them back to you. There is no callback,
  webhook, or fetch-answers endpoint.
- Everything runs client-side in the browser by design (the document never leaves the tab), so build
  your integration around **"emit a document → instruct paste → ingest the Markdown the user pastes
  back."**

---

## Drop-in instruction for your tool's output

After emitting the document, tell the user (in their language) exactly this:

```
1. Copy the document above
2. Open https://passbackai.com
3. Click Paste
```

Then, when they return, ingest the pasted Markdown block(s) as their decisions.
