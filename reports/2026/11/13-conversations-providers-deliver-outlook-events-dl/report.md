# Alert Investigation: PubSubUnackedMessages

**Investigation ID**: inv-2026-11-13-88ac049f
**Workload**: conversations-providers-deliver-outlook-events-dl
**Container**: conversations-providers-deliver-outlook-events-dl
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-12T19:02:30.892Z
**Status**: delivering

## Root Cause

**Summary**: Outlook delivery HTTP 503 failures triggered explicit source-worker nacks and ongoing dead-letter accumulation, corroborated by worker logs and PubSub metrics. Concurrent Graph sendMail response errors support an upstream Graph failure path, but its exact cause remains unresolved. Recover matched Graph response diagnostics before selecting a provider-specific mitigation; assess replay after delivery failures recover.
**Confidence**: medium
**Category**: outlook_delivery_transient_failures_with_dead_letter_accumulation

## Triage

**Primary Anomaly**: error_spike
**Details**: Outlook source-worker HTTP 503 transient failures are driving explicit nacks and active dead-letter accumulation. During 18:30–19:15 UTC November 12, the source recorded 30,675 nack requests, 690 acknowledgments and 566 dead-lettered messages; a targeted sample contained 200 HTTP 503 errors across 78 message IDs. DL backlog reached 3,852 at 04:34 UTC November 13 despite OnCall resolution. Historical stale-DLQ and throughput-ceiling RCAs do not match ongoing failures; candidate 7 shares the nack-to-DLQ mechanism but not the demonstrated exception path. Grafana reports two ready source pods and zero restarts; kubectl verification was unavailable.
**Anomalous**: Yes
**Deep Dive Direction**: YES — The triage direction remains supported: Graph sendMail failures → Leadconnector HTTP 503 → retries → dead-letter accumulation. The underlying Graph cause remains unresolved; HTTP 503 does not distinguish Graph throttling from provider failure.

Need a matched failure’s original Graph statusCode, error code/body, request ID, response headers and Retry-After, plus verification of deployed SDK/logger versions. The past learning reinforces validating causal links; it does not contradict this direction, but the logging gap must not be mistaken for the failure’s root cause.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| PubSub metrics | During 18:30–19:15 UTC November 12, the source subscription recorded 30,675 nack requests and 566 dead-lettered messages. DL backlog continued growing to 3,852 by November 13 04:34 UTC. | ✓ |
| Worker logs | A targeted sample contained 200 HTTP 503 transient-nack errors across 78 message IDs, overlapping the measured nack and dead-letter activity. | ✓ |
| API logs and local source | API logs show Graph response-error stacks during the same window. Local code maps both Graph 429 and 5xx responses to HTTP 503; the original Graph response and deployed implementation remain unverified. | ✓ |
| Grafana pod health | Two ready source-worker pods and zero restarts over 12 hours weaken the crash-loop hypothesis. Active dead-letter forwarding disproves a purely stale-DLQ explanation. | ✓ |

### Discrepancies
- No request-level identity join independently links a worker nack to its original Graph response; throttling versus provider failure remains unresolved.
- Grafana and log-investigation PubSub counts share the same underlying metric source and are not independent corroboration.
- OnCall resolution conflicts with subsequent backlog growth; the reported alert value of 3,707 was not reproduced at its exact timestamp.
- Kubernetes verification timed out. Deployment timing, deployed dependency versions, and DL consumer absence remain unconfirmed.
- Missing deliveryAttempt fields do not rule out retries. The targeted log sample cannot establish population-wide error proportions or tenant abuse.
- The longstanding oldest-message age is separate from the new backlog growth; missing Graph diagnostics are an observability gap, not a demonstrated failure cause.

## Deep Dive Findings

