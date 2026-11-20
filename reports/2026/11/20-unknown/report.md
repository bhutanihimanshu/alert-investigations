# Alert Investigation: 5XXPercentagePerAPI

**Investigation ID**: inv-2026-11-20-3d4b1cdb
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-20T00:41:34.116Z
**Status**: delivering

## Root Cause

**Summary**: Strongly supported causal chain: oversized subjects reached a defective vendored MIME validation branch, which threw ReferenceError instead of MIMETextError. This bypassed the intended 422 response, produced HTTP 500 and caused retries that amplified the route-specific error spike. Production logs, independent metrics and local reproduction support different links in this chain. Correct both error-description defects and verify permanent validation failures do not retry; encoded-header folding requires separate repair.
**Confidence**: high
**Category**: application_bug

## Triage

**Primary Anomaly**: error_spike
**Details**: leadconnector-api POST /leadconnector/gmail/deliver recorded approximately 38/174 requests as 5xx (21.84%) over five minutes versus 0.0429% over the preceding seven-day baseline. Over the overlapping ten-minute log window, 45 of 48 Gmail failures were ReferenceError: item is not defined in MIMEMessageHeader.validateLength while setting the subject, all associated with one user. Aggregate resources were within sampled weekly ranges and metrics showed zero preceding-hour restarts; Kubernetes verification was unavailable. No historical candidate matches this exception. Candidate 2 matches the route and account concentration, but its Gmail quota-exhaustion mechanism is unsupported here.
**Anomalous**: Yes
**Deep Dive Direction**: Trace createMimeBody → MIMEMessageHeader.validateLength in the deployed dependency version: identify the subject input condition that reaches the undefined item reference, verify error propagation to HTTP 500, and distinguish repeated delivery attempts from distinct messages using request/message identifiers. Check subject-length and encoding boundaries with a minimal local reproduction. An oversized or malformed subject is a hypothesis, not yet established. Determine the dependency fix or validation needed and verify retry behavior for this deterministic failure.

## Cross-Validation

**Confidence**: high

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | Gmail delivery accounted for approximately 38 of 39 service 5xx responses over five minutes: 21.84% versus a 0.0429% weekly baseline. Confirms route-specific impact. | ✓ |
| Production API and worker logs | 45 subject-validation ReferenceErrors overlapped the metric anomaly. Matching subjects exceeded the implementation's limit; trace correlation identified repeated attempts across 11 messages and HTTP-500-driven retries. | ✓ |
| Source tracing and local reproduction | Subject length 990 reproduces the undefined-item exception; 989 passes. The exception bypasses MIMETextError handling and becomes HTTP 500. Failure precedes the Gmail send call, contradicting quota exhaustion as the cause of these exceptions. | ✓ |
| Slack | Confirms the alert but provides no independent diagnosis, resolution, or confirmed production deployment. | ✗ |
| Kubernetes | Relevant endpoints were inaccessible; pod state and deployed image identity remain unverified. This is unavailable evidence, not contradictory evidence. | ✗ |

### Discrepancies
- Alert reported 19.51%; Grafana measured 21.84% in a different evaluation window. Five-minute metric estimates cannot be equated with ten-minute log counts.
- 51 NACK markers are not one-to-one with 45 ReferenceErrors; six additional markers cannot be attributed to this exception.
- Exact deployed commit and image were not verified; local reproduction and matching stack location support, but do not prove, source identity.
- Input origin, final delivery status and dead-letter arrival remain unknown.
- Cross-container infrastructure correlation was not evaluated despite a false-valued result field; sampled aggregate resources do not exclude all pod-level pressure.

## Deep Dive Findings

