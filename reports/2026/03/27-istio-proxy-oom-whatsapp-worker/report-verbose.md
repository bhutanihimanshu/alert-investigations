# PodRestartsAboveThreshold Investigation — crm-contacts-bulk-whatsapp-worker — 2026-03-27

**Author:** Himanshu Bhutani
**Generated:** 2026-03-27 20:20 IST

---

## 1. Alert Summary

| Field | Value |
|-------|-------|
| Alert type | PodRestartsAboveThreshold (#113832) |
| Workload | crm-contacts-bulk-whatsapp-worker |
| Container | istio-proxy |
| Cluster | workers-us-central-production-cluster |
| Time | 18:48:04 IST (13:18:04 UTC) |
| Threshold | 1 |
| Source | #alerts-crm via CRM-bulk-actions route |
| Oncall | virendra.sharma, anjalica.suman |

---

## 2. What Happened

1. **18:45:21 IST** — Pod `fnlcc`'s istio-proxy container exceeded its 500Mi memory limit and was OOMKilled (exit code 137).
2. **18:46:18–18:47:14 IST** — Pods `bjd5m` and `sddm6` hit the same limit within the next 2 minutes.
3. **18:47:20 IST** — All 3 pods recovered automatically — istio-proxy restarted and passed readiness probes within 4-6 seconds each.
4. **18:48:04 IST** — Alert fired (threshold: 1 restart).

<details>
<summary>Detailed timeline — full event log</summary>

| Time (IST) | Source | Pod | Event |
|---|---|---|---|
| 18:45:21 | kubelet | fnlcc | Readiness probe failed — "connection reset by peer" on :15021 |
| 18:45:22 | kubelet | fnlcc | PLEG: ContainerDied (istio-proxy) |
| 18:45:22 | kubelet | fnlcc | ContainerStarted (istio-proxy) |
| 18:45:27 | kubelet | fnlcc | Readiness probe transitions: not ready → ready |
| 18:46:18 | kubelet | bjd5m | OOMKilled — exit code 137 |
| 18:46:19 | kubelet | bjd5m | ContainerStarted (istio-proxy) |
| 18:46:22 | kubelet | bjd5m | Readiness probe: ready |
| 18:47:14 | kubelet | sddm6 | Readiness probe failed — "EOF" on :15021 |
| 18:47:14 | kubelet | sddm6 | PLEG: ContainerDied (istio-proxy) |
| 18:47:15 | kubelet | sddm6 | ContainerStarted (istio-proxy) |
| 18:47:18 | kubelet | sddm6 | Readiness probe: ready |
| 18:48:04 | Grafana OnCall | — | Alert #113832 fires |

</details>

---

## 3. Investigation Findings

### Evidence: kubectl — Pod Status (PRIMARY SOURCE)

All 3 pods were Running 3/3 Ready at the time of investigation. The istio-proxy container on each shows a single restart with Last State: OOMKilled.

| Pod | Age | istio-proxy Restarts | Last State | Exit Code | Memory Limit | Memory Request |
|-----|-----|---------------------|-----------|-----------|--------------|----------------|
| fnlcc | 8h | 1 | OOMKilled | 137 | 500Mi | 400Mi |
| bjd5m | 3d5h | 1 | OOMKilled | 137 | 500Mi | 400Mi |
| sddm6 | 4d6h | 1 | OOMKilled | 137 | 500Mi | 400Mi |

The app container (`crm-contacts-bulk-whatsapp-worker`) had **zero restarts** on all pods — no application-level crash.

### Evidence: GCP Kubelet Logs — OOM Restart Sequence

<details>
<summary>Kubelet logs — OOMKilled lifecycle for all 3 pods</summary>

> **What to look for:** Each row shows a kubelet event. The pattern is: readiness probe fails (istio-proxy process dying) → PLEG detects ContainerDied → ContainerStarted (kubelet restarts it) → readiness transitions back to ready. All within 4-6 seconds per pod.

![Kubelet Logs](screenshots/001-gcp-kubelet-logs-istio-proxy-oomkilled-and-restart.png)

```
logName="projects/highlevel-backend/logs/kubelet"
"crm-contacts-bulk-whatsapp-worker"
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=logName%3D%22projects%2Fhighlevel-backend%2Flogs%2Fkubelet%22%0A%22crm-contacts-bulk-whatsapp-worker%22;timeRange=2026-03-27T13%3A14%3A00Z%2F2026-03-27T13%3A20%3A00Z?project=highlevel-backend)
</details>

### Evidence: GCP K8s Pod Events — Readiness Probe Failures

<details>
<summary>K8s pod events — readiness probe failures confirm istio-proxy container</summary>

> **What to look for:** Two events showing `Readiness probe failed` with `involvedObject.fieldPath: spec.containers{istio-proxy}`. This confirms the sidecar failed, not the app. Error messages: "connection reset by peer" (pod fnlcc) and "EOF" (pod sddm6).

![K8s Pod Events](screenshots/002-gcp-k8s-pod-events-readiness-probe-failures-on-istio-proxy.png)

```
resource.type="k8s_pod"
resource.labels.pod_name=~"crm-contacts-bulk-whatsapp-worker.*"
resource.labels.cluster_name="workers-us-central-production-cluster"
jsonPayload.reason=~"Unhealthy|Killing"
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.pod_name%3D~%22crm-contacts-bulk-whatsapp-worker.%2A%22%0Aresource.labels.cluster_name%3D%22workers-us-central-production-cluster%22%0AjsonPayload.reason%3D~%22Unhealthy%7CKilling%22;timeRange=2026-03-27T13%3A14%3A00Z%2F2026-03-27T13%3A20%3A00Z?project=highlevel-backend)
</details>

