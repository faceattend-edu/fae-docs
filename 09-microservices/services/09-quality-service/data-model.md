# Data Model — Quality Service

> ISO/IEC 25010, ISO/IEC 29110 and ISTQB assessment instruments persisted in their own
> `quality` schema. Quality is **tooling, not a bounded context** — it has no domain model of its
> own, but its 7 tables **are declared in the DBML** (`QUALITY` section of
> `FULL/database/faceattend_edu_mr_v4.dbml`) with the standard audit block and naming rules.

---

## Database engine

**Engine:** PostgreSQL 17, schema `quality`
**Justification:** assessments are structured, relational line-items (header + items/ratings) that
must survive restarts — `src/main.ts` refuses to boot without Postgres (no in-memory fallback).

---

## Schema

> The schema and all seven tables are shipped by the **configuration** changelog:
> `FULL/database/07-ms-configuration-db/` (`01-ddl/01-schemas/002-create-quality-schema.yaml`,
> `03-tables/004..010`, `02-types/003..007`, `04-tcl/002-triggers-audit-quality.yaml`).
> Every table carries the audit block (`created_at, updated_at, deleted_at, created_by, updated_by,
> deleted_by, row_version DEFAULT 1`).

### Enum types (native PG, schema `quality`)

| Type | Values (as created) |
|------|---------------------|
| `project_status` | default `Planned` |
| `evaluation_status` | default `Draft` |
| `objective_rating` | `N` / `P` / `L` / `F` grades |
| `iso29110_process` | `PM` (project management) / `SI` (software implementation) |
| `quality_level` | ISO/IEC 29110 process levels |

### Table: `quality_project`

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| project_id | BIGSERIAL | No | Project identifier | PK |
| project_name | VARCHAR(200) | No | Software project under evaluation | Not null |
| description | VARCHAR(1000) | Yes | Scope | NULL |
| customer | VARCHAR(200) | Yes | Accepting customer (WP10 evidence) | NULL |
| planned_start_on | DATE | Yes | Planned start | `chk_quality_project_dates (start <= end)` |
| planned_end_on | DATE | Yes | Planned end | same CHECK |
| project_status | `project_status` | No | Lifecycle | Default `Planned` |

**Indexes:** `idx_quality_project_status`, `idx_quality_project_deleted`

### Table: `process_assessment`

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| assessment_id | BIGSERIAL | No | Assessment identifier | PK |
| project_id | BIGINT | No | Parent project → `quality_project` | Not null, **FK `ON DELETE CASCADE`** |
| process_code | `iso29110_process` | No | `PM` or `SI` | Not null |
| assessor_name | VARCHAR(200) | No | Who assessed | Not null |
| assessment_status | `evaluation_status` | No | State | Default `Draft` |
| comments | TEXT | Yes | Notes | NULL |
| process_score | DECIMAL(5,1) | Yes | % achievement 0–100 | Derived from `process_assessment_rating` |
| process_rating | `objective_rating` | Yes | Aggregate N/P/L/F | Derived from `process_score` |

**Indexes:** `idx_process_assessment_project`, `idx_process_assessment_process`, `idx_process_assessment_deleted`
**FK:** `fk_process_assessment_project` → `quality_project` (intra-schema — allowed: one context)

### Remaining tables (see their changesets for columns)

| Table | Changeset | Shape |
|-------|-----------|-------|
| `quality_evaluation` | `03-tables/005` | Evaluation header (`evaluation_status`) |
| `quality_evaluation_item` | `03-tables/006` | Line items; soft-delete via `UPDATE` (repository upsert) |
| `process_assessment_rating` | `03-tables/008` | Per-objective ratings `N/P/L/F`, normalized |
| `istqb_assessment` | `03-tables/009` | ISTQB header; score recalculated on read |
| `istqb_assessment_item` | `03-tables/010` | 30 Likert items feeding the header score |

---

## Relationships

- `process_assessment.project_id` → `quality_project.project_id` (intra-schema FK, cascade delete)
- Assessment items/ratings hang off their headers (same schema)
- **No cross-context references at all**: quality does not point at persons, cohorts or schools —
  it is tooling data, which is why it is outside the bounded-context model (`AGENTS.md` §1)

---

## Modeling decisions

- **Qualified SQL, validated search_path:** every statement is written as `quality.<table>` and the
  pool sets `-c search_path=<schema>,public` from `DB_SCHEMA` → `PG_SCHEMA` → `quality`, validated
  against an identifier pattern (`src/infrastructure/db/database.ts`) — no SQL injection through env.
- **Header + items:** scores are never stored alone; the item rows that produced them are kept, so
  a score can be audited and recomputed.
- **Soft delete + `row_version`:** same audit/optimistic-locking contract as every other schema.

---

## Schema migration

**Tool:** Liquibase 4.29, chained containers (ADR-010) — quality rides along in the
**configuration** chain (`fae-configuration-liquibase`), not in a chain of its own.
**Script location:** `FULL/database/07-ms-configuration-db/` (see above).
**Naming convention:** `NNN-<action>-<object>-table.yaml`, additive, each with a rollback.

**Rollback policy:**
- Assessment history is evidence: resolve/revise, do not delete.
- `05-rollbacks/changelog.yaml` carries the per-changeset down path.
