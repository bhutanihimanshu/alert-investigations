# Alert Investigation: PubSubUnackedMessages

**Investigation ID**: inv-2026-11-23-69219afa
**Workload**: crm-conversations-reporting-events-sub
**Container**: crm-conversations-reporting-events-sub
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-23T16:24:23.168Z
**Status**: delivering

## Root Cause

**Summary**: Evidence supports OpenSearch processing degradation as a contributor to reporting-worker backlog growth, followed by substantial deadline expiries and reduced acknowledgment throughput. Disproportionate queueing and latency on one OpenSearch node corroborate the downstream bottleneck. The initiating cause and publisher contribution remain unresolved; Redis warnings alone do not establish Redis as the cause.
**Confidence**: medium
**Category**: downstream_opensearch_degradation

## Triage

**Primary Anomaly**: error_spike
**Details**: WARNING volume rose from 162 to 32,699/min, with Redis slow-query and Elasticsearch latency warnings; no ERROR entries were found. Backlog grew from approximately 161,000 to over 11 million despite active acknowledgments and 300 available pods with zero recorded restarts. This supports downstream processing degradation, but does not establish its cause. Publisher throughput and incident-period nack/expiry rates remain unconfirmed.
**Anomalous**: Yes
**Deep Dive Direction**: YES — triage direction remains correct: downstream OpenSearch degradation, with approximately doubled publisher traffic contributing to initial buildup. Later timeouts, collapsed acknowledgments and deadline expiries confirm processing failure; the initiating OpenSearch cause remains unresolved.

Need affected-node versus peer shard statistics, hot threads, and storage/snapshot diagnostics around 21:30 and 22:20–22:25 IST. Verify runtime image contents and current pod health to confirm deployed retry behavior and worker status.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| Cloud Monitoring | Backlog grew to 36,103,368 messages by 22:46:59 IST, with oldest age 2,621 seconds. Later deadline expiries and reduced acknowledgments support processing failure. | ✓ |
| GCP application logs | Elasticsearch latency warnings of approximately 21 seconds coincided with initial backlog growth. Redis slow-query warnings also increased; neither identifies the initiating cause. | ✓ |
| OpenSearch CloudWatch metrics | At 22:20–22:25 IST, node fb3A8SuER8-GIDvtq5IwFA had approximately 9 times peer indexing latency and 40 times peer write-queue depth, corroborating downstream degradation. | ✓ |
| Kubernetes | All 300 current pods had zero restarts and resource usage below limits; the latest retained rollout predates the incident. This weakens crash, resource-exhaustion and deployment hypotheses, but absent probes limit processing-health conclusions. | ✓ |

### Discrepancies
- The claimed doubling of publisher traffic lacks supporting measurements in the supplied findings.
- Earlier healthy acknowledgments and unavailable expiry samples precede later processing collapse; they do not contradict it.
- Node rejection counters did not increase at onset, so later rejections cannot explain initial buildup.
- Missing retry logs do not exclude retries. Deployed retry behavior, shard-level imbalance, storage and snapshot causes remain unverified.
- Grafana and direct Cloud Monitoring observations share underlying telemetry and are not independent corroboration.

## Deep Dive Findings

