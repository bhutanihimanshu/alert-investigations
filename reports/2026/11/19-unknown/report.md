# Alert Investigation: HighCPU

**Investigation ID**: inv-2026-11-19-0a4c0308
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: conversations-search
**Channel**: slack
**Detected**: 2026-11-19T17:14:15.006Z
**Status**: delivering

## Root Cause

**Summary**: A transient OpenSearch node CPU hotspot with elevated search latency is supported by metrics; its cause remains unresolved. Pre-existing enrichment warnings and opportunity resyncs do not establish causation. No observed rejection spike, sustained JVM pressure or shard relocation supports those alternative mechanisms. Attribution requires incident-time node/shard diagnostics correlated with publisher or query activity.
**Confidence**: low
**Category**: undetermined

## Triage

**Primary Anomaly**: cpu_high
**Details**: AWS OpenSearch conversations-search node Q0X2yqGxS7WwDPpkM9g4uQ exceeded 70% CPU during 22:32–22:44 IST, peaking at 86.91%. CPU fell to 64.34% by 22:49 but remained above peers, with elevated search latency despite below-peer latest search/indexing rates. No sustained JVM pressure, shard relocation, new write/search rejections or observed 5xx. Historical indexing-load and query-efficiency RCAs are plausible hypotheses, not confirmed matches; the backfill rejection-loop signature is absent. Empty GCP results used an unverified container identity and do not rule out application errors.
**Anomalous**: Yes
**Deep Dive Direction**: YES — The triage direction remains correct: increased publisher volume likely contributed to the indexing/CPU burst, but the business trigger and pre-existing node imbalance remain unresolved.

More data needed:
- Incident-time hot threads, running tasks, and per-shard indexing/search/merge deltas.
- Publisher attribution by tenant, action, and identity to distinguish business events from application republishes.
- Deployed search-serving library version and live `_id` fielddata to assess historical-defect recurrence.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | Confirms node CPU peaked at 86.91% with elevated search latency. Latest CPU fell below threshold but remained above peers despite lower request rates; the responsible workload is unidentified. | ✓ |
| Slack | Confirms the alert notification, but provides no human diagnosis or independently verified causal deployment. The database-channel notification duplicates the same alert. | ✓ |
| Deep-dive logs and code | Opportunity-enrichment failures predate the spike and are caught without failing conversation writes. Observed resyncs concern OPPORTUNITY; neither establishes the conversations-search CPU mechanism. | ✗ |
| Layer 1 GCP logs | Empty results used an unverified container identity and cannot establish application health or exclude errors. | ✗ |
| Kubernetes | No pod mapping exists for the supplied AWS node in the retrieved evidence; Kubernetes checks provide no causal corroboration. | ✗ |

### Discrepancies
- No proposed root cause is supported by two independent sources tracing trigger through impact to CPU elevation.
- Publisher-volume contribution is asserted as plausible, but the supplied findings contain no quantified publisher timeline or attributable business trigger.
- Below-peer latest request rates weaken a simple node-volume explanation, but post-peak measurements cannot disprove incident-time overload.
- Seven-day hourly baseline maxima are not directly comparable with one-minute incident peaks.
- Historical sorting-fix history does not establish the deployed version or live fielddata; incident-time hot threads, shard deltas and deployment verification remain unavailable.

## Deep Dive Findings

