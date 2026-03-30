# Dual Alert Investigation — PubSub Unacked + CronJob Pod Restart — 2026-03-30

**Author:** Himanshu Bhutani
**Generated:** 2026-03-30T11:00:00+05:30

---

## Alert 1: PubSub Unacked Messages above 10k — clickhouse-indexing-bulk-actions-logs-v1-worker

| Field | Value |
|-------|-------|
| Alert type | Pubsub Unacked Messages above 10k |
| Alert ID | #113997 (Grafana OnCall I54PP7YVLWLVV) |
| Subscription | clickhouse-indexing-bulk-actions-logs-v1-events-sub |
| Cluster | workers-us-central-production-cluster |
| Fired | 06:58 IST (01:28 UTC), 2026-03-30 |
| Unacked count | 184,887 |
| Status | Auto-resolved |
| Channel | #alerts-crm (via CRM-bulk-actions) |
| Past occurrences | Mar 27, Mar 29 (same root cause) |

### Investigation Findings

#### Evidence: Grafana — PubSub Metrics

<details>
<summary>Worker Detailed View — Subscription stats showing backlog spike and recovery</summary>

> **What to look for:** The Oldest Unacked Message timeseries should show a spike to ~9.3 min (561s) around 06:58 IST, then recovery. The Ack Count timeseries should show acks staying positive throughout (peak ~141k acks/min at 06:50 IST). Unacked messages gauge should show the peak at ~749k.

![Worker Detailed View](screenshots/001-worker-detailed-view-subscription-stats-unacked-messages-ack.png)

[Open in Grafana — Worker Detailed View](https://prod.grafana.leadconnectorhq.com/d/a04e5483-eb8c-47ef-8198-30147926964c/worker-detailed-view?orgId=1&var-subscriptionId=clickhouse-indexing-bulk-actions-logs-v1-events-sub&from=1774830000000&to=1774842000000)

</details>

#### Evidence: Grafana — Pod Health

<details>
<summary>App Detailed View — Pod count oscillation (5→20→5) with zero restarts</summary>

> **What to look for:** The pod count panel should show a step function: steady at 5, jump to 20 at ~06:47 IST, drop back to 5 at ~07:04 IST, jump to 20 again at ~09:18 IST, drop to 5 at ~09:32 IST. The pod restarts panel should show zero restarts.

![Pod Count Panel](screenshots/002-app-detailed-view-pod-count-restarts-clickhouse-indexing-bul.png)

**Context (filters + time range):**
![Context](screenshots/002-app-detailed-view-pod-count-restarts-clickhouse-indexing-bul-context.png)

[Open in Grafana — App Detailed View](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-datasource=ber8nnhvgsjy8f&var-container=clickhouse-indexing-bulk-actions-logs-v1-worker&var-cluster=workers-us-central-production-cluster&from=1774830000000&to=1774845000000)

**Resource usage (from Prometheus):**
- CPU: max ~0.136 cores per pod (very low — confirms I/O-bound workload)
- Memory: max ~152 MiB per pod (well within limits)
- No pod restarts in the entire window

</details>

#### Evidence: GCP Logs — HPA Scaling Events

<details>
<summary>HPA oscillated twice in 4 hours — PubSub vs CPU metric conflict</summary>

> **What to look for:** The logs show two complete scale-up/scale-down cycles. In the first cycle, the scale-down reason is "cpu resource utilization below target" — this is the conflict. The worker is I/O-bound (CPU stays low even under load), so the CPU metric falsely signals that pods are idle.

**Full HPA timeline:**

| Time (IST) | Event | Trigger |
|---|---|---|
| 06:46:24 | 5 → 10 | PubSub `num_undelivered_messages` above target |
| 06:46:56 | 10 → 12 | PubSub `num_undelivered_messages` above target |
| 06:47:19 | 12 → 20 | PubSub `num_undelivered_messages` above target |
| **07:04:31** | **20 → 5** | **CPU utilization below target** (PREMATURE) |
| 09:18:08 | 5 → 10 | PubSub `num_undelivered_messages` above target |
| 09:18:40 | 10 → 11 | PubSub `num_undelivered_messages` above target |
| 09:19:01 | 11 → 20 | PubSub `num_undelivered_messages` above target |
| 09:32:05 | 20 → 5 | PubSub `num_undelivered_messages` below target (natural) |

```
resource.type="k8s_cluster"
resource.labels.cluster_name="workers-us-central-production-cluster"
jsonPayload.involvedObject.name=~"clickhouse-indexing-bulk-actions"
jsonPayload.reason=~"SuccessfulRescale|ScalingReplicaSet"
```

![HPA Events](screenshots/001-gcp-hpa-scaling-events-2-cycles-of-5-20-5-in-4-hours.png)

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0Aresource.labels.cluster_name%3D%22workers-us-central-production-cluster%22%0AjsonPayload.involvedObject.name%3D~%22clickhouse-indexing-bulk-actions%22%0AjsonPayload.reason%3D~%22SuccessfulRescale%7CScalingReplicaSet%22;timeRange=2026-03-30T00%3A30%3A00Z%2F2026-03-30T04%3A30%3A00Z?project=highlevel-backend)

