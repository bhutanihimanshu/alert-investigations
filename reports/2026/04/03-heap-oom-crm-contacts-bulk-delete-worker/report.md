# Heap out of Memory — crm-contacts-bulk-action-bulk-delete-worker — 2026-04-03

**Author:** Himanshu Bhutani | **Status:** Recurring (same root cause as March 15)

## Summary

| Field | Value |
|-------|-------|
| Alert | [#115041 Heap out of Memory](https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IIVEH71QJV2FH) |
| Service | crm-contacts-bulk-action-bulk-delete-worker |
| Fired | 2026-04-03 23:26 IST (17:56 UTC) |
| Duration | ~77 min of sustained OOM crashes (22:47–00:04 IST / 17:17–18:34 UTC) |
| Impact | 20 V8 heap OOM crashes across 17 unique pods; bulk delete requests delayed during restart cycles. Alert re-fired for ~2.5 hours with no human investigation. |

## Root Cause

**Identical to the March 15 investigation — the action items were never implemented.** The V8 JavaScript heap exhausts its auto-calculated `max-old-space-size` (~1638 MB) during bulk contact deletion. The worker processes 100 contacts per sub-batch (`singleProcessingSize: 100`) with parallel execution via `Promise.allSettled`, each sub-batch loading full contact documents from MongoDB. With `flowControl: 50`, up to 50 messages process concurrently, each spawning multiple parallel sub-batches — causing multiplicative memory pressure. No explicit `maxOldSpaceSize` is set in the deployment config; Helm auto-calculates ~1638 MB from the 2048 MB memory request.

## What Happened

1. **22:47 IST (17:17 UTC)** — First V8 OOM crash on pod wx8j9. Heap exhausted during bulk contact deletion processing.
2. **22:47–00:04 IST** — Sustained OOM storm: 20 crashes across 17 pods in 77 minutes. Pods crash, restart, pick up messages, OOM again.
3. **23:26 IST** — Grafana alert #115041 fires. OnCall (virendra.sharma, anjalica.suman) escalated — no human investigation in thread.
4. **~00:30 IST (Apr 4)** — Alert resolves automatically as bulk delete workload subsides.

## Proof

<details>
<summary>[GCP Logs] 20 V8 OOM crashes across 17 pods in 77 minutes (22:47–00:04 IST)</summary>

> **Verify:** Each entry shows `FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory` or `FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory`.

| Time (IST) | Pod suffix | Error |
|---|---|---|
| 22:47:48 | wx8j9 | Ineffective mark-compacts |
| 22:58:07 | cq9wj | Ineffective mark-compacts |
| 23:14:18 | zbqvv | Ineffective mark-compacts |
| 23:16:46 | lbflc | Ineffective mark-compacts |
| 23:20:52 | qcl8x | Ineffective mark-compacts |
| 23:24:11 | jzlsg | Ineffective mark-compacts |
| 23:25:39 | zbqvv | Ineffective mark-compacts |
| 23:30:14 | hjnrc | Ineffective mark-compacts |
| 23:32:31 | wx8j9 | Ineffective mark-compacts |
| 23:32:43 | 246mg | Ineffective mark-compacts |
| 23:36:04 | s2hqs | Ineffective mark-compacts |
| 23:44:16 | 56lpd | Reached heap limit |
| 23:44:55 | lxczm | Ineffective mark-compacts |
| 23:45:30 | vvb5d | Ineffective mark-compacts |
| 23:49:44 | 87sns | Ineffective mark-compacts |
| 23:59:11 | mk24v | Ineffective mark-compacts |
| 00:00:11 | hhwvp | Reached heap limit |
| 00:03:12 | 87sns | Ineffective mark-compacts |
| 00:04:32 | slnjl | Ineffective mark-compacts |

3 pods crashed twice: 87sns, wx8j9, zbqvv.

GCP query:
```
resource.type="k8s_container"
resource.labels.container_name="crm-contacts-bulk-action-bulk-delete-worker"
textPayload=~"heap out of memory|Ineffective mark-compacts|FATAL ERROR"
```
[Open in Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-contacts-bulk-action-bulk-delete-worker%22%0AtextPayload%3D~%22heap%20out%20of%20memory%7CIneffective%20mark-compacts%7CFATAL%20ERROR%22;timeRange=2026-04-03T17%3A00%3A00Z%2F2026-04-03T19%3A00%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[Prometheus] Termination reason: OOMKilled — confirmed by kube_pod_container_status_last_terminated_reason</summary>

> **Verify:** Prometheus metric `kube_pod_container_status_last_terminated_reason` shows `OOMKilled` for pods d5dwf and qcl8x.

- Memory limit: **2253 MB** (Helm auto-calculated from 2000 MB request × ~1.1)
- Memory request: **2048 MB**
- Termination reason: **OOMKilled** (confirmed)
</details>

<details>
<summary>[7-Day Baseline] 42 OOM crashes in 7 days — worsening trend, Apr 3 is the worst day</summary>

> **Verify:** OOM crash count is escalating week-over-week. This is not a one-off event.

| Day | OOM Crashes |
|---|---|
| 2026-03-28 | 2 |
| 2026-03-30 | 1 |
| 2026-03-31 | 11 |
| 2026-04-01 | 3 |
| 2026-04-02 | 5 |
| **2026-04-03** | **20** |
| **Total** | **42** |

</details>

<details>
<summary>[GCP Logs] "Failed to restore recurring tasks" errors still present — memory-intensive startup routine</summary>

> **Verify:** 10 entries on Apr 3 showing `Failed to restore recurring tasks` from `@platform-core/base-worker` on restart. This contributes additional heap pressure on startup.

GCP query:
```
resource.type="k8s_container"
resource.labels.container_name="crm-contacts-bulk-action-bulk-delete-worker"
jsonPayload.message=~"Failed to restore recurring tasks"
```
</details>

<details>
<summary>[Source Code] singleProcessingSize=100 + parallel sub-batches + full doc load = heap exhaustion</summary>

> **Verify:** `apps/bulk-actions/src/config/bulk_action.ts` line 157: `singleProcessingSize: 100`. `base_bulk_action_worker.ts` runs sub-batches in parallel via `Promise.allSettled`. Each sub-batch calls `Contact.findById()` loading full documents.

- `batchCount: 100` (publisher batch size)
- `singleProcessingSize: 100` (worker sub-batch size)
- `flowControl: 50` (concurrent messages)
- Sub-batches execute in parallel → multiplicative memory
- Each contact: full MongoDB document load (`findById`) + Firestore batch delete + audit log array
</details>

<details>
<summary>[Deployment] No explicit maxOldSpaceSize — Helm auto-calculates ~1638 MB from 2048 MB request</summary>

> **Verify:** `apps/bulk-actions/deployments/production/values.crm-contacts-bulk-action-bulk-delete-worker.yaml` — only `requests.memory: 2000`, no `limits` block, no `maxOldSpaceSize` field.

- Memory request: 2000 (in YAML → 2048 MB deployed)
- Memory limit: 2253 MB (Helm auto ≈ request × 1.1)
- V8 max-old-space-size: ~1638 MB (auto-calculated, not explicitly set)
- No `NODE_OPTIONS` env var in deployment config or Dockerfile
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| **Critical** | Set explicit `maxOldSpaceSize: 2500` in deployment config and increase memory request to 3072 MB | CRM bulk-actions team |
| **High** | Reduce `singleProcessingSize` from 100 to 25 for bulk-delete to lower per-batch heap usage | CRM bulk-actions team |
| **High** | Process sub-batches sequentially (`for...of` instead of `Promise.allSettled`) to prevent multiplicative memory | CRM bulk-actions team |
| **Medium** | Investigate "Failed to restore recurring tasks" in `@platform-core/base-worker` startup routine | Platform team |
| **Medium** | Fix `MONGO_URL_CRM_CONTACTS_STANDARD not found` secret error on some pods | CRM bulk-actions team |
| **Low** | Fix `SequelizeConnectionRefusedError: connect ECONNREFUSED 127.0.0.1:3306` — CloudSQL proxy startup race | CRM bulk-actions team |

**Note:** All Critical and High items were identified in the [March 15 investigation](heap-oom-crm-contacts-bulk-delete-worker-2026-03-15.md) and remain unaddressed. The problem is worsening — 42 OOM crashes in 7 days vs 7 in the March investigation's 7-day window.

## Links

- [Verbose report](report-verbose.md)
- [Grafana alert source](https://prod.grafana.leadconnectorhq.com/alerting/grafana/ae6f8968-3397-408c-ba48-9b2fbc5c1939/view?orgId=1)
- [GCP Log Explorer — OOM errors](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-contacts-bulk-action-bulk-delete-worker%22%0AtextPayload%3D~%22heap%20out%20of%20memory%7CIneffective%20mark-compacts%7CFATAL%20ERROR%22;timeRange=2026-04-03T17%3A00%3A00Z%2F2026-04-03T19%3A00%3A00Z?project=highlevel-backend)
- [App Detailed View — Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=crm-contacts-bulk-action-bulk-delete-worker&var-namespace=default&from=1775228400000&to=1775260800000)
- [Slack alert thread](https://gohighlevel.slack.com/archives/C0315RRNH1B/p1775239014617849)
- [March 15 investigation](heap-oom-crm-contacts-bulk-delete-worker-2026-03-15.md)
