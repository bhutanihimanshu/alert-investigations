# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-23-de589c50
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: servers-usc1-prd-cluster
**Channel**: slack
**Detected**: 2026-11-23T11:26:21.325Z
**Status**: delivering

## Root Cause

**Summary**: Transient live-chat latency is corroborated by metrics and human observations. Firestore RunQuery delay is the leading reported explanation, but independent request-level verification and the underlying Firestore trigger remain unresolved. Retrieve the referenced trace and match it to server or SDET request timings before confirming causation.
**Confidence**: low
**Category**: unconfirmed_dependency_latency

## Triage

**Primary Anomaly**: none
**Details**: Confirmed transient route-latency anomaly: conversations-api POST /conversations/providers/live-chat/messages P95 reached 19s at 16:49–16:53 IST, dropping below 1s at 16:54. Traffic peaked at only 10 requests/minute; average CPU and memory were normal, with no observed restarts. Redis resets at 16:55 occurred after latency recovery and do not establish causation. None of the 20 historical candidates matches the observed failure pattern.
**Anomalous**: Yes
**Deep Dive Direction**: YES — Request-level latency remains the correct direction, but Firestore is proven to explain only the five lookup/upsert delays, not the live-chat spike.

Need incident-time SDET client request IDs/timings and matching server logs or retained dependency spans for the missing 15–20s live-chat request to determine its root cause.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | Confirms route P95 of 19s at 16:49–16:53 IST, recovering below 1s at 16:54. Does not identify the slow dependency. | ✓ |
| Slack request-level report | Reports a 17s live-chat request containing a 16s Firestore RunQuery span near the spike. Underlying trace was not independently inspected. | ✓ |
| Cloud Logging | Confirms three HTTP 201 live-chat completions, but lacks durations and dependency spans; cannot independently corroborate Firestore attribution. | ✗ |
| Kubernetes and resource metrics | Normal average resources and zero observed application restarts weaken capacity or crash explanations, but do not exclude individual pod stalls. | ✓ |
| Firestore screenshot | Database-wide DEADLINE_EXCEEDED counts over seven days do not link Firestore delays to this request. | ✗ |

### Discrepancies
- Firestore causation has only one request-level report; two independent sources have not validated the causal chain.
- Slack reports 20–70 requests/minute and P99; cluster-filtered Grafana reports a peak of 10 requests/minute and P95. Scopes require reconciliation.
- Redis errors occurred at 16:55, after the principal latency spike recovered at 16:54, undermining Redis attribution.
- Firestore delays on five lookup/upsert requests do not prove causation for the live-chat route.
- Falsification remains pending: inspect the reported trace for matching route, cluster and timestamp, and verify RunQuery occupied the request's critical path. A mismatch or overlapping nonblocking span would undermine attribution.

## Deep Dive Findings

{
  "summary": "New thread evidence reports the missing 17-second live-chat request with a 16-second Firestore RunQuery span. This strengthens the Firestore hypothesis, but the underlying trace and SDET client timings were not recovered independently.",
  "root_cause": {
    "candidate": "Firestore RunQuery delay on the live-chat request",
    "confidence": "medium",
    "underlying_firestore_trigger": "Unresolved",
    "status": "Reported request-level attribution; independent span verification remains pending."
  },
  "findings": [
    {
      "source": "Slack reply posted at 17:20:19 IST",
      "observation": "Reports a live-chat request at 16:48 IST lasting 17 seconds, including a 16-second RunQuery operation. Also reports four live-chat requests exceeding 10 seconds during 16:30–17:05 IST, all HTTP 201.",
      "url": "https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795434619320709",
      "limitation": "No request ID, trace permalink or exported spans accompany the text."
    },
    {
      "source": "Attached screenshot, visually inspected",
      "observation": "Shows Firestore DEADLINE_EXCEEDED request counts for the default database, grouped by API method over seven days.",
      "url": "https://gohighlevel.enterprise.slack.com/files/U06TDEJE6KE/F0CM08U5RCY/image.png",
      "limitation": "The attachment is a database metric chart, not the claimed live-chat trace; it cannot establish request-level causation."
    },
    {
      "source": "Fresh Cloud Logging query",
      "observation": "Returned three live-chat POST completions, all HTTP 201, at 16:47:08, 16:49:18 and 16:53:54 IST. No additional matching completion was recovered.",
      "trace_ids": [
        "d6ef8649-0030-44c8-8b82-3f291a48ac3d",
        "fe8b6584-69b6-43ee-b9c2-6590f437b076",
        "cd7cf725-aedd-48bc-a67b-b84a1f9a0cf6"
      ],
      "limitation": "Returned projected fields did not contain durations or dependency spans."
    }
  ],
  "evidence_gaps": [
    "No incident-time SDET client request IDs or timing artifacts found in the alert thread or targeted Slack search.",
    "No callable Uptrace connector or documented authenticated access path was found in the inspected local tools and skills.",
    "Slack's reported 20–70 requests/minute and p99 differ from the supplied cluster-filtered metrics of 10 requests/minute and P95; scopes need reconciliation.",
    "The five previously matched Firestore reads remain evidence for lookup/upsert delays, not independent proof for live-chat."
  ],
  "log_queries_used": [
    {
      "project": "highlevel-backend",
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-api\" resource.labels.cluster_name=\"servers-usc1-prd-cluster\" timestamp>=\"2026-11-23T11:17:00Z\" timestamp<\"2026-11-23T11:26:00Z\" (httpRequest.requestUrl:\"/live-chat/messages\" OR jsonPayload.message:\"/live-chat/messages\")",
      "limit": 20,
      "returned_entries": 3
    }
  ],
  "log_explorer_urls": [
    "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%20resource.labels.container_name%3D%22conversations-api%22%20resource.labels.cluster_name%3D%22servers-usc1-prd-cluster%22%20%28httpRequest.requestUrl%3A%22%2Flive-chat%2Fmessages%22%20OR%20jsonPayload.message%3A%22%2Flive-chat%2Fmessages%22%29;timeRange=2026-11-23T11%3A17%3A00Z%2F2026-11-23T11%3A26%3A00Z?project=highlevel-backend"
  ],
  "next_action": "Retrieve the 16:48 IST Uptrace trace referenced in the Slack reply, verify route, cluster and parent-child timing, then match its request ID to SDET evidence. Investigate Firestore's underlying trigger only after that attribution is confirmed.",
  "access_path": "Existing validated Slack session and authenticated gcloud CLI; read-only. No messages posted or production changes made."
}

