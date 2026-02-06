# EPOS Docker Compose Reference

## Default Embedded Files

- `.env` embedded at `cmd/docker/dockercore/static/.env`
- `docker-compose.yaml` embedded at `cmd/docker/dockercore/static/docker-compose.yaml`

Use `epos-opensource docker export /path/to/dir` to export the defaults for editing.

## Ports and URLs

- `DATAPORTAL_PORT` default 32000
- `GATEWAY_PORT` default 33000
- `BACKOFFICE_PORT` default 34000
- `API_PATH` default `/api/v1`
- `DEPLOY_PATH` default `/`

When using a custom `.env`, set all three ports explicitly. With embedded defaults, the CLI auto-picks free ports if needed.

## Compose Services (docker-compose.yaml)

- `dataportal` (ports `DATAPORTAL_PORT:80`)
- `backoffice-ui` (ports `BACKOFFICE_PORT:80`)
- `gateway` (ports `GATEWAY_PORT:5000`, healthcheck on `/api/v1/ui`)
- `rabbitmq` (healthcheck `rabbitmq-diagnostics check_running`)
- `resources-service`
- `ingestor-service`
- `external-access-service`
- `converter-service` (volume `converter`)
- `converter-routine` (volume `converter`)
- `backoffice-service`
- `email-sender-service`
- `sharing-service`
- `metadata-database` (volume `psqldata`)

Network: `epos_network`.
Volumes: `converter`, `psqldata`.

## Key .env Sections

- Monitoring: `MONITORING`, `MONITORING_URL`, `MONITORING_USER`, `MONITORING_PWD`
- Facets: `FACETS_DEFAULT`, `FACETS_TYPE_DEFAULT`
- Component exposure toggles: `LOAD_*_API` (format `enabled:admin_only`)
- AAI: `IS_AAI_ENABLED`, `AAI_SERVICE_ENDPOINT`
- RabbitMQ: `RABBITMQ_HOST`, `RABBITMQ_USERNAME`, `RABBITMQ_PASSWORD`, `RABBITMQ_VHOST`
- Postgres: `POSTGRES_*`, `POSTGRESQL_CONNECTION_STRING`, `CONNECTION_POOL_*`, `PERSISTENCE_NAME`
- Email sender (defaults are placeholders): `MAIL_*`, `SENDER_*`, `DEV_EMAILS`
- Image tags: `*_IMAGE` (see `.env` for the full list)
