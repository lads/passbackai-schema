---
name: passback
description: Route any document into PassbackAI to collect structured feedback, and pull the answers back. ONE skill, two jobs — ROUTE (author one woven document that gives EVERY unclear point the interaction primitive that fits, while settled thinking stays prose the reviewer annotates; then deliver it) and PULL (read back what reviewers answered on a doc you sent). The MCP connection is OPTIONAL — missing, unauthorized or failing, ALWAYS deliver the same document, as a one-click link if you can run code, else ONE four-backtick code block in the chat reply — never a file, never an artifact, never a hand-rolled substitute. Use whenever someone wants feedback on a draft, wants messy input turned into precise decision requests, wants to know what came back on a routed doc, or says "passbackai" / "/passback" — OR proactively, whenever 3+ decision-shaped questions await the user — raised by the conversation, or authored by YOUR OWN output (plan, summary, artifact, doc). In ANY language.
license: Proprietary. See https://passbackai.com
metadata:
  owner: Elad Diamant
  author: elad-diamant
  version: "3.14"
  created: "05-05-2026"
  updated: "2026-09-25"
  triggers: "route this for review; get feedback on this draft; extract open questions; what's still unclear; turn this into a questionnaire; create a passback doc; what came back on the doc I sent; did anyone answer; passbackai; a plan/summary/artifact you just wrote that asks the user 3+ open decisions; the same intents in any language (e.g. Hebrew תוציא שאלות פתוחות)"
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

## The three standard asks — `/passback` with nothing typed after it

The skill is the source of these asks: the user never has to remember, write or paste one. Invoked bare — `/passback` with no document and nothing in the thread to work on — **do not ask an open "what would you like?"**. Show these three, numbered, one line each, and take a number:

1. **Summarize, then hand me the decisions** — the end of a stretch of work: what happened, and what is now waiting on them.
2. **Pull the open questions out of this** — the start: a thread, a brief, a pile of notes, before anyone builds on a guess.
3. **Explain it step by step** — understanding a process, deciding nothing.

On a pick, run it against whatever the conversation holds:

1. Open with a short, plain summary of what you did and what it changes for them. Then the decisions genuinely waiting on them — context in a sentence or two, what each option would mean in practice, and your own recommendation with the reason behind it and what would change your mind. Assume a smart reader with no technical background: simple language about a real tradeoff, never a simplified tradeoff.
2. Pull out the open questions, including the ones nobody wrote down — decisions someone assumed silently, places where two stated wants pull against each other, anything missing you would otherwise guess at. Skip style preferences and anything you can settle yourself.
3. A detailed step-by-step guide in the simplest, least technical language possible, each step saying what happens, who or what does it, and what they would see or do. **Ask nothing**; where something is unclear, make the sensible call and mark it in one line as an assumption they can comment on.

**With material in the thread but no stated ask** — they typed `/passback` at the end of a working session — don't stop to ask: run **1**, it fits that moment, and close with one line naming 2 and 3. All three obey the weave law below either way (settled thinking stays prose, open points get their primitive, nothing is manufactured to fill the page). Someone who ignores the menu and describes their own ask gets an ordinary ROUTE — the menu is a shortcut, never a gate.

## When to offer it — the proactive trigger

You don't only fire on an explicit request. The trigger is **not where the open points came from — it is that they are about to be asked.** When **3+ decision-shaped questions await the user**, **offer, once, to route them**: don't drip the questions one-by-one in chat, and don't wait to be asked. Say it in a single line — that you can weave the open points into one PassbackAI page they (or a colleague) can answer in one pass — e.g. *"There are 5 open points here — want me to turn them into one PassbackAI doc you can answer in one pass?"*

**Both provenances count, and the second one is the one that gets missed.** The conversation may have surfaced them — decisions deferred ("we'll decide later"), competing options nobody picked, missing inputs you keep having to assume. But just as often **YOU authored them**: a plan, a summary, a design doc, an artifact, a PR comment you just wrote that ends in a list of things only the user can decide. Questions you wrote yourself are still questions the user has to answer, so they fire the trigger exactly the same. Check your own outgoing deliverable, not just the transcript.

**The guard — what counts as a decision-shaped question.** It must be (a) still open, (b) addressed to the user, and (c) something only they can settle. A question you can answer yourself by reading the code, the repo, or the thread is not a decision request — go answer it. Rhetorical questions, questions you immediately answer in the next sentence, and 1–2 quick factual gaps don't count either.

**One offer, not a nag.** If they decline or ignore it, drop it for the rest of the thread. For 1–2 quick factual gaps, just ask inline — this trigger is for when the open points are **several and decision-shaped**. On "yes" → run ROUTE below.

