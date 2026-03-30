# 4XXPercentagePerAPI — CRM Conversations APIs — 2026-03-30

**Author:** Himanshu Bhutani | **Status:** Auto-resolved (all 15 alerts acknowledged)

## Summary

| Field | Value |
|-------|-------|
| Alert | 4XXPercentagePerAPI (15 alerts) |
| Services | conversations-messages-send-api, conversations-external-api, conversations-frontend-external-api, conversations-messages-external-api, conversations-providers-external-api, conversations-search-external-api, conversations-inbound-external-api, conversations-bulk-internal-api, conversations-send-message-workflows-api, conversations-messages-internal-api, crm-conversations-scoring-api |
| Fired | 14:44 – 15:09 IST (09:14 – 09:39 UTC) |
| Duration | ~25 minutes |
| Impact | **No user-facing impact.** 4XX responses are business validation rejections (DND, unsubscribed, invalid email, missing phone). These are normal client errors, not service degradation. |

## Root Cause

**False positive / noisy alert.** The `4XXPercentagePerAPI` alerts fired on **normal baseline 4XX rates** across conversation APIs. The 4XX percentage was not elevated above baseline — it oscillates 7–80% throughout the day due to business validation errors (HTTP 400). The per-route percentage temporarily crossed the alert threshold during normal traffic oscillation, triggering 15 alerts across different API routes. A sidecar resource deployment at 14:23 IST (08:53 UTC) is temporally correlated but did NOT cause a 4XX increase.

**Confidence:** High — aggregate 4XX rate at incident time (~356 req/s) was lower than 2h prior (~469 req/s); dominant errors are HTTP 400 from validation logic.

## Proof

<details>
<summary>[Grafana] 4XX % for conversations-messages-send-api oscillates 5–80% all day — no spike at alert time</summary>

> **Verify:** The green line (POST /conversations/message) spikes to ~78% at 13:10 IST — well BEFORE the 14:23 IST deployment. During the alert window (14:44–15:09 IST), the rate is ~15–25%, within the same baseline range seen throughout the day.

![4XX %](screenshots/001-4xx-for-conversations-messages-send-api-07-00-11-00-utc.png)
**Context (filters + time range):**
![Context](screenshots/001-4xx-for-conversations-messages-send-api-07-00-11-00-utc-context.png)
[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=conversations-messages-send-api&from=1774854000000&to=1774868400000&viewPanel=3)
</details>

<details>
<summary>[Grafana] conversations-bulk-internal-api runs at 15–35% 4XX baseline — always high</summary>

> **Verify:** The green line (POST /conversations/messages/bulk) stays between 15–35% throughout the entire window. This service inherently has high 4XX rates because bulk operations target contacts with invalid data (no phone, unsubscribed email, DND active).

![4XX % bulk](screenshots/003-4xx-for-conversations-bulk-internal-api-07-00-11-00-utc.png)
[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=conversations-bulk-internal-api&from=1774854000000&to=1774868400000&viewPanel=3)
</details>

<details>
<summary>[Prometheus] Aggregate 4XX rate at alert time (~356/s) was LOWER than 2h baseline (~469/s)</summary>

> **Verify:** The instant query at 09:14 UTC returns ~356 req/s total 4XX for all conversation services. The same query at 07:14 UTC (2h before) returns ~469 req/s. The 4XX absolute volume is NOT elevated.

**At incident (09:14 UTC):**
```
sum(rate(istio_requests_total{response_code=~"4..",destination_service_name=~"conversations.*"}[30m])) = ~356 req/s
```

**Baseline (07:14 UTC):**
```
sum(rate(istio_requests_total{response_code=~"4..",destination_service_name=~"conversations.*"}[30m])) = ~469 req/s
```
</details>

<details>
<summary>[GCP Logs] 51,555 WARNING+ entries — all business validation errors, not infrastructure failures</summary>

> **Verify:** Every log entry is a business validation HttpException: "Cannot send message as DND is active", "Missing phone number", "Cannot send email as X has unsubscribed", "Unable to send e-mail, contact's e-mail is invalid". Zero infrastructure errors.

![GCP Logs](screenshots/001-gcp-validation-errors-in-conversations-messages-send-api-during-.png)

```
resource.type="k8s_container"
resource.labels.container_name="conversations-messages-send-api"
severity>=WARNING
jsonPayload.message=~"HttpException: Cannot send|HttpException: Unable to send|HttpException: Missing phone"
```
[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-messages-send-api%22%0Aseverity%3E%3DWARNING%0AjsonPayload.message%3D~%22HttpException%3A%20Cannot%20send%7CHttpException%3A%20Unable%20to%20send%7CHttpException%3A%20Missing%20phone%22;timeRange=2026-03-30T09%3A00%3A00Z%2F2026-03-30T09%3A30%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[Slack] Deployment at 14:23 IST is temporally correlated but NOT causal</summary>

> **Verify:** Jenkins #213 (leadconnector production) was deployed at 14:23 IST (08:53 UTC) — ~22 min before the first alert. However, this was a **sidecar resource change** (PR #26921), not an application code change. The 4XX percentage was already 17–19% at 13:40–13:55 IST (08:10–08:25 UTC) — BEFORE the deployment.

Deployment message in `#core-crm-conversations-internal` at ts=1774860797 by himanshu.bhutani.
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| Medium | **Tune `4XXPercentagePerAPI` thresholds** for conversation APIs — current thresholds are too sensitive for services with 7–35% baseline 4XX from validation | CRM Conversations team |
| Medium | **Exclude HTTP 400 from the alert metric** or create separate thresholds for 400 (validation) vs 401/403/429 (auth/rate-limit) | Platform / CRM Conversations |
| Low | **Review if validation rejections should be logged as WARNING** — DND, unsubscribed, invalid email are expected business flows, not warning-worthy | CRM Conversations team |

## Links

- [Verbose report](report-verbose.md)
- [Grafana — API Requests Overview (send-api)](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=conversations-messages-send-api&from=1774854000000&to=1774868400000)
- [Grafana — API Requests Overview (bulk-internal)](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=conversations-bulk-internal-api&from=1774854000000&to=1774868400000)
- [GCP Log Explorer — validation errors](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-messages-send-api%22%0Aseverity%3E%3DWARNING%0AjsonPayload.message%3D~%22HttpException%3A%20Cannot%20send%7CHttpException%3A%20Unable%20to%20send%7CHttpException%3A%20Missing%20phone%22;timeRange=2026-03-30T09%3A00%3A00Z%2F2026-03-30T09%3A30%3A00Z?project=highlevel-backend)
