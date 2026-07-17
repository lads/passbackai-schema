# passbackai — interactive components format (integration spec)

> **For the implementing tool.** `passbackai.com` is the app — the **same app** also served on the legacy `omgfixmd.com`
> domain. PassbackAI is a way to author **beautiful, interactive documents for people *and* for AI tools**:
> a Markdown document with embedded decision widgets that a human fills in and an AI reads back as structured
> answers. This document is the exact, code-verified contract for those **interactive components** and the
> Markdown they return.
>
> **There are two ways to deliver such a document — pick by whether you are connected to PassbackAI over MCP:**
>
> | You are… | Deliver via | What you do |
> |---|---|---|
> | **Connected over MCP** (the AI-native path, **preferred**) | the **`route_document` MCP tool** | call `route_document` with a typed `blocks[]` array — the server validates each component as you generate it, writes the canonical fences for you, and returns a reviewer link (`/r/<id>`). No copy-paste, no smart-quote risk. |
> | **Not connected** (no MCP) | **paste** (the local fallback) | emit the Markdown document below and tell the user to paste it at **https://passbackai.com**. This is also the fallback when a route fails. |
>
> Both faces describe the **same components with the same JSON fields** — only the delivery differs. The
> sections below specify those fields once; [§0.5](#05-delivery--mcp-route-primary-paste-fallback) covers the
> MCP route in detail and [§10](#10-delivery-paths--mcp-route-vs-paste) compares the two.
>
> **Read this first — the model, not a single example.** An interactive component is a *typed
> decision request* embedded in a Markdown document. There is an **open, growing PALETTE** of them,
> not a single form — pick ONE primitive PER unclear point, by verb, and weave it into prose (prose
> itself is a response channel: the reviewer annotates it, so settled thinking stays prose). Shipped
> today:
>
> | Component | Fence tag | The verb | Use it to… |
> |---|---|---|---|
> | **Single choice** | ` ```single-choice ` | **choose one** | ask the user to pick exactly one of 2–4 options |
> | **Multi choice** | ` ```multi-choice ` | **choose many** | ask the user to pick all options that apply |
> | **Open question** | ` ```open-question ` | **write** | ask the user to type a free-text answer |
> | **Prioritize** | ` ```prioritize ` | **order** | ask the user to rank/reorder a short list by priority |
> | **Allocate** | ` ```allocate ` | **split** | ask the user to split a fixed whole (a budget/effort/resource) across categories by weight |
> | **Questionnaire** | ` ```questionnaire ` | **group** | present a batch of 3+ tightly-related questions as one unit |
> | **YouTube** | ` ```youtube ` | **watch** | show a video (display-only, no answer; plays on the routed reviewer surface only) |
> | **Mermaid** | ` ```mermaid ` | **watch** | show a diagram (rendered client-side on every surface; the reviewer can comment on a flowchart node, edge, or the whole diagram) |
>
> More are planned (see [§7](#7-the-family-is-open--adding-a-component)). **Do not assume the family
> is "a questionnaire"** — the questionnaire is the GROUP primitive, not the default vessel for every
> ambiguity. Pick the component that matches the decision you need from the user; you can embed
> several, of mixed types, in one document.
>
> Source of truth: `src/utils/component-registry.js` (the registry every consumer iterates — one entry per
> component), `src/markdown.jsx` (`parseBlocks` — the fence dispatch), `src/utils/questionnaire.js`,
> `src/utils/prioritize.js` and `src/utils/question-primitives.js` (per-component parser/validator/export),
> `public/skills/passback.md` (the authoring skill), `src/domain.js` (types). The MCP delivery path is
> `api/_lib/mcp-tools.js` (the `route_document` / `get_components_spec` / `list_responses` tools) and
> `api/_lib/route-document-blocks.js` (the typed `blocks[]` union — the same components as typed arguments).

---

## 0. CANONICAL SPEC — status, version, and where to read it

**This file (`public/ask.md`) is the single canonical, machine-readable spec for the
interactive-components format. There is exactly one renderer (the PassbackAI app, served on both
`passbackai.com` and `omgfixmd.com`) and exactly one schema. Every authoring side — our
`question-extractor` skill, VideoStudio AI's Elements/Story prompts, the studio tooling, and anything
added later — must agree with this file.**

### Where to read the spec (four faces, one source)

| Audience | Link | What it is |
|---|---|---|
| **Humans** | `https://passbackai.com/ask` | The rendered HTML page (`public/ask.html`). |
| **Agents / tools (read)** | `https://passbackai.com/ask.md` | **This file**, raw Markdown — plain `curl`, `text/markdown`, `Access-Control-Allow-Origin: *`, no JS/cookies/UA gating. |
| **Programmatic validation** | `https://passbackai.com/schema.json` | JSON Schema (2020-12) for every component, for validating a payload before you send it. |
| **Connected models (author)** | the **`route_document` MCP tool** | The AI-native delivery face. Its description inlines a compact, schema-derived shape for every component so a connected model can author the typed `blocks[]` directly; `get_components_spec` returns this full file on demand. See [§0.5](#05-delivery--mcp-route-primary-paste-fallback). |

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
| **Current `skill_version`** | `3.3` (`LATEST_SKILL_VERSION` in `src/data/skill-changelog.js`). Authoring-side build tag — see the version model below. |
| **Spec revision** | `r17` · 2026-07-17 · **Allocate list edits** ([§4.5](#45-component-allocate--split), [§4.5.3](#453-answers-output)): the reviewer can now **delete a category** from a ` ```allocate ` split (reversible; its weight redistributes proportionally among the rest so the sum stays `total`; reported back as an explicit `Removed by reader:` line) and **add categories of their own**, entering at weight 0 inside the **same** split (reported back marked `— (added by reader)`); the header tag gains `reader edited the list`. Still **AUTHORING-UNCHANGED** — the payload stays {version, items[, total, unit, title, instruction, note_field, routing]}; list edits are a review-side answer (leaf-free, keyed by item ids, so leaf numbering is untouched). Additive; nothing you author changes. r16 · 2026-07-17 · **Prioritize list edits** ([§4](#4-component-prioritize--order), [§4.3](#43-answers-output)): the reviewer can now **delete an item** from a ` ```prioritize ` list (reversible; reported back as an explicit `Removed by reader:` line) and **add items of their own**, ranked inside the **same** list (reported back marked `— (added by reader)`); the header tag gains `reader edited the list`. Still **AUTHORING-UNCHANGED** — the payload stays {version, items[, title, instruction, note_field, routing]}; list edits are a review-side answer (leaf-free, keyed by item ids, so leaf numbering is untouched). Additive; nothing you author changes. r15 · 2026-07-16 · **Diagram authoring guidance** ([§2.2](#22-common-mistakes), [§4.7](#47-component-mermaid--watch)): teaches WHEN/HOW an authoring model should reach for a ` ```mermaid ` diagram — the **reach trigger** (you're describing a STRUCTURE the reader must hold in their head: a process / flow / mechanism / sequence / state machine / decision tree / architecture / dependency order / timeline), the **comparison PAIR** (two diagrams side by side — before→after, A vs B — the highest-value move when a doc weighs a change), **restraint** (a diagram earns its place only when the structure is genuinely hard to hold in prose; three linear steps stay prose), and the **payoff** (the reviewer comments on the exact node/edge, per [§4.7.3](#473-answers-output)). A diagram is a **comprehension** aid, a different axis from the one-primitive-per-open-decision weave law. **GUIDANCE-ONLY** — the authoring payload is UNCHANGED (still `source`-only, {version, source, title}); nothing you author changes. r14 · 2026-07-16 · **Mermaid edge + whole-diagram comments** ([§4.7](#47-component-mermaid--watch), [§4.7.3](#473-answers-output)): a reviewer can now also anchor a comment to a specific **flowchart EDGE (line + its label)** and to the **WHOLE diagram**, in addition to a node. Still **AUTHORING-UNCHANGED** — `source`-only ({version, source, title}); edge/whole-diagram commenting is a review-side answer (edges keyed by the edge's source-derived endpoints, whole-diagram keyed to nothing; LEAF-FREE, so leaf numbering is untouched). Additive; nothing you author changes. r13 · 2026-07-15 · **Mermaid node comments** ([§4.7](#47-component-mermaid--watch), [§4.7.3](#473-answers-output)): a reviewer can now anchor a comment to a specific **flowchart NODE (box)** of a rendered ` ```mermaid ` diagram, not only the caption. The **AUTHORING payload is UNCHANGED** — still `source`-only ({version, source, title}); node commenting is a review-side answer keyed by the node id already in your `source` (LEAF-FREE, so leaf numbering is untouched). Only flowchart/graph diagrams expose per-node commenting; other types are commented on like prose. Additive; nothing you author changes. r12 · 2026-07-15 · **Mermaid** ([§4.7](#47-component-mermaid--watch)): a ` ```mermaid ` **embed rich media** component — verb **watch** — a diagram (flowchart/sequence/ER/…) rendered **client-side on every surface** (paste, `#s=`, routed alike) from a `source`-only payload — the diagram is drawn by an on-demand, off-budget renderer under the unchanged strict CSP, and the produced SVG is sanitized before display. Additive; nothing existing changed. r11 · 2026-07-14 · **YouTube** ([§4.6](#46-component-youtube--watch)): the first **embed rich media** component — verb **watch** — a ` ```youtube ` block that shows a video. DISPLAY-ONLY (no answer; the reviewer comments on it like prose) and **routed-only** (plays click-to-load on the routed reviewer surface via `route_document`; inert on locally-pasted / `#s=` documents, which never contact YouTube). Additive; nothing existing changed. r10 · 2026-07-12 · **the two-sided weave law + the admission gate** ([§2.2](#22-common-mistakes), [§7](#7-the-family-is-open--adding-a-component)): the settled-vs-open rule stated with a FLOOR (an open point always becomes a component; `open-question` is the floor, never a demotion to prose) as well as the ceiling (settled stays prose) — closing the under-use gap; plus the discriminator `allocate` (magnitude) vs `prioritize` (order), and the meta-gate for future components (a new primitive must name a decision-shape not yet covered, with an isomorphic gesture). Additive; nothing existing changed. r9 · 2026-07-12 · **Allocate** ([§4.5](#45-component-allocate--split)): a new ` ```allocate ` component — verb **split** — the magnitude sibling of `prioritize` (order → weight). The user drags weighted bars that always sum to `total` (100% by default) to split a fixed whole across categories; the resolved weights + deltas serialize back. Additive; nothing existing changed. r8 · 2026-07-02 · **the palette** ([§2](#2-component-index--the-palette)): three standalone question primitives — ` ```single-choice ` (choose one), ` ```multi-choice ` (choose many), ` ```open-question ` (write) — join `prioritize` (order) and `questionnaire`, which is REFRAMED as the **group** primitive (a batch of 3+ tightly-related questions), no longer the default vessel for every ambiguity. Pick ONE primitive per unclear point and weave it into prose; prose itself is a response channel. All three new shapes are additive; nothing existing changed. r7 · 2026-06-26 · added the **MCP delivery path** ([§0.5](#05-delivery--mcp-route-primary-paste-fallback)/[§10](#10-delivery-paths--mcp-route-vs-paste)): when connected over MCP, deliver via the `route_document` tool with a typed `blocks[]` array (server-validated, server-serialized fences, returns a reviewer link) — paste is now the **local fallback**, not the only loop. Corrected the former "there is no API" claim. <!-- authoring-faces:allow — this revision note must quote the corrected phrase --> r6: hardened the **fence contract** ([§2.1](#21-not-block-types--do-not-invent-fence-tags)/[§2.2](#22-common-mistakes)). r5: served at `/ask.md` + mirrored to a public repo; added `/schema.json`. r4: `/ask` reframed to the full family. r3: named the public link; documented `recommended`. |

### Governance — why this exists, and the one rule that keeps it true

This document is the **structural fix for renderer/authoring drift**. The drift it prevents: a team
authored JSON against one team's mental model of the schema (e.g. "there is no `recommended` field")
while the feature already shipped — because there was no single versioned link everyone pointed at.

> **The rule:** any change to the renderer or a validator (`src/utils/questionnaire.js`,
> `src/utils/prioritize.js`, `src/markdown.jsx` fence dispatch, `src/domain.js` types) **updates, in the
> same commit, all four published faces** — this file (`public/ask.md`), the HTML page
> (`public/ask.html`), the JSON Schema (`public/schema.json`), and the **`route_document` MCP tool surface**
> (`api/_lib/mcp-component-spec.js`, derived from `schema.json`) — and bumps the **Spec revision** line
> above. CI enforces this: `scripts/check-mcp-components.js` fails if a `schema.json` component is missing
> from the MCP tool surface, and `scripts/check-authoring-faces.js` fails if this file or the
> question-extractor skill stops naming the `route_document` / `blocks` delivery path (or reasserts the old
> "there is no API" claim). <!-- authoring-faces:allow — describing the guard requires naming the phrase it bans --> CI then mirrors `ask.md` + `schema.json` to the public repo automatically. A
> change that lands while any face is stale is the bug — a stale page is exactly how the `recommended` drift
> stayed invisible. Treat it like the extension contract: these four faces are the contract, not the code's
> private knowledge.

### The version model — read this to avoid the exact drift above

Two numbers travel with a questionnaire, and conflating them is what caused the "v1.2 vs 1.5" split.
**They are different axes:**

| Field | What it versions | Values | Gates rendering? |
|---|---|---|---|
| `version` | **The wire schema** — the shape this doc describes. | Always the literal string `"1"`. | **YES.** A block whose `version` isn't `"1"` for its tag is rejected. |
| `skill_version` | **The authoring skill build** that emitted the JSON (any primitive may carry it). | `"3.3"` today; `"1.2"`, `"1.5"`, … are older builds of the *same* skill. | **NO.** Optional metadata. Drives only the in-app "your skill is outdated" banner + the export footer. Safe to bump or omit. |

So a "1.2" payload and a "1.5" payload are **the same schema** authored by
**two builds of the same skill**. Both render correctly on **both** domains, because `recommended`
(the 1.5 addition) is an additive, **graceful-ignored** optional field — an older payload that omits
it loses nothing; a payload that includes it renders the badge. There is no second schema and no
second renderer to reconcile; sync the authoring side by matching this doc, not by matching a domain.

---

## 0.5 DELIVERY — MCP route (primary), paste (fallback)

PassbackAI has **one tool** with **two ingress faces** — a document arrives by **paste** (local origin) or by
**MCP route** (server-stored, opened at `/r/<id>`). They are equally first-class; which you use depends only on
whether you, the authoring model, are connected to PassbackAI over MCP.

### When you ARE connected over MCP — use `route_document` (preferred)

Call the **`route_document`** tool with a typed **`blocks[]`** array. Each block is **either** prose or one
interactive component, passed as **typed JSON fields** (not a string you format):

| Block | Shape (the `type` is the discriminant) |
|---|---|
| **Prose** | `{ "type": "markdown", "text": "…any Markdown prose…" }` |
| **Questionnaire** | `{ "type": "questionnaire", "version": "1", "questions": [ … ] }` |
| **Prioritize** | `{ "type": "prioritize", "version": "1", "items": [ … ] }` |
| **Allocate** | `{ "type": "allocate", "version": "1", "items": [ { "id": …, "label": …, "weight": N }, … ] }` |
| **YouTube** | `{ "type": "youtube", "version": "1", "id": "<11-char id>", "title": "…" }` |
| **Mermaid** | `{ "type": "mermaid", "version": "1", "source": "graph TD; A-->B;", "title": "…" }` |

The component fields (`questions`, `items`, every option, `routing`, …) are **identical** to the fenced JSON
specified in §3–§4 — the only difference is the wrapper carries a `type` and you pass it as an argument
instead of writing a fence. Then:

- **The server writes the fences.** You never emit ` ```questionnaire ` syntax — `route_document` serializes
  each block into the canonical document with `JSON.stringify`, in `blocks` order.
- **The component is validated as you generate it.** Because it arrives as typed arguments, the platform
  validates the shape at generation time — so **there is no smart-quote risk on this path** (the entire
  smart-quote concern in §1/§3 is a **paste-path-only** hazard; it cannot occur here).
- **You get a reviewer link back.** `route_document` returns a `/r/<id>` link the creator previews and then
  shares; the routed doc adopts into the recipient's collection exactly like a `#s=` share link.
- **`markdown` is the prose-only shortcut.** For a document with **no** component you may pass a single
  `markdown` string instead of `blocks`. **`blocks` takes precedence** if both are given, and a component
  fence hand-written inside a `markdown` string is **not** validated (it may degrade to a plain code block) —
  so use `blocks` for anything with a component.

**Reading answers back is also MCP-native — no copy-paste, no webhook.** Call **`list_responses`** for a
document you routed; it returns each reviewer response as leaf-anchored `annotations[]` + leaf-free
`componentInputs[]` (plus the verdict). This is a **pull** (you poll), not a callback. The reviewer's app
submits via `POST /api/v1/reviewer/documents/:id/responses` — that is the app's own endpoint, not a tool you
call. For the full field semantics on demand, call **`get_components_spec`** (it returns this file verbatim).

> **Privacy demarcation — keep it exact.** The **local** paste → review → export loop and **sharing** (the
> `#s=` URL hash fragment) never transmit the document. **Routing is the one deliberate, opt-in, server-side
> egress:** `route_document` stores the routed doc on the PassbackAI backend (Postgres + KMS, **server-managed
> keys — NOT zero-knowledge**). Never describe a routed document as end-to-end-encrypted. Routing is always
> creator-initiated, never silent or default.

### When you are NOT connected — paste (the local fallback)

No MCP connection (or a route failed)? Fall back to the **paste** loop: emit the Markdown document (§1–§4) and
tell the user to paste it at `https://passbackai.com`. This is the fully local, no-backend-touch path — the
document never leaves the browser. It is unchanged and fully supported; it is simply the fallback when the
AI-native route isn't available.

---

## TL;DR — the whole loop

**The AI-native loop (connected over MCP — preferred):**

1. **You call `route_document`** with a typed `blocks[]` array (prose + component blocks, §0.5). The server
   validates each component, writes the canonical fences, and returns a `/r/<id>` reviewer link.
2. **The creator previews/shares the link; the reviewer fills it in** in the same rendered app.
3. **You call `list_responses`** to pull their answers back (`annotations[]` + `componentInputs[]`) — a pull,
   no copy-paste, no webhook.

**The local loop (no MCP — the fallback):**

1. **You emit** a **Markdown document** containing one or more fenced interactive-component blocks
   (` ```questionnaire `, ` ```prioritize `), each holding bare JSON.
2. **User pastes it** into the textarea at **https://passbackai.com** → each block renders as an
   interactive widget inline in the document. No account, no login, nothing leaves the browser.
3. **User fills/orders it, clicks "Copy" (answers / their edits)** → gets a **Markdown** block.
4. **User pastes that Markdown back to you.** You parse it as their decisions.

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

**4. Straight ASCII quotes only — a PASTE-PATH concern.** Use `"` for every key and string — **never**
`"` `"` `‚` `'`. Many chat surfaces "smart-quote" straight quotes, which breaks `JSON.parse` and makes the
block render as raw text instead of the widget. The parser attempts a best-effort rescue on input that fails
to parse — it normalizes structural smart double-quotes to `"` and strips a trailing comma before a
closing `}`/`]` — but don't rely on it; emit clean JSON. Before sending, mentally run each block
through `JSON.parse`: balanced `{ }` / `[ ]`, no trailing commas.
> **This whole rule applies only to the paste path** — the document you write as text. On the **MCP
> `route_document`** path ([§0.5](#05-delivery--mcp-route-primary-paste-fallback)) you pass the component as
> typed `blocks[]` arguments and the **server** serializes the JSON, so smart-quoting cannot occur and you
> never hand-write a fence. Prefer the MCP path whenever you are connected.

**5. Mix and repeat freely.** A document may contain any number of component blocks, of mixed types,
interleaved with prose. Each renders independently and each contributes its own section to the
copied-back Markdown, in document order.

**6. `version` is the per-block SCHEMA version.** Every component's JSON carries `version: "1"`
(the literal string; a number `1` is rejected). It versions that component's wire format, not your
generator. See [§9 stability](#9-stability--versioning).

**Render flow (paste path).** Open **https://passbackai.com** → paste the document into the textarea → each
tagged block becomes its widget inline. **No account, anonymous, fully client-side — the document never
leaves the browser.** On this paste path there is no embeddable widget or preload param: it's the paste
surface. (The **MCP `route_document`** path *does* return a hosted `/r/<id>` reviewer link — see
[§0.5](#05-delivery--mcp-route-primary-paste-fallback) — because that path deliberately stores the doc
server-side.)

---

## 2. COMPONENT INDEX — the palette

**The palette is OPEN and growing. Pick ONE primitive per unclear point, by verb — these are the renderable block types today:**

| § | Component | Fence tag | Verb — choose it when you need the user to… |
|---|---|---|---|
| [§2.5](#25-the-standalone-question-primitives) | **Single choice** | ` ```single-choice ` | **choose one** — pick exactly one of 2–4 listed options |
| [§2.5](#25-the-standalone-question-primitives) | **Multi choice** | ` ```multi-choice ` | **choose many** — pick all options that apply |
| [§2.5](#25-the-standalone-question-primitives) | **Open question** | ` ```open-question ` | **write** — type a free-text answer (a name, a date, a reason) |
| [§4](#4-component-prioritize--order) | **Prioritize** | ` ```prioritize ` | **order** — rank / reorder a short list by priority |
| [§4.5](#45-component-allocate--split) | **Allocate** | ` ```allocate ` | **split** — divide a fixed whole (budget / effort / resource) across categories by weight |
| [§3](#3-component-questionnaire--choose) | **Questionnaire** | ` ```questionnaire ` | **group** — a batch of 3+ tightly-related questions presented as one unit |
| [§4.6](#46-component-youtube--watch) | **YouTube** | ` ```youtube ` | **watch** — show a video (display-only, no answer; routed reviewer surface only) |
| [§4.7](#47-component-mermaid--watch) | **Mermaid** | ` ```mermaid ` | **watch** — show a diagram (rendered client-side on every surface; the reviewer can comment on a flowchart node, edge, or the whole diagram) |

**Prose is itself a response channel.** Settled thinking stays prose — the reviewer annotates it directly.
Weave the primitives INTO the prose, each one set up by the sentence before it; don't stack them into a form.

### 2.1 NOT block types — do not invent fence tags

A fenced block whose info-string is anything other than the palette above renders as a **plain code block,
silently, with no error** — so a wrong tag looks like it "did nothing". None of these are real (a
non-exhaustive list of tempting hallucinations):

` ```multiple-choice ` (the real tags are `single-choice` / `multi-choice`) · ` ```match ` · ` ```quiz ` ·
` ```poll ` · ` ```survey ` · ` ```form ` · ` ```rank ` · ` ```question ` · ` ```answer ` — **and anything
else.** The renderer knows only the tags in the index above.

### 2.2 Common mistakes

- **One decision → one standalone primitive, woven into prose.** Reach for ` ```questionnaire ` only to
  GROUP 3+ tightly-related questions into one unit; a lone question travels as its own
  ` ```single-choice ` / ` ```multi-choice ` / ` ```open-question ` block with a prose lead-in.
  (Inside a `questionnaire`, the same three forms exist as question modes — `multi: true`,
  `options: []` — see [§3.2 Question modes](#32-question-modes).)
- **Ranking is the separate ` ```prioritize ` component**, never a question mode: what a user
  *chooses* is a choice/questionnaire; the one thing they *order* is prioritize.
- **`allocate` vs `prioritize` — magnitude, not order.** `prioritize` answers *which comes first*
  (1st / 2nd / 3rd); `allocate` answers *how much each gets* (weights summing to a whole). If the
  reviewer would reply in **percentages or dollars**, it is `allocate`; in **rank**, `prioritize`.
- **The law is TWO-SIDED — settled vs open, a floor and a ceiling.** *Ceiling:* a decision the text
  already SETTLES stays prose — never wrap a made decision in a widget (density is the form-y feel).
  *Floor:* a decision left OPEN always becomes a component, and when no sharp verb fits it the floor
  is ` ```open-question `, **never demoted back to prose**. A document with real open points is never
  component-less; one that settles everything weaves none. Restraint means not componentizing the
  *settled* — it never means leaving the *open* unasked.
- **DISPLAY vs RESPONSE — the decision-point law is for RESPONSE components.** The verbs above
  (choose / order / split / write / group) map an *open decision* to a widget. A **display**
  component — today ` ```youtube ` (**watch**) — carries no decision: reach for it whenever the
  document *references* that media, decision or not. Embed a video with the ` ```youtube ` block;
  **never** hand-write a Markdown image-link or an `img.youtube.com` URL for a video — the thumbnail
  host is CSP-blocked (broken image) and a linked image is a click-out, not a player.
- **A DIAGRAM is COMPREHENSION, a different axis from the weave law.** The response verbs above map
  one OPEN decision to one widget; a ` ```mermaid ` diagram maps a **STRUCTURE the reader must hold in
  their head** — a process / flow / mechanism / sequence / state machine / decision tree / architecture
  / dependency order / timeline — to a picture, so they grasp it at a glance. Reach for one **at the
  moment the prose turns structurally dense**, bound to the sentence it clarifies; reach for **TWO side
  by side** to compare (before → after, option A vs B) **only when each side is itself a real structure**
  — a plain two-option pick ("lead with A or B", "Tue/Thu vs Mon/Wed") is a ` ```single-choice ` and a
  sentence, not two boxes-and-arrows; diagram the comparison only when the *shapes* differ in a way prose
  can't hold (current-vs-proposed architecture). A diagram never counts against the
  one-primitive-per-point rule and never replaces a component (the decision still rides its widget). The
  same restraint still applies — three linear steps stay prose; diagram only what is genuinely hard to
  hold. Full guidance + two worked examples: [§4.7](#47-component-mermaid--watch).
- **One info-string, lowercase, nothing after it** — the opening fence is exactly the component tag
  (e.g. ` ```single-choice `) on its own line.

### 2.3 Copy-paste starter — a woven document (prose + the palette)

One document, one primitive per unclear point, each set up by the prose before it. Renders top-to-bottom:
prose, a single choice (with a recommended badge), a draggable priority list, and a free-text answer.

````markdown
Here's the Q3 plan. Three calls are still open — each one below, in place.

First, the auth method. Magic link is the lightest to ship, which is why I lean there.

```single-choice
{ "version": "1", "question": "Which auth method?", "options": ["PIN", "Room number", "Magic link"], "recommended": "Magic link" }
```

Next, the regions. Drag into rollout order — top ships first.

```prioritize
{ "version": "1", "items": [
  { "id": "eu", "label": "EU" },
  { "id": "us", "label": "US" },
  { "id": "apac", "label": "APAC" }
] }
```

Last: the launch date is genuinely open — type what you're targeting.

```open-question
{ "version": "1", "question": "Target launch date?", "placeholder": "e.g. 2026-09-01" }
```
````

### 2.5 The standalone question primitives — `single-choice` / `multi-choice` / `open-question`

The per-ambiguity forms of a question: ONE decision per fence, flat fields, no envelope. All three share
the questionnaire's answer semantics (the renderer normalizes each to a one-question form internally), so
`recommended`, `open_field`, notes, and the export shape behave exactly as documented in [§3](#3-component-questionnaire--choose).

| Field | `single-choice` | `multi-choice` | `open-question` |
|---|---|---|---|
| `version` | required, `"1"` | required, `"1"` | required, `"1"` |
| `question` | required, non-empty | required, non-empty | required, non-empty |
| `options` | required, **2–4** strings | required, **2–6** strings | — (no options; the answer field IS the answer) |
| `recommended` | optional, ONE option label | optional, ARRAY of option labels | — |
| `placeholder` | — | — | optional, example text in the empty field |
| `open_field` / `context` / `id` / `routing` / `skill_version` | optional, as in §3 | optional, as in §3 | `context`/`id`/`routing`/`skill_version` optional |

```single-choice
{ "version": "1", "question": "Which auth method ships in v1?", "options": ["Magic link", "Room number + PIN"], "recommended": "Magic link" }
```

```multi-choice
{ "version": "1", "question": "Which locales at launch?", "options": ["English", "Hebrew", "Arabic"], "recommended": ["English", "Hebrew"] }
```

```open-question
{ "version": "1", "question": "What is the data-retention constraint from legal?", "placeholder": "e.g. delete 30 days after checkout" }
```

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
optional note for the whole list. The user can also **edit the list itself** — delete an item they
consider out of place (reversible; the deletion reports back explicitly) and **add items of their
own**, which they rank inside the **same** list. Unlike a questionnaire, a prioritize block carries
a **default** (the order you authored), so it **always** reports back what the user did — including
"kept the order unchanged" (an affirmative endorsement).

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

- Header carries the tag — `reader edited the list` (the user deleted and/or added items),
  `reader reordered` (moves only), or `reader kept the order unchanged`:
  - routing present → `Priority order ({tag}) → {routing.from}`
  - title only → `## {title} ({tag})`
  - neither → `({tag})` on its own first line.
- A numbered list (`1.`, `2.`, …) in the user's **final** order — the authored items minus any the
  user deleted, plus the user's own additions; each line appends `— {context}` when the item carries
  one, and a user-added line is marked `— (added by reader)` so you never mistake it for something
  you authored.
- One `Removed by reader: {label}; {label}` line below the list **only** when the user deleted
  authored items — an explicit negative signal (do not act on a dropped item just because you
  authored it).
- A single trailing `Note: {text}` line **only** when the user wrote a note.
- Footer: `routing.return_prompt` (e.g. "Act in this order.") when routing is present.

```markdown
Priority order (reader edited the list) → Elad

1. US — highest revenue per seat
2. Localization staffing — (added by reader)
3. EU — largest existing waitlist

Removed by reader: APAC

Note: APAC is off until localization is staffed.

Act in this order.
```

> **For your parser:** the tag in the header (`reader edited the list` vs `reader reordered` vs
> `reader kept the order unchanged`) tells you whether the user changed the list itself, only your
> suggested order, or endorsed it as-is. The numbered list is always the user's resolved list — act
> on it either way; treat `(added by reader)` lines as the user's own requirements and `Removed by
> reader:` items as explicitly declined.

---

## 4.5 COMPONENT: Allocate — *split*

A short list of categories the user **splits a fixed whole across** by dragging
weighted bars that always sum to `total` (100% by default), plus one optional
note for the whole split. The user can also **edit the list itself** — delete a
category they consider out of place (reversible; its weight redistributes
proportionally so the sum stays `total`, and the deletion reports back
explicitly) and **add categories of their own**, which enter at weight 0 inside
the **same** split. The magnitude sibling of Prioritize (order → weight):
like Prioritize it carries a **default** (the split you authored), so it
**always** reports back what the user did — including "kept the allocation
unchanged" (an affirmative endorsement). Use it when the point is the
*proportions* — a budget across teams, effort across workstreams, headcount
across functions.

**Source of truth:** `isValidAllocate` / `parseAllocate` / `buildAllocateExport`
in `src/utils/allocate.js`; type `Allocate` in `src/domain.js`.

### 4.5.1 Schema

**Top-level envelope** (validated in `isValidAllocate`):

| Key | Type | Required | Notes |
|---|---|---|---|
| `version` | string | **YES** | Must be the literal string `"1"`. |
| `items` | array | **YES** | Non-empty array of item objects with **unique** `id`s (below). |
| `total` | number | no | The fixed whole the weights sum to. **Default 100.** Provided weights are rescaled to this total. |
| `unit` | string | no | Display unit for a weight — `"%"` (default), `"$"`, `"pts"`, `"people"`. |
| `title` | string | no | Shown as the header / export header. |
| `instruction` | string | no | Helper line under the title. |
| `note_field` | object | no | One optional block-level note: `{ "enabled": boolean, "label": string }`. Defaults enabled. |
| `routing` | object | no | Same shape as the questionnaire's `routing` (`from` + `return_prompt`). |

**Per-item object:**

| Key | Type | Required | Notes |
|---|---|---|---|
| `id` | string | **YES**, non-empty | Stable, **unique** across items. Keys the resolved weights. |
| `label` | string | **YES**, non-empty | The category text the user sees and reweights. |
| `weight` | number | **YES** | The category's suggested share of `total` (>= 0). Rescaled so the set sums to `total`. |
| `context` | string | no | A quiet one-line caption; also appended to the export line as `— {context}`. |

### 4.5.2 Worked example

````markdown
```allocate
{
  "version": "1",
  "title": "Q3 engineering capacity",
  "instruction": "Drag each bar to reweight — the rest rebalance to keep 100%.",
  "unit": "%",
  "total": 100,
  "items": [
    { "id": "product", "label": "Product — new features", "weight": 35, "context": "viz components, sharing" },
    { "id": "growth", "label": "Growth & GTM", "weight": 25 },
    { "id": "infra", "label": "Infra & reliability", "weight": 15, "context": "accumulated tech debt" },
    { "id": "support", "label": "Customer support", "weight": 15 },
    { "id": "hiring", "label": "Hiring", "weight": 10 }
  ],
  "note_field": { "label": "Anything blocking this split?" },
  "routing": { "from": "Elad", "return_prompt": "Plan against this split." }
}
```
````

### 4.5.3 Answers output

An allocate block **always** serializes (`buildAllocateExport`), tagged with what
the user did:

- Header carries the tag — `reader edited the list` (the user deleted and/or added
  categories), `reader reweighted` (weight moves only), or `reader kept the
  allocation unchanged`:
  - routing present → `Allocation ({tag}) → {routing.from}`
  - title only → `## {title} ({tag})`
  - neither → `({tag})` on its own first line.
- One bullet per category in the user's **final** list — the authored categories
  minus any the user deleted, plus the user's own additions — with its final
  weight + unit; an authored line appends `(was N{unit})` when the weight moved or
  `(unchanged)` when it didn't, plus `— {context}` when the category carries one.
  A user-added line is marked `— (added by reader)` instead of a delta (there is
  no authored weight to compare against) so you never mistake it for something
  you authored.
- One `Removed by reader: {label}; {label}` line below the list **only** when the
  user deleted authored categories — an explicit negative signal (its weight was
  redistributed proportionally; do not budget for a dropped category just because
  you authored it).
- A `Total: {total}{unit}` line closes the list (the invariant the reader held).
- A single trailing `Note: {text}` line **only** when the user wrote a note.
- Footer: `routing.return_prompt` when routing is present.

```markdown
Allocation (reader edited the list) → Elad

- Product — new features: 30% (was 35%) — viz components, sharing
- Growth & GTM: 20% (was 25%)
- Infra & reliability: 25% (was 15%) — accumulated tech debt
- Customer support: 15% (unchanged)
- Security review: 10% — (added by reader)

Removed by reader: Hiring

Total: 100%

Note: security can't slip again; hiring waits a quarter.

Plan against this split.
```

> **For your parser:** the tag in the header (`reader edited the list` vs
> `reader reweighted` vs `reader kept the allocation unchanged`) tells you whether
> the user changed the list itself, only your weights, or endorsed the split
> as-is. The bullet weights are always the resolved split — act on them either
> way; treat `(added by reader)` lines as the user's own budget lines and
> `Removed by reader:` categories as explicitly declined.

---

## 4.6 COMPONENT: YouTube — *watch*

The first **embed rich media** component (a growing family): a YouTube video the
reviewer watches inside the document. Unlike every other component it is
**display-only** — it collects **no answer**. The reviewer reacts to it the same
way they react to prose: by commenting on the caption and the surrounding text.
Use it to show, not ask — a walkthrough, a demo clip, a reference the decision
depends on.

**Delivery is routed-only.** A YouTube block plays **only on the routed reviewer
surface** (a document delivered via `route_document`). On a locally-pasted or
`#s=` shared document the block renders an **inert preview** and never contacts
YouTube — the "your document never leaves your browser" promise stays literally
true there. On the routed surface it is **click-to-load**: nothing reaches
YouTube until the reviewer presses play, and playback uses the privacy-enhanced
`youtube-nocookie` origin (no cookies until play).

**Source of truth:** `isValidYouTube` / `parseYouTube` in `src/utils/youtube.js`;
type `YouTube` in `src/domain.js`; render + runtime gate in
`src/components/EmbeddedYouTube.jsx`.

### 4.6.1 Schema

| Key | Type | Required | Notes |
|---|---|---|---|
| `version` | string | **YES** | Must be the literal string `"1"`. |
| `id` | string | **YES** | The **11-character** YouTube video id (e.g. `dQw4w9WgXcQ`) — matches `^[A-Za-z0-9_-]{11}$`. **Pass the id only, never a URL or query string** — the site builds the embed URL. |
| `title` | string | no | Caption shown under the video and used as the iframe title. The **one anchorable line** for comments. Clamped to 200 chars. |
| `thumbnail` | string | no | Optional preview image as a base64 `data:` image URI (`data:image/png|jpeg|webp;base64,…`), downscaled (~320×180) and under ~40 KB. An invalid or oversized value is **dropped** and a neutral facade shown. Never a live image URL — no network fetch. |

### 4.6.2 Worked example

````markdown
```youtube
{
  "version": "1",
  "id": "dQw4w9WgXcQ",
  "title": "Onboarding walkthrough (v2) — the 90-second tour"
}
```
````

### 4.6.3 Answers output

A YouTube block contributes **nothing** to the copy-out bundle — it has no
answer. The reviewer's feedback on it arrives as **ordinary leaf-anchored
comments** on its caption and the prose around it, in the normal annotations
stream. There is no `# Answers:` entry for a `youtube` block.

---

## 4.7 COMPONENT: Mermaid — *watch*

An **embed rich media** component (the same family as YouTube): a **Mermaid
diagram** the reviewer reads inside the document — a flowchart, sequence diagram,
ER diagram, gantt, and the rest of the Mermaid grammar. Use it to show a shape the
decision depends on — an architecture, a flow, a state machine — not to ask.

**The reviewer can anchor a comment to a specific FLOWCHART NODE (box), to a
specific EDGE (line + its label), or to the WHOLE diagram**, in addition to
commenting on the caption and the surrounding prose. This is a review-side answer,
not something you author: **the AUTHORING payload is unchanged — still `source`-only**
(see §4.7.1). You do not name, id, or mark nodes/edges for commenting; the reviewer
clicks a box or a line and the app keys the comment to the node/edge id derived from
the `source` you already wrote (a whole-diagram comment is keyed to nothing). Only
**flowchart** / **graph** diagrams expose per-node and per-edge commenting; other
diagram types (sequence, class, state, …) can still take a whole-diagram comment and
are otherwise commented on like prose (caption + surrounding text).

**Delivery is source-only, rendered on every surface.** The payload carries the
Mermaid **`source` text only** — there is no server-rendered image and no
inert/rendered distinction. Every surface (local paste, `#s=` share, routed
`/r/<id>`) **lazy-loads the renderer and draws the diagram client-side** from
`source`. The heavy renderer is fetched **on demand** (only when a document
contains a diagram) as a same-origin script — the strict CSP is unchanged.
Rendering is **fail-closed**: an invalid source degrades to the raw source in a
code block, and the produced SVG is **sanitized** (no scripts, no event
handlers, no external references) before it is shown.

**Source of truth:** `isValidMermaid` / `parseMermaid` in `src/utils/mermaid.js`;
type `Mermaid` in `src/domain.js`; render in `src/components/EmbeddedMermaid.jsx`
(the off-budget renderer is `src/mermaid-entry.js` → `/js/mermaid.js`).

### 4.7.0 When to reach for a diagram — the authoring guidance

**The reach trigger.** Emit a ` ```mermaid ` block whenever you are describing something the reader must
hold in their head as a **STRUCTURE** — a process, a flow, a mechanism, a sequence, a state machine, a
decision tree, an architecture, a dependency order, a timeline. Prose walks such a shape one clause at a
time and forces the reader to reassemble it; a diagram shows it at once. Place it **at the moment the
prose turns structurally dense**, immediately after the sentence it clarifies — never in an appendix at
the end. This is a **comprehension** aid, a different axis from the decision-components' one-primitive-
per-open-point weave law ([§2.2](#22-common-mistakes)): a diagram asks nothing; it makes the shape
graspable so the reviewer can react to the *right* thing.

**The power of TWO — compare with a PAIR.** When the document weighs options or a change, **two diagrams
side by side** is often the single highest-value move: `before → after`, `current vs proposed`,
`option A vs option B`. The reader sees exactly what moves. **Earn the pair, though: each side must be
itself structurally non-trivial** — a real flow / architecture / state machine per side. A plain
two-option choice ("lead with A or B", "publish Tue/Thu vs Mon/Wed") stays a ` ```single-choice ` plus a
sentence, **not** two boxes-and-arrows; reach for the pair only when the *shapes* differ in a way prose
can't hold (current-vs-proposed architecture). The *decision itself* still rides a component
(a ` ```single-choice ` after the pair); the diagrams only make the choice legible.

**Restraint — don't diagram the trivial.** A diagram earns its place only when the structure is
genuinely hard to hold in prose. Three linear steps ("paste → review → copy back") stay prose; a
fan-out, a branch, a cycle, or a multi-actor sequence is what a diagram is *for*. A document peppered
with diagrams of obvious things reads as noise — the same failure as a document peppered with widgets
reading as a form.

**The payoff — the reviewer points at the exact box.** Because a rendered flowchart is commentable **per
node, per edge, and as a whole** ([§4.7.3](#473-answers-output)), a diagram turns a vague "this
mechanism is unclear" into a comment pinned to the one box or arrow that is wrong. That is the "genius —
and now I can point at the unclear part" moment the diagram exists to create.

### 4.7.1 Schema

| Key | Type | Required | Notes |
|---|---|---|---|
| `version` | string | **YES** | Must be the literal string `"1"`. |
| `source` | string | **YES** | The Mermaid diagram **source text** (e.g. `graph TD; A-->B;`). Non-empty and under ~8 KB; a longer source is rejected and the fence renders as plain code. |
| `title` | string | no | Caption shown under the diagram. The **one anchorable line** for comments. Clamped to 200 chars. |

### 4.7.2 Worked example

````markdown
```mermaid
{
  "version": "1",
  "source": "graph TD; A[Paste] --> B[Review] --> C[Copy back];",
  "title": "The review loop"
}
```
````

### 4.7.3 Answers output

If the reviewer anchored comments to the diagram, the Mermaid block contributes a
`# Answers:` entry — whole-diagram comments first, then one bullet per commented
**node** and **edge**, each naming what it is pinned to:

```markdown
## The review loop (diagram)

- Whole diagram — this reads top-down but should be a loop
- Node `A` — this box should be "Draft", not "Paste"
- Edge `A→B` — make this arrow dashed
```

A **node** bullet names the source node id; an **edge** bullet names its endpoints
(`from→to`) or the edge's own label; a **whole-diagram** bullet is anchored to
nothing. Comments on the caption and the surrounding prose still arrive as
**ordinary leaf-anchored comments** in the normal annotations stream, as before. A
diagram the reviewer left untouched contributes **nothing** — there is no
`# Answers:` entry for it. All of these are LEAF-FREE (keyed by the node/edge id
derived from your `source`, or by nothing for a whole-diagram note), so they never
affect leaf numbering.

### 4.7.4 Worked examples — a mechanism, and a comparison pair

**Example A — a mechanism that clicks.** The prose below is correct but dense: the reader has to
simulate the branching in their head. Dropping the diagram in right after it makes the read-path
graspable at a glance — and the reviewer pins their concern to the exact node.

The dense prose it clarifies:

> On every read we check the edge cache first. On a hit we serve it. On a miss we go to origin — but if
> origin is slow we serve the last good copy and refresh in the background, and only when there is no
> cached copy at all do we block the reader on origin.

````markdown
```mermaid
{
  "version": "1",
  "source": "graph TD; R[Read request] --> C{In edge cache?}; C -->|hit| S[Serve cached]; C -->|miss| O{Origin healthy?}; O -->|healthy| F[Fetch, cache, serve]; O -->|slow| B[Serve stale + refresh in background]; O -->|no cached copy| W[Block on origin];",
  "title": "Read path — cache, origin, and the stale fallback"
}
```
````

What comes back when the reviewer commented on one **node**:

```markdown
## Read path — cache, origin, and the stale fallback (diagram)

- Node `B` — the "slow" branch should also cover a 5xx from origin, not just latency
```

That is the payoff: "this mechanism is unclear" became a comment pinned to the exact box.

**Example B — the comparison PAIR.** The change is "move JWT validation from every service to the
gateway." One diagram of *today* and one of *proposed*, side by side, shows exactly what moves — then a
single choice captures the actual decision (the diagrams clarify; the widget decides).

The prose it clarifies:

> Today every service validates the JWT itself, so each one carries the auth client and re-checks with
> the auth service. The proposal: the gateway validates once and passes a signed header the services
> trust.

````markdown
```mermaid
{ "version": "1", "source": "graph LR; U[User] --> G[Gateway]; G --> S1[Service A]; G --> S2[Service B]; S1 --> A[Auth service]; S2 --> A;", "title": "Current — every service re-validates" }
```

```mermaid
{ "version": "1", "source": "graph LR; U[User] --> G[Gateway]; G -->|signed header| S1[Service A]; G -->|signed header| S2[Service B]; G --> A[Auth service];", "title": "Proposed — the gateway validates once" }
```

The proposed shape is simpler but makes the gateway a single point of trust. Ship it in v1?

```single-choice
{ "version": "1", "question": "Move JWT validation to the gateway in v1?", "options": ["Yes — gateway validates, services trust the signed header", "No — keep per-service validation for now"], "recommended": "Yes — gateway validates, services trust the signed header" }
```
````

The reviewer can now comment on the **`signed header` edge** in the *Proposed* diagram ("what signs it —
short-lived key? rotation?") — a question that was invisible in the prose, and a decision they express
in the single choice below the pair.

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

> **The admission gate — what keeps every primitive feeling *inevitable*.** A new component is
> justified only when it names a **decision-shape not already in the palette**, with an interaction
> **gesture isomorphic to that shape** (order → drag rank; magnitude → drag length; membership →
> check; position-on-a-scale → drag a point). A candidate that merely re-skins a shape already
> covered is rejected — it would make a reviewer feel a component was *forced*. This is why the
> palette reads as a grammar of decisions rather than a gallery of widgets: each primitive is the
> *shape* of a decision, not decoration on one. (The reverse test for authors — pick the primitive
> whose gesture matches the decision, or fall to `open-question` — is §2.2's two-sided law.)

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

## 10. DELIVERY PATHS — MCP route vs paste

There are **two** ways to deliver a document and read answers back. Pick by whether you are connected to
PassbackAI over MCP. Both carry the **same components with the same JSON fields**; only the transport differs.

| | **MCP route** (preferred when connected) | **Paste** (local fallback) |
|---|---|---|
| **In** | Call `route_document` with typed `blocks[]` ([§0.5](#05-delivery--mcp-route-primary-paste-fallback)). Server validates the component, writes the fences, returns a `/r/<id>` link. | Emit the Markdown document; the user pastes it at https://passbackai.com. No POST endpoint, no preload param on this path. |
| **Out** | Call `list_responses` to **pull** the reviewer's `annotations[]` + `componentInputs[]`. A poll, not a webhook. | The user copies the Markdown answers and pastes them back to you. |
| **Smart-quote risk** | None — typed args, server-serialized. | Yes — emit straight ASCII quotes (§1.4). |
| **Where the doc lives** | Stored on the backend (opt-in, server-managed keys, **not** zero-knowledge). | Never leaves the browser. |

- **There IS a programmatic path: the `route_document` MCP tool** (plus `list_responses` to read answers and
  `get_components_spec` for the full schema on demand). It is the AI-native delivery and the preferred one
  whenever a model is connected.
- **The paste loop is the local fallback** — fully client-side, the document never leaves the tab — for when
  there is no MCP connection, or when a route fails.
- So build your integration around: **connected → `route_document` (typed `blocks[]`) → `list_responses`;
  not connected → emit a document → instruct paste → ingest the Markdown the user pastes back.**

---

## Drop-in instruction for your tool's output

**If you are connected over MCP:** don't print paste instructions — call `route_document` with the typed
`blocks[]`, then hand the user the `/r/<id>` link it returns and (when you want their answers) call
`list_responses`.

**If you are NOT connected (paste fallback):** after emitting the document, tell the user (in their language)
exactly this:

```
1. Copy the document above
2. Open https://passbackai.com
3. Click Paste
```

Then, when they return, ingest the pasted Markdown block(s) as their decisions.
