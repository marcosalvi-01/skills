---
name: notion-knowledge-capture
description: Capture conversation output into durable Notion knowledge artifacts (wiki, FAQ, how-to, decision log, learning note, documentation page). Trigger only when the primary intent is documenting knowledge from discussion context; prefer `notion-create-page` for blank page creation and `notion-research-documentation` for multi-source analysis.
---

# Notion Knowledge Capture

Convert unstructured conversation context into durable Notion knowledge artifacts.

## Quick Start

1. Identify what to capture: decision, FAQ, how-to, wiki concept, learning note, or documentation page.
2. Locate the destination page or database with Notion search and fetch tools.
3. Extract facts, rationale, actions, and links from the conversation.
4. Create or update the Notion page with consistent structure and metadata.
5. Add backlinks/relations so the new content is discoverable.

## Workflow

### 1) Classify the content

- Determine audience, purpose, and freshness.
- Decide whether this is a new page or an update to existing documentation.

### 2) Choose destination

- Find candidate destinations in Notion (team wiki, FAQ DB, decision log DB, documentation DB).
- If multiple destinations are equally valid, ask one concise routing question.
- Prefer the team's existing structure over creating new top-level pages.

### 3) Structure the write-up

- Decision: context, decision, alternatives, rationale, consequences.
- How-to: prerequisites, steps, verification, troubleshooting.
- FAQ: concise answer first, deeper explanation second.
- Wiki/learning: definition, examples, caveats, related concepts.

### 4) Create or update in Notion

- Use create-page when writing a new page.
- Use update-page when content already exists and should be revised.
- Preserve source links and add owners/tags/status when a database schema supports them.

### 5) Make it discoverable

- Link the new page from relevant hub pages or related records.
- Add short changelog notes for major updates.
- If follow-up work appears, create or link tasks.

## References

- Use `reference/` templates to match team conventions.
- Use `examples/` as structure examples before drafting.
