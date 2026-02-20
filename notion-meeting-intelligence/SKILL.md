---
name: notion-meeting-intelligence
description: Prepare meeting-specific Notion deliverables (agenda, pre-read, decision framing, follow-up notes) from existing workspace context. Trigger when the request is explicitly about an upcoming/recent meeting; prefer `notion-research-documentation` for general research and `notion-create-page` for simple page drafting.
---

# Notion Meeting Intelligence

Build concise, decision-ready meeting docs from workspace context.

## Quick Start

1. Confirm objective, attendees, timing, and expected decisions.
2. Pull relevant Notion context (prior notes, plans, risks, tasks).
3. Pick a meeting format (status, planning, decision, retro, 1:1, brainstorming).
4. Draft agenda and pre-read with owners, timeboxes, and clear outcomes.
5. Add follow-ups and update materials as details evolve.

## Workflow

### 1) Gather inputs

- Capture meeting purpose, audience, and output needed.
- Fetch only high-signal context pages, not every related page.
- Surface blockers, dependencies, and unresolved questions early.

### 2) Select template

- Match template to meeting type using `reference/template-selection-guide.md`.
- Keep sections minimal when the meeting is short or tactical.

### 3) Draft materials

- Include objective, agenda items, owners, time allocation, and decisions required.
- Add pre-read links and context bullets for attendees.
- Distinguish facts from assumptions.

### 4) Add targeted research

- Include only research that improves decisions or risk awareness.
- Cite sources for claims that could be questioned.

### 5) Finalize and track follow-up

- Add next steps, owners, and deadlines.
- Link related tasks/pages in Notion.
- Update the page after agenda changes or post-meeting outcomes.

## References

- Use `reference/` templates for each meeting type.
- Use `examples/` for end-to-end examples.
