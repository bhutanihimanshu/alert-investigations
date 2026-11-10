# Alert Investigation: 5XXPercentagePerAPI

**Investigation ID**: inv-2026-11-10-72677a1e
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-09T17:05:05.554Z
**Status**: delivering

## Root Cause

**Summary**: Outlook subscription recovery attempts continue for a torn-down integration. Token refresh fails with HTTP 400 invalid_grant; Graph wraps the failure with status -1, which insufficient status validation allows into res.status, producing a RangeError and route failures. Request traces, source code, and temporally aligned route metrics support this causal chain. Low request volume amplifies the percentage. Validate and deploy the status-handling fix, and prevent recovery and token refresh for terminally disconnected integrations.
**Confidence**: high
**Category**: application_error_handling_defect

## Triage

**Primary Anomaly**: error_spike
**Details**: leadconnector-api GET /leadconnector/outlook/subscribe/:user_Id recorded 19 failures/41 requests in the latest alert's 15-minute window, versus a 0.279% seven-day baseline. Earlier logs show token-refresh failures and invalid HTTP status -1 RangeErrors, with all 40 sampled middleware failures targeting one user. Resources appeared normal; Kubernetes verification was unavailable. Candidate 12 matches the invalid-status mechanism but concerns a different route and upstream trigger.
**Anomalous**: Yes
**Deep Dive Direction**: Prioritize a recurring Outlook subscribe error-handling defect: failed token refresh produces Graph status -1, subscribeMails passes it to res.status, and ERR_HTTP_INVALID_STATUS_CODE yields route 5xx. Verify the underlying OAuth error and request-level chain for user SK9wwSMoAbQ0F103QbGx, inspect subscribeMails status validation, and check whether PR 389 is deployed. Separately verify why leadconnector-outlook-events-worker continues subscription attempts after reported integration teardown. Treat sampled log counts as samples, not request totals; low traffic amplifies the percentage but does not explain the failures.

## Cross-Validation

**Confidence**: high

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana metrics | Confirms route-specific impact: 19 failures among 41 requests in the latest alert's 15-minute window, versus a 0.279% seven-day baseline. Resource averages were normal; no restarts were observed. | ✓ |
| GCP request traces | At 01:42:57 UTC, one trace connects subscription recovery, torn-down integration state, OAuth HTTP 400 invalid_grant, classified status -1, and ERR_HTTP_INVALID_STATUS_CODE. A separate earlier trace repeats the chain. | ✓ |
| Source code and Graph SDK | Independently establishes the mechanism: Graph can wrap errors with status -1; classification accepts that value; subscribeMails forwards it to res.status. Recovery and refresh paths permit attempts after teardown. | ✓ |
| Slack context | Reports the same mechanism and proposed PR 389, but may repeat prior investigation evidence and is not independent causal confirmation. | ✓ |

### Discrepancies
- Slack describes token revocation; observed invalid_grant with AADSTS9002313 establishes an invalid request, not revocation specifically.
- The mechanism is verified for sampled requests, but not individually linked to all 19 metric-counted failures. Sampled log counts are not request totals.
- Kubernetes verification was unavailable. Normal resource averages and restart metrics weaken a capacity hypothesis but cannot exclude individual-pod pressure; event-loop evidence was not supplied.
- PR 389 was open and failures persisted at 03:46:02 UTC; exact deployed commit and fleet-wide rollout state remain unverified.
- Terminal teardown state is verified, but its original cause and timing, and whether subsequent worker events were fresh or redelivered, remain unresolved.

## Deep Dive Findings

