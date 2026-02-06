---
name: epos-opensource
description: Local Docker Compose development workflows for the EPOS Open Source CLI. Use when working with `epos-opensource docker` commands (deploy/update/populate/clean/delete/export/list), preparing custom `.env` or `docker-compose.yaml` files, customizing ports/host, or troubleshooting local Docker environments.
---

# EPOS Open Source Docker Dev

## Overview

Guide local Docker Compose workflows for the EPOS Open Source CLI and manage environment files, ports, and common troubleshooting.

## Quick Start

- Deploy a new environment: `epos-opensource docker deploy myenv`
- Populate with TTL data: `epos-opensource docker populate myenv /path/to/ttl`
- Update configuration: `epos-opensource docker update myenv`
- Export default files for editing: `epos-opensource docker export /path/to/dir`

## Workflow

1. Choose an environment name and deploy.
2. Populate with TTL data if needed.
3. Update or reset the environment when changing `.env` or `docker-compose.yaml`.
4. Clean or delete environments when done.

## Common Tasks

- Create with custom files: `epos-opensource docker deploy myenv --env-file /path/to/.env --compose-file /path/to/docker-compose.yaml`
- Use a custom output directory: `epos-opensource docker deploy myenv --path /path/to/envs`
- Update with new images: `epos-opensource docker update myenv --update-images`
- Force reset the stack (clean volumes): `epos-opensource docker update myenv --force`
- Reset to embedded defaults: `epos-opensource docker update myenv --reset`
- List environments: `epos-opensource docker list`
- Clean data: `epos-opensource docker clean myenv`
- Delete environment: `epos-opensource docker delete myenv`

## Notes and Guardrails

- If using a custom `.env`, set `DATAPORTAL_PORT`, `GATEWAY_PORT`, and `BACKOFFICE_PORT` explicitly.
- Default ports auto-adjust if in use only when using the embedded `.env`.
- The `--host` flag changes the URLs reported after deploy/update; it does not change Docker bindings.
- `update --force` performs `docker compose down -v`, recreates the stack, and reinitializes ontologies.
- Use `docker export` to get the embedded defaults before customizing; then pass the files back via `--env-file` and `--compose-file`.
- In non-interactive contexts, use `docker delete --force` to bypass the confirmation prompt.

## Troubleshooting

- Docker not running: start Docker Desktop or your engine first.
- Environment not found: the CLI stores environments per-user in a local DB; use the same user.
- Port collisions with custom `.env`: change ports in `.env` and redeploy/update.
- Failed deploy/update: use `docker list`, `docker delete`, and redeploy.

## References

- For default `.env` keys, image tags, ports, and compose services, read `references/docker-compose-env.md`.
