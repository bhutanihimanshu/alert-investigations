# 4XXPercentagePerAPI — crm-custom-menus-api — 2026-03-30

**Author:** Himanshu Bhutani | **Status:** Resolved (manual, by Ganesh)

## Summary

| Field | Value |
|-------|-------|
| Alert | 4XXPercentagePerAPI (#114257) |
| Service | crm-custom-menus-api |
| Route | DELETE /custom-menus/:customMenuId |
| Fired | 15:04 IST (09:34 UTC), 2026-03-30 |
| Duration | Auto-grouped (2 alerts), resolved manually |
| Impact | No user impact — client-side auth failure from one integration caller |

## Root Cause

An integration client using the `Lucee (CFML Engine)` user agent is sending DELETE requests to `/custom-menus/54e08fb9-ebbd-409d-942b-4d2b06c6937d` with **expired JWT tokens** at ~1 request/minute. All 60 DELETE failures in the investigation window were **HTTP 401** (Unauthorized). The endpoint's low traffic volume (76 total DELETEs/hour, 79% failure rate) triggers the 4XX percentage threshold. The server is functioning correctly — rejecting invalid credentials as expected.

This alert fired as part of a **fleet-wide 4XX burst** (61 distinct 4XXPercentagePerAPI alerts in #alerts-crm on the same day), indicating systemic alerting sensitivity on low-traffic endpoints rather than a real incident.

## Proof

<details>
<summary>[Grafana] API traffic at ~20 req/s total, DELETE 4XX clearly visible in hits/second chart</summary>

> **Verify:** In the "API Hits/second" chart, look for the `DELETE /custom-menus/customMenuId - 4XX` series (red/orange) running at a steady ~1 req/min throughout the window. Total service RPM is ~15-25 — this is a low-traffic service.

![API Requests Overview](screenshots/001-api-requests-overview-crm-custom-menus-api-traffic-and-error.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-datasource=ber8nnhvgsjy8f&var-container=crm-custom-menus-api&from=1774859400000&to=1774866600000)
</details>

<details>
<summary>[GCP Logs] All 60 DELETE 4XX errors are HTTP 401 — same customMenuId, ~1/min cadence</summary>

> **Verify:** All rows show status `401`, method `DELETE`, targeting the same UUID `54e08fb9-ebbd-409d-942b-4d2b06c6937d`. Timestamps are spaced ~1 minute apart — a systematic retry pattern from a single client.

![401 Errors on DELETE](screenshots/001-gcp-401-errors-on-delete-custom-menus-all-from-expired-jwt.png)

```
resource.type="k8s_container"
resource.labels.container_name="crm-custom-menus-api"
httpRequest.requestMethod="DELETE"
httpRequest.status>=400 AND httpRequest.status<500
```

[Open in Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-custom-menus-api%22%0AhttpRequest.requestMethod%3D%22DELETE%22%0AhttpRequest.status%3E%3D400%20AND%20httpRequest.status%3C500;timeRange=2026-03-30T09%3A00%3A00Z%2F2026-03-30T10%3A00%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[GCP Logs] 83 "JWT decode error" warnings from IAM — confirms expired/invalid token</summary>

> **Verify:** All entries show `IAM_CONFIG_PACKAGE_WARN: JWT decode error` at matching timestamps. The 83 count (vs 60 DELETE 401s) means some JWT errors also affect non-DELETE requests from the same client.

![JWT Decode Errors](screenshots/002-gcp-iam-config-package-warn-jwt-decode-error-matches-401-timesta.png)

```
resource.type="k8s_container"
resource.labels.container_name="crm-custom-menus-api"
severity>=WARNING
jsonPayload.message=~"JWT"
```

[Open in Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-custom-menus-api%22%0Aseverity%3E%3DWARNING%0AjsonPayload.message%3D~%22JWT%22;timeRange=2026-03-30T09%3A00%3A00Z%2F2026-03-30T10%3A00%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[Grafana] No resource pressure — CPU 0.06-0.26 cores, memory 0.28-0.44 GiB, zero restarts</summary>

> **Verify:** CPU and memory traces are flat and well within limits. No pod restarts. This confirms the 4XX errors are client-side, not caused by server health issues.

![App Detailed View](screenshots/002-app-detailed-view-crm-custom-menus-api-resource-usage.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-datasource=ber8nnhvgsjy8f&var-container=crm-custom-menus-api&from=1774859400000&to=1774866600000)
</details>

<details>
<summary>[Slack] Fleet-wide pattern — 61 distinct 4XXPercentagePerAPI alerts in #alerts-crm on same day</summary>

> **Verify:** This was not an isolated event. The Alert Correlator found ~42 different services firing 4XXPercentagePerAPI between 09:31-09:42 UTC, and 61 total distinct alert groups for the day. Low-traffic endpoints are systematically triggering the threshold.

No screenshot — Slack search data from Alert Correlator subagent.
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| Low | Contact the integration owner using Lucee/CFML to rotate their JWT tokens for the custom-menus DELETE endpoint | CRM team / integration support |
| Medium | Add a minimum request volume gate to the 4XXPercentagePerAPI alert rule (e.g., require ≥10 req/min before evaluating %) to eliminate low-N false positives | Platform / Observability team |
| Low | Consider excluding 401 (auth failures) from the 4XX percentage alert — these are expected client errors, not service degradation | Platform / Observability team |

## Links

- [Verbose report](report-verbose.md)
- [Grafana — API Requests Overview](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-datasource=ber8nnhvgsjy8f&var-container=crm-custom-menus-api&from=1774859400000&to=1774866600000)
- [GCP Log Explorer — 401 errors](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-custom-menus-api%22%0AhttpRequest.requestMethod%3D%22DELETE%22%0AhttpRequest.status%3E%3D400%20AND%20httpRequest.status%3C500;timeRange=2026-03-30T09%3A00%3A00Z%2F2026-03-30T10%3A00%3A00Z?project=highlevel-backend)
