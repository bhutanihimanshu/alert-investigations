# Alert Investigation: PubSubUnackedMessages

**Investigation ID**: inv-2026-11-16-e99257ba
**Workload**: crm-conversations-last-call-events-sub
**Container**: crm-conversations-last-call-events-sub
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-16T12:47:31.395Z
**Status**: delivering

## Root Cause

**Summary**: Root cause is undetermined. Continued acknowledgements and healthy pods argue against a stopped or crashing consumer. Reaching the HPA ceiling while processing events with search-index writes is consistent with a throughput limit, but available evidence does not show that publishing exceeded processing or identify a downstream slowdown. The July ClickHouse throughput case is not supported for this worker.
**Confidence**: low
**Category**: undetermined

## Triage

**Primary Anomaly**: traffic_spike
**Details**: The alert reported 1,173 undelivered messages against a threshold of 500. Pods were healthy, and the HPA scaled up as backlog rose; ack activity continued. Queried backlog aggregates were lower than the alert value, so the backlog trajectory and trigger are unconfirmed. Logs were queried using the subscription name as the container name and returned no data.
**Anomalous**: Yes
**Deep Dive Direction**: Use the July 6 CRM subscriber throughput-ceiling case as a lead. For this last-call-events worker, compare publish rate with ack/drain rate over the alert window, determine whether backlog and oldest-message age continued rising or self-drained, check whether the HPA reached its replica ceiling, and inspect the processing path and downstream dependencies for serialized blocking work such as ClickHouse inserts. Also reconcile the 1173 alert value with the lower Grafana aggregates.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Alert payload | Reported 1,173 undelivered messages against a threshold of 500 at 2026-11-16T12:47:31Z. | ✓ |
| Grafana | Ack activity continued and rose during the event. Queried backlog aggregates peaked at 417 and ended at 260, so they do not reconcile with the alert value or establish the backlog trajectory. | ✗ |
| Kubernetes and HPA | Pods were Ready with zero restarts and low observed CPU and memory. HPA scaled from 4 to its 20-replica maximum, then began scaling down. | ✓ |
| Worker code review | Eligible events perform Elasticsearch/OpenSearch bulk updates, with retries for missing documents. No ClickHouse dependency was identified; a processing bottleneck is plausible but unconfirmed. | ✗ |
| GCP logs | The initial query used the subscription as the container name and was invalid for attribution. The correctly scoped worker error query returned no results, but publish rate, nacks, and redelivery evidence remain unavailable. | ✗ |

### Discrepancies
- Alert value 1,173 is not reconciled with queried five-minute backlog means, which peaked at 417.
- No matching publish-rate series or reliable backlog series establishes whether publishing exceeded acknowledgements or whether the backlog self-drained.
- The supplied GCP log results do not establish that worker logs were clean or identify a causal error.

## Deep Dive Findings