</details>

#### Evidence: GCP Logs — Subscriber Closed Errors

<details>
<summary>Subscriber Closed errors at exactly 07:04:31 IST — during HPA scale-down</summary>

> **What to look for:** All errors have the exact timestamp `2026-03-30T01:34:31Z` (07:04:31 IST), which matches the HPA scale-down event to the second. The error is `INVALID : Subscriber closed` from `@google-cloud/pubsub` — pods being terminated mid-processing cannot complete their ack operations.

**Sample error:**
```
unhandledRejectionError {"stack":"Error: INVALID : Subscriber closed
    at AckQueue.add (.../message-queues.js:130:23)
    at Subscriber.ack (.../subscriber.js:597:42)
    at Message.ack (.../subscriber.js:318:30)
    at Object.ack (.../clickhouse-indexing-worker.js:152:25)
    at ClickhouseIndexingWorker.processBatchMessage (...)"}
```

```
resource.type="k8s_container"
resource.labels.container_name="clickhouse-indexing-bulk-actions-logs-v1-worker"
textPayload=~"Subscriber closed"
severity>=ERROR
```

![Subscriber Closed Errors](screenshots/002-gcp-subscriber-closed-errors-at-01-34-31-utc-exactly-when-hpa-sc.png)

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22clickhouse-indexing-bulk-actions-logs-v1-worker%22%0AtextPayload%3D~%22Subscriber%20closed%22%0Aseverity%3E%3DERROR;timeRange=2026-03-30T01%3A20%3A00Z%2F2026-03-30T01%3A45%3A00Z?project=highlevel-backend)

</details>

### Cross-Validation — Alert 1

| Signal | Source | Agrees? |
|--------|--------|---------|
| HPA scaled 5→20→5 | GCP K8s cluster events | ✅ Two cycles confirmed |
| Scale-down triggered by CPU | GCP K8s cluster events | ✅ "cpu resource utilization below target" |
| Subscriber closed at scale-down time | GCP container logs | ✅ Exact timestamp match (01:34:31Z) |
| Pod count oscillation | Grafana Prometheus | ✅ Step function 5→20→5 |
| No pod restarts | Grafana Prometheus | ✅ Zero restarts |
| Backlog recovered after scale-up | Cloud Monitoring PubSub | ✅ Ack rate stayed positive, backlog drained |
| Same pattern as Mar 27/29 | Investigations DB | ✅ Investigation #48 — same root cause |

**Confidence: HIGH** — All 7 signals agree. This is a confirmed recurrence of the HPA thrashing pattern documented on Mar 27 and Mar 29.

### Root Cause — Alert 1

**HPA thrashing due to conflicting scaling metrics.** The worker HPA has two metrics:
1. **PubSub `num_undelivered_messages`** — scales UP when backlog exceeds target
2. **CPU utilization** — scales DOWN when CPU drops below target

The worker is **I/O-bound** (CPU stays at ~0.136 cores even at 20 pods processing 141k messages/min). When the PubSub metric triggers scale-up, the 20 pods quickly drain the backlog. But CPU stays low because the work is network-bound (PubSub pull + ClickHouse insert). The CPU metric then triggers a premature scale-down to 5 pods, which causes the backlog to rebuild.

**Causal chain:**
1. **06:46 IST** — Backlog builds → HPA scales 5→20 on PubSub metric
2. **06:50 IST** — 20 pods process backlog at ~141k acks/min; CPU stays at ~0.14 cores
3. **07:04 IST** — CPU below target → HPA scales 20→5 (premature — backlog not fully cleared)
4. **07:04:31 IST** — Terminating pods emit `Subscriber closed` errors (acks fail on shutting-down subscribers)
5. **06:58 IST** — Alert fires during the scale-up phase when backlog was at peak (184,887)
6. Backlog eventually drains → alert auto-resolves

<details>
<summary>Detailed timeline — full HPA + PubSub event log</summary>

