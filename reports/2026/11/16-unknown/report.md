# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-16-6364b28c
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: crm-messages-search
**Channel**: slack
**Detected**: 2026-11-16T09:05:15.122Z
**Status**: delivering

## Root Cause

**Summary**: The node experienced a sharp free-storage decline, but available evidence does not distinguish shard-relocation disk use, net index growth, or an AWS/OpenSearch node or domain event. Current cluster health and AWS-side evidence were unavailable, so the initiating cause remains unconfirmed.
**Confidence**: low
**Category**: Unconfirmed OpenSearch storage decline

## Triage

**Primary Anomaly**: none
**Details**: Free storage on OpenSearch node gcosutI9TaKBE1WKSS0saA fell from 141,275 MB at 08:50 UTC to 45,347 MB at 09:29 UTC, with no recovery observed. Slack reports concurrent shard relocations and imbalance, but their contribution versus index growth is unverified. Kubernetes and CRM-conversations logs do not explain this AWS storage signal. No historical candidate matches this storage-decline pattern.
**Anomalous**: Yes
**Deep Dive Direction**: YES. The direction is still correct: correlate AWS domain events and shard relocations with per-node/per-index storage changes and indexing throughput. More data is needed to confirm the root cause: current cluster health and storage, relocation timing and shard identities, domain events, and indexing or backfill throughput for 08:45–09:35 UTC.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | Free storage on node gcosutI9TaKBE1WKSS0saA fell from 141,275 MB at 08:50 UTC to 45,347 MB at 09:29 UTC; later status is unknown. | ✓ |
| Slack | Multiple low-storage alerts and shard rebalancing were reported around the incident, but shard movement's contribution to this node's decline is unverified. | ✓ |
| GCP logs | Indexing workers logged bulk-update timeouts at 09:34:56–09:34:59 UTC, after the available storage series ended; the logs do not establish the cause of the decline. | ✓ |
| Kubernetes | Pod and deployment state could not be verified; no matching workload was found in the current context and other production-context checks timed out. | ✓ |

### Discrepancies
- The alert reports 84,176 MB, matching the 09:03 UTC point; Grafana later shows 45,347 MB at 09:29 UTC.
- The reported shard rebalancing is not backed by retrieved relocation times, shard identities, or per-node and per-index storage deltas.
- Worker timeouts occurred after the observed decline and do not establish that indexing caused it.

## Deep Dive Findings

