---
name: notion-tasks-build
description: Execute implementation work from a specific Notion task URL while synchronizing in-progress/done status updates. Trigger when Notion task tracking is required during coding; prefer `notion-create-task` for task entry and `notion-tasks-plan` for planning-only requests.
---

# Notion Tasks Build

Use this when Notion is the source of truth for execution status.

## Workflow

1. Fetch task details and linked acceptance criteria/spec pages.
2. Update task status to in-progress.
3. Execute implementation work in the target codebase.
4. Post concise progress updates in task fields/comments.
5. Mark done with a final result summary and follow-ups.

## Guardrails

- If requirements are unclear, post one blocking question and pause risky implementation.
- Keep Notion status synchronized with real execution state.
