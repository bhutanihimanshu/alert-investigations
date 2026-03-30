# 4XXPercentagePerAPI Burst — 20 Alerts Across 12+ Services — 2026-03-30

**Author:** Himanshu Bhutani | **Status:** Resolved (by OnCall operator)

## Summary

| Field | Value |
|-------|-------|
| Alert | 4XXPercentagePerAPI (×20 firings) |
| Services | 24 distinct containers (users-api, marketplace-api, public-api, opportunities-*, ai-employees-api, integrations-api, oauth-*, cdp-core-api, bulk-actions-api, appengine-*) |
| Fired | 14:37–15:01 IST (09:07–09:31 UTC) |
| Duration | ~24 minutes |
| Impact | Elevated 4XX rates across CRM services; no infrastructure downtime, no pod restarts, no data loss |

## Root Cause

**Multiple independent application-level 4XX sources** crossed the `4XXPercentagePerAPI` alert threshold simultaneously during a period of elevated traffic. There was no single infrastructure failure or shared dependency outage. The 4XX rate genuinely increased (not a percentage artifact from traffic drop — total 2XX traffic was higher than the previous day).

**Top contributors:**
1. **marketplace-api** — 404 Not Found (56 of 200-sample; ~53% 4XX share, baseline ~45%)
2. **public-api** — 429 Rate Limiting on `/v1/contacts` endpoints (~63% 4XX share, baseline ~35%)
3. **users-api** — 401 Unauthorized on `/users/search` from many distinct accounts (~3.2% 4XX, baseline ~0.8%; rate jumped ~13x from 2.6 to 34.5 req/s)
4. **ai-employees-api** — 403 Forbidden on `/ai-employees/employees/:id`
5. **integrations-api** — 400/404 on integration-specific endpoints

## Proof

<details>
<summary>[Grafana] users-api 4XX percentage spiked — max ~3.2% (baseline ~0.8%)</summary>

> **Verify:** The %4XX timeseries should show elevated 4XX during 14:37–15:01 IST compared to the surrounding hours.

![users-api %4XX](screenshots/001-api-requests-overview-users-api-4xx-responses.png)

**Context (filters + time range):**
![Context](screenshots/001-api-requests-overview-users-api-4xx-responses-context.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=users-api&from=1774859400000&to=1774864800000&viewPanel=3)
</details>

<details>
<summary>[Grafana] users-api traffic was higher than baseline — not a traffic drop</summary>

> **Verify:** API Hits/second should show traffic was at or above normal levels during the incident (max ~1092 req/s vs baseline ~337).

![users-api Hits/s](screenshots/002-api-requests-overview-users-api-hits-second.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=users-api&from=1774859400000&to=1774864800000&viewPanel=1)
</details>

<details>
<summary>[Grafana] public-api 4XX percentage elevated — max ~63% (baseline ~35%)</summary>

> **Verify:** The %4XX timeseries for public-api should show elevated 4XX. public-api has a high baseline 4XX rate (rate limits + external API consumer errors are normal), but the spike is above normal.

![public-api %4XX](screenshots/003-api-requests-overview-public-api-4xx-responses.png)

**Context (filters + time range):**
![Context](screenshots/003-api-requests-overview-public-api-4xx-responses-context.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=public-api&from=1774859400000&to=1774864800000&viewPanel=3)
</details>

<details>
<summary>[GCP] Mixed 4XX status codes across services — no single failure mode</summary>

> **Verify:** Log Explorer histogram should show 4XX entries distributed across multiple containers with different status codes (404, 429, 401, 403, 400).

![4XX Distribution](screenshots/001-gcp-4xx-distribution-across-affected-services-09-05-09-35-utc.png)

```
resource.type="k8s_container"
httpRequest.status>=400
httpRequest.status<500
resource.labels.container_name=("users-api" OR "marketplace-api" OR "public-api" OR "ai-employees-api" OR "integrations-api")
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0AhttpRequest.status%3E%3D400%0AhttpRequest.status%3C500%0Aresource.labels.container_name%3D%28%22users-api%22%20OR%20%22marketplace-api%22%20OR%20%22public-api%22%20OR%20%22ai-employees-api%22%20OR%20%22integrations-api%22%29;timeRange=2026-03-30T09%3A05%3A00Z%2F2026-03-30T09%3A35%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[GCP] users-api 401 auth failures — many distinct accounts, not one bad caller</summary>

> **Verify:** Log entries should show 401 on /users/search with different companyId/locationId values — confirming this is not a single client issue.

![users-api 401s](screenshots/002-gcp-users-api-401-unauthorized-errors.png)

```
resource.type="k8s_container"
resource.labels.container_name="users-api"
httpRequest.status=401
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22users-api%22%0AhttpRequest.status%3D401;timeRange=2026-03-30T09%3A05%3A00Z%2F2026-03-30T09%3A35%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[GCP] public-api 429 rate limits — external API consumers hitting rate limits</summary>

> **Verify:** Log entries should show 429 on /v1/contacts endpoints — rate limiting working as expected on high-volume external API traffic.

![public-api 429s](screenshots/003-gcp-public-api-429-too-many-requests.png)

```
resource.type="k8s_container"
resource.labels.container_name="public-api"
httpRequest.status=429
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22public-api%22%0AhttpRequest.status%3D429;timeRange=2026-03-30T09%3A05%3A00Z%2F2026-03-30T09%3A35%3A00Z?project=highlevel-backend)
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| Medium | Tune `4XXPercentagePerAPI` alert thresholds per service — marketplace-api and public-api have high baseline 4XX rates (45-63%) that frequently trigger this alert | Platform / Alert Config |
| Medium | Investigate `/v1/contactsTEST` traffic on public-api — a client (axios/1.13.2) is making repeated requests to a non-existent endpoint, generating 404s/429s | CRM-marketplace |
| Low | Review users-api 401 patterns — many accounts receiving auth denials during the window; determine if this is a token rotation issue or expected behavior | CRM-users-internal |
| Low | Consider per-status-code alerting (401 vs 404 vs 429) instead of aggregate 4XX — would help distinguish auth issues from rate limits from missing endpoints | Platform / Alert Config |

## Links

- [Verbose report](report-verbose.md)
- [Grafana — API Requests Overview](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&from=1774859400000&to=1774864800000)