{
  "workload": "leadconnector-api",
  "route": "GET /leadconnector/outlook/subscribe/:user_Id",
  "userId": "SK9wwSMoAbQ0F103QbGx",
  "summary": "Confirmed recurring error-handling defect: Microsoft token refresh returns HTTP 400 invalid_grant; Graph SDK wraps the failure with statusCode -1; classifyOutlookError accepts -1 and subscribeMails passes it to res.status, causing ERR_HTTP_INVALID_STATUS_CODE. Worker recovery continues for an already torn-down integration.",
  "confidence": {
    "invalid_status_mechanism": "high",
    "recovery_after_teardown": "high",
    "original_reason_credentials_became_unusable": "unresolved"
  },
  "request_level_evidence": {
    "trace": "projects/highlevel-backend/traces/23328801780886559",
    "time_ist": "2026-11-10 07:12:57",
    "time_utc": "2026-11-10T01:42:57Z",
    "sequence": [
      "01:42:57.535Z: worker identifies isSubscriptionBroken=true for the affected user and invokes subscription recovery.",
      "01:42:57.542Z: API logs KV2_READ_NO_ID_UNEXPECTED with isTornDown=true; token refresh nevertheless proceeds.",
      "01:42:57.712Z: Microsoft returns HTTP 400, invalid_grant, AADSTS9002313: Invalid request. Request is malformed or invalid.",
      "01:42:57.714Z: subscribeMails logs category=declared_4xx and statusCode=-1 for the same user.",
      "01:42:57.715Z: matching route and trace log ERR_HTTP_INVALID_STATUS_CODE; at 01:43:02Z the request also logs LONG_RUNNING_REQUEST."
    ],
    "independent_repetition": "The same refresh-failure-to-RangeError chain was verified at 2026-11-09T17:06:12Z under trace 23325468208211898.",
    "correction_to_prior_claim": "The observed OAuth response does not establish token revocation specifically. The precise invalid credential/request condition remains unverified."
  },
  "source_findings": {
    "status_validation": "classifyOutlookError checks that declaredStatus is a finite number but does not enforce an HTTP range. Consequently -1 becomes declared_4xx and reaches res.status unchanged.",
    "graph_wrapper": "Installed Graph SDK GraphErrorHandler defaults statusCode to -1 when wrapping an Error without an HTTP response status.",
    "continued_attempts": "Worker preProcessEmail invokes handleUserSubscription for messages absent from the email store. That function treats isSubscriptionBroken as a recovery trigger. Disconnect retains the integration row, marks it broken and clears keychainSecretId; it does not make this recovery path terminal.",
    "refresh_guard_gap": "getAccessToken checks integration identity but not terminal disconnection. Missing-secret handling permits torn-down integrations to continue, allowing another refresh attempt.",
    "runtime_confirmation": "The request trace logs both isTornDown=true and OUTLOOK_DISCONNECT_SKIPPED_ALREADY_TORNDOWN before the same invalid-status failure.",
    "source_url": "https://github.com/GoHighLevel/leadconnector/blob/8769ee52fb6236fe4fdf16aea9d4a3e2d1a01181/workers/leadconnector-outlook-events-worker.ts#L1543"
  },
  "pr_389": {
    "url": "https://github.com/GoHighLevel/leadconnector/pull/389",
    "state": "OPEN",
    "merged": false,
    "head_sha": "940ecbdeaaeaaf55c275f843e1242beb4723152f",
    "change": "Validates integer HTTP statuses within 100–599, recovers the wrapped Axios status from its message, and tests wrapped 400 plus unknown-error fallback.",
    "deployment_assessment": "Fix was not effective on sampled production requests: statusCode=-1 remained observable at 2026-11-10T03:46:02Z. PR is unmerged; exact deployed image/commit and fleet-wide rollout state remain unverified."
  },
  "scope_and_limits": [
    "Supplied metrics report 19 failures/41 requests; sampled log counts are not request totals. Low traffic amplifies the percentage but does not cause these failures.",
    "Historical DB investigation inv-2026-08-21-8b7cf0ac matches the invalid-status mechanism on Outlook deliver, with a different upstream trigger.",
    "Incoming worker processing is verified; whether repeated events are fresh notifications, redeliveries, or both was not established.",
    "No OUTLOOK_DISCONNECTED entry was returned for this user in the queried Nov 9–Nov 10 window. Existing torn-down state is verified, but its original transition time and cause are not.",
    "Used authenticated gcloud and GitHub CLI after the GitHub connector returned 404. Subagent launch failed in this executor. No production changes or messages were sent."
  ],
  "recommended_actions": [
    "Review and deploy PR 389, then verify the affected route no longer emits status -1 and handles this OAuth failure as a valid HTTP response.",
    "Separate terminal disconnection from recoverable subscription breakage; block subscription recovery and token refresh for terminally disconnected integrations.",
    "Trace event IDs and delivery attempts to distinguish stale notifications from redelivery amplification; investigate the original credential failure before attributing it to revocation."
  ],
  "log_queries": [
    {
      "purpose": "Request-level chain across worker and API; rerun with limit=5 verified results",
      "project": "highlevel-backend",
      "query": "resource.type=\"k8s_container\"\nresource.labels.container_name=(\"leadconnector-api\" OR \"leadconnector-outlook-events-worker\")\ntrace=\"projects/highlevel-backend/traces/23328801780886559\"\ntimestamp>=\"2026-11-10T01:42:50Z\"\ntimestamp<=\"2026-11-10T01:43:05Z\"",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D(%22leadconnector-api%22%20OR%20%22leadconnector-outlook-events-worker%22)%0Atrace%3D%22projects%2Fhighlevel-backend%2Ftraces%2F23328801780886559%22;timeRange=2026-11-10T01%3A42%3A50Z%2F2026-11-10T01%3A43%3A05Z?project=highlevel-backend"
    },
    {
      "purpose": "Later production persistence; limit=5 returned two classified -1 failures",
      "project": "highlevel-backend",
      "query": "resource.type=\"k8s_container\"\nresource.labels.container_name=\"leadconnector-api\"\njsonPayload.userId=\"SK9wwSMoAbQ0F103QbGx\"\njsonPayload.message=\"Error in outlook subscribeMails\"\ntimestamp>=\"2026-11-10T03:45:00Z\"\ntimestamp<=\"2026-11-10T04:00:00Z\"",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22leadconnector-api%22%0AjsonPayload.userId%3D%22SK9wwSMoAbQ0F103QbGx%22%0AjsonPayload.message%3D%22Error%20in%20outlook%20subscribeMails%22;timeRange=2026-11-10T03%3A45%3A00Z%2F2026-11-10T04%3A00%3A00Z?project=highlevel-backend"
    }
  ]
}

