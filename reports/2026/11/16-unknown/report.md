# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-16-02dfa227
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: crm-messages-search
**Channel**: slack
**Detected**: 2026-11-16T09:02:04.914Z
**Status**: delivering

## Root Cause

**Summary**: Multi-node free-storage loss is supported by Grafana metrics and concurrent Slack alerts. Shard relocation coincided with the decline, but the initiating event and the relative contributions of shard movement and index growth remain unknown.
**Confidence**: low
**Category**: Unconfirmed OpenSearch storage depletion

## Triage

**Primary Anomaly**: none
**Details**: OpenSearch free storage is below the 100,000 MB threshold: the alert reports 83,940.64 MB, and the latest Grafana sample is 66,980 MB. CPU is low and indexing rate is below its seven-day median. Slack reports a near-simultaneous low-storage alert on another node and a same-day RCA citing elevated write and search load, but that cause is not independently verified. Historical candidates do not establish a matching RCA for this storage alert.
**Anomalous**: Yes
**Deep Dive Direction**: YES. The triage direction is still correct: investigate the time-aligned storage trends across both nodes and the domain, shard allocation and disk growth, workload and latency metrics, and AWS domain events or changes. Shard relocation is a plausible lead, but the evidence does not establish what triggered it or whether it explains the full storage loss.

More data is needed to determine root cause: AWS domain events and CloudTrail change history for Nov 11 and 16; shard allocation, per-index storage, and node filesystem data; and producer or backfill throughput with confirmation of any scheduled or manual jobs.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana CloudWatch-backed metrics | Free storage on the alert node fell sharply and was below the 100,000 MB threshold; CPU was low and the latest indexing rate was below its seven-day median. | ✓ |
| Slack alert context | A near-simultaneous low-storage alert reported another node below threshold, supporting a multi-node issue. A Slack RCA attributes strain to high write and search load, but that cause was not independently verified. | ✓ |
| Grafana shard and latency metrics | Shard relocations peaked while free storage declined, and latency rose. Initializing shards remained at zero and node count stayed at 93; relocation is a plausible mechanism, not a confirmed trigger. | ✓ |
| GCP indexing-worker logs | Sampled warnings show slow search operations and successful bulk batches. They support degraded operation but do not establish a write surge or explain the storage decline. | ✗ |

### Discrepancies
- The alert reports 83,940.64 MB free at 08:55 UTC, while the latest Grafana sample was 66,980 MB at 08:13 UTC.
- Slack attributes strain to high write and search load, but sampled Grafana rates were below their seven-day medians and logs do not measure total write volume.
- AWS domain events, CloudTrail changes, shard allocation, per-index storage, and producer or backfill throughput were not retrieved.

## Deep Dive Findings

