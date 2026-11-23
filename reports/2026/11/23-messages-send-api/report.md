# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-23-0fdd77dd
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: web-users-servers-usc1-prd-cluster
**Channel**: slack
**Detected**: 2026-11-23T12:08:43.120Z
**Status**: delivering

## Root Cause

**Summary**: A transient latency incident is corroborated by route metrics and request logs. A 13.005s awaited Firestore lookup explains the shorter request's blocking delay, but its underlying mechanism is unknown. The longer request's 36.807s pre-hydration delay remains unattributed. Request-correlated preflight spans and Firestore RPC attempt diagnostics are needed before assigning a common dependency or deployment cause.
**Confidence**: medium
**Category**: unresolved_partial_dependency_latency

## Triage

**Primary Anomaly**: error_spike
**Details**: Transient route latency elevation with a concurrent warning surge: POST /conversations/message recorded two successful requests exceeding 12s, producing ~39s P95; later P95 fell to 3.15s. Warnings peaked at 1,060/min versus 121–179/min baseline, predominantly Marketplace UBP errors in the sample. Seven AI manual-message timeout failures were logged across seven pods. No traffic surge, fleet resource spike, or restarts were observed. Dependency causation remains unproven.
**Anomalous**: Yes
**Deep Dive Direction**: YES — the pre-hydration investigation direction remains correct. A 13.005s Firestore `mailgun_accounts` lookup explains the shorter request, but its underlying cause and the longer request’s 36.807s delay remain unknown.

Need Firestore diagnostic evidence for the slow lookup, deployed-source/instrumentation coverage verification, and request-correlated stage timings for the longer request. AI timeouts and UBP warnings remain unproven causes.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | Two successful requests exceeded 12s on a low-volume route, producing approximately 39s P95. Later P95 was 3.15s. No traffic surge, fleet resource spike or restarts were observed. | ✓ |
| Request-correlated logs and local source | A 13.005s client-observed Firestore mailgun_accounts lookup supports attribution for the shorter request. The longer request's 36.807s pre-hydration delay remains unexplained; contact fetching itself took approximately 19ms. Deployed-source parity is unverified. | ✓ |
| Kubernetes | Forty pods were ready with zero restarts. Deployment completed approximately 125 minutes before detection. This supports the absence of a restart-driven incident, but post-alert resource snapshots cannot exclude incident-time pod pressure. | ✓ |
| Application error and warning analysis | Seven AI timeout failures and a Marketplace UBP warning surge were observed, but neither was linked to the two slow requests. These findings do not establish latency causation. | ✗ |
| Slack | Nearby discussion reports slow Firestore operations, supporting a dependency hypothesis but providing no independent confirmation of this request's underlying cause. | ✓ |

### Discrepancies
- Confidence applies to the latency incident and partial blocking-operation attribution; the overall root cause remains unresolved.
- The triage label error_spike is misleading: the route anomaly involved two successful slow requests, not a demonstrated HTTP error spike.
- Firestore duration includes SDK retries, network delay and client scheduling; server-side Firestore latency is not established.
- Falsification checks weaken traffic-surge, restart and slow-contact-fetch explanations. Individual pod saturation and event-loop blocking remain unexcluded.
- Nearby Firestore reports and warning timing are contextual correlation, not independent proof of causation. Interrupted diagnostic queries provide no negative evidence.

## Deep Dive Findings