{
  "summary": "Confirmed deterministic MIME validation defect amplified by retries: subjects of 991–996 UTF-16 code units enter a broken length-error branch, produce ReferenceError instead of MIMETextError, return HTTP 500, and trigger Pub/Sub redelivery.",
  "confidence": "high",
  "window_ist": "2026-11-20 06:02–06:12",
  "root_cause": {
    "trigger": "All 45 matching request logs exceeded the implementation's 989-unit subject limit: 991 units × 5, 992 × 30, 993 × 8, 996 × 2.",
    "implementation": "createMimeBody → setSubject → setHeader → MIMEMessageHeader.validateLength computes name.length + value.length + 2. For 'subject', values above 989 exceed maxLineLength=998.",
    "defects": [
      "The rejection branch interpolates undefined item.name, throwing ReferenceError before constructing MIMETextError.",
      "The same error description contains adjacent template literals. Fixing item.name alone exposes TypeError; both defects require correction."
    ],
    "http_propagation": "ReferenceError bypasses the existing MIMETextError-to-422 handler. With no upstream HTTP status, sendEmailFromGmail returns code 500; deliver sends that HTTP status. Live response and worker logs confirm HTTP 500.",
    "upstream_exclusion": "Failure occurs before gmail.users.messages.send. Gmail quota exhaustion does not explain these exceptions.",
    "input_origin": "A sampled worker payload contains body-like invitation text in its subject. How that content entered the subject field remains unverified."
  },
  "version_evidence": {
    "executed_module": "/app/dist/src/util/mimetext.js",
    "dependency": "The handler imports vendored ../util/mimetext, not the installed npm package. Local yarn.lock resolves mimetext 2.1.0.",
    "source_match": "Transpiling the local vendored file with comments removed places the defective throw at line 159, matching the production stack.",
    "limitation": "Exact deployed image digest and source commit were not verified. Updating the npm dependency alone would not change this imported file."
  },
  "reproduction": {
    "method": "Executed the local vendored implementation with synthetic subjects; no network sends.",
    "results": [
      "ASCII length 989 passes; length 990 throws ReferenceError: item is not defined.",
      "494 emoji occupy 988 UTF-16 units and pass; 495 occupy 990 units and fail.",
      "989 accented characters pass despite occupying 1978 UTF-8 bytes.",
      "ASCII length 732 serializes to a 997-byte Subject header; length 733 serializes to 1001 bytes. Validation ignores Base64 expansion and does not fold the header.",
      "An in-memory correction of both error-description defects produces MIMETextError: INVALID_HEADER."
    ]
  },
  "retry_findings": {
    "distinct_messages": 11,
    "reference_error_events": 45,
    "identity_method": "Mapped Cloud Logging trace IDs from API exceptions to worker messageId fields. These are repeated attempts across 11 messages, not 45 distinct messages.",
    "worker_observation": "51 HTTP-500 NACK markers matched those 11 traces: nine messages had five markers, one had four, and one had two within the window.",
    "count_caveat": "The 51 NACK markers and 45 captured ReferenceErrors are not one-to-one; the additional six NACKs cannot independently be assigned this exception.",
    "example": {
      "messageId": "aZT4CLKGpHaXVRoZLdhd",
      "emailMessageId": "auwtYAxnVf303meM9Ucj",
      "nack_times_ist": ["06:10:50", "06:11:05", "06:11:22", "06:11:41"],
      "observed_decision": "HTTP 500 → nack-retry"
    },
    "live_subscription": {
      "name": "conversations-providers-deliver-gmail-events",
      "maxDeliveryAttempts": 5,
      "minimumBackoff": "10s",
      "maximumBackoff": "600s",
      "deadLetterTopic": "conversations-providers-deliver-gmail-events-dl"
    },
    "limitation": "Actual dead-letter arrival and final message status were not checked."
  },
  "recommended_actions": [
    "Correct both defects in the vendored validation error so invalid input reaches the existing HTTP 422 permanent-failure path.",
    "Validate subject type and length before delivery; return a clear permanent validation error rather than broadly converting ReferenceErrors to 4xx.",
    "Replace or repair MIME serialization to fold encoded headers correctly; raw character limits alone do not enforce serialized byte limits.",
    "Add boundary tests for ASCII, Unicode and encoded header lengths, plus an integration test proving invalid subjects return 422 and are acknowledged as failed without retries."
  ],
  "source_references": [
    "leadconnector/src/util/mimetext.js:186",
    "leadconnector/src/handlers/gmail_functions.ts:396",
    "leadconnector/src/handlers/gmail_functions.ts:415",
    "leadconnector/src/handlers/gmail_functions.ts:556",
    "marketplace-backend/apps/conversations/workers/conversations-providers-deliver-gmail-worker.ts:157"
  ],
  "log_queries": {
    "common_time_filter": "timestamp>=\"2026-11-20T00:32:00Z\" timestamp<=\"2026-11-20T00:42:00Z\"",
    "exceptions": "resource.type=\"k8s_container\" resource.labels.container_name=\"leadconnector-api\" severity>=ERROR jsonPayload.message=\"Failed to send email\" jsonPayload.error.errorName=\"ReferenceError\"",
    "subject_lengths": "resource.type=\"k8s_container\" resource.labels.container_name=\"leadconnector-api\" jsonPayload.userId=\"lU2kqjlenMywLucPrkGJ\" jsonPayload.message=\"Processing Gmail request\"",
    "worker_retries": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-providers-deliver-gmail-worker\" jsonPayload.message=\"[DELIVER_GMAIL_TRANSIENT_NACK]\" jsonPayload.error.errorMessage=\"Request failed with status code 500\"",
    "sample_trace": "resource.type=\"k8s_container\" trace=\"projects/highlevel-backend/traces/086b5497-cfdf-4aaa-b79c-3a4fa21caef5\""
  },
  "log_explorer_urls": {
    "subject_lengths": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22leadconnector-api%22%0AjsonPayload.userId%3D%22lU2kqjlenMywLucPrkGJ%22%0AjsonPayload.message%3D%22Processing%20Gmail%20request%22;timeRange=2026-11-20T00%3A32%3A00Z%2F2026-11-20T00%3A42%3A00Z?project=highlevel-backend",
    "confirmed_retries": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-providers-deliver-gmail-worker%22%0AjsonPayload.message%3D%22%5BDELIVER_GMAIL_TRANSIENT_NACK%5D%22%0Atrace%3D%22projects%2Fhighlevel-backend%2Ftraces%2F086b5497-cfdf-4aaa-b79c-3a4fa21caef5%22;timeRange=2026-11-20T00%3A32%3A00Z%2F2026-11-20T00%3A42%3A00Z?project=highlevel-backend"
  },
  "access": "Existing authenticated gcloud CLI; read-only production queries. No messages posted or production changes made."
}

