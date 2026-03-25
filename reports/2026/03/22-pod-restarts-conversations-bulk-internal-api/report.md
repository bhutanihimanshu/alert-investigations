# PodRestartsAboveThreshold — conversations-bulk-internal-api — 2026-03-22

**Author:** Himanshu Bhutani | **Status:** Auto-resolved

## Summary

| Field | Value |
|-------|-------|
| Alert | PodRestartsAboveThreshold (#113183) |
| Service | conversations-bulk-internal-api |
| Cluster | servers-us-central-production-cluster |
| Fired | 01:36 IST (20:06 UTC), March 22, 2026 |
| Duration | ~2.5 hours of repeated escalations, auto-resolved |
| Impact | False positive — no actual application crash. KEDA HPA scaling churn caused K8s to count pod creation/deletion as "restarts." |

## Root Cause

**KEDA HPA scaling churn (false positive alert).** CPU pressure triggered KEDA to rapidly scale from 150→266 pods in ~1 minute (20:00–20:01 UTC), then immediately scale back to 150 when CPU dropped below target. During this rapid cycle, newly created pods failed startup probes (HTTP 500 — app not ready) and terminating pods had istio-proxy readiness probe failures ("connection reset by peer"). K8s counted these as "restarts," triggering the alert. No actual application crash occurred — CPU peaked at 80.5% of request, memory at 68.6%.

## What Happened

1. **01:30 IST (19:30 UTC)** — KEDA HPA scaled 150→168 pods due to CPU above target, then back to 150 at 01:06 IST.
2. **01:30 IST (20:00 UTC)** — Aggressive scale-up: 150→173→204→238→266 pods in ~1 minute (CPU pressure).
3. **01:38 IST (20:08 UTC)** — KEDA `FailedGetExternalMetric` errors for PubSub metric — metrics adapter unable to fetch `crm-contacts-bulk-email-v2-events-sub` metric.
4. **01:39 IST (20:09 UTC)** — Scale-down 266→150 (CPU below target). Newly created pods terminated before becoming ready.
5. **02:36 IST (20:36 UTC)** — Final scale-down 177→150. Alert auto-resolved.

## Proof

<details>
<summary>[Grafana] Pod count spiked 150 → 266 at 01:31 IST — KEDA HPA aggressive scale-up</summary>

> **Verify:** Pod count line jumps from 150 to 266 in ~1 minute, then drops back to 150. This is HPA churn, not a crash-recovery cycle.

![Pod Count](screenshots/003-app-detailed-view-pod-count-annotated.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=conversations-bulk-internal-api&var-cluster=servers-us-central-production-cluster&from=1774121400000&to=1774125900000&viewPanel=32)
</details>

<details>
<summary>[Grafana] CPU peaked at 0.8 cores (80.5% of 1-core request) — not saturated</summary>

> **Verify:** CPU usage stays below the 1-core request line. No CPU saturation — the HPA scaling was triggered by the 50% CPU threshold in KEDA config, not by actual resource exhaustion.

![CPU by Pod](screenshots/001-app-detailed-view-cpu-by-pod-annotated.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=conversations-bulk-internal-api&var-cluster=servers-us-central-production-cluster&from=1774121400000&to=1774125900000&viewPanel=16)
</details>

<details>
<summary>[Grafana] Memory peaked at 1054Mi (68.6% of 1536Mi request) — well within limits</summary>

> **Verify:** Memory usage stays well below the 1536Mi request line. No memory pressure.

![Memory by Pod](screenshots/002-app-detailed-view-memory-by-pod.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=conversations-bulk-internal-api&var-cluster=servers-us-central-production-cluster&from=1774121400000&to=1774125900000&viewPanel=30)
</details>

<details>
<summary>[GCP] KEDA HPA SuccessfulRescale events — scale-up 150→168, then back to 150</summary>

> **Verify:** `SuccessfulRescale` events show CPU-driven scaling. Scale-up to 168 followed by scale-down to 150 — normal HPA lifecycle, not a crash.

![HPA Scaling Events](screenshots/005-gcp-k8s-keda-hpa-successfulrescale-events-annotated.png)

GCP query:
```
resource.type="k8s_cluster"
jsonPayload.involvedObject.name=~"conversations-bulk-internal-api"
jsonPayload.reason="SuccessfulRescale"
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0AjsonPayload.involvedObject.name%3D~%22conversations-bulk-internal-api%22%0AjsonPayload.reason%3D%22SuccessfulRescale%22;timeRange=2026-03-22T00%3A30%3A00Z%2F2026-03-22T02%3A30%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[GCP] 1,721 readiness probe failures on terminating pods during HPA scale-down</summary>

> **Verify:** Kubelet logs show readiness probe failures on both `conversations-bulk-internal-api` container (connection refused on port 15020 — istio sidecar shutting down) and `istio-proxy` container (HTTP 503). All failures are on pods being terminated during HPA scale-down — not on healthy running pods.

![Kubelet Probe Failures](screenshots/006-gcp-kubelet-probe-failures-during-pod-termination-annotated.png)

Key log entries:
```
Probe failed probeType="Readiness"
  pod="conversations-bulk-internal-api-cbb57876f-5jhbx"
  containerName="conversations-bulk-internal-api"
  output="dial tcp 10.11.126.87:15020: connect: connection refused"

Probe failed probeType="Readiness"
  pod="conversations-bulk-internal-api-cbb57876f-5jhbx"
  containerName="istio-proxy"
  output="HTTP probe failed with statuscode: 503"
```

GCP query:
```
resource.type="k8s_node"
logName="projects/highlevel-backend/logs/kubelet"
jsonPayload.MESSAGE=~"Probe failed"
jsonPayload.MESSAGE=~"conversations-bulk-internal-api"
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_node%22%0AlogName%3D%22projects%2Fhighlevel-backend%2Flogs%2Fkubelet%22%0AjsonPayload.MESSAGE%3D~%22Probe%20failed%22%0AjsonPayload.MESSAGE%3D~%22conversations-bulk-internal-api%22;timeRange=2026-03-22T00%3A30%3A00Z%2F2026-03-22T02%3A30%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[GCP] KEDA FailedGetExternalMetric — 7 PubSub metric fetch failures</summary>

> **Verify:** Warning-level events showing KEDA's metrics adapter unable to fetch the PubSub subscription metric, forcing fallback to CPU-only scaling.

![KEDA Metric Errors](screenshots/007-gcp-keda-failedgetexternalmetric-errors-annotated.png)

GCP query:
```
resource.type="k8s_cluster"
jsonPayload.involvedObject.name=~"conversations-bulk-internal-api"
jsonPayload.reason="FailedGetExternalMetric"
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0AjsonPayload.involvedObject.name%3D~%22conversations-bulk-internal-api%22%0AjsonPayload.reason%3D%22FailedGetExternalMetric%22;timeRange=2026-03-22T00%3A30%3A00Z%2F2026-03-22T02%3A30%3A00Z?project=highlevel-backend)
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| Low | Tune KEDA `cooldownPeriod` (currently 600s) and add `stabilizationWindowSeconds` to prevent aggressive scale-up/down cycles | Platform / CRM Conversations |
| Low | Consider suppressing PodRestartsAboveThreshold for threshold=1 on high-replica deployments (150+ pods) where single "restart" during HPA scaling is expected | Platform |
| Info | KEDA `FailedGetExternalMetric` for PubSub metric `crm-contacts-bulk-email-v2-events-sub` — investigate why the metrics adapter fails intermittently | Platform |

## Links

- [Verbose report](report-verbose.md)
- [Grafana — App Detailed View](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=conversations-bulk-internal-api&var-cluster=servers-us-central-production-cluster&from=1774121400000&to=1774125900000)
- [Slack Alert Thread](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1774123564845819)
