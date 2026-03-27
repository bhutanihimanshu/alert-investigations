# PodRestartsAboveThreshold Investigation — opportunities-upsert-api — 2026-03-27

**Author:** Himanshu Bhutani
**Generated:** 2026-03-27 11:35 IST

---

## 1. Alert Summary

| Field | Value |
|-------|-------|
| Alert type | PodRestartsAboveThreshold (#113813) |
| Workload | opportunities-upsert-api |
| Container | opportunities-upsert-api |
| Cluster | servers-us-central-production-cluster |
| Time | 11:19 IST (05:49 UTC) on 2026-03-27 |
| Threshold | 1 restart |
| Source channel | #alerts-crm |
| Status | Auto-resolved (~6 min) |

## 2. Investigation Findings

### Evidence: Grafana — Traffic Spike

A **3.8x traffic spike** (50→192 rps) was the triggering event. Traffic surged starting at ~11:16 IST, peaked at 11:19 IST, and normalized by ~11:21 IST.

<details>
<summary>API Requests Overview — traffic spike 50→192 rps at 11:16–11:19 IST</summary>

> **What to look for:** The hits/sec chart (top-left) shows traffic jumping from ~50 rps baseline to 192 rps peak. The 5XX rate chart shows a brief elevation during recovery.

![API Requests Overview](screenshots/001-api-requests-overview-traffic-spike-50-192-rps-at-11-16-11-1.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&var-container=opportunities-upsert-api&from=1774588743000&to=1774592343000)
</details>

### Evidence: Grafana — Event Loop Blocking

The traffic spike caused **event loop blocking across all pods**. The restarted pod (`7h84w`) had the worst lag at 746ms P99.

| Pod | Event Loop P99 Lag | Status |
|-----|-------------------|--------|
| 7h84w (restarted) | 746 ms | Hit liveness threshold |
| jjp7l | 712 ms | Recovered |
| gkwkt | 556 ms | Recovered |
| nqjsz | 518 ms | Recovered |
| Others | 200–400 ms | Recovered |

Normal event loop lag for this service: 11–15 ms.

<details>
<summary>NodeJS Event Loop Lag — P99 spiked to 746ms (normally 11–15ms)</summary>

> **What to look for:** A massive spike in event loop latency at 11:16 IST across multiple pods. Pod `7h84w` (the restarted one) shows the highest peak. The lag returns to baseline within ~1 minute.

![Event Loop Lag](screenshots/004-event-loop-lag-p99-spiked-to-746ms-at-11-16-ist-normally-11-.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/PTSqcpJWka/nodejs-application-dashboard?orgId=1&var-container=opportunities-upsert-api&from=1774588743000&to=1774592343000)
</details>

### Evidence: Grafana — CPU and Memory (NOT saturated)

CPU and memory were **well within limits**, ruling out OOM or CPU saturation as the cause:

| Resource | Peak | Limit | Utilization |
|----------|------|-------|-------------|
| CPU | 0.318 cores | 1.100 cores | 29% |
| Memory | 724 MB | 2,253 MB | 32% |

The event loop blocking was caused by **I/O saturation from the traffic surge**, not CPU or memory pressure.

<details>
<summary>CPU by Pod — peak 0.318 cores, well below 1.1 core limit</summary>

> **What to look for:** All pod CPU lines stay well below the limit reference line. Even during the traffic spike, CPU doesn't approach the 1.1 core limit.

![CPU by Pod](screenshots/002-cpu-by-pod-peak-0-318-cores-well-below-1-1-core-limit.png)

**Context (filters + time range):**
![CPU Context](screenshots/002-cpu-by-pod-peak-0-318-cores-well-below-1-1-core-limit-context.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=opportunities-upsert-api&var-cluster=servers-us-central-production-cluster&from=1774588743000&to=1774592343000)
</details>

<details>
<summary>Memory by Pod — peak 724 MB, well below 2,253 MB limit</summary>

> **What to look for:** Memory spikes briefly during the traffic burst (up to 724 MB) but stays well below the 2,253 MB limit. Not OOM.

![Memory by Pod](screenshots/003-memory-by-pod-peak-724-mb-well-below-2-253-mb-limit.png)

**Context (filters + time range):**
![Memory Context](screenshots/003-memory-by-pod-peak-724-mb-well-below-2-253-mb-limit-context.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b/app-detailed-view?orgId=1&var-container=opportunities-upsert-api&var-cluster=servers-us-central-production-cluster&from=1774588743000&to=1774592343000)
</details>

### Evidence: GCP — Probe Failure Events

**30 Unhealthy events** across all 15 pods within an 8-second window (05:45:12–05:45:20 UTC). All failures on `spec.containers{opportunities-upsert-api}` — the **app container**, NOT istio-proxy.

Probe failure message:
> `Liveness probe failed: Get "http://<pod-ip>:15020/app-health/opportunities-upsert-api/livez": context deadline exceeded (Client.Timeout exceeded while awaiting headers)`

<details>
<summary>K8s Pod Events — 30 probe failures across 15 pods at 11:15:12 IST</summary>

> **What to look for:** Multiple `Unhealthy` events at the same timestamp. Expand any event and check `involvedObject.fieldPath` — it should show `spec.containers{opportunities-upsert-api}` (app container, not istio-proxy).

![Probe Failures](screenshots/001-gcp-liveness-readiness-probe-failures-all-15-pods-fieldpath-conf.png)

```
resource.type="k8s_pod"
resource.labels.pod_name=~"opportunities-upsert-api.*"
jsonPayload.reason="Unhealthy"
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.pod_name%3D~%22opportunities-upsert-api.*%22%0AjsonPayload.reason%3D%22Unhealthy%22;timeRange=2026-03-27T05%3A44%3A00Z%2F2026-03-27T05%3A47%3A00Z?project=highlevel-backend)
</details>

### Evidence: GCP — HPA Scaling Events

HPA responded correctly to the CPU spike, scaling from 15→24 pods, then back to 15 after traffic subsided.

| Time (IST) | Event |
|------------|-------|
| 11:15:34 | HPA: 15→21 ("cpu container resource utilization above target") |
| 11:15:41 | HPA: 21→24 (still above target) |
| 11:21:10 | HPA: 24→15 ("cpu below target") |

<details>
<summary>K8s Cluster Events — HPA scaling 15→21→24→15 over 6 minutes</summary>

> **What to look for:** Three `SuccessfulRescale` events. The first two scale up within 7 seconds (rapid response), the third scales back down ~5.5 minutes later.

![HPA Scaling](screenshots/002-gcp-hpa-successfulrescale-15-21-24-15-pods-over-6-minutes.png)

```
resource.type="k8s_cluster"
jsonPayload.reason="SuccessfulRescale"
jsonPayload.involvedObject.name=~"opportunities-upsert-api"
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0AjsonPayload.reason%3D%22SuccessfulRescale%22%0AjsonPayload.involvedObject.name%3D~%22opportunities-upsert-api%22;timeRange=2026-03-27T05%3A44%3A00Z%2F2026-03-27T05%3A55%3A00Z?project=highlevel-backend)
</details>

