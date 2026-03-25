# PodRestartsAboveThreshold — conversations-bulk-internal-api — 2026-03-19

**Author:** Himanshu Bhutani | **Status:** Auto-resolved

## Summary

| Field | Value |
|-------|-------|
| Alert | PodRestartsAboveThreshold (#113183) |
| Service | conversations-bulk-internal-api |
| Cluster | servers-us-central-production-cluster |
| Fired | 15:36 IST (10:06 UTC) — March 19, 2026 |
| Duration | ~1.5 hours (auto-resolved) |
| Impact | 10 pods killed during a cluster-wide Istio sidecar disruption affecting 58 workloads. Service recovered via HPA rescheduling. |

## Root Cause

**Cluster-wide Istio sidecar instability** caused readiness probe failures across 58 workloads in `servers-us-central-production-cluster` within a 2-minute window (09:30–09:31 UTC). The istio-proxy container on `conversations-bulk-internal-api` pods returned HTTP 503 on readiness probes, which also caused the app container's health check (routed through Istio's :15020 aggregation endpoint) to fail with "connection refused." K8s killed 10 pods. The service was already under HPA churn — scaling from 150 → 278 → 206 pods due to CPU pressure — and a deployment rollout (new ReplicaSet `cbb57876f`) was in progress, making it more vulnerable to sidecar disruptions.

Concurrently, istiod KEDA HPA was thrashing: scaled down from 600 → 478 pods (memory below target at 09:55 UTC), then back up to 600 (CPU above target at 09:58 UTC). This control plane churn destabilized sidecar proxy connections across the cluster.

## What Happened

1. **14:30 IST (09:00 UTC)** — KEDA HPA scaled `conversations-bulk-internal-api` from 150 → 278 pods in 35 seconds due to CPU pressure.
2. **14:40 IST (09:10 UTC)** — A deployment rollout started, creating new ReplicaSet `cbb57876f` (0 → 38 pods) while old ReplicaSet `59c778546c` scaled down.
3. **15:00 IST (09:30 UTC)** — Cluster-wide Istio sidecar disruption: istio-proxy readiness probe failures (HTTP 503) across **58 workloads** simultaneously.
4. **15:00 IST (09:30:44 UTC)** — K8s killed 10 `conversations-bulk-internal-api` pods (both app and istio-proxy containers stopped).
5. **15:25 IST (09:55 UTC)** — istiod KEDA HPA thrashed: 600 → 478 → 543 → 600 pods within 3 minutes.
6. **16:06 IST (10:36 UTC)** — Alert auto-resolved after pods rescheduled and recovered.

## Proof

<details>
<summary>[K8s Events] Readiness probe failures on istio-proxy — 58 workloads at 09:30 UTC</summary>

> **Verify:** `fieldPath` is `spec.containers{istio-proxy}` (NOT the app container). Multiple unrelated workloads (conversations, contacts, payments, oauth, etc.) all failed simultaneously.

**Top affected workloads (by event count):**

| Workload | Events |
|----------|--------|
| conversations-messages-internal-api | 27 |
| conversations-api | 16 |
| conversations-get-message-workflows-api | 13 |
| conversations-bulk-internal-api | 13 |
| conversations-providers-internal-api | 12 |
| location-get-custom-fields-workflows-api | 10 |
| payments-api | 5 |
| contacts-api | 4 |
| oauth-login-api | 2 |

200 events across 58 unique workloads in a 2-minute window.

GCP query:
```
resource.type="k8s_pod"
jsonPayload.reason="Unhealthy"
jsonPayload.involvedObject.fieldPath="spec.containers{istio-proxy}"
jsonPayload.message=~"Readiness probe failed.*503"
resource.labels.cluster_name="servers-us-central-production-cluster"
```

![GCP Logs](screenshots/002-gcp-cluster-wide-istio-proxy-readiness-failures-58-workloads.png)

</details>

<details>
<summary>[K8s Events] 10 pods killed — both app and istio-proxy containers at 09:30:44 UTC</summary>

> **Verify:** `reason=Killing` events show both `spec.containers{conversations-bulk-internal-api}` and `spec.containers{istio-proxy}` being stopped on the same pods at the same timestamp.

**Affected pods:**
`chcvl`, `6qbfd`, `ktfgf`, `n6j7g`, `lf2z4`, `jwjpv`, `n2m6c`, `qrtwr`, `l87rj`, `wwwtx`

