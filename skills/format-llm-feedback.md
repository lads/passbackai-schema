---
name: format-llm-feedback
title: Format multi-point feedback as anchored Markdown for an LLM
description: |
  Convert a list of "the LLM should fix this passage" notes into a single
  Markdown export that the originating model can apply to its last answer
  in one round-trip. Each note is paired with the verbatim quote it
  applies to AND the heading chain (or table row/column) the quote sits
  under, so the model lands every fix on its exact target.
version: "1.0.0"
license: MIT
homepage: https://passbackai.com/
documentation: https://passbackai.com/agents.md
tags:
  - llm-feedback
  - markdown
  - annotation
  - prompt-engineering
  - claude
  - chatgpt
  - cursor
  - gemini
  - lovable
---

# Format multi-point feedback as anchored Markdown

## When to use this skill

Use this skill when the user has a long Markdown answer from an LLM (Claude, ChatGPT, Cursor, Gemini, Lovable, v0, Perplexity, etc.) and wants to send back more than two distinct corrections in a single message. The chat box collapses on multi-point feedback because:

1. Writing each correction as prose is hard enough work that the user gives up on points 4 and 5.
2. Even when all points are written, the chat box can't anchor a correction to the exact passage it refers to.

This skill produces the format that solves both problems.

## Inputs

- `original_answer` (string, required): the full Markdown answer from the upstream LLM.
- `comments` (array, required): each item has:
  - `quote` (string, required): the exact passage the comment applies to. Must be a verbatim substring of `original_answer` (or, for a table cell, the cell's text).
  - `note` (string, required): the user's correction, in their own voice.
  - `label` (string, optional): one of `Delete`, `Too long`, `Off tone`, `Made it up`, `Too vague`. Use sparingly — the freeform `note` carries the meat.
  - `location` (string, optional): if the same quote appears more than once in the document, the heading chain or table address that disambiguates it. Examples:
    - `under h2 "Checkout"`
    - `under h2 "Checkout" > h3 "Payment"`
    - `the "Free" row`
    - `in the "Status" column`

## Output

A single Markdown block. Top line: `# My Feedback:`. Each comment is a stanza of three lines: a curly-quoted passage with optional location anchor in parens, the note (with optional `[Label]` prefix in square brackets), separated from the next stanza by a line containing only `---`.

### Output template

```
# My Feedback:
---

"{quote}" ({location})
[{label}] {note}

---

"{quote}" ({location})
{note}
```

### Worked example

Given:

```json
{
  "comments": [
    {
      "quote": "You can change this later in account settings.",
      "location": "under h2 \"Onboarding\"",
      "label": "Too vague",
      "note": "which settings? Name them — billing, notification preferences, or connected accounts."
    },
    {
      "quote": "Free",
      "location": "the \"Free\" row",
      "label": "Made it up",
      "note": "we don't have a Free tier — change to Trial."
    },
    {
      "quote": "Status",
      "location": "in the \"Status\" column",
      "label": "Off tone",
      "note": "this column reads like a release tracker. Rename to \"Availability\"."
    }
  ]
}
```

Produce:

```markdown
# My Feedback:
---

"You can change this later in account settings." (under h2 "Onboarding")
[Too vague] which settings? Name them — billing, notification preferences, or connected accounts.

---

"Free" (the "Free" row)
[Made it up] we don't have a Free tier — change to Trial.

---

"Status" (in the "Status" column)
[Off tone] this column reads like a release tracker. Rename to "Availability".
```

## Why this format

- The curly quotation marks (`"…"`) make the boundary between user note and quoted passage unambiguous to the model.
- The location anchor in parens tells the model which instance of a repeated phrase to fix.
- The `---` separators let the model parse the list as paired data instead of prose.
- Ordering by document position lets the user diff "what landed vs. what was skipped" by reading the next answer top-to-bottom.

## Failure modes to flag back to the user

- A `quote` that is not a verbatim substring of `original_answer` will silently fail to land — flag and ask the user to copy-paste the exact span.
- A `location` that names a heading not present in the document will confuse the model — better to omit `location` than to invent one.
- More than 25 comments in one batch starts to overflow context windows on the smaller models — suggest splitting the review into two or three batches.
