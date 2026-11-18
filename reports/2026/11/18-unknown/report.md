# Alert Investigation: HighCPU

**Investigation ID**: inv-2026-11-18-716b49db
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: conversations-search
**Channel**: slack
**Detected**: 2026-11-18T16:12:16.200Z
**Status**: delivering

## Root Cause

**Summary**: Root cause remains undetermined. Recurring shared load peaks coincide with additional node-specific CPU consumption, but indexing, expensive queries, merges, retries and configuration changes remain unproven. Restore diagnostic access to identify which operations account for the excess CPU.
**Confidence**: low
**Category**: unknown

## Triage

**Primary Anomaly**: cpu_high
**Details**: AWS OpenSearch conversations-search node 4ewQ7roITImJo_hiDB41Hg peaked at 80.59% CPU and remained 17–22 percentage points hotter than peers during recurring fleet-wide peaks. Search/indexing latency increased, while JVM pressure, cluster health and sampled 5xx counts showed no corresponding deterioration. Historical indexing-pressure incidents are partial matches only: current evidence does not establish backfill, retry amplification, write-pool saturation or an expensive query. GCP queries targeted the unverified CRM-conversations container identity and provide no diagnostic evidence.
**Anomalous**: Yes
**Deep Dive Direction**: YES — The triage direction remains correct: investigate node-local CPU overhead during shared load peaks. Root cause remains unknown because node diagnostics are inaccessible.

Need simultaneous affected-node and peer hot threads, active tasks, and two per-shard indexing/search/merge statistics snapshots 30–60 seconds apart during a confirmed CPU peak, through working OpenSearch access.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | CPU peaked at 80.59%, with recurring 17–22 percentage-point excess over peers and elevated latency. Metrics establish the anomaly, but not its cause. | ✓ |
| Slack | Thread analysis reports periodic indexing bursts but explicitly leaves unequal node CPU unexplained. Reported metrics may repeat the same underlying telemetry and are not independent causal confirmation. | ✓ |
| OpenSearch deep dive | Endpoint connectivity failed before authentication; no hot threads, tasks or shard-statistics deltas were collected. This is an investigation access blocker, not evidence of a production outage. | ✓ |
| GCP logs | Empty results used an unverified CRM-conversations container identity and cannot corroborate or exclude causes on the AWS OpenSearch node. | ✗ |
| Kubernetes | No applicable pod or deployment was identified; unavailable checks provide no independent causal evidence. | ✗ |

### Discrepancies
- Indexing bursts explain shared load timing only; affected-node operation rates below peer medians weaken a simple volume-based explanation for its excess CPU.
- Five-minute incident values and hourly baseline values have different resolutions, limiting peak comparisons.
- Manual alert resolution does not establish CPU recovery; subsequent sampled CPU remained elevated against baseline.
- Normal sampled JVM pressure, cluster health and 5xx counts weaken memory-pressure or availability-failure hypotheses but do not exclude latency impact.
- No mechanism has two independent supporting sources. Falsification requires simultaneous affected-node and peer hot threads, active tasks and two shard-statistics snapshots 30–60 seconds apart during a CPU peak.

## Deep Dive Findings

{
  "status": "blocked",
  "root_cause": null,
  "confidence": "low",
  "summary": "OpenSearch node diagnostics remain inaccessible. This pass found no new evidence explaining the affected node's CPU excess.",
  "findings": [
    {
      "finding": "OpenSearch connectivity still fails before authentication.",
      "verified_at_ist": "2026-11-18 23:04:58",
      "evidence": "HTTPS attempts through DNS and the historically documented pinned IP both timed out before TCP/TLS establishment. Authentik returned HTTP 200.",
      "interpretation": "The endpoint is unreachable from this executor; this does not establish an IAM denial, OpenSearch outage, or specific network cause."
    },
    {
      "finding": "No usable browser access path was established.",
      "evidence": "Native browser tools were unavailable. Chrome inspection through AppleScript failed with an XPC connection error. No listener was found on local debugging port 9222."
    },
    {
      "finding": "Recent investigation history does not establish a matching cause.",
      "evidence": "Read-only SQLite lookup found two November 16 HighCPU investigations with undetermined mechanisms; neither confirmed indexing, backfill, fielddata, deployment, or shard activity as causal."
    }
  ],
  "diagnostics_collected": {
    "simultaneous_affected_and_peer_hot_threads": false,
    "active_tasks": false,
    "two_per_shard_statistics_snapshots": false,
    "indexing_search_merge_time_deltas": null
  },
  "limitations": [
    "CPU peaks and peer imbalance are supplied prior-pass evidence, not newly measured in this pass.",
    "Historical SAML access guidance was used to locate the endpoint; authenticated access could not be verified.",
    "No diagnostic logs were retrieved. The unverified CRM-conversations GCP container filter was not repeated."
  ],
  "next_action": "Restore a reachable authenticated OpenSearch session, then capture affected-node and peer hot threads plus active tasks concurrently, and per-shard indexing/search/merge statistics twice, 30–60 seconds apart, during a confirmed CPU peak.",
  "log_queries_used": [],
  "log_explorer_urls": [],
  "production_changes": false,
  "messages_posted": false
}

## Evidence

### Dashboards (incident window)
- [CPU — conversations-search: affected node spikes above peers](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=All&from=1795016520000&to=1795022880000&viewPanel=66)
- [JVM memory — conversations-search: affected node has no sustained pressure](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=4ewQ7roITImJo_hiDB41Hg&from=1795016520000&to=1795022880000&viewPanel=70)
- [Search latency — conversations-search: affected node latency elevated](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=4ewQ7roITImJo_hiDB41Hg&from=1795016520000&to=1795022880000&viewPanel=71)
- [HTTP responses — conversations-search: no corresponding 5xx surge](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=All&from=1795016520000&to=1795022880000&viewPanel=34)

### Related incidents
- [Alert marked resolved by Balaji; hotter-node cause remains open — 18 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795022688874479)
- [Second conversations-search node CPU alert — 18 Nov, 22:29 IST](https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1795021154294759)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 685.5s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 12.4s | ✓ |
| grafana-worker | worker | 100.4s | ✓ |
| kubectl-worker | worker | 16.5s | ✓ |
| log-explorer-worker | worker | 55.1s | ✓ |
| slack-worker | worker | 77.6s | ✓ |
| triage | worker | 15.2s | ✓ |
| triage_llm_ranking | decision | 25.4s | ✓ |
| deep_dive_pass_1 | worker | 197.3s | ✓ |
| follow_up_check | decision | 11.9s | ✓ |
| deep_dive_pass_2 | worker | 85.1s | ✓ |
| follow_up_check | decision | 12.7s | ✓ |
| deep_dive_pass_3 | worker | 51.8s | ✓ |
| cross_validation | worker | 24.1s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IRP5N6Y2T4Z39|#140215 AWS ES Data Node CPU Utilization>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#cpu|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having high CPU usage. Please check the details below, Cluster: conversations-search Node: 4ewQ7roITImJo_hiDB41Hg CPU Utilization: 82.32925849303308% Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=66&amp;var-domainName=conversations-search&amp;var-nodeId=4ewQ7roITImJo_hiDB41Hg> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IRP5N6Y2T4Z39|#140215 AWS ES Data Node CPU Utilization>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#cpu|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having high CPU usage. Please check the details below, Cluster: conversations-search Node: 4ewQ7roITImJo_hiDB41Hg CPU Utilization: 82.32925849303308% Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=66&amp;var-domainName=conversations-search&amp;var-nodeId=4ewQ7roITImJo_hiDB41Hg>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-18T17:35:49.735Z*