# PodRestartsAboveThreshold — crm-contacts-bulk-whatsapp-worker — 2026-03-27

**Author:** Himanshu Bhutani | **Status:** Auto-resolved

## Summary

| Field | Value |
|-------|-------|
| Alert | PodRestartsAboveThreshold (#113832) |
| Service | crm-contacts-bulk-whatsapp-worker |
| Container | istio-proxy |
| Cluster | workers-us-central-production-cluster |
| Fired | 18:48 IST (13:18 UTC) |
| Duration | ~2 min (13:15–13:17 UTC). Auto-resolved in 4-6s per pod. |
| Impact | 3/3 pods restarted within 2 min. App container unaffected — no message processing disruption beyond brief sidecar unavailability. |

## Root Cause

**istio-proxy OOMKilled (exit code 137)** on all 3 pods. The Envoy sidecar's 500Mi memory limit was exceeded after gradual memory growth (pods had been running 8h to 4.6 days). This is a chronic pattern — same workload has alerted on Feb 9, Mar 3, 10, 11, and 27.

## Proof

<details>
<summary>[kubectl] OOMKilled — exit code 137, 500Mi limit on all 3 pods</summary>

> **Verify:** `kubectl describe pod` shows Last State: Terminated, Reason: OOMKilled, Exit Code: 137 on the istio-proxy container. Memory limit: 500Mi, request: 400Mi.

| Pod | OOM Time (IST) | Exit Code | Memory Limit |
|-----|----------------|-----------|--------------|
| fnlcc | 18:45:21 | 137 | 500Mi |
| bjd5m | 18:46:18 | 137 | 500Mi |
| sddm6 | 18:47:14 | 137 | 500Mi |

```bash
kubectl describe pod crm-contacts-bulk-whatsapp-worker-6d5c6fccbb-fnlcc -n default | grep -A 10 "istio-proxy"
```
</details>

<details>
<summary>[GCP Kubelet] Container lifecycle — OOM → restart → ready in 4-6s</summary>

> **Verify:** Kubelet logs show the sequence: readiness probe fails → ContainerDied (PLEG) → ContainerStarted → readiness transitions back to ready.

![Kubelet Logs](screenshots/001-gcp-kubelet-logs-istio-proxy-oomkilled-and-restart.png)

```
logName="projects/highlevel-backend/logs/kubelet"
"crm-contacts-bulk-whatsapp-worker"
```
[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=logName%3D%22projects%2Fhighlevel-backend%2Flogs%2Fkubelet%22%0A%22crm-contacts-bulk-whatsapp-worker%22;timeRange=2026-03-27T13%3A14%3A00Z%2F2026-03-27T13%3A20%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[GCP K8s Events] Readiness probe failures on istio-proxy (fieldPath confirms)</summary>

> **Verify:** Both events show `involvedObject.fieldPath: spec.containers{istio-proxy}` — the sidecar probe failed, not the app container. Error messages: "connection reset by peer" and "EOF" on port 15021.

![K8s Pod Events](screenshots/002-gcp-k8s-pod-events-readiness-probe-failures-on-istio-proxy.png)

```
resource.type="k8s_pod"
resource.labels.pod_name=~"crm-contacts-bulk-whatsapp-worker.*"
resource.labels.cluster_name="workers-us-central-production-cluster"
jsonPayload.reason=~"Unhealthy|Killing"
```
[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.pod_name%3D~%22crm-contacts-bulk-whatsapp-worker.%2A%22%0Aresource.labels.cluster_name%3D%22workers-us-central-production-cluster%22%0AjsonPayload.reason%3D~%22Unhealthy%7CKilling%22;timeRange=2026-03-27T13%3A14%3A00Z%2F2026-03-27T13%3A20%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[Slack] Correlated istio-proxy alert — contacts-bulk-import-contact-worker at 17:50 IST</summary>

> **Verify:** Another istio-proxy OOMKilled alert fired ~1 hour earlier on a sibling worker in the same cluster. That alert had a 384Mi limit; this one has 500Mi. Both are insufficient.

- contacts-bulk-import-contact-worker: 12:20 UTC (17:50 IST) — istio-proxy, 8 restarts
- crm-contacts-bulk-whatsapp-worker: 13:18 UTC (18:48 IST) — istio-proxy, 3 restarts
</details>

<details>
<summary>[Slack] Chronic pattern — 108 prior alerts for this workload</summary>

> **Verify:** Slack search shows 108 matches for "crm-contacts-bulk-whatsapp-worker" in #alerts-crm. Alerts cluster on Feb 9, Mar 3, 10-11, 27. All are istio-proxy restarts with no human investigation replies — only automated escalations.

No deployment within 4 days (last deploy: Mar 23). Team has "ISTIO Cleanup for bulk actions" as a known priority item.
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| High | Increase istio-proxy memory limit from 500Mi to 750Mi–1Gi for crm-contacts-bulk-whatsapp-worker | CRM Bulk Actions team |
| Medium | Review istio-proxy memory limits across all bulk action workers (contacts-bulk-import-contact-worker has only 384Mi) | CRM Bulk Actions team + Platform |
| Low | Suppress or adjust alert threshold for chronic istio-proxy OOMKills on workers with known tight limits | Platform team |

## Links

- [Verbose report](report-verbose.md)
- [Grafana — App Detailed View](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=crm-contacts-bulk-whatsapp-worker&var-cluster=workers-us-central-production-cluster&from=1774615680000&to=1774619280000)
- [GCP Kubelet Logs](https://console.cloud.google.com/logs/query;query=logName%3D%22projects%2Fhighlevel-backend%2Flogs%2Fkubelet%22%0A%22crm-contacts-bulk-whatsapp-worker%22;timeRange=2026-03-27T13%3A14%3A00Z%2F2026-03-27T13%3A20%3A00Z?project=highlevel-backend)