{
  "alert": {
    "name": "AWS ES Free Storage Space",
    "domain": "crm-messages-search",
    "node": "gcosutI9TaKBE1WKSS0saA",
    "alert_time_utc": "2026-11-16T09:05:15Z"
  },
  "findings": [
    {
      "finding": "The node’s free storage fell sharply during the alert window; its later status remains unknown.",
      "evidence": "Grafana shows 141275 MB at 08:50 UTC, 84176 MB at 09:03, and 45347 MB at 09:29. The available series ends at 09:29 UTC.",
      "confidence": "high",
      "source_urls": [
        "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-domainName=crm-messages-search&var-nodeId=gcosutI9TaKBE1WKSS0saA&from=1794818100000&to=1794821700000&viewPanel=39"
      ]
    },
    {
      "finding": "Shard relocations coincided with the storage decline, but their contribution versus net index growth is not established.",
      "evidence": "Slack reports concurrent shard movements and imbalance. No relocation start/end times, shard identities, or per-node and per-index storage deltas were available to compare with the decline.",
      "confidence": "low",
      "source_urls": [
        "https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794819915122739?thread_ts=1794819915.122739",
        "https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794821147707599?thread_ts=1794819724.914529"
      ]
    },
    {
      "finding": "Indexing workers showed concurrent OpenSearch impact, but available logs do not establish them as the cause of storage loss.",
      "evidence": "The prior read-only GCP query returned eight 30-second bulk-update timeout samples across multiple worker pods at 09:34:56–09:34:59 UTC. The broader CRM-conversations log query returned no ERROR or WARNING entries in its alert-time window.",
      "confidence": "medium",
      "source_urls": [
        "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-elastic-indexing-worker%22%0Atimestamp%3E%3D%222026-11-16T08%3A30%3A00Z%22%0Atimestamp%3C%3D%222026-11-16T09%3A35%3A00Z%22%0Aseverity%3E%3DERROR;timeRange=2026-11-16T08%3A30%3A00Z%2F2026-11-16T09%3A35%3A00Z?project=highlevel-backend",
        "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22CRM-conversations%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3A%2A;timeRange=2026-11-16T09%3A00%3A15Z%2F2026-11-16T09%3A10%3A15Z?project=highlevel-backend"
      ]
    },
    {
      "finding": "This pass could not retrieve the AWS-side evidence needed to confirm the initiating cause.",
      "evidence": "Grafana authentication validated and the AWS Elasticsearch dashboard definition was readable, but this executor has no AWS CLI or configured AWS environment credentials. Current cluster health, domain events, shard allocation history, per-index/per-node storage, and indexing or backfill throughput were not retrieved. The local investigation database contains no additional completed AWS-side results for this incident.",
      "confidence": "high",
      "source_urls": []
    },
    {
      "finding": "The nearby metrics deployment is not established as causal.",
      "evidence": "Slack records a message-search metrics change deploying to green before the alert and promotion afterward; no evidence links it to storage consumption or shard relocation.",
      "confidence": "medium",
      "source_urls": [
        "https://gohighlevel.slack.com/archives/C096N87SBFW/p1794819112828199?thread_ts=1794819112.828199"
      ]
    }
  ],
  "root_cause": {
    "status": "unconfirmed",
    "assessment": "The node lost substantial free storage while shard relocations were reported and indexing workers later timed out. Available evidence does not distinguish relocation-related disk use from index growth or an AWS domain/node event.",
    "hypotheses_not_confirmed": [
      "Shard relocation temporarily increased storage use on this node or changed its shard mix.",
      "Normal indexing, a backfill, or another producer caused net index growth.",
      "An AWS/OpenSearch domain or node event initiated the storage change."
    ],
    "disproof_checks": [
      "Compare domain-wide and per-node storage: offsetting changes on destination nodes would support shard movement; declines across nodes would support net growth.",
      "Align AWS domain events and relocation start/end times and shard identities with the 08:45–09:35 UTC storage series.",
      "Compare per-index store-size changes with indexing and backfill throughput over the same interval.",
      "Retrieve current cluster health and free storage to determine whether the decline continued after 09:29 UTC."
    ],
    "recommended_action": "OpenSearch owner should check current free storage and cluster health, then correlate domain events and shard allocation history with per-node/per-index storage and indexing or backfill throughput for 08:45–09:35 UTC."
  },
  "log_queries_used": [
    {
      "purpose": "Check indexing-worker errors during the storage decline and afterward",
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"conversations-elastic-indexing-worker\" AND timestamp>=\"2026-11-16T08:30:00Z\" AND timestamp<=\"2026-11-16T09:35:00Z\" AND severity>=ERROR",
      "result": "Prior pass returned eight projected samples reporting 30-second bulk-update timeouts across multiple worker pods at 09:34:56–09:34:59 UTC.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-elastic-indexing-worker%22%0Atimestamp%3E%3D%222026-11-16T08%3A30%3A00Z%22%0Atimestamp%3C%3D%222026-11-16T09%3A35%3A00Z%22%0Aseverity%3E%3DERROR;timeRange=2026-11-16T08%3A30%3A00Z%2F2026-11-16T09%3A35%3A00Z?project=highlevel-backend"
    },
    {
      "purpose": "Check CRM-conversations errors around alert detection",
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"CRM-conversations\" AND severity>=ERROR AND jsonPayload.message:*",
      "result": "No projected ERROR or WARNING entries or histogram time series were returned for the checked alert-time window and baseline.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22CRM-conversations%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3A%2A;timeRange=2026-11-16T09%3A00%3A15Z%2F2026-11-16T09%3A10%3A15Z?project=highlevel-backend"
    }
  ],
  "limitations": [
    "AWS domain events, current cluster health, shard allocation, per-index/per-node storage, and indexing/backfill throughput were unavailable in this executor.",
    "Grafana’s supplied storage series ends at 09:29 UTC; subsequent storage status is unknown.",
    "Worker timeouts confirm concurrent OpenSearch impact but do not prove that indexing caused the storage decline."
  ],
  "memory_context": "Prior memory lookup concerned an unrelated OpenSearch _id sorting migration and did not inform these findings."
}

