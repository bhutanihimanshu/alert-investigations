# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-23-dc656f50
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: web-users-servers-usc1-prd-cluster
**Channel**: slack
**Detected**: 2026-11-23T13:07:43.846Z
**Status**: delivering

## Root Cause

**Summary**: Confirmed intermittent email-resend latency, with Firestore SDK waiting the best-supported proximate explanation from supplied request analysis and contemporaneous dependency evidence. Increased database activity coincided with the slowdown but is not proven causal. Obtain request-level RPC attempts/backoff and database-owner caller attribution before selecting remediation.
**Confidence**: medium
**Category**: dependency_latency_underlying_cause_unresolved

## Triage

**Primary Anomaly**: none
**Details**: The anomaly is latency, which is absent from the allowed categories: conversations-api POST /conversations/messages/resend/email reached p99 110.4s at 18:35 IST; logs independently show 34.204s and 13.654s requests. Traffic and aggregate resources remained within historical ranges. Current pod health cannot establish incident-time health. Concurrent IAM Redis and OAuth timeouts are not yet linked to the slow requests.
**Anomalous**: Yes
**Deep Dive Direction**: YES — The Firestore direction remains correct: SDK waits account for 91.2% and 74.2% of the two slow requests, but the underlying cause remains unresolved.

Need request-level RPC attempts/backoff timing or sampled dependency spans, plus database-owner backend diagnostics and read-volume analysis for 18:30–18:33 IST to distinguish retries, server/network delays, and increased load.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | Cluster-filtered metrics confirm intermittent route latency: p99 reached 110.4s at 18:35 IST. Traffic and aggregate resources remained within historical ranges; metrics do not identify the dependency responsible. | ✓ |
| Cloud Logging | Two requests took 34.204s and 13.654s, independently confirming slow requests. Supplied prior analysis attributes 91.2% and 74.2% to Firestore SDK waits, but underlying request-level timing evidence is not included here. | ✓ |
| Cloud Monitoring and local SDK source | Database-wide reads and deadline failures increased during 18:30–18:33 IST. The local logger measures the full awaited SDK operation, including possible retries; neither retry amplification nor load-induced latency is established. | ✓ |
| Slack | Human reports describe cross-service Firestore delays before this alert, supporting the dependency hypothesis. Automated findings for another route are not independent proof of this route's cause. | ✓ |
| Kubernetes | Current pods were healthy and the latest recorded rollout preceded the alert by approximately three hours. This does not establish incident-time health or corroborate Firestore causation. | ✗ |

### Discrepancies
- Real latency is corroborated by metrics and logs; the underlying Firestore failure mechanism is unresolved.
- The 110.4s percentile, 51.5s/96s alert values and sampled request durations represent different observations and do not reproduce one another.
- Database-wide read growth cannot identify the responsible workload or distinguish original demand from retries.
- Concurrent IAM Redis and OAuth errors are not linked to the slow requests; sampled error counts are not incident totals.
- Falsification remains incomplete: request-level RPC spans are needed to distinguish backend delay, network delay and retry backoff. Incident-time per-pod throttling and event-loop lag were not established, and deployed SDK equivalence is unverified.

## Deep Dive Findings

