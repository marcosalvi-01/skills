---
name: brainstorm
description: Brainstorm practical software solutions from ambiguous or early-stage problems. Use when the user asks for ideas, approaches, tradeoffs, architectures, implementation directions, MVP scope, migration paths, or debugging strategy options before choosing what to build.
---

# Software Brainstorm

## Overview

Turn unclear software problems into concrete, comparable options. Keep ideas realistic, call out tradeoffs, and end with a recommended next step.

## Workflow

1. Define the problem in one sentence.
2. List assumptions and constraints.
3. Generate 3-5 viable solution options.
4. Compare options by effort, risk, and impact.
5. Recommend one path and propose a first experiment.

## Inputs To Collect

- Problem statement and desired outcome
- Scale and performance expectations
- Team size, timeline, and delivery pressure
- Existing stack and integration constraints
- Non-functional requirements (security, reliability, compliance, cost)

If details are missing, state assumptions explicitly and continue.

## Option Generation Rules

- Prefer options that can be implemented incrementally.
- Include at least one low-effort option and one scalable option.
- Avoid suggesting tooling changes unless they solve a clear bottleneck.
- Tie each option to concrete implementation ideas (components, services, data flow, rollout).
- Keep language software-specific, not generic business advice.

## Comparison Format

For each option, provide:

- One-line approach summary
- Pros (2-4 bullets)
- Cons (2-4 bullets)
- Estimated effort: S / M / L
- Risk level: Low / Medium / High
- Best fit: when this option is the right choice

## Recommendation Rules

- Recommend exactly one primary option unless the user asks for multiple finalists.
- Explain why it wins given the stated constraints.
- Identify the main failure mode and a mitigation.
- Propose a first step that can be done in 1-3 days.

## Default Output Template

Use this structure in responses:

1. Problem framing (1-2 sentences)
2. Assumptions and constraints
3. Option A / B / C comparison
4. Recommended approach
5. First implementation milestone
6. Open questions (only if blocking)

## Quality Bar

- Favor clarity over breadth.
- Avoid repeating equivalent options with different names.
- Ground suggestions in common software patterns and operational realities.
- Keep momentum: give actionable next moves, not just analysis.

## Scope Guardrails

- Do not write production code unless explicitly requested.
- Do not pretend certainty where major assumptions are unknown.
- Do not optimize prematurely; prefer the simplest path that can be validated quickly.
