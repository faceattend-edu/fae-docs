# Runbook — Quality Service

> A runbook is for someone on-call at 3am. Each section must be executable without needing additional context.

**Service:** 09-quality-service (`FULL/back-end/09-ms-quality`)
**Stack:** TypeScript + Fastify + Zod + `pg` (Pool)
**Version:** 1.0
**Last updated:** 2026-10-17

---

## 1. Quick service information

| Field | Value |
|-------|-------|
| Local port | 8089 |
| Compose service / container | `ms-quality` / `fae-ms-quality` |
| Through Kong | `http://localhost:8000/api/v1/quality…` (rate-limit 100/min, Redis) |
| Health endpoints | `GET /health` (compose healthcheck), `GET /health/ready`, `GET /health/quality` |
| DB | PostgreSQL schema `quality` (container `fae-postgres`, db `faceattend_db`) |
| Key env vars | `DB_SCHEMA` / `PG_SCHEMA` (default `quality`), `DATABASE_URL`-style pool config |
| Dashboard / alert channel | Not deployed — see `13-operations/observability.md` |
| Target RTO / RPO | Not defined (`NFR-010` pending) |

> **Gateway quirk:** `GET /health/quality` **through Kong** is routed to the `quality-health`
> upstream, whose URL is `http://ms-academic:8083` — i.e. the gateway answers it from Academic.
> To probe *this* service, hit `http://localhost:8089/health/quality` directly.

---

## 2. Verify the service is healthy

```bash
curl -s http://localhost:8089/health
curl -s http://localhost:8089/health/ready     # includes the DB probe
docker compose -f back-end/docker-compose.yml ps ms-quality
```

`/health/ready` returns `200 {"status":"ready",…}` only when the Postgres probe is `up`
(`503 not-ready` otherwise).

---

## 3. Frequent alerts and what to do

### Symptom: container restarting loop, log says `postgres unavailable — refusing to start`

This service has **no in-memory fallback** (DEC-001) — it refuses to boot without the DB, and
Compose restarts it.

```bash
docker compose -f back-end/docker-compose.yml logs --tail=100 ms-quality
docker exec fae-postgres pg_isready -U postgres -d faceattend_db
```

**Decision tree:**
- `fae-postgres` unhealthy → fix Postgres first; quality comes back automatically
- Schema `quality` missing (fresh environment) → the configuration Liquibase chain has not run →
  section 4.2
- Credentials/schema env wrong → check `DB_SCHEMA`/`PG_SCHEMA` (must be a plain identifier)

### Symptom: 429 responses on `/api/v1/quality…`

Expected behavior: Kong rate-limits this upstream to **100/min** (Redis policy). A burst of
assessment saves + reads from one client can hit it; back off or raise the limit in
`99-api-gateway/kong/kong.yml`.

### Symptom: readiness flapping but queries work

Check pool settings and slow queries (`SET search_path TO quality;` then inspect
`pg_stat_activity`); readiness only mirrors the health probe.

---

## 4. Procedures by component

### 4.1 Database

```bash
docker exec -it fae-postgres psql -U postgres -d faceattend_db
```
```sql
SET search_path TO quality;
SELECT * FROM quality_project WHERE deleted_at IS NULL ORDER BY project_id;
SELECT assessment_id, project_id, process_code, assessment_status, process_score
FROM process_assessment WHERE deleted_at IS NULL;
```

### 4.2 Migrations

Quality has **no changelog of its own**: its schema and tables ship with the **configuration**
chain (see `data-model.md`).

```bash
docker logs fae-configuration-liquibase 2>&1 | tail -50
```

A failed changeset stops the chain — fix **forward** with a new changeset; never edit an applied one.

---

## 5. Maintenance operations

### 5.1 Rebuild / restart

```bash
docker compose -f back-end/docker-compose.yml up -d --build ms-quality
```

### 5.2 Rollback

```bash
git revert <bad-sha>
docker compose -f back-end/docker-compose.yml up -d --build ms-quality
```

Schema rollbacks live in `FULL/database/07-ms-configuration-db/05-rollbacks/` — run one only if
you accept losing the assessment rows created after it.

### 5.3 Recompute a score (data repair)

ISTQB header scores are recalculated on read from `istqb_assessment_item`, and
`process_score`/`process_rating` derive from `process_assessment_rating`. If a header looks wrong,
re-read it through the API before touching rows by hand.

---

## 6. Communication during incident

| Event | Channel | Sample message |
|-------|---------|---------------|
| P0 detected | #incidents | [P0 START] Quality Service down (DB dependency). Investigating. |
| Update every 15 min | #incidents | [P0 UPDATE] Cause under review. Action in progress. |
| Resolution | #incidents + stakeholders | [P0 RESOLVED] Recovery completed. Post-mortem scheduled. |

---

## 7. Post-incident checklist

- [ ] `/health/ready` returns `ready` and assessment reads/writes succeed
- [ ] Incident recorded in `13-operations/incident-management.md`
- [ ] Stakeholders notified of resolution
- [ ] Improvement ticket created in project backlog