| Time (IST) | Source | Event |
|---|---|---|
| 06:46:24 | K8s HPA | SuccessfulRescale: 5→10 on PubSub undelivered above target |
| 06:46:56 | K8s HPA | SuccessfulRescale: 10→12 |
| 06:47:19 | K8s HPA | SuccessfulRescale: 12→20 |
| ~06:50 | Cloud Monitoring | Ack rate peaks at ~141k acks/min |
| ~06:55 | Cloud Monitoring | Backlog at ~90k undelivered |
| **06:58** | **Grafana Alert** | **Pubsub Unacked Messages above 10k — 184,887** |
| ~07:00 | Cloud Monitoring | Oldest unacked peaks at ~561s (9.3 min) |
| **07:04:31** | K8s HPA | SuccessfulRescale: **20→5 on CPU below target** |
| **07:04:31** | GCP Logs | Multiple `INVALID : Subscriber closed` errors |
| ~07:20 | Cloud Monitoring | Backlog continues draining at reduced rate (5 pods) |
| ~08:00 | Cloud Monitoring | Backlog near zero |
| 09:18:08 | K8s HPA | SuccessfulRescale: 5→10 (second cycle) |
| 09:19:01 | K8s HPA | SuccessfulRescale: 11→20 |
| 09:32:05 | K8s HPA | SuccessfulRescale: 20→5 (PubSub below target — natural) |

</details>

---

## Alert 2: PodRestartsAboveThreshold — crm-marketplace-update-installation-counts-cron

| Field | Value |
|-------|-------|
| Alert type | PodRestartsAboveThreshold |
| Alert ID | #114003 (Grafana OnCall ISMVJL77Q5RNC) |
| Workload | crm-marketplace-update-installation-counts-cron-29580690 |
| Container | crm-marketplace-update-installation-counts-cron |
| Cluster | workers-us-central-production-cluster |
| Fired | 09:14 IST (03:44 UTC), 2026-03-30 |
| Current value | 1 restart |
| Threshold | 1 |
| Status | Acknowledged by Karan Kaneria |
| Channel | #alerts-crm (via CRM-marketplace) |
| Past occurrences | Mar 15, Mar 16 (same pattern, investigations #21, #23) |

### Investigation Findings

#### Evidence: GCP Logs — CronJob Events

<details>
<summary>CronJob lifecycle — Created 09:00 IST, Completed 10:16 IST (76 min)</summary>

> **What to look for:** Only two events: `SuccessfulCreate` at 09:00 IST and `Completed` at 10:16 IST. No `Failed`, `Backoff`, or `Killing` events. The 76-minute runtime matches the known pattern (API takes 27–37 min, plus restart overhead).

| Time (IST) | Event | Details |
|---|---|---|
| 09:00:00 | SuccessfulCreate | Pod crm-marketplace-update-installation-counts-cron-29580690-pjfrd created |
| 10:16:09 | Completed | Job completed |

```
resource.type="k8s_cluster"
resource.labels.cluster_name="workers-us-central-production-cluster"
jsonPayload.involvedObject.name=~"crm-marketplace-update-installation-counts-cron-29580690"
```

![CronJob Events](screenshots/003-gcp-cronjob-events-created-03-30-utc-completed-04-46-utc-76-min-.png)

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0Aresource.labels.cluster_name%3D%22workers-us-central-production-cluster%22%0AjsonPayload.involvedObject.name%3D~%22crm-marketplace-update-installation-counts-cron-29580690%22;timeRange=2026-03-30T03%3A00%3A00Z%2F2026-03-30T05%3A00%3A00Z?project=highlevel-backend)

</details>

#### Evidence: Grafana — Pod Restarts

<details>
<summary>Single restart on CronJob container at 09:14 IST — no crash loop</summary>

> **What to look for:** Exactly 1 restart bar at 09:14 IST on pod `...-pjfrd`. No other restarts in the window. Pod count toggles between 3 and 4 (normal for short-lived Job pods + sidecars).

![Cron Restarts](screenshots/003-app-detailed-view-pod-restarts-crm-marketplace-update-instal.png)

**Context (filters + time range):**
![Context](screenshots/003-app-detailed-view-pod-restarts-crm-marketplace-update-instal-context.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-datasource=ber8nnhvgsjy8f&var-container=crm-marketplace-update-installation-counts-cron&var-cluster=workers-us-central-production-cluster&from=1774838000000&to=1774846000000)

</details>

#### Evidence: GCP Logs — No Errors

