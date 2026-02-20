---
name: notion-create-database-row
description: Insert structured rows into a non-task Notion database using schema-aware mapping from natural language fields. Trigger for generic database-entry requests (projects, CRM, bugs, contacts); prefer `notion-create-task` for dedicated task workflows.
---

# Notion Create Database Row

Use this for schema-driven record insertion.

## Workflow

1. Resolve target database from user-provided name/URL.
2. Fetch schema and identify title plus required properties.
3. Map provided values to schema fields with light normalization.
4. Ask for missing required values only.
5. Create row and report key fields.

## Mapping Rules

- Match property names case-insensitively.
- Prefer exact field names when conflicts exist.
- Explain inferred fields in the final confirmation.
