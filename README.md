# AMDP: data platform for asset managers

FinFiles product that collects fund data from several providers, maps it to one model,
selects one trusted value per data point and delivers it (API, UI, exports).
Fundscreener consumes the delivery API read-only.

Start with [CLAUDE.md](CLAUDE.md) and [docs/](docs/): architecture, data model,
assumptions for IT Lead review, and backlog.

## Layout

```
services/core/   Java + Spring Boot: ingestion, raw store, normalisation, golden copy, schema (Flyway)
services/web/    PHP + Laravel: delivery API, web interface, exports, users and API tokens
config/          catalogue, mappings, priority rules (configuration, not code)
docs/            architecture, data model, assumptions, backlog
```

The services are created in backlog steps 1a-1c. Local development runs in Docker
(PostgreSQL 16, core, web), so only Git and Docker are needed on a developer machine.

## Data rules

Only synthetic data in this repository. Real provider data used for demos goes to
`demo_data/`, which is git-ignored (decision C-05 in docs/assumptions.md).
