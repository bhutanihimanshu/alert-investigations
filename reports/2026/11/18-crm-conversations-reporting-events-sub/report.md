# Alert Investigation: PubSubOldestUnackedAge

**Investigation ID**: inv-2026-11-18-02bcb4fd
**Workload**: crm-conversations-reporting-events-sub
**Container**: crm-conversations-reporting-events-sub
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-18T02:08:48.660Z
**Status**: delivering

## Root Cause

**Summary**: OpenSearch write saturation caused reporting-worker bulk-item failures that were nacked, contributing to retry amplification and a severe Pub/Sub backlog despite active consumers. The trigger for disproportionate node degradation remains unresolved; publisher overload, insufficient replicas and storage throttling are not proven root causes.
**Confidence**: medium
**Category**: opensearch_write_saturation

## Triage

**Primary Anomaly**: error_spike
**Details**: Confirmed backlog of 116.17M messages with oldest age 5h54m44s. Sampled OpenSearch HTTP 429 rejected_execution_exception errors show 11,020 queued tasks against 10,000 capacity; handler analysis indicates failed items are nacked. Earlier sent/ack ratio reached 1.962 and deadline expirations peaked at 7.72M/hour. Consumers remain active with 300 ready pods and no recorded restarts. Historical candidate inv-2026-08-13-ffc1f0ff partially matches downstream write rejection plus capped HPA, but its publisher-overload RCA is not established here.
**Anomalous**: Yes
**Deep Dive Direction**: YES — Triage direction remains correct: OpenSearch write saturation is confirmed, with severe degradation concentrated on one node. The underlying trigger remains unresolved.

Need executor-to-node mapping, historical shard allocation and per-shard indexing rates, merge/storage activity, and AWS domain configuration/CloudTrail change history around 00:00–00:20 IST on November 18. Publisher overload and insufficient consumer replicas remain unproven.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| GCP application logs and handler analysis | Verified samples show OpenSearch HTTP 429 write rejections with 11,020 queued tasks against 10,000 capacity. The handler excludes failed items from acknowledgments; the base worker nacks them. | ✓ |
| Pub/Sub metrics | Backlog reached 123.67M messages; 116.17M remained at 07:43 IST with oldest age 5h54m44s. Earlier sent/ack ratio reached 1.962 and deadline expirations peaked at 7.72M/hour, corroborating impaired processing and redelivery. | ✓ |
| Kubernetes and Grafana pod health | 300 ready consumers, zero recorded restarts and low current CPU argue against a current consumer crash or CPU-capacity bottleneck. The latest rollout predates the incident. | ✓ |
| Saved OpenSearch node telemetry | One node's write queue rose from 8 to 4,879 around midnight IST while peers remained near empty, independently supporting concentrated downstream write pressure. | ✓ |

### Discrepancies
- OpenSearch rejections and the nack path establish a processing bottleneck, but its contribution to the entire backlog is not quantified; publisher ingress was not measured.
- The rejected executor has not been mapped to the degraded node. The verified log samples are later than initial queue growth, limiting onset attribution.
- Falsification check: throughput throttling also occurred outside the onset window and queue growth preceded the cited throttle bucket; storage throttling is not established as the initiating cause.
- Nacks were below their historical median. Empty deliveryAttempt queries do not exclude retries because this batch handler may omit that field.
- Grafana authentication subsequently failed; deep-dive telemetry was saved rather than refreshed. Shard allocation, merge activity, infrastructure changes and cross-service correlation remain unverified.

## Deep Dive Findings