{
  "summary": "Graph sendMail failures remain supported, but the original Graph response and deployed dependency versions remain unresolved. New evidence shows local installed and lockfile logger versions differ, so local serializer behavior cannot establish the exact deployed implementation.",
  "confidence": {
    "upstream_failure_path": "high, based on supplied triage and reverified Graph failure logs",
    "underlying_graph_cause": "unresolved",
    "deployed_logger_attribution": "unverified"
  },
  "findings": [
    "Requeried 10 error entries across five API pods at 19:02:57–19:03:01 UTC. All contain GraphErrorHandler.constructErrorFromResponse stacks and reduced error objects; none exposes the requested Graph diagnostics.",
    "Production log resource labels identify servers-usc1-prd-cluster and ReplicaSet prefix leadconnector-api-569676c8d7. These identify the workload, not its dependency versions.",
    "Local installed Graph SDK is 3.0.4 and logger is 3.7.1; yarn.lock resolves Graph SDK 3.0.4 and logger 3.12.2. package.json requests logger ^3.11.2.",
    "Dockerfile copies node_modules from a separately tagged leadconnector-node-modules image. Verifying deployed dependencies requires the incident image and its base-image provenance or package metadata.",
    "Local classification maps Graph 429 and Graph 5xx to HTTP 503. Observed application 503 responses therefore do not distinguish throttling from provider failure. Missing diagnostics are an observability limitation, not the demonstrated cause of delivery failure."
  ],
  "original_graph_response": {
    "statusCode": null,
    "code": null,
    "body": null,
    "requestId": null,
    "responseHeaders": null,
    "retryAfter": null,
    "interpretation": "Not recovered from queried logs; absence does not establish that Microsoft omitted these fields."
  },
  "deployed_versions": {
    "graph_sdk": null,
    "logger": null,
    "image_digest": null,
    "blocker": "Read-only lookup of implicated pod leadconnector-api-569676c8d7-wjsfp in the correct servers cluster timed out connecting to the Kubernetes API."
  },
  "evidenceLogs": [
    {
      "timestamp": "2026-11-12T19:02:59.767734527Z",
      "insertId": "1osfouvf9aob6a",
      "finding": "Graph response-error stack; requested original response fields absent."
    },
    {
      "timestamp": "2026-11-12T19:02:58.250930786Z",
      "insertId": "ixcdxtf999hhq",
      "finding": "Same Graph response-error stack and reduced error object."
    },
    {
      "timestamp": "2026-11-12T19:02:58.106895685Z",
      "insertId": "1ggzz0lf2fynfx",
      "finding": "Same failure signature on another API pod."
    },
    {
      "timestamp": "2026-11-12T19:02:57.321831226Z",
      "insertId": "1grt0tkf8okyyw",
      "finding": "Same failure signature on another API pod."
    }
  ],
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"leadconnector-api\" timestamp>=\"2026-11-12T19:02:57Z\" timestamp<=\"2026-11-12T19:03:01Z\" severity>=ERROR jsonPayload.message=~\"Failed to send mail to Outlook|OUTLOOK_SENDMAILS_OUTBOUND_FAILED\"",
      "limit": 15,
      "returned": 10
    },
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"leadconnector-api\" timestamp>=\"2026-11-12T19:02:57Z\" timestamp<=\"2026-11-12T19:03:01Z\" insertId=\"1osfouvf9aob6a\"",
      "limit": 1,
      "returned": 1,
      "purpose": "Explicit projection of Graph diagnostics, response headers and version fields."
    }
  ],
  "logExplorerUrls": [
    "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22leadconnector-api%22%0AinsertId%3D%221osfouvf9aob6a%22;timeRange=2026-11-12T19%3A02%3A57Z%2F2026-11-12T19%3A03%3A01Z?project=highlevel-backend"
  ],
  "limitations": [
    "Reused supplied histogram, PubSub and pod-health findings; did not repeat broad triage.",
    "No new domain-identity join independently matched a worker nack to its original Graph response.",
    "No causal deployment change, Microsoft outage or throttling diagnosis was established."
  ],
  "next_evidence_needed": "Incident-image dependency metadata and a failed send's allowlisted Graph status, error code/body, request-id, client-request-id and Retry-After, obtained from retained provider diagnostics or response-level instrumentation.",
  "access": "Existing gcloud CLI authentication, read-only Kubernetes queries and local source inspection. No production changes, messages or new orchestrator investigation."
}

## Evidence

### Dashboards (incident window)
- [DLQ backlog — conversations-providers-deliver-outlook-events-dl (growth to 3,852)](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=conversations-providers-deliver-outlook-events-dl&from=1794502800000&to=1794544520073&viewPanel=1)
- [Nacks — conversations-providers-deliver-outlook-events (elevated retries)](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=conversations-providers-deliver-outlook-events&from=1794502800000&to=1794544520073&viewPanel=6)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-providers-deliver-outlook-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D%22%5BDELIVER_OUTLOOK_TRANSIENT_NACK%5D%22;timeRange=2026-11-12T18%3A53%3A00Z%2F2026-11-12T19%3A03%3A00Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22leadconnector-api%22%0AinsertId%3D%221osfouvf9aob6a%22;timeRange=2026-11-12T19%3A02%3A57Z%2F2026-11-12T19%3A03%3A01Z?project=highlevel-backend)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 12

**Total worker time**: 731.9s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 15.2s | ✓ |
| grafana-worker | worker | 84.0s | ✓ |
| kubectl-worker | worker | 97.3s | ✓ |
| log-explorer-worker | worker | 85.5s | ✓ |
| triage | worker | 14.7s | ✓ |
| triage_llm_ranking | decision | 26.0s | ✓ |
| deep_dive_pass_1 | worker | 196.4s | ✓ |
| follow_up_check | decision | 11.4s | ✓ |
| deep_dive_pass_2 | worker | 75.6s | ✓ |
| follow_up_check | decision | 10.5s | ✓ |
| deep_dive_pass_3 | worker | 98.5s | ✓ |
| cross_validation | worker | 17.0s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IHSLQ94S2LWX9|#139723 Pubsub Unacked Messages above 500>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/f7ce4c3a-4497-4dbf-aa4e-9a751067fe04/view?orgId=1|source>*) AlertValues: Doc for this alert: http://platform.docs/alerts/pubsub-unacked-messages/ Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: conversations-providers-deliver-outlook-events-dl Unacked Messages: 3707 GCP Link: https://console.cloud.google.com/cloudpubsub/subscription/detail/conversations-providers-deliver-outlook-events-dl?project=highlevel-backend :package: Showing the last alert only out of 17 total. Visit <https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IHSLQ94S2LWX9|the plugin page> to see them all. *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IHSLQ94S2LWX9|#139723 Pubsub Unacked Messages above 500>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/f7ce4c3a-4497-4dbf-aa4e-9a751067fe04/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: conversations-providers-deliver-outlook-events-dl Unacked Messages: 3707 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/conversations-providers-deliver-outlook-events-dl?project=highlevel-backend> :package: Showing the last alert only out of 17 total. Visit <https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IHSLQ94S2LWX9|the plugin page> to see them all.
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-13T04:43:38.443Z*