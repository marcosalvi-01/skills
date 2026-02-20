---
name: notion-create-page
description: Create a new Notion page with minimal intent-matched structure under a chosen parent. Trigger when the main action is page creation; prefer `notion-knowledge-capture`, `notion-meeting-intelligence`, or `notion-research-documentation` for workflow-heavy authored content.
---

# Notion Create Page

Create pages safely and with just enough structure.

## Workflow

1. Parse title, parent, and page intent from the request.
2. Resolve parent via search/fetch if not explicit.
3. Create page content with an intent-matched scaffold.
4. Confirm title, location, and identifier to the user.

## Scaffolds

- Meeting notes: attendees, agenda, notes, decisions, action items.
- Project page: overview, goals, scope, milestones, risks, links.
- Spec/brief: context, requirements, constraints, success criteria.

## Guardrails

- Avoid duplicate creation in the same parent when an exact title already exists.
- Ask one short clarification only if parent ambiguity materially affects location.