## Evidence

### Dashboards (incident window)
- [Gmail delivery 5xx — leadconnector-api (error spike around 06:11 IST)](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&from=1795133463000&to=1795137063000&var-datasource=ber8nnhvgsjy8f&var-cluster=servers-usc1-prd-cluster&var-container=leadconnector-api&var-method=POST&var-route=%2Fleadconnector%2Fgmail%2Fdeliver&var-pod_name=All&viewPanel=14)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22leadconnector-api%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D%22Failed%20to%20send%20email%22;timeRange=2026-11-20T00%3A32%3A00Z%2F2026-11-20T00%3A42%3A00Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22leadconnector-api%22%0AjsonPayload.userId%3D%22lU2kqjlenMywLucPrkGJ%22%0AjsonPayload.message%3D%22Processing%20Gmail%20request%22;timeRange=2026-11-20T00%3A32%3A00Z%2F2026-11-20T00%3A42%3A00Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-providers-deliver-gmail-worker%22%0AjsonPayload.message%3D%22%5BDELIVER_GMAIL_TRANSIENT_NACK%5D%22%0Atrace%3D%22projects%2Fhighlevel-backend%2Ftraces%2F086b5497-cfdf-4aaa-b79c-3a4fa21caef5%22;timeRange=2026-11-20T00%3A32%3A00Z%2F2026-11-20T00%3A42%3A00Z?project=highlevel-backend)

### Related incidents
- [Gmail scheduled-send sync fix reported on staging — 19 Nov](https://gohighlevel.slack.com/archives/C096N87SBFW/p1795066115877509)

## Investigation Timeline

**Deep dive passes**: 1
**Worker calls**: 10

**Total worker time**: 651.1s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 12.2s | ✓ |
| grafana-worker | worker | 87.8s | ✓ |
| kubectl-worker | worker | 88.0s | ✓ |
| log-explorer-worker | worker | 172.0s | ✓ |
| slack-worker | worker | 68.0s | ✓ |
| triage | worker | 14.3s | ✓ |
| triage_llm_ranking | decision | 28.6s | ✓ |
| deep_dive_pass_1 | worker | 144.0s | ✓ |
| follow_up_check | decision | 11.1s | ✓ |
| cross_validation | worker | 25.4s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IGMXPX7A8FEG9|#140375 5XXPercentagePerAPI>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=10727950922855835787&amp;alert_id=7596075387597633507|source>*) app: leadconnector-api route: /leadconnector/gmail/deliver method: POST summary: 5XX Error Rate Above Threshold - Per API logs_link: <https://hyperdx.platform.prd.msgsndr.net/dashboards/6abc04ea06e1c8259f0f477e?source=6aa77741acd7321c4deda7ba&amp;where=container_name%253A%2522leadconnector-api%2522%2520AND%2520http_status%253A%255B500%2520TO%2520%252A%255D&amp;select=timestamp%252C%2520deployment_name%252C%2520log_level%252Clog&amp;whereLanguage=lucene&amp;filters=%255B%255D&amp;orderBy=timestamp%2520DESC&amp;from=&amp;to=&amp;isLive=true&amp;liveInterval=1800000&amp;refreshFrequency=10000&amp;mode=results&amp;denoise=false&amp;granularity=auto|here> threshold: 1% current_value: 19.51219512195122 dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-15m&amp;to=now&amp;viewPanel=14&amp;var-container=leadconnector-api|here> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IGMXPX7A8FEG9|#140375 5XXPercentagePerAPI>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=10727950922855835787&amp;alert_id=7596075387597633507|source>*) app: leadconnector-api route: /leadconnector/gmail/deliver method: POST summary: 5XX Error Rate Above Threshold - Per API logs_link: <https://hyperdx.platform.prd.msgsndr.net/dashboards/6abc04ea06e1c8259f0f477e?source=6aa77741acd7321c4deda7ba&amp;where=container_name%253A%2522leadconnector-api%2522%2520AND%2520http_status%253A%255B500%2520TO%2520%252A%255D&amp;select=timestamp%252C%2520deployment_name%252C%2520log_level%252Clog&amp;whereLanguage=lucene&amp;filters=%255B%255D&amp;orderBy=timestamp%2520DESC&amp;from=&amp;to=&amp;isLive=true&amp;liveInterval=1800000&amp;refreshFrequency=10000&amp;mode=results&amp;denoise=false&amp;granularity=auto|here> threshold: 1% current_value: 19.51219512195122 dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-15m&amp;to=now&amp;viewPanel=14&amp;var-container=leadconnector-api|here>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-20T00:52:05.520Z*