{
  "alert": {
    "type": "AWS ES Free Storage Space",
    "domain": "crm-messages-search",
    "node_id": "gjjwJfeMS8C-EF-UiiMXlg",
    "started_at_utc": "2026-11-16T09:01:34Z",
    "reported_free_storage_mb": 83940.64,
    "threshold_free_storage_mb": 100000
  },
  "findings": [
    {
      "finding": "The storage breach is confirmed on multiple nodes. On the alert node, free storage fell from about 423,677 MB at 00:00 UTC to 66,980 MB by 08:13 UTC; the alert’s 08:55 value is 83,940.64 MB. A second node also fell below threshold during the alert window.",
      "confidence": "high",
      "evidence_source": "Grafana CloudWatch-backed FreeStorageSpace metrics and concurrent Slack alert",
      "url": "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=gjjwJfeMS8C-EF-UiiMXlg&from=1794214800000&to=1794821400000&viewPanel=39"
    },
    {
      "finding": "Shard relocation is a plausible mechanism, not a confirmed cause. Relocations peaked near 160 around 06:00 UTC while free storage fell sharply; initializing shards remained at zero and node count stayed at 93. A similar coincident pattern was reported on Nov 11. Available evidence does not identify what initiated relocation or show whether it explains the full storage loss.",
      "confidence": "medium",
      "evidence_source": "Grafana relocation, initializing-shard, node-count, and free-storage metrics from the prior pass",
      "url": "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=All&from=1794807000000&to=1794820500000&viewPanel=21"
    },
    {
      "finding": "The available domain metrics do not show a sustained indexing or search surge: indexing and search rates were below their seven-day medians in the cited samples. Latency rose around 06:00 UTC, and indexing-worker logs later showed slow bulk-operation warnings alongside successful batches. This supports degraded operation but does not establish a write surge or explain the storage decline.",
      "confidence": "medium",
      "evidence_source": "Grafana domain metrics and projected GCP indexing-worker logs",
      "url": "https://console.cloud.google.com/logs/query?project=highlevel-backend&query=resource.type%3D%22k8s_container%22%20AND%20resource.labels.container_name%3D%22conversations-elastic-indexing-messages-worker%22%20AND%20timestamp%3E%3D%222026-11-16T05%3A30%3A00Z%22%20AND%20timestamp%3C%3D%222026-11-16T09%3A15%3A00Z%22%20AND%20jsonPayload.message%3D%22Elasticsearch%20query%20latency%20is%20too%20high%20for%20Conversation%20Messages%20Search%22&timeRange=2026-11-16T05%3A30%3A00.000Z%2F2026-11-16T09%3A15%3A00.000Z"
    },
    {
      "finding": "Root cause remains unconfirmed. The supplied Slack RCA attributes strain to elevated write and search load but does not independently verify that cause; it also reports no deploy or scheduled-job correlation. AWS domain events, CloudTrail changes, shard-to-node allocation, per-index storage growth, and producer/backfill throughput were not retrieved. The local executor has no AWS CLI, so those AWS checks could not be performed here.",
      "confidence": "low",
      "evidence_source": "Prior investigation results, supplied Slack context, and local tool availability",
      "url": "https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794819915088539?thread_ts=1794819915.088539"
    }
  ],
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"conversations-elastic-indexing-messages-worker\" AND timestamp>=\"2026-11-16T05:30:00Z\" AND timestamp<=\"2026-11-16T09:15:00Z\" AND jsonPayload.message=\"Elasticsearch query latency is too high for Conversation Messages Search\"",
      "result": "Prior pass returned warnings at 09:11–09:14 UTC, with sampled latency about 5.3–30.1 seconds."
    },
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"conversations-elastic-indexing-messages-worker\" AND timestamp>=\"2026-11-16T05:30:00Z\" AND timestamp<=\"2026-11-16T09:15:00Z\" AND jsonPayload.message=~\"(?i)(bulk|index|write|processed|publish|batch|documents)\"",
      "result": "Prior bounded sample contained successful batches of 72–86 items and a bulk completion message; it does not measure total write volume."
    },
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"conversations-message-search-backfill-conversation-worker\" AND timestamp>=\"2026-11-16T05:30:00Z\" AND timestamp<=\"2026-11-16T09:15:00Z\" AND jsonPayload.message=~\"(?i)(bulk|index|write|processed|publish|batch|documents)\"",
      "result": "Prior pass returned no matching records."
    }
  ],
  "log_explorer_urls": [
    "https://console.cloud.google.com/logs/query?project=highlevel-backend&query=resource.type%3D%22k8s_container%22%20AND%20resource.labels.container_name%3D%22conversations-elastic-indexing-messages-worker%22%20AND%20timestamp%3E%3D%222026-11-16T05%3A30%3A00Z%22%20AND%20timestamp%3C%3D%222026-11-16T09%3A15%3A00Z%22%20AND%20jsonPayload.message%3D%22Elasticsearch%20query%20latency%20is%20too%20high%20for%20Conversation%20Messages%20Search%22&timeRange=2026-11-16T05%3A30%3A00.000Z%2F2026-11-16T09%3A15%3A00.000Z"
  ],
  "root_cause": {
    "status": "unconfirmed",
    "confidence": "low",
    "assessment": "Multi-node free-storage loss is established. Relocation coincided with the decline, while workload metrics do not establish a sustained traffic surge. The initiating event and the contribution of shard movement versus index growth remain unknown.",
    "next_evidence_needed": [
      "AWS OpenSearch domain events, service-software update history, CloudTrail changes, and AWS Health signals for Nov 11 and Nov 16.",
      "Time-aligned shard allocation, per-index store size, and node filesystem data.",
      "Producer and backfill throughput, including confirmation of scheduled or manual jobs."
    ],
    "recommended_action": "Have the domain owner verify headroom on affected nodes and inspect domain events and shard allocation before choosing a mitigation."
  }
}

