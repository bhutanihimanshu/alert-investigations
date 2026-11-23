# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-23-c3118be7
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: web-users-servers-usc1-prd-cluster
**Channel**: slack
**Detected**: 2026-11-23T13:37:44.502Z
**Status**: delivering

## Root Cause

**Summary**: Elevated export latency is confirmed, with Firestore calls accounting for most of one measured slow request. Firestore backend degradation, client-side delays, pod throttling and Redis involvement remain unresolved; no complete causal chain is independently validated.
**Confidence**: low
**Category**: undetermined

## Triage

**Primary Anomaly**: error_spike
**Details**: Export latency is elevated: mean 2.85s versus 0.50s baseline and p99 74.75s versus 6.25s, with stable traffic and negligible 5xx. At 19:08 IST, application errors reached 535/min alongside spikes across more than 15 containers; sampled errors show IAM Redis failures and rate-limit counter timeouts. Slack separately reports Firestore consuming 31s of a 42s export. Neither dependency is confirmed causal. Later rollout and replica growth do not establish the initial trigger.
**Anomalous**: Yes
**Deep Dive Direction**: YES — the Firestore-path direction remains correct: deadline failures increased before the error spike, but backend root cause is unconfirmed. All sampled exports used one pod; Redis correlation does not prove a shared cause.

Need trace-level SDK attempts/retry timings, Firestore frontend/backend timing, historical pod network/event-loop metrics, and incident-image OAuth middleware code with measured awaited Redis durations.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | Export latency increased with stable traffic and negligible 5xx; this confirms degradation but does not identify its cause. | ✓ |
| Request logs | One 53.472-second export spent 41.563 seconds in measured Firestore calls (77.7%). This confirms Firestore-path contribution, not backend execution time. | ✓ |
| Slack | Reports describe slow Firestore calls across export and other routes. Independence from the supplied log evidence is not established. | ✓ |
| Cloud Monitoring and application errors | Redis errors and cross-container error spikes indicate a concurrent shared disturbance. No measured Redis wait links it to export latency. | ✗ |
| Historical pod metrics and Kubernetes | The sampled pod had increased CPU throttling. Later rollout and scaling events occurred after alert detection; neither establishes the initial trigger. | ✗ |

### Discrepancies
- Metrics and request logs independently confirm latency degradation, but two independent sources do not establish the underlying causal mechanism.
- Firestore-call wall time can include SDK retries, network waits and client scheduling delays; backend slowness remains unverified.
- Slow-request samples came from one pod, limiting fleet-wide attribution. Missing event-loop series do not establish healthy behavior.
- The largest error spike followed alert detection; sampled Redis errors existed earlier, but their request-level impact is unknown.
- Falsification remains incomplete: compare Firestore backend and SDK attempt timing across pods. Normal backend timing with client stalls would disprove backend attribution; negligible awaited Redis time would weaken Redis attribution.

## Deep Dive Findings

{
  "workload": "conversations-isolated-api",
  "summary": "Firestore calls account for 77.7% of one verified slow export. Historical pod throttling provides another possible contributor, but backend execution, SDK retries and client stalls remain unresolved.",
  "confidence": {
    "firestore_path_contribution": "medium",
    "underlying_root_cause": "unconfirmed"
  },
  "findings": [
    {
      "finding": "A sampled export includes two separately measured Firestore waits.",
      "trace_id": "05cddd84-6c96-4b9a-ba3e-85428a5e0075",
      "completed_at_ist": "2026-11-23 19:07:56.990",
      "request_seconds": 53.472,
      "location_document_get_seconds": 10.748,
      "message_batch_fetch_seconds": 30.815,
      "elasticsearch_seconds": 0.245,
      "message_count": 33,
      "interpretation": "Firestore-call wall time totals 41.563 seconds. Approximately 11.664 seconds remain outside these measured phases; that remainder cannot be attributed to Redis from available evidence."
    },
    {
      "finding": "The sampled pod experienced increased CPU throttling.",
      "pod": "conversations-isolated-api-6c7457cf77-27rzc",
      "baseline_ist": "18:30–19:00",
      "baseline_median_throttled_periods_percent": 11.24,
      "incident_ist": "19:05–19:10",
      "incident_throttled_periods_percent": "21.49–25.20",
      "cpu_usage_cores": "1.44–1.52",
      "historical_replicaset_resources": {
        "cpu_request_cores": 2,
        "cpu_limit_cores": 2.2,
        "memory_request": "3Gi",
        "memory_limit": "3379Mi"
      },
      "interpretation": "These historical limits differ from the later rollout's 1.1-core limit. Throttling was already present before alert detection. The metric measures the fraction of throttled scheduling periods, not request delay or percentage of CPU time lost."
    },
    {
      "finding": "Available request logs do not provide SDK attempt or retry timing.",
      "evidence": "The exact-trace Cloud Logging query returned eight entries without pagination. Entries had no spanId; request headers included x-b3-sampled=0.",
      "limitations": "The unsampled header does not prove all tracing was disabled. Cloud Trace lookup returned HTTP 429. No measured awaited Redis duration or Firestore retry/backoff duration was obtained."
    },
    {
      "finding": "Historical network throughput was available, but network latency and event-loop timing were not.",
      "receive_bytes_per_second": "2.56M–3.74M during 18:30–19:30 IST",
      "limitations": "Event-loop p99 and network receive-error queries returned no series. Empty results do not establish healthy networking or event-loop behavior."
    },
    {
      "finding": "Backend timing and incident-image middleware remain unverified.",
      "evidence": "The returned Firestore metric catalog exposes request_latencies described as frontend RPC latency, with no backend-execution timing metric identified.",
      "limitations": "Registry manifest lookup returned 404; the native image-description command did not complete and was cancelled. Local source cannot establish incident-image behavior."
    }
  ],
  "queries_used": {
    "logs": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-isolated-api\" timestamp>=\"2026-11-23T13:35:00Z\" timestamp<=\"2026-11-23T13:51:00Z\" trace=\"projects/highlevel-backend/traces/05cddd84-6c96-4b9a-ba3e-85428a5e0075\"",
    "prometheus": [
      "rate(container_cpu_usage_seconds_total{pod=\"conversations-isolated-api-6c7457cf77-27rzc\",container=\"conversations-isolated-api\"}[5m])",
      "rate(container_cpu_cfs_throttled_periods_total{pod=\"conversations-isolated-api-6c7457cf77-27rzc\",container=\"conversations-isolated-api\"}[5m]) / rate(container_cpu_cfs_periods_total{pod=\"conversations-isolated-api-6c7457cf77-27rzc\",container=\"conversations-isolated-api\"}[5m])",
      "nodejs_eventloop_lag_p99_seconds{container=\"conversations-isolated-api\"}",
      "rate(container_network_receive_bytes_total{pod=\"conversations-isolated-api-6c7457cf77-27rzc\"}[5m])"
    ],
    "metrics_window_utc": "2026-11-23T13:00:00Z/2026-11-23T14:00:00Z",
    "metrics_step_seconds": 60
  },
  "log_explorer_url": "https://console.cloud.google.com/logs/query;query=resource.labels.container_name%3D%22conversations-isolated-api%22%0Atrace%3D%22projects%2Fhighlevel-backend%2Ftraces%2F05cddd84-6c96-4b9a-ba3e-85428a5e0075%22;timeRange=2026-11-23T13%3A35%3A00Z%2F2026-11-23T13%3A51%3A00Z?project=highlevel-backend",
  "next_action": "Obtain sampled Firestore RPC attempts and backend timing, plus deployed-image provenance and awaited Redis spans. Compare slow requests across pods before attributing the incident to Firestore backend or pod throttling.",
  "access": "Authenticated Cloud Logging and Cloud Monitoring APIs; explicit production kubectl context. Grafana session returned 401. Read-only; no production changes, messages or orchestrator investigation launched."
}

