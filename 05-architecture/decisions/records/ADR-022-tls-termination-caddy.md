# ADR-022 — TLS termination with Caddy in front of Kong

- **ID:** ADR-022
- **Date:** 2026-10-23
- **Status:** Accepted
- **Decision (2026-10-23):** accepted; implemented in `docker-compose.prod.yml`. Certificate issuance on the EC2 host still needs confirmation.
- **Authors:** FaceAttend EDU Team
- **Implementation note (2026-10-23, verified against `FULL/`):** implemented in the repository, not ratified. Whether the certificate is actually issued on the EC2 host cannot be proven from code; the only external evidence is that `front-end/Mobile/eas.json` points the `preview` profile at `https://faceattend-edu.duckdns.org`.

---

## Context

ADR-020 deployed the backend on a single EC2 host with Kong on plain HTTP `:8000` and the web client on Vercel (HTTPS). That left two problems recorded as open risks in ADR-020:

1. **Mixed content:** a page served from an HTTPS origin cannot call an HTTP API, so the Vercel web client could not reach Kong.
2. **Biometric capture:** `getUserMedia` (camera) requires a secure context, and the session bearer travels in every request.

Kong 3.6 OSS can terminate TLS, but certificate issuance and renewal would be manual. The team has no dedicated DevOps role (ADR-007).

**Known constraints:**
- Ports 80/443 must be open to the public for ACME HTTP-01 and HTTP/3.
- A DNS name must point at the host (a free DuckDNS name is used, not a purchased domain).
- Kong and the services must keep working unchanged inside the Docker network.

---

## Decision

**We decided:** terminate TLS in a **Caddy 2** container (`caddy:2-alpine`) that reverse-proxies to `kong-gateway:8000`, with automatic Let's Encrypt certificates.

**What exists in the code:**

| File | Role |
|------|------|
| `docker-compose.prod.yml` | `caddy` service (ports 80, 443, 443/udp), mounts `Caddyfile.prod`, requires `API_DOMAIN` |
| `Caddyfile.prod` | Single site `{$API_DOMAIN}` → `kong-gateway:8000`; frontend is **not** served here (it lives on Vercel) |
| `docker-compose.tls.yml` + `Caddyfile` | Optional overlay for local/LAN: `DOMAIN` → `frontend-web:80`, `api.DOMAIN` → Kong; without a real domain Caddy uses its internal CA |

**Justification:**
1. Certificates are issued and renewed with zero configuration (two lines of Caddyfile).
2. Kong stays DB-less and unchanged; TLS is an edge concern outside the gateway config.
3. HTTP/3 for free.
4. ADR-005 already scored Caddy (18) as a viable edge proxy; the decision here is the concrete use.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Caddy in front of Kong (chosen)** | Auto-HTTPS, tiny config, one container | One more hop; ACME needs 80/443 public | — (chosen) |
| Kong native TLS (`KONG_SSL_CERT`) | No extra container | Manual issuance/renewal; certificate files to distribute | Operational burden for a 3-person team |
| nginx + certbot | Familiar | Cron/renewal glue, more config | More moving parts than Caddy |
| AWS ALB + ACM | Managed certs, no host ports | Cost, AWS lock-in, outside the free/cheap setup of ADR-020 | Over budget for one institution |
| Vercel rewrite proxying `/api` | No TLS on EC2 | Proxy hop for every call including biometric uploads; breaks mobile (calls Kong directly) | Mobile needs a public HTTPS API anyway |

---

## Consequences

**Positive:**
- Vercel web and the Expo mobile app can call the API over HTTPS; `getUserMedia` works.
- Renewal is automatic (volumes `caddy_data` / `caddy_config` persist the account and certificates).

**Negative / Trade-offs:**
- Ports 8000/8001 of Kong are still published by the base compose; closing them to the public is a separate hardening step (open risk in ADR-020).
- The DuckDNS name is a single point of dependency on a third party.
- `CORS_ORIGIN` remains `["*"]` in Kong; HTTPS does not change that.

**Impact on the system:**
- Affected: `docker-compose.prod.yml`, `Caddyfile.prod`, `docker-compose.tls.yml`, `Caddyfile`, EAS profiles (`EXPO_PUBLIC_API_URL`).
- Documents updated: ADR-020 (risk/trade-off notes), `10-devops/environments.md` (`API_DOMAIN`).

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| ACME fails (DNS not pointing at host, ports closed) | Medium | High | Check `docker logs fae-caddy`; keep 80/443 open; verify DNS before deploy |
| Let's Encrypt rate limits when re-creating volumes | Low | Medium | Never delete `caddy_data` on redeploy |
| Direct HTTP access to Kong `:8000` bypasses TLS | Medium | Medium | Restrict `:8000` in the EC2 security group once clients use `https://` |

---

## References

- `FULL/docker-compose.prod.yml`, `FULL/Caddyfile.prod`, `FULL/docker-compose.tls.yml`, `FULL/Caddyfile`, `FULL/front-end/Mobile/eas.json`
- Related to: ADR-005 (edge proxy scoring), ADR-007 (Kong), ADR-020 (deployment platforms; supersedes its "TLS pending" trade-off in practice), ADR-019
