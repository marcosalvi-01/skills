---
name: notion-create-task
description: Create one or more tasks in a Notion task database with schema-aware property mapping. Trigger for direct task-entry requests; prefer `notion-tasks-plan` when task generation depends on spec decomposition.
---

# Notion Create Task

Use this for one-off or batch task creation.

## Workflow

1. Extract title (required) and optional owner, due date, status, priority, project.
2. Resolve the right task database by name and schema.
3. Map user fields to existing properties.
4. Create the task and return key fields.

## Defaults

- Status defaults to a backlog/to-do state when available.
- Due date and owner stay unset unless explicitly provided.
- Preserve user wording in task title and objective.

## Guardrails

- If multiple task DB candidates exist, ask one disambiguation question.
- If required properties are missing, request only missing values.