{
  "alert": {
    "type": "PubSubUnackedMessages",
    "subscription": "crm-conversations-last-call-events-sub",
    "alert_value": 1173,
    "threshold": 500,
    "started_at": "2026-11-16T12:47:31Z",
    "status_at_triage": "Firing; acknowledged at 12:48:26Z"
  },
  "findings": {
    "consumer": "crm-conversations-last-call-worker",
    "publish_vs_ack_rate": {
      "result": "Unconfirmed. The available Grafana summary reports 18,457 acks in the five-minute bucket ending 12:47Z, but provides no matching publish-rate series or reliable backlog series to compare."
    },
    "backlog_and_age": {
      "result": "Unconfirmed whether the backlog continued rising or self-drained. Queried five-minute backlog means peaked at 417 and ended at 260; neither reconciles with the alert’s 1,173."
    },
    "hpa_and_pods": {
      "result": "HPA scaled from 4 to 20 replicas, then down to 12; maximum is 20. All 12 current pods were Ready and Running with zero restarts and low observed CPU and memory. It reached its configured ceiling during the event."
    },
    "processing_path": {
      "result": "The worker applies one Elasticsearch/OpenSearch bulk update per eligible event. On document-missing responses it retries up to five times, sleeping 300 ms between retries. Other bulk errors cause the message to be returned as unsuccessful for retry. No ClickHouse dependency appears in this worker path, so the July ClickHouse insert-ceiling case is not directly supported as this workload’s cause."
    },
    "causal_assessment": "Active acknowledgements and healthy pods argue against a stopped or crashing consumer. The replica ceiling and per-message search-index writes are consistent with a throughput limit, but available evidence does not establish that processing was slower than publishing or identify a downstream slowdown. Root cause remains undetermined.",
    "metric_reconciliation": "The alert payload reports 1,173 while the queried Grafana five-minute means peaked at 417 and hourly values were much lower. The supplied data does not establish whether this reflects aggregation/alignment differences, a brief peak, or another discrepancy."
  },
  "log_queries": [
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"crm-conversations-last-call-events-sub\" AND severity>=ERROR",
      "result": "No usable histogram or log results; the container label was the subscription name, not the worker name, so this does not establish that worker logs were clean."
    },
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"crm-conversations-last-call-worker\" AND timestamp>=\"2026-11-16T12:30:00Z\" AND timestamp<=\"2026-11-16T13:00:00Z\" AND severity>=ERROR",
      "result": "Attempted against the resolved container name; no result was returned during this investigation."
    }
  ],
  "log_explorer_urls": [
    "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-last-call-events-sub%22%0Aseverity%3E%3DERROR;timeRange=2026-11-16T12%3A42%3A00Z%2F2026-11-16T12%3A52%3A00Z?project=highlevel-backend"
  ],
  "limitations": [
    "Direct Cloud Monitoring metrics and the correctly scoped worker log query were not confirmed here; publish rate, oldest-message age, nacks, and redelivery evidence remain unresolved.",
    "The supplied data does not support a causal RCA."
  ]
}

## Evidence

### Dashboards (incident window)
- [Undelivered messages — crm-conversations-last-call-events-sub (alert at 12:47Z; panel 1)](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc/?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-last-call-events-sub&from=1794831451000&to=1794833251000&viewPanel=1)
- [Ack rate — crm-conversations-last-call-events-sub (alert at 12:47Z; panel 3)](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc/?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-last-call-events-sub&from=1794831451000&to=1794833251000&viewPanel=3)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-last-call-events-sub%22%0Aseverity%3E%3DERROR;timeRange=2026-11-16T12%3A42%3A00Z%2F2026-11-16T12%3A52%3A00Z?project=highlevel-backend)

## Investigation Timeline

**Deep dive passes**: 1
**Worker calls**: 9

**Total worker time**: 784.3s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.3s | ✓ |
| grafana-worker | worker | 102.7s | ✓ |
| kubectl-worker | worker | 301.6s | ✓ |
| log-explorer-worker | worker | 173.1s | ✓ |
| triage | worker | 16.2s | ✓ |
| triage_llm_ranking | decision | 24.5s | ✓ |
| deep_dive_pass_1 | worker | 124.2s | ✓ |
| follow_up_check | decision | 10.8s | ✓ |
| cross_validation | worker | 19.9s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IYYT8GCV71FT7|#139992 Pubsub Unacked Messages above 500>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/f7ce4c3a-4497-4dbf-aa4e-9a751067fe04/view?orgId=1|source>*) AlertValues: Doc for this alert: http://platform.docs/alerts/pubsub-unacked-messages/ Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-last-call-events-sub Unacked Messages: 1173 GCP Link: https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-last-call-events-sub?project=highlevel-backend *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IYYT8GCV71FT7|#139992 Pubsub Unacked Messages above 500>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/f7ce4c3a-4497-4dbf-aa4e-9a751067fe04/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-last-call-events-sub Unacked Messages: 1173 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-last-call-events-sub?project=highlevel-backend>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-16T12:58:53.520Z*