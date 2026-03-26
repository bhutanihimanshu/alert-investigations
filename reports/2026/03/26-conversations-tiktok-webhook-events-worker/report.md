# PodRestartsAboveThreshold — conversations-tiktok-webhook-events-worker — 2026-03-26

**Author:** Himanshu Bhutani | **Status:** Recurring (prior alerts Mar 16, 17)

## Summary

| Field | Value |
|-------|-------|
| Alert | #113689 PodRestartsAboveThreshold |
| Service | conversations-tiktok-webhook-events-worker |
| Cluster | workers-us-central-production-cluster |
| Fired | 11:02 IST (05:32 UTC), 2026-03-26 |
| Duration | ~25 min crash loop (11:00–11:25 IST) |
| Impact | 3 pods entered CrashLoopBackOff; HPA scaled 2→8 pods to compensate |

## Root Cause

**JavaScript heap out of memory.** The V8 heap limit (`--max-old-space-size=760` MiB) is exhausted under normal processing load. GC becomes ineffective → `FATAL ERROR: Ineffective mark-compacts near heap limit` → process exits with SIGSEGV (code 139). Multiple pods (rhwmz, zvtw9, 4m4h8) entered the same crash cycle, confirming this is fleet-wide — not a single-pod anomaly.

## Proof

<details>
<summary>[Grafana] Memory peaked at 795 MiB / 845 MiB limit (94%) at 11:24 IST</summary>

> **Verify:** Memory by Pod panel shows lines climbing into the high 700s MiB range, approaching the container limit reference line at 845 MiB.

![Memory by Pod](screenshots/002-app-detailed-view-memory-by-pod.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=workers-us-central-production-cluster&var-container=conversations-tiktok-webhook-events-worker&from=1774499400000&to=1774506600000&viewPanel=30)
</details>

<details>
<summary>[Grafana] Pod Restarts — rhwmz 4 restarts, 4m4h8 3 restarts, zvtw9 2 restarts</summary>

> **Verify:** Pod Restarts panel shows staircase pattern with restart counter incrementing from 05:32 through 05:55 UTC across 3 pods.

![Pod Restarts](screenshots/003-app-detailed-view-pod-restarts.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=workers-us-central-production-cluster&var-container=conversations-tiktok-webhook-events-worker&from=1774499400000&to=1774506600000&viewPanel=36)
</details>

<details>
<summary>[GCP stderr] FATAL ERROR: Ineffective mark-compacts near heap limit — first at 11:00 IST</summary>

> **Verify:** Log entries show `FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory` on multiple pods starting at 05:30:53 UTC.

![Heap OOM logs](screenshots/001-gcp-javascript-heap-out-of-memory-fatal-error-in-stderr.png)

GCP query:
```
resource.type="k8s_container"
resource.labels.container_name="conversations-tiktok-webhook-events-worker"
textPayload=~"FATAL ERROR"
```

[Open in Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-tiktok-webhook-events-worker%22%0AtextPayload%3D~%22FATAL%20ERROR%22;timeRange=2026-03-26T05%3A00%3A00Z%2F2026-03-26T06%3A00%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[Kubelet] CrashLoopBackOff — ContainerDied at 11:00:54 IST, back-off 10s→20s</summary>

> **Verify:** Kubelet logs show PLEG ContainerDied → ContainerStarted → CrashLoopBackOff cycle on pod rhwmz starting 05:30:54 UTC.

![Kubelet logs](screenshots/002-gcp-kubelet-containerdied-and-crashloopbackoff-events.png)

GCP query:
```
logName="projects/highlevel-backend/logs/kubelet"
"conversations-tiktok-webhook-events-worker"
```

[Open in Log Explorer](https://console.cloud.google.com/logs/query;query=logName%3D%22projects%2Fhighlevel-backend%2Flogs%2Fkubelet%22%0A%22conversations-tiktok-webhook-events-worker%22;timeRange=2026-03-26T05%3A25%3A00Z%2F2026-03-26T05%3A40%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[Grafana] NodeJS heap used — rhwmz at 723 MiB (95% of 760 MiB V8 limit)</summary>

> **Verify:** Heap used graph shows sawtooth pattern — linear growth to ~723 MiB then drop (crash) and regrowth.

![NodeJS Heap](screenshots/004-nodejs-application-heap-used.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/PTSqcpJWka/nodejs-application?orgId=1&var-datasource=ber8nnhvgsjy8f&var-app=conversations-tiktok-webhook-events-worker&from=1774499400000&to=1774506600000)
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| High | Increase memory limit and `maxOldSpaceSize` — current 845 MiB limit / 760 MiB heap is too tight | CRM Conversations |
| Medium | Align `maxOldSpaceSize` in deployment YAML (currently 550) with actual runtime (760) | CRM Conversations |
| Medium | Fix Redis target — `ECONNREFUSED` to `127.0.0.1:6379` suggests no Redis sidecar | CRM Conversations |
| Low | Profile memory to identify leak vs genuine workload growth | CRM Conversations |

## Links

- [Verbose report](report-verbose.md)
- [App Detailed View](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=workers-us-central-production-cluster&var-container=conversations-tiktok-webhook-events-worker&from=1774499400000&to=1774506600000)
- [NodeJS Application Dashboard](https://prod.grafana.leadconnectorhq.com/d/PTSqcpJWka/nodejs-application?orgId=1&var-datasource=ber8nnhvgsjy8f&var-app=conversations-tiktok-webhook-events-worker&from=1774499400000&to=1774506600000)
- [GCP Log Explorer — heap OOM](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-tiktok-webhook-events-worker%22%0AtextPayload%3D~%22FATAL%20ERROR%22;timeRange=2026-03-26T05%3A00%3A00Z%2F2026-03-26T06%3A00%3A00Z?project=highlevel-backend)
- [Slack alert thread](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1774503154039219)
- [ClickUp ticket (Reaper)](https://app.clickup.com/t/86d2eb2j9)