## Evidence

### Dashboards (incident window)
- [P95 latency — conversations-api live-chat POST: transient 19s spike](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795431381000&to=1795434360000&var-datasource=ber8nnhvgsjy8f&var-cluster=servers-usc1-prd-cluster&var-container=conversations-api&var-method=POST&var-route=%2Fconversations%2Fproviders%2Flive-chat%2Fmessages&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&var-latencyThreshold=10&viewPanel=17)
- [Request rate — conversations-api live-chat POST: sparse, elevated traffic](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795431381000&to=1795434360000&var-datasource=ber8nnhvgsjy8f&var-cluster=servers-usc1-prd-cluster&var-container=conversations-api&var-method=POST&var-route=%2Fconversations%2Fproviders%2Flive-chat%2Fmessages&var-pod_name=All&viewPanel=1)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-api%22%0Aresource.labels.cluster_name%3D%22servers-usc1-prd-cluster%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22ECONNRESET%7CDYNAMIC_CONFIG_PACKAGE_ERROR%22;timeRange=2026-11-23T11%3A20%3A00Z%2F2026-11-23T11%3A30%3A00Z?project=highlevel-backend)

### Related incidents
- [Outlook delivery latency alert 11 minutes earlier — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795432541929029)
- [Conversations isAutoReply fix promoted before alert — 23 Nov](https://gohighlevel.slack.com/archives/C013CG4N7U0/p1795423225028339)
- [Inbound email extraction change: production build shared — 23 Nov](https://gohighlevel.slack.com/archives/C096N87SBFW/p1795414948575729)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 1722.0s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 10.7s | ✓ |
| grafana-worker | worker | 110.5s | ✓ |
| kubectl-worker | worker | 138.3s | ✓ |
| log-explorer-worker | worker | 287.4s | ✓ |
| slack-worker | worker | 65.3s | ✓ |
| triage | worker | 15.8s | ✓ |
| triage_llm_ranking | decision | 39.8s | ✓ |
| deep_dive_pass_1 | worker | 467.4s | ✓ |
| follow_up_check | decision | 15.3s | ✓ |
| deep_dive_pass_2 | worker | 391.4s | ✓ |
| follow_up_check | decision | 14.2s | ✓ |
| deep_dive_pass_3 | worker | 145.4s | ✓ |
| cross_validation | worker | 20.4s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/ITSHV8MBMKX9M|#140713 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&alert_id=16999526078515305045|source>*) cluster: servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-api` route `/conversations/providers/live-chat/messages` [POST] on servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 19s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&from=now-30m&to=now&var-datasource=ber8nnhvgsjy8f&var-container=conversations-api&var-method=POST&var-route=/conversations/providers/live-chat/messages&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&var-latencyThreshold=5|here> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/ITSHV8MBMKX9M|#140713 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&amp;alert_id=16999526078515305045|source>*) cluster: servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-api` route `/conversations/providers/live-chat/messages` [POST] on servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 19s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-30m&amp;to=now&amp;var-datasource=ber8nnhvgsjy8f&amp;var-container=conversations-api&amp;var-method=POST&amp;var-route=/conversations/providers/live-chat/messages&amp;var-pod_name=All&amp;var-minLatency=0&amp;var-maxLatency=100000&amp;var-latencyThreshold=5|here>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-23T12:08:12.851Z*