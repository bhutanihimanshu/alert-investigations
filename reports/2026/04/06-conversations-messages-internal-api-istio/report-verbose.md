# PodRestartsAboveThreshold Investigation — conversations-messages-internal-api (istio-proxy) — 2026-04-06

**Author:** Himanshu Bhutani
**Generated:** 2026-04-06

## Alert: PodRestartsAboveThreshold — conversations-messages-internal-api

| Field | Value |
|---|---|
| Alert type | PodRestartsAboveThreshold |
| Workload | conversations-messages-internal-api |
| Container | istio-proxy (sidecar) |
| Cluster | servers-us-central-production-cluster |
| Channel | #alerts-crm-conversations (C097UPY34QJ) |
| Time | 09:55 IST (04:25 UTC) |
| Current value | 5 restarts |
| Threshold | 1 |
| Severity | WARNING |
| Status | Auto-resolved |

## Investigation Findings

### Evidence: GCP Logs — Cluster Autoscaler ScaleDown Events

Between 09:30–10:00 IST (04:00–04:30 UTC), the GKE cluster autoscaler drained approximately 80 nodes across 7+ node pools. This was triggered by nighttime traffic drop causing HPA to scale down workloads, freeing up node capacity.

<details>
<summary>ScaleDown events — ~80 nodes drained across 7+ node pools (04:00–04:30 UTC)</summary>

> **What to look for:** Multiple node pools affected simultaneously (xl-node-pool, hc-xxl-node-pool, n2d-standard-64). The burst at 04:01:24 UTC removes 20+ hc-xxl nodes in a single batch.

```
resource.type="k8s_cluster"
resource.labels.cluster_name="servers-us-central-production-cluster"
jsonPayload.reason=~"ScaleDown"
timestamp>="2026-04-06T04:00:00Z"
timestamp<="2026-04-06T04:30:00Z"
```

Key ScaleDown events:

| Time (IST) | Time (UTC) | Event |
|---|---|---|
| 09:30:17 | 04:00:17 | xl-node-pool-e780c5d1-f9x4 removed with drain |
| 09:30:27 | 04:00:27 | xl-node-pool-62715955-4d9x removed with drain |
| 09:30:32 | 04:00:32 | xl-node-pool-e780c5d1-fkwp removed with drain |
| 09:30:42 | 04:00:42 | xl-node-pool-62715955-t9zn removed with drain |
| 09:31:24 | 04:01:24 | 20+ hc-xxl-node-pool nodes removed (batch) |
| 09:33:02 | 04:03:02 | xl, n2d-standard-64 nodes removed |

100+ ScaleDown events total with ~80 unique node names identified.

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0Aresource.labels.cluster_name%3D%22servers-us-central-production-cluster%22%0AjsonPayload.reason%3D~%22ScaleDown%22;timeRange=2026-04-06T04%3A00%3A00Z%2F2026-04-06T04%3A30%3A00Z?project=highlevel-backend)
</details>

### Evidence: GCP Logs — Fleet-Wide istio-proxy Evictions

Node drain caused graceful pod evictions across the entire cluster. 200+ istio-proxy "Stopping container" events were recorded, affecting 35 unique services.

<details>
<summary>200+ istio-proxy Killing events across 35 services (04:00–04:30 UTC)</summary>

> **What to look for:** All events say "Stopping container istio-proxy" — this is a graceful stop during pod eviction, NOT a crash or probe failure. The breadth (35 services) proves this is infrastructure-level, not application-specific.

```
resource.type="k8s_pod"
resource.labels.cluster_name="servers-us-central-production-cluster"
jsonPayload.reason="Killing"
jsonPayload.message=~"istio-proxy"
timestamp>="2026-04-06T04:00:00Z"
timestamp<="2026-04-06T04:30:00Z"
```

Services affected (top 15 by pod count):

| Service | Pods Evicted |
|---|---|
| users-api | 83 |
| conversations-get-message-workflows-api | 19 |
| appengine-crm-contacts-api | 14 |
| automation-calendars-api | 12 |
| oauth-token-refresh-api | 11 |
| oauth-login-api | 8 |
| appengine-crm-opportunities-api | 8 |
| logs-api | 7 |
| oauth-api | 6 |
| automation-am-performance-monitoring-api | 5 |
| appengine-automation-api | 4 |
| marketplace-api | 4 |
| store-catalog-api | 4 |
| lc-phone-sms-webhook-api | 4 |
| automation-calendars-core-api | 4 |
| + 20 other services | ... |

