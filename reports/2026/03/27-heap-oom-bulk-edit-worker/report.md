# Heap OOM — bulk-edit-worker — 2026-03-27

**Author:** Himanshu Bhutani | **Status:** Recurring (4th occurrence in 2 months)

## Summary

| Field | Value |
|-------|-------|
| Alert 1 | PodRestartsAboveThreshold (#113838) |
| Alert 2 | Heap out of Memory (#113840) |
| Service | `crm-contacts-bulk-action-bulk-edit-worker` |
| Cluster | `workers-us-central-production-cluster` |
| Fired | 20:41 IST (15:11 UTC) — pod restarts; 20:50 IST (15:20 UTC) — heap OOM |
| Duration | ~25 min per wave (two waves: 20:48–21:14 IST and 21:53–22:00+ IST) |
| Impact | 54 main container restarts, 20 V8 heap OOM crashes, HPA maxed at 200/200 pods, cluster FailedScheduling |

## Root Cause

**Load-driven JavaScript heap OOM.** Two massive PubSub load surges scaled the worker from 10 → 200 pods. The V8 heap limit (`--max-old-space-size=760MB`) with only ~85MB headroom to the container memory limit (845Mi) is too tight for concurrent bulk-edit HTTP calls. Pods processing large batches exhausted the V8 heap and crashed with `FATAL ERROR: Reached heap limit`. This is NOT a memory leak — pods return to 250–300MB after restart.

## What Happened

1. **20:37 IST** (15:07 UTC) — PubSub backlog surge triggered HPA scale-up from 10 → 200 pods in ~4 minutes.
2. **20:48 IST** (15:18 UTC) — First wave of V8 heap OOM crashes: 12 pods crashed in 13 minutes as bulk-edit payloads exhausted the 760MB V8 heap.
3. **21:11 IST** (15:41 UTC) — First wave subsides. HPA scales down to 17 pods.
4. **21:53 IST** (16:23 UTC) — Second PubSub surge triggers HPA back to 200 pods. 6 more OOM crashes in 7 minutes.

## Proof

<details>
<summary>[Grafana] Memory at 100% of limit — 37 pods peaked at 800–845 MB</summary>

> **Verify:** Memory by Pod graph shows multiple pod traces hitting the 845Mi limit line. Peak: 844.9 MB on pod `9d5c4` at 20:56 IST.

![Memory by Pod](screenshots/001-memory-by-pod-peaks-at-844-9-mb-100-of-845mi-limit.png)
[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=crm-contacts-bulk-action-bulk-edit-worker&from=1774620000000&to=1774630800000&viewPanel=30)
</details>

<details>
<summary>[Grafana] 54 main container restarts — OOMKilled dominant</summary>

> **Verify:** Pod Restarts panel shows two distinct restart waves. Termination Reason shows 23 OOMKilled + 5 Error.

![Pod Restarts](screenshots/002-pod-restarts-two-restart-waves-matching-oom-crash-windows.png)
![Termination Reason](screenshots/003-termination-reason-23-oomkilled-5-error.png)
[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=crm-contacts-bulk-action-bulk-edit-worker&from=1774620000000&to=1774630800000&viewPanel=36)
</details>

<details>
<summary>[Grafana] HPA maxed at 200/200 pods — horizontal scaling exhausted</summary>

> **Verify:** Pod Count graph jumps from 10 to 200 (HPA max) within ~4 minutes during each surge.

![Pod Count](screenshots/005-pod-count-hpa-scaled-10-200-max-during-load-surge.png)
[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=crm-contacts-bulk-action-bulk-edit-worker&from=1774620000000&to=1774630800000&viewPanel=32)
</details>

<details>
<summary>[GCP] 20 FATAL ERROR heap OOM crashes across 20 pods</summary>

> **Verify:** Log entries show `FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory` with timestamps clustering at 15:18–15:31 UTC and 16:23–16:30 UTC.

![OOM Logs](screenshots/001-gcp-fatal-error-reached-heap-limit-20-crashes-in-two-waves.png)

```
resource.type="k8s_container"
resource.labels.container_name="crm-contacts-bulk-action-bulk-edit-worker"
textPayload=~"FATAL ERROR.*heap"
severity>=ERROR
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-contacts-bulk-action-bulk-edit-worker%22%0AtextPayload%3D~%22FATAL%20ERROR.%2Aheap%22%0Aseverity%3E%3DERROR;timeRange=2026-03-27T14%3A00%3A00Z%2F2026-03-27T16%3A30%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[Config] V8 heap (760MB) + 85MB headroom = 845Mi limit — too tight</summary>

> **Verify:** Container limit is 845Mi. `NODE_OPTIONS=--max-old-space-size=760`. That leaves only ~85MB for Node.js native overhead, buffers, and concurrent HTTP response data.

| Component | Value |
|-----------|-------|
| Container memory limit | 845Mi |
| V8 `--max-old-space-size` | 760MB |
| Headroom | ~85MB |
| CPU request / limit | 250m / 275m |
| flowControl (concurrent messages) | 10 |

Compare: the similar `bulk-delete` worker gets 1024Mi request (vs 756Mi for bulk-edit).
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| **High** | Increase memory limit to ~1.5–2Gi and `--max-old-space-size` to ~1200–1500MB | CRM bulk-actions team |
| **High** | Increase CPU limit from 275m to 500m+ to reduce throttling (currently 52% of pods throttled) | CRM bulk-actions team |
| **Medium** | Reduce `flowControl` from 10 → 5 to halve concurrent memory usage as interim mitigation | CRM bulk-actions team |
| **Medium** | Add streaming/chunked processing for large batch payloads instead of holding all in memory | CRM bulk-actions team |
| **Low** | Add Node.js Prometheus metrics (`prom-client`) for heap/GC observability | CRM bulk-actions team |

## Links

- [Verbose report](report-verbose.md)
- [Grafana App Detailed View](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=crm-contacts-bulk-action-bulk-edit-worker&from=1774620000000&to=1774630800000)
- [GCP OOM Logs](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-contacts-bulk-action-bulk-edit-worker%22%0AtextPayload%3D~%22FATAL%20ERROR.%2Aheap%22%0Aseverity%3E%3DERROR;timeRange=2026-03-27T14%3A00%3A00Z%2F2026-03-27T16%3A30%3A00Z?project=highlevel-backend)
