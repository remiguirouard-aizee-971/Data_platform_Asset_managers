# Backlog

One step = one branch = one pull request. Each step ships with tests and updated docs.
Steps marked ⏸ need explicit approval before starting.
Prerequisite for step 1: IT Lead review of the ⚠ items in [assumptions.md](assumptions.md)
(at least A-01, A-03, A-04).

## Phase 1: Foundations and database schema

| # | Step | Done when |
|---|---|---|
| 1a ⏸ | **Repository skeleton: `core`.** `services/core/` Spring Boot project from Spring Initializr, Gradle wrapper, dependency locking + checksum verification (A-21), packages `ingestion`, `rawstore`, `normalisation`, `golden`, `catalogue`, `audit`, `access`, health endpoint, JUnit smoke test, Testcontainers PostgreSQL test, Dockerfile. | `./gradlew check` green; image builds. |
| 1b ⏸ | **Repository skeleton: `web`.** `services/web/` Laravel project via Composer, `composer.lock`, PHPStan (Larastan), Pint, Pest, health route, Dockerfile (nginx + PHP-FPM, PHP 8.4). | `composer test` green; image builds. |
| 1c | **Local stack + CI.** Root `docker-compose.yml` (PostgreSQL 16, core, web), `.env.example`, quality gate script for both services, CI pipeline (A-20), secret scan. | `docker compose up` runs all three; both health endpoints respond; CI green; no secrets. |
| 2 | **`platform` schema migration.** Tables `client`, `provider`, `licence`, `data_point`, `config_snapshot`, `app_user`, `user_client_role`, `api_client`, `audit_event`. | Flyway migration (in `core`) applies cleanly to an empty DB in a Testcontainers test; constraints tested (codes, `MANUAL` reserved, licence dates). |
| 3 | **Client schema template + provisioning.** Migration template for `client_<code>` (all tables in data-model §2), per-client DB role, append-only triggers, CLI `provision-client <code>`. | Two synthetic clients provisioned; tests prove UPDATE/DELETE on append-only tables fail and **client A cannot read client B**. |
| 4 | **Tenant context in both services.** `core`: `TenantContext` + transaction hook doing `SET LOCAL ROLE/search_path`. `web`: middleware resolving the client from the user/API token and setting the read-only role per request. Both refuse to query without a context. | Tests in both services for missing context and cross-tenant access. |
| 5 | **Catalogue config + loader.** YAML schema, first-scope catalogue (data-model §5), validation, load into `data_point` + `config_snapshot`. | Invalid config fails CI; loader idempotent; snapshot hash stable. |

## Phase 2: One connector end to end on synthetic data

| # | Step | Done when |
|---|---|---|
| 6 | **Synthetic data generator.** Script generating a fictional fund universe (e.g. 3 funds, 8 share classes, fake but valid-format ISINs with check digit, clearly marked as test data) and **Sample Provider A** JSON responses for static, fees and 30 days of NAV, including deliberate errors (missing value, NAV spike, unknown ISIN). | Deterministic output from a seed; fixtures committed; README marks them synthetic. |
| 7 | **Connector interface + raw store.** `Connector` protocol, `RawPayload`, `ingestion_run`, raw store write with SHA-256 and duplicate flag. | Unit tests; immutability test; `received_at` set by the platform, not the provider. |
| 8 | **Sample Provider A API connector.** Small mock HTTP server in Docker Compose serving the generated JSON; connector with auth via env var, paging, retry/backoff, timeouts. | `ingest SAMPLE_A --client demo_a` stores raw payloads; tests use the mock only. |
| 9 | **Mapping config + normaliser.** Mapping YAML for SAMPLE_A, JSON-path extraction, transforms, entity matching on ISIN (A-09), mapping errors → exception queue. | Normalised values with both dates and source; reprocessing from raw is reproducible. |
| 10 | **Golden copy v1.** Resolver with global default priority only, licence filter, basic quality checks (`required`, `range`, `type`), golden versioning. | Golden values computed; no-licence client gets none; blocking check excludes a value. |
| 11 | **Delivery API v1.** Read-only endpoints: list entities, golden values by ISIN with `as_of_date` and `known_at`, lineage; OpenAPI spec exported to `docs/api/openapi.yaml`; API-client auth (A-11), client-scoped. | Contract tests; cross-client and unlicensed access return 403/404; spec published for Fundscreener team. |
| 12a ⏸ | **Morningstar demo data via MCP** (C-05, after A-25). Fetch a small agreed fund list through the Morningstar MCP connector into `demo_data/morningstar/` (git-ignored), add a file mapping config for this layout, ingest through the same pipeline. No real data in tests. | Demo client shows Morningstar values with lineage; `git status` shows no demo data. |
| 12 ⏸ | **Demo checkpoint.** End-to-end run on synthetic data: ingest → raw → normalised → golden → API. Review with product owner and IT Lead. | Demo script in `docs/demo.md`. |

## Phase 3: Multi-source and stewardship

| # | Step |
|---|---|
| 13 | Sample Provider B as **file connector** (CSV, EMT-like layout) with its own mapping; overlapping funds with A, some conflicting values. |
| 14 | Divergence, NAV jump and staleness checks; exception queue API. |
| 15 | Priority overrides per client and per data point (DB tables, API, audit). |
| 16 | Manual corrections as source `MANUAL` (API, audit, withdrawal). |
| 17 | Audit trail coverage review: every write path emits `audit_event`; platform audit for licences/users. |
| 18 | Web interface v1: login (OIDC), browse values and lineage, exception queue, corrections, overrides. |
| 19 | Scheduled exports (CSV/XLSX/JSON) to folder/SFTP. |
| 20 | Scheduling of ingestion in the worker (A-10), run monitoring. |

## Phase 4: Real provider and Fundscreener

| # | Step |
|---|---|
| 21 ⏸ | **Morningstar connector**, only after A-18 is confirmed: licence terms, API docs, credential model. Mapping config only; tests on synthetic fixtures shaped like the documented responses. |
| 22 | Fundscreener integration support: API keys for Fundscreener per client, example calls in its language (A-16), performance check on its query patterns. |
| 23 | Hardening for production: backups/restore test (A-15), log shipping, rate limiting, pen-test findings, ISMS asset register and risk entries. |
