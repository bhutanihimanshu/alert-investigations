# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-16-41228911
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: conversations-search
**Channel**: slack
**Detected**: 2026-11-16T10:22:19.985Z
**Status**: delivering

## Root Cause

**Summary**: The storage alert coincided with a transient cluster disruption and recovered shortly afterward. A connection to node or shard activity is plausible but unverified; the specific cause is unknown.
**Confidence**: low
**Category**: Transient OpenSearch node or shard-allocation event (unconfirmed)

## Triage

**Primary Anomaly**: none
**Details**: The alert is for low free storage on OpenSearch node d_mZ4APpS8WI51_lagk5NA: 87,233 MB at alert time, about 36% below its seven-day median and below the 100,000 MB threshold. Other observed nodes stayed above threshold. A recent same-cluster node disruption is documented, but it involved a different node; causation is unconfirmed. Logs and Kubernetes checks did not verify the storage alert’s cause or current state.
**Anomalous**: Yes
**Deep Dive Direction**: Investigate the alerted node’s storage usage and shard allocation over time, then check cluster health and node-level events around the alert. Verify whether the same-cluster disruption affected this node; treat historical candidates as unrelated unless new evidence links them.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana metrics | The alerted node dropped below the 100,000 MB threshold, then recovered above it by 10:24Z. Cluster status was Yellow during part of the same interval and returned Green at 10:18Z; other observed nodes stayed above threshold. | ✓ |
| Slack context | A same-cluster node disruption and Yellow status were reported, but on a different node. The alert thread has no diagnosis linking that disruption to this storage dip. | ✓ |
| Kubernetes checks | Pod and deployment state could not be verified; queries timed out without matching the workload. | ✗ |
| GCP logs | The queried CRM-conversations container logs returned no entries and do not verify OpenSearch storage or shard allocation. | ✗ |

### Discrepancies
- The alert reported 87,233 MB free at 10:22Z; Grafana shows recovery above threshold by 10:24Z.
- The related node disruption involved a different node, and no evidence links it to the alerted node's storage dip.

## Deep Dive Findings

{
  "assessment": {
    "finding": "The alerting node’s free storage fell from about 128,788 MB at 10:00Z to 86,871 MB at 10:23Z, then rose to 123,467 MB at 10:24Z. The drop was transient; the node was above the 100,000 MB threshold in the latest retrieved datapoint at 10:31Z (122,635 MB).",
    "correlation": "The cluster was Yellow from 09:58Z through 10:17Z, and the reported node count fell from 36 to 34 around 10:10Z before returning to 36 by 10:15Z. Cluster status returned Green at 10:18Z. This timing aligns with the storage drop and rebound, consistent with a brief cluster/node or shard-allocation event, but the metrics do not identify the mechanism.",
    "same_cluster_disruption": "The earlier Fm8p disruption was a different node and occurred around 09:18–09:22Z. It does not explain the later storage dip based on available evidence. The 09:58–10:17Z Yellow period is a separate, temporally aligned cluster event; a direct link to this node or specific shard movement is unverified.",
    "disproof_check": "Other nodes remained above 100,000 MB during the sampled window, so the storage drop was node-specific rather than a cluster-wide free-storage decline. Cluster status and node count changed during the same broader interval, which argues against treating the storage dip as an isolated metric anomaly.",
    "confidence": "Medium that the alert coincided with a transient cluster disruption; low on the specific cause. Shard allocation history and node-level event details were not available.",
    "action": "If the pattern recurs, check OpenSearch shard allocation/recovery and AWS domain node events for this exact interval and node ID before attributing cause."
  },
  "queries_used": [
    {
      "source": "Grafana CloudWatch datasource, AWS/ES",
      "metrics": [
        "FreeStorageSpace by NodeId",
        "Nodes",
        "ClusterStatus.green",
        "ClusterStatus.yellow",
        "ClusterStatus.red",
        "ClusterUsedSpace"
      ],
      "filters": {
        "region": "us-east-1",
        "domain": "conversations-search",
        "clientId": "696084496265",
        "nodeId": "d_mZ4APpS8WI51_lagk5NA"
      },
      "windows": [
        "2026-11-09T10:40:00Z to 2026-11-16T10:40:00Z",
        "2026-11-16T09:00:00Z to 2026-11-16T10:45:00Z"
      ],
      "result": "Node-specific storage trajectory and cluster-status/node-count timelines retrieved."
    },
    {
      "source": "GCP Log Explorer",
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"CRM-conversations\" severity>=ERROR jsonPayload.message:*",
      "result": "No entries returned in the supplied Layer 1 query. This filters for a Kubernetes container and does not verify OpenSearch domain events or shard allocation."
    }
  ],
  "log_queries_used": [],
  "source_urls": {
    "grafana_alert_window": "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=d_mZ4APpS8WI51_lagk5NA&from=1794822739985&to=1794826339985&viewPanel=21",
    "grafana_seven_day_view": "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=d_mZ4APpS8WI51_lagk5NA&from=now-7d&to=now&viewPanel=21",
    "original_alert_thread": "https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794824539985319?thread_ts=1794824539.985319",
    "related_fm8p_disruption_thread": "https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794823800151019?thread_ts=1794820995.087279"
  }
}

