# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-16-a4621a40
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: crm-messages-search
**Channel**: slack
**Detected**: 2026-11-16T09:52:04.434Z
**Status**: delivering

## Root Cause

**Summary**: The storage threshold breach is confirmed, but its cause is unconfirmed. Index growth, shard relocation, and deployment-related effects remain unverified; AWS domain events, CloudTrail, per-node shard history, and per-index storage deltas are needed to distinguish them.
**Confidence**: low
**Category**: Unconfirmed AWS OpenSearch storage consumption or allocation change

## Triage

**Primary Anomaly**: none
**Details**: Free storage on OpenSearch node WzX5t4-HQeeX8JTy0AMNaA fell to 89,008.597 MB, below the 100,000 MB alert threshold. CPU, JVM pressure, indexing rate, and search rate show no clear anomaly. Slack notes earlier storage declines and shard relocations on this domain, but the initiating cause is unconfirmed; GCP logs and Kubernetes checks provide no relevant evidence. Historical candidates do not establish a matching cause.
**Anomalous**: Yes
**Deep Dive Direction**: YES. The direction is still correct: investigate OpenSearch storage usage and shard allocation or relocation, and verify the metrics deployment’s scope. The cause remains unconfirmed; we need AWS domain events and CloudTrail changes, plus time-aligned per-node shard allocation and per-index storage deltas for 08:45–10:00 UTC.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana AWS Elasticsearch dashboard | Confirms free storage on node WzX5t4-HQeeX8JTy0AMNaA was 89,008.597 MB, below the 100,000 MB threshold. CPU, JVM pressure, indexing, and search metrics show no clear corresponding surge. | ✓ |
| Slack alert thread and related messages | Confirms the alert and reports earlier storage declines and shard relocations, but records no cause for this node’s storage drop. The nearby shard movement alert concerned a different cluster. | ✓ |
| GCP logs and Kubernetes checks | No matching application WARNING/ERROR entries or Kubernetes evidence were found. These checks do not cover AWS domain events or storage allocation. | ✓ |
| Deep dive | Could not retrieve AWS domain events, CloudTrail changes, time-aligned shard assignments, or per-index storage deltas; the deployment’s scope also remains unverified. | ✓ |

### Discrepancies
- Grafana confirms the threshold breach, but available evidence does not identify what consumed storage.
- Prior reports of shard relocations and the nearby production deployment are temporally relevant but are not linked to this node’s storage change.
- GCP and Kubernetes checks do not provide evidence about AWS-side storage or shard allocation.

## Deep Dive Findings

{
  "deep_dive_findings": [
    {
      "finding": "This pass found no new evidence identifying what consumed storage on node WzX5t4-HQeeX8JTy0AMNaA. The recorded alert value is 89,008.597 MB, below the 100,000 MB threshold.",
      "assessment": "The storage breach is established; its cause remains unconfirmed. Prior metrics show no clear CPU, JVM pressure, indexing-rate, or search-rate surge.",
      "source_urls": [
        "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=WzX5t4-HQeeX8JTy0AMNaA&from=1794820894592&to=1794824494592&viewPanel=39"
      ],
      "confidence": "high"
    },
    {
      "finding": "Earlier same-domain storage declines and shard relocations were reported, but available records do not provide time-aligned shard assignments or per-index storage deltas for 08:45–10:00 UTC.",
      "assessment": "Relocation and index growth remain untested explanations; neither is established as the driver.",
      "source_urls": [
        "https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794819724914529?thread_ts=1794819724.914529"
      ],
      "confidence": "low"
    },
    {
      "finding": "The production message-search metrics deployment was reported before the alert, but its exact change scope and any relationship to AWS domain or index operations are not available in the inspected evidence.",
      "assessment": "Temporal proximity alone does not establish causation or prove the deployment unrelated.",
      "source_urls": [
        "https://gohighlevel.slack.com/archives/C096N87SBFW/p1794819112828199?thread_ts=1794819112.828199"
      ],
      "confidence": "low"
    },
    {
      "finding": "The available GCP log query returned no matching WARNING/ERROR entries, and the checked Kubernetes contexts did not identify this managed AWS OpenSearch node.",
      "assessment": "These checks do not explain AWS-side storage consumption or allocation changes.",
      "source_urls": [
        "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22CRM-conversations%22%0Aseverity%3DWARNING%0AjsonPayload.message%3A*;timeRange=2026-11-16T09%3A47%3A00Z%2F2026-11-16T09%3A57%3A00Z?project=highlevel-backend"
      ],
      "confidence": "medium"
    }
  ],
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"CRM-conversations\" AND severity=WARNING AND timestamp>=\"2026-11-16T09:47:00Z\" AND timestamp<=\"2026-11-16T09:57:00Z\"",
      "result": "Prior Layer 1 result reports no matching entries. This query covers the application routing label, not AWS OpenSearch domain events."
    }
  ],
  "log_explorer_urls": [
    "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22CRM-conversations%22%0Aseverity%3DWARNING%0AjsonPayload.message%3A*;timeRange=2026-11-16T09%3A47%3A00Z%2F2026-11-16T09%3A57%3A00Z?project=highlevel-backend"
  ],
  "cause": {
    "status": "unconfirmed",
    "confidence": "low",
    "assessment": "This pass could not retrieve AWS OpenSearch domain events, CloudTrail changes, per-node shard allocation, or per-index storage deltas. The metrics deployment’s scope also remains unverified.",
    "falsification": "Per-index growth on this node without matching shard arrivals would weigh against relocation as the main driver. Stable index sizes alongside shard arrivals whose sizes and timing match the storage drop would weigh against ordinary index growth."
  },
  "limitations": [
    "AWS CLI and a dedicated AWS/OpenSearch connector were unavailable in this executor; browser access was also unavailable.",
    "No AWS event, CloudTrail, shard-allocation, or per-index storage query was performed.",
    "No new orchestrator investigation was launched."
  ],
  "next_evidence_needed": [
    "Read-only AWS domain events and CloudTrail changes for 2026-11-16 08:45–10:00 UTC.",
    "Time-aligned shard assignments and relocation history for node WzX5t4-HQeeX8JTy0AMNaA, plus per-index store-size deltas across the domain.",
    "The metrics deployment’s exact diff and promotion time, compared with AWS domain and index operations."
  ]
}

