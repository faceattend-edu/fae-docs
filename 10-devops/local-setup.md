# Local Project Setup — FaceAttend EDU

> This document ensures that any developer can have a functional local environment
> in less than 1 hour. If it takes longer, the document is incomplete.
> **Rule:** If you have to guess something or look elsewhere, add what's missing here.
>
> **The golden path is Docker.** The entire stack (Postgres + Liquibase chain +
> MongoDB + Redis + Kafka + 9 microservices + Kong + web frontend) starts with a
> single command from the repo root. Running services natively outside Docker is
> optional and only needed when you want hot-reload or IDE debugging on one
> specific service.

---

## Prerequisites

### Universal tools (all stacks)

| Tool | Minimum version | Install from | Verify with |
|------|----------------|--------------|-------------|
| Docker | 24.0+ (Desktop on Win/mac; Engine + Compose plugin on Linux) | docker.com | `docker --version` |
| Docker Compose | 2.20+ | Included with Docker Desktop / `docker-compose-plugin` | `docker compose version` |
| Git | 2.40+ | git-scm.com | `git --version` |
| Node.js (for running TS services natively, Expo dev, and the seed tool) | 20 LTS+ | nodejs.org | `node --version` |

> **RAM note:** the full stack runs ~15 containers (4 JVM Spring services,
> Postgres, Mongo, Kafka, Kong…). Give Docker at least **8 GB of RAM**
> (Docker Desktop → Resources). With less, `ms-*` healthchecks time out and
> Kong never becomes healthy.

### Project language tools (polyglot monorepo — ADR-005)

You only need the toolchain for the services you'll run **natively**; Docker
images carry everything else.

| Tool | Minimum version | Used by | Install from | Verify with |
|------|----------------|---------|--------------|-------------|
| Java (JDK) | 21 | `01-ms-identity`, `02-ms-authorization`, `04-ms-scheduling`, `05-ms-attendance` (Spring Boot 4.1.1) | adoptium.net | `java -version` |
| Maven wrapper | bundled (`./mvnw`) | same Java services | — (no global install needed) | `./mvnw -v` |
| Node.js + npm | 20 LTS+ | `03-ms-academic`, `07-ms-configuration`, `08-ms-notification`, `09-ms-quality` (Fastify+TS), Web/Mobile frontends, `tools/seed-api-test-data` | nodejs.org | `node -v && npm -v` |
| Python + Poetry | 3.12, Poetry 1.8+ | `06-ms-biometric` (FastAPI) | python.org / python-poetry.org | `python --version && poetry --version` |
| ~~Go~~ | — | ~~`08-ms-notification` (Gin)~~ **Removed 2026-10-18:** the service was rewritten in TypeScript (ADR-021); no Go remains in the repo | — | — |
| Expo CLI | via `npx expo` (no global install) | `front-end/Web`, `front-end/Mobile` | included with project deps | `npx expo --version` |

**Stack tool reference (project docs):**
- Java + Spring Boot → `back-end/0X-ms-*/STACK.md` (hexagonal layout per `_stacks/java-spring.md`)
- Node.js + TypeScript → `back-end/03-ms-academic/STACK.md`, `07-ms-configuration/STACK.md`, `08-ms-notification/STACK.md`, `09-ms-quality/STACK.md`
- Python + FastAPI → `back-end/06-ms-biometric/STACK.md`
- Full service map (stacks, ports, DBs) → `back-end/SERVICES.md`

**Recommended (not required):**
- Editor: IntelliJ IDEA (Java), VS Code (TS/Python/frontends)
- Insomnia/Postman or plain `curl` for manual endpoint testing
- Mailpit (one `docker run`, see Step 7) if you work on the email channel

---

## Local ports in use

Verified against `docker-compose.yml` (root), `back-end/docker-compose.yml` and
`database/docker-compose.yml`:

| Port | Service | Description |
|------|---------|-------------|
| **8000** | `kong-gateway` (proxy) | Main API entry point — published on `0.0.0.0` (browsers + phones) |
| **8090** | `frontend-web` | SPA served by nginx (Expo export); proxies `/api/*` to Kong — published on `0.0.0.0` |
| 8001 | `kong-gateway` (admin) | DB-less Kong admin API — loopback only (`BIND_IP`) |
| 8081 | `ms-identity` | Persons, users, auth, sessions (Java) |
| 8082 | `ms-authorization` | RBAC roles/permissions (Java) |
| 8083 | `ms-academic` | Schools, programs, cohorts, courses, enrollments (TS) |
| 8084 | `ms-scheduling` | Environments, schedule blocks, class sessions (Java) |
| 8085 | `ms-attendance` | Attendance records, justifications (Java) |
| 8086 | `ms-biometric` | Facial/fingerprint embeddings (Python + MongoDB) |
| 8087 | `ms-configuration` | Config params + biometric update cases (TS) |
| 8088 | `ms-notification` | Alerts + SMTP email channel + Kafka consumer (TS) |
| 8089 | `ms-quality` | ISO 25010 instrument & evaluations (TS) |
| 5432 | PostgreSQL 17 | Single instance, 8 schemas (`faceattend_db`) |
| 27017 | MongoDB 7 | Biometric vector store |
| 6379 | Redis 7 | Kong rate-limiting/cache + identity sessions |
| 9092 | Kafka 3.8 (KRaft) | Domain events |
| 1025 / 8025 | Mailpit (**optional**, not in compose) | Dev SMTP catch-all + web UI |

> Everything except 8000 and 8090 is bound to `BIND_IP` (default `127.0.0.1`):
> reachable from this machine, invisible to the LAN. Set `BIND_IP=0.0.0.0` in
> `.env` only to debug from another device/phone, then revert.

**Check free ports:**
```bash
# Windows
netstat -ano | findstr ":8000"

# macOS / Linux
lsof -i :8000
```

---

## Step-by-step installation

### Step 1: Clone the repository

```bash
git clone <your-org>/FaceAttend-Edu.git
cd FaceAttend-Edu          # repo root (the "FULL/" folder)
git checkout develop
```

### Step 2: Environment variables

```bash
# Windows
copy .env.example .env
# Linux/macOS
cp .env.example .env
```

**Important:** every compose file has safe defaults (`${POSTGRES_USER:-postgres}`),
so **the stack works even without a `.env`**. The file exists to override ports,
credentials, the Liquibase image tag, or host binding. There are no real secrets
in it — do not invent required variables; check `.env.example` comments first.

Key variables (full documentation lives in the comments of `.env.example`):

| Variable | Description | Value for local |
|----------|-------------|-----------------|
| `POSTGRES_USER/PASSWORD/DB` | PostgreSQL credentials | `postgres` / `postgres` / `faceattend_db` |
| `MONGO_USER/PASSWORD/DB` | MongoDB credentials | `mongoadmin` / `mongopass` / `faceattend_biometric` |
| `KONG_PROXY_PORT` / `KONG_ADMIN_PORT` | Gateway ports | `8000` / `8001` |
| `FRONTEND_PORT` | Web SPA port | `8090` (do not use 8082 — belongs to ms-authorization) |
| `EXPO_PUBLIC_API_URL` | Backend URL baked into the frontend **at build time** | `http://localhost:8000` (PC LAN IP, e.g. `http://192.168.1.10:8000`, to test from a phone) |
| `BIND_IP` | Host binding for all non-edge ports | `127.0.0.1` |
| `SMTP_HOST/PORT/...` | Email channel (ms-notification only) | empty = disabled (202 skipped); Mailpit: `host.docker.internal:1025`, STARTTLS `false` |
| `EMAIL_INTERNAL_TOKEN` | Shared secret identity→notification (`X-Internal-Token`) | empty = no check; any GUID locally |
| `SEED_*` / `BOOTSTRAP_*` | Test accounts for `tools/seed-api-test-data` | defaults in `.env.example` |