## Evidence

### Dashboards (incident window)
- [5xx percentage — leadconnector-api Outlook subscribe, reproduced 33.33% alert](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1794273215000&to=1794276815000&var-datasource=ber8nnhvgsjy8f&var-cluster=All&var-container=leadconnector-api&var-method=GET&var-route=%2Fleadconnector%2Foutlook%2Fsubscribe%2F%3Auser_Id&var-pod_name=All&viewPanel=14)
- [Request counts — leadconnector-api Outlook subscribe, 19 failures among 41 requests](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1794274115000&to=1794275015000&var-datasource=ber8nnhvgsjy8f&var-cluster=All&var-container=leadconnector-api&var-method=GET&var-route=%2Fleadconnector%2Foutlook%2Fsubscribe%2F%3Auser_Id&var-pod_name=All&viewPanel=6)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22leadconnector-api%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22Error%20in%20outlook%20subscribeMails%7CERR_HTTP_INVALID_STATUS_CODE%22;timeRange=2026-11-09T16%3A56%3A00Z%2F2026-11-09T17%3A12%3A00Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D(%22leadconnector-api%22%20OR%20%22leadconnector-outlook-events-worker%22)

### Related incidents
- [Current alert thread reports 34 failures for one revoked Outlook token — 9 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794244000767129?thread_ts=1794243905.554859)
- [Earlier same-route alert #139335 contains RCA and proposed fix — 9 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794243487093879?thread_ts=1794237410.689039)

## Investigation Timeline

**Deep dive passes**: 1
**Worker calls**: 10

**Total worker time**: 564.8s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 14.2s | ✓ |
| grafana-worker | worker | 74.7s | ✓ |
| kubectl-worker | worker | 90.3s | ✓ |
| log-explorer-worker | worker | 80.9s | ✓ |
| slack-worker | worker | 43.5s | ✓ |
| triage | worker | 12.2s | ✓ |
| triage_llm_ranking | decision | 63.5s | ✓ |
| deep_dive_pass_1 | worker | 156.9s | ✓ |
| follow_up_check | decision | 9.7s | ✓ |
| cross_validation | worker | 18.9s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IDG6WIAXLS9NG|#139353 5XXPercentagePerAPI>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=10727950922855835787&alert_id=14607817006474276545|source>*) app: leadconnector-api route: /leadconnector/outlook/subscribe/:user_Id method: GET summary: 5XX Error Rate Above Threshold - Per API logs_link: <https://console.cloud.google.com/logs/query;query=resource.labels.container_name%3D%22leadconnector-api%22%0AhttpRequest.status%3E%3D500;duration=PT30M?project=highlevel-backend|here> threshold: 1% current_value: 33.33333333333333 dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&from=now-15m&to=now&viewPanel=14&var-container=leadconnector-api|here> :package: Showing the last alert only out of 28 total. Visit <https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IDG6WIAXLS9NG|the plugin page> to see them all. *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IDG6WIAXLS9NG|#139353 5XXPercentagePerAPI>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=10727950922855835787&amp;alert_id=14607817006474276545|source>*) app: leadconnector-api route: /leadconnector/outlook/subscribe/:user_Id method: GET summary: 5XX Error Rate Above Threshold - Per API logs_link: <https://console.cloud.google.com/logs/query;query=resource.labels.container_name%3D%22leadconnector-api%22%0AhttpRequest.status%3E%3D500;duration=PT30M?project=highlevel-backend|here> threshold: 1% current_value: 33.33333333333333 dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-15m&amp;to=now&amp;viewPanel=14&amp;var-container=leadconnector-api|here> :package: Showing the last alert only out of 28 total. Visit <https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IDG6WIAXLS9NG|the plugin page> to see them all.
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-10T03:54:47.512Z*