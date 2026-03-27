# HTTPLatencyPerApp Investigation — ai-employees-initiate-api — 2026-03-27

**Author:** Himanshu Bhutani
**Generated:** 2026-03-27 10:20 IST

---

## 1. Alert Summary

| Field | Value |
|-------|-------|
| Alert type | HTTPLatencyPerApp |
| Grafana OnCall | [#113795](https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I7VD5DCC26I3R) |
| Workload | `ai-employees-initiate-api` |
| Endpoint | `POST /ai-employees/text/initiate` |
| Cluster | `servers-us-central-production-cluster` |
| Time | 06:22:57 IST (00:52:57 UTC) — 2026-03-27 |
| Threshold | 10s |
| Current value | 17.21s (P95) |
| Source channel | `#alerts-crm` |
| Slack ts | `1774572777.889629` |

---

## 2. What Happened

1. **06:22 IST** — HTTPLatencyPerApp alert fired: P95 latency for `ai-employees-initiate-api` exceeded 10s threshold at 17.21s.
2. **Investigation** — Latency is stable at P99 ~23s, P95 ~16s throughout the day (and week). No spike — this is the baseline.
3. **Root cause** — The endpoint `POST /ai-employees/text/initiate` calls an upstream LLM for AI text generation. ~29% of requests inherently take >10s due to model response time. The 10s threshold is too aggressive.
4. **Resolution** — Alert needs threshold adjustment. No infrastructure action required.

<details>
<summary>Detailed timeline — full investigation log</summary>

| Time (IST) | Source | Event |
|---|---|---|
| 06:22:57 | Grafana OnCall | HTTPLatencyPerApp fired — ai-employees-initiate-api at 17.21s (threshold 10s) |
| 06:22:57 | Slack #alerts-crm | Alert posted, Grafana OnCall escalation bots triggered |
| — | Investigation | No correlated alerts in any channel (±15 min) |
| — | Investigation | No deployment within 2h (last deploy: 2026-03-26 ~09:37 UTC, 15h before) |
| — | Investigation | Recurring pattern: 6+ alerts over 5 weeks (Feb 17, Feb 18, Mar 2, Mar 16, Mar 26, Mar 27) |
| — | Grafana | P99 latency: avg 23.07s, peak 25.62s. P95: avg 15.77s, peak 17.20s. Stable throughout. |
| — | Grafana | 1 week ago: P99 was worse — avg 27.41s, peak 38.90s. Current is actually better. |
| — | Grafana | Traffic: 11.3 req/s, stable (+10% from 1 week ago, no spike) |
| — | Grafana | Zero 5XX errors. All requests returning 200 or 4XX. |
| — | Grafana | CPU: 0.055 cores avg / 1.10 core limit (5% utilization) |
| — | Grafana | Memory: ~510MB avg / 3,942MB limit (13% utilization) |
| — | Grafana | Event loop: P99 9.7ms, max 1.26s on one pod (isolated). Healthy. |
| — | Grafana | Pod count: 100 (stable HPA). Zero restarts. Zero terminations. |
| — | GCP Logs | ERROR logs: operational only (AI suggestions, attachment processing). No DEADLINE_EXCEEDED, no timeouts. |

</details>

---

## 3. Investigation Findings

### 3a. Latency Pattern — Sustained, Not a Spike

The alert threshold (10s) is well below the normal operating latency for this endpoint.

| Metric | Current Window | 1 Week Ago | Assessment |
|--------|---------------|------------|------------|
| P99 Latency | avg 23.07s, peak 25.62s | avg 27.41s, peak 38.90s | Current is **better** |
| P95 Latency | avg 15.77s, peak 17.20s | avg 17.01s, peak 22.04s | Current is **better** |
| P50 Latency | avg 7.39s, peak 7.97s | — | Half of requests complete <8s |
| % Requests >10s | ~29% | — | Nearly 1 in 3 requests exceeds threshold |

**Latency distribution breakdown:**

| Bucket | Cumulative % | Meaning |
|--------|-------------|---------|
| <= 1s | 22.4% | Fast responses (cached/early-exit) |
| <= 5s | 32.0% | ~32% complete within 5s |
| <= 10s | 71.2% | **29% of requests exceed the 10s threshold** |
| <= 15s | 93.3% | Most complete by 15s |
| <= 20s | 97.8% | Tail: ~2% take 20-30s |
| <= 30s | 99.4% | |
| > 30s | 0.6% | Small fraction of very long requests |

<details>
<summary>Evidence: Grafana — API Requests Overview (latency + traffic + errors)</summary>

> **What to look for:** In the RPM chart, traffic is stable at ~10-12 req/min with no spike. In the API Hits/second chart, green (2XX) dominates with minimal 4XX and zero 5XX. In the Average API Latency chart, latency is flat at ~4-8s average — this is the mean; the P95/P99 are higher due to the long tail of LLM responses.

![API Requests Overview](screenshots/001-api-requests-overview-latency-traffic-for-ai-employees-initi.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=ai-employees-initiate-api&from=1774570800000&to=1774574700000)
</details>

### 3b. Resource Utilization — Heavily Underutilized

| Resource | Per Pod | Limit | Utilization |
|----------|---------|-------|-------------|
| CPU | 0.055 cores | 1.10 cores | **5%** |
| Memory | ~510MB | 3,942MB | **13%** |
| Pod count | 100 | 100 (HPA) | Stable |
| Pod restarts | 0 | — | Healthy |

<details>
<summary>Evidence: Grafana — CPU by Pod</summary>

> **What to look for:** All 100 pod lines are flat near zero at the bottom of the chart. The dashed line at 1 core is the CPU limit. No pod approaches even 20% of the limit. This rules out CPU saturation as a latency cause.

![CPU by Pod](screenshots/002-app-detailed-view-cpu-by-pod.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=ai-employees-initiate-api&from=1774570800000&to=1774574700000&viewPanel=16)
</details>

<details>
<summary>Evidence: Grafana — Memory by Pod</summary>

> **What to look for:** All pod lines are flat at ~512 MiB. The dashed line at ~3.5 GiB is the memory limit. There is massive headroom — memory is not a factor.

![Memory by Pod](screenshots/003-app-detailed-view-memory-by-pod.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=ai-employees-initiate-api&from=1774570800000&to=1774574700000&viewPanel=30)
</details>

### 3c. Event Loop — Healthy

<details>
<summary>Evidence: Grafana — NodeJS Event Loop Dashboard</summary>

> **What to look for:** The Event Loop Lag chart shows periodic spikes with mean 55.9ms, max 498ms, P90 56.8ms. This is healthy for a NodeJS application under load. Active Handlers stable at ~94, Active Requests near 0. No sustained event loop blocking that would cause latency.

![NodeJS Dashboard](screenshots/004-nodejs-application-dashboard-event-loop-lag.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/PTSqcpJWka/nodejs-application-dashboard?orgId=1&var-app=ai-employees-initiate-api&from=1774570800000&to=1774574700000)
</details>

### 3d. GCP Log Evidence — No Infrastructure Errors

ERROR logs during the investigation window contain only operational/business-logic errors:

| Error Pattern | Count | Type |
|---------------|-------|------|
| `Error in handling AI suggetsions` | Multiple | AI model processing failure |
| `Attachment processing triggered` | Multiple | Whitelabel URL conversion error |
| `Error in sending for evaluate` | Multiple | Evaluation pipeline error |
| `winstonError: Log entry exceeds 256K` | Sparse | Oversized log entries |
| `Failed to convert whitelabeled URL` | Multiple | URL format validation |

**Critically absent:** No `DEADLINE_EXCEEDED`, no `ETIMEDOUT`, no Redis connection errors, no Firestore timeouts. The latency is from the upstream LLM, not from infrastructure dependencies.

<details>
<summary>Evidence: GCP Logs — ERROR pattern distribution</summary>

> **Verify:** Run the query below. All errors are application-level AI/attachment processing — no infrastructure failure patterns.

```
resource.type="k8s_container"
resource.labels.container_name="ai-employees-initiate-api"
severity>=ERROR
timestamp>="2026-03-27T00:22:00Z"
timestamp<="2026-03-27T01:23:00Z"
```

**Timeout-specific query (returned 0 relevant results):**
```
resource.type="k8s_container"
resource.labels.container_name="ai-employees-initiate-api"
(jsonPayload.message=~"DEADLINE_EXCEEDED" OR jsonPayload.message=~"ETIMEDOUT")
timestamp>="2026-03-27T00:22:00Z"
timestamp<="2026-03-27T01:23:00Z"
```
Result: 0 entries. Only 4 metric-registration warnings matched the "timeout" keyword — benign `PLATFORM_CORE_OBSERVABILITY` histogram name conflicts.
</details>

### 3e. Correlated Alerts — None

| Channel | Alerts in ±15 min window |
|---------|--------------------------|
| `#alerts-crm` | Only this alert |
| `#alerts-crm-conversations` | 0 |
| `#alerts-platform` | 0 |
| `#alerts-database` | 0 |

This is an isolated alert — no correlated infrastructure event.

### 3f. Deployment Check — No Recent Deploy

Last deployment: 2026-03-26 ~09:37 UTC (15 hours before the alert). Three deployment requests were found for ai-employees on March 26, all 15-16 hours before the alert — well outside the typical 10-45 min deployment-triggered pattern.

---

## 4. Cross-Validation

| Signal | Source | Finding | Agrees? |
|--------|--------|---------|---------|
| Latency is sustained baseline | Grafana P99/P95 | Flat at 23s/16s, no spike | Yes |
| Latency was worse 1 week ago | Grafana 7-day comparison | P99 was 27.4s last week | Yes |
| ~29% requests >10s | Grafana histogram | Inherent distribution | Yes |
| Zero errors | Grafana error rate | 0 5XX | Yes |
| No resource pressure | Grafana CPU/Memory | 5% CPU, 13% memory | Yes |
| No event loop blocking | Grafana NodeJS | 9.7ms P99 event loop | Yes |
| No infrastructure errors | GCP logs | Only operational errors | Yes |
| No correlated alerts | Slack (all channels) | 0 other alerts | Yes |
| No recent deployment | Slack + investigation | 15h gap to last deploy | Yes |
| Recurring pattern | Slack search | 6+ alerts over 5 weeks | Yes |

**Confidence: HIGH** — All 10 signals converge. Every infrastructure health metric is green. The latency is inherent to the LLM-backed endpoint.

---

## 5. Root Cause

**Category:** Alert threshold misconfiguration (false positive)

**Statement:** The `ai-employees-initiate-api` service handles `POST /ai-employees/text/initiate` — an AI text generation endpoint that calls an upstream LLM. The P99 latency is inherently ~23s and P95 is ~16s due to LLM response time. The 10s HTTPLatencyPerApp threshold is appropriate for typical API endpoints but not for an LLM-backed text generation endpoint where response times are inherently 5-25s depending on model processing. The service is healthy on every dimension: zero errors, zero restarts, 5% CPU, 13% memory, healthy event loop, no dependency failures.

**Causal chain:**
1. LLM text generation inherently takes 5-25s per request depending on prompt complexity and model load
2. ~29% of requests exceed the 10s alert threshold as part of normal operation
3. The HTTPLatencyPerApp alert fires whenever P95 crosses 10s, which happens regularly
4. This has been recurring 6+ times over 5 weeks with no degradation (current latency is actually better than 1 week ago)

<details>
<summary>Probable noise — operational errors during window (not root cause)</summary>

| Pattern | Count | Why it's noise |
|---------|-------|----------------|
| `Error in handling AI suggetsions` | Multiple | Business logic — AI model processing failure, not infra |
| `Attachment processing triggered` | Multiple | URL format validation failure, not related to latency |
| `Error in sending for evaluate` | Multiple | Evaluation pipeline error, exists at baseline |
| `winstonError: Log entry exceeds 256K` | Sparse | Winston log size limit, not related to latency |
| `PLATFORM_CORE_OBSERVABILITY Metric name conflict` | 4 | Benign metric registration warning |

None of these errors correlate with the latency pattern — they exist at a constant baseline rate.
</details>

---

## 6. Action Items

| Priority | Action | Owner | Reasoning |
|----------|--------|-------|-----------|
| **High** | Raise HTTPLatencyPerApp threshold for `ai-employees-initiate-api` from 10s to **25s** | On-call / Platform | P99 is ~23s. Current threshold causes recurring false positives (~weekly) |
| Medium | Create a dedicated latency SLO tier for AI/LLM endpoints | CRM AI team | AI endpoints have fundamentally different latency profiles than CRUD APIs |
| Medium | Track AI provider response time as a separate metric | CRM AI team | Separate model latency from infra latency for better observability |
| Low | Fix typo "suggetsions" → "suggestions" in error logging | CRM AI team | Spotted in logs: `Error in handling AI suggetsions` |

---

## 7. Deployment Details

| Setting | Value |
|---------|-------|
| CPU request | 1.00 core |
| CPU limit | 1.10 cores |
| Memory request | 3,584MB |
| Memory limit | 3,942MB |
| Pod count | 100 (HPA, stable) |
| Cluster | `servers-us-central-production-cluster` |
| Node.js version | v22.x (from dashboard) |

---

## 8. Links

- [Concise report](http-latency-ai-employees-initiate-api-2026-03-27.md)
- [API Requests Overview — Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=ai-employees-initiate-api&from=1774570800000&to=1774574700000)
- [App Detailed View — Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=ai-employees-initiate-api&from=1774570800000&to=1774574700000)
- [NodeJS Dashboard — Grafana](https://prod.grafana.leadconnectorhq.com/d/PTSqcpJWka/nodejs-application-dashboard?orgId=1&var-app=ai-employees-initiate-api&from=1774570800000&to=1774574700000)
- [Grafana OnCall Alert](https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I7VD5DCC26I3R)
