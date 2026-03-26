# PodRestartsAboveThreshold — opportunities-pipelines-get-api — 2026-03-26

**Author:** Himanshu Bhutani | **Status:** Auto-resolved

## Summary

| Field | Value |
|-------|-------|
| Alert | [#113753 PodRestartsAboveThreshold](https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IGUQCV233DUVY) |
| Service | `opportunities-pipelines-get-api` |
| Cluster | `servers-us-central-production-cluster` |
| Fired | 21:34 IST (16:04 UTC) — 2026-03-26 |
| Duration | ~45 min (21:06–21:53 IST) |
| Impact | 3 pod restarts counted; HPA oscillated 15→27→17 pods; brief 5xx during pod terminations |

## Root Cause

**HPA CPU-driven scale-down → ioredis "Connection is closed" unhandled promise rejection → exit code 1.** When HPA scaled down pods, SIGTERM closed Redis TCP sockets. The ioredis client emitted the close event as an unhandled promise rejection, crashing the Node.js process (exit 1) instead of shutting down gracefully. This is the **same ioredis vulnerability** documented in the [2025-12-09 incident](https://gohighlevel.slack.com/archives/C0315RRNH1B/p1765278655918829) where all 30 pods crashed.

## What Happened

1. **21:06 IST** — HPA scaled up 15→17 pods (CPU above target). Two new pods (`ggkkt`, `ktkgn`) failed startup probes (HTTP 500).
2. **21:15 IST** — Kubelet killed the failing pods after 10 min of startup probe failures. Redis sockets closed → `Connection is closed` crash (exit 1).
3. **21:31 IST** — HPA aggressively scaled to 27 pods (CPU above target from earlier error storm).
4. **21:39–21:53 IST** — HPA scaled back down 27→17. Each terminated pod produced an ioredis unhandled rejection crash → exit code 1 → counted as restart.
5. **21:34 IST** — Alert fired when restart count crossed threshold of 1.

## Proof

<details>
<summary>[GCP Logs] ioredis "Connection is closed" crash — 7 instances between 21:15–21:53 IST</summary>

> **Verify:** Log entries show `triggerUncaughtException(err, true /* fromPromise */)` followed by `Error: Connection is closed` from ioredis. Timestamps cluster during HPA scale-down events.

| Time (IST) | Event |
|---|---|
| 21:15:53 | `triggerUncaughtException` + `Error: Connection is closed` (pod termination) |
| 21:39:26 | Same pattern ×2 (HPA 27→24) |
| 21:40:35 | Same (HPA 24→22) |
| 21:41:49 | Same (HPA 20→19) |
| 21:42:05 | Same (HPA 19→18) |
| 21:53:23 | Same (later scale-down) |

Stack trace:
```
Error: Connection is closed.
    at close (/app/node_modules/ioredis/built/redis/event_handler.js:184:25)
    at Socket.<anonymous> (/app/node_modules/ioredis/built/redis/event_handler.js:151:20)
```

![GCP Logs](screenshots/001-gcp-ioredis-connection-is-closed-unhandled-rejection-crash-logs.png)

```
resource.type="k8s_container"
resource.labels.container_name="opportunities-pipelines-get-api"
textPayload=~"triggerUncaughtException|Connection is closed"
severity>=WARNING
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22opportunities-pipelines-get-api%22%0Aresource.labels.cluster_name%3D~%22servers-us-central.%2A%22%0AtextPayload%3D~%22triggerUncaughtException%7CConnection%20is%20closed%22%0Aseverity%3E%3DWARNING;timeRange=2026-03-26T15%3A30%3A00Z%2F2026-03-26T16%3A30%3A00Z?project=highlevel-backend)

</details>

<details>
<summary>[Grafana] HPA oscillated 15→27→17 pods during incident window</summary>

> **Verify:** Pod count line shows: 15 → spike to 27 around 21:31 IST → gradual descent back to 17 by 21:53 IST. The doubling is HPA scaling, not a deployment rollout.

![Pod Count](screenshots/004-app-detailed-view-pod-count-hpa-scaling.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=servers-us-central-production-cluster&var-container=opportunities-pipelines-get-api&from=1774539000000&to=1774542600000&viewPanel=32)

</details>

<details>
<summary>[Grafana] Memory at ~50% — not OOM (exit code 1, not 137)</summary>

> **Verify:** Memory lines stay at 450–570 MB, well below the 1126 MB limit. Dips to ~3 MB indicate post-restart startup. Termination reason is "Error" (exit 1), not "OOMKilled" (137).

![Memory by Pod](screenshots/003-app-detailed-view-memory-by-pod.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=servers-us-central-production-cluster&var-container=opportunities-pipelines-get-api&from=1774539000000&to=1774542600000&viewPanel=30)

</details>

<details>
<summary>[Grafana] CPU not at limit — 0.45–0.66 cores avg vs 1.1 core limit</summary>

> **Verify:** CPU lines peak at ~0.98 cores on one pod (startup spike) but average 0.45–0.66. HPA triggered on utilization % relative to CPU request, not limit.

![CPU by Pod](screenshots/002-app-detailed-view-cpu-by-pod.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=servers-us-central-production-cluster&var-container=opportunities-pipelines-get-api&from=1774539000000&to=1774542600000&viewPanel=16)

</details>

<details>
<summary>[K8s Events] Startup probe failed HTTP 500 on new pods — app container, not istio-proxy</summary>

> **Verify:** Probe failure events show `fieldPath: spec.containers{opportunities-pipelines-get-api}` — confirming the APPLICATION container failed, not the sidecar. Pods `ggkkt` and `ktkgn` failed startup probes at 21:05–21:06 IST and were killed at 21:15:53 IST.

| Time (IST) | Pod | Event |
|---|---|---|
| 21:05:59 | `ggkkt` | Startup probe failed: HTTP 500 (`spec.containers{opportunities-pipelines-get-api}`) |
| 21:06:00 | `ktkgn` | Startup probe failed: HTTP 500 (`spec.containers{opportunities-pipelines-get-api}`) |
| 21:15:53 | Both | Killing (Stopping container) |

![K8s Events](screenshots/002-gcp-k8s-pod-events-startup-probe-failures-killing-events.png)

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.pod_name%3D~%22opportunities-pipelines-get-api.%2A%22%0Aresource.labels.cluster_name%3D~%22servers-us-central.%2A%22%0AjsonPayload.reason%3D~%22Unhealthy%7CKilling%7CBackOff%22;timeRange=2026-03-26T15%3A30%3A00Z%2F2026-03-26T16%3A30%3A00Z?project=highlevel-backend)

</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| **High** | Add `process.on('unhandledRejection')` handler to catch ioredis connection close during shutdown | CRM Opportunities team |
| **High** | Implement graceful Redis disconnect in SIGTERM handler (`redis.disconnect()` before exit) | CRM Opportunities team |
| **Medium** | Tune HPA `behavior.scaleDown.stabilizationWindowSeconds` to reduce rapid oscillation | CRM Opportunities team |
| **Low** | Investigate why new pods fail startup probe with HTTP 500 (app health check failing during init) | CRM Opportunities team |

## Cross-Validation

| Signal | Source | Confirms |
|--------|--------|----------|
| Exit code 1 (not 137) | GCP container logs | App crash, not OOM |
| `Connection is closed` stack trace | GCP container logs | ioredis unhandled rejection |
| Memory at 50% of limit | Grafana App Detailed View | Not OOM |
| HPA 15→27→17 oscillation | Grafana + K8s cluster events | Scale-down caused terminations |
| `fieldPath: spec.containers{app}` | K8s pod events | App container, not istio-proxy |
| Same pattern as 2025-12-09 | Slack thread (Amith's findings) | Known recurring vulnerability |

**Confidence: HIGH** — 6 independent sources confirm the same causal chain.

## Links

- [Verbose report](report-verbose.md)
- [Grafana — App Detailed View](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=servers-us-central-production-cluster&var-container=opportunities-pipelines-get-api&from=1774539000000&to=1774542600000)
- [GCP Log Explorer — crash logs](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22opportunities-pipelines-get-api%22%0Aresource.labels.cluster_name%3D~%22servers-us-central.%2A%22%0AtextPayload%3D~%22triggerUncaughtException%7CConnection%20is%20closed%22%0Aseverity%3E%3DWARNING;timeRange=2026-03-26T15%3A30%3A00Z%2F2026-03-26T16%3A30%3A00Z?project=highlevel-backend)
- [Slack alert thread](https://gohighlevel.slack.com/archives/C0315RRNH1B/p1774541060663219)
