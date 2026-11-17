# Alert Investigation: PubSubOldestUnackedAge

**Investigation ID**: inv-2026-11-17-4a7db10d
**Workload**: crm-conversations-reporting-events-sub
**Container**: crm-conversations-reporting-events-sub
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-17T19:27:59.438Z
**Status**: delivering

## Root Cause

**Summary**: OpenSearch node-local write saturation is a strongly corroborated processing bottleneck: write rejections coincided with insufficient acknowledgement throughput and growing backlog. Redelivery added overhead, but its precise attribution to rejected writes versus unidentified timeouts is unresolved. The trigger for saturation is not established.
**Confidence**: medium
**Category**: opensearch_node_local_write_saturation

## Triage

**Primary Anomaly**: error_spike
**Details**: OpenSearch write saturation is confirmed: 197/200 sampled errors were HTTP 429 rejected_execution_exception, including 11,054 queued tasks against capacity 10,000. Backlog reached approximately 25.3 million and oldest age 3,083s. Acknowledgements fell to 2,237/s against 9,730/s published; sent/ack reached 2.335 and expired deadlines peaked at 644,780 per five minutes. All 300 pods were ready with zero recorded restarts. Candidate inv-2026-08-13-ffc1f0ff partially matches downstream write rejection and capped consumers, but its retry overhead was much lower; no high-confidence historical RCA match exists.
**Anomalous**: Yes
**Deep Dive Direction**: YES — the triage direction remains correct: node-local OpenSearch write saturation is confirmed, but its initiating cause remains unconfirmed.

Need onset-time node-to-shard placement, shard indexing/merge statistics, and internal-task evidence; deployed base-worker artifact verification; and timeout stack traces correlated with deadline expirations to quantify retry amplification.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| GCP application logs | 197/200 sampled errors were OpenSearch HTTP 429 write rejections; one rejection reported 11,054 queued tasks against capacity 10,000. | ✓ |
| Saved CloudWatch node metrics | Only one of 12 nodes accumulated additional write rejections, reaching 124,305, with elevated indexing latency. This independently supports node-local write saturation. | ✓ |
| Pub/Sub metrics via Grafana | Backlog reached 25.34 million; acknowledgements were 2,237/s versus 9,730/s published. Sent/ack ratio 2.335 and anomalous deadline expirations support redelivery overhead. | ✓ |
| Kubernetes and pod metrics | 300 pods were ready with zero recorded restarts; the latest rollout preceded the incident by days. This weakens crash-loop and recent-deployment explanations. | ✓ |

### Discrepancies
- The initiating cause of node-local saturation remains unknown: historical shard placement, merges and internal tasks were not obtained.
- Comparable node indexing rates weaken simple unequal-volume attribution; publishing was within historical p95.
- Nack volume was below its historical median, so high absolute nacks alone do not establish an anomalous retry loop.
- Deadline expirations followed timeout samples, but message linkage and timeout stacks are missing; their attributable fraction is unproven.
- Local failed-item acknowledgement handling was not verified against the deployed artifact. CloudWatch evidence was reused, and fresh deep-dive queries remained pending.

## Deep Dive Findings

