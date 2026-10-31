# Incident Management — FaceAttend-Edu

> **Current state (verified 2026-10-17):** detection today is **manual + healthchecks** —
> `docker compose ps` / Portainer on the EC2 host, `docker compose logs -f [service]`, Kong
> access logs, and user reports. **Prometheus alerting and PagerDuty are NOT deployed**
> (see `observability.md` and ADR-012), so the automated-detection path below is the target.
> The playbook itself (severity, roles, diagnose/mitigate steps) applies as written.

> An incident is any unplanned event that interrupts or degrades service. In a biometric system like FaceAttend-Edu, incidents can range from API timeouts to hardware failures (biometric readers) or database inconsistencies.
> Effective incident response is trained before incidents occur, not during. Follow this playbook in order — in an incident, stress blocks free thinking.

---

## Severity Classification

We classify incidents based on the impact on the academic process (attendance recording) and the number of affected users.

| Level | Name | Definition | FaceAttend-Edu Example | Response time |
|-------|------|------------|---------------------|---------------|
| **P0** | Critical | System completely down. Total impact to all users. | Kong Gateway down, PostgreSQL inaccessible, or `ms-attendance` failing entirely. | **Immediate** (<5 min) |
| **P1** | High | Core functionality degraded. Impact to > 50% of users. | `ms-biometric` failing to match faces (biometric failure), Identity service unable to issue tokens. | **15 minutes** |
| **P2** | Medium | Secondary functionality degraded. Workaround available. | `ms-notification` alerts not sending, `ms-quality` reports failing, or slow Academic queries. | **1 hour** |
| **P3** | Low | Minor nuisance. No impact on business operations. | UI glitches in the dashboard, typos in reports. | **Next sprint** |

---

## Roles during an Incident

| Role | Responsibility | When activated |
|------|----------------|----------------|
| **Incident Commander (IC)** | Coordinates response, manages the "War Room", and makes final decisions on mitigation. | P0 and P1 always |
| **Tech Lead** | Leads the technical diagnosis across the polyglot stack (Java, TS, Python). | P0 and P1 always |
| **On-Call Engineer** | First responder. Executes technical actions (restarts, rollbacks, config changes). | All incidents |
| **Comms Lead** | Communicates updates to institution administrators and users. | P0 and P1 |

---

## Response Playbook

### Step 1: DETECT (0-5 min)
**How do we know there is an incident in FaceAttend-Edu?**
- **Automated:** Prometheus alerts $\rightarrow$ PagerDuty (e.g., `HighErrorRate` in `ms-attendance`).
- **Manual:** Reports from instructors that the "Take Attendance" button is not working.
- **Infrastructure:** Docker container healthcheck failure (seen in `docker ps`).

**Immediate Action:**
1. Acknowledge the alert to stop the notification loop.
2. Notify the team in Slack `#incidents`: *"Investigating P[N] - [Brief Description] - [Your Name]"*.
3. Check the general health of the three networks (`fae-edge`, `app`, `data`).

### Step 2: CLASSIFY (5-10 min)
Determine severity:
- Is the biometric recognition failing? $\rightarrow$ **P1**
- Is the entire API unreachable via Kong? $\rightarrow$ **P0**
- Is only the Notification service down? $\rightarrow$ **P2**

**If P0 or P1:**
- Activate the Incident Commander and Tech Lead.
- Create an incident ticket (e.g., `INC-XXXX`).
- Open a War Room (Slack channel or Zoom call).

### Step 3: COMMUNICATE (10-15 min)
Initial update template for stakeholders:
> 🔴 **INCIDENT P[N] — [INC-XXXX]**
> **Affected Service:** [e.g., ms-biometric]
> **Impact:** Students cannot record attendance via facial recognition.
> **Status:** Investigating.
> **Next update:** in 30 minutes.
> **IC:** [Name]

### Step 4: DIAGNOSE (10-30 min)
**Diagnostic path for FaceAttend-Edu:**
1. **Kong Gateway:** Check `kong.yml` and logs. Is the request reaching the microservice?
2. **Microservice Logs:** Check the specific runtime log (Spring Boot, Fastify or FastAPI).
3. **Network Connectivity:** Verify if the service can reach the data tier (`fae-data` network).
4. **Biometric Hardware:** If the issue is only with fingerprints, check if the *DigitalPersona Runtime* is installed on the client machines (see `COMPOSE.md` §9).
5. **Database State:** Check for locked rows in PostgreSQL or high CPU in MongoDB.

### Step 5: MITIGATE (15-60 min)
**Mitigation Options (Safe $\rightarrow$ Risky):**
1. **Container Restart:** `docker compose restart [service]` (fixes transient memory leaks or deadlocks).
2. **Rollback:** Revert to a previous stable image version.
3. **Scaling:** Increase resources if the JVM services (`ms-identity`, `ms-attendance`) are hitting RAM limits.
4. **Bypass Biometrics:** Temporarily allow manual attendance (if business logic allows) to avoid blocking the academic process.
5. **Maintenance Mode:** Activate via Kong Gateway to show a "System Under Maintenance" page.

### Step 6: RESOLVE AND COMMUNICATE
The incident is resolved when:
- All `/health` endpoints return `200 OK`.
- Biometric matching latency returns to normal (P95 < 2s).
- Users confirm they can record attendance.

**Resolution Message:**
> ✅ **RESOLVED — [INC-XXXX]**
> **Root cause:** [e.g., MongoDB connection pool exhausted in ms-biometric]
> **Resolution:** [e.g., Increased maxPoolSize and restarted service]
> **Total duration:** [X] minutes.
> **Post-mortem:** [Date]

---

## Post-Mortem (Incident Retrospective)

Held within 2 business days after resolution. Focus on **Why**, not **Who**.

**Structure:**
- **Timeline:** From first alert to resolution.
- **Root Cause Analysis:** The "5 Whys" technique.
- **Contributing Factors:** (e.g., "Lack of connection pool monitoring in the Quality dashboard").
- **Action Items:** Concrete tasks to prevent recurrence (e.g., "Add MongoDB pool metrics to Grafana").

---

## Communication Channels

| Channel | Purpose |
|---------|---------|
| Slack `#incidents` | General alerts and awareness. |
| Slack `#war-room-[INC-XXXX]` | High-intensity technical coordination for P0/P1. |
| PagerDuty / Email | On-call notification. |
| Institution Admin Portal | Status updates for academic coordinators. |

---

## Correlations

- **Detection** $\rightarrow$ `13-operations/observability.md`
- **Recovery** $\rightarrow$ `13-operations/README.md` (Backup & Restore scripts)
- **Service Mapping** $\rightarrow$ `back-end/SERVICES.md`
- **Hardware Requirements** $\rightarrow$ `back-end/COMPOSE.md` (DigitalPersona Runtime)
