---
name: printf-debug
description: Add printf/logging-only debugging statements to existing functions without changing behavior. Use when the user asks to debug a function by adding prints/logs, instrument code paths, follow debugprint.nvim-style output with a DEBUGPRINT tag, or inspect runtime state while preserving original control flow.
---

# Printf Debug

## Overview

Add observability to a target function using only debug prints/logs.
Keep logic intact: do not change signatures, behavior, data flow, or control flow.

## Workflow

1. Locate the exact target function and read enough surrounding code to understand inputs, branches, side effects, and return points.
2. Detect the existing logging strategy before adding anything:
   - Reuse project logger/import style if present.
   - Prefer debug-level logger output (for example `logger.debug(...)`) when available.
   - If no logger exists in that scope, use standard language print (`print`, `printf`, `println`, `console.debug`, etc.).
3. Insert DEBUGPRINT statements at:
   - function entry,
   - key branches/decision points,
   - loop checkpoints when needed,
   - before each return (or throw) point.
4. Include the runtime state needed to reconstruct execution:
   - key parameters,
   - branch condition values,
   - derived intermediate values,
   - return/exit values when available.
5. Keep edits tightly scoped to debug statements only.
6. If a likely bug is discovered, add a short comment at that location explaining why it appears wrong. Do not fix the logic unless explicitly asked.

## Required Constraints

- Do not modify function signature.
- Do not alter behavior, logic, data structures, or control flow.
- Do not remove existing logs.
- Do not refactor unrelated code.
- Keep added output concise but sufficient for step-by-step tracing.

## DEBUGPRINT Style

- Every inserted line must include the `DEBUGPRINT` tag.
- Use language-idiomatic formatting and placeholders.
- Prefer structured state where the language/logger supports it.
- Message pattern: `DEBUGPRINT <function> <event> <state>`.
- Use stable event labels such as `entry`, `branch:<name>`, `loop`, `pre-return`, `error-path`.

## Language Conventions

- Python: prefer `logger.debug("DEBUGPRINT ... %s", value)`; fallback `print(f"DEBUGPRINT ... {value=}")`.
- JavaScript/TypeScript: prefer project logger; fallback `console.debug("DEBUGPRINT ...", { value })`.
- Java/Kotlin: prefer `logger.debug("DEBUGPRINT ... {}", value)`.
- Go: prefer existing logger; fallback `fmt.Printf("DEBUGPRINT ... value=%v\n", value)`.
- C/C++: use existing logging macros or `printf("DEBUGPRINT ... value=%d\\n", value)` as appropriate.
- Rust: prefer `log::debug!`; fallback `println!("DEBUGPRINT ... value={:?}", value)`.

## Output Expectations

- Add only the debug instrumentation required to understand runtime behavior.
- Preserve compilation/runtime validity.
- Keep messages readable for humans scanning logs in sequence.
