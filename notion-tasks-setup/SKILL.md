---
name: notion-tasks-setup
description: Set up or validate a Notion task board schema so task automation can run safely. Trigger for first-time setup, schema repair, or board migration; do not use for day-to-day task creation or execution.
---

# Notion Tasks Setup

Use this before task automation if board structure is uncertain.

## Workflow

1. Determine whether user wants a fresh board or to reuse an existing one.
2. Inspect database schema and required properties.
3. Validate minimum task fields (title, status, owner/date/project as applicable).
4. Recommend minimal schema changes when gaps exist.
5. Confirm board is ready for task-create/task-build workflows.

## Minimum Recommended Fields

- Title property.
- Status/select property.
- Optional but useful: assignee, due date, priority, project relation.
