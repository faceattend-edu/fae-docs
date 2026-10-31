# Runbook — Scheduling Service

> A runbook is for someone on-call at 3am. Each section must be executable without needing additional context.

**Service:** 04-scheduling-service (`FULL/back-end/04-ms-scheduling`)
**Stack:** Java 21 + Spring Boot (Maven)
**Version:** 1.0
**Last updated:** 2026-10-17

---

## 1. Quick service information

| Field | Value |
|-------|-------|
| Local port | 8084 |
| Compose service / container | `ms-scheduling` / `fae-ms-scheduling` |
| Through Kong | `http://localhost:8000/api/v1/{environments,schedule-blocks,class-sessions}` |
| Health endpoint | `GET /actuator/health` → `"status":"UP"` (compose healthcheck) |
| DB | PostgreSQL schema `scheduling` (container `fae-postgres`, db `faceattend_db`) |
| Dashboard / alert channel | Not deployed — see `13-operations/observability.md` |
| Target RTO / RPO | Not defined (`NFR-010` pending) |

---

## 2. Verify the service is healthy

```bash
curl -s http://localhost:8084/actuator/health
docker compose -f back-end/docker-compose.yml ps ms-scheduling

# Through the gateway
curl -s http://localhost:8000/health
```

---

## 3. Frequent alerts and what to do

### Symptom: healthcheck DOWN / container restarting

```bash
docker compose -f back-end/docker-compose.yml logs --tail=100 ms-scheduling
```

**Decision tree:**
- Liquibase migration failed → section 4.2
- `postgres` unreachable → check `fae-postgres` health; scheduling cannot work without DB
- Crash loop after a deploy → roll back (section 5.2)

### Symptom: "slot not available" errors spike

Usually not an outage: `uq_block_environment_slot` / `uq_block_instructor_slot` rejected an overlap.
Identify the conflicting rows (same room or same instructor at the same `starts_at`) before
"fixing" anything:

```sql
SET search_path TO scheduling;
SELECT environment_id, day_of_week, starts_at, COUNT(*)
FROM schedule_block WHERE deleted_at IS NULL
GROUP BY 1,2,3 HAVING COUNT(*) > 1;
```

---

## 4. Procedures by component

### 4.1 Database

```bash
docker exec -it fae-postgres psql -U postgres -d faceattend_db
```
```sql
SET search_path TO scheduling;
SELECT * FROM schedule_block WHERE deleted_at IS NULL LIMIT 20;
SELECT * FROM class_session WHERE session_date = CURRENT_DATE;
```

### 4.2 Migrations

Changesets live in `FULL/database/04-ms-scheduling-db/` and run automatically on `docker compose up`
(Liquibase chain, ADR-010). Inspect the run:

```bash
docker logs fae-scheduling-liquibase 2>&1 | tail -50
```

A failed changeset stops the chain — fix **forward** with a new changeset; never edit an applied one.

---

## 5. Maintenance operations

### 5.1 Rebuild / restart

```bash
docker compose -f back-end/docker-compose.yml up -d --build ms-scheduling
```

### 5.2 Rollback

```bash
git revert <bad-sha>
docker compose -f back-end/docker-compose.yml up -d --build ms-scheduling
```

Schema rollbacks are per changeset in `FULL/database/04-ms-scheduling-db/05-rollbacks/` — run one
only when you accept the data loss it implies.

---

## 6. Communication during incident

| Event | Channel | Sample message |
|-------|---------|---------------|
| P0 detected | #incidents | [P0 START] Scheduling Service degraded. Investigating. |
| Update every 15 min | #incidents | [P0 UPDATE] Cause under review. Action in progress. |
| Resolution | #incidents + stakeholders | [P0 RESOLVED] Recovery completed. Post-mortem scheduled. |

---

## 7. Post-incident checklist

- [ ] Service healthy at `/actuator/health`
- [ ] Incident recorded in `13-operations/incident-management.md`
- [ ] Stakeholders notified of resolution
- [ ] Improvement ticket created in project backlog
