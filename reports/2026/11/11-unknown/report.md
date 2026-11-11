# Alert Investigation: HighCPU

**Investigation ID**: inv-2026-11-11-d8bc3f6b
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: conversations-search
**Channel**: slack
**Detected**: 2026-11-11T15:12:17.839Z
**Status**: delivering

## Root Cause

**Summary**: A CPU and latency hotspot is established on conversations-search node VZqSzDJmSwKRC_So18KrXA. Increased indexing coincided with the event, but does not explain the node's excess CPU relative to peers. Write contention, costly searches and background work remain unconfirmed alternatives.
**Confidence**: low
**Category**: undetermined

## Triage

**Primary Anomaly**: cpu_high
**Details**: AWS OpenSearch conversations-search node VZqSzDJmSwKRC_So18KrXA sustained >70% CPU during 15:00–15:14 UTC, peaking at 90.69%. Recent mean CPU was 81.21% versus 30.64% seven-day hourly baseline and 54.64% concurrent peer median. Search and indexing latency increased despite comparable search activity and lower indexing activity than peers. JVM pressure and shard relocation were not elevated. Historical indexing-driven incidents partially match, but current evidence does not establish their specific causes. GCP queries targeted the OnCall routing label, so their empty results do not rule out application errors.
**Anomalous**: Yes
**Deep Dive Direction**: YES — Write contention remains plausible, but increased writes do not explain the node-specific CPU hotspot. Retry amplification is weakened by acknowledgments scaling with deliveries.

Need incident-time location-level document writes mapped to primary/replica shard placement, hot threads, ordinary merge-counter deltas, and slow logs to distinguish write contention from expensive searches, update costs, or shard-specific background work.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | Confirms a node-specific CPU and latency hotspot during 15:00–15:14 UTC. CPU peaked at 90.69%; comparable search activity and lower indexing activity than peers weaken a simple volume-driven explanation. | ✓ |
| Slack | Confirms the alert identity but provides no current diagnosis. The duplicate alert repeats the same signal and is not independent corroboration. Previous contact-indexing bursts establish a hypothesis only. | ✓ |
| Source inspection | Confirms plausible mechanisms: location-based routing, scripted updates, conflict retries and multiple conversation-index writers. No incident-time evidence establishes which mechanism consumed CPU; deployed revision is unverified. | ✓ |
| GCP logs and Kubernetes | Provide no causal corroboration: initial queries targeted an OnCall routing label, the corrected application query timed out, and Kubernetes checks do not apply to the managed OpenSearch node. | ✗ |

### Discrepancies
- No two independent sources establish the causal chain from workload operations to affected shards to CPU saturation.
- Normal JVM pressure and zero reported shard relocations weaken those explanations; they do not exclude expensive queries, update contention or ordinary background merges.
- Acknowledgments scaling with deliveries weakens PubSub retry amplification, but does not exclude internal OpenSearch conflict retries.
- Routing telemetry records attempted writes, not successful writes; the force-merge panel cannot measure ordinary merges.
- Falsification requires incident-time shard placement and operation-level evidence: normal write costs alongside elevated search or merge work would contradict write contention. These observations are unavailable; deployment causality and recovery remain unverified.

## Deep Dive Findings