### Evidence: Application Health

The app container is healthy. Application ERROR logs show business-level message processing failures (specific bulk request `P3OGjPNyHBI9eERlLmm9` failing for multiple contacts) — these are pre-existing and unrelated to the istio-proxy OOMKill.

### Evidence: Correlated Alerts (Alert Correlator)

4 alerts within the ±15 min window, all in the same cluster:

| Time (IST) | Channel | Alert | Container |
|---|---|---|---|
| 17:50:39 | #alerts-crm | PodRestartsAboveThreshold | contacts-bulk-import-contact-worker (istio-proxy, 8 restarts) |
| 18:36:51 | #alerts-leadgen | PodRestartsAboveThreshold | leadgen-payments-authorize-net-webhook-worker-dl (istio-proxy) |
| 18:46:35 | #alerts-crm | PubSub Unacked Messages >10k | CRM-bulk-actions subscription |
| **18:48:04** | **#alerts-crm** | **PodRestartsAboveThreshold** | **crm-contacts-bulk-whatsapp-worker (istio-proxy)** |

Multiple istio-proxy restarts across unrelated workloads today, all in `workers-us-central-production-cluster`. However, the kubectl investigation found this specific workload's OOM was isolated (no cross-cluster event detected in pod events).

### Evidence: Slack — Chronic Pattern

| Date | Alerts for this workload |
|------|--------------------------|
| 2026-02-09 | 1 |
| 2026-03-03 | 2 |
| 2026-03-10–11 | 5 |
| 2026-03-27 | 1 (this alert) |

Total 108 Slack matches for "crm-contacts-bulk-whatsapp-worker" in #alerts-crm. All threads show only automated oncall escalations — no human investigation context. Team has "ISTIO Cleanup for bulk actions" as a recognized priority.

Last deployment: 2026-03-23 (4 days ago) — deployment is not the trigger.

---

## 4. Cross-Validation

| Signal | Source | Finding | Agrees? |
|--------|--------|---------|---------|
| Termination reason | kubectl describe | OOMKilled, exit 137 | ✅ |
| Container | kubectl describe | istio-proxy (not app) | ✅ |
| Memory limit | kubectl describe | 500Mi limit, 400Mi request | ✅ |
| Restart lifecycle | GCP kubelet logs | Readiness fail → ContainerDied → Restart → Ready | ✅ |
| fieldPath | GCP k8s_pod events | spec.containers{istio-proxy} | ✅ |
| App health | kubectl + GCP app logs | App container: 0 restarts, healthy | ✅ |
| No deployment | Slack search | Last deploy Mar 23 (4 days ago) | ✅ |
| Recurring pattern | Slack + Alert Correlator | 108 prior matches, chronic | ✅ |

**Confidence: HIGH** — 5 independent sources confirm the same root cause. No contradictory evidence.

