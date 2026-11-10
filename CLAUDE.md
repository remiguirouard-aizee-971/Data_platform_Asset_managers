# CLAUDE.md: Asset Manager Data Platform (FinFiles)

## Project context

FinFiles (Utrecht, NL) builds SaaS products for the fund industry and works within
an ISO/IEC 27001 ISMS. This repository holds a new product: a **data management
platform for asset managers**.

Asset managers receive fund data from several providers (first target: Morningstar
APIs, others later). The platform:

1. collects the data,
2. maps it to one canonical model,
3. picks one trusted value per data point (golden copy),
4. delivers it (REST API, web interface, scheduled exports).

The data must also be shown in FinFiles' existing product **Fundscreener**, which is
a **read-only consumer of the delivery API**. We have no access to its code or database.
Fundscreener is built with PHP/Laravel, but it still integrates only through the API
contract (OpenAPI), never through shared code or database access.

**Status: design phase.** No application code until the backlog step is approved.
See `docs/architecture.md`, `docs/data-model.md`, `docs/assumptions.md`, `docs/backlog.md`.

## First scope

- Data: fund and share class **characteristics** (static), **fees and costs**
  (EMT/EPT style), and daily **NAV** (time series).
- Tenancy: each **asset manager is a client**. FinFiles hosts one installation on its
  own Linux servers.
- Providers: Morningstar is the first real provider. Until licence, API documentation
  and credential handling are confirmed, we build and test against **fictional
  synthetic providers** only.

## Architecture (five layers + supporting functions)

Two services in one repository, one PostgreSQL database (decision C-07):

- **`core`** (Java, Spring Boot), in `services/core/`: the data engine.
- **`web`** (PHP, Laravel), in `services/web/`: everything users and API consumers see.

| Layer | Service | Module | Responsibility |
|---|---|---|---|
| 1 Ingestion | core | `ingestion` | One connector per provider (API or file), common interface |
| 2 Raw store | core | `rawstore` | Every payload kept exactly as received, never modified |
| 3 Normalised store | core | `normalisation` | Values mapped to the canonical model |
| 4 Golden copy | core | `golden` | One value per data point via priority rules, quality checks, exception queue |
| 5 Delivery | web | `Delivery` | REST API (incl. Fundscreener), web interface, scheduled exports |
| Supporting | core + web | `catalogue`, `audit`, `access` | Catalogue and audit written by core; users, roles, API tokens in web; licences enforced in both |

- **core owns the database schema** (Flyway migrations) and is the only writer of data
  tables. web reads through a read-only role and sends changes (corrections, overrides,
  exception decisions) as commands to core's internal API (A-27, A-28).
- Inside each service, modules talk through explicit interfaces, not each other's tables.

## Rules that must hold from the first commit

1. **Configuration, not code.** Mapping rules and priority rules live in versioned
   configuration (`config/`, plus override tables in the database). No provider-specific
   field names or source orderings hard-coded in Java or PHP.
2. **Two dates on every value.** `effective_date` (the date the value applies to) and
   `received_at` (when we received it). Stores are append-only, so any past state can be
   reproduced ("what did we know at time T about date D").
3. **Every value keeps its source.** A manual correction is stored as its own source
   (`MANUAL`) with user and timestamp. It never overwrites provider data.
4. **Priority rules cascade:** global default → client override → data point override
   (most specific wins).
5. **One database schema per client.** No query may cross clients. Shared reference data
   lives in the `platform` schema, which holds no client fund data.
6. **Licences.** A client only reads data from providers it holds a valid licence for.
   Enforced in the golden copy and in delivery, and covered by tests.
7. **Fundscreener is a read-only API consumer.** Never assume access to its code or DB.
8. **Industry-standard names.** Align data point codes with FinDatEx (EMT, EPT, EET) and
   openfunds where a field exists. Record the reference in the catalogue.

## Security (ISO 27001)

- **Synthetic data in the repository and in tests.** Generate sample provider files and
  API responses ourselves. Never commit real provider data.
- **Exception, demo only (decision C-05):** real Morningstar data fetched through the
  Morningstar MCP connector may be used for the local demo. It is written to
  `demo_data/` (git-ignored), never committed, never used in tests or CI, and deleted
  after the demo. Licence terms for this use must be confirmed (A-25).
- **No secrets in the repository.** Use environment variables. Keep `.env.example` current
  with placeholder values only. `.env` is git-ignored.
- **Pin dependencies** (lock file with exact versions and hashes).
- **Tests for every module.** Tenant isolation and licence enforcement need explicit tests.
- **Raw store is immutable:** enforced by DB privileges and a trigger, not only by convention.
- Log no payload contents or credentials at INFO level.

## Technical defaults (assumptions, see docs/assumptions.md)

- **core:** Java LTS + Spring Boot, Gradle (dependency locking + checksum verification),
  Flyway, JUnit 5 + Testcontainers.
- **web:** PHP 8.4 + Laravel (current major), Composer (`composer.lock`), Pest/PHPUnit,
  Laravel Sanctum for API tokens, Blade for the web interface, Laravel scheduler for exports.
- PostgreSQL 16, Docker Compose for local development. Runs on standard Linux servers
  without managed cloud services. Exact versions: A-29.
- Python is no longer used (replaced 2026-11-10, decision C-07).

## How to work

- Small changes. **One branch and one pull request per feature/backlog step.**
- Follow `docs/backlog.md` in order. Stop for approval when a step says so.
- Any new design decision goes into `docs/assumptions.md` for the IT Lead to confirm.
- Update `docs/data-model.md` in the same PR as any schema migration.
- Every PR: tests pass, no secrets, dependencies pinned, docs updated.
