# Alert Investigation: PubSubUnackedMessages

**Investigation ID**: inv-2026-11-19-be311a9e
**Workload**: crm-conversations-reporting-events-sub
**Container**: crm-conversations-reporting-events-sub
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-19T19:10:27.534Z
**Status**: delivering

## Root Cause

**Summary**: OpenSearch write-pool saturation is the corroborated processing bottleneck: bulk writes were rejected, acknowledgements fell and the reporting subscription backlog grew, with redeliveries likely amplifying load. The initiating cause of saturation remains unresolved. Low consumer resource usage and zero recorded restarts weaken a consumer-capacity explanation; sparse storage and GC observations do not exclude brief dependency stalls. Obtain node/shard diagnostics for November 20, 00:14–00:19 IST before selecting mitigation.
**Confidence**: medium
**Category**: opensearch_write_pool_saturation

## Triage

**Primary Anomaly**: error_spike
**Details**: OpenSearch 429 rejected_execution_exception errors coincide with acknowledgement collapse and backlog growth to 18.15M. Node 817397e44e2bfe92700e08758331ccf9 had all 8 write threads active and queued tasks above its 10,000 capacity. Consumers had 300 ready pods, zero restarts and low CPU. Elevated expired deadlines and reported nack activity support retry amplification; missing deliveryAttempt fields prevent log-level confirmation. This matches the write-pool bottleneck in inv-2026-11-18-195daa5d, but neither investigation establishes the saturation trigger.
**Anomalous**: Yes
**Deep Dive Direction**: YES — Triage direction remains correct: node-local OpenSearch write saturation is the established bottleneck; its initiating trigger remains unresolved. Deployed acknowledgement handling is verified.

Need AWS/OpenSearch diagnostics for the affected node on November 20, 00:14–00:19 IST: shard-level indexing/merge activity, blocked write-thread stacks, storage stalls, and shard relocation events.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| GCP application logs | OpenSearch returned 429 write rejections with all 8 write threads active and queued tasks exceeding 10,000 on node 817397e44e2bfe92700e08758331ccf9. | ✓ |
| Cloud Monitoring | Acknowledgements collapsed while backlog reached 18.15M. Elevated expired deadlines and delivery/ack ratio support redelivery amplification. | ✓ |
| Retained CloudWatch metrics | Write queue grew from zero to 11,027 during 00:14–00:19 IST; rejection counter increased by 7,221. Falling indexing throughput supports downstream write saturation. | ✓ |
| Kubernetes and VictoriaMetrics | 300 ready consumers, zero recorded application restarts and low resource usage argue against consumer resource exhaustion. Latest deployment preceded the incident by six days. | ✓ |

### Discrepancies
- Nack evidence conflicts: one query reported 186,532/min while another returned no series. Missing data cannot establish zero nacks.
- Triage asserts deployed acknowledgement handling was verified, but supplied detailed evidence only documents local source review.
- CloudWatch results were retained rather than freshly retrieved; node mapping was not independently refreshed.
- Missing deliveryAttempt fields prevent log-level retry confirmation. The short log sample cannot establish pod or tenant concentration.
- Saturation onset and later backlog growth are consistent, but shard activity, thread stacks, storage stalls and relocations remain unverified. Cross-container impact was not assessed.

## Deep Dive Findings

