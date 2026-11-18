# Alert Investigation: PubSubUnackedMessages

**Investigation ID**: inv-2026-11-18-195daa5d
**Workload**: crm-conversations-reporting-events-sub
**Container**: crm-conversations-reporting-events-sub
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-18T03:29:42.906Z
**Status**: delivering

## Root Cause

**Summary**: OpenSearch write-pool saturation is a supported ongoing contributor: rejected writes prevent successful processing and plausibly drive redelivery, reducing effective throughput while backlog drains. Independent queue/latency metrics and application rejection logs support this bottleneck. The initiating trigger and underlying OpenSearch slowdown remain unresolved; historical node mapping and shard-level telemetry are needed before selecting a specific remediation.
**Confidence**: medium
**Category**: opensearch_write_saturation_underlying_trigger_unresolved

## Triage

**Primary Anomaly**: error_spike
**Details**: Severe backlog peaked at 123.4M and remains approximately 94–96M while draining; oldest-message age reached 6.57 hours. OpenSearch write-pool rejections are confirmed ongoing errors, although an onset error spike was not established. Earlier sent/ack ratios above 2 and deadline expirations indicate retry amplification. Current consumption is active with 300 ready pods, zero observed recent restarts, and HPA at its cap. Historical candidates offer partial downstream-capacity analogies, but none establishes the same RCA: SLA reporting uses ClickHouse, mapping failures have a different error signature, and DL cases concern different subscription types.
**Anomalous**: Yes
**Deep Dive Direction**: YES — Triage direction remains correct: OpenSearch write saturation and retry amplification are strongly supported, but the underlying node slowdown remains unresolved.

Need affected-node shard placement, per-shard indexing, merge/throttle and indexing-pressure statistics for 00:00–00:20 IST, plus executor-to-node mapping. Existing metrics weaken node-wide CPU saturation and disk-latency explanations.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana PubSub metrics | Backlog peaked at 123.4M and fell to 95.98M by 09:01 IST. Earlier sent/ack ratios above 2 and deadline expirations support redelivery amplification; metrics do not identify its initiating cause. | ✓ |
| GCP application logs and local handler | OpenSearch rejected writes with eight active threads and queues exceeding 10,000 tasks. Local code leaves rejected items unacknowledged, supporting a retry mechanism; deployed-source parity remains unverified. | ✓ |
| Saved OpenSearch node metrics | Indexing latency and write queues rose together during 00:08–00:18 IST. This independently supports write-path congestion, but the logged executor is not mapped to this node. | ✓ |
| Kubernetes and worker metrics | 300 ready pods, low CPU and zero recent observed restarts weaken a current worker-crash or CPU-capacity explanation. Readiness lacks probes and cannot establish processing health. | ✓ |

### Discrepancies
- The claimed nack rates are unreliable: nack_requests is a GAUGE aggregated with ALIGN_SUM. Do not use those values as measured nack throughput.
- No onset error spike or publisher-ingress measurement establishes what initiated backlog growth.
- Shard contention, merge throttling and indexing pressure remain unproven; existing metrics weaken node-wide CPU and disk-latency explanations.
- Absent deliveryAttempt matches do not exclude retries. Sampled pod concentration does not establish tenant abuse or a poison message.
- Deep-dive evidence was reused, not freshly measured. Cross-service correlation was not assessed; false does not establish absence of an infrastructure event.

## Deep Dive Findings

