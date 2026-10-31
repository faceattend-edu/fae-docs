# Runbook — Notification Service

> A runbook is for someone on-call at 3am. Each section must be executable without needing additional context.

**Service:** 08-notification-service (`FULL/back-end/08-ms-notification`)
**Stack:** TypeScript + Fastify + Zod + Drizzle + pino (rewritten from Go/Gin 2026-10-18 — ADR-021)
**Version:** 1.0
**Last updated:** 2026-10-20

---

## 1. Quick service information

| Field | Value |
|-------|-------|
| Local port | 8088 |
| Compose service / container | `ms-notification` / `fae-ms-notification` |
| Through Kong | `http://localhost:8000/api/v1/{alerts,alert-types}` |
| Health endpoints | `GET /health`, `GET /api/v1/health`, `GET /health/quality` |
| Internal endpoint | `POST /api/v1/emails` (not routed publicly; called by Identity) |
| DB | PostgreSQL schema `notification` (container `fae-postgres`, db `faceattend_db`) |
| Key env vars | `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`/`SMTP_PASSWORD`, `EMAIL_INTERNAL_TOKEN`, `KAFKA_BROKERS` (enables the Kafka consumers — without it the service starts with none; the class-session consumer and schedulers also need `NOTIFICATION_SERVICE_USERNAME`/`PASSWORD`), `IDENTITY_URL`, `AUTHORIZATION_URL`, `WEBHOOK_URL`/`WEBHOOK_SECRET` |
| Dashboard / alert channel | Not deployed — see `13-operations/observability.md` |
| Target RTO / RPO | Not defined (`NFR-010` pending) |

---

## 2. Verify the service is healthy

```bash
curl -s http://localhost:8088/health
docker compose -f back-end/docker-compose.yml ps ms-notification

# Through the gateway
curl -s "http://localhost:8000/api/v1/alert-types" -H "Authorization: Bearer $TOKEN"
```

---

## 3. Frequent alerts and what to do

### Symptom: healthcheck DOWN / container restarting

```bash
docker compose -f back-end/docker-compose.yml logs --tail=100 ms-notification
```

**Decision tree:**
- `postgres unavailable — refusing to start` (or similar boot error) → check `fae-postgres`
- Liquibase seed failed → section 4.2
- Crash loop after a deploy → roll back (section 5.2)

### Symptom: "mail not arriving"

Not an outage of this service — check configuration first:

```bash
docker compose -f back-end/docker-compose.yml exec ms-notification env | grep -i smtp
```

- Empty `SMTP_HOST` → the API answers `202 {"status":"skipped","reason":"smtp_not_configured"}`:
  mails are being **intentionally skipped**, fix the variable.
- `smtp_not_configured` absent but errors in logs → `docker compose logs ms-notification | grep -i smtp`
  (auth/connectivity to the relay).

### Symptom: 401 on `POST /api/v1/emails`

The caller must send `X-Internal-Token` matching `EMAIL_INTERNAL_TOKEN` (and go through Kong's
session auth when routed externally). Rotate the token in the env file and restart the container.

---

## 4. Procedures by component

### 4.1 Database

```bash
docker exec -it fae-postgres psql -U postgres -d faceattend_db
```
```sql
SET search_path TO notification;
SELECT * FROM alert_type WHERE deleted_at IS NULL;          -- 5 seeded rows
SELECT * FROM alert WHERE resolved_at IS NULL ORDER BY raised_at DESC LIMIT 20;
```

### 4.2 Migrations

Changesets live in `FULL/database/08-ms-notification-db/` and run on `docker compose up`
(Liquibase chain, ADR-010):

```bash
docker logs fae-notification-liquibase 2>&1 | tail -50
```

Seeds are idempotent; a failed changeset stops the chain — fix **forward**, never edit an applied
changeset, and never edit seeded rows (create a new incremental changeset instead).

---

## 5. Maintenance operations

### 5.1 Rebuild / restart

```bash
docker compose -f back-end/docker-compose.yml up -d --build ms-notification
```

### 5.2 Rollback

```bash
git revert <bad-sha>
docker compose -f back-end/docker-compose.yml up -d --build ms-notification
```

### 5.3 Resolve stuck alerts (operational data fix)

```sql
SET search_path TO notification;
UPDATE alert SET resolved_at = NOW(), updated_at = NOW(), row_version = row_version + 1
WHERE resolved_at IS NULL AND raised_at < NOW() - INTERVAL '30 days' AND deleted_at IS NULL;
```
Prefer the API (`PUT /api/v1/alerts/{id}/resolve`) when Kong is healthy.

---

## 6. Communication during incident

| Event | Channel | Sample message |
|-------|---------|---------------|
| P0 detected | #incidents | [P0 START] Notification Service degraded / mails not delivered. Investigating. |
| Update every 15 min | #incidents | [P0 UPDATE] Cause under review. Action in progress. |
| Resolution | #incidents + stakeholders | [P0 RESOLVED] Recovery completed. Post-mortem scheduled. |

---

## 7. Post-incident checklist

- [ ] `/health` returns OK and mails flow again (or the skip reason is understood)
- [ ] Incident recorded in `13-operations/incident-management.md`
- [ ] Stakeholders notified of resolution
- [ ] Improvement ticket created in project backlog