{
  "workload": "crm-conversations-reporting-events-sub",
  "rootCauseCategory": "opensearch_node_local_write_saturation",
  "confidence": {
    "processingBottleneck": "high",
    "retryAmplification": "high",
    "initiatingCause": "unconfirmed"
  },
  "summary": "Saved metrics strengthen node-local saturation attribution but do not establish its initiating cause. Historical shard placement, merge/task evidence, deployed package verification and timeout-stack attribution remain unresolved.",
  "newFindings": [
    {
      "finding": "Only one of 12 nodes accumulated additional write rejections.",
      "nodeId": "fb3A8SuER8-GIDvtq5IwFA",
      "windowIST": "2026-11-17 23:30–2026-11-18 01:10",
      "details": "Its rejection counter rose from 0 to 124305; the other 11 counters remained unchanged. The first observed nonzero value was 608 at 00:16 IST.",
      "source": "/private/tmp/target-es-nodes.json",
      "evidenceBoundary": "Reanalysis of a prior-pass CloudWatch response, not a fresh query."
    },
    {
      "finding": "Node indexing volume does not show a large imbalance.",
      "details": "The affected node's recorded IndexingRate values were comparable to peers. Its IndexingLatency peaked at 10.906 at 00:37 IST, versus peer maxima of 1.617–5.622 across the window.",
      "interpretation": "Weakens a simple unequal-volume explanation; unequal document cost, shard contention, merges and internal tasks remain possible.",
      "source": "/private/tmp/target-es-nodes.json"
    },
    {
      "finding": "Deadline-expiration bursts followed the previously observed timeout sample.",
      "details": "Saved one-minute expiration buckets contained 362424 expirations ending 00:46 IST and 173323 ending 00:47 IST. Previously supplied timeout entries occurred at 00:38:04–00:38:06 IST.",
      "interpretation": "Temporal ordering is compatible with lease expiration after processing failures, but lacks batch/message linkage and cannot quantify the timeout-attributable fraction.",
      "source": "/private/tmp/deep-expired_ack_deadlines_count.json"
    }
  ],
  "verificationGaps": {
    "historicalShardPlacementAndTasks": "No onset-time shard, merge or task evidence obtained. Native authenticated browser access was unavailable; prior direct OpenSearch access returned 401.",
    "deployedArtifact": "Registry build-metadata lookup was started using native gcloud authentication but remained pending without output. Installed package bytes and lockfile integrity were not matched.",
    "timeoutStacks": "Fresh projected log query remained pending without output; the timed-out operation is still unidentified.",
    "parallelExecution": "Subagent launch failed with a thread-store error."
  },
  "logQueriesUsed": [
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"crm-conversations-reporting-worker\" AND severity>=ERROR AND timestamp>=\"2026-11-17T19:05:00Z\" AND timestamp<=\"2026-11-17T19:15:00Z\" AND jsonPayload.message=~\"Platform_Services_P0_Batch_Processing_Worker_failure\"",
      "limit": 8,
      "projectedFields": "timestamp, message, error, stack, stacktrace, metadata.error, metadata.stack, pod_name",
      "result": "Pending without output; not an empty-result finding."
    }
  ],
  "logExplorerUrls": [
    {
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-reporting-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22Platform_Services_P0_Batch_Processing_Worker_failure%22;timeRange=2026-11-17T19%3A05%3A00Z%2F2026-11-17T19%3A15%3A00Z?project=highlevel-backend",
      "verification": "Returned timeout entries in the supplied previous pass; not reverified this pass."
    }
  ],
  "actions": "Read-only investigation. No messages posted, production changes made or additional orchestrator investigation launched."
}

## Evidence

### Dashboards (incident window)
- [Oldest age — crm-conversations-reporting-events-sub (rising above 30 minutes)](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794940200000&to=1794944729180&viewPanel=10)
- [Backlog — crm-conversations-reporting-events-sub (25.34 million messages)](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794940200000&to=1794944729180&viewPanel=1)
- [Acknowledgements — crm-conversations-reporting-events-sub (throughput declining)](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794940200000&to=1794944729180&viewPanel=3)
- [Expired deadlines — crm-conversations-reporting-events-sub (large retry spike)](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794940200000&to=1794944729180&viewPanel=7)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-reporting-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22rejected%20execution%22;timeRange=2026-11-17T19%3A05%3A00Z%2F2026-11-17T19%3A15%3A00Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-reporting-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22Platform_Services_P0_Batch_Processing_Worker_failure%22;timeRange=2026-11-17T19%3A05%3A00Z%2F2026-11-17T19%3A15%3A00Z?project=highlevel-backend)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 12

**Total worker time**: 1270.3s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 15.5s | ✓ |
| grafana-worker | worker | 98.1s | ✓ |
| kubectl-worker | worker | 171.3s | ✓ |
| log-explorer-worker | worker | 336.5s | ✓ |
| triage | worker | 12.2s | ✓ |
| triage_llm_ranking | decision | 21.2s | ✓ |
| deep_dive_pass_1 | worker | 265.8s | ✓ |
| follow_up_check | decision | 9.7s | ✓ |
| deep_dive_pass_2 | worker | 193.4s | ✓ |
| follow_up_check | decision | 7.8s | ✓ |
| deep_dive_pass_3 | worker | 122.4s | ✓ |
| cross_validation | worker | 16.3s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IMT2N52XLHMJZ|#140139 Pubsub Oldest Unacked Messages age above 30mins>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/demqv0uteacjkf/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-age/> Hey, looks like the age of the Oldest Unacked message in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-reporting-events-sub Age of the Oldest Unacked Messages(seconds): 2056 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-reporting-events-sub?project=highlevel-backend> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IMT2N52XLHMJZ|#140139 Pubsub Oldest Unacked Messages age above 30mins>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/demqv0uteacjkf/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-age/> Hey, looks like the age of the Oldest Unacked message in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-reporting-events-sub Age of the Oldest Unacked Messages(seconds): 2056 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-reporting-events-sub?project=highlevel-backend>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-17T20:01:21.124Z*