# Alert Investigation: PubSubUnackedMessages

**Investigation ID**: inv-2026-11-13-e34535ac
**Workload**: conversations-providers-deliver-outlook-events-dl
**Container**: conversations-providers-deliver-outlook-events-dl
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-13T14:52:05.344Z
**Status**: delivering

## Root Cause

**Summary**: Failed Microsoft Graph sendMail calls were followed by Leadconnector POST /outlook/deliver HTTP 503 responses; worker retries and dead-letter forwarding are supported by logs and subscription metrics. The underlying Graph failure and the cause of the full DL backlog remain unresolved. No deployment cause was established.
**Confidence**: medium
**Category**: upstream_dependency_failure_unresolved

## Triage

**Primary Anomaly**: none
**Details**: The Outlook dead-letter subscription has 4,558 undelivered messages, far above its seven-day median of 11, and the count was flat in the latest three hourly samples. No DL consumer is deployed, which is expected; the related main worker is healthy at 2/2 replicas with zero restarts. Available data does not establish when messages accumulated or whether dead-lettering is still active.
**Anomalous**: Yes
**Deep Dive Direction**: YES — The triage direction needs updating: the evidence shows active dead-lettering during the backlog ramp, correlated with repeated HTTP 503s from `leadconnector-api` at `POST /outlook/deliver`. Focus on why that API returns 503 and how those failures lead to nacks and dead-lettering; the earlier check of DL age and main-subscription metrics has already provided useful evidence.

More data is needed to determine the root cause: inspect `leadconnector-api` logs and code for the failing requests, and check deployment or incident history around the backlog ramp. The exact cause of the 503s is not established.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| Cloud Monitoring / Grafana | The Outlook DL subscription reached 4,558 undelivered messages versus a seven-day median of 11. Main-subscription metrics for ack, nack, and expired deadlines were unavailable, so this source confirms the backlog but not its cause. | ✓ |
| Leadconnector API production logs | Four matched message IDs show Microsoft Graph response errors followed 48–116 ms later by worker HTTP 503 responses. The Graph status, error body, request ID, and Retry-After were not logged. | ✓ |
| Outlook worker logs and subscription metrics | Repeated delivery attempts and 566 dead-letter forwards were observed during 2026-11-12 18:30–19:15 UTC, alongside 690 acknowledgments. This supports retry and dead-letter forwarding, but does not establish the underlying Graph failure. | ✓ |
| Kubernetes workload health | The related main worker was 2/2 replicas with zero restarts. No DL deployment or consumer was found, which is expected. No causal deployment or lifecycle event was established. | ✓ |

### Discrepancies
- Triage said active dead-lettering was correlated with repeated 503s; deep-dive evidence supports a failure path, but the queried windows differ and do not establish that these 503s explain the entire 4,558-message backlog.
- The DL backlog was flat in the latest hourly samples, while the DL subscription showed no consumption in a separate queried window; current dead-lettering activity is undetermined.
- Ack, nack, and expired-deadline metrics returned no series in the Layer 1 query, limiting independent validation of retry behavior.
- The Graph failure cannot be classified as throttling or provider error from available logs.

## Deep Dive Findings