GCP query:
```
resource.type="k8s_pod"
resource.labels.pod_name=~"conversations-bulk-internal-api.*"
jsonPayload.reason=~"Unhealthy|Killing"
timestamp>="2026-03-19T09:30:00Z"
timestamp<="2026-03-19T09:31:00Z"
```

![GCP Logs](screenshots/001-gcp-istio-proxy-readiness-probe-failures-conversations-bulk-inte.png)

</details>

<details>
<summary>[Kubelet] Readiness probe failures — both istio-proxy (503) and app (connection refused to :15020)</summary>

> **Verify:** Kubelet logs show interleaved probe failures: istio-proxy returns 503, app container gets "connection refused" on :15020 (Istio's health aggregation port). This confirms the app was healthy but Istio sidecar was not.

Key kubelet entries at 09:30:00 UTC:
- `containerName="istio-proxy" probeResult="failure" output="HTTP probe failed with statuscode: 503"` — on pods pf4rp, jlvmg, 6bvhs, dtfj7, xbpfq, xsf46
- `containerName="conversations-bulk-internal-api" probeResult="failure" output="dial tcp 10.x.x.x:15020: connect: connection refused"` — on pods 4rxlf, xbpfq, 4hfng, jlvmg, pf4rp

GCP query:
```
logName="projects/highlevel-backend/logs/kubelet"
"conversations-bulk-internal-api"
```

</details>

<details>
<summary>[K8s Cluster] istiod KEDA HPA thrashing — 600 → 478 → 543 → 600 in 3 minutes</summary>

> **Verify:** istiod scaled down by memory (below target) then immediately back up by CPU (above target). This competing metric oscillation destabilizes the control plane.

| Time (IST) | Event |
|---|---|
| 15:25:05 (09:55:05 UTC) | istiod scaled down 600 → 491 (memory below target) |
| 15:25:12 (09:55:12 UTC) | istiod scaled down 491 → 478 (memory below target) |
| 15:27:48 (09:57:48 UTC) | istiod scaled up 478 → 543 (CPU above target) |
| 15:28:41 (09:58:41 UTC) | istiod scaled up 543 → 600 (CPU above target) |

GCP query:
```
resource.type="k8s_cluster"
jsonPayload.reason=~"SuccessfulRescale|ScalingReplicaSet"
jsonPayload.involvedObject.name=~"istiod.*"
resource.labels.cluster_name="servers-us-central-production-cluster"
```

</details>

<details>
<summary>[K8s Cluster] HPA scaled service 150 → 278 pods + deployment rollout in progress</summary>

> **Verify:** Rapid HPA scaling (150 → 278 in 35s) followed by a deployment rollout (new ReplicaSet created at 09:10 UTC) made the service more vulnerable to sidecar disruptions.

| Time (IST) | Event |
|---|---|
| 14:30:44 (09:00:44 UTC) | HPA: 150 → 172 → 174 (CPU above target) |
| 14:30:50 (09:00:50 UTC) | Scaled 174 → 194 |
| 14:30:59 (09:00:59 UTC) | Scaled 194 → 223 |
| 14:31:09 (09:01:09 UTC) | Scaled 223 → 248 |
| 14:31:19 (09:01:19 UTC) | Scaled 248 → 278 |
| 14:39:47 (09:09:47 UTC) | HPA scale-down: 278 → 214 → 209 → 206 |
| 14:40:56 (09:10:56 UTC) | New ReplicaSet cbb57876f: 0 → 38 pods |

GCP query:
```
resource.type="k8s_cluster"
jsonPayload.reason=~"ScalingReplicaSet|SuccessfulRescale"
jsonPayload.involvedObject.name=~"conversations-bulk-internal-api.*"
```

</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| High | Tune istiod KEDA HPA to prevent competing CPU/memory metric oscillation — increase `stabilizationWindowSeconds`, add `behavior.scaleUp.policies` to limit pods-per-minute | Platform team |
| Medium | Investigate why 58 workloads experienced simultaneous istio-proxy readiness failures — check for node-level events, GCE incidents, or network partitions at 09:30 UTC | Platform team |
| Low | Consider increasing readiness probe `failureThreshold` for conversations-bulk-internal-api to tolerate transient sidecar disruptions | CRM Conversations team |

## Links

- [Verbose report](report-verbose.md)
- [App Detailed View — Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=conversations-bulk-internal-api&var-cluster=servers-us-central-production-cluster&from=1774119000000&to=1774130400000)
- [Slack Alert Thread](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1774123564845819)
