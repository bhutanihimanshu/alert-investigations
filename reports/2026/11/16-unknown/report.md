# Alert Investigation: HighCPU

**Investigation ID**: inv-2026-11-16-792ee677
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: conversations-search
**Channel**: slack
**Detected**: 2026-11-16T16:19:18.907Z
**Status**: delivering

## Root Cause

**Summary**: Cause undetermined. The alert indicates anomalously high node CPU, but available data does not connect it to indexing/search activity, fielddata, backfill, a deployment, or another specific mechanism.
**Confidence**: low
**Category**: Undetermined — node CPU anomaly

## Triage

**Primary Anomaly**: cpu_high
**Details**: Alert reports 78.39% node CPU, above the 70% threshold and well above the 7-day median (23.60%). Hourly data shows CPU near peers afterward, but cannot verify the 10-minute alert window. Search latency, JVM memory, and fleet indexing/search rates show no confirmed anomaly in the available hourly samples. Logs returned no entries; Kubernetes state could not be verified. Prior CPU RCAs are plausible historical matches, but current evidence does not identify their causes.
**Anomalous**: Yes
**Deep Dive Direction**: YES. The triage direction is still appropriate as a set of hypotheses, but no cause is established; first verify the affected cluster and node-level CPU and indexing/search activity during the alert window. Then check relevant worker health, Pub/Sub redelivery, `_id` sort fielddata, and backfill/write-pool activity.

More data is needed to determine root cause: OpenSearch diagnostics and rates at alert time, plus matching worker deployment/restart and Pub/Sub evidence. The Redis-learning does not contradict this direction; there is no Redis evidence here, so don’t attribute a fleet-wide issue to this workload without checking peer workers and comparing their signals.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | The alert reports node CPU at 78.39%, above the 70% threshold and 7-day median. Available hourly data does not cover the 10-minute alert window; the latest sample was near peers. | ✓ |
| GCP Logs | No ERROR or WARNING entries were returned for CRM-conversations in the queried alert window. Log histograms had no time-series points, so logs do not establish a cause. | ✗ |
| Kubernetes | Pod and deployment state could not be verified because the active context had no CRM-conversations pods and the production context failed TLS verification. | ✗ |
| Slack | The current alert thread and same-day CPU discussion provide no confirmed cause or resolution; workload attribution remains unknown. | ✓ |
| Historical investigations | Indexing activity, _id fielddata, and backfill were proposed in earlier investigations but were not confirmed for this node or alert window. | ✗ |

### Discrepancies
- The reported 10-minute CPU condition cannot be checked against the available hourly Grafana samples.
- Alert-time logs returned no entries, and Kubernetes state was unavailable; neither confirms nor rules out a workload-level cause.

## Deep Dive Findings

