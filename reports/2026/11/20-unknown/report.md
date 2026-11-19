# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-19-ede06af2
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: crm-messages-search
**Channel**: slack
**Detected**: 2026-11-19T21:16:51.919Z
**Status**: delivering

## Root Cause

**Summary**: Persistent node-level low storage is supported by metrics. November 16 node reduction and shard redistribution are a plausible trigger, independently corroborated as events, but their causal connection to storage depletion is unproven. Application impact remains unverified. Obtain live domain configuration, change history, per-node filesystem and shard statistics, index blocks, and the writer's domain binding before confirming an RCA.
**Confidence**: low
**Category**: undetermined_storage_capacity_or_allocation

## Triage

**Primary Anomaly**: none
**Details**: Confirmed storage anomaly, outside the supplied categories: crm-messages-search node gcosutI9TaKBE1WKSS0saA has 33,181 MB free versus a 100,000 MB threshold; another node has 12,345 MB while the median across 80 reporting nodes is 2,437,990 MB. No concurrent CPU, sustained JVM pressure, or 5xx anomaly was established. Candidate inv-2026-11-16-41228911 is relevant but has no confirmed RCA; its transient recovery does not match the current persistent low storage. Slack reports 90→80 nodes and shard redistribution, corroborating a topology change but not storage causation. GCP queries used an unverified container mapping and provide no evidence of application health.
**Anomalous**: Yes
**Deep Dive Direction**: YES — The triage direction remains correct: November 16 shard redistribution is the leading suspected trigger, but causation and the storage-loss mechanism remain unconfirmed.

Need AWS change history/CloudTrail, per-node disk capacities and shard sizes, filesystem/index statistics, and allocation settings/explanations. Verify index blocks and the writer’s live domain binding separately to establish application impact.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana / CloudWatch | Confirms persistent low free storage on two nodes and a node-count reduction. Does not establish disk capacities, shard imbalance, or the storage-loss mechanism. | ✓ |
| Slack human context | Independently corroborates a 90-to-80 node reduction and shard redistribution, but does not confirm that redistribution caused the current storage depletion. | ✓ |
| Deep-dive CloudWatch queries | Approximately 0.51% domain storage growth weakens sudden domain-wide growth as the explanation. These queries reuse the Grafana data source and are not independent confirmation. | ✓ |
| GCP logs / Kubernetes | Unverified workload mapping and unavailable Kubernetes context provide no evidence for the proposed cause or application health. | ✗ |
| Terraform / historical investigation | Local configuration conflicts with observed topology; the historical confirmation flag accompanies an explicitly unknown cause. Neither validates the RCA. | ✗ |

### Discrepancies
- CloudWatch reports 93-to-83 total nodes; Slack reports 90-to-80. Different node-role accounting could explain this, but remains unverified.
- Only topology change has independent corroboration. Storage causation lacks per-node capacity, shard-size, filesystem, and allocation evidence.
- Historical write blocking occurred on November 17; recent sampled values are zero. Neither establishes current successful application indexing.
- No concurrent CPU, sustained JVM-pressure, or 5xx anomaly was established; missing GCP logs cannot establish absence of application impact.
- Falsification remains incomplete: compare affected-node capacities and shard allocation with healthy peers and pre-change history. Capacity differences or node-local growth could explain low free space without redistribution being causal.

## Deep Dive Findings