**Disproof check:** Could this be a cluster-wide event rather than workload-specific OOM? The broader GCP pod events query found no other workloads with istio-proxy issues in the same 13:14-13:20 UTC window. The correlated alerts (contacts-bulk-import-contact-worker at 12:20 UTC) were ~1 hour earlier, suggesting independent OOM events rather than a single cluster trigger. However, multiple workloads hitting istio-proxy memory limits on the same day suggests the limits are broadly too tight across bulk action workers.

---

## 5. Root Cause

**istio-proxy OOMKilled** on all 3 pods of `crm-contacts-bulk-whatsapp-worker` due to the 500Mi memory limit being insufficient for the Envoy proxy's steady-state memory growth.

**Causal chain:**
1. Envoy proxy accumulates memory over time (connection tables, route cache, xDS config) as pods run for hours/days
2. All 3 pods hit the 500Mi limit within a 2-minute window (13:15:21–13:17:14 UTC)
3. Kernel OOM killer sends SIGKILL (exit code 137) to the istio-proxy container
4. Readiness probe fails immediately (connection reset / EOF on port 15021)
5. Kubelet restarts the container → passes readiness in 4-6 seconds → pod recovers
6. Alert fires at threshold of 1 restart

This is NOT a deployment issue, NOT a cluster-wide infrastructure event, and NOT an application error.

<details>
<summary>Probable noise — transient errors during disruption (not root cause)</summary>

| Time | Pattern | Why it's noise |
|------|---------|----------------|
| 18:45–18:47 IST | Readiness probe failed "connection reset by peer" / "EOF" | Expected symptom of OOMKill — istio-proxy is dying, so its health endpoint is unreachable |
| Pre-existing | `Error processing a message for sending whatsapp - bulkRequestId(P3OGjPNyHBI9eERlLmm9)` | Business logic errors for a specific bulk request, unrelated to infra |

</details>

---

## 6. Action Items

| Priority | Action | Owner | Reasoning |
|----------|--------|-------|-----------|
| **High** | Increase istio-proxy memory limit from 500Mi to 750Mi–1Gi for crm-contacts-bulk-whatsapp-worker | CRM Bulk Actions team | Direct fix — prevents recurring OOMKills |
| **Medium** | Audit istio-proxy memory limits across all bulk action workers | CRM Bulk Actions + Platform | contacts-bulk-import-contact-worker has only 384Mi and alerted today too |
| **Low** | Consider suppressing or tuning alert threshold for chronic istio-proxy OOMKills on workers with known tight limits | Platform team | 108 alerts with no human investigation — alert fatigue |

### Separate issues found

| Issue | Details |
|-------|---------|
| Bulk request processing failures | bulkRequestId `P3OGjPNyHBI9eERlLmm9` failing for multiple contacts — unrelated to infra, needs investigation by the feature team |

---

## 7. Deployment Details

| Setting | Value |
|---------|-------|
| Pods | 3 |
| App container memory | Not inspected (healthy) |
| istio-proxy memory limit | 500Mi |
| istio-proxy memory request | 400Mi |
| HPA | No scaling events during incident |
| Last deploy | 2026-03-23 (4 days prior) |

---

## 8. Links

- [Concise report](report.md)
- [Grafana — App Detailed View](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=crm-contacts-bulk-whatsapp-worker&var-cluster=workers-us-central-production-cluster&from=1774615680000&to=1774619280000)
- [GCP Kubelet Logs](https://console.cloud.google.com/logs/query;query=logName%3D%22projects%2Fhighlevel-backend%2Flogs%2Fkubelet%22%0A%22crm-contacts-bulk-whatsapp-worker%22;timeRange=2026-03-27T13%3A14%3A00Z%2F2026-03-27T13%3A20%3A00Z?project=highlevel-backend)
- [GCP K8s Pod Events](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.pod_name%3D~%22crm-contacts-bulk-whatsapp-worker.%2A%22%0Aresource.labels.cluster_name%3D%22workers-us-central-production-cluster%22%0AjsonPayload.reason%3D~%22Unhealthy%7CKilling%22;timeRange=2026-03-27T13%3A14%3A00Z%2F2026-03-27T13%3A20%3A00Z?project=highlevel-backend)