{
  "status": "partial_root_cause",
  "summary": "Publisher-volume contribution remains plausible; the business trigger and node-specific CPU imbalance remain unresolved. This pass found pre-existing worker warnings but no evidence establishing them as the CPU trigger.",
  "confidence": "low for underlying root cause",
  "new_findings": [
    {
      "finding": "Opportunity-seeding connection failures predate the CPU burst.",
      "evidence": "Five sampled warnings at 22:34:59 IST report 'No Living connections'; the same warning appears in five samples at 22:04:59 IST, before the 22:32 CPU threshold crossing.",
      "code_check": "conversation-opportunity-seed.ts reads Index.Contact, catches the failure, logs a warning and returns undefined without failing the conversation write.",
      "interpretation": "This is a real degraded enrichment path, but neither its timing nor the inspected code establishes causation for conversations-search CPU. Sample counts do not measure error rates."
    },
    {
      "finding": "Resync activity was observed, but conversation resync was not established.",
      "evidence": "Five sampled resync-start logs during 22:30–22:35 IST all identify model OPPORTUNITY. A targeted CONVERSATION query across 22:25–22:45 IST returned no entries.",
      "interpretation": "Local code supports publishing conversation resync events, but the observed opportunity resyncs cannot be attributed as the conversation-topic burst. Empty results do not exclude other publishers or unlogged republishes."
    },
    {
      "finding": "Publisher identity remains unavailable.",
      "evidence": "The topic-scoped Publisher.Publish audit-log query returned no entries.",
      "interpretation": "No tenant, action or publishing principal can be assigned from this result; it does not mean no publishes occurred."
    },
    {
      "finding": "Historical _id defect recurrence remains unverified.",
      "evidence": "Git history confirms marketplace-backend commit 5b1cf2842de pinned crm-common-backend-libs 0.5.0. Historical notes identify that release with the sorting fix.",
      "limitation": "This does not verify the deployed search-serving version or live _id fielddata."
    }
  ],
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-elastic-indexing-worker\" severity>=ERROR timestamp>=\"2026-11-19T17:00:00Z\" timestamp<\"2026-11-19T17:05:00Z\"",
      "limit": 10,
      "result": "No entries."
    },
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-elastic-indexing-worker\" severity=WARNING jsonPayload.message:\"Failed to resolve opportunities\"",
      "windows_utc": [
        "2026-11-19T17:00:00Z/2026-11-19T17:05:00Z",
        "2026-11-19T16:30:00Z/2026-11-19T16:35:00Z"
      ],
      "limit_per_window": 5,
      "result": "Both samples contain No Living connections."
    },
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"elastic-indexing-resync-worker\" jsonPayload.message:\"Resyncing ES Objects\" jsonPayload.model=\"CONVERSATION\" timestamp>=\"2026-11-19T16:55:00Z\" timestamp<\"2026-11-19T17:15:00Z\"",
      "limit": 5,
      "result": "No entries."
    },
    {
      "query": "protoPayload.serviceName=\"pubsub.googleapis.com\" protoPayload.methodName=\"google.pubsub.v1.Publisher.Publish\" resource.labels.topic_id=\"elastic-indexing-conversations-events\" timestamp>=\"2026-11-19T17:00:00Z\" timestamp<\"2026-11-19T17:05:00Z\"",
      "limit": 5,
      "result": "No entries."
    }
  ],
  "log_explorer_urls": [
    "https://console.cloud.google.com/logs/query;query=resource.labels.container_name%3D%22conversations-elastic-indexing-worker%22%20severity%3DWARNING%20jsonPayload.message%3A%22Failed%20to%20resolve%20opportunities%22;timeRange=2026-11-19T17%3A00%3A00Z%2F2026-11-19T17%3A05%3A00Z?project=highlevel-backend",
    "https://console.cloud.google.com/logs/query;query=resource.labels.container_name%3D%22elastic-indexing-resync-worker%22%20jsonPayload.message%3A%22Resyncing%20ES%20Objects%22%20jsonPayload.model%3D%22CONVERSATION%22;timeRange=2026-11-19T16%3A55%3A00Z%2F2026-11-19T17%3A15%3A00Z?project=highlevel-backend"
  ],
  "limitations": [
    "Existing gcloud authentication succeeded; stored Grafana session validation returned HTTP 401.",
    "No AWS CLI, ~/.aws configuration or native OpenSearch connector was available. Hot threads, tasks, shard deltas and fielddata were not retrieved.",
    "Subagent launch failed. No production changes, messages or additional orchestrator investigation were made."
  ],
  "next_evidence_needed": [
    "Incident-time node diagnostics or retained profiling evidence identifying expensive shards, searches or merges.",
    "Publisher telemetry carrying tenant, action, event identity and publish identity.",
    "Deployed search-serving artifact version and authenticated live fielddata statistics."
  ]
}

## Evidence

### Dashboards (incident window)
- [CPU — conversations-search nodes: affected node peaks at 86.91%](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=All&from=1795106654000&to=1795110254000&viewPanel=66)
- [Search latency — conversations-search: affected node elevated versus peers](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=All&from=1795106654000&to=1795110254000&viewPanel=71)
- [JVM memory — conversations-search: no sustained pressure increase](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=Q0X2yqGxS7WwDPpkM9g4uQ&from=1795106654000&to=1795110254000&viewPanel=70)
- [HTTP responses — conversations-search: zero observed incident 5xx](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=All&from=1795106654000&to=1795110254000&viewPanel=34)

### Related incidents
- [Production Conversations server deployment succeeded — 19 Nov](https://gohighlevel.slack.com/archives/G01FL9T6UGL/p1795084362866119)
- [Inbound SMS workers deployed and sanity passed — 19 Nov](https://gohighlevel.slack.com/archives/C03FV5D6PAQ/p1794395529627949)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 927.7s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 12.4s | ✓ |
| grafana-worker | worker | 114.7s | ✓ |
| kubectl-worker | worker | 29.0s | ✓ |
| log-explorer-worker | worker | 53.9s | ✓ |
| slack-worker | worker | 89.0s | ✓ |
| triage | worker | 15.4s | ✓ |
| triage_llm_ranking | decision | 24.7s | ✓ |
| deep_dive_pass_1 | worker | 218.9s | ✓ |
| follow_up_check | decision | 47.1s | ✓ |
| deep_dive_pass_2 | worker | 159.7s | ✓ |
| follow_up_check | decision | 12.4s | ✓ |
| deep_dive_pass_3 | worker | 132.0s | ✓ |
| cross_validation | worker | 18.4s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IHYKPY42DSLRQ|#140329 AWS ES Data Node CPU Utilization>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#cpu|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having high CPU usage. Please check the details below, Cluster: conversations-search Node: Q0X2yqGxS7WwDPpkM9g4uQ CPU Utilization: 77.53903298942177% Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=66&amp;var-domainName=conversations-search&amp;var-nodeId=Q0X2yqGxS7WwDPpkM9g4uQ> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IHYKPY42DSLRQ|#140329 AWS ES Data Node CPU Utilization>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#cpu|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having high CPU usage. Please check the details below, Cluster: conversations-search Node: Q0X2yqGxS7WwDPpkM9g4uQ CPU Utilization: 77.53903298942177% Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=66&amp;var-domainName=conversations-search&amp;var-nodeId=Q0X2yqGxS7WwDPpkM9g4uQ>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-19T17:31:48.438Z*