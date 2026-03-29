# PodRestartsAboveThreshold — crm-marketplace-update-installation-counts-cron — 2026-03-29

**Author:** Himanshu Bhutani | **Status:** Auto-resolved (Job completed)

## Summary

| Field | Value |
|-------|-------|
| Alert | PodRestartsAboveThreshold (#113953) |
| Service | crm-marketplace-update-installation-counts-cron |
| Cluster | workers-us-central-production-cluster |
| Fired | 09:09 IST (03:39 UTC), 2026-03-29 |
| Duration | ~7 min (first container died at 09:07 IST, second started immediately) |
| Impact | None — Job completed successfully on second attempt at 10:06 IST |

## Root Cause

**Known false positive — CronJob container timeout → restart.** The CronJob makes a long-running HTTP POST to `/marketplace/core/update/installation-counts`. The first container received a 503 (Envoy upstream connection termination) after ~7 min. K8s restarted the container (`restartPolicy: OnFailure`), and the second attempt completed in ~60 min with HTTP 201. The Job reached "Completed" status. This alert fires every 1-2 days (41+ occurrences since Feb 21, 8 in the last 12 days).

**Confidence:** HIGH — 5/5 sources agree, matches documented known root cause pattern, 2 prior investigations confirmed identical behavior.

## Proof

<details>
<summary>[Grafana] Single pod restart at ~09:07 IST — restart count = 1</summary>

> **Verify:** A single green dot appears at ~09:05 IST on the Pod Restarts chart. Y-axis max is 1. Pod name is `crm-marketplace-update-installation-counts-cron-29579250-l6kvx`.

![Pod Restarts](screenshots/001-app-detailed-view-pod-restarts-panel-36.png)

**Context (filters + time range):**
![Context](screenshots/001-app-detailed-view-pod-restarts-panel-36-context.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=workers-us-central-production-cluster&var-container=crm-marketplace-update-installation-counts-cron&from=1774751400000&to=1774762200000&viewPanel=36)
</details>

<details>
<summary>[GCP] K8s Job lifecycle — Created at 09:00 IST, Completed at 10:06 IST</summary>

> **Verify:** Two events: "Created pod" at 09:00:00 and "Job completed" at 10:06:46. The Job reached Completed status despite the restart.

![K8s Cluster Events](screenshots/001-gcp-k8s-cluster-events-job-created-completed.png)

```
resource.type="k8s_cluster"
resource.labels.project_id="highlevel-backend"
jsonPayload.involvedObject.name=~"crm-marketplace-update-installation-counts-cron-29579250"
```

[Open in Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0Aresource.labels.project_id%3D%22highlevel-backend%22%0AjsonPayload.involvedObject.name%3D~%22crm-marketplace-update-installation-counts-cron-29579250%22;timeRange=2026-03-29T03%3A00%3A00Z%2F2026-03-29T05%3A00%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[GCP] Container logs — 503 at 09:07 IST (first attempt), 201 at 10:06 IST (second attempt)</summary>

> **Verify:** First cluster of logs at 09:06:59 shows "HTTP status: 503", "upstream connect error or disconnect/reset before headers. reset reason: connection termination", and "Request failed". Second cluster at 10:06:44 shows "HTTP status: 201" and "HTTP/1.1 201 Created".

![Container Logs](screenshots/002-gcp-container-logs-request-failed-09-07-ist-then-http-201-succes.png)

```
resource.type="k8s_container"
resource.labels.container_name="crm-marketplace-update-installation-counts-cron"
resource.labels.project_id="highlevel-backend"
textPayload=~"upstream connect error|Request failed|Installation counters|HTTP"
```

[Open in Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-marketplace-update-installation-counts-cron%22%0Aresource.labels.project_id%3D%22highlevel-backend%22%0AtextPayload%3D~%22upstream%20connect%20error%7CRequest%20failed%7CInstallation%20counters%7CHTTP%22;timeRange=2026-03-29T03%3A30%3A00Z%2F2026-03-29T04%3A40%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[Slack] Alert thread — Smitha asked Ved to check OOM vs timeout</summary>

> **Verify:** Thread reply from Smitha: "can you increase the memory as suggested by Reaper and check if it's an actual OOM and not a timeout as we thought?"

Alert thread: [#alerts-crm permalink](https://gohighlevel.slack.com/archives/C0315RRNH1B/p1774755552547879)
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| Medium | Optimize `/marketplace/core/update/installation-counts` API to reduce 60-min response time (batching/streaming instead of loading all installations in memory) | Marketplace team |
| Medium | Increase Istio/Envoy upstream timeout for this CronJob's requests (first attempt fails at ~7 min due to connection termination) | Marketplace team |
| Low | Suppress PodRestartsAboveThreshold for CronJob workloads where a single restart is expected behavior | Platform team |
| Low | Validate OOM theory — increase memory limit and monitor whether first-attempt failure changes (per Smitha's request to Ved) | Marketplace team |

## Links

- [Verbose report](report-verbose.md)
- [Grafana — App Detailed View](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=workers-us-central-production-cluster&var-container=crm-marketplace-update-installation-counts-cron&from=1774751400000&to=1774762200000)
- [Grafana OnCall alert group](https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IG45WUL3SR8UA)
