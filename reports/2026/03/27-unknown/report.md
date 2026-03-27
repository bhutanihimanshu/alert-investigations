# HTTPLatencyPerApp — sentinel-api — 2026-03-27

**Author:** Himanshu Bhutani | **Status:** Auto-resolved (transient)

## Summary

| Field | Value |
|-------|-------|
| Alert | HTTPLatencyPerApp #113821 |
| Service | sentinel-api |
| Fired | 15:13 IST (09:43 UTC) |
| Duration | ~6 minutes (self-resolved by 15:14 IST) |
| Impact | No user-facing impact — internal script validation endpoint, no 5xx errors |

## Root Cause

A single slow external HTTP call in `fetchScriptContent()` blocked for ~15 seconds when customer-hosted URLs on `mediasimplifiednow.com` were unresponsive. With sentinel-api processing only ~1 request every 2.5 minutes, one slow request pushed the P99 latency above the 10s alert threshold. **This is a recurring pattern** — 55 historical alerts on this service.

## Proof

<details>
<summary>[Grafana] P99 latency spiked from 2.5s → 14.97s at 15:08 IST, resolved by 15:14 IST</summary>

> **Verify:** Look at the P99 Latency panel — a single spike to ~15s in an otherwise low-latency service. Traffic is ~0.006 rps — too low for reliable percentile calculation.

![API Requests Overview](screenshots/001-api-requests-overview-sentinel-api-p99-latency-spike-and-tra.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-datasource=ber8nnhvgsjy8f&var-container=sentinel-api&from=1774603800000&to=1774605300000)
</details>

<details>
<summary>[GCP Logs] 3 fetchScriptContent errors at 15:06-15:07 IST — external URLs unresponsive</summary>

> **Verify:** All 3 errors are `fetchScriptContent: Error fetching script for src:` pointing to `mediasimplifiednow.com` — a customer-hosted external URL that timed out.

```
resource.type="k8s_container"
resource.labels.container_name="sentinel-api"
severity>=ERROR
jsonPayload.message=~"fetchScriptContent"
```

| Time (IST) | Error |
|---|---|
| 15:06:57 | `fetchScriptContent` failed for `amandajohn.mediasimplifiednow.com` |
| 15:07:06 | `fetchScriptContent` failed for `payments.mediasimplifiednow.com` (latestupdates) |
| 15:07:07 | `fetchScriptContent` failed for `payments.mediasimplifiednow.com` (tourguide) |

![GCP Logs](screenshots/001-gcp-fetchscriptcontent-errors-external-script-fetch-timeouts.png)

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22sentinel-api%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22fetchScriptContent%22;timeRange=2026-03-27T09%3A30%3A00Z%2F2026-03-27T09%3A50%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[Grafana] Zero resource pressure — CPU <2%, memory 22% of limits</summary>

> **Verify:** CPU usage is flat at ~0.01 cores (limit: 1.1 cores). Memory stable at ~350MB (limit: 1.69GB). No resource contention.

![CPU by Pod](screenshots/002-app-detailed-view-cpu-by-pod-panel-16.png)
![Memory by Pod](screenshots/003-app-detailed-view-memory-by-pod-panel-30.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=sentinel-api&from=1774602000000&to=1774607400000)
</details>

<details>
<summary>[Code] axios timeout in fetchScriptContent() is 10s — matches alert threshold exactly</summary>

> **Verify:** `apps/sentinel/src/helper/customJSValidator.ts` uses `axios` with `timeout: 10000` (10s) in `fetchScriptContent()`. A single timeout pushes total request latency past the 10s alert threshold.

Team member in the Slack thread already identified this: *"reduce the timeout for axios call in customjshelper.ts file in marketplace-backend"*.
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| Medium | Reduce axios timeout in `fetchScriptContent()` from 10s → 3-5s | CRM-Marketplace team |
| Low | Raise HTTPLatencyPerApp threshold for sentinel-api (or exclude from alert) — too low-traffic for reliable P99 | CRM-Marketplace / Platform |
| Low | Add circuit breaker for repeated failures to same external domains | CRM-Marketplace team |

## Links

- [Verbose report](report-verbose.md)
- [API Requests Overview — Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-datasource=ber8nnhvgsjy8f&var-container=sentinel-api&from=1774603800000&to=1774605300000)
- [App Detailed View — Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=sentinel-api&from=1774602000000&to=1774607400000)
- [GCP Log Explorer — fetchScriptContent errors](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22sentinel-api%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22fetchScriptContent%22;timeRange=2026-03-27T09%3A30%3A00Z%2F2026-03-27T09%3A50%3A00Z?project=highlevel-backend)
