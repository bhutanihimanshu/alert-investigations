# Alert Investigation: HighCPU

**Investigation ID**: inv-2026-11-16-96df73a8
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: conversations-search
**Channel**: slack
**Detected**: 2026-11-16T14:24:15.260Z
**Status**: delivering

## Root Cause

**Summary**: Elevated indexing and search activity coincided with the node CPU alert, but the causal mechanism and trigger are unconfirmed. No evidence establishes that earlier shard rebalancing, worker errors, or a deployment caused this spike.
**Confidence**: low
**Category**: Unconfirmed; elevated OpenSearch activity is a plausible contributor

## Triage

**Primary Anomaly**: traffic_spike
**Details**: CPU reached 85.69% against a 70% threshold, with indexing at 35,743 versus a 7-day average of 14,837 and search rate also above average. Slack reports cluster-average CPU at 48% versus a 21% baseline. The evidence points to elevated indexing activity accompanying the CPU spike, but does not establish its cause. Prior indexing-driven incidents are plausible matches; this alert’s evidence does not confirm their specific causal mechanisms.
**Anomalous**: Yes
**Deep Dive Direction**: Deep dive into whether elevated indexing or search load is driving the node CPU. Check OpenSearch write-pool rejections, shard distribution/rebalancing, query latency and sort behavior, backfill or reindex activity, and worker liveness/Pub/Sub redelivery. Compare the affected node and timing with peer nodes; current evidence does not yet support shortcut verification of a specific historical RCA.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana / CloudWatch metrics | Node CPU reached 85.69% against a 70% threshold; indexing and search rates were above their 7-day averages. This supports elevated activity coinciding with the alert, but does not establish causation. | ✓ |
| Slack alert thread | Reports cluster-average CPU at 48% versus a 21% baseline and higher search latency. Earlier shard rebalancing was mentioned, but no link to this node's CPU spike was verified; impact and recovery are unconfirmed. | ✓ |
| GCP logs | Selected ERROR and WARNING queries for one indexing worker returned no entries. This is limited evidence and does not rule out other workers, log severities, or Pub/Sub redelivery. | ✗ |
| OpenSearch node diagnostics | Write-pool rejections, shard placement or relocation, hot threads, active tasks, and node-level query behavior were not checked. | ✗ |

### Discrepancies
- The alert reports 85.69% CPU, while the queried CloudWatch series was 71.15% at the window end; these are different measurements or timestamps.
- Slack reports 48% cluster-average CPU, while the affected node was 71.15% at the queried window end.
- GCP and Kubernetes checks did not cover the AWS OpenSearch data node directly.

## Deep Dive Findings