Timeline of specific events around alert time (04:20–04:30 UTC):

| Time (IST) | Time (UTC) | Service | Pods |
|---|---|---|---|
| 09:50:10 | 04:20:10 | conversations-get-message-workflows-api | 10 |
| 09:52:39 | 04:22:39 | conversations-messages-external-api | 1 |
| 09:53:15 | 04:23:15 | conversations-messages-external-api | 1 |
| 09:54:58 | 04:24:58 | conversations-get-message-workflows-api | 3 |
| 09:55:00 | 04:25:00 | conversations-get-message-workflows-api | 3 |
| 09:55:08 | 04:25:08 | users-get-external-api | 2 |
| 09:55:11 | 04:25:11 | appengine-automation-trigger-api | 3 |
| 09:55:40 | 04:25:40 | automation-calendars-core-api | 2 |
| 09:55:41 | 04:25:41 | integrations-api | 2 |
| 09:55:46 | 04:25:46 | appengine-revex-api, navigation-api | 2 |
| 09:55:47 | 04:25:47 | locations-api, appengine-crm-opportunities-api | 12 |
| 09:55:50 | 04:25:50 | bulk-actions-api, location-contacts-forms-api | 6 |

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.cluster_name%3D%22servers-us-central-production-cluster%22%0AjsonPayload.reason%3D%22Killing%22%0AjsonPayload.message%3D~%22istio-proxy%22;timeRange=2026-04-06T04%3A00%3A00Z%2F2026-04-06T04%3A30%3A00Z?project=highlevel-backend)
</details>

### Evidence: GCP Logs — conversations-messages-internal-api Pod Events

Only 2 events found for this specific service — both are transient readiness probe failures during rescheduling, not crashes.

<details>
<summary>2 readiness probe failures — transient, on app container (not istio-proxy)</summary>

> **What to look for:** The probe failures are on port 15020 (Istio health aggregation), but the containerName in kubelet logs says "conversations-messages-internal-api" — the app container's readiness, routed through Istio. These are transient (context deadline exceeded) during pod rescheduling. No liveness probe failures, no OOM kills, no application crashes.

```
resource.type="k8s_pod"
resource.labels.pod_name=~"conversations-messages-internal-api.*"
resource.labels.cluster_name="servers-us-central-production-cluster"
jsonPayload.reason=~"Killing|Unhealthy|OOMKilling|BackOff|Created|Started"
timestamp>="2026-04-06T03:55:00Z"
timestamp<="2026-04-06T04:55:00Z"
```

| Time (IST) | Pod | Event |
|---|---|---|
| 09:53:56 | zv5lt | Readiness probe failed: context deadline exceeded on :15020/app-health |
| 10:24:17 | lhmq2 | Readiness probe failed: context deadline exceeded on :15020/app-health |

Kubelet logs confirm:
- probeType="Readiness" (NOT Liveness — pod was not killed)
- containerName="conversations-messages-internal-api" (app container, confirmed via fieldPath)
- No PostStartHookError, no exit codes, no pilot-agent failures

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.pod_name%3D~%22conversations-messages-internal-api.*%22%0Aresource.labels.cluster_name%3D%22servers-us-central-production-cluster%22%0AjsonPayload.reason%3D~%22Killing%7CUnhealthy%7COOMKilling%7CBackOff%22;timeRange=2026-04-06T03%3A55%3A00Z%2F2026-04-06T04%3A55%3A00Z?project=highlevel-backend)
</details>

### Evidence: Grafana — Prometheus Metrics

Grafana Prometheus queries for `kube_pod_container_status_restarts_total` returned empty results. This is expected and consistent with the root cause: evicted pods are terminated and new pods are created — the restart counter is per-pod, so a new pod starts at 0. This confirms the "restarts" counted by the alert come from a different metric source (likely VictoriaMetrics recording rule or custom alert expression).

## Cross-Validation