{
  "summary": "The Outlook DL backlog is being fed by failed Microsoft Graph sendMail calls: matching Graph errors are followed by Leadconnector HTTP 503 responses, worker retries, and dead-letter forwarding. The underlying Graph failure—throttling versus provider error—remains unresolved.",
  "confidence": {
    "failure_path": "high",
    "underlying_graph_cause": "unresolved",
    "deployment_causation": "not established"
  },
  "findings": {
    "failure_path": "Outlook delivery worker → Leadconnector POST /outlook/deliver → Microsoft Graph /me/sendMail. Four matched message IDs had Graph response-error logs followed by worker HTTP 503 logs within 48–116 ms.",
    "retry_and_dead_letter": {
      "source_subscription": "conversations-providers-deliver-outlook-events",
      "max_delivery_attempts": 5,
      "minimum_backoff_seconds": 10,
      "maximum_backoff_seconds": 600,
      "evidence": "Repeated delivery attempts were observed. During 2026-11-12 18:30–19:15 UTC, the source subscription recorded 690 acknowledgments and 566 dead-letter forwards. The DL subscription showed no consumption in the queried 2026-11-13 02:30–04:35 UTC window."
    },
    "why_503_is_not_specific": "Leadconnector’s local Outlook error classifier maps both Graph 429 and Graph 5xx to HTTP 503. The matched production logs omit Graph status, error code/body, request ID, response headers, and Retry-After; therefore the 503 does not establish a Microsoft outage or throttling.",
    "deployment_and_health": "The source worker and Leadconnector API remained ready with zero restart increase during the investigated 2026-11-12 17:00–19:15 UTC window. Pod creation preceded the error ramp by several hours. No matching lifecycle events or causal deployment/config change were established. The incident image and deployed dependency versions could not be verified.",
    "negative_evidence": "No Keychain-specific exception was demonstrated. Local code comments mentioning KV2 are not evidence for this incident. The bounded sample covered 78 message IDs and 57 user IDs; it did not establish tenant concentration or abuse."
  },
  "evidence_logs": [
    {
      "timestamp_utc": "2026-11-12T19:02:59.767734527Z",
      "insert_id": "1osfouvf9aob6a",
      "message_id": "lXx1pZSmO4gU2UYKcnJR",
      "finding": "Leadconnector Graph response error; matching worker HTTP 503 followed at 19:02:59.867235422Z."
    },
    {
      "timestamp_utc": "2026-11-12T19:02:58.250930786Z",
      "insert_id": "ixcdxtf999hhq",
      "message_id": "Kp2noUDGk6KRfAqB5EJy",
      "finding": "Leadconnector Graph response error; matching worker HTTP 503 followed at 19:02:58.298697710Z."
    },
    {
      "timestamp_utc": "2026-11-12T19:02:58.106895685Z",
      "insert_id": "1ggzz0lf2fynfx",
      "message_id": "hqiXPH0PyG4AQgiiBUEz",
      "finding": "Leadconnector Graph response error; matching worker HTTP 503 followed at 19:02:58.222632646Z."
    },
    {
      "timestamp_utc": "2026-11-12T19:02:57.321831226Z",
      "insert_id": "1grt0tkf8okyyw",
      "message_id": "Efa6QP9oJnUyFGNPC1OC",
      "finding": "Leadconnector Graph response error; matching worker HTTP 503 followed at 19:02:57.391286373Z."
    }
  ],
  "log_queries_used": [
    {
      "purpose": "Leadconnector Outlook send errors",
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"leadconnector-api\" severity>=ERROR jsonPayload.message=\"Failed to send mail to Outlook\"",
      "window_utc": "2026-11-12T18:53:00Z to 2026-11-12T19:03:00Z",
      "limit": 30
    },
    {
      "purpose": "Source worker transient nacks",
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-providers-deliver-outlook-worker\" severity>=ERROR jsonPayload.message=\"[DELIVER_OUTLOOK_TRANSIENT_NACK]\"",
      "window_utc": "2026-11-12T18:53:00Z to 2026-11-12T19:03:00Z",
      "limit": 4
    }
  ],
  "log_explorer_urls": [
    "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22leadconnector-api%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D%22Failed%20to%20send%20mail%20to%20Outlook%22;timeRange=2026-11-12T18%3A53%3A00Z%2F2026-11-12T19%3A03%3A00Z?project=highlevel-backend",
    "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-providers-deliver-outlook-worker%22%0Aseverity%3DERROR%0AjsonPayload.message%3D%22%5BDELIVER_OUTLOOK_TRANSIENT_NACK%5D%22;timeRange=2026-11-12T18%3A53%3A00Z%2F2026-11-12T19%3A03%3A00Z?project=highlevel-backend"
  ],
  "limitations": [
    "Original Graph response diagnostics were not present in the queried logs.",
    "The live incident image, deployed dependency versions, and a causal deployment change were not verified.",
    "No new orchestrator investigation was launched and no production changes or messages were made."
  ],
  "next_evidence_needed": "A retained failed-send record or provider diagnostic with Graph status, error code/body, request ID, response headers, and Retry-After; incident image metadata would help verify the deployed logger and SDK."
}

## Evidence

### Dashboards (incident window)
- [Undelivered messages — conversations-providers-deliver-outlook-events-dl (7-day rise to 4,558; panel 1)](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc/conversations-workers-health-overview?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=conversations-providers-deliver-outlook-events-dl&from=1793976856000&to=1794581656000&viewPanel=1)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-providers-deliver-outlook-events-dl%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22error%7Cfailed%7Ctimeout%7CECONNREFUSED%7CDEADLINE_EXCEEDED%22;timeRange=2026-11-13T14%3A47%3A00Z%2F2026-11-13T14%3A57%3A00Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22leadconnector-api%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D%22Failed%20to%20send%20mail%20to%20Outlook%22;timeRange=2026-11-12T18%3A53%3A00Z%2F2026-11-12T19%3A03%3A00Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-providers-deliver-outlook-worker%22%0Aseverity%3DERROR%0AjsonPayload.message%3D%22%5BDELIVER_OUTLOOK_TRANSIENT_NACK%5D%22;timeRange=2026-11-12T18%3A53%3A00Z%2F2026-11-12T19%3A03%3A00Z?project=highlevel-backend)

## Investigation Timeline

**Deep dive passes**: 2
**Worker calls**: 11

**Total worker time**: 1098.6s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 18.6s | ✓ |
| grafana-worker | worker | 92.1s | ✓ |
| kubectl-worker | worker | 236.3s | ✓ |
| log-explorer-worker | worker | 109.1s | ✓ |
| triage | worker | 14.6s | ✓ |
| triage_llm_ranking | decision | 23.8s | ✓ |
| deep_dive_pass_1 | worker | 393.3s | ✓ |
| follow_up_check | decision | 10.2s | ✓ |
| deep_dive_pass_2 | worker | 172.8s | ✓ |
| follow_up_check | decision | 11.7s | ✓ |
| cross_validation | worker | 16.1s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/INTLIPK5THGB8|#139801 Pubsub Unacked Messages above 500>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/f7ce4c3a-4497-4dbf-aa4e-9a751067fe04/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: conversations-providers-deliver-outlook-events-dl Unacked Messages: 4558 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/conversations-providers-deliver-outlook-events-dl?project=highlevel-backend> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/INTLIPK5THGB8|#139801 Pubsub Unacked Messages above 500>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/f7ce4c3a-4497-4dbf-aa4e-9a751067fe04/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: conversations-providers-deliver-outlook-events-dl Unacked Messages: 4558 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/conversations-providers-deliver-outlook-events-dl?project=highlevel-backend>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-13T15:08:11.926Z*