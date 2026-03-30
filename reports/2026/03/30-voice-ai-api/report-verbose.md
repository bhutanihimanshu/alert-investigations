# 4XXPercentagePerAPI Investigation — voice-ai-api, opportunities-api, oauth-login-api, oauth-token-refresh-api — 2026-03-30

**Author:** Himanshu Bhutani
**Generated:** 2026-03-30 16:30 IST

## Alert Summary

| Field | Value |
|-------|-------|
| Alert type | 4XXPercentagePerAPI (5 alerts) |
| Services | voice-ai-api, opportunities-api (x2), oauth-login-api, oauth-token-refresh-api |
| Alert channels | CRM-conversations-ai, CRM-opportunities, CRM-marketplace |
| Time | 15:01:09–15:01:39 IST (09:31 UTC) |
| Severity | WARNING |
| Status | Auto-resolved (false positive) |

### Alert Messages

| # | Time (IST) | OnCall # | Integration | App |
|---|-----------|----------|-------------|-----|
| 1 | 15:01:09 | #114154 | CRM-conversations-ai | voice-ai-api |
| 2 | 15:01:23 | #114158 | CRM-opportunities | opportunities-api |
| 3 | 15:01:30 | #114172 | CRM-opportunities | opportunities-api |
| 4 | 15:01:38 | #114187 | CRM-marketplace | oauth-login-api |
| 5 | 15:01:39 | #114189 | CRM-marketplace | oauth-token-refresh-api |

---

## Investigation Findings

### Context: Part of a Larger Alert Storm

These 5 alerts are part of a **massive 100+ alert storm** that affected all CRM channels between 14:37–15:55 IST (09:07–10:25 UTC). The storm was so large that Grafana OnCall hit Slack rate limits at 15:20 IST for 5 integrations (CRM-conversations-ai, CRM-opportunities, CRM-users-internal, CRM-marketplace, CRM-integrations).

**Two prior investigations of the same event** concluded:
1. "Multiple independent 4XX sources crossed the threshold simultaneously during elevated traffic. No single infrastructure failure — convergence of application-level errors. **Not actionable as an incident.**"
2. "**False positive** — 4XX rates are within normal baseline driven by business validation errors."

### Evidence: Grafana — 4XX Rate Per Service

<details>
<summary>voice-ai-api — 4XX peaked at 21.13%, dominated by 403 Forbidden (92%)</summary>

> **What to look for:** The response code breakdown shows 403 (Forbidden) as the overwhelming contributor. The 4XX% rose from ~15% baseline to 21% — this is an auth/permission-denial pattern, not a service failure.

**4XX rate timeline (around alert time):**

| Time (IST) | 4XX % | Notes |
|-----------|-------|-------|
| 14:50–15:00 | 15.4–16.7% | Steady baseline |
| **15:01 (alert)** | **16.86%** | Alert threshold crossed |
| 15:06–15:09 | 18.9–21.1% | Peak |
| 15:10 | 20.15% | Declining |

**Status code breakdown:**

| Status | Avg Rate (req/s) | % of 4XX |
|--------|-----------------|----------|
| 403 | 4.39 | 92% |
| 400 | 0.25 | 5% |
| 422 | 0.08 | 2% |
| 401 | 0.017 | <1% |
| 429 | 0.0004 | <1% |

![voice-ai-api API Requests Overview](screenshots/001-api-requests-overview-voice-ai-api-4xx-breakdown.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=voice-ai-api&from=1774859400000&to=1774866600000)
</details>

<details>
<summary>opportunities-api — steady 5.3% baseline, mixed 400/429/422 errors</summary>

> **What to look for:** The 4XX rate shows no change at alert time. The baseline is composed of duplicate opportunity attempts (400), rate limiting (429), and validation errors (422) — all business logic.

**Status code breakdown:**

| Status | Avg Rate (req/s) | Primary Cause |
|--------|-----------------|---------------|
| 400 | 4.03 | Duplicate opportunity creation |
| 429 | 2.34 | Rate limiting |
| 422 | 1.22 | Validation errors |
| 401 | 1.13 | Auth failures |
| 403 | 1.02 | Permission denials |
| 404 | 0.52 | Not found |

![opportunities-api API Requests Overview](screenshots/002-api-requests-overview-opportunities-api-steady-baseline.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=opportunities-api&from=1774859400000&to=1774866600000)
</details>

