---
name: notion-database-query
description: Query a known Notion database/view with explicit filters and sorts, then return compact summaries. Trigger when the request is retrieval/analysis of existing rows; prefer `notion-search`/`notion-find` for locating pages and databases.
---

# Notion Database Query

Use this for filtered retrieval from known databases.

## Workflow

1. Resolve target database/view from name or URL.
2. Translate user intent into filter and sort conditions.
3. Run query with practical limits unless user asks for full export.
4. Return table-like concise output and key insights.

## Output Rules

- Default to 20-50 rows.
- Show only the most useful columns.
- If no matches, suggest 1-2 alternate filters.
- Do not return raw JSON unless explicitly requested.
