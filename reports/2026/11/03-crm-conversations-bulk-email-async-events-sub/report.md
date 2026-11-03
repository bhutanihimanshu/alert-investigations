# Alert Investigation: PubSubUnackedMessages

**Investigation ID**: inv-2026-11-03-a0790d3b
**Workload**: crm-conversations-bulk-email-async-events-sub
**Container**: crm-conversations-bulk-email-async-events-sub
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-03T15:18:31.749Z
**Status**: delivering

## Root Cause

**Summary**: Root cause remains unconfirmed. A publish burst exceeding effective consumer throughput, potentially prolonged by processing or callback latency, is plausible but not independently corroborated. Validate aligned publish, delivery, ack, nack and latency timelines before attributing the backlog to traffic or a specific dependency.
**Confidence**: low
**Category**: undetermined

## Triage

**Primary Anomaly**: none
**Details**: Alert reports 1,145,220 unacked messages on crm-conversations-bulk-email-async-events-sub at 2026-11-03T15:18:31.749Z, 14.522% above the 1,000,000 threshold. The backlog is unverified: Grafana DNS failed and GCP/Kubernetes authentication was blocked. No CPU, memory, traffic or error anomaly is established; empty findings do not indicate health. Candidate 10 concerns the same subscription but a much smaller, verified transient surge; current evidence cannot establish recurrence.
**Anomalous**: Yes
**Deep Dive Direction**: YES — The triage direction remains correct: a publish burst exceeding effective consumer throughput, compounded by increased processing duration, is the leading hypothesis. Redis and template-download errors do not establish the bottleneck; root cause remains unconfirmed.

Need dependency latency and per-message timings, pre-burst error baselines through recovery, deployed middleware/config verification, rollout/restart/throttling history, completed redelivery/nack analysis, and active subscriber counts across clusters.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Deep-dive Grafana metrics | Zero sampled restarts, no observed image transition, and no aggregate throttling increase weaken crash-loop and CPU-throttling explanations. They do not establish a publish burst or processing bottleneck. | ✗ |
| Deep-dive GCP logs | Template-download and IAM Redis errors occurred before onset and during drainage. No normalized error-rate or dependency-latency correlation establishes that these errors caused the backlog. | ✗ |
| Local source inspection | Acknowledgment can await processing and callback completion, providing a plausible slowdown mechanism. Incident-image equivalence and actual processing timings remain unverified. | ✗ |
| Layer 1 | Access failures prevented independent verification of backlog, throughput and pod state. Missing results cannot corroborate health or root cause. | ✗ |

### Discrepancies
- The publish-burst hypothesis lacks supplied publish-rate, ack-rate and processing-duration time series demonstrating the causal sequence.
- The cited previous-pass sent/ack ratio of approximately 1.02 lacks underlying samples and an exact window; nack and redelivery checks remain incomplete.
- Container inventory is not active subscriber capacity; zero restarts does not exclude pod replacement or eviction.
- Dependency errors outside onset weaken error-presence attribution but do not exclude latency degradation during the incident.
- Later browser access partially overcame initial access failures, but two independent sources still do not establish the proposed cause.

## Deep Dive Findings