## Evidence

### Dashboards (incident window)
- [Free storage — crm-messages-search node WzX5t4-HQeeX8JTy0AMNaA (below 100,000 MB threshold)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=WzX5t4-HQeeX8JTy0AMNaA&from=1794820894592&to=1794824494592&viewPanel=39)
- [CPU — crm-messages-search node WzX5t4-HQeeX8JTy0AMNaA (within seven-day range)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=WzX5t4-HQeeX8JTy0AMNaA&from=1794820894592&to=1794824494592&viewPanel=66)
- [JVM memory pressure — crm-messages-search node WzX5t4-HQeeX8JTy0AMNaA (near weekly baseline)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=WzX5t4-HQeeX8JTy0AMNaA&from=1794820894592&to=1794824494592&viewPanel=70)
- [Indexing rate — crm-messages-search node WzX5t4-HQeeX8JTy0AMNaA (below weekly median)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=WzX5t4-HQeeX8JTy0AMNaA&from=1794820894592&to=1794824494592&viewPanel=69)

### Related incidents
- [Earlier crm-messages-search storage alerts and investigation summary; cause unconfirmed](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794819724914529?thread_ts=1794819724.914529)
- [Nearby conversations-search Yellow alert; two shard movements reported, then resolved](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794819253149729?thread_ts=1794819253.149729)

## Investigation Timeline

**Deep dive passes**: 2
**Worker calls**: 12

**Total worker time**: 576.2s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.2s | ✓ |
| grafana-worker | worker | 152.2s | ✓ |
| kubectl-worker | worker | 77.4s | ✓ |
| log-explorer-worker | worker | 55.3s | ✓ |
| slack-worker | worker | 51.3s | ✓ |
| triage | worker | 12.4s | ✓ |
| triage_llm_ranking | decision | 37.9s | ✓ |
| deep_dive_pass_1 | worker | 82.1s | ✓ |
| follow_up_check | decision | 10.9s | ✓ |
| deep_dive_pass_2 | worker | 51.6s | ✓ |
| follow_up_check | decision | 12.6s | ✓ |
| cross_validation | worker | 21.4s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IX4CE8RX86LJ1|#139972 AWS ES Free Storage Space>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#free-storage|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having low storage space. Please check the details below, Cluster: crm-messages-search Node Id: WzX5t4-HQeeX8JTy0AMNaA Storage Space: 89008.597 MB Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=39&amp;var-domainName=crm-messages-search&amp;var-nodeId=WzX5t4-HQeeX8JTy0AMNaA> Help Doc : <http://platform.docs/data/faq/elasticsearch/elasticsearch-common-errors/#aws-es-free-storage-space-via-platform-critical> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IX4CE8RX86LJ1|#139972 AWS ES Free Storage Space>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#free-storage|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having low storage space. Please check the details below, Cluster: crm-messages-search Node Id: WzX5t4-HQeeX8JTy0AMNaA Storage Space: 89008.597 MB Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=39&amp;var-domainName=crm-messages-search&amp;var-nodeId=WzX5t4-HQeeX8JTy0AMNaA> Help Doc : <http://platform.docs/data/faq/elasticsearch/elasticsearch-common-errors/#aws-es-free-storage-space-via-platform-critical>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-16T10:00:22.679Z*