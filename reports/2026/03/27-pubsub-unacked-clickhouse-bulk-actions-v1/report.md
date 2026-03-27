# PubSub Unacked Messages — clickhouse-indexing-bulk-actions-logs-v1-worker — 2026-03-27

**Author:** Himanshu Bhutani | **Status:** Active (backlog still draining)

## Summary

| Field | Value |
|-------|-------|
| Alert | #113831 — Pubsub Unacked Messages above 10k |
| Service | `clickhouse-indexing-bulk-actions-logs-v1-worker` |
| Subscription | `clickhouse-indexing-bulk-actions-logs-v1-events-sub` |
| Cluster | `workers-us-central-production-cluster` |
| Fired | 18:46 IST (13:16 UTC) |
| Peak backlog | ~1,120,447 messages at ~19:06 IST (13:36 UTC) |
| Impact | ClickHouse indexing of bulk action logs delayed; no user-facing impact |
| Recurring | Yes — 7+ prior alerts, historically up to 1.5M messages |

## Root Cause

**HPA thrashing due to competing scaling metrics.** The worker's HPA uses both PubSub `num_undelivered_messages` (external metric, scales UP) and CPU utilization (scales DOWN). When a traffic burst arrives, HPA scales up to 20 pods. The new pods process messages quickly, dropping CPU utilization below the target. CPU-based scaling then aggressively scales down to 5 pods within minutes. With only 5 pods, the worker can't keep up with the bursty publish rate (up to 315k msgs/min), and the backlog rebuilds. This oscillation cycle repeats, preventing the backlog from fully draining.

## What Happened

1. **18:29 IST (12:59 UTC)** — HPA scaled DOWN from 9 → 5 pods (CPU below target), while backlog was building.
2. **18:34 IST (13:04 UTC)** — Backlog surged; HPA scaled UP 5 → 10 → 20 in two rapid steps driven by PubSub external metric.
3. **18:55 IST (13:25 UTC)** — With 20 pods, CPU utilization dropped; HPA scaled DOWN 20 → 12 → 9 → 8 → 5 in 4 steps over 4 minutes.
4. **19:00 IST (13:30 UTC)** — With 5 pods, backlog rebuilt; HPA scaled UP 5 → 10 → 15 → 20 in 3 rapid steps.
5. **19:06 IST (13:36 UTC)** — Backlog peaked at ~1,120,447. Worker processing at ~92.6k msgs/min (ack rate stayed positive throughout).

**Confidence: HIGH** — HPA events, PubSub metrics, and GCP logs all converge on this pattern.

## Proof

<details>
<summary>[Cloud Monitoring] Backlog peaked at 1.12M with sent/ack ratio ~1.0 (no retry amplification)</summary>

> **Verify:** num_undelivered_messages rose from ~541 at 17:30 IST to 1.12M at 19:06 IST. Ack rate remained high (~92.6k msgs/min average). Sent/ack ratio stayed ~1.0, ruling out nack-based retry storm.

![Worker Detailed View](screenshots/001-worker-detailed-view-subscription-stats-unacked-messages-ack.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a04e5483-eb8c-47ef-8198-30147926964c/worker-detailed-view?orgId=1&var-subscriptionId=clickhouse-indexing-bulk-actions-logs-v1-events-sub&var-projectId=highlevel-backend&from=1774611000000&to=1774621800000)
</details>

<details>
<summary>[K8s Events] HPA oscillated 5→20→5→20 pods in 30 minutes — competing CPU + PubSub metrics</summary>

> **Verify:** SuccessfulRescale events show scale-up driven by `num_undelivered_messages` and scale-down driven by `cpu resource utilization below target`, repeating within minutes.

| Time (IST) | Event |
|---|---|
| 18:29 | Scale DOWN 9 → 5 (CPU below target) |
| 18:34 | Scale UP 5 → 10 → 20 (PubSub backlog) |
| 18:55–18:59 | Scale DOWN 20 → 12 → 9 → 8 → 5 (CPU below target) |
| 19:00–19:01 | Scale UP 5 → 10 → 15 → 20 (PubSub backlog) |