{
  "summary": "Database-wide read activity increased during the slow requests, but its trigger remains unresolved. New metrics do not establish retry amplification; request-level RPC attempts and backoff timing remain unavailable.",
  "confidence": {
    "database_read_growth": "high",
    "load_caused_latency": "unproven",
    "request_retries": "unproven"
  },
  "new_findings": {
    "scope": "highlevel-backend/(default), all API methods; 60-second sums with timestamps denoting bucket ends.",
    "minute_buckets_ist": [
      {"time": "18:30", "lookup_reads": 5838794, "query_reads": 7908061, "successful_api_calls": 8275688, "deadline_exceeded": 642},
      {"time": "18:31", "lookup_reads": 7549745, "query_reads": 10030747, "successful_api_calls": 10505598, "deadline_exceeded": 1643},
      {"time": "18:32", "lookup_reads": 8361183, "query_reads": 9950110, "successful_api_calls": 11690231, "deadline_exceeded": 8101},
      {"time": "18:33", "lookup_reads": 8884194, "query_reads": 10034178, "successful_api_calls": 12442791, "deadline_exceeded": 7794}
    ],
    "interpretation": "Between 18:30 and 18:33 IST, lookups increased 52.2%, query reads 26.9%, and successful API calls 50.4%. Deadline failures at 18:32 were approximately 0.069% of successful calls. These failures alone do not explain the volume increase as retries; successful duplicate reads and retries invisible to these counters remain possible.",
    "comparability": "These are broader metrics than the previous pass's BatchGetDocuments-specific figures and should not replace or be directly compared with those counts."
  },
  "rpc_evidence": {
    "source_verified": "Local @platform-core/firestore observability.js wraps the entire awaited DocumentReference.get and emits duration after success. The slow-query logger does not record individual RPC attempts or backoff.",
    "limitations": [
      "Local package equivalence to the deployed version remains unverified.",
      "A correlation-ID keyword search returned three HTTP request logs, not RPC attempt evidence.",
      "The narrower message-field retry search stalled and was cancelled; its result is unknown.",
      "No new dependency spans or database-owner backend diagnostics were obtained."
    ]
  },
  "causal_assessment": "The supplied prior findings attribute most of both requests to Firestore SDK waits. This pass confirms coincident database activity growth, but cannot distinguish increased original demand, duplicate reads, backend delay, network delay or SDK retries. Database aggregates cannot identify the responsible workload.",
  "metrics_queries_used": {
    "window_utc": "2026-11-23T12:57:00Z/2026-11-23T13:04:00Z",
    "filters": [
      "metric.type=\"firestore.googleapis.com/document/read_ops_count\" AND resource.labels.database_id=\"(default)\"",
      "metric.type=\"firestore.googleapis.com/api/request_count\" AND resource.labels.database_id=\"(default)\""
    ],
    "aggregation": "ALIGN_SUM and REDUCE_SUM over 60s; grouped by read type or response_code respectively."
  },
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_container\" timestamp>=\"2026-11-23T13:00:35Z\" timestamp<=\"2026-11-23T13:02:20Z\" (trace=\"projects/highlevel-backend/traces/251ecfaa-057e-4367-97ef-396edc083684\" OR trace=\"projects/highlevel-backend/traces/0694ae53-798b-4ec2-87ec-2608a57f1b4c\") (\"retry\" OR \"backoff\" OR \"attempt\" OR \"BatchGetDocuments\" OR \"DEADLINE_EXCEEDED\")",
      "limit": 5,
      "returned": 3,
      "result": "HTTP request logs; no verified RPC timing."
    }
  ],
  "log_explorer_urls": [
    "https://console.cloud.google.com/logs/query;query=trace%3D%22projects%2Fhighlevel-backend%2Ftraces%2F251ecfaa-057e-4367-97ef-396edc083684%22;timeRange=2026-11-23T13%3A00%3A35Z%2F2026-11-23T13%3A02%3A20Z?project=highlevel-backend"
  ],
  "next_action": "Database owner should attribute the read increase by caller and collection and inspect backend diagnostics for 18:30–18:33 IST. Request per-attempt RPC spans with status, duration and backoff before selecting remediation.",
  "access_path": "Existing gcloud credentials, Cloud Monitoring, Cloud Logging, read-only SQLite and local source. No messages posted, production changes made or orchestrator investigation launched."
}

## Evidence

### Dashboards (incident window)
- [P99 latency — conversations-api resend email, intermittent spikes; panel combines clusters](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795437600000&to=1795442880000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-api&var-method=POST&var-route=%2Fconversations%2Fmessages%2Fresend%2Femail&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&var-latencyThreshold=10&viewPanel=11)
- [Request rate — conversations-api resend email, low traffic during latency spikes](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795437600000&to=1795442880000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-api&var-method=POST&var-route=%2Fconversations%2Fmessages%2Fresend%2Femail&var-pod_name=All&viewPanel=1)
- [CPU by pod — conversations-api, inspect resource distribution during incident](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b?orgId=1&from=1795437600000&to=1795442880000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-api&var-pod_name=All&var-gcpProject=highlevel-backend&viewPanel=16)
- [Memory by pod — conversations-api, inspect working-set distribution during incident](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b?orgId=1&from=1795437600000&to=1795442880000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-api&var-pod_name=All&var-gcpProject=highlevel-backend&viewPanel=30)

### Related incidents
- [Team reports persistent cross-service Firestore delays — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795434619320709)
- [Nearby snippets latency; email alert merged across workloads — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795437462582059)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 1741.6s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.7s | ✓ |
| grafana-worker | worker | 134.6s | ✓ |
| kubectl-worker | worker | 139.2s | ✓ |
| log-explorer-worker | worker | 428.3s | ✓ |
| slack-worker | worker | 87.8s | ✓ |
| triage | worker | 17.7s | ✓ |
| triage_llm_ranking | decision | 44.8s | ✓ |
| deep_dive_pass_1 | worker | 320.5s | ✓ |
| follow_up_check | decision | 21.1s | ✓ |
| deep_dive_pass_2 | worker | 263.0s | ✓ |
| follow_up_check | decision | 20.8s | ✓ |
| deep_dive_pass_3 | worker | 228.2s | ✓ |
| cross_validation | worker | 23.8s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IU2ICL5PFI1QG|#140724 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&alert_id=14092078223828995011|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-api` route `/conversations/messages/resend/email` [POST] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 51.5s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&from=now-30m&to=now&var-datasource=ber8nnhvgsjy8f&var-container=conversations-api&var-method=POST&var-route=/conversations/messages/resend/email&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&var-latencyThreshold=5|here> :package: Showing the last alert only out of 2 total. Visit <https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IU2ICL5PFI1QG|the plugin page> to see them all. *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IU2ICL5PFI1QG|#140724 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&amp;alert_id=14092078223828995011|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-api` route `/conversations/messages/resend/email` [POST] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 51.5s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-30m&amp;to=now&amp;var-datasource=ber8nnhvgsjy8f&amp;var-container=conversations-api&amp;var-method=POST&amp;var-route=/conversations/messages/resend/email&amp;var-pod_name=All&amp;var-minLatency=0&amp;var-maxLatency=100000&amp;var-latencyThreshold=5|here> :package: Showing the last alert only out of 2 total. Visit <https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IU2ICL5PFI1QG|the plugin page> to see them all.
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-23T14:30:42.171Z*