<details>
<summary>oauth-login-api — 4XX peaked at 08:36 UTC, declining by alert time</summary>

> **What to look for:** The 4XX peak occurred ~55 minutes before the alert. At alert time (15:01 IST), the rate was already declining from 8.46% to 4.92%, composed of failed login attempts (400 Bad Request).

**Timeline:**

| Time (IST) | 4XX % | Notes |
|-----------|-------|-------|
| 14:06 (08:36 UTC) | 8.46% | Peak |
| 15:01 (alert) | 4.92% | Already declining |
| 15:07–15:10 | 2.95–3.51% | Subsiding |

![oauth-login-api API Requests Overview](screenshots/003-api-requests-overview-oauth-login-api-declining-4xx.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=oauth-login-api&from=1774859400000&to=1774866600000)
</details>

<details>
<summary>oauth-token-refresh-api — noise-level 0.74% (401 expired tokens)</summary>

> **What to look for:** The 4XX rate barely registers at 0.6–1.0%. All 401s are expired refresh tokens — normal OAuth lifecycle. This alert should not have fired.

**Status code breakdown:**

| Status | Avg Rate (req/s) | Primary Cause |
|--------|-----------------|---------------|
| 401 | 0.29 | Expired/invalid refresh tokens |
| 400 | 0.056 | Malformed requests |
| 404 | 0.012 | Not found |

![oauth-token-refresh-api API Requests Overview](screenshots/004-api-requests-overview-oauth-token-refresh-api-noise-level-4x.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=oauth-token-refresh-api&from=1774859400000&to=1774866600000)
</details>

### Evidence: GCP Logs — Error Patterns

<details>
<summary>voice-ai-api — CastError from undefined agentId (genuine spike, business logic bug)</summary>

> **What to look for:** All error entries show the same Mongoose CastError: `Cast to ObjectId failed for value "undefined" at path "agentId" for model "voiceAIAgentCalls"`. This is a business logic bug in `CallsService.fetchCallDetails` — callers sending requests without a valid agentId parameter.

**Error volume timeline:**

| Time (IST) | ERROR count/5min | Notes |
|-----------|-----------------|-------|
| 14:00–14:45 | 3–33 | Baseline |
| 14:55 | 203 | Spike start |
| **15:00** | **214** | Alert threshold |
| **15:05** | **304** | Growing |
| **15:10** | **414** | Peak approach |
| **15:15** | **431** | Peak |
| 15:20 | 419 | Sustained |
| 15:30+ | 209–255 | Settling |

**Top error patterns:**

| Count | Pattern | Classification |
|-------|---------|---------------|
| Dominant | `CastError: Cast to ObjectId failed for value "undefined" at path "agentId"` | Business logic bug |
| Minor | `TypeError: Cannot read properties of null (reading 'translation')` | Null check missing |
| Minor | `TypeError: Cannot read properties of null (reading '_id')` — SynthflowCallService | Null data from webhook |
| Minor | `UnprocessableEntityException` (422) | NestJS validation |

```
resource.type="k8s_container"
resource.labels.container_name="voice-ai-api"
severity>=ERROR
jsonPayload.message=~"Cast to ObjectId failed"
timestamp>="2026-03-30T09:20:00Z"
timestamp<="2026-03-30T09:45:00Z"
```

![CastError logs](screenshots/001-gcp-voice-ai-api-casterror-flood-undefined-agentid.png)

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22voice-ai-api%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22Cast%20to%20ObjectId%20failed%22;timeRange=2026-03-30T09%3A00%3A00Z%2F2026-03-30T10%3A00%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>opportunities-api — steady 800–1100 ERROR/5min, all business logic validation</summary>

> **What to look for:** Error volume was flat throughout the window. No spike at alert time. All errors are BadRequestExceptions from expected business validations.

**Top error patterns (all steady, no spikes):**

| Pattern | Classification |
|---------|---------------|
| `BadRequestException: Can not create duplicate opportunity` | Expected — duplicate prevention |
| `BadRequestException: The pipeline id is invalid` | Expected — stale references |
| `BadRequestException: Moving opportunity backward not allowed` | Expected — pipeline constraint |
| `BadRequestException: opportunity contact is deleted` | Expected — stale contact |
| `Invalid Follower` | Expected — invalid userId |

</details>