![HPA Events](screenshots/002-gcp-hpa-thrashing-competing-metrics-cause-oscillation-between-5-.png)

GCP query:
```
resource.type="k8s_cluster"
(jsonPayload.reason="SuccessfulRescale" OR jsonPayload.reason="ScalingReplicaSet")
"clickhouse-indexing-bulk-actions-logs-v1"
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0A(jsonPayload.reason%3D%22SuccessfulRescale%22%20OR%20jsonPayload.reason%3D%22ScalingReplicaSet%22)%0A%22clickhouse-indexing-bulk-actions-logs-v1%22;timeRange=2026-03-27T12%3A00%3A00Z%2F2026-03-27T14%3A00%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[GCP Logs] 3,490 Subscriber closed errors — pods terminated mid-batch during scale-down</summary>

> **Verify:** Errors cluster around 12:17 UTC and 13:57 UTC, correlating with HPA scale-down events. The `Message.ack()` call fails because the PubSub subscriber was already closed when the pod was terminated.

![Subscriber Closed Errors](screenshots/001-gcp-subscriber-closed-errors-during-hpa-scale-down-events.png)

GCP query:
```
resource.type="k8s_container"
resource.labels.container_name="clickhouse-indexing-bulk-actions-logs-v1-worker"
severity>=ERROR
"Subscriber closed"
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22clickhouse-indexing-bulk-actions-logs-v1-worker%22%0Aseverity%3E%3DERROR%0A%22Subscriber%20closed%22;timeRange=2026-03-27T12%3A00%3A00Z%2F2026-03-27T14%3A00%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[24h Pattern] Chronic spikes (200k–1.1M) repeating throughout the day — not a one-off</summary>

> **Verify:** The 24h backlog trend shows repeated surges: 833k (Mar 26 20:00 IST), 895k (Mar 27 09:30 IST), 1.12M (Mar 27 19:30 IST). Between each spike, the backlog drains to <10k. This is the HPA thrashing cycle repeating.

| Time (IST) | Peak Backlog |
|---|---|
| Mar 26 20:00 | 833,272 |
| Mar 26 21:30 | 720,830 |
| Mar 27 05:00 | 578,778 |
| Mar 27 09:30 | 895,136 |
| Mar 27 16:00 | 170,061 |
| Mar 27 19:00 | 488,343 (alert time) |
| Mar 27 19:30 | 1,120,447 (peak) |

</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| **High** | Fix HPA competing metrics: either remove CPU-based scaling when external PubSub metric is configured, or add `stabilizationWindowSeconds` (300s+) to `scaleDown` behavior to prevent rapid scale-down | CRM-bulk-actions team |
| **High** | Increase `minReplicas` from current baseline (scales down to 5) to 10-15 to handle burst traffic without oscillation | CRM-bulk-actions team |
| **Medium** | Review alert threshold — 10k may be too low for this subscription which regularly operates at 50k–100k during normal burst processing | CRM-bulk-actions team |
| **Low** | Investigate ClickHouse auto-retry warnings (13 in 2h) — may indicate intermittent ClickHouse connectivity affecting processing speed | CRM-bulk-actions team |

## Links

- [Verbose report](report-verbose.md)
- [Grafana — Worker Detailed View](https://prod.grafana.leadconnectorhq.com/d/a04e5483-eb8c-47ef-8198-30147926964c/worker-detailed-view?orgId=1&var-subscriptionId=clickhouse-indexing-bulk-actions-logs-v1-events-sub&var-projectId=highlevel-backend&from=1774611000000&to=1774621800000)
- [Grafana — App Detailed View](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=clickhouse-indexing-bulk-actions-logs-v1-worker&var-cluster=workers-us-central-production-cluster&from=1774611000000&to=1774621800000)
- [GCP Log Explorer — HPA Events](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0A(jsonPayload.reason%3D%22SuccessfulRescale%22%20OR%20jsonPayload.reason%3D%22ScalingReplicaSet%22)%0A%22clickhouse-indexing-bulk-actions-logs-v1%22;timeRange=2026-03-27T12%3A00%3A00Z%2F2026-03-27T14%3A00%3A00Z?project=highlevel-backend)
