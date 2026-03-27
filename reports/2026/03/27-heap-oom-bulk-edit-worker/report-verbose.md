# Heap OOM Investigation — bulk-edit-worker — 2026-03-27

**Author:** Himanshu Bhutani
**Generated:** 2026-03-27 22:30 IST

## Alert Summary

| Field | Value |
|-------|-------|
| Alert 1 | PodRestartsAboveThreshold (#113838) via CRM-bulk-actions |
| Alert 2 | Heap out of Memory (#113840) via CRM-bulk-actions |
| Workload | `crm-contacts-bulk-action-bulk-edit-worker` |
| Cluster | `workers-us-central-production-cluster` |
| Namespace | `default` |
| Alert 1 time | 20:41:53 IST (15:11:53 UTC) |
| Alert 2 time | 20:50:54 IST (15:20:54 UTC) |
| Team | CRM / bulk-actions |
| Recurrence | 4th Heap OOM in ~2 months (Jan 24, Jan 31, Feb 24, Mar 27) |

## Root Cause

**Load-driven JavaScript heap OOM with dangerously tight memory configuration.**

Two massive PubSub load surges triggered HPA scale-up from 10 → 200 pods (max). Under heavy bulk-edit processing, individual pods' memory spiked to the 845Mi container limit. The V8 heap limit (`--max-old-space-size=760MB`) leaves only ~85MB for Node.js native overhead, buffers, and concurrent HTTP response data. This headroom is insufficient when processing bulk operations with `flowControl=10` (10 concurrent messages, each dispatching `Promise.allSettled` on all sub-batches simultaneously).

**This is NOT a memory leak** — pods return to 250–300MB after restart. It is a per-request memory spike driven by:
1. Concurrent bulk-edit HTTP calls holding response buffers in memory
2. 30-second lock hold after processing, creating memory overlap between message lifecycles
3. `JSON.stringify(resp)` in error logging on potentially large response objects
4. CPU throttling (52% of pods exceeded 275m limit) slowing processing and extending memory retention

## What Happened

1. **20:37 IST** (15:07 UTC) — PubSub backlog surge on `crm-contacts-bulk-action-bulk-edit-events-sub`. HPA triggered by CPU utilization above target, then PubSub `num_undelivered_messages`.
2. **20:37–20:44 IST** (15:07–15:14 UTC) — HPA scaled from 10 → 200 pods in ~4 minutes. Many new pods hit `FailedScheduling` ("0/369 nodes are available: 254 Insufficient CPU").
3. **20:48 IST** (15:18 UTC) — First V8 heap OOM crash (pod `bdwqd`). 12 pods crashed in 13 minutes.
4. **21:00 IST** (15:30 UTC) — Peak restart rate: 56 restarts in 10-min window.
5. **21:08 IST** (15:38 UTC) — First wave subsiding. HPA begins scale-down.
6. **21:32 IST** (16:02 UTC) — Near baseline (17 pods).
7. **21:48 IST** (16:18 UTC) — Second PubSub surge. HPA scales back to 200.
8. **21:53 IST** (16:23 UTC) — Second wave of OOM crashes: 6 pods in 7 minutes.

<details>
<summary>Detailed timeline — full event log</summary>

| Time (IST) | Time (UTC) | Source | Event |
|---|---|---|---|
| 20:37:12 | 15:07:12 | K8s HPA | SuccessfulRescale: 10 → 12 (CPU above target) |
| 20:37:35 | 15:07:35 | K8s HPA | SuccessfulRescale: 12 → 17 (CPU above target) |
| 20:39:39 | 15:09:39 | K8s Pod | Readiness probe failed on istio-proxy (connection reset) |
| 20:39:53 | 15:09:53 | K8s HPA | SuccessfulRescale: 17 → 33 (PubSub backlog) |
| ~20:44 | ~15:14 | K8s HPA | Scaled to ~195 pods |
| ~20:46 | ~15:16 | K8s HPA | Hit 200 pods (max) |
| 20:48:17 | 15:18:17 | GCP Logs | FATAL ERROR: Reached heap limit — pod `bdwqd` |
| 20:48:20 | 15:18:20 | GCP Logs | FATAL ERROR: Reached heap limit — pod `g2sk7` |
| 20:48:33 | 15:18:33 | GCP Logs | FATAL ERROR: Reached heap limit — pod `h2ztn` |
| 20:48:47 | 15:18:47 | GCP Logs | FATAL ERROR: Reached heap limit — pod `skzdh` |
| 20:49:49 | 15:19:49 | GCP Logs | FATAL ERROR: Reached heap limit — pod `9lhj7` |
| 20:49:59 | 15:19:59 | GCP Logs | FATAL ERROR: Ineffective mark-compacts — pod `5mz6q` |
| 20:50:46 | 15:20:46 | GCP Logs | FATAL ERROR: Ineffective mark-compacts — pod `kxps8` |
| 20:54:09 | 15:24:09 | GCP Logs | FATAL ERROR: Ineffective mark-compacts — pod `skzdh` (2nd crash) |
| 20:58:03 | 15:28:03 | GCP Logs | FATAL ERROR: Reached heap limit — pod `xk8dv` |
| 20:59:25 | 15:29:25 | GCP Logs | FATAL ERROR: Ineffective mark-compacts — pod `zzlvs` |
| 20:59:32 | 15:29:32 | GCP Logs | FATAL ERROR: Ineffective mark-compacts — pod `hsjmg` |
| 21:00:57 | 15:30:57 | GCP Logs | FATAL ERROR: Reached heap limit — pod `bzjrq` |
| ~21:08 | ~15:38 | Grafana | Restart count declining (33 in 10-min window) |
| ~21:12 | ~15:42 | Grafana | Down to 10 restarts in 10-min window |
| ~21:32 | ~16:02 | Grafana | Pod count back to 17 |
| ~21:48 | ~16:18 | Grafana | Second surge begins — pods 30, scaling up |
| ~21:52 | ~16:22 | K8s HPA | Hit 200 pods (max) again |
| 21:53:38 | 16:23:38 | GCP Logs | FATAL ERROR: Reached heap limit — pod `8bqw6` |
| 21:55:35 | 16:25:35 | GCP Logs | FATAL ERROR: Reached heap limit — pod `rqkfn` |
| 21:56:18 | 16:26:18 | GCP Logs | FATAL ERROR: Reached heap limit — pod `x6zc8` |
| 21:56:24 | 16:26:24 | GCP Logs | FATAL ERROR: Reached heap limit — pod `jbnll` |
| 21:57:20 | 16:27:20 | GCP Logs | FATAL ERROR: Reached heap limit — pod `fglr8` |
| 21:59:44 | 16:29:44 | GCP Logs | FATAL ERROR: Ineffective mark-compacts — pod `99wnq` |

</details>

## Investigation Findings

### Evidence: Grafana — Memory Usage

<details>
<summary>Memory by Pod — peaks at 844.9 MB (100% of 845Mi limit)</summary>

> **What to look for:** Multiple pod traces spiking to the top of the chart (845Mi limit line). At least 37 pods reached 800–845 MB range. After restart, pods drop to 250–300 MB — confirming spike pattern, not gradual leak.

![Memory by Pod](screenshots/001-memory-by-pod-peaks-at-844-9-mb-100-of-845mi-limit.png)

**Context (filters + time range):**
![Context](screenshots/001-memory-by-pod-peaks-at-844-9-mb-100-of-845mi-limit-context.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=crm-contacts-bulk-action-bulk-edit-worker&from=1774620000000&to=1774630800000&viewPanel=30)

**Top memory offenders:**

| Pod | Peak Memory | % of Limit | Time (UTC) |
|-----|------------|-----------|------------|
| `9d5c4` | 844.9 MB | 100% | 15:26 |
| `skzdh` | 843.1 MB | 100% | 15:24 |
| `cjchr` | 842.2 MB | 100% | 15:30 |
| `267b8` | 841.8 MB | 100% | 15:26 |
| `kqct5` | 840.2 MB | 99% | 15:32 |

</details>

### Evidence: Grafana — Pod Restarts & Termination

<details>
<summary>Pod Restarts — two distinct waves matching OOM crash windows</summary>

> **What to look for:** Two restart spikes — the first at ~20:48–21:14 IST (peaking at 56 restarts/10min at 21:02 IST), the second starting ~21:50 IST. Gap between waves is ~40 minutes.

![Pod Restarts](screenshots/002-pod-restarts-two-restart-waves-matching-oom-crash-windows.png)

**Context (filters + time range):**
![Context](screenshots/002-pod-restarts-two-restart-waves-matching-oom-crash-windows-context.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=crm-contacts-bulk-action-bulk-edit-worker&from=1774620000000&to=1774630800000&viewPanel=36)
</details>

<details>
<summary>Termination Reason — 23 OOMKilled, 5 Error</summary>

> **What to look for:** OOMKilled should be the dominant termination reason. The 5 "Error" terminations are likely also memory-related but where V8 crashed before the kernel OOM-killed.

![Termination Reason](screenshots/003-termination-reason-23-oomkilled-5-error.png)

**Context:**
![Context](screenshots/003-termination-reason-23-oomkilled-5-error-context.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=crm-contacts-bulk-action-bulk-edit-worker&from=1774620000000&to=1774630800000&viewPanel=50)
</details>

### Evidence: Grafana — CPU & HPA

<details>
<summary>CPU by Pod — 52% of pods exceeded 275m limit</summary>

> **What to look for:** Many pod traces spiking above the CPU limit line (275m). Peak at 348m (127% of limit). CPU throttling slows processing, keeping bulk payloads in memory longer and exacerbating memory pressure.

![CPU by Pod](screenshots/004-cpu-by-pod-52-of-pods-exceeded-275m-limit.png)

**Context:**
![Context](screenshots/004-cpu-by-pod-52-of-pods-exceeded-275m-limit-context.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=crm-contacts-bulk-action-bulk-edit-worker&from=1774620000000&to=1774630800000&viewPanel=16)
</details>

<details>
<summary>Pod Count — HPA scaled 10 → 200 (max) during load surge</summary>

> **What to look for:** Pod count jumps from baseline (~10) to 200 (HPA max) in under 4 minutes. Two surge episodes visible. HPA was unable to scale further — horizontal scaling exhausted.

![Pod Count](screenshots/005-pod-count-hpa-scaled-10-200-max-during-load-surge.png)

**Context:**
![Context](screenshots/005-pod-count-hpa-scaled-10-200-max-during-load-surge-context.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=crm-contacts-bulk-action-bulk-edit-worker&from=1774620000000&to=1774630800000&viewPanel=32)
</details>

### Evidence: GCP Logs — OOM Crashes

<details>
<summary>FATAL ERROR: Reached heap limit — 20 crashes in two waves</summary>

> **What to look for:** Log entries with `FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory` and `FATAL ERROR: Ineffective mark-compacts near heap limit`. Timestamps should cluster in two windows: 15:18–15:31 UTC and 16:23–16:30 UTC.

![OOM Logs](screenshots/001-gcp-fatal-error-reached-heap-limit-20-crashes-in-two-waves.png)

```
resource.type="k8s_container"
resource.labels.container_name="crm-contacts-bulk-action-bulk-edit-worker"
textPayload=~"FATAL ERROR.*heap"
severity>=ERROR
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-contacts-bulk-action-bulk-edit-worker%22%0AtextPayload%3D~%22FATAL%20ERROR.%2Aheap%22%0Aseverity%3E%3DERROR;timeRange=2026-03-27T14%3A00%3A00Z%2F2026-03-27T16%3A30%3A00Z?project=highlevel-backend)

**Error breakdown:**

| Error Type | Count |
|-----------|-------|
| `Reached heap limit Allocation failed` | 12 |
| `Ineffective mark-compacts near heap limit` | 8 |

</details>

### Evidence: GCP Logs — K8s Pod Events

<details>
<summary>K8s pod events — Unhealthy/Killing/OOMKilling events</summary>

> **What to look for:** `Unhealthy` events with readiness probe failures on `istio-proxy` (port 15021). These are SECONDARY to the main container OOM — when the worker container crashes, the pod restarts and istio-proxy readiness probes fail during the restart.

![K8s Events](screenshots/002-gcp-k8s-pod-events-unhealthy-killing-oomkilling-events.png)

```
resource.type="k8s_pod"
resource.labels.pod_name=~"crm-contacts-bulk-action-bulk-edit-worker.*"
jsonPayload.reason=~"Unhealthy|Killing|OOMKilling|BackOff"
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.pod_name%3D~%22crm-contacts-bulk-action-bulk-edit-worker.%2A%22%0AjsonPayload.reason%3D~%22Unhealthy%7CKilling%7COOMKilling%7CBackOff%22;timeRange=2026-03-27T14%3A00%3A00Z%2F2026-03-27T16%3A30%3A00Z?project=highlevel-backend)

**Notable events:**

| Event | Details |
|-------|---------|
| FailedScheduling (15:07 UTC) | "0/369 nodes are available: 254 Insufficient CPU, 3 Insufficient memory" |
| NotTriggerScaleUp | "15 in backoff after failed scale-up, 3 max node group size reached" |
| Readiness probe failures | istio-proxy on port 15021 — "connection reset by peer" / "connection refused" |
| kubectl describe | Main container: Last State = **OOMKilled, Exit Code 137** |

</details>

### Evidence: Error Volume Timeline

| Time (IST) | Time (UTC) | ERROR Count (5min) | Phase |
|---|---|---|---|
| 20:30 | 15:00 | 73 | Baseline |
| 20:35 | 15:05 | 7 | Baseline |
| **20:40** | **15:10** | **231** | Load starting |
| **20:45** | **15:15** | **9,053** | OOM wave 1 peak |
| **20:50** | **15:20** | **4,323** | OOM wave 1 |
| **20:55** | **15:25** | **10,029** | OOM wave 1 peak |
| **21:00** | **15:30** | **6,350** | OOM wave 1 declining |
| 21:05 | 15:35 | 1,000 | Recovery |
| 21:10–21:45 | 15:40–16:15 | 28–365 | Calm |
| **21:50** | **16:20** | **2,025** | OOM wave 2 starting |
| **21:55** | **16:25** | **9,059** | OOM wave 2 peak |

## Cross-Validation

| Signal | Grafana | GCP Logs | Config | Slack | Agreement |
|--------|---------|----------|--------|-------|-----------|
| Memory at limit | 37 pods at 800-845 MB | 20 FATAL ERROR heap OOM | 760MB heap + 85MB headroom = 845Mi | — | ✅ All agree |
| OOM termination | 23 OOMKilled + 5 Error | Exit code 137 in kubectl | V8 crashes before kernel OOM | — | ✅ All agree |
| Two load surges | HPA 10→200 twice | HPA scaling events in k8s_cluster | — | — | ✅ Agree |
| CPU throttling | 52% pods >275m | — | 250m/275m request/limit | — | ✅ Agree |
| No deployment | — | — | Last deploy 4 days ago | No deploys found | ✅ Ruled out |
| Recurring pattern | — | — | — | 4th occurrence in 2 months | ✅ Confirmed |
| Cluster pressure | — | FailedScheduling, NotTriggerScaleUp | — | — | ✅ Confirmed |

**Confidence: HIGH** — All 3 primary sources (Grafana, GCP, Config) agree on root cause with corroborating timestamps.

## Deployment Details

| Setting | YAML | Live Deployment |
|---------|------|-----------------|
| Memory request | 756Mi | 768Mi |
| Memory limit | (auto) | **845Mi** |
| CPU request | 250m | 250m |
| CPU limit | (auto) | **275m** |
| V8 --max-old-space-size | (Helm default) | **760MB** |
| HPA minReplicas | 10 | 10 |
| HPA maxReplicas | 200 | 200 |
| HPA targetCPU | 65% | 65% |
| flowControl | 10 | 10 |
| ackDeadline | 600s | 600s |
| Last deploy | — | 2026-03-23 |

**V8 vs Container alignment:**
- V8 heap max: 760MB
- Container limit: 845Mi (~886MB)
- Available for non-heap: ~126MB (but this includes Node.js overhead, buffers, compiled code)
- **Effective headroom for HTTP buffers: ~60–85MB** — insufficient for 10 concurrent bulk operations

**Comparison with similar worker:**
- `bulk-delete` worker gets 1024Mi memory request (vs 756Mi for bulk-edit)
- Both process similar batch sizes with concurrent HTTP calls

## Memory-Intensive Code Patterns

1. **`Promise.allSettled` on all sub-batches simultaneously** — all document edits within a message dispatched concurrently, all HTTP responses held in memory at once
2. **`JSON.stringify(resp)` in error logging** — stringifying potentially large response objects (lines 317, 322, 373)
3. **30-second lock hold via `setTimeout`** — keeps lock and associated closures in memory for 30s after processing, causing memory overlap between message lifecycles
4. **flowControl=10** — with 10 concurrent messages and their batch data, HTTP responses, and lock timers, memory can spike significantly
5. **No Node.js Prometheus metrics** — blind to heap details, GC behavior, event loop lag

<details>
<summary>Probable noise — transient errors during disruption (not root cause)</summary>

| Time | Pattern | Why it's noise |
|------|---------|----------------|
| 15:09–15:10 UTC | istio-proxy readiness probe failures (connection reset, connection refused on :15021) | Secondary to main container OOM crash — sidecar restarts when main container dies |
| 15:07 UTC | FailedScheduling: "254 Insufficient CPU" | Cluster-level resource pressure from 200 pods spinning up — not the cause of OOM |
| Throughout | `PLATFORM_CORE_OBSERVABILITY` duplicate metric registration errors | Expected post-restart metric re-registration — see known root cause: Post-Restart Metric Registration Errors |
| 15:07+ | istio-proxy OOMKilled (exit 137) | istio-proxy has its own 500Mi limit; restarts are caused by main container crash destabilizing the pod, not independent memory pressure |

</details>

## Action Items

| Priority | Action | Reasoning |
|----------|--------|-----------|
| **High** | Increase memory limit to ~1.5–2Gi and `--max-old-space-size` to ~1200–1500MB | 845Mi is insufficient for bulk operations with 10 concurrent messages. The similar bulk-delete worker gets 1024Mi. |
| **High** | Increase CPU limit from 275m to 500m+ | 52% of pods are CPU-throttled, which slows processing and extends memory retention per request. |
| **Medium** | Reduce `flowControl` from 10 → 5 as interim mitigation | Halves concurrent memory usage while proper fix is implemented. |
| **Medium** | Implement chunked/streaming processing for large batch payloads | `Promise.allSettled` on all sub-batches holds all HTTP responses simultaneously. Process in smaller sequential chunks. |
| **Medium** | Investigate what triggers the PubSub load surges | Two surges drove HPA to max. Understanding the trigger (bulk import? workflow?) enables upstream throttling. |
| **Low** | Add Node.js Prometheus metrics (`prom-client`) | Currently blind to heap used/total, GC pause times, and event loop lag. Cannot monitor memory health proactively. |
| **Low** | Remove `JSON.stringify(resp)` from error logging for large responses | Allocates a copy of potentially large objects just for logging. |

## Recurring Pattern Analysis

This is the **4th Heap OOM occurrence in ~2 months**:

| Date | Alert | Outcome |
|------|-------|---------|
| ~2026-01-24 | Heap OOM (#99271) | Auto-resolved, no investigation |
| ~2026-01-31 | Heap OOM (#100652) | Auto-resolved, 4 sub-alerts |
| ~2026-02-24 | Heap OOM | Auto-resolved, no investigation |
| **2026-03-27** | **Heap OOM (#113840) + PodRestarts (#113838)** | **Current — 54 restarts, 20 OOM crashes** |

The pattern is worsening: earlier occurrences were single-digit restarts; this one had 54 restarts and maxed HPA at 200. Without a memory limit increase, this will recur — and likely worsen as bulk-edit usage grows.
