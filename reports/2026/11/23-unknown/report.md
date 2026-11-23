# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-23-ce5d45cf
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: web-users-servers-usc1-prd-cluster
**Channel**: slack
**Detected**: 2026-11-23T12:26:13.172Z
**Status**: delivering

## Root Cause

**Summary**: Metrics and logs independently confirm an isolated successful slow OAuth callback. Subscription processing accounts for most of the reported duration, but Graph latency, retry delays, webhook validation and Mongo persistence remain indistinguishable. No specific root cause has two-source support; correlated operation timings are needed.
**Confidence**: low
**Category**: undetermined

## Triage

**Primary Anomaly**: none
**Details**: An isolated latency anomaly is confirmed on leadconnector-api GET /leadconnector/outlook/finish_oauth: one of eight requests exceeded 10s, with a successful 10.821s callback logged at 17:49:08 IST. The reported 29s was not reproduced. No concurrent traffic, error, memory-pressure or restart anomaly was established. Historical matches largely reflect the incorrect CRM-conversations routing label; the earlier live-chat Firestore hypothesis is unverified for this route and time.
**Anomalous**: Yes
**Deep Dive Direction**: YES — Subscription processing remains the correct direction; Firestore is unsupported. Synchronous subscription setup explains most callback latency, but the cause of the 8.343s interval remains unresolved.

Need correlated per-attempt Microsoft Graph timings, statuses and retry delays; Mongo update timings; subscription-validation webhook timings; and confirmation that inspected source matches the deployed version.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | One of eight callback requests exceeded 10s; all returned 2xx. No concurrent traffic, restart or resource-pressure anomaly was established. | ✓ |
| Application logs | A successful callback took 10.821s at 17:49:08 IST, corroborating the isolated latency anomaly. Later subscription errors are not linked to this request. | ✓ |
| Deep dive and deployed-revision source | The reported 8.343s interval falls within sequential subscription processing involving Graph and Mongo persistence. Source supports this execution path but cannot identify which operation caused the delay. | ✓ |
| Kubernetes | Thirty pods were ready with zero application restarts and low resource usage approximately 22 minutes after detection; this cannot establish incident-time health. | ✓ |
| Slack | A simultaneous subscribe-route latency alert supports investigating subscription processing, but neither thread independently establishes a cause. | ✓ |

### Discrepancies
- The reported 29s was not reproduced: logs show 10.821s, while sparse histogram data estimates P95 at 22s.
- Triage labels primaryAnomaly as none despite metrics and logs corroborating an isolated slow callback.
- The log investigation's duplicated-alert characterization conflicts with Slack identifying a distinct subscribe-route alert.
- Falsification checks found no established traffic surge, restart or memory-pressure anomaly. Firestore involvement and specific Graph retry or Mongo slowdown claims lack request-level evidence.
- The deployed-revision source corrects the checkout mismatch, but image contents were not verified. Additional trace and webhook queries were interrupted, leaving the causal chain incomplete.

## Deep Dive Findings

{
  "summary": "Deployed-revision source confirms sequential Graph subscription processing and Mongo persistence. The cause of the 8.343s interval remains unresolved.",
  "workload": "leadconnector-api",
  "trace_id": "c4daa6a0-1439-4063-b224-8d2c4b06ead7",
  "findings": [
    {
      "finding": "The previously inspected checkout differs from the deployment revision.",
      "evidence": "Live deployment revision 41 identifies COMMIT_ID a16293adb92f5735ff0cc2c983666a74ca0b8c52, PR 389, and the supplied image digest. Local HEAD is 8769ee52fb6236fe4fdf16aea9d4a3e2d1a01181.",
      "confidence": "high",
      "limitation": "Deployment metadata establishes the declared revision; image contents were not independently verified."
    },
    {
      "finding": "Source retrieved at the declared deployment revision preserves the relevant execution order.",
      "evidence": "subscribeMails awaits inbound Graph creation or renewal, then OutlookIntegration.updateOne, before outbound subscription processing. These operations have no explicit per-operation timing logs in the inspected function.",
      "source": "https://github.com/GoHighLevel/leadconnector/blob/a16293adb92f5735ff0cc2c983666a74ca0b8c52/src/outlook/outlook_functions.ts",
      "confidence": "high"
    },
    {
      "finding": "No additional request-level timing evidence was obtained.",
      "evidence": "Trace and webhook log queries remained pending and were interrupted. This is an access/query limitation, not evidence that matching logs are absent.",
      "confidence": "high"
    }
  ],
  "root_cause": {
    "status": "partially_resolved",
    "prior_pass_evidence": "Successful 10.821s callback containing an 8.343054s interval during subscription processing.",
    "conclusion": "Graph response latency, SDK retry delays, webhook validation and Mongo persistence cannot yet be separated.",
    "unsupported_causes": ["Firestore", "Confirmed Graph retry", "Confirmed Mongo slowdown"]
  },
  "missing_evidence": [
    "Correlated Graph attempt durations, statuses, Retry-After values and request IDs.",
    "Mongo update duration or database span for this request.",
    "Subscription-validation webhook arrival and response timings."
  ],
  "recommended_action": "Obtain correlated Graph-attempt, Mongo-operation and validation-webhook timings; if unavailable historically, add instrumentation for recurrence without recording credentials or OAuth query parameters.",
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"leadconnector-api\" trace=\"projects/highlevel-backend/traces/c4daa6a0-1439-4063-b224-8d2c4b06ead7\" timestamp>=\"2026-11-23T12:18:50Z\" timestamp<=\"2026-11-23T12:19:15Z\"",
      "result": "Interrupted while pending; no results obtained."
    },
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"leadconnector-api\" timestamp>=\"2026-11-23T12:18:58Z\" timestamp<=\"2026-11-23T12:19:09Z\" (jsonPayload.message:\"/outlook/webhook\" OR jsonPayload.message:\"/outlook/outboundWebhook\")",
      "result": "Interrupted while pending; no results obtained."
    }
  ],
  "log_explorer_urls": [
    {
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%20trace%3D%22projects%2Fhighlevel-backend%2Ftraces%2Fc4daa6a0-1439-4063-b224-8d2c4b06ead7%22;timeRange=2026-11-23T12%3A18%3A50Z%2F2026-11-23T12%3A19%3A15Z?project=highlevel-backend",
      "status": "Prior-pass evidence link; results not revalidated this pass."
    }
  ],
  "access": "Read-only kubectl, authenticated gcloud CLI and GitHub CLI. GitHub connector returned 404; CLI retrieved the declared revision. No production changes or messages posted."
}

