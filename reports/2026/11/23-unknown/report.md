# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-23-64f6b7f9
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: web-users-servers-usc1-prd-cluster
**Channel**: slack
**Detected**: 2026-11-23T15:57:17.975Z
**Status**: delivering

## Root Cause

**Summary**: A slow Firestore update explains approximately 99% of one slow call-error request. The reason for that wait and its connection to the broader Firestore incident remain unresolved. Authorized application traces with RPC attempt timings, statuses and backoff are needed to distinguish the competing causes.
**Confidence**: low
**Category**: unresolved

## Triage

**Primary Anomaly**: none
**Details**: Latency is anomalous but outside the available categories: conversations-api POST /conversations/messages/call-error P95 reached 18.5s at 21:25 IST on approximately six requests per five minutes, recovering to 0.295s by 21:57. No route traffic surge, error-volume spike, or fleet-average resource anomaly was established. Historical candidates dc656f50 and de589c50 suggest Firestore waiting, but neither proves this route's cause. Low volume increases percentile sensitivity; it does not establish a false positive.
**Anomalous**: Yes
**Deep Dive Direction**: YES — The Firestore direction remains correct: its update consumed 15.792s of the initial 15.930s request, but the underlying cause remains unresolved.

Need authorized trace-backend access, correlated application trace IDs, Firestore RPC attempt timings/statuses/retries/backoff and server/network timing, plus secondary-request operation spans. A shared cause with the wider incident remains unproven.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | Route P95 reached 18.5s at 21:25 IST and recovered to 0.295s by 21:57. Low request volume increases percentile sensitivity; no route traffic surge or fleet-average resource anomaly was established. | ✓ |
| Request logs and deep dive | The initial 15.930s request spent 15.792s, approximately 99%, in a Firestore update. This identifies the slow operation, but not why it was slow. | ✓ |
| Slack | Contemporaneous human reports describe widespread Firestore latency. This supports a shared-dependency hypothesis but does not independently connect this request to that incident. | ✓ |
| Kubernetes | Current pods had zero application restarts. A readiness timeout and subsequent CPU-driven scaling do not establish the route's latency mechanism. | ✗ |
| Application error histograms | No alert-aligned error-volume spike was established. Sampled Redis and downstream errors were not linked to the slow request. | ✗ |

### Discrepancies
- Metrics and request logs support real latency, but two independent sources do not establish its underlying cause; repeated retrieval of the same logs is not independent corroboration.
- The Firestore operation was logged at 21:20:21.778 IST, while the sampled P95 peak was at 21:25. Aggregation may explain this offset, but exact request-to-histogram attribution is unverified.
- Revision 247 completed at 19:45:49 IST, within two hours before the alert despite being outside the narrower incident window; deployment causality remains untested.
- Falsification check: route-specific logs confirm Firestore use, but retries, backoff, client scheduling and network delay remain alternatives to server-side degradation. Fleet averages cannot exclude individual pod pressure.
- Trace access returned HTTP 401. Recovered B3 headers do not establish application OpenTelemetry correlation; secondary-request spans and event-loop evidence remain unavailable.

## Deep Dive Findings

