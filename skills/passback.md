---
name: passback
description: Route any document into PassbackAI to collect structured feedback, and pull the answers back. ONE skill, two jobs — ROUTE (author one woven document that gives EVERY unclear point the interaction primitive that fits — single choice, multi choice, open question, or ranking — while settled thinking stays prose the reviewer annotates; then deliver it) and PULL (read back what reviewers answered on a doc you sent). There is no "review vs ask" mode — a doc with zero open points weaves zero components, one full of them weaves many — the unit of thinking is the individual ambiguity, never the document. Use whenever someone wants feedback on a draft, wants messy input turned into precise decision requests, wants to know what came back on a routed doc, or says "passbackai" / "/passback" — OR proactively, when a working conversation has itself surfaced 3+ unresolved open decisions that would be better answered in one interactive page. In ANY language (e.g. Hebrew "תוציא שאלות פתוחות").
license: Proprietary. See https://passbackai.com
metadata:
  owner: Elad Diamant
  author: elad-diamant
  version: "3.6"
  created: "05-05-2026"
  updated: "2026-07-15"
  triggers: "route this for review; get feedback on this draft; extract open questions; what's still unclear; turn this into a questionnaire; create a passback doc; what came back on the doc I sent; did anyone answer; passbackai; the same intents in any language (e.g. Hebrew תוציא שאלות פתוחות)"
---

# /passback — one woven document, a primitive per ambiguity

**What PassbackAI is.** One tool that closes a feedback loop: you **route** a document (prose woven with interactive decision widgets) to a reviewer, the reviewer answers in a beautiful interactive surface, and you **pull** their answers back as structured data. The reviewer is often the user themselves; sometimes it is someone they hand the link to.

**The model (the one idea everything else follows from).** The unit of thinking is the **individual unclear point**, never the document. For EACH ambiguity — a decision not made, missing info, a conflict, an undefined scope line — you pick the ONE interaction primitive that lets the reviewer communicate their intent most easily. Settled thinking stays **prose**, and prose is itself a response channel: the reviewer annotates it directly (that is the `annotations[]` half of every response; component answers are the `componentInputs[]` half). So there is **no "review mode vs ask mode"**: a polished draft weaves zero components and collects annotations; a messy brief weaves many; most real documents sit in between — and ALL of them are one woven document.

## This skill has two jobs

| Job | The user wants… | You do… |
|---|---|---|
| **ROUTE** | a document delivered for feedback — whatever its mix of settled prose and open points | scan for unclear points, give each one its primitive (or none), weave, deliver |
| **PULL** | to know what came back on a doc they already sent | call `list_responses` and synthesize — **never author a new document** |

Detect PULL from the request ("what came back", "did anyone respond", "pull the feedback on <doc>"). Everything else is ROUTE. **Do not ask the user to choose a mode** — the old REVIEW-vs-ASK question is retired; the document's own content decides how much gets woven. Ask a clarifying question only when a SPECIFIC point can't be shaped (see "Targeted clarifications" below).

## When to offer it — the proactive trigger

You don't only fire on an explicit request. When a **working conversation has itself surfaced 3+ unresolved open points** — decisions deferred ("we'll decide later"), competing options nobody picked, missing inputs you keep having to assume, "we still need to figure out…" — **offer, once, to route them**: don't drip the questions one-by-one in chat, and don't wait to be asked. Say it in a single line — that you can weave the open points into one PassbackAI page they (or a colleague) can answer in one pass — e.g. *"There are 5 open points here — want me to turn them into one PassbackAI doc you can answer in one pass?"*

**One offer, not a nag.** If they decline or ignore it, drop it for the rest of the thread. For 1–2 quick factual gaps, just ask inline — this trigger is for when the open points are **several and decision-shaped**. On "yes" → run ROUTE below.

## The palette — pick by verb, one primitive per point

The live palette is whatever the `route_document` tool surface (or `get_components_spec`) advertises — **new primitives appear there first; trust it over this table.** Shipped today:

| Primitive | Verb | Reach for it when the reviewer must… |
|---|---|---|
| *prose* | **annotate** | react to settled thinking — no widget; write the thinking well and they comment on it |
| `single-choice` | **choose one** | pick exactly one of 2–4 genuinely distinct options |
| `multi-choice` | **choose many** | pick all that apply (features, regions, stakeholders) |
| `open-question` | **write** | type a free answer (a name, a date, a reason, a description) |
| `prioritize` | **order** | rank ≥3 concrete, comparable, already-existing peers |
| `allocate` | **split** | divide a fixed whole (budget / effort / headcount) across categories — the point is HOW MUCH, by weight |
| `questionnaire` | **group** | work through 3+ TIGHTLY-related questions as one unit (one decision cluster) |
| `mermaid` | **watch** | read a diagram the decision depends on — display-only, no answer (a `source`-only Mermaid block rendered client-side; the reviewer comments on it like prose) |

The ladder, applied per point: *pick one of known options → `single-choice` · pick several → `multi-choice` · order a shortlist of ≥3 peers → `prioritize` · split a fixed whole by weight → `allocate` · genuinely open, nothing to enumerate → `open-question` · 3+ questions that truly form ONE decision cluster → `questionnaire` · thinking already settled → prose.* A two-way "ranking" is a `single-choice` (picking which goes first). **`allocate` vs `prioritize`:** order is *which comes first*; allocate is *how much each gets* (magnitudes summing to a whole). If the reviewer would answer in **percentages or dollars**, it's `allocate`; if in **1st/2nd/3rd**, it's `prioritize`. Don't manufacture filler options to force a choice where the point is open — that's an `open-question`. Don't split a real cluster (e.g. four fields of one config) into four lonely blocks — that's a `questionnaire`.

**Response vs display — two kinds of block.** The ladder above is for **response** primitives: each maps one *open decision* to a widget the reviewer answers. Some blocks are **display** — they carry no decision and collect no answer; they embed media the reviewer reacts to and annotates like any prose (today **`youtube`** and **`mermaid`**, both verb **watch**). Reach for **`youtube`** whenever the document *references* a video: embed it as a `youtube` block — `{ "type": "youtube", "version": "1", "id": "<11-char id>" }` — and **never** hand-write a Markdown image link or an `img.youtube.com` URL for a video (that thumbnail host is CSP-blocked, so it renders as a broken image, and a linked image is a click-out, not an inline player). Display blocks sit **outside** the settled-vs-open law below: a referenced video (or diagram) is embedded whether or not anything about it is undecided.

**The law is two-sided — a floor AND a ceiling, and both matter.**

- **Ceiling (against a form-y feel):** a decision the text already *settles* stays prose. Never wrap a made decision in a widget; density is what makes a doc feel like a form.
- **Floor (against a hollow doc):** a decision the text leaves *open* always becomes a component — and when no sharp verb fits it, the floor is `open-question`, **never a demotion back to prose**. An open point dropped to prose is a missed question, not restraint. So a doc with real open points is *never* component-less; only a doc that genuinely settles everything weaves none.

The line is **settled vs open**, not "fits a shape vs doesn't." Restraint means not componentizing the *settled* — it never means leaving the *open* unasked.

## JOB: ROUTE — scan, shape, weave, check, deliver

**1. Scan the decision space, not the text.** The marked gaps ("TBD", "we haven't decided X", "ask the team", conflicts between stated preferences) are the easy half — a text-only scan finds only what the author already knows is open. The gaps that make a document feel *sharp* are the unmarked ones, and they surface from a three-move scan:

1. **Reconstruct the goal** — in one line, what must this document's work succeed at? (Ship the feature; win the account; get legal sign-off.)
2. **Enumerate what the goal requires deciding** — independent of what the text says: the decisions this kind of work cannot proceed without.
3. **Diff** that list against what the text actually settles. What's missing splits into: decisions made **silently** (the text assumes an answer without flagging it — surface it as a component with the silent assumption as the `recommended`) and decisions **never made** (nothing in the text covers them — the reviewer is the only one who can).

Skip: style preferences with no consequence, questions answered later in the same text, pure implementation details (unless the input IS a tech spec). Cap the woven components at **12** — if more surface, keep the most blocking.