> **NEVER** commit the `.env` file. It is in `.gitignore` for a reason.
> **There is no `JWT_SECRET`:** authentication uses opaque server-side sessions
> created by ms-identity in Redis and validated through Kong/authorization —
> see `back-end/SERVICES.md`.

### Step 3: Start the whole stack

```bash
# From the repo root — this is the official entrypoint
docker compose up -d --build
```

Startup order is enforced by the root compose dependency chain (see `COMPOSE.md` §2):
`postgres` healthy → 8 Liquibase migrations **in series** (identity → authorization
→ academic → scheduling → attendance → biometric → configuration → notification) →
each `ms-*` waits for its own migration → `kong-gateway` waits for all services
healthy → `frontend-web` waits for Kong. First build takes ~15–30 min (image pulls +
4 Maven builds); subsequent starts are much faster.

```bash
# Verify: every service must show (healthy)
docker compose ps
```

> Do **not** run `back-end/docker-compose.yml` standalone — it fails with
> `service "ms-identity" depends on undefined service "postgres"` by design
> (postgres lives in `database/`). Always launch from the root.

### Step 4: Install dependencies (only for native development)

Skip entirely if you only work through Docker. Otherwise, per service:

| Stack | Services | Command (inside the service dir) |
|-------|----------|----------------------------------|
| Java + Maven wrapper | 01, 02, 04, 05 | `./mvnw install -DskipTests` |
| Node.js + npm | 03, 07, 08, 09 | `npm install` |
| Python + Poetry | 06 | `poetry install` |
| Frontends | `front-end/Web`, `front-end/Mobile` | `npm install` |
| Seed tool | `tools/seed-api-test-data` | none (zero dependencies, Node ≥ 20) |

Services run natively read the **same root `.env`** at startup (Java via
`spring.config.import optional:file:...`, Node via `process.loadEnvFile('../../.env')`,
Python via `_load_root_env()`), so no extra config is needed — just run from the
service directory.

### Step 5: Run migrations

**Nothing to do manually.** The Liquibase chain runs automatically inside Step 3
(one container per bounded context, chained with `service_completed_successfully`).
Useful checks / isolated runs:

```bash
# See the migrations applied in a schema
docker compose logs identity-liquibase        # or authorization-/academic-/...-liquibase

# Migrate only one bounded context against a running postgres
cd database/01-ms-identity-db && docker compose --env-file ../.env up liquibase

# The 8 schemas are created automatically on FIRST boot only
# (database/database-init/01-init-schemas.sql, when postgres_data is empty)
```

### Step 6: Start the services

**Option A (recommended):** already done in Step 3 — everything runs in Docker.
To rebuild just one service after code changes:

```bash
docker compose up -d --build ms-identity      # Java example
docker compose up -d --build frontend-web     # after changing EXPO_PUBLIC_API_URL or UI code
```

**Option B (hot reload while developing one service):** keep infra in Docker
(`cd database && docker compose up -d` plus mongodb/redis/kafka), stop only the
service you're editing, and run it natively from its directory:

| Service | Native dev command |
|---------|--------------------|
| Java (01, 02, 04, 05) | `./mvnw spring-boot:run` |
| TS (03, 07, 08, 09) | `npm run dev` (ts-node-dev --respawn) |
| Python (06) | `poetry run uvicorn main:app --reload --port 8086` |

Note: a natively-run service still answers on its usual port (8081–8089) and
Kong routes to it via `host.docker.internal` where configured; if routing gets
confusing, prefer Option A.

**Web frontend dev server (Metro, port 3000):**
```bash
cd front-end/Web
EXPO_PUBLIC_API_URL=http://localhost:8000 npx expo start --web
```

### Step 7: Verify it works