{
  "workload": "crm-conversations-reporting-events-sub",
  "status": "partial_root_cause",
  "confidence": "medium",
  "summary": "OpenSearch node-specific processing degradation remains supported. Current workers have not crashed, but backlog continues growing. The initiating storage, snapshot or shard-level cause remains unverified.",
  "findings": [
    {
      "category": "affected_node_vs_peers",
      "source": "Reanalysis of saved incident CloudWatch response /tmp/dd-es.json",
      "node": "fb3A8SuER8-GIDvtq5IwFA",
      "window_IST": "2026-11-23 22:20–22:25",
      "affected_vs_peer_mean": {
        "indexing_latency_ms": [7.79, 0.84],
        "search_latency_ms": [89.97, 20.04],
        "write_queue": [11261.6, 281.76],
        "indexing_rate": [78562.6, 94990.76]
      },
      "additional_evidence": {
        "affected_node_peak_write_queue": 14764,
        "affected_node_rejection_counter_increase_between_available_window_samples": 22691,
        "nodes_with_positive_throughput_throttle_samples": 10,
        "nodes_compared": 12
      },
      "conclusion": "The affected node processed fewer indexing operations with approximately nine times peer indexing latency and forty times peer write-queue depth. Shared throughput throttling does not alone explain its disproportionate degradation."
    },
    {
      "category": "onset_cross_check",
      "window_IST": "2026-11-23 21:30–21:35",
      "affected_vs_peer_mean": {
        "indexing_latency_ms": [1.67, 1.20],
        "write_queue": [931.6, 184.96]
      },
      "conclusion": "Queue growth was already disproportionate at onset, but the affected node's rejection counter did not increase during this window. Later rejections cannot explain the initial buildup.",
      "limitation": "These are node-level metrics, not shard-level service-time measurements."
    },
    {
      "category": "current_worker_health",
      "source": "Live Kubernetes API using the explicit production worker-cluster context",
      "running_pods": 300,
      "ready_pods": 300,
      "container_restarts": 0,
      "image_digest_all_pods": "sha256:857371170de3798c6f35eb284d1c275c60fd4f7b6a5d22a331c943974e45125f",
      "sampled_pod_health_probes": {
        "readiness": false,
        "liveness": false,
        "startup": false
      },
      "conclusion": "Pod crashes do not explain the processing collapse. Kubernetes readiness does not establish batch-processing health."
    },
    {
      "category": "latest_pubsub_state",
      "source": "Live Cloud Monitoring API using existing gcloud authentication",
      "backlog_sample_IST": "2026-11-23 22:46:59",
      "undelivered_messages": 36103368,
      "oldest_unacked_seconds": 2621,
      "acknowledgments_per_minute": {
        "22:43:59": 2328,
        "22:44:59": 190977,
        "22:45:59": 102655
      },
      "expired_deadlines_per_minute": {
        "22:43:59": 308994,
        "22:44:59": 253999
      },
      "conclusion": "Acknowledgments partially resumed after the earlier near-zero phase, but substantial expiries and increasing backlog show no recovery."
    }
  ],
  "runtime_verification": {
    "image_identity": "Confirmed across all current pods.",
    "image_contents": "Unverified.",
    "blockers": [
      "Kubernetes denied pods/exec: missing container.pods.exec permission.",
      "Direct registry manifest request for the deployed digest returned HTTP 404."
    ],
    "retry_behavior": "Previous package/source analysis remains consistent with deadline-driven redelivery, but cannot be promoted to runtime-confirmed behavior."
  },
  "limitations": [
    "Grafana validation returned HTTP 401 session.token.rotate; no native authenticated browser tool was available.",
    "Incident-period shard statistics, hot threads, snapshot state and storage diagnostics were not obtained.",
    "OpenSearch endpoint uses a Kubernetes valueFrom reference; direct connectivity was not tested and no secret was retrieved.",
    "Required parallel-agent launch failed with a thread-store error; independent API checks ran concurrently.",
    "Historical incident notes supplied diagnostic pointers only; they do not establish tonight's initiating cause."
  ],
  "logQueriesUsed": [],
  "logExplorerUrls": [],
  "log_evidence_note": "No new log queries were executed in this pass; earlier timeout evidence remains supplied context.",
  "next_check": "Obtain retained shard, hot-thread, snapshot and storage diagnostics for 21:30 and 22:20–22:25 IST through authenticated OpenSearch access or AWS/platform support. Verify deployed files through authorized image access.",
  "actions": "Read-only checks only. No production changes, messages or additional orchestrator investigation."
}

## Evidence

### Dashboards (incident window)
- [Unacked messages — crm-conversations-reporting-events-sub rises to 11.16 million](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1795449242895&to=1795452842895&viewPanel=1)
- [Ack throughput — crm-conversations-reporting-events-sub continues processing](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1795449242895&to=1795452842895&viewPanel=3)
- [Oldest unacked age — crm-conversations-reporting-events-sub reaches 20 minutes](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1795449242895&to=1795452842895&viewPanel=10)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%20resource.labels.container_name%3D%22crm-conversations-reporting-worker%22%20severity%3DWARNING%20jsonPayload.message%3D%22Elasticsearch%20query%20latency%20is%20too%20high%20for%20Elasticsearch%20Data%20Ware%20House%22;timeRange=2026-11-23T16%3A06%3A00Z%2F2026-11-23T16%3A16%3A00Z?project=highlevel-backend)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 12

**Total worker time**: 1872.5s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 16.7s | ✓ |
| grafana-worker | worker | 112.2s | ✓ |
| kubectl-worker | worker | 123.6s | ✓ |
| log-explorer-worker | worker | 400.5s | ✓ |
| triage | worker | 16.6s | ✓ |
| triage_llm_ranking | decision | 33.6s | ✓ |
| deep_dive_pass_1 | worker | 515.9s | ✓ |
| follow_up_check | decision | 13.9s | ✓ |
| deep_dive_pass_2 | worker | 434.4s | ✓ |
| follow_up_check | decision | 14.7s | ✓ |
| deep_dive_pass_3 | worker | 170.2s | ✓ |
| cross_validation | worker | 20.1s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IBI4MWE5JZLF7|#140754 Pubsub Unacked Messages above 5million>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/e86b2cb5-9fa9-4384-b29b-2eaf16b57717/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-reporting-events-sub Unacked Messages: 10058142 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-reporting-events-sub?project=highlevel-backend> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IBI4MWE5JZLF7|#140754 Pubsub Unacked Messages above 5million>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/e86b2cb5-9fa9-4384-b29b-2eaf16b57717/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-reporting-events-sub Unacked Messages: 10058142 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-reporting-events-sub?project=highlevel-backend>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-23T17:18:19.248Z*