{
  "workload": "crm-conversations-reporting-events-sub",
  "verdict": "OpenSearch write saturation remains supported. Shard contention, merge throttling and indexing pressure remain unproven; executor-to-node mapping is still unavailable.",
  "confidence": {
    "sustaining_bottleneck": "high",
    "underlying_cause": "unresolved"
  },
  "findings": [
    {
      "source": "/private/tmp/target-es-nodes.json",
      "freshness": "Previously captured response, reanalyzed this pass; not a fresh measurement.",
      "node": "fb3A8SuER8-GIDvtq5IwFA",
      "window_ist": "2026-11-18 00:00–00:20",
      "observations": [
        "At 00:08, IndexingLatency was 6.48 and ThreadpoolWriteQueue was 4,879.",
        "At 00:16, IndexingLatency reached 10.46 and ThreadpoolWriteQueue reached 9,834.",
        "At 00:17, latency fell to 2.38 and queue depth to 1,572; both rose again at 00:18."
      ],
      "interpretation": "Intermittent indexing slowdown coincides with queue accumulation. This supports an indexing-path bottleneck but does not identify its cause."
    },
    {
      "source": "/private/tmp/reporting-onset-rejections.json",
      "executor": "817397e44e2bfe92700e08758331ccf9/write",
      "observations": [
        "Four saved rejection entries at 00:17:04 IST identify the same executor.",
        "They show eight active threads and 10,650–10,660 queued tasks against capacity 10,000.",
        "Affected indices include messages-2026-04 and email-messages-2026-04."
      ],
      "limitation": "Executor identity has not been mapped to the anomalous CloudWatch node. Matching timing alone does not establish that mapping."
    },
    {
      "source": "/private/tmp/target-reporting-esdash.json",
      "observation": "The saved dashboard's Merge Thread Pool panel queries ThreadpoolForce_merge metrics.",
      "limitation": "The dashboard definition supplies no historical shard allocation, per-shard indexing, ordinary merge activity or indexing-pressure evidence."
    }
  ],
  "access_checks": [
    {
      "path": "Grafana API using stored session",
      "request": "GET /api/org",
      "result": "401 session.token.rotate"
    },
    {
      "path": "Existing Kubernetes credentials; explicit production worker context",
      "request": "Read the secret referenced by ELASTICSEARCH_CRM_CONVERSATION_REPORTINGS_URL",
      "result": "Forbidden: container.secrets.get permission required.",
      "consequence": "Direct OpenSearch node-map and shard-stat requests could not be authenticated. No secret value was returned."
    }
  ],
  "logQueriesUsed": [],
  "logExplorerUrls": [],
  "limitations": [
    "No fresh log query or backlog measurement was performed.",
    "Prior pod-health and retry findings were reused from supplied context.",
    "Historical shard placement, per-shard indexing, merge/throttle and indexing-pressure statistics remain unavailable."
  ],
  "next_evidence_needed": {
    "window_ist": "2026-11-18 00:00–00:20",
    "required": [
      "Historical node ID-to-name mapping for executor 817397e44e2bfe92700e08758331ccf9.",
      "Shard allocation and per-shard indexing statistics for both affected indices.",
      "Merge activity, indexing throttle time and indexing-pressure rejection statistics."
    ],
    "qualification": "A current OpenSearch snapshot would help establish current mapping, but cannot reconstruct onset conditions without retained historical telemetry."
  },
  "actions": "Read-only. No messages posted, production changes made or additional investigation launched."
}

## Evidence

### Dashboards (incident window)
- [Unacked messages — reporting subscription: 123.4M peak, now draining](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794940200000&to=1794972677632&viewPanel=1)
- [Oldest message age — reporting subscription: rising above six hours](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794940200000&to=1794972677632&viewPanel=10)
- [Acknowledgments — reporting subscription: consumption remains active](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794940200000&to=1794972677632&viewPanel=3)
- [Expired ack deadlines — reporting subscription: earlier timeout spikes](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794940200000&to=1794972677632&viewPanel=7)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-reporting-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22rejected%20execution%22;timeRange=2026-11-18T03%3A27%3A00Z%2F2026-11-18T03%3A34%3A00Z?project=highlevel-backend)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 12

**Total worker time**: 1271.6s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 15.9s | ✓ |
| grafana-worker | worker | 100.5s | ✓ |
| kubectl-worker | worker | 141.5s | ✓ |
| log-explorer-worker | worker | 296.2s | ✓ |
| triage | worker | 13.8s | ✓ |
| triage_llm_ranking | decision | 22.3s | ✓ |
| deep_dive_pass_1 | worker | 394.7s | ✓ |
| follow_up_check | decision | 8.4s | ✓ |
| deep_dive_pass_2 | worker | 169.7s | ✓ |
| follow_up_check | decision | 7.6s | ✓ |
| deep_dive_pass_3 | worker | 85.9s | ✓ |
| cross_validation | worker | 15.2s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IJRFW25C1ZZ7W|#140159 Pubsub Unacked Messages above 5million>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/e86b2cb5-9fa9-4384-b29b-2eaf16b57717/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-reporting-events-sub Unacked Messages: 97035623 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-reporting-events-sub?project=highlevel-backend> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IJRFW25C1ZZ7W|#140159 Pubsub Unacked Messages above 5million>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/e86b2cb5-9fa9-4384-b29b-2eaf16b57717/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-reporting-events-sub Unacked Messages: 97035623 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-reporting-events-sub?project=highlevel-backend>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-18T03:47:37.292Z*