**2. Shape** each point with the ladder above — and hold options to the decision-space bar: the options of a `single-choice`/`multi-choice` must **partition the plausible answers** (mutually exclusive for single, collectively covering what a reasonable reviewer might pick — the built-in Other row catches the tail, it doesn't excuse a missing mainstream option). Every option label names a *position*, not a mood — "Room number + PIN", never "Something more secure". Zero points found? Then the document routes as pure prose — that IS the right output, not a failure to extract.

**3. Weave — an argument, not an alternation.** The reviewer should read a short document whose decision points arrive in the order the *work* needs them, each set up with exactly what's needed to answer it well. The rules that make that feel:

- **Order by what blocks what — and name the hinge.** Put the decision the others depend on first, and say that it's the hinge ("everything below assumes an answer here"). When one answer to X would moot Y, say so in Y's lead-in ("if you picked magic link above, skip this one"). Never default to source-text order.
- **Prose before every component — context, tradeoff, falsifier.** One–three sentences that set the point up: the context, what each real option *costs* (what breaks or gets harder if it's wrong), and — when you set `recommended` — the *why* behind the lean plus what would change your mind ("I'd flip to magic link if support load matters more than checkout speed"). A lead-in that states the tradeoff lets the reviewer beat a coin flip without leaving the page; a lead-in that only restates the question is filler. The explanation lives in this lead-in, NOT in the per-question `context` field (it renders below the prompt; leave it empty or a terse one-line caption).
- **Open with 1–3 sentences of plain framing** — who it's from, what it's for, that none of it is a test — and close with the send-back line. Framing lives in prose: the embedded renderer does not display `title` / `routing.return_prompt`, so never rely on those fields to be seen.
- **One point per component.** Never lump unrelated questions into one `questionnaire` — that's the form feel this skill exists to kill.
- **Settled prose is claims, not narrative.** Write the settled parts as short, explicit assertions the reviewer can cheaply confirm or contest ("We're assuming EU data residency is out of scope for v1") — crisp annotation targets, not a story. Prose is a first-class response channel; give it edges.

**3.5 Check — read it as the reviewer.** Before delivering, one pass through the woven document *as the recipient, cold*: for each component — is this the real question or a proxy for it? Can they answer it better than a coin flip from what's on the page alone? Would their answer actually unblock the work? Fix or cut whatever fails; if a needed fact lives only in your head, it belongs in the lead-in.

**4. Deliver** — by whether you are connected to PassbackAI over MCP:

- **Connected (PREFERRED):** call **`route_document`** with a typed **`blocks[]`** array — the woven sequence, in reading order: `{ "type": "markdown", "text": "…" }` for each prose run, and each component as its own typed block (`{ "type": "single-choice", "version": "1", "question": "…", "options": […] }`, etc.). The platform **validates each component as you generate it, writes the canonical fences for you, and returns a reviewer link** (`/r/<id>`). No smart-quote risk on this path. Stamp `"skill_version": "3.6"` on the first component block (and it's harmless on all). If the result carries **`weave.hints`**, treat them as review notes on your weaving: fix what they name, call `route_document` again with the corrected blocks, share the NEW link, and `revoke_document` the old id.
- **NOT connected, or a route fails:** fall back to **paste** — emit the entire woven Markdown document inside **one single outer code fence** (see the Output contract), then the three-line paste instruction. Fully local; the document never leaves the browser.

> **The component fields are identical on both paths** — only delivery differs. The **full, code-verified component schema (every field, all primitives) lives at <https://passbackai.com/ask>** (raw: `/ask.md`, JSON Schema: `/schema.json`). Read it there rather than expecting this file to restate it.

> **Privacy — keep it accurate.** The local **paste** loop never transmits the document. **Routing over MCP is an opt-in, server-side egress** — `route_document` stores the routed doc on the PassbackAI backend (server-managed keys, **not** zero-knowledge). Never describe a routed document as end-to-end-encrypted.

### Component field notes (the renderer's contract, compressed)

- `version` is always the string `"1"` (the schema version); `skill_version` is this skill's `"3.6"` — different fields.
- **`single-choice` / `multi-choice`:** `question` (never the key `q`), `options` (2–4 short, genuinely distinct labels; 2–6 for multi), optional `recommended` (a label for single, an ARRAY of labels for multi — set it whenever you have an honest lean, which is most of the time; the badge marks *which*, your lead-in prose says *why*; must exactly match option labels). Don't put "Other" in `options` — the renderer adds a localized Other row with an edit-into-Other gesture; a custom `open_field.label` ("I need to check with:") replaces it only when the escape-hatch genuinely needs a directed phrase.
- **`open-question`:** `question` + optional `placeholder`. The answer field IS the answer — no options.
- **`prioritize`:** `items` (unique `id` + `label` each); the array order is your suggested starting order; optional `title`/`instruction`.
- **`allocate`:** `items` (unique `id` + `label` + a numeric `weight` each — your proposed split); optional `total` (default 100), `unit` (`"%"` default, `"$"`, `"pts"`), `title`/`instruction`. The reviewer drags weighted bars that always sum to `total`.
- **`questionnaire`:** the group envelope — `questions[]` where each question carries `id`, `question`, `options` (may be `[]` + `open_field` for free-text), optional `multi`/`recommended`/`section`.
- **`youtube` (display — no answer):** `id` is the **11-char video id ONLY** — never a full URL or query string; pass the bare id even for a Shorts or `youtu.be` link (the site builds the privacy-enhanced embed). Optional `title`. It renders a click-to-load player; the reviewer comments on it — and the prose around it — like any passage. No `routing`/answer fields.
- **Routing to someone else:** when the doc is sent to another person, include `routing: { "from": "<sender>", "return_prompt": "…send back to <sender>." }` on the first component (the `return_prompt` must contain the literal `from` value) — and say it in the opening/closing prose too, which is what the reviewer actually sees. Self-answer → omit `routing`. If the conversation doesn't say which, ask once: "Are you answering these yourself, or sending them to someone else?"
- **RTL:** auto-detected from content. Hebrew/Arabic input → Hebrew/Arabic chrome. Don't set a language field.

### Targeted clarifications — the only questions you ask first

Never ask a generic upfront question about modes or formats. Ask (once, briefly, together) only when:
- a SPECIFIC point can't be shaped — e.g. "I can't tell if these four items are alternatives to pick from or a sequence to order — which?"
- self-answer vs send-to-someone is genuinely unclear (see routing above).

## JOB: PULL — read back the answers, then synthesize

The user asks what came back on a doc they already routed. **Do not author a new document.**

- **Connected:** call **`list_responses`** with the document id. It returns each reviewer's `verdict`, leaf-anchored `annotations[]` (their comments on the prose), and leaf-free `componentInputs[]` (their component answers) — a pull, no webhook. Then **synthesize for the user**: the verdict, the structured answers, and the free-form notes in their own language; surface conflicts and anything still unanswered.
- **Not connected:** the answers come back as the reviewer's **pasted export bundle**. Read it and synthesize the same way.

If you don't know the document id, ask the user which routed document they mean.

## Output contract & validation — PASTE path only

On the `route_document` path the platform validates the typed `blocks[]` for you, so the hardening below does **not** apply — but the field shapes are still the shapes you pass as typed arguments.

**Output two things, in this exact order, nothing else:**
1. **The whole woven Markdown document, inside ONE single outer code fence** — the copy-once/paste-once contract: one code block, one copy button, one paste into PassbackAI.
   - **Use a FOUR-backtick outer fence** (` ```` `) so it can contain the inner three-backtick component fences without the nesting breaking. No info-string on the outer fence. The copy button strips the outer markers, so the clipboard receives the exact document, inner fences intact.
   - Inside it: the interleaved structure — prose paragraphs with each component in its **own inner fence** (` ```single-choice `, ` ```open-question `, ` ```prioritize `, ` ```allocate `, ` ```multi-choice `, ` ```questionnaire `), a complete bare-JSON object in each.
   - **Straight ASCII quotes (`"`) only** — never `“ ” ‚ '` — for every JSON key and string; smart quotes break `JSON.parse` and the block renders as raw code.
2. A short closing message in the user's language (below), **outside** the outer fence.

No commentary, no category breakdown, no schema explanation.

**Validate before you output (paste path):**
- Every inner fence body must `JSON.parse` cleanly — balanced brackets, no trailing commas, straight quotes.
- Exact key names: `version` (`"1"`), `question` (**never `q`**), `options`, `items`, `questions` — per the primitive's shape.
- `recommended`, when set, EXACTLY matches an option label (array for `multi-choice`) — graceful-ignored otherwise, so a mismatch is a wasted nudge.
- Stamp `"skill_version": "3.6"` on the first component fence. Unsure of your own version? **Omit it** rather than guess low (a wrong low value triggers a false "update your skill" banner).

### Closing message (paste path)

After the document, output exactly this (translate to the user's language; N = how many decision points are woven in — count a prioritize as one):

```
N decision points.

1. Copy the block above
2. Open https://passbackai.com
3. Click Paste
```

Hebrew variant:

```
N נקודות החלטה.

1. העתק את הבלוק למעלה
2. פתח את https://passbackai.com
3. לחץ Paste
```

That's the entire post-output text.

## Worked example (ROUTE — a woven palette document)

**Input:** "Mobile guest check-in feature — sending the open decisions to Elad. Haven't decided PIN vs room number; want to know which launch integrations to include; legal must confirm retention (genuinely open); and we have four launch markets queued — US, UK, Germany, Japan — that need a rollout order."

**Output (connected: the same content as an interleaved `blocks[]` array — markdown block, component block, repeated. Paste: the single fenced block below, verbatim):**

> ````
> # Guest check-in — the open decisions
>
> Hi Elad — these are the points we didn't lock on the mobile check-in brief. Everything else below is written as settled; if any of it reads wrong, comment on it directly. None of this is a test — pick what fits, or leave a note, and send it back to me when you're done.
>
> First, the front door. We never settled how a guest proves who they are at check-in — room number alone is the lightest, but it's also the weakest. I'd lean to room number + PIN: one extra field, and it closes the "anyone who sees a door number is in" hole.
>
> ```single-choice
> {"version":"1","skill_version":"3.6","question":"What authentication method should guests use at check-in?","options":["Room number only","Room number + PIN","Last name + booking ref","Magic link"],"recommended":"Room number + PIN","routing":{"from":"Dana","return_prompt":"When done, send your answers back to Dana."}}
> ```
>
> Next, launch integrations — pick everything that should be in v1. I'd start with the two the front desk already lives in.
>
> ```multi-choice
> {"version":"1","question":"Which integrations ship in v1?","options":["PMS sync","Keycard system","Payments","Housekeeping app"],"recommended":["PMS sync","Keycard system"]}
> ```
>
> Retention is genuinely open — the brief flags legal sign-off as pending, so just tell me what legal said (or who to wait on).
>
> ```open-question
> {"version":"1","question":"What retention policy did legal approve for check-in data?","placeholder":"e.g. delete 30 days after checkout — or: waiting on the DPO"}
> ```
>
> Last — four launch markets are queued with no order. Drag them into the sequence you'd ship them in:
>
> ```prioritize
> {"version":"1","items":[{"id":"us","label":"United States"},{"id":"uk","label":"United Kingdom"},{"id":"de","label":"Germany"},{"id":"jp","label":"Japan"}]}
> ```
>
> That's everything — thanks. Send it back to me when you're done.
> ````

```
4 decision points.

1. Copy the block above
2. Open https://passbackai.com
3. Click Paste
```

Note the shape: **each point got the primitive its verb demands** — a pick-one (`single-choice`, with `recommended` and the *why* in its lead-in), a pick-many (`multi-choice`), a genuinely-open point (`open-question` — no invented filler options), and an ordering of 4 concrete peers (`prioritize`). No `questionnaire` appears because no 3+ questions formed one cluster. `routing` rides the FIRST component; the send-back instruction also lives in the opening and closing prose, which is what the reviewer actually sees. The settled prose invites annotation explicitly. *(The leading/trailing `` ```` `` is the real four-backtick outer fence — one code block, one copy button; the `>` marks are only this doc's way of showing the block.)*

## Changelog

### v3.6 (2026-07-15)
- **`youtube` (verb: watch) — a DISPLAY block.** Embed a video the reviewer watches and reacts to; it collects no answer and sits outside the settled-vs-open law. Reach for it whenever the doc references a video — pass the 11-char video id only, **never** a hand-written Markdown image link or an `img.youtube.com` URL.

### v3.5 (2026-07-12)
- **`allocate` (verb: split) joins the palette** — divide a fixed whole (budget / effort / headcount) across categories by weight. Discriminator: order is *which comes first* (`prioritize`); allocate is *how much each gets* (magnitudes summing to a whole) — percentages/dollars → allocate, 1st/2nd/3rd → prioritize.
- **The law is stated two-sided — a floor as well as a ceiling.** Ceiling: a *settled* decision stays prose (don't componentize the decided — density is the form-y feel). Floor: an *open* decision always becomes a component, and when no sharp verb fits it the floor is `open-question`, never a demotion back to prose. The line is settled-vs-open, not fits-a-shape-vs-doesn't; restraint never means leaving the open unasked. This closes the under-use gap (a doc with real open points is never hollow) while keeping the over-use gap shut.

### v3.4 (2026-07-09)
- **The epistemic formula — genius inside, simple outside.** Scan the DECISION SPACE, not the text: reconstruct the goal, enumerate what it requires deciding, diff against what the text settles — so silently-made and never-made decisions surface, not just "TBD" markers. Options must partition the plausible answers; every lead-in states the tradeoff and the falsifier behind `recommended`; components arrive in blocking order with the hinge decision named; settled prose is written as contestable claims. New pre-delivery check: read the doc as the reviewer, cold. On the MCP path, `route_document` now returns advisory `weave.hints` when the woven structure has lapses — fix, re-route, revoke the old id.

### v3.3 (2026-07-07)
- **Proactive offer.** When a working conversation itself accumulates 3+ unresolved open decisions, offer once (non-pushy) to weave them into one PassbackAI page — the skill no longer waits only for an explicit "route this" request. One offer per thread; a decline is respected. Also: strict-spec frontmatter (only the agentskills.io-allowed keys) so the skill installs on third-party hosts (e.g. Dust).

### v3.2 (2026-07-05)
- **Shorter description.** The frontmatter `description` field exceeded the 1024-character limit some install surfaces enforce, blocking upload. Trimmed it (the removed trigger-phrase list is redundant with the `triggers:` array below). No behavior change.

### v3.1 (2026-07-03)
- **Language-agnostic triggers.** The skill fires on its trigger intents in ANY language — the trigger list no longer hard-codes one non-English set; Hebrew stays as an example. Authorship metadata corrected (`created_by` / `owner`).

### v3.0 (2026-07-03)
- **The palette.** One primitive per unclear point — `single-choice` (choose one), `multi-choice` (choose many), `open-question` (write), `prioritize` (order) — woven into prose; `questionnaire` is now the GROUP primitive (3+ tightly-related questions as one unit), no longer the default vessel. Prose is a first-class response channel (the reviewer annotates it), so the REVIEW-vs-ASK upfront question is retired: the document's content decides how much gets woven. Two jobs remain — ROUTE and PULL.

### v2.3 (2026-07-02)
- **Copy once, paste once.** The paste output is the whole document inside **one outer four-backtick fence** — a single code block with one copy button.
- **Ask REVIEW vs ASK when unspecified** *(retired in v3.0 — no mode question exists anymore)*.

### v2.2 (2026-06-30)
- **Lean in with `recommended`.** Reach for `recommended` by default where there's an honest lean — the *why* lives in the prose lead-in.
- **Prefer the default Other.** Omitting `open_field` keeps the edit-into-Other gesture; a custom label is the narrow directed-escape-hatch case.

### v2.1 (2026-06-30)
- **Document, not a form.** Prose sets up each question *before* it; one question per embedded fence; the explanation lives in the lead-in prose, not the `context` field.

### v2.0 (2026-06-27)
- **One skill, three jobs** (REVIEW / ASK / PULL — since consolidated into ROUTE/PULL by v3.0). Supersedes and renames the old `question-extractor` skill. Sharpened prioritize gate.

### v1.6 (2026-06-26)
- MCP delivery (`route_document`): route as typed `blocks[]` (server-validated, returns a reviewer link); paste is the fallback.

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
