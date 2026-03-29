# PubSub Unacked Messages Investigation — clickhouse-indexing-bulk-actions-logs-v1 — 2026-03-29

**Author:** Himanshu Bhutani
**Generated:** 2026-03-29 20:30 IST

## 1. Alert Summary

| Field | Value |
|-------|-------|
| Alert type | Pubsub Unacked Messages above 10k |
| Alert ID | #113980 |
| Workload | clickhouse-indexing-bulk-actions-logs-v1-worker |
| Subscription | clickhouse-indexing-bulk-actions-logs-v1-events-sub |
| Cluster | workers-us-central-production-cluster |
| Namespace | default |
| Time | 19:10 IST (13:40 UTC), 2026-03-29 |
| Channel | #alerts-crm via CRM-bulk-actions integration |
| Severity | WARNING |
| Alert rule | [Grafana source](https://prod.grafana.leadconnectorhq.com/alerting/grafana/a869c751-a88d-4f06-b984-c4413e22edc3/view?orgId=1) |

The alert reported 202,870 unacked messages on `clickhouse-indexing-bulk-actions-logs-v1-events-sub`. The alert group contained 2 total firing alerts.

## 2. What Happened

1. **~18:30 IST (13:00 UTC)** — Publish burst to the topic caused `num_undelivered_messages` to start rising from baseline.
2. **~19:00 IST (13:30 UTC)** — Backlog peaked at ~832k undelivered messages. HPA began scaling up: 5→8→16→20 pods.
3. **~19:10 IST (13:40 UTC)** — Alert fired. Workers were processing (ack rate 7k-149k/min) but couldn't keep up with the publish burst.
4. **19:48-19:55 IST (14:18-14:25 UTC)** — Backlog drained to ~200k. HPA scaled DOWN aggressively: 20→17→8→5 within 2 minutes (reason: "CPU below target").
5. **19:55:06 IST (14:25:06 UTC)** — Scale-down terminated pods mid-processing. Burst of 50+ `INVALID: Subscriber closed` errors as in-flight acks failed.
6. **~20:00 IST (14:30 UTC)** — Remaining pods caught up. Backlog drained to <3k. Self-resolved.

<details>
<summary>Detailed timeline — full event log</summary>

| Time (IST) | Source | Event |
|---|---|---|
| ~18:30 | Cloud Monitoring | num_undelivered_messages begins rising from baseline |
| 19:00:27 | K8s HPA | SuccessfulRescale: New size 20 (from undelivered messages metric) |
| ~19:03 | Cloud Monitoring | Backlog peaks at ~832k undelivered |
| 19:10:34 | Grafana OnCall | Alert #113980 fired — 202,870 unacked (second wave value) |
| ~19:30 | Cloud Monitoring | Backlog dropping as workers process at 100k+/min |
| 19:36:16 | K8s HPA | ScalingReplicaSet: 16 → 20 pods |
| 19:36:16 | K8s HPA | SuccessfulRescale: New size 20 (undelivered messages metric) |
| 19:45:31 | K8s HPA | ScalingReplicaSet: Scaled up from 5 to 8 |
| 19:48:15 | K8s HPA | SuccessfulRescale: New size 8 (cpu below target) |
| 19:48:25 | K8s HPA | SuccessfulRescale: New size 5 (cpu below target) |
| 19:53:19 | K8s HPA | SuccessfulRescale: New size 17 (cpu below target) |
| 19:53:19 | K8s HPA | ScalingReplicaSet: 20 → 17 |
| 19:54:15 | K8s HPA | SuccessfulRescale: New size 8 (cpu below target) |
| 19:54:15 | K8s HPA | ScalingReplicaSet: 17 → 8 |
| 19:55:05 | K8s HPA | SuccessfulRescale: New size 5 (undelivered messages below target) |
| 19:55:05 | K8s HPA | ScalingReplicaSet: 8 → 5 |
| 19:55:06 | GCP Logs | Burst of 50+ `INVALID: Subscriber closed` errors during batch ack |
| 19:57:15 | Kubelet | Kill container failed / DeadlineExceeded for pod `...2pfnb` |
| ~20:00 | Cloud Monitoring | Backlog drained to <3k. Self-resolved |

</details>

## 3. Investigation Findings

### Evidence: Grafana — Pod Health

<details>
<summary>HPA pod count oscillated 5→20→5 repeatedly — the smoking gun for HPA thrashing</summary>

> **What to look for:** The green line shows pod count jumping between 5 (minReplicas) and 20 (maxReplicas, orange line) multiple times. Each oscillation cycle takes only a few minutes. The rapid scale-down while backlog is still draining is the direct cause of the alert.

![Pod Count](screenshots/005-app-detailed-view-number-of-pods-hpa-scaling.png)

**Context (filters + time range):**

![Context](screenshots/005-app-detailed-view-number-of-pods-hpa-scaling-context.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=clickhouse-indexing-bulk-actions-logs-v1-worker&from=1774785600000&to=1774796400000&viewPanel=32)
</details>

<details>
<summary>CPU stayed at 0.1-0.2 cores across all pods — I/O-bound worker, low CPU triggers premature scale-down</summary>

> **What to look for:** All pod CPU lines stay well below 0.5 cores (the request amount) even during peak processing. This low CPU is what the HPA uses to justify scaling DOWN — even while hundreds of thousands of messages are still in the backlog.

![CPU by Pod](screenshots/004-app-detailed-view-cpu-by-pod.png)

**Context (filters + time range):**

![Context](screenshots/004-app-detailed-view-cpu-by-pod-context.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=clickhouse-indexing-bulk-actions-logs-v1-worker&from=1774785600000&to=1774796400000&viewPanel=16)
</details>

### Evidence: GCP Logs — HPA Scaling Events

<details>
<summary>20 HPA events showing rapid scale-up (undelivered metric) and premature scale-down (CPU metric)</summary>

> **What to look for:** The "reason" field in the SuccessfulRescale events. Scale-UP events cite `num_undelivered_messages` as the reason. Scale-DOWN events cite `cpu resource utilization (percentage of request) below target`. These are two competing metrics — one says "scale up, there's a backlog" while the other says "scale down, CPU is low." The CPU metric wins the tiebreak, causing premature scale-down.

![HPA Events](screenshots/002-gcp-hpa-successfulrescale-events-rapid-scale-up-and-scale-down.png)

```
resource.type="k8s_cluster"
jsonPayload.involvedObject.name=~"clickhouse-indexing-bulk-actions-logs-v1-worker"
jsonPayload.reason=~"SuccessfulRescale|ScalingReplicaSet"
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0AjsonPayload.involvedObject.name%3D~%22clickhouse-indexing-bulk-actions-logs-v1-worker%22%0AjsonPayload.reason%3D~%22SuccessfulRescale%7CScalingReplicaSet%22;timeRange=2026-03-29T13%3A30%3A00Z%2F2026-03-29T14%3A30%3A00Z?project=highlevel-backend)
</details>

### Evidence: GCP Logs — Subscriber Closed Errors

<details>
<summary>50+ "Subscriber closed" errors at 19:55:06 IST — 1 second after HPA scaled 8→5</summary>

> **What to look for:** The timestamps are sub-second clustered at 14:25:06 UTC (19:55:06 IST), exactly 1 second after the HPA SuccessfulRescale event at 14:25:05Z. The stack trace shows `AckQueue.add → Subscriber.ack → ClickhouseIndexingWorker.processBatchMessage` — confirming the worker was in the middle of batch-acking messages when the PubSub subscriber was shut down during pod termination.

```
# Sample (5 of 50+ identical entries):
2026-03-29T14:25:06.440Z | unhandledRejectionError: INVALID : Subscriber closed (at AckQueue.add → Subscriber.ack → processBatchMessage)
2026-03-29T14:25:06.439Z | unhandledRejectionError: INVALID : Subscriber closed (at AckQueue.add → Subscriber.ack → processBatchMessage)
2026-03-29T14:25:06.439Z | unhandledRejectionError: INVALID : Subscriber closed (at AckQueue.add → Subscriber.ack → processBatchMessage)
2026-03-29T14:25:06.439Z | unhandledRejectionError: INVALID : Subscriber closed (at AckQueue.add → Subscriber.ack → processBatchMessage)
2026-03-29T14:25:06.439Z | unhandledRejectionError: INVALID : Subscriber closed (at AckQueue.add → Subscriber.ack → processBatchMessage)
```

GCP query:
```
resource.type="k8s_container"
resource.labels.container_name="clickhouse-indexing-bulk-actions-logs-v1-worker"
jsonPayload.message=~"Subscriber closed"
severity>=ERROR
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22clickhouse-indexing-bulk-actions-logs-v1-worker%22%0AjsonPayload.message%3D~%22Subscriber%20closed%22%0Aseverity%3E%3DERROR;timeRange=2026-03-29T14%3A20%3A00Z%2F2026-03-29T14%3A30%3A00Z?project=highlevel-backend)
</details>