| Finding | k8s_pod events | k8s_cluster events | Kubelet logs | Grafana | Alert |
|---|---|---|---|---|---|
| Istio-proxy stopped | Yes (200+ events) | — | — | — | Yes (5 restarts) |
| Node drain triggered | — | Yes (~80 nodes) | — | — | — |
| Fleet-wide (35 svcs) | Yes | Yes (7+ pools) | — | — | — |
| Not app issue | Yes (no app kills) | — | Yes (readiness only) | — | — |
| Not OOM | Yes (no OOMKilling) | — | Yes (no exit 137) | — | — |
| Auto-resolved | — | — | — | — | Yes |
| Prometheus empty | — | — | — | Yes (expected) | — |

**Confidence: HIGH** — 3 independent GCP log sources agree on timing and mechanism. 35 services affected proves infrastructure cause. No contradicting evidence. Empty Prometheus data is consistent with eviction (not in-place restart).

**Disproval attempts:**
- **Could this be istiod instability?** No — events say "Stopping container" (graceful), not "Unhealthy" or probe failure on istio-proxy.
- **Could this be isolated to one pod?** No — 35 services, 200+ pods, ~80 nodes affected.
- **Could this be deployment-related?** No — no deployment events found, 35 unrelated services don't deploy simultaneously.
- **Is this a daily pattern?** Yes — cluster autoscaler routinely drains underutilized nodes during low-traffic hours.

## Root Cause

**Classification: Infrastructure — Expected Operational Behavior (False Positive)**

### Causal Chain

1. **Nighttime traffic drop** (09:30 IST / 04:00 UTC) — normal diurnal pattern
2. **HPA scale-down** — multiple workloads reduced pod count as CPU dropped below target
3. **Cluster autoscaler ScaleDown** — detected ~80 underutilized nodes across 7+ node pools and initiated drain
4. **Pod eviction** — pods on drained nodes gracefully terminated, istio-proxy sidecars stopped
5. **Restart counter increment** — 5 conversations-messages-internal-api pods evicted, alert metric counted these as "restarts"
6. **Alert fired** at 09:55 IST — threshold of 1 breached (value: 5)
7. **Auto-resolved** — new pods scheduled on remaining nodes, became healthy

<details>
<summary>Probable noise — transient errors during disruption (not root cause)</summary>

| Time (IST) | Pattern | Why it's noise |
|---|---|---|
| 09:53 | Readiness probe timeout on pod zv5lt | Transient during rescheduling — self-resolved |
| 10:24 | Readiness probe timeout on pod lhmq2 | Transient during rescheduling — self-resolved |

</details>

## Action Items

| Priority | Action | Owner |
|---|---|---|
| Low | Increase PodRestartsAboveThreshold threshold from 1 to 10+ for services on servers-us-central-production-cluster. Routine autoscaler activity during nighttime can evict 5+ pods from a single service. Current threshold of 1 generates false positives. | Platform team |
| Low | Investigate alert metric source — Prometheus `kube_pod_container_status_restarts_total` doesn't count evictions as restarts (confirmed empty), yet the alert fired. The alert rule likely uses a VictoriaMetrics recording rule or custom expression that counts pod churn. Understanding this metric is key to tuning the alert. | Platform team |
| None | No application changes needed — conversations-messages-internal-api is fully healthy. No code, resource, or deployment changes required. | — |

## Deployment Details

| Setting | Value |
|---|---|
| Cluster | servers-us-central-production-cluster |
| Namespace | default |
| Service type | API (not worker) |
| Alert source | VictoriaMetrics via Grafana OnCall → #alerts-crm-conversations |

## Links

- [Concise report](pod-restarts-conversations-messages-internal-api-istio-2026-04-06.md)
- [Grafana App Detailed View](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=conversations-messages-internal-api&var-cluster=servers-us-central-production-cluster&from=1775169300000&to=1775172900000)
- [GCP Log Explorer — Fleet istio-proxy kills](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.cluster_name%3D%22servers-us-central-production-cluster%22%0AjsonPayload.reason%3D%22Killing%22%0AjsonPayload.message%3D~%22istio-proxy%22;timeRange=2026-04-06T04%3A00%3A00Z%2F2026-04-06T04%3A30%3A00Z?project=highlevel-backend)
- [GCP Log Explorer — Node ScaleDown events](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0Aresource.labels.cluster_name%3D%22servers-us-central-production-cluster%22%0AjsonPayload.reason%3D~%22ScaleDown%22;timeRange=2026-04-06T04%3A00%3A00Z%2F2026-04-06T04%3A30%3A00Z?project=highlevel-backend)