## Evidence

### Dashboards (incident window)
- [Free storage — crm-messages-search node gcosutI9TaKBE1WKSS0saA (alert-window decline)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-domainName=crm-messages-search&var-nodeId=gcosutI9TaKBE1WKSS0saA&from=1794818100000&to=1794821700000&viewPanel=39)
- [Free storage — crm-messages-search node gcosutI9TaKBE1WKSS0saA (7-day context)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-domainName=crm-messages-search&var-nodeId=gcosutI9TaKBE1WKSS0saA&from=1794216720000&to=1794816720000&viewPanel=39)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22CRM-conversations%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3A%2A;timeRange=2026-11-16T09%3A00%3A15Z%2F2026-11-16T09%3A10%3A15Z?project=highlevel-backend)

### Related incidents
- [Message-search metrics deployment: green before alert; production promotion after alert](https://gohighlevel.slack.com/archives/C096N87SBFW/p1794819112828199?thread_ts=1794819112.828199)
- [Nearby conversations-search Yellow alert; two concurrent shard movements reported, then resolved](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794819253149729?thread_ts=1794819253.149729)
- [Investigation summary: storage decline coincided with shard relocations; initiating event and contribution from shard movement versus index growth remain unknown](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794821147707599?thread_ts=1794819724.914529)

## Investigation Timeline

**Deep dive passes**: 2
**Worker calls**: 12

**Total worker time**: 599.3s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 12.0s | ✓ |
| grafana-worker | worker | 83.8s | ✓ |
| kubectl-worker | worker | 66.3s | ✓ |
| log-explorer-worker | worker | 60.6s | ✓ |
| slack-worker | worker | 97.6s | ✓ |
| triage | worker | 14.8s | ✓ |
| triage_llm_ranking | decision | 31.3s | ✓ |
| deep_dive_pass_1 | worker | 99.6s | ✓ |
| follow_up_check | decision | 10.7s | ✓ |
| deep_dive_pass_2 | worker | 93.8s | ✓ |
| follow_up_check | decision | 13.4s | ✓ |
| cross_validation | worker | 15.4s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I8L469WY2ZGSD|#139963 AWS ES Free Storage Space>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#free-storage|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having low storage space. Please check the details below, Cluster: crm-messages-search Node Id: gcosutI9TaKBE1WKSS0saA Storage Space: 84176.273 MB Dashboard Link: https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&viewPanel=39&var-domainName=crm-messages-search&var-nodeId=gcosutI9TaKBE1WKSS0saA Help Doc : http://platform.docs/data/faq/elasticsearch/elasticsearch-common-errors/#aws-es-free-storage-space-via-platform-critical *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I8L469WY2ZGSD|#139963 AWS ES Free Storage Space>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#free-storage|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having low storage space. Please check the details below, Cluster: crm-messages-search Node Id: gcosutI9TaKBE1WKSS0saA Storage Space: 84176.273 MB Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=39&amp;var-domainName=crm-messages-search&amp;var-nodeId=gcosutI9TaKBE1WKSS0saA> Help Doc : <http://platform.docs/data/faq/elasticsearch/elasticsearch-common-errors/#aws-es-free-storage-space-via-platform-critical>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-16T09:37:49.031Z*