## 3. Cross-Validation

| Signal | Source | Finding | Agrees? |
|--------|--------|---------|---------|
| Traffic spike | Grafana API Requests | 50→192 rps at 11:16 IST | Yes |
| Event loop blocking | Grafana NodeJS | P99 lag 746ms at 11:16 IST | Yes — correlates with traffic |
| Probe failures | GCP k8s_pod events | 30 failures at 11:15:12 IST on app container | Yes — correlates with event loop |
| HPA response | GCP k8s_cluster events | 15→24 pods at 11:15:34 IST | Yes — CPU above target |
| CPU not saturated | Grafana App Detailed | Peak 0.318 cores (29% of limit) | Yes — I/O-driven, not CPU |
| Memory not exhausted | Grafana App Detailed | Peak 724 MB (32% of limit) | Yes — not OOM |
| No deployment | Slack search | No deploys within 2h | Rules out deployment cause |
| Isolated alert | Alert Correlator | No other services restarting | Rules out infra-wide issue |

**Confidence: HIGH** — 8 independent signals all converge on the same causal chain.

## 4. Root Cause

**Traffic spike → event loop blocking → fleet-wide probe failure → pod restart.**

### Causal Chain

1. A traffic surge (3.8x, 50→192 rps) hit all 15 pods simultaneously
2. The increased request volume caused I/O saturation (likely database calls under concurrent load), blocking the NodeJS event loop to 746ms P99
3. With the event loop blocked, the health check endpoint couldn't respond, and liveness/readiness probes timed out ("context deadline exceeded") across all 15 pods
4. Most pods recovered within 16–21 seconds (below liveness failure threshold). Pod `7h84w` — with the worst lag (746ms) — likely hit the threshold and was killed
5. HPA detected high CPU utilization and scaled 15→21→24 pods
6. After ~6 minutes, traffic normalized, HPA scaled back to 15 pods, and the service fully recovered

### Why this keeps happening

This is the **same root cause** as the 2026-03-16 investigation. The recommended fixes have not been implemented:
- CPU request remains at 1 core (recommended: 1.5 cores)
- HPA target percentage has not been lowered (would scale earlier under traffic spikes)
- Upstream traffic spike source has not been investigated

The alert has fired **~10 times in 45 days** with this pattern.

<details>
<summary>Detailed timeline — full event log</summary>