## Evidence

### Dashboards (incident window)
- [Export p99 — conversations-isolated-api: elevated tail latency; panel aggregates clusters](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795437464000&to=1795446461000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-isolated-api&var-method=GET&var-route=%2Fconversations%2Fmessages%2Fexport&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&viewPanel=11)
- [Export traffic — conversations-isolated-api: no request-volume spike](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795437464000&to=1795446461000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-isolated-api&var-method=GET&var-route=%2Fconversations%2Fmessages%2Fexport&var-pod_name=All&viewPanel=1)

### Related incidents
- [Same workload also experienced message-search latency — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795441364422479)
- [Cross-route latency discussion attributes delays to Firestore — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795433181325409)
- [Conversations isAutoReply fix promoted before alert — 23 Nov](https://gohighlevel.slack.com/archives/C013CG4N7U0/p1795423225028339)
- [Inbound email extraction change; production build shared — 23 Nov](https://gohighlevel.slack.com/archives/C096N87SBFW/p1795414948575729)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 1981.7s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.3s | ✓ |
| grafana-worker | worker | 132.6s | ✓ |
| kubectl-worker | worker | 163.6s | ✓ |
| log-explorer-worker | worker | 274.3s | ✓ |
| slack-worker | worker | 70.9s | ✓ |
| triage | worker | 19.5s | ✓ |
| triage_llm_ranking | decision | 49.4s | ✓ |
| deep_dive_pass_1 | worker | 634.1s | ✓ |
| follow_up_check | decision | 17.0s | ✓ |
| deep_dive_pass_2 | worker | 312.4s | ✓ |
| follow_up_check | decision | 19.7s | ✓ |
| deep_dive_pass_3 | worker | 256.1s | ✓ |
| cross_validation | worker | 20.8s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IN8Q8IUQ4V8ES|#140729 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&alert_id=3067617420937153666|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-isolated-api` route `/conversations/messages/export` [GET] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 20.5824074074s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&from=now-30m&to=now&var-datasource=ber8nnhvgsjy8f&var-container=conversations-isolated-api&var-method=GET&var-route=/conversations/messages/export&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&var-latencyThreshold=5|here> :package: Showing the last alert only out of 4 total. Visit <https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IN8Q8IUQ4V8ES|the plugin page> to see them all. *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IN8Q8IUQ4V8ES|#140729 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&amp;alert_id=3067617420937153666|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-isolated-api` route `/conversations/messages/export` [GET] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 20.5824074074s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-30m&amp;to=now&amp;var-datasource=ber8nnhvgsjy8f&amp;var-container=conversations-isolated-api&amp;var-method=GET&amp;var-route=/conversations/messages/export&amp;var-pod_name=All&amp;var-minLatency=0&amp;var-maxLatency=100000&amp;var-latencyThreshold=5|here> :package: Showing the last alert only out of 4 total. Visit <https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IN8Q8IUQ4V8ES|the plugin page> to see them all.
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-23T15:05:16.482Z*