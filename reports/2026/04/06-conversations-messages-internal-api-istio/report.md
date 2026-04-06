# PodRestartsAboveThreshold — conversations-messages-internal-api (istio-proxy) — 2026-04-06

**Author:** Himanshu Bhutani | **Status:** Auto-resolved

## Summary

| Field | Value |
|-------|-------|
| Alert | PodRestartsAboveThreshold |
| Service | conversations-messages-internal-api |
| Container | istio-proxy (sidecar) |
| Cluster | servers-us-central-production-cluster |
| Fired | 09:55 IST (04:25 UTC) |
| Duration | Auto-resolved |
| Impact | No user impact — graceful pod evictions during routine cluster autoscaler activity |

## Root Cause

**Cluster autoscaler ScaleDown during low-traffic hours.** The GKE cluster autoscaler drained ~80 nodes across 7+ node pools between 09:30–10:00 IST (04:00–04:30 UTC) as nighttime traffic dropped. Pods on drained nodes were gracefully evicted, causing istio-proxy sidecars to be stopped. This affected 35+ services and 200+ pods — not just conversations-messages-internal-api. The alert threshold of 1 restart was breached when 5 pods from this service were evicted. This is expected operational behavior, not an incident.

## Proof

<details>
<summary>[GCP k8s_cluster] ~80 nodes drained between 04:00–04:03 UTC across 7+ node pools</summary>

> **Verify:** ScaleDown events show nodes removed with drain from xl-node-pool, hc-xxl-node-pool, and n2d-standard-64 pools. The drain wave started at 04:00:17 UTC and 20+ hc-xxl nodes were removed at 04:01:24 UTC in a single batch.

```
resource.type="k8s_cluster"
resource.labels.cluster_name="servers-us-central-production-cluster"
jsonPayload.reason=~"ScaleDown"
timestamp>="2026-04-06T04:00:00Z"
timestamp<="2026-04-06T04:10:00Z"
```

Key events:
| Time (IST) | Event |
|---|---|
| 09:30:17 | ScaleDown: gke-...-xl-node-pool-e780c5d1-f9x4 removed with drain |
| 09:30:27 | ScaleDown: gke-...-xl-node-pool-62715955-4d9x removed with drain |
| 09:31:24 | 20+ hc-xxl nodes removed in single batch |
| 09:33:02 | Additional xl, n2d-standard-64 nodes removed |

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0Aresource.labels.cluster_name%3D%22servers-us-central-production-cluster%22%0AjsonPayload.reason%3D~%22ScaleDown%22;timeRange=2026-04-06T04%3A00%3A00Z%2F2026-04-06T04%3A30%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[GCP k8s_pod] 200+ istio-proxy Killing events across 35 services in 04:00–04:30 UTC</summary>

> **Verify:** All events say "Stopping container istio-proxy" (graceful eviction), NOT "Unhealthy" or probe failure. 35 unique services affected including users-api (83 pods), appengine-crm-contacts-api (14), automation-calendars-api (12).

```
resource.type="k8s_pod"
resource.labels.cluster_name="servers-us-central-production-cluster"
jsonPayload.reason="Killing"
jsonPayload.message=~"istio-proxy"
timestamp>="2026-04-06T04:00:00Z"
timestamp<="2026-04-06T04:30:00Z"
```

Top affected services:
| Service | Pods evicted |
|---|---|
| users-api | 83 |
| appengine-crm-contacts-api | 14 |
| automation-calendars-api | 12 |
| oauth-token-refresh-api | 11 |
| conversations-get-message-workflows-api | 19 |
| locations-api | 4 |
| + 29 other services | ... |

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.cluster_name%3D%22servers-us-central-production-cluster%22%0AjsonPayload.reason%3D%22Killing%22%0AjsonPayload.message%3D~%22istio-proxy%22;timeRange=2026-04-06T04%3A00%3A00Z%2F2026-04-06T04%3A30%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[GCP kubelet] Only 2 transient readiness probe failures for conversations-messages-internal-api — app container, not istio-proxy</summary>

> **Verify:** containerName field says "conversations-messages-internal-api" (the app), NOT "istio-proxy". These are transient failures during pod rescheduling after eviction. probeType is "Readiness" (not "Liveness"), so pods were never killed — just briefly removed from service.

```
logName="projects/highlevel-backend/logs/kubelet"
"conversations-messages-internal-api"
timestamp>="2026-04-06T03:55:00Z"
timestamp<="2026-04-06T04:55:00Z"
```

| Time (IST) | Pod | Event |
|---|---|---|
| 09:53:56 | zv5lt | Readiness probe failed: context deadline exceeded |
| 10:24:17 | lhmq2 | Readiness probe failed: context deadline exceeded |

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=logName%3D%22projects%2Fhighlevel-backend%2Flogs%2Fkubelet%22%0A%22conversations-messages-internal-api%22;timeRange=2026-04-06T03%3A55%3A00Z%2F2026-04-06T04%3A55%3A00Z?project=highlevel-backend)
</details>

## What Happened

1. **09:30 IST** — HPA scaled down multiple workloads as nighttime traffic dropped. Cluster autoscaler began draining underutilized nodes.
2. **09:31 IST** — Massive ScaleDown wave: 20+ hc-xxl nodes removed in a single batch, followed by xl-node-pool and n2d-standard-64 nodes. ~80 total nodes drained.
3. **09:40–09:55 IST** — Pods on drained nodes gracefully evicted. istio-proxy sidecars stopped across 35+ services (200+ pods total). conversations-messages-internal-api had 5 pods evicted.
4. **09:55 IST** — Alert fired: PodRestartsAboveThreshold (5 > threshold of 1).
5. **Auto-resolved** — Evicted pods rescheduled on remaining nodes and became healthy.

<details>
<summary>Probable noise — transient errors during disruption (not root cause)</summary>

| Time (IST) | Pattern | Why it's noise |
|---|---|---|
| 09:53 | Readiness probe timeout on pod zv5lt | Transient during rescheduling — pod recovered |
| 10:24 | Readiness probe timeout on pod lhmq2 | Same — transient post-eviction rescheduling |

</details>

## Action Items

| Priority | Action | Owner |
|---|---|---|
| Low | Increase PodRestartsAboveThreshold threshold from 1 to 10+ for services on this cluster, or add rate-based condition (e.g., 5 restarts in 5 minutes in-place, excluding evictions) | Platform team |
| Low | Investigate why the alert metric counts pod evictions as "restarts" — Prometheus `kube_pod_container_status_restarts_total` doesn't increment for evicted pods, suggesting the alert uses a different metric source | Platform team |
| None | No application changes needed — conversations-messages-internal-api is healthy | — |

## Links

- [Verbose report](report-verbose.md)
- [Grafana App Detailed View](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=conversations-messages-internal-api&var-cluster=servers-us-central-production-cluster&from=1775169300000&to=1775172900000)