{
  "alert": {
    "type": "HighCPU",
    "workload": "CRM-conversations",
    "cluster": "conversations-search",
    "node": "9z5rx125RDGGUp_mrV6Rvw",
    "received_at_ist": "2026-11-16 21:49",
    "reported_cpu_percent": 78.39,
    "threshold": "CPU >70% for 10 minutes"
  },
  "findings": [
    {
      "finding": "Historical same-day investigations recorded elevated indexing and search activity near an earlier CPU alert, but did not establish a causal mechanism. The available hourly samples for this alert do not verify activity during its 10-minute window.",
      "confidence": "Low",
      "source": "Grafana hourly samples and prior investigation inv-2026-11-16-96df73a8"
    },
    {
      "finding": "A previous conversations-search incident found heavy _id sort fielddata use. The code path was changed to sort by id, but production fielddata reduction was not verified; recurrence here is unconfirmed.",
      "confidence": "Low",
      "source": "Historical production OpenSearch measurements and code-change status; no alert-time fielddata metrics"
    },
    {
      "finding": "A prior single-node hotspot remained unexplained; silent backfill workers were circumstantial evidence only. This alert’s data does not confirm backfill activity or an orphaned OpenSearch task.",
      "confidence": "Low",
      "source": "Historical node-hotspot investigation; no alert-time task or worker evidence"
    },
    {
      "finding": "No ERROR or WARNING entries were returned for the queried container and alert window. This does not rule out OpenSearch-internal activity, other containers, or unlogged causes. Kubernetes state was not verified because the available context lacked the workload and the production context failed TLS verification.",
      "confidence": "Low",
      "source": "GCP Logs and Kubernetes worker outputs"
    },
    {
      "finding": "Slack context contains no confirmed cause or resolution for this alert. Earlier same-day discussion mentioned cluster-wide CPU and search-latency increases but did not identify workload attribution.",
      "confidence": "Low",
      "source": [
        "https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794845958933149?thread_ts=1794845958.933149",
        "https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794839055479519?thread_ts=1794839055.479519"
      ]
    }
  ],
  "causal_assessment": {
    "conclusion": "Cause undetermined. Historical indexing, _id fielddata, and backfill patterns are hypotheses only; none is tied to this node’s alert window.",
    "disconfirming_evidence_and_gaps": [
      "The latest completed hourly CPU sample was 54.96%, near the peer median of 49.94%, but does not cover the alert window.",
      "Available hourly indexing and search rates do not establish an alert-time spike or node-level workload imbalance.",
      "No live OpenSearch hot-thread, task, shard, fielddata, or thread-pool diagnostics were available.",
      "Worker deployment, restart, Pub/Sub ack/nack/redelivery, and backfill activity were not verified."
    ],
    "next_evidence_needed": [
      "Alert-window node and peer CPU plus indexing/search rates, hot threads, active tasks, shard placement or movement, and search/write queue and rejection metrics.",
      "Conversation index _id fielddata and eviction metrics on this node; verify whether production uses the id sort path.",
      "Matching indexing/backfill worker deployments, restarts, and Pub/Sub delivery or redelivery evidence."
    ]
  },
  "log_queries": [
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"CRM-conversations\" AND jsonPayload.message=~\".+\" AND severity=ERROR; 2026-11-16T16:14:18Z–2026-11-16T16:24:18Z",
      "result": "0 entries returned.",
      "log_explorer_url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22CRM-conversations%22%0AjsonPayload.message%3D~%22.%2B%22%0Aseverity%3DERROR;timeRange=2026-11-16T16%3A14%3A18Z%2F2026-11-16T16%3A24%3A18Z?project=highlevel-backend"
    },
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"CRM-conversations\" AND jsonPayload.message=~\".+\" AND severity=WARNING; 2026-11-16T16:14:18Z–2026-11-16T16:24:18Z",
      "result": "0 entries returned.",
      "log_explorer_url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22CRM-conversations%22%0AjsonPayload.message%3D~%22.%2B%22%0Aseverity%3DWARNING;timeRange=2026-11-16T16%3A14%3A18Z%2F2026-11-16T16%3A24%3A18Z?project=highlevel-backend"
    }
  ],
  "metric_urls": [
    "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-nodeId=9z5rx125RDGGUp_mrV6Rvw&from=1794844158000&to=1794847758000&viewPanel=66"
  ],
  "limitations": [
    "No alert-window OpenSearch diagnostics were available.",
    "Kubernetes verification failed due to context mismatch and TLS certificate verification failure.",
    "Hourly metrics cannot validate the reported 10-minute threshold."
  ]
}

## Evidence

### Dashboards (incident window)
- [CPU utilization — conversations-search node 9z5rx125RDGGUp_mrV6Rvw (alert window)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-nodeId=9z5rx125RDGGUp_mrV6Rvw&from=1794844158000&to=1794847758000&viewPanel=66)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22CRM-conversations%22%0AjsonPayload.message%3D~%22.%2B%22%0Aseverity%3DERROR;timeRange=2026-11-16T16%3A14%3A18Z%2F2026-11-16T16%3A24%3A18Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22CRM-conversations%22%0AjsonPayload.message%3D~%22.%2B%22%0Aseverity%3DWARNING;timeRange=2026-11-16T16%3A14%3A18Z%2F2026-11-16T16%3A24%3A18Z?project=highlevel-backend)

### Related incidents
- [Current node alert and responder invitations — 16 Nov](https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794845958933149?thread_ts=1794845958.933149)
- [Earlier same-day CPU and latency alert — 16 Nov](https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794839055479519?thread_ts=1794839055.479519)

## Investigation Timeline

**Deep dive passes**: 2
**Worker calls**: 12

**Total worker time**: 847.2s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 17.3s | ✓ |
| grafana-worker | worker | 92.0s | ✓ |
| kubectl-worker | worker | 106.1s | ✓ |
| log-explorer-worker | worker | 383.7s | ✓ |
| slack-worker | worker | 75.6s | ✓ |
| triage | worker | 10.5s | ✓ |
| triage_llm_ranking | decision | 19.4s | ✓ |
| deep_dive_pass_1 | worker | 59.1s | ✓ |
| follow_up_check | decision | 9.5s | ✓ |
| deep_dive_pass_2 | worker | 46.5s | ✓ |
| follow_up_check | decision | 12.4s | ✓ |
| cross_validation | worker | 15.3s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IKYNHFHF31DVF|#140015 AWS ES Data Node CPU Utilization>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#cpu|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having high CPU usage. Please check the details below, Cluster: conversations-search Node: 9z5rx125RDGGUp_mrV6Rvw CPU Utilization: 78.3937900684669% Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=66&amp;var-domainName=conversations-search&amp;var-nodeId=9z5rx125RDGGUp_mrV6Rvw> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IKYNHFHF31DVF|#140015 AWS ES Data Node CPU Utilization>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#cpu|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having high CPU usage. Please check the details below, Cluster: conversations-search Node: 9z5rx125RDGGUp_mrV6Rvw CPU Utilization: 78.3937900684669% Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=66&amp;var-domainName=conversations-search&amp;var-nodeId=9z5rx125RDGGUp_mrV6Rvw>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-16T16:33:29.593Z*