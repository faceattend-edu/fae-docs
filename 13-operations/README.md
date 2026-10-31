# 13 — Operations

> **What is this?** How the system is operated in production: how problems are detected,
> how incidents are responded to, and how availability commitments are guaranteed.

## Why this section exists

A system that "works" in staging but is impossible to operate in production is useless.
The key operations questions are:
- Do we know when something fails BEFORE users call?
- Do we know exactly what to do when something fails at 3am?
- Do we have formal availability commitments?

---

## Key concepts: SLA, SLO, SLI

**SLI (Service Level Indicator):** the metric you measure — e.g.: percentage of successful requests.

**SLO (Service Level Objective):** the target for that SLI — e.g.: "99.9% of successful requests in 30 days".

**SLA (Service Level Agreement):** the contract with the customer about those targets, with consequences.

**Error Budget:** how much you can "fail" within the SLO. If the SLO is 99.9%, you have 43.8 minutes/month of available downtime.

---

## Operational Overview: FaceAttend-Edu

FaceAttend-Edu operates as a **Polyglot Modular Monolith** deployed via Docker Compose, utilizing a tiered network strategy for security and isolation.

### Infrastructure Topology
The system is divided into three isolated network tiers to ensure that the data layer is never directly exposed to the public internet:
- **`fae-edge`**: Handles traffic between the Frontend (Expo Web/Nginx) and the API Gateway (Kong).
- **`faceattend-app`**: Internal communication between the API Gateway and the microservices (Java, TS, Python) and Redis cache.
- **`fae-data`**: Restricted access for microservices to reach PostgreSQL, MongoDB, Kafka, and Liquibase migration tools.

**Security Note:** This design ensures that the database and message broker are unreachable from the `fae-edge` network, preventing direct external access to the data tier.

### Component Stack
| Layer | Technology | Purpose |
|-------|------------|---------|
| **Edge/Gateway** | Kong OSS + Redis | Routing, opaque-session auth (ms-identity), Rate-limiting, CORS |
| **Backend (Java)** | Spring Boot 4.1.1 | Identity, Authorization, Scheduling, Attendance |
| **Backend (TS)** | Fastify + Drizzle | Academic, Configuration, Notification, Quality |
| **Backend (Python)**| FastAPI | Biometric (Face/Fingerprint embeddings) |
| **Persistence** | PostgreSQL 17, MongoDB 7 | Relational data (8 bounded-context schemas; the `quality` tables are migrated with Configuration) and Vector data |
| **Migrations** | Liquibase | Sequential DB schema evolution |

---

## What is here and how to fill it in

### `observability.md` ⭐
How the system makes its internal state visible.
**The 3 pillars of observability:**
- **Logs:** What each service records, in what format (structured JSON recommended), where they go.
- **Metrics:** What each service measures (latency, error rate, throughput), with what tool (Prometheus).
- **Traces:** How a request that passes through multiple services is tracked (OpenTelemetry, Jaeger).

**Fill in based on current stack:**
- **Gateway Logs:** Kong `kong.yml` declarative config provides the routing logic; logs should be captured from the Kong container.
- **Service Logs:** Standard output from Spring Boot, Fastify and FastAPI containers.
- **Health Checks:** Use the `/health` endpoints (as seen in the `frontend-web` healthcheck) to monitor service availability.

### `incident-management.md` ⭐
Incident response process.
**Fill in:** how severity is classified (P0/P1/P2/P3), who responds, communication channel, when to escalate, post-mortem process.

**Proposed Severities for FaceAttend-Edu:**
| Level | Description | Response SLA | Resolution SLA |
|-------|-------------|-------------|----------------|
| P0 | Biometric or Attendance system completely down | 5 min | 1 hour |
| P1 | Critical API failure (e.g., Identity/Auth) | 15 min | 4 hours |
| P2 | Non-critical service degraded (e.g., Notification/Quality) | 1 hour | 24 hours |
| P3 | Minor UI bug or documentation error | 4 hours | 1 week |

### `backup-and-recovery.md` ✅ written → [`backup-and-recovery.md`](./backup-and-recovery.md)
Data backup and recovery strategy.
- **PostgreSQL:** manual `pg_dump` of the single `faceattend_db` (8 schemas) — there is **no** separate `face_auth` database.
- **MongoDB:** `mongodump` of the biometric embedding collections.
- **Liquibase:** All schema changes are versioned in the `database/` directory, allowing for deterministic recovery of the DB state.
- **Existing Tools:** `database/scripts/backup.sh` and `database/scripts/restore.sh` (manual, no automated retention/rotation).
- **Production:** `faceattend_db` runs on Neon (managed backups + PITR, ADR-020); MongoDB on the EC2 host is manual.

---

## Correlations with other sections

| This section is fed by... | And feeds into... |
|---------------------------|-------------------|
| `back-end/SERVICES.md` | Mapping of services to monitoring targets |
| `docker-compose.yml` | Network tiering and healthcheck definitions |
| `database/ESTRUCTURA.md` | Backup requirements per schema |
| Incidents documented here | `15-project-control/technical-backlog.md` → post-incident improvements |

---

## Questions this section must answer

- **How do we know the system is failing before users report it?** Through Kong Gateway logs and container healthchecks.
- **What exactly do we do when service X fails at 3am?** Check the specific microservice logs and verify the `fae-data` network connectivity.
- **How much downtime can we have per month?** Defined by the SLO (e.g., 99.9%).
- **How do we recover data if the DB fails?** Restore PG/Mongo backups and re-run the Liquibase migration chain.