```bash
# 1. All containers healthy
docker compose ps

# 2. Gateway routes respond (public smoke route via Kong -> ms-quality)
curl -s http://localhost:8000/health/quality

# 3. Direct service health (Java: actuator; TS/Python: /health)
curl -s http://localhost:8081/actuator/health   # {"status":"UP"}
curl -s http://localhost:8083/health

# 4. Login through the gateway (real auth check)
curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"<a seeded user>","password":"<password>"}'

# 5. Frontend loads
open http://localhost:8090        # browser

# Optional: load cross-service test data (also serves as an E2E smoke test;
# exit 0 = every request OK or only documented known bugs)
cd tools/seed-api-test-data && npm run seed

# Optional: dev email mailbox (SMTP_HOST points here when enabled)
docker run -d -p 1025:1025 -p 8025:8025 axllent/mailpit   # UI: http://localhost:8025
```

If `docker compose ps` shows everything `(healthy)` and step 2 returns JSON,
your environment is functional.

---

## Daily workflow

```bash
# At the start of the day
git pull origin develop
docker compose up -d --build      # rebuilds only what changed; re-runs Liquibase chain (idempotent)
docker compose ps                 # confirm (healthy)

# Optional: refresh test data (safe to re-run; 409 = already exists)
cd tools/seed-api-test-data && npm run seed

# During the day: watch one service
docker compose logs -f ms-attendance

# At the end
git add [specific files]
git commit -m "feat(scope): description"
docker compose down               # stops everything, KEEPS data volumes
```

---

## Common issues

### "Port already in use" / `port is already allocated`

```bash
lsof -i :8000                      # macOS/Linux
netstat -ano | findstr :8000       # Windows
kill -9 [PID]                      # macOS/Linux
taskkill /PID [PID] /F             # Windows
```
Classic clash: using **8082** for the frontend — it belongs to `ms-authorization`;
the frontend goes on **8090** (`FRONTEND_PORT`).

### "Docker containers not starting"

```bash
docker compose logs [service-name]     # why did it fail?
docker compose ps                       # which ones are unhealthy?
```
- Kong stuck `starting`: it waits for **all 8 ms-\* healthy**; find the failing
  upstream with `docker compose logs ms-X`.
- JVM services OOM/timeouts: increase Docker RAM to ≥ 8 GB.
- Nuclear option (**deletes all data**): `docker compose down -v && docker compose up -d --build`.
  Prefer plain `docker compose down` (keeps `postgres_data`, `mongo_data`, `redis_data`, `kafka_data`).

### "Database connection refused"

```bash
docker compose ps | grep postgres
docker compose logs postgres
```
Remember services talk to postgres over the `fae-data` network by name
(`postgres:5432`), not `localhost`. If you run a service **natively**, it connects
via `localhost:5432` using the root `.env` — make sure `database/` is up
(`cd database && docker compose up -d`).

### "Migration failed"

```bash
docker compose logs identity-liquibase    # replace with the failing context
# Fix the changelog under database/0X-ms-*-db/, then:
docker compose up -d --build              # chain re-runs; Liquibase is idempotent
```
To start clean (drops all data): `docker compose down -v && docker compose up -d --build`.
Never hand-edit schemas: every DDL change goes through a new Liquibase changeset
(see `database/CONVENCIONES.md`).

### Authentication errors / "session invalid"

There is **no JWT_SECRET to set**. Sessions are created by `ms-identity` in Redis
and validated by each service through Kong/authorization. Checklists:
1. Is `ms-identity` healthy? `curl -s http://localhost:8081/actuator/health`
2. Did login hit the gateway (`:8000/api/v1/auth/login`) and did you keep the returned session token/cookie?
3. Redis up? `docker compose ps | grep redis`
4. Seeded users exist? run `tools/seed-api-test-data` (`SEED_USERNAME`/`SEED_PASSWORD` in `.env`).

### `502 Bad Gateway` from Kong

The upstream microservice isn't healthy yet: `docker compose ps` + `docker compose logs ms-X`.

### Browser can't reach the API / phone shows nothing

`EXPO_PUBLIC_API_URL` is baked at **build** time. For physical devices use the PC
LAN IP (`http://192.168.1.10:8000`), then rebuild:
`docker compose up -d --build frontend-web`. Add that origin to Kong CORS in
`back-end/99-api-gateway/kong/kong.yml` (CORS origins are `["*"]` — any origin is accepted).