**A visual deliverable does not discharge the trigger — the two are complementary.** Building a rendered surface for the same content (an HTML artifact, a slide deck, a report, a diagram) feels like the deliverable is done, and that is exactly how the routing gets skipped. They do different jobs: **the artifact is what the user LOOKS AT; the passback document is how they ANSWER.** A beautiful page whose open questions can only be replied to in chat prose collects no structured answers, so `list_responses` has nothing to pull. So: if the artifact you just built contains the open decisions, that is the **trigger**, not the exemption — ship the artifact AND offer the routed doc. (Don't duplicate: the artifact carries the exposition, the routed doc carries the questions, and it links to the artifact.) **This never runs backwards — the passback document itself is never delivered AS an artifact, a canvas or a file.** It is a `/r/<id>` link when connected and a code block in the chat reply when not; see step 0.

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
| `mermaid` | **watch** | see a STRUCTURE at a glance — a diagram the decision depends on (`source`-only, rendered client-side). A **comprehension** aid, not a decision: display-only, no answer; the reviewer comments per node / edge / whole diagram. See "Diagrams" below for when to reach for one. |

The ladder, applied per point: *pick one of known options → `single-choice` · pick several → `multi-choice` · order a shortlist of ≥3 peers → `prioritize` · split a fixed whole by weight → `allocate` · genuinely open, nothing to enumerate → `open-question` · 3+ questions that truly form ONE decision cluster → `questionnaire` · thinking already settled → prose.* A two-way "ranking" is a `single-choice` (picking which goes first). **`allocate` vs `prioritize`:** order is *which comes first*; allocate is *how much each gets* (magnitudes summing to a whole). If the reviewer would answer in **percentages or dollars**, it's `allocate`; if in **1st/2nd/3rd**, it's `prioritize`. Don't manufacture filler options to force a choice where the point is open — that's an `open-question`. Don't split a real cluster (e.g. four fields of one config) into four lonely blocks — that's a `questionnaire`.

**Response vs display — two kinds of block.** The ladder above is for **response** primitives: each maps one *open decision* to a widget the reviewer answers. Some blocks are **display** — they carry no decision and collect no answer; they embed media the reviewer reacts to and annotates like any prose (today **`youtube`** and **`mermaid`**, both verb **watch**). Reach for **`youtube`** whenever the document *references* a video: embed it as a `youtube` block — `{ "type": "youtube", "version": "1", "id": "<11-char id>" }` — and **never** hand-write a Markdown image link or an `img.youtube.com` URL for a video (that thumbnail host is CSP-blocked, so it renders as a broken image, and a linked image is a click-out, not an inline player). Display blocks sit **outside** the settled-vs-open law below: a referenced video (or diagram) is embedded whether or not anything about it is undecided.

**The law is two-sided — a floor AND a ceiling, and both matter.**

- **Ceiling (against a form-y feel):** a decision the text already *settles* stays prose. Never wrap a made decision in a widget; density is what makes a doc feel like a form.
- **Floor (against a hollow doc):** a decision the text leaves *open* always becomes a component — and when no sharp verb fits it, the floor is `open-question`, **never a demotion back to prose**. An open point dropped to prose is a missed question, not restraint. So a doc with real open points is *never* component-less; only a doc that genuinely settles everything weaves none.

The line is **settled vs open**, not "fits a shape vs doesn't." Restraint means not componentizing the *settled* — it never means leaving the *open* unasked.

## Diagrams — reach for `mermaid` when structure gets dense

A `mermaid` diagram is a **comprehension** aid, a *different axis* from the weave law above. The verbs there map one OPEN decision to one widget; a diagram maps a **STRUCTURE the reader must hold in their head** to a picture. It asks nothing — it makes the shape graspable so the reviewer reacts to the *right* thing. Reach for one and it lands as "genius, now it's clear."

- **The reach trigger.** Emit a ` ```mermaid ` block whenever you're describing a **process, flow, mechanism, sequence, state machine, decision tree, architecture, dependency order, or timeline** — anything the reader would otherwise have to reassemble from prose one clause at a time. Place it **at the moment the prose turns structurally dense**, right after the sentence it clarifies — never in an appendix.
- **The power of TWO — compare with a PAIR.** When the doc weighs a change or options, **two diagrams side by side** (`before → after`, `current vs proposed`, `A vs B`) is often the highest-value move — the reader sees exactly what moves. **Earn it, though: each side must be structurally non-trivial** (a real flow / architecture / state machine per side). A plain two-option choice ("lead with A or B", "publish Tue/Thu vs Mon/Wed") stays a `single-choice` + a sentence, **not** two boxes-and-arrows; diagram the comparison only when the *shapes* differ in a way prose can't hold, like current-vs-proposed architecture. The *decision* still rides a component (a `single-choice` after the pair); the diagrams only make the choice legible.
- **Restraint — don't diagram the trivial.** A diagram earns its place only when the structure is genuinely hard to hold in prose. Three linear steps ("paste → review → copy back") stay prose; a fan-out, a branch, a cycle, or a multi-actor sequence is what it's *for*. Diagrams sprinkled on obvious things read as noise, the same way stacked widgets read as a form.
- **The payoff.** A rendered flowchart is commentable **per node, per edge, and as a whole**, so a diagram turns a vague "this mechanism is unclear" into a comment pinned to the one box or arrow that's wrong. That's the moment: *"genius — and now I can point at the unclear part."*
- **Authoring is `source`-only** — `{ "version": "1", "source": "graph TD; A-->B;", "title": "…" }`. You never name or mark nodes for commenting; the app derives node/edge ids from your `source`.

**Worked example — a mechanism that clicks.** Dense prose like *"on a read we check the edge cache; on a miss we hit origin, but if origin is slow we serve the last good copy and refresh in the background, and only with no cached copy at all do we block on origin"* forces the reader to simulate the branching. Drop the diagram in right after it:

```mermaid
{ "version": "1", "source": "graph TD; R[Read request] --> C{In edge cache?}; C -->|hit| S[Serve cached]; C -->|miss| O{Origin healthy?}; O -->|healthy| F[Fetch, cache, serve]; O -->|slow| B[Serve stale + refresh in background]; O -->|no cached copy| W[Block on origin];", "title": "Read path — cache, origin, and the stale fallback" }
```

The reviewer comes back with a comment on **node `B`**: *"the 'slow' branch should also cover a 5xx from origin, not just latency"* — pinned to the exact box, not a vague note.

**Worked example — the comparison PAIR.** The change is "move JWT validation from every service to the gateway." One diagram of *today*, one of *proposed*, side by side — then a `single-choice` for the actual call:

```mermaid
{ "version": "1", "source": "graph LR; U[User] --> G[Gateway]; G --> S1[Service A]; G --> S2[Service B]; S1 --> A[Auth service]; S2 --> A;", "title": "Current — every service re-validates" }
```

```mermaid
{ "version": "1", "source": "graph LR; U[User] --> G[Gateway]; G -->|signed header| S1[Service A]; G -->|signed header| S2[Service B]; G --> A[Auth service];", "title": "Proposed — the gateway validates once" }
```

```single-choice
{ "version": "1", "question": "Move JWT validation to the gateway in v1?", "options": ["Yes — gateway validates, services trust the signed header", "No — keep per-service validation for now"], "recommended": "Yes — gateway validates, services trust the signed header" }
```

The reviewer sees exactly what moves and can comment on the **`signed header` edge** in the *Proposed* diagram (*"what signs it — short-lived key? rotation?"*), then expresses the decision in the choice below the pair.

## Step 0 — know your delivery path BEFORE you author

There are exactly two delivery paths and no third. Establish which one you are on **before** you start weaving, not after the document exists:

| | When | The last step is… |
|---|---|---|
| **Path A — connected** | the PassbackAI MCP tools (`route_document`, `list_responses`) are present and callable | call `route_document` with typed `blocks[]`; the server returns a `/r/<id>` reviewer link |
| **Path B — not connected** | the tool is missing, unauthorized, erroring, never installed, or a route failed | **B1** — you can run a shell command: pack the document into a one-click `#s=` link with the recipe below. **B2** — you can't, or B1 failed: emit it as **one four-backtick code block in your chat reply** |

**Path B is not a degraded mode, an error, or something to apologise for.** It is the local loop the product shipped with — fully featured — and it is the path most installs are on. So: don't report the missing connector as a problem, don't ask the user to connect anything first, don't offer to "try again once it's set up", and above all don't let the absence change WHAT you author. **The document is byte-for-byte the same work on both paths; only the final step differs.** Deciding this first is what stops the output shape from being improvised at the end — which is the one way this skill reliably fails.

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

**3.5 Check — read it as the reviewer.** Before delivering, one pass through the woven document *as the recipient, cold*: for each component — is this the real question or a proxy for it? Can they answer it better than a coin flip from what's on the page alone? Would their answer actually unblock the work? Fix or cut whatever fails; if a needed fact lives only in your head, it belongs in the lead-in. Then run **The shape check** (below) over every component you're about to emit — the reviewer pass catches a weak question, the shape check catches a block that won't render at all.

**4. Deliver.** Two paths (step 0), one document. These three rules hold on **both** — they are the output contract, and none of them bends:

- **Deliver in THIS turn.** You already authored it; ship it. Never "want me to write it up?", never a promise to produce it next turn, never a summary of what the document would contain instead of the document.
- **Never a file. Never an artifact.** The document is delivered as **text inside your reply** and nowhere else: don't write it to disk, don't create it as an artifact / canvas / side document, don't offer it as a download or an attachment, don't put it in a repo. On a coding or agentic surface (Claude Code, Cursor, an IDE agent) the reflex for a multi-screen Markdown document is to `Write` it to a file — **here that reflex IS the bug**, and it is the single most common way this skill fails: a file forces the user to open it, hunt for the text, select it and copy it out, which is precisely the friction the one-code-block contract exists to delete. The user asked for a document to paste, not a document to find. Only an explicit request for a file overrides this.
- **Never hand-roll a substitute.** No invented `/r/` link, no `#s=` link you typed or "encoded" yourself (only the recipe's printed output is a link), no ad-hoc "form" of your own design, no plain-prose list of the questions in the chat body, no bulleted recap standing in for the woven document. Path A and Path B below are the only two outputs this skill has.

**Path A — connected (`route_document`):** call **`route_document`** with a typed **`blocks[]`** array — the woven sequence, in reading order: `{ "type": "markdown", "text": "…" }` for each prose run, and each component as its own typed block (`{ "type": "single-choice", "version": "1", "question": "…", "options": […] }`, etc.). The server **writes the canonical fences for you and returns a reviewer link** (`/r/<id>`) — and **refuses the whole call** when any block would render as raw JSON instead of a widget: you get an error naming the block and the broken field, and **nothing is stored, no link exists**. That is a **server error message, not the approval dead-end below**: fix the named fields and call `route_document` again in the same turn — never retry it unchanged, and never fall back to paste over it. Run **The shape check** below before you call, and the retry costs you nothing at all. Stamp `"skill_version": "3.14"` on the first component block (and it's harmless on all). If the result carries **`weave.hints`**, treat them as review notes on your weaving: fix what they name, call `route_document` again with the corrected blocks, share the NEW link, and `revoke_document` the old id. (Hints are about the WEAVE — framing prose, a wasted `recommended` label, too many components. A shape that can't render never reaches a hint: it was rejected before anything was stored.)

**Path B — not connected.** Missing, unauthorized, erroring, never-installed and route-failed are ONE case; none of them block delivery. The same document ships one of two ways — **B1 first whenever you can run a shell command**, B2 otherwise.

**B1 — the one-click link (you can run a shell command: Claude Code, claude.ai with code execution, Cowork, any agent with a terminal).** A PassbackAI share link carries the WHOLE document inside the URL fragment (`#s=` — the part after `#`, which the browser never sends to a server), so the user clicks once instead of copy → open → paste. The token is gzip + base64url, which no model can compute by hand, so **run this exact recipe** — paste the woven document between the two `PASSBACK_DOC` lines, change nothing else:

<!-- passback-link-recipe:start -->
```bash
python3 -c '
import sys, json, gzip, base64, re
d = sys.stdin.read().strip()
m = re.fullmatch(r"`{4,}[^\n]*\n(.*)\n`{4,}", d, re.S)
if m: d = m.group(1).strip()
bad = []
for t, b in re.findall(r"^```(questionnaire|single-choice|multi-choice|open-question|prioritize|allocate|youtube)[ \t]*\n(.*?)\n```[ \t]*$", d, re.S | re.M):
    try: json.loads(b)
    except ValueError as e: bad.append(t + " - " + str(e))
if bad: sys.exit("FIX and re-run -- " + "; ".join(bad))
k = base64.urlsafe_b64encode(gzip.compress(json.dumps({"markdown": d}, ensure_ascii=False).encode(), 9, mtime=0)).decode().rstrip("=")
if len(k) > 30000: sys.exit("TOO LONG for a link -- deliver the code block")
print("https://passbackai.com/review#s=" + k)
' <<'PASSBACK_DOC'
<the woven document — exactly what B2 would put inside the four-backtick fence>
PASSBACK_DOC
```
<!-- passback-link-recipe:end -->

- **It prints one URL → that URL is your link.** Copy it **verbatim, character for character** from the command output into your reply — never retype, shorten, or "tidy" it (one wrong character and the link opens nothing).
- **`FIX and re-run`** → a component's JSON doesn't parse. Fix the named block and run again — this is the shape check executing for you.
- **`TOO LONG`**, no `python3`, the sandbox refuses, any other error → go to **B2** with the same document. No apology, no explanation — B2 is an equal path.

**The B1 reply is exactly this, in the user's language, and nothing else** (N = decision points, a `prioritize` counts as one):

```
[Open in PassbackAI →](<the printed URL>)

N decision points — the whole document is packed into the link. Answer, click **Share back**, and paste the link it copies here.
```

Hebrew variant:

```
[פתח ב-PassbackAI ←](<the printed URL>)

N נקודות החלטה — כל המסמך ארוז בתוך הלינק. ענה, לחץ **Share back**, והדבק כאן את הלינק שהועתק.
```

**B2 — the paste document.** The document is fully local and never leaves the browser. Output **exactly two things, in this order, and nothing else:**

**(1) The whole woven document inside ONE outer FOUR-backtick fence** — the copy-once/paste-once contract: one code block, one copy button, one paste into PassbackAI.

- **FOUR backticks** (` ```` `) on the outer fence, **no info-string**. Four, because the document contains three-backtick component fences: a three-backtick outer fence is closed by the first inner component fence, and the rest of the document spills into the chat as loose prose — that is exactly the "it didn't keep the format" failure. The copy button strips the outer markers, so the clipboard receives the exact document, inner fences intact.
- Inside it: the interleaved structure — prose paragraphs with each component in its **own inner fence** (` ```single-choice `, ` ```open-question `, ` ```prioritize `, ` ```allocate `, ` ```multi-choice `, ` ```questionnaire `), a complete bare-JSON object in each.
- **Straight ASCII quotes (`"`) only** — never `“ ” ‚ ’` — for every JSON key and string. (The paste parser is tolerant enough to recover smart quotes and a trailing comma; that safety net is there for a HUMAN paste, not as your licence to be sloppy. What it can never recover is a wrong SHAPE — run **The shape check** below.)
- Stamp `"skill_version": "3.14"` on the first component fence. Unsure of your own version? **Omit it** rather than guess low (a wrong low value triggers a false "update your skill" banner).

The literal shape (the outer four-backtick fence is the real output; this example is wrapped in five so it can show it):

`````markdown
````
# <title>

<1–3 sentences of framing — who it's from, what it's for, that none of it is a test>

<lead-in prose: context, tradeoff, falsifier>

```single-choice
{"version":"1","skill_version":"3.14","question":"…","options":["…","…"],"recommended":"…"}
```

<lead-in prose for the next point>

```open-question
{"version":"1","question":"…"}
```

<closing prose — the send-back line>
````
`````

**(2) The closing message**, in the user's language, **outside** the outer fence. N = how many decision points are woven in (count a `prioritize` as one):

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

**That is the entire reply.** No preamble, no commentary, no category breakdown, no schema explanation, no "here's what I built" — and nothing at all after those three lines.

**A stalled or denied tool APPROVAL is a fail — one attempt, then paste.** The first `route_document` call on a Claude surface may ask the user to approve the tool, and some surfaces (notably the mobile app) offer only a one-shot approval that does **not** re-arm the interrupted call — the turn stalls, or the approved retry never fires. Never retry in a loop. After ONE stalled/denied attempt, deliver via the paste fallback above so the user leaves with the document — and tell them the one-time fix: open the same conversation or connector from a surface that shows the approval dialog reliably (claude.ai on desktop), route one document there, and pick the lasting "always allow" option where offered; routing then works from mobile without prompts.

**Long documents — communicate, don't shrink.** Routing re-generates the whole document as the tool call's arguments, so a long document means a long visible wait *before* the link appears. Never trim capability or drop open points to dodge that wait. Do two things instead:

- **Say so, up front.** When the woven document will be long (roughly 8+ components, or several screens of prose), tell the user in one line *before* calling `route_document` — e.g. *"This one is long — routing will take a minute or two."* An expected wait is patience; an unexplained one is a bug report.
- **Offer a staged split ONLY when it adds value.** The one split worth offering is a *dependency* split: when answers to the hinge decisions would genuinely sharpen the later prose and questions, offer routing round 1 (the hinges) now, then weaving round 2 from their answers (pulled via `list_responses`). Name the value when you offer — *"your round-1 answers will sharpen round 2"* — and skip the offer entirely when the rounds would be independent: a split with no dependency is just two links and twice the friction.

> **The component fields are identical on both paths** — only delivery differs. The **full, code-verified component schema (every field, all primitives) lives at <https://passbackai.com/ask>** (raw: `/ask.md`, JSON Schema: `/schema.json`). Read it there rather than expecting this file to restate it.

> **Privacy — keep it accurate.** The local **paste** loop never transmits the document. **Routing over MCP is an opt-in, server-side egress** — `route_document` stores the routed doc on the PassbackAI backend (server-managed keys, **not** zero-knowledge). Never describe a routed document as end-to-end-encrypted.

### Component field notes (the renderer's contract, compressed)

- `version` is always the string `"1"` (the schema version); `skill_version` is this skill's `"3.14"` — different fields.
- **`single-choice` / `multi-choice`:** `question` (never the key `q`), `options` (2–4 short, genuinely distinct labels; 2–6 for multi), optional `recommended` (a label **string** for single, an **ARRAY** of labels for multi — the wrong one of the two does not degrade, it kills the block; see The shape check — set it whenever you have an honest lean, which is most of the time; the badge marks *which*, your lead-in prose says *why*; must exactly match option labels). Don't put "Other" in `options` — the renderer adds a localized Other row with an edit-into-Other gesture; a custom `open_field.label` ("I need to check with:") replaces it only when the escape-hatch genuinely needs a directed phrase.
- **`open-question`:** `question` + optional `placeholder`. The answer field IS the answer — no options.
- **`prioritize`:** `items` (unique `id` + `label` each); the array order is your suggested starting order; optional `title`/`instruction`.
- **`allocate`:** `items` (unique `id` + `label` + a numeric `weight` each — your proposed split); optional `total` (default 100), `unit` (`"%"` default, `"$"`, `"pts"`), `title`/`instruction`. The reviewer drags weighted bars that always sum to `total`.
- **`questionnaire`:** the group envelope — `questions[]` where each question carries `id`, `question`, `options` (may be `[]` + `open_field` for free-text), optional `multi`/`recommended`/`section`.
- **`youtube` (display — no answer):** `id` is the **11-char video id ONLY** — never a full URL or query string; pass the bare id even for a Shorts or `youtu.be` link (the site builds the privacy-enhanced embed). Optional `title`. It renders a click-to-load player; the reviewer comments on it — and the prose around it — like any passage. No `routing`/answer fields. The whole block:

  ````markdown
  ```youtube
  { "version": "1", "id": "dQw4w9WgXcQ", "title": "Onboarding walkthrough — the 90-second tour" }
  ```
  ````
- **Routing to someone else:** when the doc is sent to another person, include `routing: { "from": "<sender>", "return_prompt": "…send back to <sender>." }` on the first component (the `return_prompt` must contain the literal `from` value) — and say it in the opening/closing prose too, which is what the reviewer actually sees. Self-answer → omit `routing`. If the conversation doesn't say which, ask once: "Are you answering these yourself, or sending them to someone else?"
- **RTL:** auto-detected from content. Hebrew/Arabic input → Hebrew/Arabic chrome. Don't set a language field.

### Targeted clarifications — the only questions you ask first

Never ask a generic upfront question about modes or formats. Ask (once, briefly, together) only when:
- a SPECIFIC point can't be shaped — e.g. "I can't tell if these four items are alternatives to pick from or a sequence to order — which?"
- self-answer vs send-to-someone is genuinely unclear (see routing above).

## The shape check — run it on BOTH paths, before you deliver

A component whose JSON doesn't match **its own fence tag** doesn't degrade politely and doesn't error: the renderer **rejects the whole block** and the reviewer sees a wall of raw JSON in a grey code box. The link works, the tool result looks clean, nothing in your turn goes red — you find out when the reviewer opens the document, which is the worst possible moment. A doc that loses its widgets has lost the only thing it was for.

**Who catches it.** On the `route_document` path the server does: a block off its published shape is **rejected**, with the broken field named — so the failure is loud, cheap and in front of you, never in front of the reviewer. On the **paste** path nobody catches it: there is no server in the loop, so **you are the gate**. Either way, read every component you emit against this list before you deliver — on the routed path it saves you a round-trip, on paste it is the only check that exists.

**FATAL — the block renders as raw code:**

| The mistake | The shape that renders |
|---|---|
| `"recommended": ["A"]` on a **`single-choice`** | a **string** — `"recommended": "A"` |
| `"recommended": "A"` on a **`multi-choice`** | an **array** — `"recommended": ["A"]` |
| `"multi": true` inside a `single-choice` / `multi-choice` | drop it — the **TAG** is the discriminator; `multi` exists only on a `questionnaire` sub-question |
| `"version": 1` (number) | `"version": "1"` (string), always |
| `"options"` with fewer than 2 entries | 2+ entries — or it isn't a choice, it's an `open-question` |
| `"options": [{"label": "A"}]` | plain strings — `"options": ["A", "B"]` |
| `q` / `text` / `prompt` as the prompt key | `question` |
| a `questionnaire` sub-question missing `id` or `options` | both required (`"options": []` + `open_field` for a free-text one) |
| `"multi": "true"` (string) on a questionnaire sub-question | a real boolean `true` |
| an `open-question` carrying `options`, or missing `version` | no `options` at all, and `"version": "1"` |
| duplicate `id`s inside one `prioritize` / `allocate` | every id unique |
| a full URL in `youtube.id` | the bare 11-char video id |
| a field borrowed from another component (`questions` on a choice, `title` on a `single-choice`) | drop it — each tag has its own field set |

**COSMETIC — the block still renders; fix it, but this is not what breaks a document:** a `recommended` whose *label* doesn't exactly match an option (the badge is silently dropped — a wasted nudge); smart quotes or a trailing comma in a fence body (the tolerant parser recovers them); an extra `context`/`placeholder` on a component that ignores it.

**The confusion that produced this list.** `recommended` is the one field whose TYPE depends on the tag, and "a wrong `recommended` is only a wasted nudge" is true **only of a wrong label**. A wrong *type* takes the entire block down — a whole page of decisions can ship as JSON nobody can answer, because one array should have been a string. When you author several choice components in a row, check the `recommended` type on **each** one; that is exactly where the copy-paste momentum carries the wrong shape forward.

## JOB: PULL — read back the answers, then synthesize

The user asks what came back on a doc they already routed. **Do not author a new document.**

- **Connected:** call **`list_responses`** with the document id. It returns each reviewer's `verdict`, leaf-anchored `annotations[]` (their comments on the prose), and leaf-free `componentInputs[]` (their component answers) — a pull, no webhook. Then **synthesize for the user**: the verdict, the structured answers, and the free-form notes in their own language; surface conflicts and anything still unanswered.
- **Not connected:** the answers come back pasted into the chat, one of two ways. **An export bundle** (text) → read it and synthesize the same way. **A `passbackai.com/review#s=…` link** → a reviewer who opened a link (every B1 document) returns it with **Share back**, which packs their answers into a link the same way. You can't read it by eye. Run this exact recipe with the pasted link between the `PASSBACK_LINK` lines. It prints the reviewer's name, each component paired with its answer (`null` = unanswered), and their comments on the prose. Synthesize from that. No shell, or the recipe says `PASSWORD-PROTECTED` → ask them to open the link and use **⋯ → Copy document + comments**, then paste the text here.

<!-- passback-pull-recipe:start -->
```bash
python3 -c '
import sys, json, gzip, base64, re
from collections import Counter
m = re.search(r"#s=([A-Za-z0-9_-]+)", sys.stdin.read())
if not m: sys.exit("NO #s= LINK -- ask for the link they copied")
raw = base64.urlsafe_b64decode(m.group(1) + "=" * (-len(m.group(1)) % 4))
if raw[:2] != b"\x1f\x8b": sys.exit("PASSWORD-PROTECTED -- ask them to use Copy instead")
s = json.loads(gzip.decompress(raw))
P = {"questionnaire": "q", "prioritize": "p", "allocate": "a", "single-choice": "sc", "multi-choice": "mc", "open-question": "oq"}
n, out = Counter(), []
for t, b in re.findall(r"^```(" + "|".join(P) + r")[ \t]*\n(.*?)\n```[ \t]*$", s.get("markdown", ""), re.S | re.M):
    try: c = json.loads(b)
    except ValueError: c = b
    out.append({"component": t, "spec": c, "answer": (s.get("embeddedAnswers") or {}).get(P[t] + "-embed-" + str(n[t]))}); n[t] += 1
notes = [{k: v for k, v in c.items() if k in ("quoted", "label", "text", "kind", "replacement", "author", "replies")} for c in s.get("comments") or []]
print(json.dumps({"from": s.get("sharedBy"), "answers": out, "comments": notes}, ensure_ascii=False, indent=1))
' <<'PASSBACK_LINK'
<the link the user pasted>
PASSBACK_LINK
```
<!-- passback-pull-recipe:end -->

If you don't know the document id, ask the user which routed document they mean.

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
> {"version":"1","skill_version":"3.14","question":"What authentication method should guests use at check-in?","options":["Room number only","Room number + PIN","Last name + booking ref","Magic link"],"recommended":"Room number + PIN","routing":{"from":"Dana","return_prompt":"When done, send your answers back to Dana."}}
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

*Recent versions only — the full history (v1.0 → today) is published at <https://passbackai.com/skill#whats-new>.*

### v3.14 (2026-09-25)
- **Without the connector, the document arrives as a one-click link, not a block to copy.** A PassbackAI share link already carries a whole document inside its `#s=` fragment, but the token is gzip + base64url, which no model can compute by hand. So Path B now splits: wherever the model can run a shell command, it runs one fixed recipe that packs the document into a `passbackai.com/review#s=` link and hands the user that link, one click instead of copy → open → paste. The code block stays as the equal fallback (no shell, recipe error, or a document too long for a link). The recipe also parses every component's JSON before it packs, so a block that would render as raw JSON is fixed before the user ever sees it. The return trip closes the same way. A reviewer who opened a link answers with **Share back**, which hands back another link. PULL now has a matching recipe that decodes that link into the reviewer's name, each component paired with its answer, and their comments, so the answers come back as structured data. A CI test runs both exact recipes from this file against the app's real encoder and decoder, so neither can drift from the codec or the component registry.

### v3.13 (2026-09-22)
- **The three standard asks live in the skill, so nobody has to remember a prompt.** The asks that produce the best documents — summarize-then-decide, pull-the-open-questions, explain-step-by-step — were folklore: they worked, they were re-typed from memory, and a first-time user had no way to know they existed. A bare `/passback` now shows the three by name and takes a number, and `/passback` at the end of a working session runs the first one instead of stopping to ask. Each ask also carries the clause that makes it work: a recommendation WITH its reason and what would change it (the half that was always missing), "simple language about a real tradeoff, never a simplified tradeoff" in place of the explain-like-I'm-15 framing that reliably produced condescension, and — on the no-questions guide — "state the assumption in one line" so an unclear point stops becoming a silent guess.

### v3.12 (2026-09-17)
- **The no-MCP path is now a specified path, not a fallback improvised at the end.** Delivery without the connector was the skill's least reliable moment: it would sometimes write the document to a FILE (on a coding surface the reflex for a long Markdown document is `Write` — and the skill, having never once said not to, lost to that reflex), sometimes print the questions as loose chat prose with the component fences broken or gone. Three causes, three fixes. (1) The paste format was a *cross-reference* — step 4 said "see the Output contract", 76 lines away, past four unrelated sections — so the format was reconstructed from memory at the exact moment it mattered; it is now stated **literally and in full, inline at the point of delivery**, with a skeleton of the four-backtick wrapper. (2) Nothing anywhere forbade a file, an artifact or a canvas; now an explicit rule does — the document is text in the reply, nowhere else, because a file re-introduces exactly the find-select-copy friction the one-code-block contract exists to delete. (3) The path was discovered *after* authoring, and "PREFERRED" / "fallback" framing cast paste as degradation, which invites improvisation; a new **step 0** settles the path before weaving and states that paste is the full local loop, not a broken connector. Also: the changelog moved to the website, cutting ~2.4KB of pure history out of the file every invocation loads.

### v3.11 (2026-09-10)
- **The shape check, on both paths — the fatal/cosmetic line drawn where it actually falls.** A routed document shipped with every one of its choice blocks rendering as raw JSON: each carried `recommended` as a one-element ARRAY on a `single-choice`, whose `recommended` must be a string. The skill had made that outcome hard to see — it warned loudest about smart quotes (which the tolerant parser recovers) and described a wrong `recommended` as "graceful-ignored… a wasted nudge", which is true of a wrong LABEL and false of a wrong TYPE. It also told the model the `blocks[]` path was validated for it, so nothing needed checking there. Now: one FATAL list (every shape that makes the renderer drop the block to plain code) against one COSMETIC list, run on the MCP path as well as paste — and, because advice a model can skip is not a guarantee, `route_document` itself now **rejects** a block that cannot render (the call errors, nothing is stored, you fix the named field and call again), so a link with a raw-JSON widget can no longer be created at all.

### v3.10 (2026-08-18)
- **The proactive trigger keys on the SHAPE of the ask, not on who raised it.** It used to fire only when "a working conversation has itself surfaced" the open points — provenance — so 3+ decision-shaped questions the skill's own host had *authored* (in a plan, a summary, an artifact, a PR comment) fell outside it and got pasted into chat. Now: 3+ open decisions awaiting the user fire it whichever side wrote them, with a guard (still open, addressed to the user, only they can settle it) so a question you can answer yourself never counts.
- **A visual deliverable is complementary, not a substitute.** The skill was silent on artifacts/decks/reports, and that silence let both jobs collapse into one rendered page. Stated now: the artifact is what you LOOK AT, the routed doc is how you ANSWER — an artifact containing the open questions is the trigger, not the exemption.

*Older entries (v3.9 and below) — see <https://passbackai.com/skill#whats-new>.*