## Evidence

### Dashboards (incident window)
- [Request volume — leadconnector-api Outlook OAuth callback (sparse traffic)](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795434972000&to=1795436772000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=leadconnector-api&var-method=GET&var-route=%2Fleadconnector%2Foutlook%2Ffinish_oauth&var-pod_name=All&viewPanel=1)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22leadconnector-api%22%0Aresource.labels.cluster_name%3D%22web-users-servers-usc1-prd-cluster%22%0Aseverity%3E%3DERROR%0A%28jsonPayload.message%3D~%22OAuth%7COUTLOOK%7Coutlook%7Csubscriptions%22%20OR%20textPayload%3D~%22OAuth%7COUTLOOK%7Coutlook%22%29;timeRange=2026-11-23T12%3A21%3A00Z%2F2026-11-23T12%3A31%3A00Z?project=highlevel-backend)

### Related incidents
- [Simultaneous Outlook subscribe latency on the same cluster — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795436773202609)
- [Outlook error-handling change promoted; default sanity passed — 23 Nov](https://gohighlevel.slack.com/archives/C096N87SBFW/p1795064245350139)
- [Outlook delivery delays attributed to mailbox movement — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795432541929029)
- [Live-chat thread reports broader Firestore delays — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795433181325409)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 1425.3s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.1s | ✓ |
| grafana-worker | worker | 113.2s | ✓ |
| kubectl-worker | worker | 127.0s | ✓ |
| log-explorer-worker | worker | 301.5s | ✓ |
| slack-worker | worker | 89.8s | ✓ |
| triage | worker | 17.3s | ✓ |
| triage_llm_ranking | decision | 46.0s | ✓ |
| deep_dive_pass_1 | worker | 305.7s | ✓ |
| follow_up_check | decision | 16.6s | ✓ |
| deep_dive_pass_2 | worker | 206.0s | ✓ |
| follow_up_check | decision | 17.9s | ✓ |
| deep_dive_pass_3 | worker | 150.8s | ✓ |
| cross_validation | worker | 22.5s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I1H52XBJFZCFJ|#140719 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&alert_id=5562857972713705657|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `leadconnector-api` route `/leadconnector/outlook/finish_oauth` [GET] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 29s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&from=now-30m&to=now&var-datasource=ber8nnhvgsjy8f&var-container=leadconnector-api&var-method=GET&var-route=/leadconnector/outlook/finish_oauth&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&var-latencyThreshold=5|here> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I1H52XBJFZCFJ|#140719 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&amp;alert_id=5562857972713705657|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `leadconnector-api` route `/leadconnector/outlook/finish_oauth` [GET] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 29s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-30m&amp;to=now&amp;var-datasource=ber8nnhvgsjy8f&amp;var-container=leadconnector-api&amp;var-method=GET&amp;var-route=/leadconnector/outlook/finish_oauth&amp;var-pod_name=All&amp;var-minLatency=0&amp;var-maxLatency=100000&amp;var-latencyThreshold=5|here>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-23T13:04:39.792Z*