{
  "summary": "The 13.005s Firestore lookup remains the established blocking operation for the shorter request, but its underlying cause is unresolved. The longer request's 36.807s pre-hydration delay remains unattributed.",
  "new_findings": {
    "firestore_diagnostics": {
      "verified": "Refetched the request-correlated warning at 17:36:46.494 IST: duration 13004.554ms on email-isv-api-6856459b4d-blxx4.",
      "measurement_boundary": "The inspected local Firestore instrumentation measures elapsed time around the awaited SDK Query.get() call. This is client-observed duration, not isolated Firestore server execution time.",
      "limitation": "The warning cannot distinguish server latency, SDK retries, network delay, or client scheduling. No request-specific evidence establishing these mechanisms was retrieved."
    },
    "deployment_and_instrumentation": {
      "email_isv": {
        "revision": 116,
        "replica_set": "email-isv-api-6856459b4d",
        "created_at_ist": "2026-11-19T14:01:39+05:30",
        "image_digest": "sha256:ed44d857ecbfb816133d7d83cd8ba70438f786445581e03cde283790ae0507e1",
        "verified_configuration": {
          "NODE_ENV": "production",
          "ENABLE_EMAIL_VERIFICATION_OBSERVABILITY": "true"
        },
        "incident_match": "The slow-query pod belongs to this ReplicaSet."
      },
      "send_api": {
        "revision": 246,
        "image_digest": "sha256:c42c9fd2b99c38be377ba3a395d7f68a458400533b493264dca5500b1e09fbe0"
      },
      "local_coverage": "Email-verification instrumentation records operation duration histograms and OpenTelemetry spans, including configuration lookup. Runtime diagnostics are span attributes; successful operations do not emit equivalent per-request timing logs.",
      "source_parity": "Unverified. Registry describe returned the email-isv image digest but no build provenance. Enabled configuration does not prove successful metric collection or span export."
    },
    "longer_request": {
      "request_id": "125c5410-d038-4f8e-809d-7b707070d29a",
      "refetched_timeline_ist": {
        "long_running_warning": "17:36:07.513",
        "contact_hydration_started": "17:36:39.321",
        "contact_fetched": "17:36:39.340",
        "send_completed": "17:36:39.790"
      },
      "finding": "Contact fetching took approximately 19ms once started; it does not explain the preceding delay. The 32 retrieved cross-service entries contained no stage timings explaining that delay.",
      "instrumentation_gap": "Inspected local preflight code awaits parallel unsubscribe, location, user and verification operations without individual request-correlated duration logs."
    }
  },
  "causality": {
    "shorter_request_blocking_dependency": "Supported by refreshed correlated logs and local source; deployed source parity remains unverified.",
    "underlying_firestore_cause": "Unknown",
    "longer_request_cause": "Unknown",
    "ai_timeouts_and_ubp_warnings": "Still unproven causes; no new request-correlated evidence supports attribution."
  },
  "log_queries_used": [
    {
      "filter": "timestamp>=\"2026-11-23T12:05:50Z\" timestamp<=\"2026-11-23T12:07:05Z\" trace=(\"projects/highlevel-backend/traces/1058661d-c5a6-44db-9feb-fadf0c09612f\" OR \"projects/highlevel-backend/traces/125c5410-d038-4f8e-809d-7b707070d29a\")",
      "result": "32 entries returned."
    },
    {
      "filter": "protoPayload.serviceName=\"firestore.googleapis.com\" timestamp>=\"2026-11-23T12:06:30Z\" timestamp<=\"2026-11-23T12:06:50Z\" protoPayload.methodName=~\"RunQuery|BatchGetDocuments\"",
      "result": "Interrupted after prolonged execution without output; not an empty-result finding."
    }
  ],
  "limitations": [
    "Additional five-entry slow-query and long-running-request detail queries were also interrupted without output.",
    "No request-correlated Firestore server diagnostics or additional distributed spans were obtained."
  ],
  "next_evidence_needed": [
    "Image-to-commit provenance and confirmation that operation metrics and spans were exported.",
    "Firestore RPC attempt timings and status codes for the shorter request.",
    "Request-correlated authorization, preflight, template-download and business-lookup timings for the longer request."
  ],
  "log_explorer_urls": [
    "https://console.cloud.google.com/logs/query;query=trace%3D%28%22projects%2Fhighlevel-backend%2Ftraces%2F1058661d-c5a6-44db-9feb-fadf0c09612f%22%20OR%20%22projects%2Fhighlevel-backend%2Ftraces%2F125c5410-d038-4f8e-809d-7b707070d29a%22%29;timeRange=2026-11-23T12%3A05%3A50Z%2F2026-11-23T12%3A07%3A05Z?project=highlevel-backend"
  ],
  "access": "Existing gcloud credentials and explicit production kubectl context; read-only queries and local source inspection.",
  "production_changes_or_messages": "None."
}

## Evidence

### Dashboards (incident window)
- [Request volume — send API POST /conversations/message, low traffic during latency spike](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795433922000&to=1795436221439&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-messages-send-api&var-method=POST&var-route=%2Fconversations%2Fmessage&var-pod_name=All&viewPanel=1)

### Related incidents
- [Live-chat latency: team reports slow Firestore operations — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795433181325409)
- [Outlook delivery latency attributed to mailbox move — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795432541929029)
- [Conversations isAutoReply fix promoted; default sanity reported good — 23 Nov](https://gohighlevel.slack.com/archives/C013CG4N7U0/p1795423225028339)
- [Outlook error classification change promoted — 23 Nov](https://gohighlevel.slack.com/archives/C096N87SBFW/p1795064245350139)
- [Gmail oversized-subject fix deployed; production sanity passed — 23 Nov](https://gohighlevel.slack.com/archives/C096N87SBFW/p1795411885150389)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 1734.8s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.4s | ✓ |
| grafana-worker | worker | 103.0s | ✓ |
| kubectl-worker | worker | 149.5s | ✓ |
| log-explorer-worker | worker | 294.8s | ✓ |
| slack-worker | worker | 86.5s | ✓ |
| triage | worker | 19.6s | ✓ |
| triage_llm_ranking | decision | 40.7s | ✓ |
| deep_dive_pass_1 | worker | 447.9s | ✓ |
| follow_up_check | decision | 13.0s | ✓ |
| deep_dive_pass_2 | worker | 250.1s | ✓ |
| follow_up_check | decision | 13.6s | ✓ |
| deep_dive_pass_3 | worker | 281.1s | ✓ |
| cross_validation | worker | 23.6s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IWU64LC67CSLG|#140716 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&alert_id=15251649224155274897|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-messages-send-api` route `/conversations/message` [POST] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 38.5s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&from=now-30m&to=now&var-datasource=ber8nnhvgsjy8f&var-container=conversations-messages-send-api&var-method=POST&var-route=/conversations/message&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&var-latencyThreshold=5|here> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IWU64LC67CSLG|#140716 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&amp;alert_id=15251649224155274897|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-messages-send-api` route `/conversations/message` [POST] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 38.5s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-30m&amp;to=now&amp;var-datasource=ber8nnhvgsjy8f&amp;var-container=conversations-messages-send-api&amp;var-method=POST&amp;var-route=/conversations/message&amp;var-pod_name=All&amp;var-minLatency=0&amp;var-maxLatency=100000&amp;var-latencyThreshold=5|here>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-23T12:38:53.527Z*