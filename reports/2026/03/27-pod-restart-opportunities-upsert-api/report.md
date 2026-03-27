# PodRestartsAboveThreshold — opportunities-upsert-api — 2026-03-27

**Author:** Himanshu Bhutani | **Status:** Auto-resolved | **Confidence:** High

## Summary

| Field | Value |
|-------|-------|
| Alert | PodRestartsAboveThreshold (#113813) |
| Service | opportunities-upsert-api |
| Cluster | servers-us-central-production-cluster |
| Fired | 11:19 IST (05:49 UTC) |
| Duration | ~6 minutes (auto-resolved via HPA scaling) |
| Impact | 1 pod restarted, 30 probe timeouts across 15 pods, brief 5XX elevation (~0.17 rps peak) |

## Root Cause

**Traffic spike (3.8x, 50→192 rps) caused event loop blocking (P99 lag 746ms) across all pods, triggering fleet-wide liveness/readiness probe timeouts.** One pod (`7h84w`) hit the liveness failure threshold and restarted. HPA correctly scaled 15→24 pods; the service auto-recovered within 6 minutes.

This is a **recurring pattern** — same root cause identified on 2026-03-16 (#113571). The recommended fix (CPU increase from 1→1.5 cores) has not been implemented, causing continued recurrence (~10 firings in 45 days).

## What Happened

1. **11:15 IST** — Traffic surged from ~50 rps to 192 rps (3.8x), causing event loop lag to spike to 746ms across all pods.
2. **11:15:12 IST** — All 15 pods' liveness and readiness probes timed out simultaneously ("context deadline exceeded").
3. **11:15:34 IST** — HPA scaled 15→21→24 pods (CPU above target). Pods self-recovered readiness within 16–21 seconds.
4. **11:18 IST** — Pod `7h84w` restarted (hit liveness failure threshold). Brief 5XX error spike.
5. **11:21 IST** — Traffic normalized, HPA scaled back to 15 pods. Alert auto-resolved.

## Proof

<details>
<summary>[Grafana] Traffic spike — 50→192 rps at 11:16 IST</summary>

> **Verify:** The hits/sec chart shows traffic jumping from ~50 rps baseline to 192 rps peak at 11:19 IST (05:49 UTC).

![API Requests Overview](screenshots/001-api-requests-overview-traffic-spike-50-192-rps-at-11-16-11-1.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=opportunities-upsert-api&from=1774588743000&to=1774592343000)
</details>

<details>
<summary>[Grafana] Event loop lag — P99 spiked to 746ms (normally 11–15ms)</summary>

> **Verify:** The event loop lag chart shows a massive spike at 11:16 IST. Pod `7h84w` (the restarted one) shows the highest lag at 746ms. Multiple other pods also exceeded 500ms.

![Event Loop Lag](screenshots/004-event-loop-lag-p99-spiked-to-746ms-at-11-16-ist-normally-11-.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/PTSqcpJWka/nodejs-application-dashboard?orgId=1&var-container=opportunities-upsert-api&from=1774588743000&to=1774592343000)
</details>

<details>
<summary>[GCP] Probe failures — all 15 pods, fieldPath confirms app container</summary>

> **Verify:** 30 `Unhealthy` events at 05:45:12–05:45:20 UTC. All show `fieldPath: spec.containers{opportunities-upsert-api}` — the **app container** failed probes, NOT istio-proxy. Message: "context deadline exceeded."

![Probe Failures](screenshots/001-gcp-liveness-readiness-probe-failures-all-15-pods-fieldpath-conf.png)

```
resource.type="k8s_pod"
resource.labels.pod_name=~"opportunities-upsert-api.*"
jsonPayload.reason="Unhealthy"
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.pod_name%3D~%22opportunities-upsert-api.*%22%0AjsonPayload.reason%3D%22Unhealthy%22;timeRange=2026-03-27T05%3A44%3A00Z%2F2026-03-27T05%3A47%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[GCP] HPA scaled 15→21→24→15 pods over 6 minutes</summary>

> **Verify:** Three `SuccessfulRescale` events — 15→21 at 05:45:34, 21→24 at 05:45:41, then 24→15 at 05:51:10 after traffic subsided.

![HPA Scaling](screenshots/002-gcp-hpa-successfulrescale-15-21-24-15-pods-over-6-minutes.png)

```
resource.type="k8s_cluster"
jsonPayload.reason="SuccessfulRescale"
jsonPayload.involvedObject.name=~"opportunities-upsert-api"
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0AjsonPayload.reason%3D%22SuccessfulRescale%22%0AjsonPayload.involvedObject.name%3D~%22opportunities-upsert-api%22;timeRange=2026-03-27T05%3A44%3A00Z%2F2026-03-27T05%3A55%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[Grafana] CPU and memory well below limits — not OOM, not CPU-saturated</summary>

> **Verify:** CPU peaked at 0.318 cores (29% of 1.1 core limit). Memory peaked at 724 MB (32% of 2,253 MB limit). The event loop was blocked by I/O, not CPU or memory pressure.

![CPU by Pod](screenshots/002-cpu-by-pod-peak-0-318-cores-well-below-1-1-core-limit.png)
![Memory by Pod](screenshots/003-memory-by-pod-peak-724-mb-well-below-2-253-mb-limit.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=opportunities-upsert-api&var-cluster=servers-us-central-production-cluster&from=1774588743000&to=1774592343000)
</details>

## Action Items

| Priority | Action | Status |
|----------|--------|--------|
| **Medium** | Increase CPU request from 1→1.5 cores or tune HPA to scale earlier (lower targetCPUUtilizationPercentage) | **Not implemented** (first recommended 2026-03-16) |
| **Low** | Investigate upstream callers causing periodic traffic spikes (workflows, bulk actions) | **Not implemented** |
| **Info** | This alert has fired ~10 times in 45 days with the same root cause | Recurring — needs the above fixes |

## Links

- [Verbose report](report-verbose.md)
- [Grafana — App Detailed View](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=opportunities-upsert-api&var-cluster=servers-us-central-production-cluster&from=1774588743000&to=1774592343000)
- [Grafana — API Requests Overview](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=opportunities-upsert-api&from=1774588743000&to=1774592343000)
- [Grafana — NodeJS Event Loop](https://prod.grafana.leadconnectorhq.com/d/PTSqcpJWka/nodejs-application-dashboard?orgId=1&var-container=opportunities-upsert-api&from=1774588743000&to=1774592343000)
- [Prior investigation — 2026-03-16](../../../docs/public/conversations/pod-restart-opportunities-upsert-api-2026-03-16.md)