### Evidence: Cloud Monitoring — PubSub Metrics

<details>
<summary>PubSub metrics show burst-and-drain pattern with healthy processing throughout</summary>

> **What to look for:** The ack rate never dropped to zero — workers were processing the entire time. The backlog was caused by a publish burst exceeding momentary capacity, not by a processing failure. Zero nacks confirm no retry amplification.

| Metric | Value | Source |
|--------|-------|--------|
| Peak undelivered messages | ~832,476 @ 13:33 UTC (19:03 IST) | Cloud Monitoring `num_undelivered_messages` |
| Second wave peak | ~315k @ 14:06 UTC, ~202k @ 14:14 UTC | Cloud Monitoring |
| Alert value | 202,870 | Slack alert message |
| Oldest unacked age (peak) | ~504 seconds (~8.4 min) | Cloud Monitoring `oldest_unacked_message_age` |
| Ack rate (per min) | 7k–149k (never zero) | Cloud Monitoring `ack_message_count` |
| Nack count | Zero (no data points) | Cloud Monitoring `nack_requests` |
| Sent/ack ratio | ~1.0x (9.08M sent / 9.05M ack) | Cloud Monitoring |
| Backlog drain time | ~10-15 min per wave | Cloud Monitoring timeline |

</details>

### Evidence: Pod Resources — No Resource Pressure