{
  "status": "inconclusive",
  "root_cause_confidence": "low",
  "summary": "Source inspection identifies location-routing telemetry and scripted updates as useful investigation paths, but no incident-time evidence distinguishes write contention, expensive searches, or background merges.",
  "window_utc": "2026-11-11T15:00:00Z/2026-11-11T15:15:00Z",
  "findings": [
    {
      "finding": "Routing telemetry records attempted operations, not confirmed successful writes.",
      "evidence": "The conversation worker emits conversationId, routingKey and operation to crm-conversations-elastic-indexing-routing-update-events-YYYY-MM-DD with samplingPercentage=100 before submitting the bulk request.",
      "implication": "The November 11 index is a candidate source for location-level attempted-write distribution. Records require success correlation and primary/replica placement before attributing this node's CPU.",
      "source": "apps/conversations/workers/conversations-elastic-indexing-worker.ts:251,387,897,1018"
    },
    {
      "finding": "Conversation updates execute a Painless script and configure retry_on_conflict=10.",
      "evidence": "The script clears and repopulates document source while preserving selected fields; updates use locationId routing.",
      "implication": "Update cost and internal conflict retries remain plausible even when PubSub acknowledgments scale with deliveries. Neither mechanism was observed during this incident.",
      "source": "apps/conversations/workers/conversations-elastic-indexing-worker.ts:926"
    },
    {
      "finding": "Conversation-worker telemetry alone cannot cover all conversation-index writes.",
      "evidence": "The contact indexing handler independently constructs conversation updates using contact.locationId routing and submits bulk(Index.Conversation).",
      "source": "apps/elastic-indexing/workers/elastic-indexing-v3-worker/handlers/elasticsearch_functions.ts:1108"
    },
    {
      "finding": "The saved dashboard's Merge Thread Pool panel measures force-merge metrics only.",
      "implication": "It cannot establish ordinary background merge activity or exclude merges as the CPU cause.",
      "source": "/private/tmp/es139587-dashboard.json, panel 47"
    }
  ],
  "evidence_limitations": [
    "Source inspected at local revision 4f439f19855992d3d31e9dc8b6fdd269344888a8; deployed revision was not verified.",
    "Fresh Grafana validation returned HTTP 401 session.token.rotate; native browser discovery reported browser access host unavailable.",
    "No incident-time shard placement, hot threads, ordinary merge-counter deltas, or OpenSearch slow logs were obtained. Prior credential-access denial was not bypassed.",
    "SQLite's recent investigations also retain unconfirmed causes; historical similarity provides no causal confirmation.",
    "Parallel-agent launch failed because executor thread context could not be loaded."
  ],
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-elastic-indexing-worker\" timestamp>=\"2026-11-11T15:00:00Z\" timestamp<=\"2026-11-11T15:05:00Z\" severity>=WARNING (jsonPayload.message:\"Version conflict - newer version already indexed\" OR jsonPayload.message:\"Error occurred while updating conversation\" OR jsonPayload.message:\"Error occurred while bulk updating conversations\")",
      "limit": 5,
      "result": "Timed out after 35 seconds; no result, not evidence of absent errors."
    }
  ],
  "log_explorer_urls": [],
  "next_step": "Retrieve the incident-window routing telemetry and successful item outcomes from all conversation-index writers; join them to incident-time primary/replica placement. Compare affected-shard indexing, search and ordinary merge deltas with peers, supplemented by retained hot-thread captures or slow logs. Current captures cannot reconstruct past execution.",
  "production_changes": false,
  "messages_posted": false
}

## Evidence

### Dashboards (incident window)
- [CPU — conversations-search nodes: affected node reaches 90.69%](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=All&from=1794408137553&to=1794411737553&viewPanel=66)
- [Indexing rate — conversations-search: increased load near CPU onset](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=All&from=1794408137553&to=1794411737553&viewPanel=69)
- [Search latency — conversations-search: affected node above peers](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=All&from=1794408137553&to=1794411737553&viewPanel=71)
- [HTTP responses — conversations-search: no concurrent 5xx surge](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=All&from=1794408137553&to=1794411737553&viewPanel=34)

### Related incidents
- [Duplicate platform alert for the same CPU event — 11 Nov](https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794409937855609)
- [Previous hot node linked by team to contact-indexing bursts — 10–11 Nov](https://gohighlevel.slack.com/archives/C0ANDK1TU30/p1794384129597979)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 1126.3s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 15.5s | ✓ |
| grafana-worker | worker | 92.4s | ✓ |
| kubectl-worker | worker | 13.1s | ✓ |
| log-explorer-worker | worker | 158.2s | ✓ |
| slack-worker | worker | 97.2s | ✓ |
| triage | worker | 12.1s | ✓ |
| triage_llm_ranking | decision | 55.2s | ✓ |
| deep_dive_pass_1 | worker | 380.5s | ✓ |
| follow_up_check | decision | 10.3s | ✓ |
| deep_dive_pass_2 | worker | 152.8s | ✓ |
| follow_up_check | decision | 11.7s | ✓ |
| deep_dive_pass_3 | worker | 108.7s | ✓ |
| cross_validation | worker | 18.5s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IICRIM869MDDT|#139587 AWS ES Data Node CPU Utilization>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#cpu|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having high CPU usage. Please check the details below, Cluster: conversations-search Node: VZqSzDJmSwKRC_So18KrXA CPU Utilization: 81.52781891319329% Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=66&amp;var-domainName=conversations-search&amp;var-nodeId=VZqSzDJmSwKRC_So18KrXA> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IICRIM869MDDT|#139587 AWS ES Data Node CPU Utilization>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#cpu|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having high CPU usage. Please check the details below, Cluster: conversations-search Node: VZqSzDJmSwKRC_So18KrXA CPU Utilization: 81.52781891319329% Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=66&amp;var-domainName=conversations-search&amp;var-nodeId=VZqSzDJmSwKRC_So18KrXA>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-11T15:29:29.085Z*