<details>
<summary>oauth-login-api — noisy 62–295 ERROR/5min, failed login attempts</summary>

> **What to look for:** Error volume was variable with no clear change at alert time. All errors are expected auth failures.

**Top error patterns (all steady):**

| Pattern | Classification |
|---------|---------------|
| `Invalid email or password` | Expected — failed login |
| `User does not exist` | Expected — non-existent user |
| `No token found for otp` | Expected — invalid OTP |
| `Error creating audit log for support agent login` | Side-effect failure |

</details>

<details>
<summary>oauth-token-refresh-api — flat 3.5K–4.8K ERROR/5min, expired refresh tokens</summary>

> **What to look for:** Extremely high but completely flat error rate. All errors are from expired/invalid refresh tokens. This is a steady-state issue, not incident-related.

**Top error patterns (all steady):**

| Pattern | Classification |
|---------|---------------|
| `Error refreshing V2 token` (~99%) | Expected — expired tokens |
| `BadRequestException: Invalid Refresh Token. Please login again!` | Expected — validation |
| `Location id: dashboard is invalid` | Data quality |

</details>

### Evidence: Infrastructure Health

| Check | Result |
|-------|--------|
| Pod restarts | **0** across all 4 services (08:30–10:30 UTC) |
| CPU pressure | None — all services well within limits |
| Memory pressure | None — no approaching-limit patterns |
| Deployments | None within 2 hours before alert |
| Database alerts (#alerts-database) | None |
| Platform alerts (#alerts-platform) | JenkinsNodeOffline only (unrelated) |
| Istio sidecar issues | Readiness probe failures at 10:28 UTC (57 min post-alert, unrelated) |

---

## Cross-Validation

| Signal | Grafana | GCP Logs | Slack |
|--------|---------|----------|-------|
| voice-ai-api 4XX spike | 403s dominate (92%), peak 21% | CastError from undefined agentId | Part of 100+ alert storm |
| opportunities-api steady 4XX | Flat 5.3% baseline | Steady BadRequestExceptions | Resolved by Ganesh |
| oauth-login-api declining 4XX | Peaked at 08:36, declining | Failed logins | Resolved by Ganesh |
| oauth-token-refresh-api noise | 0.74% — negligible | Flat 4K/5min expired tokens | Part of same storm |
| Infrastructure trigger | None (0 restarts, no pressure) | No infra errors | No GCE/Istio/DB alerts |

**Confidence: HIGH** — All three sources (Grafana metrics, GCP logs, Slack context) agree: this is a convergence of independent application-level 4XX errors with no infrastructure trigger. Two prior investigations of the same event reached identical conclusions.

---

## Root Cause

**False positive — convergence of independent application-level 4XX errors.** Four unrelated services with normal-to-elevated 4XX baselines (from business validation errors, auth failures, rate limiting) simultaneously crossed the `4XXPercentagePerAPI` alert threshold during a period of elevated traffic. No single infrastructure failure triggered the burst.

### What Happened

1. **14:37 IST** — First wave of `4XXPercentagePerAPI` alerts fires across CRM-cdp, CRM-marketplace, CRM-opportunities as multiple services' 4XX baselines cross the alert threshold.
2. **14:55 IST** — voice-ai-api CastError spike begins (~15x from baseline), caused by callers sending requests with undefined agentId.
3. **15:01 IST** — Massive second wave: 50+ `4XXPercentagePerAPI` alerts fire in 30 seconds across all CRM channels, including the 5 alerts under investigation. All triggered by independent application-level 4XX sources.
4. **15:20 IST** — Grafana OnCall hits Slack rate limits for 5 integrations due to alert volume.
5. **15:30+ IST** — Alerts auto-resolve as 4XX rates fluctuate below threshold.

<details>
<summary>Detailed timeline — per-service error breakdown</summary>

| Time (IST) | Service | Event |
|-----------|---------|-------|
| 14:00–14:45 | voice-ai-api | Baseline: 3–33 errors/5min |
| 14:37 | Multiple | First wave of 4XX alerts (cdp-core-api, marketplace, opportunities) |
| 14:50 | voice-ai-api | 4XX rate at 15.4% (steady) |
| 14:55 | voice-ai-api | CastError spike begins: 203 errors/5min |
| 15:00 | voice-ai-api | 214 errors/5min, 4XX at 16.86% |
| 15:01:09 | voice-ai-api | **Alert #114154 fires** |
| 15:01:23 | opportunities-api | **Alert #114158 fires** (steady 5.3% baseline) |
| 15:01:30 | opportunities-api | **Alert #114172 fires** (duplicate) |
| 15:01:38 | oauth-login-api | **Alert #114187 fires** (4XX declining from earlier peak) |
| 15:01:39 | oauth-token-refresh-api | **Alert #114189 fires** (0.74% — noise) |
| 15:05 | voice-ai-api | 304 errors/5min |
| 15:09 | voice-ai-api | Peak 4XX at 21.13% |
| 15:15 | voice-ai-api | Peak error volume: 431/5min |
| 15:20 | Grafana OnCall | Slack rate limits hit for 5 integrations |
| 15:30+ | All | Alerts auto-resolve |

</details>

---

<details>
<summary>Probable noise — transient errors during disruption (not root cause)</summary>

| Time (IST) | Pattern | Why it's noise |
|-----------|---------|---------------|
| 15:58 (10:28 UTC) | Istio readiness probe failures on voice-ai-api, opportunities-api pods | 57 minutes post-alert, unrelated istio sidecar event |
| Ongoing | oauth-token-refresh-api ~4K ERROR/5min | Pre-existing steady state — expired refresh tokens |
| Ongoing | opportunities-api ~1K ERROR/5min | Pre-existing steady state — business validation |
| Ongoing | oauth-login-api variable ERROR rate | Pre-existing — failed login attempts |

</details>

---

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| **Low** | Fix voice-ai-api CastError: validate `agentId` parameter before passing to Mongoose query in `CallsService.fetchCallDetails`. Add a guard clause returning 400 if agentId is undefined. | Voice AI team |
| **Low** | Review `4XXPercentagePerAPI` alert threshold. Consider: (1) excluding endpoints with known high 4XX baselines from client errors, (2) raising the threshold for services like oauth-token-refresh-api, (3) differentiating between client-error 4XX (400/401/403/429) and server-error 4XX (408/500-adjacent). | Platform / Alert owners |
| **Info** | oauth-token-refresh-api has a steady ~4K ERROR/5min baseline from expired refresh tokens. These should arguably be logged at WARN level, not ERROR. | OAuth team |
| **Info** | voice-ai-api has secondary null-check bugs: `TypeError: Cannot read properties of null (reading 'translation')` and `(reading '_id')` in `CallsService` and `SynthflowCallService`. | Voice AI team |

---

## Deployment Details

| Service | Total Request Rate | Pod Count | CPU/Memory |
|---------|-------------------|-----------|------------|
| opportunities-api | 161 req/s avg, 205 peak | Multiple | 5.25/7.30 cores, 11.8/14.8 GB |
| oauth-token-refresh-api | 59 req/s avg, 81 peak | Multiple | 3.39/5.14 cores, 20.2/23.2 GB |
| voice-ai-api | 30 req/s avg, 37 peak | Multiple | 1.03/1.59 cores, 3.6/7.9 GB |
| oauth-login-api | 6.9 req/s avg, 7.8 peak | Multiple | 1.66/3.48 cores, 23.6/32.4 GB |

---

## Cross-Validation Summary

| Source | Finding | Confidence |
|--------|---------|------------|
| Grafana (4XX rates) | All 4XX from client errors (400, 401, 403, 429). No 5XX. | HIGH |
| GCP Logs (error patterns) | All errors are business logic (validation, auth, expired tokens). Only voice-ai-api has a real spike. | HIGH |
| Slack (alert context) | 100+ alerts in same storm. Two prior investigations: "not actionable" / "false positive". | HIGH |
| Infrastructure (restarts, CPU, DB) | Zero issues across all checks. | HIGH |
| **Overall** | **False positive convergence. Not an incident.** | **HIGH** |

---

## Links

- [Concise report](4xx-burst-voice-ai-opportunities-oauth-2026-03-30.md)
- [Earlier multi-service investigation (same event)](https://github.com/bhutanihimanshu/alert-investigations/blob/main/reports/2026/03/30-4xx-burst-multi-service/report.md)
- [Earlier multi-service verbose report](https://github.com/bhutanihimanshu/alert-investigations/blob/main/reports/2026/03/30-4xx-burst-multi-service/report-verbose.md)