<details>
<summary>Memory at 17% of limit, CPU at 36% of request — worker is I/O-bound</summary>

> **What to look for:** These low resource numbers confirm the worker is I/O-bound (ClickHouse writes, network). CPU and memory are not the bottleneck. This is why the HPA's CPU-based scale-down is inappropriate for this workload.

| Metric | Value | Limit/Request |
|--------|-------|---------------|
| Memory (peak) | ~193 MiB | 1,126 MiB (17%) |
| CPU (peak) | ~0.178 cores | 0.5 cores (36%) |
| Pod restarts | 0 | — |

</details>

## 4. Cross-Validation

| Signal | Source 1 | Source 2 | Source 3 | Agree? |
|--------|----------|----------|----------|--------|
| HPA oscillation (5↔20) | Grafana pod count panel | GCP k8s_cluster HPA events | — | Yes |
| CPU low → triggers scale-down | Grafana CPU by Pod (0.1-0.2) | GCP HPA reason: "cpu below target" | VictoriaMetrics (36% of request) | Yes |
| Subscriber closed during scale-down | GCP ERROR logs (14:25:06Z) | GCP HPA event (14:25:05Z) | Kubelet DeadlineExceeded | Yes |
| Backlog self-resolved | Cloud Monitoring (drains to <3k) | No ongoing Slack thread activity | — | Yes |
| No pod restarts | VictoriaMetrics `kube_pod_container_status_restarts_total = 0` | No OOM in GCP logs | No heap errors | Yes |

**Confidence: HIGH** — All 3 evidence sources (Grafana, GCP logs, Cloud Monitoring) agree on the causal chain. This is a known recurring pattern — identical root cause was found for the same subscription on 2026-03-27.

## 5. Root Cause

**HPA thrashing due to competing scaling metrics** causes recurring PubSub backlog spikes on `clickhouse-indexing-bulk-actions-logs-v1-events-sub`.

### Causal Chain

