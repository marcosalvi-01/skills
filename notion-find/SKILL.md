---
name: notion-find
description: Run precision title-oriented Notion lookup for pages/databases when likely names are known. Trigger for exact-ish find/open requests; prefer `notion-search` for exploratory discovery by topic or intent.
---

# Notion Find

Use this as a precision search helper.

## Behavior

- Treat user input as title-biased keywords.
- Search both pages and databases.
- Return top 5-10 most likely matches with type and parent context.
- If no clear hit, suggest alternative terms instead of guessing.

## Escalation

- If the user asks exploratory questions, switch to `notion-search`.
- If a result is selected, fetch it before taking write actions.