## Evidence

### Dashboards (incident window)
- [Free storage — crm-messages-search node gjjwJfeMS8C-EF-UiiMXlg (threshold breach; 7-day view)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=gjjwJfeMS8C-EF-UiiMXlg&from=1794217894856&to=1794821494856&viewPanel=39)
- [Node CPU — crm-messages-search node gjjwJfeMS8C-EF-UiiMXlg (latest 5.12%; 7-day view)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=gjjwJfeMS8C-EF-UiiMXlg&from=1794217894856&to=1794821494856&viewPanel=66)
- [Indexing rate — crm-messages-search node gjjwJfeMS8C-EF-UiiMXlg (latest below 7-day median)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=gjjwJfeMS8C-EF-UiiMXlg&from=1794217894856&to=1794821494856&viewPanel=69)
- [Aws Elasticsearch — panel 39](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=gjjwJfeMS8C-EF-UiiMXlg&from=1794214800000&to=1794821400000&viewPanel=39)
- [Aws Elasticsearch — panel 21](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=All&from=1794807000000&to=1794820500000&viewPanel=21)

### Related incidents
- [Same-cluster RCA and investigation gaps](https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794819915088539?thread_ts=1794819915.088539)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 982.3s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 12.4s | ✓ |
| grafana-worker | worker | 131.7s | ✓ |
| kubectl-worker | worker | 106.1s | ✓ |
| log-explorer-worker | worker | 43.1s | ✓ |
| slack-worker | worker | 52.5s | ✓ |
| triage | worker | 12.8s | ✓ |
| triage_llm_ranking | decision | 32.9s | ✓ |
| deep_dive_pass_1 | worker | 71.6s | ✓ |
| follow_up_check | decision | 10.0s | ✓ |
| deep_dive_pass_2 | worker | 398.7s | ✓ |
| follow_up_check | decision | 12.2s | ✓ |
| deep_dive_pass_3 | worker | 82.5s | ✓ |
| cross_validation | worker | 15.8s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IMZ6U4G77UZS8|#139962 AWS ES Free Storage Space>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#free-storage|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having low storage space. Please check the details below, Cluster: crm-messages-search Node Id: gjjwJfeMS8C-EF-UiiMXlg Storage Space: 83940.64 MB Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=39&amp;var-domainName=crm-messages-search&amp;var-nodeId=gjjwJfeMS8C-EF-UiiMXlg> Help Doc : <http://platform.docs/data/faq/elasticsearch/elasticsearch-common-errors/#aws-es-free-storage-space-via-platform-critical> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IMZ6U4G77UZS8|#139962 AWS ES Free Storage Space>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#free-storage|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having low storage space. Please check the details below, Cluster: crm-messages-search Node Id: gjjwJfeMS8C-EF-UiiMXlg Storage Space: 83940.64 MB Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=39&amp;var-domainName=crm-messages-search&amp;var-nodeId=gjjwJfeMS8C-EF-UiiMXlg> Help Doc : <http://platform.docs/data/faq/elasticsearch/elasticsearch-common-errors/#aws-es-free-storage-space-via-platform-critical>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-16T09:25:35.546Z*