1. **Publish burst** → traffic spike to the topic → `num_undelivered_messages` jumps to ~832k
2. **HPA scales up** (5→20 pods) based on `num_undelivered_messages` external metric
3. Workers process the backlog rapidly (ack rate stays 7k-149k/min) → backlog drains
4. **CPU is low** (0.1-0.2 cores) because the worker is I/O-bound (ClickHouse writes, not compute)
5. **HPA scales down** aggressively (20→5 within 2 minutes) — reason: "cpu resource utilization below target"
6. During scale-down, **pods are terminated while processing** → PubSub subscriber is closed
7. **`INVALID: Subscriber closed`** errors → in-flight acks fail → messages become unacked again
8. Brief backlog recurrence → HPA scales up again → **oscillation loop**
9. Eventually self-resolves as the publish burst subsides and remaining pods catch up

### Why the HPA Behavior is Wrong

The HPA has two metrics:
- `num_undelivered_messages` (external PubSub metric) — says "scale UP, there's a backlog"
- CPU utilization (resource metric) — says "scale DOWN, CPU is low"

For this **I/O-bound** worker, CPU will ALWAYS be low regardless of load. The CPU metric wins scale-down tiebreaks, causing pods to be removed while the backlog is still draining. This is a configuration issue, not a code issue.

<details>
<summary>Probable noise — transient errors during disruption (not root cause)</summary>

| Time (IST) | Pattern | Why it's noise |
|------------|---------|----------------|
| 19:39:29 | `[PLATFORM_CORE_CLICKHOUSE] Clickhouse error` + auto-retry | Single transient ClickHouse network error, auto-retried successfully. Not correlated with the backlog onset. |
| 19:57:15 | Kubelet: `Kill container failed` / `DeadlineExceeded` / `RST_STREAM` | Expected during aggressive pod termination. Side-effect of HPA scale-down, not independent issue. |
| 19:56 | Kubelet: `RemoveStaleState` / CPUSet cleanup | Normal post-container-stop housekeeping. |

</details>

## 6. Action Items

### For the alert (prevent recurrence)

| Priority | Action | Rationale |
|----------|--------|-----------|
| **High** | Increase HPA `scaleDown.stabilizationWindowSeconds` to 300-600s | Prevents premature scale-down while backlog is still draining |
| **High** | Remove CPU utilization metric from HPA, or set it to a very low target (e.g., 10%) | CPU will always be low for this I/O-bound worker; it causes incorrect scale-down decisions |
| **Medium** | Add graceful PubSub shutdown handler: stop pulling → wait for in-flight acks → close subscriber | Prevents `Subscriber closed` errors during pod termination |
| **Low** | Increase `minReplicas` to 8-10 | Reduces scale-down amplitude if this traffic pattern recurs daily |

### Separate observations

| Item | Detail |
|------|--------|
| Recurring pattern | Same alert fired 2 days ago (2026-03-27), same root cause. Also fired multiple times on 2026-03-28. HPA config fix would resolve all of these. |
| Other CRM-bulk-actions alerts | Recent HeapOOM and PodRestart alerts on the same channel suggest this worker group has broader stability issues worth investigating. |

## 7. Deployment Details

| Config | Value |
|--------|-------|
| Container | clickhouse-indexing-bulk-actions-logs-v1-worker |
| Cluster | workers-us-central-production-cluster |
| Namespace | default |
| CPU request | 0.5 cores |
| Memory limit | ~1,126 MiB |
| HPA minReplicas | 5 |
| HPA maxReplicas | 20 |
| HPA metrics | `num_undelivered_messages` (external), CPU utilization (resource) |
| Subscription | clickhouse-indexing-bulk-actions-logs-v1-events-sub |

## 8. Cross-Validation Summary

| Source | Finding | Confidence |
|--------|---------|------------|
| Grafana (App Detailed View) | Pod count oscillates 5↔20, CPU low at 0.1-0.2 cores | High |
| Cloud Monitoring (PubSub) | Backlog burst→drain pattern, healthy ack rate, zero nacks | High |
| GCP Logs (k8s_cluster) | HPA events: scale-up on undelivered, scale-down on CPU | High |
| GCP Logs (k8s_container) | Subscriber closed errors at scale-down timestamp | High |
| GCP Kubelet | DeadlineExceeded during container kill — confirms aggressive termination | High |
| Slack (Alert Correlator) | No correlated alerts in ±15 min, no recent deployment | Confirmed |

**Overall confidence: HIGH** — 5 independent sources agree. Known recurring pattern.