## Evidence

### Dashboards (incident window)
- [Free storage — conversations-search node d_mZ4APpS8WI51_lagk5NA (alert window)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-nodeId=d_mZ4APpS8WI51_lagk5NA&from=1794822739985&to=1794826339985&viewPanel=39)
- [Aws Elasticsearch — panel 21](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=d_mZ4APpS8WI51_lagk5NA&from=1794822739985&to=1794826339985&viewPanel=21)
- [Aws Elasticsearch — panel 21](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=d_mZ4APpS8WI51_lagk5NA&from=now-7d&to=now&viewPanel=21)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22CRM-conversations%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3A%2A;timeRange=2026-11-16T10%3A17%3A19Z%2F2026-11-16T10%3A27%3A19Z?project=highlevel-backend)

### Related incidents
- [Same-cluster Yellow alerts #139959/#139969: node Fm8p disconnected repeatedly, then cluster returned green — 16 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794823800151019?thread_ts=1794820995.087279)
- [Related conversations-search Yellow alert #139959](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794819253149729?thread_ts=1794819253.149729)

## Investigation Timeline

**Deep dive passes**: 1
**Worker calls**: 10

**Total worker time**: 643.4s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.4s | ✓ |
| grafana-worker | worker | 157.3s | ✓ |
| kubectl-worker | worker | 86.5s | ✓ |
| log-explorer-worker | worker | 79.3s | ✓ |
| slack-worker | worker | 68.0s | ✓ |
| triage | worker | 14.2s | ✓ |
| triage_llm_ranking | decision | 36.6s | ✓ |
| deep_dive_pass_1 | worker | 161.5s | ✓ |
| follow_up_check | decision | 12.2s | ✓ |
| cross_validation | worker | 16.3s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IQ4P9W2VAJDKN|#139980 AWS ES Free Storage Space>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#free-storage|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having low storage space. Please check the details below, Cluster: conversations-search Node Id: d_mZ4APpS8WI51_lagk5NA Storage Space: 87233.054 MB Dashboard Link: https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&viewPanel=39&var-domainName=conversations-search&var-nodeId=d_mZ4APpS8WI51_lagk5NA Help Doc : http://platform.docs/data/faq/elasticsearch/elasticsearch-common-errors/#aws-es-free-storage-space-via-platform-critical *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IQ4P9W2VAJDKN|#139980 AWS ES Free Storage Space>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#free-storage|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having low storage space. Please check the details below, Cluster: conversations-search Node Id: d_mZ4APpS8WI51_lagk5NA Storage Space: 87233.054 MB Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=39&amp;var-domainName=conversations-search&amp;var-nodeId=d_mZ4APpS8WI51_lagk5NA> Help Doc : <http://platform.docs/data/faq/elasticsearch/elasticsearch-common-errors/#aws-es-free-storage-space-via-platform-critical>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-16T10:33:10.692Z*