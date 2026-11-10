# Data model

Status: **proposal.** Updated in the same PR as every migration.
Conventions: PostgreSQL 16, `snake_case`, surrogate keys `bigint generated always as
identity` unless stated, timestamps `timestamptz` in UTC, business dates `date`.
"Append-only" means: no `UPDATE`/`DELETE` granted to the application role, plus a
trigger that rejects them.

## 1. Schema `platform` (shared, no client fund data)

### client
| Field | Type | Notes |
|---|---|---|
| client_id | bigint PK | |
| code | text unique | lowercase, `[a-z0-9_]{2,30}`; schema name is `client_<code>` |
| name | text | |
| status | text | `active`, `suspended`, `offboarded` |
| created_at | timestamptz | |

### provider
| Field | Type | Notes |
|---|---|---|
| provider_code | text PK | e.g. `SAMPLE_A`, `SAMPLE_B`, later `MORNINGSTAR`. `MANUAL` (corrections) and `CLIENT` (data entered when a client creates a fund) are reserved |
| name | text | |
| connector_type | text | `api`, `file`, `manual` |
| active | boolean | |

### licence
| Field | Type | Notes |
|---|---|---|
| licence_id | bigint PK | |
| client_id | FK client | |
| provider_code | FK provider | |
| valid_from | date | |
| valid_to | date null | null = open-ended |
| scope | jsonb null | optional restriction (data point groups); null = all (A-05) |
| created_by, created_at | | audited |

`MANUAL` needs no licence. A client without a valid licence for a provider never sees
that provider's values in golden copy, API, UI or exports.

### data_point (catalogue)
| Field | Type | Notes |
|---|---|---|
| data_point_code | text PK | canonical code, e.g. `share_class.isin`, `fees.ongoing_charges`, `nav.value` |
| label | text | |
| entity_level | text | `fund`, `share_class` |
| temporal_kind | text | `static` (valid from effective date until replaced) or `time_series` (one value per date) |
| value_type | text | `text`, `numeric`, `date`, `boolean`, `code` |
| unit | text null | `percent`, `currency`, `ratio`, ... |
| code_list | text null | reference to an allowed-values list for `code` |
| standard | text null | `EMT`, `EPT`, `EET`, `openfunds` |
| standard_ref | text null | field number/ID in that standard, **to be filled from the official spec** |
| standard_version | text null | e.g. EMT version |
| quality_checks | jsonb | list of check definitions (see 4.3) |
| config_version | text | hash of the catalogue file it came from |

### config_snapshot
| Field | Type | Notes |
|---|---|---|
| config_version | text PK | content hash |
| kind | text | `catalogue`, `mapping`, `priority_default` |
| name | text | e.g. provider code |
| git_commit | text | |
| content | jsonb | full config as loaded, so old versions stay reproducible |
| loaded_at | timestamptz | |

### app_user / user_client_role / api_client
| Table | Key fields |
|---|---|
| app_user | user_id, email (unique), display_name, auth_subject, status, created_at |
| user_client_role | user_id, client_id, role (`viewer`, `steward`, `client_admin`), granted_by, granted_at |
| api_client | api_client_id, client_id, name, secret_hash (argon2), scopes, expires_at, revoked_at |

FinFiles operators have a platform role (`operator`) stored on `app_user` (A-11).

## 2. Schema `client_<code>` (one per asset manager)

### entity
Funds and share classes known to this client.
| Field | Type | Notes |
|---|---|---|
| entity_id | bigint PK | |
| entity_type | text | `fund`, `share_class` |
| parent_entity_id | FK entity null | share class → fund |
| origin | text | `provider` (found in provider data) or `client` (created by the client in the universe) |
| created_by | text | user id, or `system` for provider-matched entities |
| created_at | timestamptz | |

Descriptive attributes (name, legal form, domicile, SFDR, currency...) are **not** columns
here: they are data points in `normalised_value`, so they keep source and both dates. For a
client-created fund they are stored with `source_code = 'CLIENT'`.

### universe_event (append-only)
Which entities the client follows. Membership at time T = last event per entity with
`occurred_at ≤ T`. Only entities in the universe are collected, resolved and delivered.
| Field | Type | Notes |
|---|---|---|
| universe_event_id | bigint PK | |
| entity_id | FK entity | share class (fund membership follows its share classes) |
| action | text | `add`, `remove`, `restore` |
| reason | text null | optional for add, recommended for remove |
| exception_id | FK null | when added from a "not in universe" exception |
| actor | text | user id |
| occurred_at | timestamptz | |