{
  "workload": "crm-conversations-reporting-events-sub",
  "status": "partial_access_blocked",
  "conclusion": "OpenSearch write saturation remains confirmed. Saved telemetry adds evidence of throughput throttling, but does not establish the trigger for disproportionate degradation on one node.",
  "confidence": {
    "processing_bottleneck": "high",
    "underlying_trigger": "unresolved"
  },
  "additional_findings": [
    {
      "source": "/tmp/nov18-capacity.json",
      "node": "fb3A8SuER8-GIDvtq5IwFA",
      "finding": "Write queue increased from 8 at 23:55 IST November 17 to 2,211 at 00:00 and 4,879 at 00:05 November 18. Other nodes ranged 0–67 and 0–7 at those latter timestamps.",
      "interpretation": "Queue growth was initially concentrated on this node, before broader write pressure developed."
    },
    {
      "source": "/tmp/nov18-io.json",
      "finding": "Affected-node ThroughputThrottle was 1 in the five-minute bucket timestamped 00:10 IST; IopsThrottle remained zero.",
      "counterevidence": "Throughput throttling also occurred before midnight and on other nodes. Queue growth preceded the 00:10 bucket.",
      "interpretation": "Storage throughput constraints are a plausible contributor, not a proven initiating cause."
    },
    {
      "source": "/tmp/nov18-capacity.json",
      "finding": "At 00:00 IST, affected-node IndexingRate was 135,869 versus 107,110–145,961 on peers. SearchRate was 11,035 versus 6,680–11,419.",
      "limitation": "These are saved five-minute Maximum metric values, not per-shard rates or publisher ingress. They do not demonstrate exceptional traffic volume on the affected node."
    },
    {
      "source": "/tmp/nov18-io.json",
      "finding": "BurstBalance was zero throughout all 195 returned affected-node samples; EBSByteBalance% and EBSIOBalance% supplied no points in the target window.",
      "interpretation": "No incident-specific credit depletion transition is demonstrated. Storage configuration is needed to interpret these signals."
    }
  ],
  "remaining_checks": {
    "executor_to_node_mapping": "Unresolved; executor 817397e44e2bfe92700e08758331ccf9 cannot yet be linked conclusively to the CloudWatch node.",
    "historical_shard_allocation_and_rates": "Not obtained.",
    "merge_activity": "Not obtained; throughput throttling alone does not prove merge saturation.",
    "aws_configuration_and_cloudtrail": "Not obtained."
  },
  "access_checks": [
    "Fresh GET /api/org using the existing Grafana cookie session returned HTTP 401 session.token.rotate.",
    "AWS CLI was unavailable; ~/.aws/config and ~/.aws/credentials were absent.",
    "No native browser or OpenSearch connector was exposed in this executor."
  ],
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"crm-conversations-reporting-worker\" AND timestamp>=\"2026-11-17T18:30:00Z\" AND timestamp<=\"2026-11-17T18:50:00Z\" AND severity>=ERROR AND jsonPayload.index.error.type=\"rejected_execution_exception\"",
      "limit": 5,
      "result": "Interrupted after returning no output while still running; this is not evidence of zero matching logs."
    }
  ],
  "log_explorer_urls": [
    {
      "verification": "Inherited previously verified URL; not freshly verified in this pass.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-reporting-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.index.error.type%3D%22rejected_execution_exception%22;timeRange=2026-11-17T18%3A45%3A00Z%2F2026-11-17T18%3A55%3A00Z?project=highlevel-backend"
    }
  ],
  "needed_evidence": "Authorized node ID/name mapping, historical shard and merge telemetry, EBS throughput configuration, and CloudTrail/domain change history around November 18 00:00–00:20 IST.",
  "limitations": "Additional metric findings were reanalyzed from saved responses, not refreshed live. Publisher overload, insufficient replicas, shard imbalance, and merge saturation remain unproven. No production changes, messages, or additional investigations were initiated."
}

## Evidence

### Dashboards (incident window)
- [Oldest age — reporting-events-sub (rising toward six hours)](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794938400000&to=1794968040000&viewPanel=10)
- [Unacked messages — reporting-events-sub (116M remaining after peak)](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794938400000&to=1794968040000&viewPanel=1)
- [Ack throughput — reporting-events-sub (consumers still processing)](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794938400000&to=1794968040000&viewPanel=3)
- [Expired ack deadlines — reporting-events-sub (millions during backlog)](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794938400000&to=1794968040000&viewPanel=7)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-reporting-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.index.error.type%3D%22rejected_execution_exception%22;timeRange=2026-11-17T23%3A51%3A00Z%2F2026-11-18T00%3A01%3A00Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-reporting-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.index.error.type%3D%22rejected_execution_exception%22;timeRange=2026-11-17T18%3A45%3A00Z%2F2026-11-17T18%3A55%3A00Z?project=highlevel-backend)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 12

**Total worker time**: 1093.1s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 16.7s | ✓ |
| grafana-worker | worker | 97.3s | ✓ |
| kubectl-worker | worker | 99.7s | ✓ |
| log-explorer-worker | worker | 289.3s | ✓ |
| triage | worker | 13.1s | ✓ |
| triage_llm_ranking | decision | 20.3s | ✓ |
| deep_dive_pass_1 | worker | 315.2s | ✓ |
| follow_up_check | decision | 9.0s | ✓ |
| deep_dive_pass_2 | worker | 92.7s | ✓ |
| follow_up_check | decision | 7.4s | ✓ |
| deep_dive_pass_3 | worker | 115.8s | ✓ |
| cross_validation | worker | 16.5s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IZTYRCX67T33C|#140154 Pubsub Oldest Unacked Messages age above 30mins>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/demqv0uteacjkf/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-age/> Hey, looks like the age of the Oldest Unacked message in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-reporting-events-sub Age of the Oldest Unacked Messages(seconds): 20972 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-reporting-events-sub?project=highlevel-backend> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IZTYRCX67T33C|#140154 Pubsub Oldest Unacked Messages age above 30mins>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/demqv0uteacjkf/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-age/> Hey, looks like the age of the Oldest Unacked message in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-reporting-events-sub Age of the Oldest Unacked Messages(seconds): 20972 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-reporting-events-sub?project=highlevel-backend>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-18T02:27:37.620Z*