{
  "status": "partial",
  "subscription": "crm-conversations-bulk-email-async-events-sub",
  "alertTimestamp": "2026-11-03T15:18:31.749Z",
  "summary": "The publish-burst hypothesis remains strongest. This pass found no restart activity, no observed image change, and no increase in aggregate CPU throttling relative to baseline. The dependency responsible for increased processing duration remains unconfirmed.",
  "access": {
    "path": "Existing main Chrome session: Grafana datasource API and GCP Logs Explorer; read-only local source, state.json and debug.json.",
    "parallelAgents": "Both required launches failed because the executor could not load parent-thread context."
  },
  "newFindings": [
    {
      "finding": "Restart metric remained zero throughout 14:45–16:15 UTC at five-minute sampling.",
      "cluster": "servers-usc1-prd-cluster",
      "query": "sum by (cluster) (increase(kube_pod_container_status_restarts_total{container=\"conversations-bulk-internal-async-api\"}[5m]))",
      "interpretation": "Evidence against a container crash loop. Does not exclude pod replacement or eviction."
    },
    {
      "finding": "Container inventory showed 450 at 14:45 UTC, 260 at 14:50–15:00, and 450 from 15:05 through 16:15.",
      "query": "count by (cluster) (kube_pod_container_info{container=\"conversations-bulk-internal-async-api\"})",
      "interpretation": "A pre-burst reduction followed by scale-up is verified. Only servers-usc1-prd-cluster returned a series; container inventory does not establish active subscriber counts or complete cross-cluster coverage."
    },
    {
      "finding": "Aggregate throttled-period fraction was approximately 3.6–5.5% before the burst and 2.3–5.4% during 15:05–15:55 UTC.",
      "query": "sum by (cluster) (rate(container_cpu_cfs_throttled_periods_total{container=\"conversations-bulk-internal-async-api\"}[5m])) / sum by (cluster) (rate(container_cpu_cfs_periods_total{container=\"conversations-bulk-internal-async-api\"}[5m]))",
      "interpretation": "No aggregate throttling increase explains the slowdown. Individual-pod hotspots remain possible."
    },
    {
      "finding": "Historical container inventory returned one image digest across the sampled incident window.",
      "image": "sha256:4d9415c74f259f49eacc616b67b0f07db81526762fc122ed5d17e0fbb0ff9e5b",
      "query": "count by (cluster,image) (kube_pod_container_info{container=\"conversations-bulk-internal-async-api\"})",
      "interpretation": "No observed image transition. The digest was not mapped to a commit or middleware version; configuration-only changes remain unverified."
    },
    {
      "finding": "The same targeted dependency-error filter returned 10 entries before the burst and 37 during late drainage.",
      "baselineWindow": "2026-11-03T14:50:00Z/2026-11-03T15:00:00Z",
      "drainageWindow": "2026-11-03T15:45:00Z/2026-11-03T15:55:00Z",
      "interpretation": "Template-download and IAM Redis errors existed before onset and persisted during recovery. Their presence alone is insufficient causal evidence; these counts are not normalized by processed volume."
    }
  ],
  "sourceFindings": {
    "retryObservability": "Local middleware logs Handling message with messageId/path at INFO; the full received message is logged at codeDebug. The generic deliveryAttempt:[2-9] text search is therefore insufficient to exclude redelivery.",
    "ackTiming": "Local middleware awaits HTTP or Pub/Sub callback delivery before acknowledging, unless successful-response callback skipping is enabled. Ack latency can include callback time, not just sendBulkEmails execution.",
    "templateDownload": "bulk.service.ts:314 awaits axios.get(emailTemplateDownloadUrl) without a per-call timeout option. No measured download duration establishes this call as the incident bottleneck.",
    "deploymentCaveat": "These are local-source observations; equivalence to the incident image remains unverified."
  },
  "evidenceLogs": [
    {
      "timestamp": "2026-11-03T14:50:01.012Z",
      "message": "Failed to download email template",
      "significance": "Pre-burst occurrence."
    },
    {
      "timestamp": "2026-11-03T14:51:10.880Z",
      "message": "IAM_REDIS (Cluster) node error",
      "significance": "Pre-burst occurrence."
    },
    {
      "timestamp": "2026-11-03T15:45:49.687Z",
      "message": "Failed to download email template",
      "significance": "Persists during late drainage."
    },
    {
      "timestamp": "2026-11-03T15:45:50.918Z",
      "message": "IAM_REDIS (Cluster) node error",
      "significance": "Persists during late drainage."
    }
  ],
  "logQueriesUsed": [
    {
      "query": "resource.type=\"k8s_container\"\nresource.labels.container_name=\"conversations-bulk-internal-async-api\"\nseverity>=ERROR\njsonPayload.message=~\"Error processing message|Callback delivery failed|Failed to download email template|companies-locations-cluster-cache|IAM_REDIS\"",
      "windows": [
        {
          "start": "2026-11-03T14:50:00Z",
          "end": "2026-11-03T15:00:00Z",
          "displayedResults": 10
        },
        {
          "start": "2026-11-03T15:45:00Z",
          "end": "2026-11-03T15:55:00Z",
          "displayedResults": 37
        }
      ]
    },
    {
      "query": "resource.type=\"k8s_container\"\nresource.labels.container_name=\"conversations-bulk-internal-async-api\"\n(jsonPayload.message=~\"deliveryAttempt:[2-9]|Error processing message|Callback delivery failed\" OR jsonPayload.metadata.meta.deliveryAttempt>1)",
      "start": "2026-11-03T15:00:00Z",
      "end": "2026-11-03T15:10:00Z",
      "result": "Query remained loading with approximately zero displayed results. Inconclusive; not evidence of zero retries."
    }
  ],
  "logExplorerUrls": [
    "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-bulk-internal-async-api%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22Error%20processing%20message%7CCallback%20delivery%20failed%7CFailed%20to%20download%20email%20template%7Ccompanies-locations-cluster-cache%7CIAM_REDIS%22;timeRange=2026-11-03T14%3A50%3A00Z%2F2026-11-03T15%3A00%3A00Z?project=highlevel-backend",
    "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-bulk-internal-async-api%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22Error%20processing%20message%7CCallback%20delivery%20failed%7CFailed%20to%20download%20email%20template%7Ccompanies-locations-cluster-cache%7CIAM_REDIS%22;timeRange=2026-11-03T15%3A45%3A00Z%2F2026-11-03T15%3A55%3A00Z?project=highlevel-backend"
  ],
  "rootCause": {
    "confirmed": false,
    "leadingHypothesis": "Publish burst exceeded effective consumer throughput; increased message-processing and/or callback duration prolonged drainage.",
    "confidence": "medium",
    "retryAmplification": "Previous-pass sent/ack ratio near 1.02 argues against substantial amplification in that sampled window; nack metrics remain unverified."
  },
  "remainingGaps": [
    "Per-message dependency and callback timings, including template-download latency.",
    "Incident image-to-commit mapping, deployed middleware/configuration and configuration-only rollout history.",
    "Completed redelivery analysis, nack metrics and identity-skew analysis.",
    "Active subscriber counts across clusters and exact alert-evaluation attribution.",
    "Continuous error histogram through onset and recovery; this pass verified two targeted comparison windows only."
  ],
  "investigationComplete": false,
  "productionChanges": false,
  "messagesPosted": false,
  "orchestratorLaunched": false
}

