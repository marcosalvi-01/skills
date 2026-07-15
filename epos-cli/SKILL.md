---
name: epos-cli
description: Local EPOS Docker Compose environment management with `epos-opensource docker`. Use when deploying, updating, listing, cleaning, deleting, rendering, or populating local Docker environments, especially when testing a custom locally built image by editing YAML image refs and updating an existing env.
---

# Epos Cli

## Purpose

Use `epos-opensource docker` to manage local EPOS Docker Compose environments for build-test loops.

Use this skill when the task is to:
- deploy new local envs
- update existing envs after rebuilding an image
- point envs at custom local tags or registries via YAML `images.*`
- inspect applied config and installed envs
- populate, clean, or delete local envs

Ignore k8s for this skill.

## Workflow

1. Start from current env or a fresh config.
2. For any new env config, start from closest example YAML and edit in place.
3. Keep output minimal: change only fields that matter for task.
4. Do not regenerate full YAML unless user explicitly asks for full file.
5. For custom image testing, edit YAML config `images.*` to the local image refs you want to run.
6. Deploy new env with `epos-opensource docker deploy <env> --config <file>`.
7. Update existing env after rebuild with `epos-opensource docker update <env> --config <file>`.
8. Use `-u` to pull remote images, `--force` to recreate containers, and `--reset` to swap back to embedded defaults.
9. Verify by running `epos-opensource docker list` to get generated GUI, API, and backoffice URLs, then check `epos-opensource docker get <env>` if you need applied config.
10. Use `populate` for data setup, `clean` for data reset, and `delete` for teardown.

## Custom Environment Backoffice Access

For custom environments, always place all users and products in the `all` group for simplicity.

- The first user to log in becomes an admin.
- Admins can join groups without requesting access.
- Every other user must be accepted before joining a group.

## Database Debugging

To let the agent query the metadata database directly from the host, set `components.metadata_database.published_port` to an available host port, such as `5432`. Then use `psql` with connection values from the YAML config or `epos-opensource docker get <env> --output <file>`:

```bash
PGPASSWORD=<components.metadata_database.password> psql \
  -h localhost \
  -p <components.metadata_database.published_port> \
  -U <components.metadata_database.user> \
  -d <components.metadata_database.db_name>
```

Use direct database queries for debugging when API or UI inspection is insufficient. Keep the published port internal-only (`0`) when direct host access is not needed, and check for port conflicts before using `5432`.

## Command Use

Before using any docker command, read live help from the installed CLI:

- `epos-opensource docker --help`
- `epos-opensource docker <subcommand> --help`

Prefer `--help` over static docs for flags, args, and current behavior.

## Custom Image Loop

1. Get current config with `epos-opensource docker get <env> --output <file>` if needed.
2. Edit YAML `images.*` to the local image refs that should run.
3. Rebuild image locally.
4. Run `epos-opensource docker update <env> --config <file>`.
5. Use `--force` only when you need fresh containers or wiped volumes.
6. Use `epos-opensource docker list` and the printed API URL to test the new image.

For a full commented base config, start from [default-with-comments.yaml](references/default-with-comments.yaml).
For backoffice envs, start from [backoffice-aai-minimal.yaml](references/backoffice-aai-minimal.yaml).
Copy example YAML and modify in place instead of regenerating configs from scratch.
If you need to show config in final answer, show only changed lines or a diff-style snippet.
If test data is needed, use `epos-opensource docker populate <env> --example`.

## AAI Token

When `components.aai_service.enabled` is true, get auth admin token from local AAI service with config-derived values:

```bash
curl -s -X POST "http://localhost:<components.aai_service.port>/oauth/token" \
  -u local-dev-client:dev-secret \
  -d grant_type=password \
  -d username=<components.aai_service.email> \
  -d password=<components.aai_service.password>
```

Use current YAML or `epos-opensource docker get <env> --output <file>` to read exact port and admin values before running it.

## API Flow

- Use gateway URL for API calls.
- Use backoffice URL only for UI/browser access.
- Use `epos-opensource docker list` for getting the generated URLs.
- Open gateway Swagger/OpenAPI at `/ui` to discover proxied routes and request shapes.
- Treat spec as source of truth for path, body, query params, and auth.
- Services are exposed 1:1 behind gateway, with auth layer on top.
- Avoid container introspection unless user explicitly asks for it.
- Do not guess proxy paths; if route is unknown, ask or inspect live docs.

## Backoffice Smoke Test

Use this host-side flow when validating backoffice with OSS AAI:

1. Run `epos-opensource docker list`.
2. Get AAI token with config-derived credentials.
3. Read backoffice route from gateway Swagger at `/ui`.
4. Call gateway route from host with token.
5. Verify create/read response shape against spec.