View `universe_current`: entities whose last event is `add` or `restore`.

### entity_identifier
| Field | Type | Notes |
|---|---|---|
| entity_id | FK entity | |
| scheme | text | `ISIN`, `LEI`, `PROVIDER_ID` |
| value | text | |
| provider_code | text null | set when scheme = `PROVIDER_ID` |
| valid_from, valid_to | date | |
| received_at | timestamptz | |
Unique on `(scheme, value, provider_code, valid_from)`. ISIN is the primary match key for
share classes (A-09).

### ingestion_run
| Field | Type | Notes |
|---|---|---|
| ingestion_run_id | bigint PK | |
| provider_code | text | |
| trigger | text | `schedule`, `manual`, `reprocess` |
| triggered_by | text | user id or `system` |
| started_at, finished_at | timestamptz | |
| status | text | `running`, `succeeded`, `failed`, `partial` |
| stats | jsonb | payloads, values, exceptions counts |

### raw_payload (append-only)
| Field | Type | Notes |
|---|---|---|
| raw_payload_id | bigint PK | |
| ingestion_run_id | FK | |
| provider_code | text | |
| received_at | timestamptz | **the "received" date for all values derived from it** |
| source_ref | text | URL path (no credentials) or file name |
| content_type | text | |
| sha256 | text | duplicates are stored too but flagged via `duplicate_of` |
| duplicate_of | FK raw_payload null | |
| size_bytes | bigint | |
| payload | bytea | exact bytes as received |

### normalised_value (append-only)
| Field | Type | Notes |
|---|---|---|
| normalised_value_id | bigint PK | |
| entity_id | FK entity | |
| data_point_code | text | catalogue code |
| source_code | text | provider code or `MANUAL` |
| effective_date | date | **date the value applies to** |
| received_at | timestamptz | **date we received it** (copied from raw payload, or entry time for MANUAL) |
| value_text | text null | exactly one `value_*` column is set, matching catalogue type |
| value_numeric | numeric null | no floats |
| value_date | date null | |
| value_boolean | boolean null | |
| currency | char(3) null | ISO 4217 when unit = currency |
| raw_payload_id | FK raw_payload null | null only for MANUAL |
| ingestion_run_id | FK null | |
| mapping_version | text null | config_version of the mapping used |
| manual_correction_id | FK null | set for MANUAL values |

Index: `(entity_id, data_point_code, effective_date, source_code, received_at desc)`.

### manual_correction (append-only)
| Field | Type | Notes |
|---|---|---|
| manual_correction_id | bigint PK | |
| entity_id, data_point_code, effective_date | | target |
| reason | text | mandatory |
| exception_id | FK null | if raised from the exception queue |
| created_by | text | user id |
| created_at | timestamptz | |
| withdrawn_by, withdrawn_at | | withdrawal is a new correction row of type `withdrawal`, not an update (A-06) |
| kind | text | `value`, `withdrawal` |

The value itself is a `normalised_value` row with `source_code = 'MANUAL'`.
Provider rows are never changed.

### priority_override (append-only, versioned)
| Field | Type | Notes |
|---|---|---|
| priority_override_id | bigint PK | |
| level | text | `client` or `data_point` |
| data_point_code | text null | required when level = `data_point` |
| source_order | text[] | e.g. `{MANUAL, SAMPLE_B, SAMPLE_A}` |
| valid_from | timestamptz | rule applies to computations from this moment |
| superseded_by | FK null | set by a new version (only allowed update, via a dedicated function) |
| created_by, created_at | | |

Global default comes from `config/priority/default.yaml` (`config_snapshot`).

### quality_exception (exception queue)
| Field | Type | Notes |
|---|---|---|
| exception_id | bigint PK | |
| entity_id, data_point_code, effective_date | | |
| source_code | text null | |
| normalised_value_id | FK null | |
| check_code | text | e.g. `required`, `range`, `nav_jump`, `divergence`, `mapping_error`, `stale`, `not_in_universe` |
| severity | text | `blocking` (value not eligible for golden) or `warning` |
| details | jsonb | |
| created_at | timestamptz | |
| status | text | `open`, `resolved`, `accepted`, `ignored` |

Status changes are written to `quality_exception_event` (exception_id, status, actor,
comment, at), so the history is kept and the current status is derived from the last event.