{
  "status": "incomplete",
  "summary": "Reverified the initial request's 15.792-second Firestore operation and recovered its B3 trace ID. RPC retries, backoff, server/network timing, and the secondary request's operation spans remain unresolved.",
  "findings": [
    {
      "finding": "Initial-request retrieval completed",
      "request_id": "9a8d776c-ad04-4052-bc9e-aa7ede8b51ed",
      "evidence": "Cloud Logging returned six entries with no continuation token. The Firestore slow-operation entry at 2026-11-23 21:20:21.778 IST records 15792.260634 milliseconds.",
      "limitation": "This confirms elapsed operation time, not its underlying cause."
    },
    {
      "finding": "Initial B3 correlation recovered",
      "b3_trace_id": "3f934c6397b0094991ff0fff9123bd33",
      "b3_span_id": "91ff0fff9123bd33",
      "b3_sampled": false,
      "evidence": "Request payload headers contain these values. None of the six entries has a top-level spanId; no traceparent field was found."
    },
    {
      "finding": "B3 IDs may not identify application traces",
      "evidence": "Locally installed @platform-core/observability/src/opentelemetry/instrument_trace.ts explicitly configures W3CTraceContextPropagator and enables gRPC instrumentation.",
      "implication": "Neither recovered B3 ID establishes the application OpenTelemetry trace ID or sampling decision.",
      "limitation": "The deployed library version was not verified."
    },
    {
      "finding": "Trace access remains blocked",
      "evidence": "Grafana /api/org returned HTTP 401 with session.token.rotate.",
      "limitation": "No trace-backend spans were retrieved. Native browser/session tools were unavailable; session credentials were not modified."
    }
  ],
  "assessment": {
    "underlying_root_cause": "Unresolved",
    "shared_firestore_incident": "Plausible but unproven for these requests.",
    "secondary_request": "No new operation-level evidence obtained.",
    "next_action": "Restore authorized trace access, correlate application traces by request ID, service and timestamp, then inspect Firestore RPC attempts, statuses and backoff. If those details were not recorded, client/server telemetry is required."
  },
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-api\" timestamp>=\"2026-11-23T15:50:00Z\" timestamp<=\"2026-11-23T15:50:23Z\" trace=\"projects/highlevel-backend/traces/9a8d776c-ad04-4052-bc9e-aa7ede8b51ed\"",
      "result": "Six entries; complete retrieval through Cloud Logging API, also checked with gcloud."
    }
  ],
  "log_explorer_urls": [
    "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-api%22%0Atrace%3D%22projects%2Fhighlevel-backend%2Ftraces%2F9a8d776c-ad04-4052-bc9e-aa7ede8b51ed%22;timeRange=2026-11-23T15%3A50%3A00Z%2F2026-11-23T15%3A50%3A23Z?project=highlevel-backend"
  ],
  "access": "Existing gcloud credentials, Grafana CLI and local source inspection. Read-only; no messages, production changes or new orchestrator investigation."
}

## Evidence

### Dashboards (incident window)
- [P95 latency — conversations-api call-error, transient 21:25 IST spike](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795447637000&to=1795451237000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-api&var-method=POST&var-route=%2Fconversations%2Fmessages%2Fcall-error&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&viewPanel=17)
- [Request rate by status — conversations-api call-error, low-volume successful traffic](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795447637000&to=1795451237000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-api&var-method=POST&var-route=%2Fconversations%2Fmessages%2Fcall-error&var-pod_name=All&viewPanel=1)
- [CPU and memory — conversations-api, no fleet-average resource spike](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b?orgId=1&from=1795447637000&to=1795451237000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-api&var-pod_name=All&var-gcpProject=highlevel-backend)

### Related incidents
- [Team escalates cross-service Firestore latency to P0 — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795449294114479)
- [Same-cluster SLA users latency with reported Firestore waits — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795446829068679)
- [Conversations isAutoReply fix promoted; production sanity completed — 23 Nov](https://gohighlevel.slack.com/archives/C013CG4N7U0/p1795423225028339)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 1719.1s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.5s | ✓ |
| grafana-worker | worker | 108.2s | ✓ |
| kubectl-worker | worker | 154.8s | ✓ |
| log-explorer-worker | worker | 275.9s | ✓ |
| slack-worker | worker | 85.4s | ✓ |
| triage | worker | 18.4s | ✓ |
| triage_llm_ranking | decision | 42.9s | ✓ |
| deep_dive_pass_1 | worker | 614.4s | ✓ |
| follow_up_check | decision | 13.4s | ✓ |
| deep_dive_pass_2 | worker | 204.6s | ✓ |
| follow_up_check | decision | 11.8s | ✓ |
| deep_dive_pass_3 | worker | 158.1s | ✓ |
| cross_validation | worker | 19.6s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I3R5BEH3W6J7W|#140749 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&alert_id=15580562699234378947|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-api` route `/conversations/messages/call-error` [POST] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 18.5s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&from=now-30m&to=now&var-datasource=ber8nnhvgsjy8f&var-container=conversations-api&var-method=POST&var-route=/conversations/messages/call-error&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&var-latencyThreshold=5|here> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I3R5BEH3W6J7W|#140749 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&amp;alert_id=15580562699234378947|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-api` route `/conversations/messages/call-error` [POST] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 18.5s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-30m&amp;to=now&amp;var-datasource=ber8nnhvgsjy8f&amp;var-container=conversations-api&amp;var-method=POST&amp;var-route=/conversations/messages/call-error&amp;var-pod_name=All&amp;var-minLatency=0&amp;var-maxLatency=100000&amp;var-latencyThreshold=5|here>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-23T16:48:37.936Z*