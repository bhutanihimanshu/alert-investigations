# 4XX Error Rate — conversations-ai-api — 2026-03-30

**Author:** Himanshu Bhutani | **Status:** Auto-resolved (12 min)

## Summary

| Field | Value |
|-------|-------|
| Alert | #114288 4XXPercentagePerAPI |
| Service | conversations-ai-api |
| Route | POST /conversations-ai/update-snapshot-mappings |
| Fired | 15:09:51 IST (09:39:51 UTC) |
| Resolved | 15:21:59 IST (09:51:59 UTC) |
| Duration | ~12 minutes |
| Impact | No user impact — 422 DTO validation rejections on an internal endpoint, part of a platform-wide 4XX alert storm affecting 20+ services |

## Root Cause

**Platform-wide 4XX alert storm + chronic DTO validation issue.** This alert fired as part of a massive 4XX spike affecting 20+ services across 8 Grafana OnCall integrations (conversations-ai, marketplace, users-internal, integrations, opportunities, bulk-actions, marketplace-modules, cdp). The `conversations-ai-api` was not even in the top 50 containers by 4XX volume (~1 req/s vs 346 req/s for funnel-preview-cache). The 4XX responses on `/update-snapshot-mappings` are exclusively HTTP 422 (NestJS ValidationPipe rejections) from an internal caller sending invalid payloads — this is a steady-state issue (~1-5 req/min), not a new incident.

## What Happened

1. **~14:37 IST** — Platform-wide 4XX rate began rising across 20+ services simultaneously (no single deployment trigger identified).
2. **15:05-15:09 IST** — 4XX rate on `/update-snapshot-mappings` spiked from baseline ~0.08 to ~1.1 req/s, crossing the 1% threshold (since 2XX traffic on this route is only ~0.01 req/s, even a small 4XX increase yields a high percentage).
3. **15:09:51 IST** — Alert fired. Current value: 75% (i.e., 75% of traffic to this route was 4XX).
4. **15:21:59 IST** — Alert auto-resolved. Resolved by Ganesh (bulk resolution during the storm).

## Proof

<details>
<summary>[Grafana] API Requests Overview — 4XX spike visible on update-snapshot-mappings at 15:05 IST</summary>

> **Verify:** In the "API Hits/second" chart, the yellow line (4XX) spikes to ~3 req/s around 15:05 IST while the green line (2XX) stays near zero. The RPM chart shows overall traffic is stable (~200-300 RPM).

![API Requests Overview](screenshots/api-requests-overview.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=conversations-ai-api&var-route=/conversations-ai/update-snapshot-mappings&var-method=POST&from=1774861200000&to=1774865700000)
</details>

<details>
<summary>[Grafana] App Detailed View — 0.0258% error rate, no pod restarts</summary>

> **Verify:** The "ERROR %age" stat shows 0.0258% — negligible. All dashboard rows (App Level Stats, ISTIO STATS, API Stats) are collapsed, indicating no anomalies flagged.

![App Detailed View](screenshots/app-detailed-view.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=conversations-ai-api&from=1774861200000&to=1774865700000)
</details>

<details>
<summary>[GCP Logs] 491 HTTP 422 responses — all from NestJS ValidationPipe, 0.002s latency</summary>

> **Verify:** All rows show status 422, method POST, route /conversations-ai/update-snapshot-mappings. The timeline histogram shows them spread across the window (steady-state, not correlated with the alert time).

![GCP 422 Responses](screenshots/gcp-422-responses.png)

```
resource.type="k8s_container"
resource.labels.container_name="conversations-ai-api"
httpRequest.requestUrl=~"update-snapshot-mappings"
httpRequest.status=422
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-ai-api%22%0AhttpRequest.requestUrl%3D~%22update-snapshot-mappings%22%0AhttpRequest.status%3D422;timeRange=2026-03-30T09%3A00%3A00Z%2F2026-03-30T10%3A15%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[Grafana] Platform-wide: 20+ services with 4XX spikes at the same time</summary>

> **Verify:** Top containers by 4XX rate at 09:39 UTC: funnel-preview-cache (346 req/s), funnels-page-data-api (321 req/s), location-contacts-api (185 req/s), oauth-api (100 req/s), companies-api (93 req/s), and 15+ more. conversations-ai-api is not in the top 50.

| Container | 4XX Rate (req/s) |
|---|---|
| funnel-preview-cache | 346.48 |
| funnels-page-data-api | 321.85 |
| location-contacts-api | 185.53 |
| oauth-api | 100.94 |
| companies-api | 93.86 |
| billing-config-api | 80.32 |
| + 14 more containers | 22–71 req/s each |

conversations-ai-api: ~1.1 req/s (not in top 50).
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| Low | Fix chronic 422: an internal caller (axios/1.13.2 from 127.0.0.6) sends payloads that fail NestJS ValidationPipe on `/update-snapshot-mappings`. Investigate the caller and fix the DTO mismatch. | CRM AI team |
| Low | Tune alert threshold: this route has ~0.01 req/s of 2XX traffic, so even 1 extra 4XX/s causes 75%+ 4XX rate. Consider absolute count thresholds for low-traffic endpoints. | Platform / CRM AI |
| Info | Separate finding: MongoDB `operation exceeded time limit` spike at 14:55 IST (09:25 UTC) in `getActiveWorfklowLocation` — 853 errors in 5 min vs baseline ~70. Not related to this alert but worth investigating. | CRM AI team |

## Links

- [Verbose report](report-verbose.md)
- [API Requests Overview (Grafana)](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=conversations-ai-api&var-route=/conversations-ai/update-snapshot-mappings&var-method=POST&from=1774861200000&to=1774865700000)
- [App Detailed View (Grafana)](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=conversations-ai-api&from=1774861200000&to=1774865700000)
- [422 Logs (GCP)](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-ai-api%22%0AhttpRequest.requestUrl%3D~%22update-snapshot-mappings%22%0AhttpRequest.status%3D422;timeRange=2026-03-30T09%3A00%3A00Z%2F2026-03-30T10%3A15%3A00Z?project=highlevel-backend)
- [Slack alert](https://gohighlevel.slack.com/archives/C0315RRNH1B/p1774863591434289)