### Duplicate/stale networks (`back-end_*`, `full_*`)

Abandoned compose projects. `docker compose down` in each folder, then
`docker network rm <name>` (volumes are untouched).

---

## Utility commands

```bash
# Whole stack (from repo root)
docker compose up -d --build            # start/rebuild everything
docker compose ps                       # status + health
docker compose logs -f [service]        # follow logs
docker compose down                     # stop, keep data
docker compose config                   # validate the compose model (exit 1 = error)
docker compose config --services        # list services

# Sub-layers
cd database && docker compose up -d                    # data + migrations only
cd back-end/99-api-gateway && docker compose up -d     # restart Kong alone (stack already up)
cd database/0X-ms-*-db && docker compose up liquibase  # migrate one context

# Tests (per service directory)
./mvnw test                              # Java: 01, 02, 04, 05
npm test                                 # TS: 03, 07, 08, 09 (jest; 03 has supertest/testcontainers)
poetry run pytest                        # Python: 06
cd tools/seed-api-test-data && npm run seed            # E2E smoke across all services

# Database access
docker exec -it $(docker ps -qf name=postgres) psql -U postgres -d faceattend_db   # 8 schemas
docker exec -it fae-mongo mongosh -u mongoadmin -p mongopass faceattend_biometric
docker exec -it fae-redis redis-cli
docker exec fae-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list

# Frontends
cd front-end/Web    && npx expo start --web        # dev server on :3000 (Kong CORS allows it)
cd front-end/Web    && npm run lint                # eslint.config.js
cd front-end/Mobile && npx expo start              # use PC LAN IP for EXPO_PUBLIC_API_URL

# Kong admin (loopback only)
curl -s http://localhost:8001/services | head      # declarative: edit kong.yml, then restart kong
```

---

## Useful local URLs

| Service | URL | Credentials (local only) |
|---------|-----|--------------------------|
| API Gateway (Kong proxy) | http://localhost:8000 | session-based login via `/api/v1/auth/login` |
| Web SPA (nginx in Docker) | http://localhost:8090 | seeded users (`SEED_USERNAME` / `SEED_PASSWORD`) |
| Web dev server (Expo/Metro) | http://localhost:3000 | — |
| Kong Admin API | http://localhost:8001 | DB-less; edit `kong.yml`, don't POST config |
| Mailpit UI (if started) | http://localhost:8025 | none |
| Per-service health | http://localhost:808X/actuator/health (Java) · http://localhost:808X/health (TS/Py) | — |

> Swagger/OpenAPI UI is **not exposed through Kong** today; each service's
> contract is documented in its `SERVICE.md` (e.g. `back-end/01-ms-identity/SERVICE.md`).

---

## Correlations

- Full infrastructure reference (files, networks, startup chain, known issues) → `COMPOSE.md` (repo root)
- Environment variables per environment → `10-devops/environments.md`
- Microservice map (stacks, ports, responsibilities) → `back-end/SERVICES.md`
- Per-service detail → `back-end/0X-ms-*/SERVICE.md` + `STACK.md` + `DATA_MODEL.md`
- Database design, conventions & seeds → `database/ESTRUCTURA.md`, `database/CONVENCIONES.md`, `database/SEEDS.md`, `back-end/DATABASE.md`
- Gateway routes/CORS/rate-limit → `back-end/99-api-gateway/kong/kong.yml`
- Test data seeding → `tools/seed-api-test-data/README.md`
- Quality/testing reports (IEEE 829, ISTQB, ISO 25010) → `back-end/QUALITY.md`, `back-end/IEEE829_TEST_PLAN.md`, `back-end/ISTQB_QUALITY_REPORT.md`, `back-end/ISO25010_QUALITY_REPORT.md`
- Production troubleshooting → per-service `SERVICE.md` (§ runbook sections); Kubernetes-era runbooks planned