### golden_value (append-only, versioned)
| Field | Type | Notes |
|---|---|---|
| golden_value_id | bigint PK | |
| entity_id, data_point_code, effective_date | | |
| value_* / currency | | copy of the chosen value |
| normalised_value_id | FK | chosen candidate |
| source_code | text | |
| rule_level | text | `data_point`, `client`, `global` |
| rule_ref | text | priority_override_id or config_version |
| fallback_reason | text null | why a higher-priority source was skipped |
| computed_at | timestamptz | |
| superseded_at | timestamptz null | set when a newer version exists (A-13) |

View `golden_value_current`: latest version per `(entity_id, data_point_code, effective_date)`.

### export_job / delivery_log
| Table | Key fields |
|---|---|
| export_job | export_job_id, name, format, schedule (cron), filter (jsonb), destination_ref, active, created_by |
| export_run | export_run_id, export_job_id, started_at, finished_at, status, file_name, sha256, row_count |

### audit_event (append-only)
| Field | Type | Notes |
|---|---|---|
| audit_event_id | bigint PK | |
| occurred_at | timestamptz | |
| actor | text | user id, api client id or `system` |
| action | text | e.g. `correction.create`, `override.create`, `exception.resolve`, `export.run` |
| object_type, object_id | text | |
| details | jsonb | no secrets, no full payloads |
| request_id | text | correlation id |

Platform-level actions (licences, users, client provisioning) are audited in
`platform.audit_event` with the same structure.

## 3. Reproducing past states

- Normalised view as known at time T:
  latest `received_at ≤ T` per `(entity, data_point, source, effective_date)`.
- Golden copy as delivered at time T:
  `golden_value` rows with `computed_at ≤ T`, latest per key.
- Static data at effective date D: latest `effective_date ≤ D`.

## 4. Configuration file formats (sketch)

### 4.1 Mapping (`config/mappings/sample_a.yaml`)
```yaml
provider: SAMPLE_A
format: json
entity:
  identifier: { scheme: ISIN, path: "$.shareClass.isin" }
  level: share_class
effective_date: { path: "$.asOf", format: "%Y-%m-%d" }
fields:
  - data_point: share_class.name
    path: "$.shareClass.name"
  - data_point: fees.ongoing_charges
    path: "$.costs.ocf"
    transform: { multiply: 100 }   # provider sends ratio, canonical is percent
  - data_point: nav.value
    path: "$.nav.amount"
    currency_path: "$.nav.currency"
```

### 4.2 Priority default (`config/priority/default.yaml`)
```yaml
default: [MANUAL, SAMPLE_A, SAMPLE_B, CLIENT]   # CLIENT last: provider data wins once it exists (A-30)
```

### 4.3 Catalogue entry (`config/catalogue/fees.yaml`)
```yaml
- code: fees.ongoing_charges
  label: Ongoing charges
  entity_level: share_class
  temporal_kind: static
  value_type: numeric
  unit: percent
  standard: EMT
  standard_ref: TBD        # fill from official EMT field list
  quality_checks:
    - { check: required, severity: warning }
    - { check: range, min: 0, max: 10, severity: blocking }
    - { check: divergence, tolerance_abs: 0.05, severity: warning }
```

## 5. First-scope catalogue (draft)

Codes are canonical; `standard_ref` to be completed from the official FinDatEx and
openfunds field lists (A-02).

| Code | Level | Kind | Type | Standard |
|---|---|---|---|---|
| fund.name | fund | static | text | openfunds |
| fund.legal_form | fund | static | code | openfunds |
| fund.domicile | fund | static | code (ISO 3166) | openfunds / EMT |
| fund.management_company_lei | fund | static | text | EMT |
| fund.sfdr_classification | fund | static | code (6/8/9) | EET |
| share_class.isin | share_class | static | text | EMT / openfunds |
| share_class.name | share_class | static | text | EMT / openfunds |
| share_class.currency | share_class | static | code (ISO 4217) | EMT / openfunds |
| share_class.distribution_policy | share_class | static | code | openfunds |
| share_class.inception_date | share_class | static | date | openfunds |
| share_class.sri | share_class | static | numeric (1–7) | EMT |
| fees.ongoing_charges | share_class | static | numeric % | EMT / EPT |
| fees.management_fee | share_class | static | numeric % | EMT / EPT |
| fees.max_entry_cost | share_class | static | numeric % | EMT |
| fees.max_exit_cost | share_class | static | numeric % | EMT |
| fees.transaction_costs | share_class | static | numeric % | EMT / EPT |
| fees.performance_fee | share_class | static | numeric % | EMT / EPT |
| nav.value | share_class | time_series | numeric + currency | openfunds |
| nav.total_net_assets | fund | time_series | numeric + currency | openfunds |