{
  "domain": "crm-messages-search",
  "root_cause": {
    "confirmed": false,
    "assessment": "Node reduction and shard redistribution remain the leading suspected trigger. New metrics weaken a sudden domain-wide data-growth explanation, but do not establish why two nodes have disproportionately low free storage.",
    "confidence": "Medium for trigger; low for storage-loss mechanism."
  },
  "new_findings": [
    {
      "finding": "Domain storage growth was modest despite the node-count reduction.",
      "evidence": "CloudWatch ClusterUsedSpace increased approximately 0.51% across November 14–20 IST. Nodes fell from 93 to 83 in the hourly bucket beginning November 16, 17:46 IST, without a corresponding abrupt increase in ClusterUsedSpace.",
      "interpretation": "Consistent with existing data being redistributed onto fewer nodes. Does not prove unequal shard allocation or exclude node-specific growth."
    },
    {
      "finding": "Historical write blocking was independently reconfirmed.",
      "evidence": "ClusterIndexWritesBlocked hourly Maximum reached 1 in the bucket beginning November 17, 20:46 IST. The final two returned buckets around this alert were 0.",
      "limitation": "This does not verify individual index block settings or successful application indexing."
    },
    {
      "finding": "Local Terraform cannot establish live disk capacity.",
      "evidence": "platform-backend/terraform/aws-opensearch/production/crm-clusters.tf:373–406 specifies 3 data nodes, r7g.2xlarge.search and 3096 GiB EBS per node. This conflicts with the supplied incident evidence of 80 reporting storage nodes.",
      "limitation": "The file has no local modifications; deployed configuration and change history remain unverified."
    },
    {
      "finding": "The historical RCA confirmation flag remains misleading.",
      "evidence": "Read-only SQLite lookup reconfirmed rca_confirmed=1 for inv-2026-11-16-41228911, while its stored conclusion explicitly states the specific cause is unknown."
    }
  ],
  "application_impact": {
    "writer_live_domain_binding": "Unverified.",
    "individual_index_blocks": "Unverified.",
    "successful_indexing": "Unverified; absence of application ERROR logs in the previous pass is insufficient."
  },
  "access_limitations": [
    "Existing Grafana session validated; CloudWatch queries succeeded through its read-only query API.",
    "AWS CLI and ~/.aws remain absent; no native AWS or browser tool was available.",
    "CloudTrail, domain change history, per-node capacities, shard sizes, filesystem statistics and allocation explanations remain unavailable.",
    "Parallel delegation failed with a thread-store error."
  ],
  "required_next_evidence": [
    "AWS domain configuration and CloudTrail records for the November 16 node reduction.",
    "Authenticated _cat/allocation, _cat/shards and _nodes/stats/fs,indices for affected nodes and healthy peers.",
    "Cluster allocation settings, allocation explanations, individual index block settings and the writer's live domain binding."
  ],
  "metric_queries_used": {
    "namespace": "AWS/ES",
    "region": "us-east-1",
    "domain": "crm-messages-search",
    "client_id": "696084496265",
    "window_utc": ["2026-11-13T21:16:51Z", "2026-11-19T21:26:51Z"],
    "period_seconds": 3600,
    "metrics": {
      "ClusterUsedSpace": "Average",
      "Nodes": "Minimum",
      "ClusterIndexWritesBlocked": "Maximum",
      "DeletedDocuments": "Average",
      "SearchableDocuments": "Average"
    }
  },
  "source_url": "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&from=1794604611000&to=1795123611000",
  "log_queries_used": [],
  "log_explorer_urls": []
}

## Evidence

### Dashboards (incident window)
- [Free storage — crm-messages-search: two nodes below 100,000 MB](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=gcosutI9TaKBE1WKSS0saA&from=1795121211615&to=1795123166851&viewPanel=21)
- [CPU — affected crm-messages-search node: low utilization](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=gcosutI9TaKBE1WKSS0saA&from=1795121211615&to=1795123166851&viewPanel=66)
- [JVM pressure — affected crm-messages-search node: no sustained elevation](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=gcosutI9TaKBE1WKSS0saA&from=1795121211615&to=1795123166851&viewPanel=70)
- [HTTP responses — crm-messages-search: no concurrent 5xx spike](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=gcosutI9TaKBE1WKSS0saA&from=1795121211615&to=1795123166851&viewPanel=34)

### Related incidents
- [Team reports 90→80 nodes and shard redistribution — 16 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794825127209359)
- [Production message-read Go canary announcement — 19 Nov](https://gohighlevel.slack.com/archives/C096N87SBFW/p1795083793657419)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 697.3s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.7s | ✓ |
| grafana-worker | worker | 102.8s | ✓ |
| kubectl-worker | worker | 26.3s | ✓ |
| log-explorer-worker | worker | 58.8s | ✓ |
| slack-worker | worker | 74.1s | ✓ |
| triage | worker | 14.1s | ✓ |
| triage_llm_ranking | decision | 38.1s | ✓ |
| deep_dive_pass_1 | worker | 100.4s | ✓ |
| follow_up_check | decision | 10.4s | ✓ |
| deep_dive_pass_2 | worker | 135.6s | ✓ |
| follow_up_check | decision | 12.4s | ✓ |
| deep_dive_pass_3 | worker | 93.9s | ✓ |
| cross_validation | worker | 18.7s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IUD66LB9158PM|#140358 AWS ES Free Storage Space>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#free-storage|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having low storage space. Please check the details below, Cluster: crm-messages-search Node Id: gcosutI9TaKBE1WKSS0saA Storage Space: 33212.414 MB Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=39&amp;var-domainName=crm-messages-search&amp;var-nodeId=gcosutI9TaKBE1WKSS0saA> Help Doc : <http://platform.docs/data/faq/elasticsearch/elasticsearch-common-errors/#aws-es-free-storage-space-via-platform-critical> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IUD66LB9158PM|#140358 AWS ES Free Storage Space>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#free-storage|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having low storage space. Please check the details below, Cluster: crm-messages-search Node Id: gcosutI9TaKBE1WKSS0saA Storage Space: 33212.414 MB Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=39&amp;var-domainName=crm-messages-search&amp;var-nodeId=gcosutI9TaKBE1WKSS0saA> Help Doc : <http://platform.docs/data/faq/elasticsearch/elasticsearch-common-errors/#aws-es-free-storage-space-via-platform-critical>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-19T21:27:17.120Z*