No WARNING or ERROR logs found for the CronJob container in the 02:00–05:00 UTC window. The restart is a timeout-induced lifecycle event, not an application crash.

### Cross-Validation — Alert 2

| Signal | Source | Agrees? |
|--------|--------|---------|
| Single restart | Grafana Prometheus | ✅ Exactly 1 restart |
| Job completed | GCP K8s cluster events | ✅ `reason: Completed` at 10:16 IST |
| No application errors | GCP container logs | ✅ Zero WARNING+ logs |
| CronJob schedule hash in name | Alert message | ✅ `-29580690` suffix |
| Same pattern as Mar 15/16 | Investigations DB | ✅ Investigations #21, #23 |

**Confidence: HIGH** — This is a confirmed recurrence of the CronJob timeout false positive.

### Root Cause — Alert 2

**CronJob container timeout → restart → successful completion.** The CronJob makes an HTTP POST to an internal API that runs heavy MongoDB aggregations on installation counts. The API takes 27–76 minutes. The first container is killed by a timeout mechanism (likely Istio sidecar timeout or container-level deadline) at ~20 min. K8s restarts it (`restartPolicy: OnFailure`), and the second attempt completes. The PodRestartsAboveThreshold alert fires for this expected single restart.

---

## Cross-Alert Assessment

These two alerts are **independent incidents** that happened to fire in the same Slack channel within ~2 hours:

| Factor | Alert 1 | Alert 2 |
|--------|---------|---------|
| Workload | clickhouse-indexing-bulk-actions | marketplace-update-installation-counts-cron |
| Alert type | PubSub unacked | Pod restart |
| Root cause | HPA metric conflict | CronJob timeout |
| Recurrence | 3rd time in 4 days | 10+ times since Feb 21 |
| Relationship | None | None |

**No deployment, infrastructure event, or shared dependency connects them.** The GCP Places API quota alert (#113999) in the same window is also unrelated.

---

<details>
<summary>Probable noise — transient errors during disruption (not root cause)</summary>

| Time (IST) | Pattern | Why it's noise |
|---|---|---|
| 07:04:31 | `Subscriber closed` errors in clickhouse worker | Expected side-effect of HPA scale-down — pods terminated mid-processing |
| 07:44:59 | `[PLATFORM_CORE_CLICKHOUSE] Clickhouse error` (single occurrence) | Isolated ClickHouse error, not correlated with the backlog spike |

</details>

---

## Action Items

| Priority | Action | Owner | Alert |
|----------|--------|-------|-------|
| **Medium** | Fix HPA metric conflict: either remove CPU-based scale-down metric or add `scaleDown.stabilizationWindowSeconds: 300` to prevent premature scale-down while backlog exists | CRM Bulk Actions / Platform | Alert 1 |
| **Medium** | Consider using `behavior.scaleDown.policies` to limit pods-per-minute in scale-down events | CRM Bulk Actions / Platform | Alert 1 |
| **Low** | Optimize `update-installation-counts` API to complete within 20 min, or increase the Istio sidecar / container timeout | CRM Marketplace | Alert 2 |
| **Low** | Suppress PodRestartsAboveThreshold for CronJob workloads where `threshold=1` and the Job completes successfully | Platform | Alert 2 |

---

## Deployment Details

### clickhouse-indexing-bulk-actions-logs-v1-worker

| Setting | Value |
|---------|-------|
| HPA min replicas | 5 |
| HPA max replicas | 20 |
| HPA metrics | PubSub `num_undelivered_messages` (up), CPU utilization (down) |
| CPU per pod | ~0.14 cores (observed peak) |
| Memory per pod | ~152 MiB (observed peak) |
| Cluster | workers-us-central-production-cluster |

### crm-marketplace-update-installation-counts-cron

| Setting | Value |
|---------|-------|
| Schedule | CronJob (hash 29580690) |
| Restart policy | OnFailure |
| API runtime | 27–76 minutes (varies by data volume) |
| Cluster | workers-us-central-production-cluster |

---

## Correlated Alerts (from Alert Correlator)

| Time (IST) | Alert | Channel | Related? |
|---|---|---|---|
| 06:58 | PubSub Unacked 10k — clickhouse bulk actions | #alerts-crm | **This alert (Alert 1)** |
| 08:24 | GCP Quota Exceeded — Places API | #alerts-platform | No — different service |
| 09:14 | PodRestartsAboveThreshold — marketplace cron | #alerts-crm | **This alert (Alert 2)** |

No deployment found in `#core-crm-conversations-internal` within 2 hours before either alert.
