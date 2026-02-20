---
name: notion-tasks-explain-diff
description: Produce deep Notion documentation that explains a completed code change (background, intuition, walkthrough, verification). Trigger after implementation when the main deliverable is explanatory write-up; do not use for planning or code execution itself.
---

# Notion Tasks Explain Diff

Use this for deep, teachable change documentation.

## Document Structure

1. Background: system context needed to understand the change.
2. Intuition: core idea and why this approach.
3. Code Walkthrough: grouped explanation of modified files.
4. Verification: tests run and manual checks.
5. Alternatives: optional, only when meaningfully different.

## Formatting Guidance

- Prefer narrative clarity over exhaustive diff dumps.
- Include file references and key snippets where useful.
- Keep verification actionable and reproducible.
- Publish as a Notion page and return the page URL.