## Evidence

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-bulk-internal-async-api%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22Error%20processing%20message%7CCallback%20delivery%20failed%7CFailed%20to%20download%20email%20template%7Ccompanies-locations-cluster-cache%7CIAM_REDIS%22;timeRange=2026-11-03T14%3A50%3A00Z%2F2026-11-03T15%3A00%3A00Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-bulk-internal-async-api%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22Error%20processing%20message%7CCallback%20delivery%20failed%7CFailed%20to%20download%20email%20template%7Ccompanies-locations-cluster-cache%7CIAM_REDIS%22;timeRange=2026-11-03T15%3A45%3A00Z%2F2026-11-03T15%3A55%3A00Z?project=highlevel-backend)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 12

**Total worker time**: 709.6s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 13.2s | ✓ |
| grafana-worker | worker | 18.7s | ✓ |
| kubectl-worker | worker | 19.3s | ✓ |
| log-explorer-worker | worker | 27.9s | ✓ |
| triage | worker | 11.3s | ✓ |
| triage_llm_ranking | decision | 24.2s | ✓ |
| deep_dive_pass_1 | worker | 216.5s | ✓ |
| follow_up_check | decision | 9.7s | ✓ |
| deep_dive_pass_2 | worker | 189.7s | ✓ |
| follow_up_check | decision | 9.4s | ✓ |
| deep_dive_pass_3 | worker | 149.6s | ✓ |
| cross_validation | worker | 20.2s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I3TDQGPILG1WW|#138298 Pubsub Unacked Messages above 1million >* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/afbf342d-1b2a-40c7-8344-621ddb6a4ba4/view?orgId=1|source>*) AlertValues: Doc for this alert: http://platform.docs/alerts/pubsub-unacked-messages/ Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-bulk-email-async-events-sub Unacked Messages: 1145220 GCP Link: https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-bulk-email-async-events-sub?project=highlevel-backend :package: Showing the last alert only out of 2 total. Visit <https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I3TDQGPILG1WW|the plugin page> to see them all. *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I3TDQGPILG1WW|#138298 Pubsub Unacked Messages above 1million >* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/afbf342d-1b2a-40c7-8344-621ddb6a4ba4/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-bulk-email-async-events-sub Unacked Messages: 1145220 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-bulk-email-async-events-sub?project=highlevel-backend> :package: Showing the last alert only out of 2 total. Visit <https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I3TDQGPILG1WW|the plugin page> to see them all.
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-03T22:11:38.084Z*