{
  "alert": {
    "type": "HighCPU",
    "cluster": "conversations-search",
    "node": "9z5rx125RDGGUp_mrV6Rvw",
    "started_at_utc": "2026-11-16T14:24:14Z",
    "started_at_ist": "2026-11-16 19:54:14 IST",
    "reported_cpu_percent": 85.69,
    "threshold_percent": 70
  },
  "findings": [
    {
      "finding": "Elevated indexing and search coincide with the CPU alert, but available metrics do not establish that either caused this node's spike.",
      "evidence": "Indexing was 35,743 versus a 7-day average of 14,837; search was 3,522 versus 2,404. Node CPU was 71.15% at the queried window end, versus a 7-day average of 27.05%; peer median was 46.7%. Slack reported cluster-average CPU at 48% versus a 21% baseline and search latency at 8.68 ms versus 3.78 ms.",
      "confidence": "Medium for elevated activity; low for causation."
    },
    {
      "finding": "Worker ERROR logs and warning/error patterns searched for the indexing worker returned no entries in the queried interval.",
      "evidence": "Two Cloud Logging queries for `conversations-elastic-indexing-messages-worker` returned empty results: severity>=ERROR from 14:19–14:29Z; and severity>=WARNING with `rejected|AckError|429|redeliver|backfill|reindex|index` from 14:10–14:40Z. This does not rule out other worker names, INFO logs, or Pub/Sub redelivery.",
      "confidence": "Low; searches were limited to one worker container and specific severities/patterns."
    },
    {
      "finding": "The earlier Yellow cluster alert and reported shard rebalancing may be relevant, but no evidence links them to this CPU event.",
      "evidence": "Slack context says shard rebalancing was underway earlier on 16 Nov. Its timing and node-level impact relative to this alert were not verified.",
      "confidence": "Low."
    },
    {
      "finding": "The specific OpenSearch mechanisms requested remain unverified.",
      "evidence": "No live read-only OpenSearch queries were obtained for write-pool rejections/queue, shard placement or relocation, per-node query latency/sort behavior, active tasks/backfills, hot threads, or node-by-node comparisons at alert time. The authenticated Chrome session was unavailable to this executor; AWS CLI was not installed. GKE pod inventory does not represent this AWS data node.",
      "confidence": "Not assessed."
    }
  ],
  "causal_assessment": {
    "conclusion": "Elevated indexing/search activity is a plausible contributor, not a confirmed root cause. The evidence does not identify a specific trigger or support attributing this alert to a historical backfill/reindex mechanism.",
    "disconfirming_evidence_or_gaps": [
      "No OpenSearch write-pool or rejection measurements were available.",
      "No node-level shard distribution, relocation, or hot-thread evidence was available.",
      "No verified link between the earlier shard rebalancing and this node's CPU was established.",
      "The GCP log queries returned no matches, but covered only one container name and selected severities/patterns."
    ],
    "next_evidence_needed": [
      "Read-only OpenSearch checks for this node and peers: write/search thread-pool queues and rejections, shard allocation/relocation, node stats, hot threads, and active tasks.",
      "Time-align indexing/search rates, latency, and node CPU across affected and peer nodes.",
      "Check the actual indexing worker fleet's deployments, liveness/restarts, Pub/Sub ack/nack/redelivery, and any backfill/reindex activity around 14:24Z."
    ]
  },
  "log_queries": [
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"conversations-elastic-indexing-messages-worker\" AND timestamp>=\"2026-11-16T14:19:00Z\" AND timestamp<=\"2026-11-16T14:29:00Z\" AND severity>=ERROR",
      "result": "No entries returned.",
      "log_explorer_url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-elastic-indexing-messages-worker%22%0Atimestamp%3E%3D%222026-11-16T14%3A19%3A00Z%22%0Atimestamp%3C%3D%222026-11-16T14%3A29%3A00Z%22%0Aseverity%3E%3DERROR;timeRange=2026-11-16T14%3A19%3A00Z%2F2026-11-16T14%3A29%3A00Z?project=highlevel-backend"
    },
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"conversations-elastic-indexing-messages-worker\" AND timestamp>=\"2026-11-16T14:10:00Z\" AND timestamp<=\"2026-11-16T14:40:00Z\" AND severity>=WARNING AND jsonPayload.message=~\"rejected|AckError|429|redeliver|backfill|reindex|index\"",
      "result": "No entries returned.",
      "log_explorer_url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-elastic-indexing-messages-worker%22%0Atimestamp%3E%3D%222026-11-16T14%3A10%3A00Z%22%0Atimestamp%3C%3D%222026-11-16T14%3A40%3A00Z%22%0Aseverity%3E%3DWARNING%0AjsonPayload.message%3D~%22rejected%7CAckError%7C429%7Credeliver%7Cbackfill%7Creindex%7Cindex%22;timeRange=2026-11-16T14%3A10%3A00Z%2F2026-11-16T14%3A40%3A00Z?project=highlevel-backend"
    }
  ],
  "metric_urls": [
    "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=9z5rx125RDGGUp_mrV6Rvw&from=1794837240000&to=1794840840000&viewPanel=66",
    "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=9z5rx125RDGGUp_mrV6Rvw&from=1794837240000&to=1794840840000&viewPanel=69",
    "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=9z5rx125RDGGUp_mrV6Rvw&from=1794837240000&to=1794840840000&viewPanel=68"
  ],
  "slack_context": {
    "current_alert_thread": "https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794839055479519?thread_ts=1794839055.479519",
    "earlier_yellow_and_rebalancing_thread": "https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794819253198779?thread_ts=1794819253.198779",
    "summary": "Slack reports no captured slow queries, consumer errors, or cron/deploy events in the sources checked; cause, impact, and recovery remain unconfirmed."
  }
}

## Evidence

### Dashboards (incident window)
- [CPU utilization — conversations-search node 9z5rx125RDGGUp_mrV6Rvw (alert 85.69%, queried end 71.15%)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=9z5rx125RDGGUp_mrV6Rvw&from=1794837240000&to=1794840840000&viewPanel=66)
- [Indexing rate — conversations-search (latest above 7-day average)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=9z5rx125RDGGUp_mrV6Rvw&from=1794837240000&to=1794840840000&viewPanel=69)
- [Search rate — conversations-search (latest above 7-day average)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=9z5rx125RDGGUp_mrV6Rvw&from=1794837240000&to=1794840840000&viewPanel=68)

### Related incidents
- [Same-cluster Yellow alert and shard rebalancing — 16 Nov](https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794819253198779?thread_ts=1794819253.198779)
- [Earlier conversations-search CPU alert — 11 Nov](https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794409937855609?thread_ts=1794409937.855609)
- [Current conversations-search CPU alert and RCA — 16 Nov](https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794839055479519?thread_ts=1794839055.479519)

## Investigation Timeline

**Deep dive passes**: 1
**Worker calls**: 10

**Total worker time**: 732.7s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 16.6s | ✓ |
| grafana-worker | worker | 96.3s | ✓ |
| kubectl-worker | worker | 104.2s | ✓ |
| log-explorer-worker | worker | 60.2s | ✓ |
| slack-worker | worker | 72.5s | ✓ |
| triage | worker | 12.5s | ✓ |
| triage_llm_ranking | decision | 21.4s | ✓ |
| deep_dive_pass_1 | worker | 321.3s | ✓ |
| follow_up_check | decision | 12.4s | ✓ |
| cross_validation | worker | 15.5s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/INKZVTPT3N9FF|#140002 AWS ES Data Node CPU Utilization>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#cpu|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having high CPU usage. Please check the details below, Cluster: conversations-search Node: 9z5rx125RDGGUp_mrV6Rvw CPU Utilization: 85.68562268963808% Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=66&amp;var-domainName=conversations-search&amp;var-nodeId=9z5rx125RDGGUp_mrV6Rvw> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/INKZVTPT3N9FF|#140002 AWS ES Data Node CPU Utilization>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#cpu|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having high CPU usage. Please check the details below, Cluster: conversations-search Node: 9z5rx125RDGGUp_mrV6Rvw CPU Utilization: 85.68562268963808% Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=66&amp;var-domainName=conversations-search&amp;var-nodeId=9z5rx125RDGGUp_mrV6Rvw>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-16T14:35:32.364Z*