{
  "workload": "crm-conversations-reporting-events-sub",
  "verdict": "OpenSearch write saturation remains established; the initiating trigger remains unresolved.",
  "window_ist": "2026-11-20 00:14–00:19",
  "new_findings": {
    "source": "Reanalysis of retained CloudWatch responses in /tmp/nov19-aws-nodes.json and /tmp/nov19-aws-extra.json; not a fresh API retrieval.",
    "node_mapping": "CloudWatch node fb3A8SuER8-GIDvtq5IwFA, using the previously recorded mapping to 817397e44e2bfe92700e08758331ccf9; mapping not independently refreshed.",
    "onset_sequence": [
      "Write queue was zero at 00:14–00:15 IST, then rose to 3,225 at 00:16, 8,847 at 00:17 and 11,027 at 00:19.",
      "Reported IndexingLatency rose from 1.124 at 00:14 to 7.492 at 00:17, while IndexingRate fell from 77,663 to 41,238.",
      "The cumulative write-rejection counter increased by 7,221 between 00:15 and 00:19."
    ],
    "alternative_causes_checked": [
      "CPU remained approximately 20.8–29.9%; JVM memory pressure remained 53.3–72.1%. Old-generation GC count and time were zero in returned onset samples.",
      "IopsThrottle and ThroughputThrottle were zero at the 00:18 sample. ReadLatency and WriteLatency were similar to their 00:13 samples; these sparse measurements cannot exclude short storage stalls.",
      "Force-merge queue and thread metrics were zero. This does not exclude ordinary background merges.",
      "DiskQueueDepth, IndexingThrottleTime and ShardIndexingPressureMemoryLimitBreached had no target-node series; absence is not evidence of zero activity."
    ]
  },
  "requested_diagnostics": {
    "shard_level_indexing_and_merges": "Not obtained.",
    "blocked_write_thread_stacks": "Not obtained.",
    "storage_stalls": "No supporting signal in retained sparse metrics; host-level stalls remain unexcluded.",
    "shard_relocations": "No historical relocation evidence obtained."
  },
  "access_blocker": {
    "path": "Existing Grafana session via the documented CLI",
    "validation": "GET /api/org returned HTTP 401, messageId=session.token.rotate.",
    "impact": "Fresh CloudWatch queries could not proceed. No AWS CLI was available on PATH."
  },
  "logQueriesUsed": [
    {
      "source": "Retained CloudWatch Logs Insights result",
      "query": "fields @timestamp, @logStream, @message | sort @timestamp asc | limit 100",
      "window_utc": "2026-11-19T18:44:00Z/2026-11-19T18:49:00Z",
      "artifact": "/tmp/pass3-vendorresults.json",
      "result": "Complete; zero records scanned and matched.",
      "rerun_this_pass": false
    }
  ],
  "logExplorerUrls": [],
  "next_evidence_needed": "AWS/OpenSearch support capture for this node and window: per-shard indexing/merge timelines, write-thread stacks, host storage latency and relocation events. Aggregate metrics cannot distinguish these triggers.",
  "actions": "Read-only. No messages posted, production changes made or additional orchestrator investigation launched."
}

## Evidence

### Dashboards (incident window)
- [Backlog — reporting subscription: rapid growth to 17.24 million](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1795112070000&to=1795115670000&viewPanel=1)
- [Acknowledgements — reporting subscription: throughput collapse](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1795112070000&to=1795115670000&viewPanel=3)
- [Expired deadlines — reporting subscription: elevated message timeouts](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1795112070000&to=1795115670000&viewPanel=7)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-reporting-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22rejected%20execution%22;timeRange=2026-11-19T19%3A04%3A00Z%2F2026-11-19T19%3A14%3A00Z?project=highlevel-backend)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 12

**Total worker time**: 955.3s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.9s | ✓ |
| grafana-worker | worker | 90.2s | ✓ |
| kubectl-worker | worker | 113.8s | ✓ |
| log-explorer-worker | worker | 117.4s | ✓ |
| triage | worker | 15.6s | ✓ |
| triage_llm_ranking | decision | 26.6s | ✓ |
| deep_dive_pass_1 | worker | 192.5s | ✓ |
| follow_up_check | decision | 11.7s | ✓ |
| deep_dive_pass_2 | worker | 286.8s | ✓ |
| follow_up_check | decision | 14.8s | ✓ |
| deep_dive_pass_3 | worker | 54.5s | ✓ |
| cross_validation | worker | 19.3s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I7YC825NWS98C|#140345 Pubsub Unacked Messages above 5million>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/e86b2cb5-9fa9-4384-b29b-2eaf16b57717/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-reporting-events-sub Unacked Messages: 14554035 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-reporting-events-sub?project=highlevel-backend> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I7YC825NWS98C|#140345 Pubsub Unacked Messages above 5million>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/e86b2cb5-9fa9-4384-b29b-2eaf16b57717/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-reporting-events-sub Unacked Messages: 14554035 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-reporting-events-sub?project=highlevel-backend>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-19T19:26:15.894Z*