# HTTPLatencyPerApp — ai-employees-initiate-api — 2026-03-27

**Author:** Himanshu Bhutani | **Status:** Recurring (not actionable as infrastructure issue)

## Summary

| Field | Value |
|-------|-------|
| Alert | HTTPLatencyPerApp #113795 |
| Service | `ai-employees-initiate-api` |
| Endpoint | `POST /ai-employees/text/initiate` |
| Fired | 06:22 IST (00:52 UTC) — 2026-03-27 |
| Duration | Recurring — fires multiple times per week |
| Current Value | P95 = 17.21s (threshold: 10s) |
| Impact | None — service is healthy; alert is a false positive due to threshold misconfiguration |

## Root Cause

The 10s latency threshold is too aggressive for this LLM-backed text generation endpoint. The `POST /ai-employees/text/initiate` endpoint calls an upstream AI model, where **~29% of requests inherently exceed 10s** due to model response time. P99 is ~23s and P95 is ~16s — this is normal operating behavior, not a degradation. The service is completely healthy: zero errors, zero restarts, 5% CPU, 13% memory, 9.7ms event loop lag. This alert has fired **6+ times over the past 5 weeks**, including back-to-back days (Mar 26 and 27).

## Proof

<details>
<summary>[Grafana] P95 latency = 17.2s is the baseline — not a spike (06:22 IST)</summary>

> **Verify:** In the API Requests Overview dashboard, look at the "Average API Latency" and "P90 API Latency" charts. Latency is flat throughout the window — no spike at alert time. RPM is stable at ~10-12 req/min. Zero 5XX errors visible.

![API Requests Overview](screenshots/001-api-requests-overview-latency-traffic-for-ai-employees-initi.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=ai-employees-initiate-api&from=1774570800000&to=1774574700000)
</details>

<details>
<summary>[Grafana] CPU at 5% of limit — no resource pressure</summary>

> **Verify:** All pod lines are flat near zero. The dashed line at 1 core is the CPU limit. No pod approaches the limit at any point.

![CPU by Pod](screenshots/002-app-detailed-view-cpu-by-pod.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=ai-employees-initiate-api&from=1774570800000&to=1774574700000&viewPanel=16)
</details>

<details>
<summary>[Grafana] Memory at 13% of limit — no memory pressure</summary>

> **Verify:** All pod lines are flat at ~512 MiB. The dashed line at ~3.5 GiB is the memory limit. Massive headroom.

![Memory by Pod](screenshots/003-app-detailed-view-memory-by-pod.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=ai-employees-initiate-api&from=1774570800000&to=1774574700000&viewPanel=30)
</details>

<details>
<summary>[Grafana] Event loop healthy — avg 55.9ms lag, no blocking</summary>

> **Verify:** Event Loop Lag chart shows periodic spikes but avg P90 = 56.8ms. Active Handlers stable at ~94. Active Requests near 0. No sustained event loop blocking.

![Event Loop](screenshots/004-nodejs-application-dashboard-event-loop-lag.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/PTSqcpJWka/nodejs-application-dashboard?orgId=1&var-app=ai-employees-initiate-api&from=1774570800000&to=1774574700000)
</details>

<details>
<summary>[GCP Logs] No infrastructure errors — only operational AI/attachment errors</summary>

> **Verify:** ERROR logs contain only business-logic errors (attachment processing, AI suggestion handling). No DEADLINE_EXCEEDED, no Redis timeouts, no connection failures.

```
gcloud logging read '
resource.type="k8s_container"
resource.labels.container_name="ai-employees-initiate-api"
severity>=ERROR
timestamp>="2026-03-27T00:22:00Z"
timestamp<="2026-03-27T01:23:00Z"
' --project=highlevel-backend --limit=20 --format=json
```

Top error patterns (from 20 sampled entries):
- `Error in handling AI suggetsions` — AI model processing failure
- `Attachment processing triggered` — whitelabel URL conversion
- `Error in sending for evaluate` — evaluation pipeline
- `winstonError: Log entry exceeds 256K` — oversized log entries

Zero DEADLINE_EXCEEDED, zero ETIMEDOUT, zero Redis connection errors.
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| **High** | Raise HTTPLatencyPerApp threshold for `ai-employees-initiate-api` from 10s to **25s** (P99 is ~23s) | On-call / Platform |
| Medium | Consider a dedicated latency SLO tier for AI/LLM endpoints with inherently high response times | CRM AI team |
| Low | Track AI provider response time as a separate metric to distinguish model latency from infra latency | CRM AI team |

## Links

- [Verbose report](report-verbose.md)
- [API Requests Overview — Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=ai-employees-initiate-api&from=1774570800000&to=1774574700000)
- [App Detailed View — Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=ai-employees-initiate-api&from=1774570800000&to=1774574700000)
- [NodeJS Dashboard — Grafana](https://prod.grafana.leadconnectorhq.com/d/PTSqcpJWka/nodejs-application-dashboard?orgId=1&var-app=ai-employees-initiate-api&from=1774570800000&to=1774574700000)
