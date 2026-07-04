---
name: question-extractor
description: DEPRECATED — this skill has been replaced by the consolidated `/passback` skill. It now does everything question-extractor did (turn input into a routable questionnaire) plus routing prose for review and pulling answers back. Install `/passback` and retire this skill; do NOT patch this file in place. Source — https://passbackai.com/skills/passback.md (the route_document MCP tool and a paste fallback are its delivery faces; `blocks` carries the typed components).
version: 2.0
created: 05-05-2026
updated: 2026-07-03
created_by: elad-diamant
owner: Elad Diamant
---

# question-extractor is now `/passback`

This skill has been **renamed and superseded**. The single `/passback` skill replaces it: one skill with three internal jobs — **review** (route existing prose for open-ended feedback), **ask** (turn ambiguity into a questionnaire, plus a prioritize when ordering a shortlist of ≥3 peers, then route it), and **pull** (read the answers back with `list_responses`). All of question-extractor's questionnaire authoring lives in `/passback` unchanged.

## How to install `/passback`

1. Fetch the new skill: <https://passbackai.com/skills/passback.md>.
2. Install it as the `/passback` skill (it carries its current `version` in its own frontmatter).
3. **Retire this `question-extractor` skill** so it no longer triggers — do **not** edit this file in place to "version 2.0". The upgrade is a rename/supersede, not a patch.

The component schema and delivery contract (the `route_document` MCP tool with typed `blocks`, and the paste fallback) are documented at <https://passbackai.com/ask>.