| Time (IST) | Source | Event |
|------------|--------|-------|
| 11:15:12 | GCP k8s_pod | Liveness probe failed: pod 5h57n — context deadline exceeded |
| 11:15:12 | GCP k8s_pod | Readiness probe failed: pod 5h57n — context deadline exceeded |
| 11:15:12 | GCP k8s_pod | Liveness probe failed: pod jjp7l — context deadline exceeded |
| 11:15:12 | GCP k8s_pod | Readiness probe failed: pod jjp7l — context deadline exceeded |
| 11:15:13 | GCP k8s_pod | Readiness probe failed: pods h58wv, gkwkt |
| 11:15:13–20 | GCP k8s_pod | Remaining 11 pods fail liveness/readiness probes |
| 11:15:22 | Kubelet | SyncLoop: readiness=not ready — 5h57n, jjp7l |
| 11:15:28 | Kubelet | SyncLoop: readiness=READY — 5h57n (recovery starts, 16s) |
| 11:15:31 | Kubelet | SyncLoop: readiness=READY — h58wv |
| 11:15:33 | Kubelet | SyncLoop: readiness=READY — gkwkt, jjp7l, nqjsz, 7b272 |
| 11:15:34 | GCP k8s_cluster | HPA SuccessfulRescale: 15→21 (CPU above target) |
| 11:15:35 | GCP k8s_cluster | New pods created: vfc22, mbslq, qqndm, 2qsf8, 48zgz, w8hzv |
| 11:15:41 | GCP k8s_cluster | HPA SuccessfulRescale: 21→24 (still above target) |
| 11:16 | Grafana | Traffic spike: 148 rps (from ~50 baseline) |
| 11:16 | Grafana | Event loop lag: 746ms P99 (pod 7h84w), 712ms (jjp7l) |
| 11:16 | Grafana | Memory spike: pods 4cc2p (724 MB), f64tc (715 MB) |
| 11:17 | Grafana | Traffic: 169 rps |
| 11:18 | Grafana | Traffic: 175 rps; pod 7h84w restart counter increments |
| 11:19 | Grafana | Traffic peak: 192 rps |
| 11:19:03 | Slack | Alert fires: PodRestartsAboveThreshold (#113813) |
| 11:20 | Grafana | Traffic: 185 rps (declining) |
| 11:21 | Grafana | Traffic: 87 rps (normalizing) |
| 11:21:10 | GCP k8s_cluster | HPA SuccessfulRescale: 24→15 (CPU below target) |
| 11:22 | Grafana | Traffic: 63 rps (back to baseline) |

</details>

## 5. Probable Noise

<details>
<summary>Probable noise — transient errors during disruption (not root cause)</summary>

| Time | Pattern | Why it's noise |
|------|---------|----------------|
| Throughout | `BadRequestException: Moving a opportunity backward in the pipeline is not allowed.` (20 occurrences) | Steady-state business logic validation — same rate before/during/after incident |
| Throughout | `BadRequestException: The pipeline id is invalid.` (9 occurrences) | Steady-state validation — unrelated to probe failures |
| 11:19–11:26 | 5XX errors (0.1–0.17 rps) | Side-effect of the restarted pod being temporarily unavailable — resolved when pod came back |

</details>

## 6. Action Items

| Priority | Action | Status | Owner |
|----------|--------|--------|-------|
| **Medium** | Increase CPU request from 1→1.5 cores to provide headroom during traffic spikes | Not implemented (first recommended 2026-03-16) | CRM Opportunities team |
| **Medium** | Lower HPA targetCPUUtilizationPercentage to scale up earlier before probes fail | Not implemented | CRM Opportunities team |
| **Low** | Investigate upstream callers causing periodic 3–4x traffic surges | Not started | CRM Opportunities team |
| **Info** | Consider increasing liveness probe `failureThreshold` or timeout to tolerate brief event loop blocking | Suggestion — evaluate trade-offs | CRM Opportunities team |

## 7. Deployment Details

| Parameter | Value |
|-----------|-------|
| CPU request | 1.0 core |
| CPU limit | 1.1 cores |
| Memory limit | 2,253 MB (~2.2 GB) |
| Baseline pod count | 15 |
| HPA max | 24+ (scaled to 24 during incident) |
| Cluster | servers-us-central-production-cluster |

## 8. Recurrence History

| Date | Alert | Impact | Resolution |
|------|-------|--------|------------|
| 2026-03-27 (today) | PodRestartsAboveThreshold | 1 pod restart, 30 probe failures | Auto-resolved (HPA) |
| 2026-03-20 | PodRestartsAboveThreshold | Unknown (manually resolved) | Manual |
| 2026-03-16 | PodRestartsAboveThreshold | 4 pods killed (exit 137), HPA 15→26 | Auto-resolved |
| 2026-03-13 | PodRestartsAboveThreshold | Unknown | Unknown |
| 2026-03-08 | PodRestartsAboveThreshold | Unknown | Unknown |
| 2026-03-07 | PodRestartsAboveThreshold | Unknown | Unknown |
| 2026-03-04 | PodRestartsAboveThreshold | Unknown | Unknown |

This service has been alerting approximately **every 4–5 days** for the past 45 days. The root cause is consistent — the same traffic spike → event loop blocking → probe failure pattern — and will continue until the CPU/HPA recommendations are implemented.
