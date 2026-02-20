---
name: notion-search
description: Run broad Notion workspace discovery for topic-level or natural-language queries and return ranked, concise results. Trigger for exploratory lookup across unknown locations; prefer `notion-find` when the user already knows likely page/database names.
---

# Notion Search

Use this for broad workspace lookup before narrowing into specific pages.

## Workflow

1. Parse the user intent into 1-3 focused search queries.
2. Run Notion search, then optionally fetch top hits for context.
3. Return a short, scannable result list (title, type, why relevant).
4. If results are weak, suggest one improved query and retry.

## Output Rules

- Prioritize relevance over volume.
- Avoid raw JSON dumps.
- Include page/database identifiers or URLs when available.
- Call out uncertainty when matches are ambiguous.
