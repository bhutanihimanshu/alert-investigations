# PodRestartsAboveThreshold — contacts-bulk-import-contact-worker (istio-proxy) — 2026-03-27

**Author:** Himanshu Bhutani | **Status:** Auto-resolved (transient sidecar restarts)

## Summary

| Field | Value |
|-------|-------|
| Alert | [#113827 PodRestartsAboveThreshold](https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IFA4Q9964NN52) |
| Service | contacts-bulk-import-contact-worker |
| Container | istio-proxy (sidecar) |
| Cluster | workers-us-central-production-cluster |
| Fired | 17:50 IST (12:20 UTC) on 2026-03-27 |
| Duration | ~8 minutes (restarts: 17:45–17:53 IST) |
| Impact | **None** — app container healthy, zero errors. Worker reportedly not in use since March 2025. |

## Root Cause

**istio-proxy sidecar OOMKilled** due to a tight 384 MB memory limit. Memory usage across all 12 pods averaged 92% of the limit with peaks at 96–99%. Five pods hit the ceiling and were OOMKilled by the kernel. This is a **cluster-wide issue** — 403+ pods across the cluster had istio-proxy restarts in the same window, affecting CRM, LeadGen, Automations, and other teams.

## What Happened

1. **17:24 IST** — istiod HPA scaled down from 600 → 527 pods (memory below target), then back to 600 at 17:32 IST.
2. **17:45 IST** — istio-proxy readiness probes start failing on 5 pods (`connection reset by peer`, `connection refused` on port 15021). Memory at 372–382 MB of 384 MB limit.
3. **17:46–17:53 IST** — 5 of 12 pods' istio-proxy containers OOMKilled and restarted. Each recovered within 7–10 seconds.
4. **17:50 IST** — Alert fires with 8 cumulative restarts (threshold: 1).

## Proof

<details>
<summary>[Grafana] ISTIO Memory at 96–99% of 384 MB limit across all pods</summary>

> **Verify:** All pod lines in the memory graph are clustered at 350–382 MB, very close to the 384 MB limit line. The 5 pods that restarted touched the ceiling.

![ISTIO Memory](screenshots/001-istio-memory-usage-by-pod-contacts-bulk-import-contact-worke.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=contacts-bulk-import-contact-worker&var-cluster=workers-us-central-production-cluster&from=1774612200000&to=1774615800000&viewPanel=40)
</details>

<details>
<summary>[Grafana] Pod Restarts — 5 pods restarted between 17:45–17:53 IST</summary>

> **Verify:** Bar chart shows restart spikes on pods dt2dt, fhmd8, 2rn7g, sj6gg, 9dnf8 clustered in a 5-minute window.

![Pod Restarts](screenshots/002-pod-restarts-contacts-bulk-import-contact-worker.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=contacts-bulk-import-contact-worker&var-cluster=workers-us-central-production-cluster&from=1774612200000&to=1774615800000&viewPanel=36)
</details>

<details>
<summary>[Prometheus] Termination Reason = OOMKilled for all 5 restarts</summary>

> **Verify:** Prometheus metric `kube_pod_container_status_last_terminated_reason` confirms OOMKilled on all 5 pods. (Grafana table panel showed "No data" at screenshot time — data confirmed via direct Prometheus query.)

All 5 restarted pods (dt2dt, fhmd8, 2rn7g, sj6gg, 9dnf8) show `reason=OOMKilled` on the `istio-proxy` container.

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=contacts-bulk-import-contact-worker&var-cluster=workers-us-central-production-cluster&from=1774612200000&to=1774615800000&viewPanel=50)
</details>

<details>
<summary>[GCP] Readiness probe failures on istio-proxy — connection reset/refused on port 15021</summary>

> **Verify:** All events show `fieldPath: spec.containers{istio-proxy}` and messages contain "connection reset by peer" or "connection refused" on port 15021.

```
resource.type="k8s_pod"
resource.labels.pod_name=~"contacts-bulk-import-contact-worker.*"
resource.labels.cluster_name="workers-us-central-production-cluster"
jsonPayload.reason=~"Killing|Unhealthy|BackOff|OOMKilling"
jsonPayload.involvedObject.fieldPath:"istio-proxy"
```

| Time (IST) | Pod | Event |
|---|---|---|
| 17:45:05 | …-dt2dt | Readiness probe failed: connection reset by peer |
| 17:45:34 | …-fhmd8 | Readiness probe failed: connection reset by peer |
| 17:45:56 | …-2rn7g | Readiness probe failed: connection reset by peer |
| 17:45:57 | …-2rn7g | Readiness probe failed: connection refused |
| 17:47:37 | …-9dnf8 | Readiness probe failed: connection reset by peer |

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.pod_name%3D~%22contacts-bulk-import-contact-worker.%2A%22%0Aresource.labels.cluster_name%3D%22workers-us-central-production-cluster%22%0AjsonPayload.reason%3D~%22Killing%7CUnhealthy%7CBackOff%7COOMKilling%22%0AjsonPayload.involvedObject.fieldPath%3A%22istio-proxy%22;timeRange=2026-03-27T11%3A50%3A00Z%2F2026-03-27T12%3A50%3A00Z?project=highlevel-backend)
</details>

<details>
<summary>[GCP] Cluster-wide scope — 403+ pods with istio-proxy restarts in same window</summary>

> **Verify:** Events span multiple unrelated workloads across CRM, LeadGen, Automations teams — confirming infrastructure-level issue, not app-specific.

```
resource.type="k8s_pod"
resource.labels.cluster_name="workers-us-central-production-cluster"
jsonPayload.reason="Unhealthy"
jsonPayload.involvedObject.fieldPath:"istio-proxy"
```

Sample affected workloads: `crm-evaluations-ai-process-events-worker`, `leadgen-payments-stripe-events-worker`, `automation-workflows-start-worker`, `lc-email-billing-request-worker`, and 399+ more.

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.cluster_name%3D%22workers-us-central-production-cluster%22%0AjsonPayload.reason%3D%22Unhealthy%22%0AjsonPayload.involvedObject.fieldPath%3A%22istio-proxy%22;timeRange=2026-03-27T11%3A50%3A00Z%2F2026-03-27T12%3A50%3A00Z?project=highlevel-backend)
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| Medium | Increase istio-proxy sidecar memory limit from 384 MB → 512 MB+ | Platform team |
| Low | Consider decommissioning contacts-bulk-import-contact-worker (reportedly unused since March 2025) | CRM team |
| Low | Investigate what drives istio-proxy memory to 92% baseline — mesh complexity, xDS config size, or sidecar version | Platform team |

## Links

- [Verbose report](report-verbose.md)
- [Grafana — App Detailed View](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=contacts-bulk-import-contact-worker&var-cluster=workers-us-central-production-cluster&from=1774612200000&to=1774615800000)
- [GCP Log Explorer — istio-proxy events](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.pod_name%3D~%22contacts-bulk-import-contact-worker.%2A%22%0Aresource.labels.cluster_name%3D%22workers-us-central-production-cluster%22%0AjsonPayload.reason%3D~%22Killing%7CUnhealthy%7CBackOff%7COOMKilling%22%0AjsonPayload.involvedObject.fieldPath%3A%22istio-proxy%22;timeRange=2026-03-27T11%3A50%3A00Z%2F2026-03-27T12%3A50%3A00Z?project=highlevel-backend)
