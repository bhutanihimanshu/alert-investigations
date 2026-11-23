# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-23-0c6029eb
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: web-users-servers-usc1-prd-cluster
**Channel**: slack
**Detected**: 2026-11-23T17:07:47.387Z
**Status**: delivering

## Triage

**Primary Anomaly**: error_spike
**Details**: conversations-reporting-api recorded 2,413 ERROR logs at 22:30 IST; 48/50 sampled entries concerned OpenSearch search failures, with duplicate logging. Agent-aggregation latency peaked at 12.075s p95, and route 5xx reached 4.082% at 22:32 IST. Traffic and fleet-average resources stayed within sampled historical ranges; no restarts were found. No historical RCA is confirmed: prior Firestore incidents concern different routes, while the historical OpenSearch query-limit failure is only a diagnostic lead.
**Anomalous**: Yes
**Deep Dive Direction**: YES — the OpenSearch direction is correct: search-pool saturation caused failures, and reporting workers share the impaired dependency. The initiating cause remains unresolved.

Need caller/route/retry attribution for the 22:31 search burst, plus finer-grained per-node EBS throttling and shard service-time data to distinguish demand amplification from storage bottlenecks.

## Deep Dive Findings

{
  "summary": "Confirmed OpenSearch search-pool saturation caused agent-aggregation failures around 22:30 IST. Reporting-worker timeouts and write rejections establish a shared downstream bottleneck. EBS throughput pressure is corroborated background context, but its causal role in both latency episodes remains unproven.",
  "confidence": {
    "immediate_search_failure": "high",
    "shared_reporting_dependency": "high",
    "ebs_as_initiating_cause": "low"
  },
  "dependency": {
    "domain_investigated": "conversations-reporting-prod",
    "domain_mapping_limit": "Slack identifies this domain, and its metrics corroborate the incident. Live endpoint hostname verification was blocked by Kubernetes secret-read permissions.",
    "index_pattern": "email-messages-*",
    "evidence": "Search logs identify this pattern. Agent aggregation executes two concurrent, location-routed wildcard searches for current and comparison periods.",
    "backlog_connection": "crm-conversations-reporting-events-sub is consumed by the reporting worker, which imports the same esReportingClient and uses the same configured endpoint. Failed bulk items are excluded from successful acknowledgements."
  },
  "findings": [
    {
      "time_ist": "2026-11-23 22:30:59.781",
      "finding": "Route-specific search rejection confirmed.",
      "nested_reason": "search_phase_execution_exception → rejected_execution_exception",
      "stack": "ConversationsReportingService.getEmailAggregateData → ConversationsReportingController.getEmailAggregateStats",
      "node": "bb766deb41ffc7b6dd9202ab95d50aac",
      "pool": "search",
      "active_threads": 13,
      "pool_size": 13,
      "queued_tasks": 1002,
      "queue_capacity": 1000,
      "interpretation": "This failure is capacity rejection, not a demonstrated malformed-query or query-limit error."
    },
    {
      "window_ist": "22:30–22:33",
      "finding": "Search demand and rejection counters rose together.",
      "domain_search_rate": {
        "22:29": 73420,
        "22:31": 248709,
        "22:32": 79899,
        "unit": "sum of per-node SearchRate, operations/minute",
        "caveat": "Shard-level operations, not unique API requests; caller demand versus retry amplification remains unresolved."
      },
      "search_rejection_delta": {
        "count": 40659,
        "nodes": 11,
        "counter_bounds_ist": "22:29–22:33"
      },
      "maximum_search_queue": 1006,
      "maximum_node_search_latency_ms": 114.987,
      "maximum_node_indexing_latency_ms": 14.01,
      "maximum_write_queue": 16615,
      "write_rejection_delta": 22631,
      "correlation": "Matches supplied route 5xx peak at 22:32 and latency elevation at 22:32–22:33. Rejection counters count tasks, not failed requests."
    },
    {
      "window_ist": "22:53–22:56",
      "finding": "Queueing persisted during the second latency episode.",
      "maximum_node_search_latency_ms": 99.626,
      "maximum_node_indexing_latency_ms": 9.62,
      "maximum_search_queue": 478,
      "maximum_write_queue": 14496,
      "search_rejection_delta_between_22_53_and_22_56": 0,
      "write_rejection_delta_between_22_53_and_22_56": 14402,
      "interpretation": "The later episode does not show the same search-rejection burst. Slow processing and queueing remain supported contributors."
    },
    {
      "finding": "Disk metrics do not establish EBS throttling as the immediate trigger.",
      "first_window_maximum_disk_latency_ms": {
        "read": 0.762,
        "write": 0.938
      },
      "second_window_maximum_disk_latency_ms": {
        "read": 0.785,
        "write": 0.912
      },
      "throughput_throttle": "Positive samples preceded the first spike at 22:28–22:29; samples at 22:32–22:34 were zero. Positive samples occurred on some nodes at 22:52–22:54.",
      "limitations": "Disk/throttle series were sparse, approximately five-minute samples despite requesting one-minute periods. The persistently slow node fb3A8SuER8-GIDvtq5IwFA had a zero throttle sample at 22:53. DiskQueueDepth returned no usable data.",
      "slack_context": "AWS's quoted diagnosis identifies volume and instance EBS limits for earlier write throttling; it is not incident-specific proof."
    },
    {
      "finding": "Concurrent reporting backlog shares an impaired OpenSearch dependency.",
      "first_window": "Five sampled worker errors at 22:32:48–22:32:50 were OpenSearch Request timed out failures.",
      "second_window": "At 22:54:53, worker logs explicitly show write-pool rejections on node 817397e44e2bfe92700e08758331ccf9: all eight threads active, 10,000–10,320 queued tasks against capacity 10,000.",
      "interpretation": "Logs plus write-queue metrics establish processing failures that can sustain backlog. Publisher contribution and the entire backlog's origin were not quantified."
    }
  ],
  "next_actions": [
    "Attribute the 22:31 domain search burst to routes, callers and retries before assigning its initiating cause.",
    "Inspect shard-level service time and EBS limits on the persistently slow node; do not infer that increasing search queues fixes the bottleneck.",
    "Evaluate AWS's instance-size or node-count recommendations against current throughput and shard distribution; no production changes were made."
  ],
  "log_queries_used": {
    "project": "highlevel-backend",
    "limit_per_query": 5,
    "queries": [
      "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-reporting-api\" resource.labels.cluster_name=\"web-users-servers-usc1-prd-cluster\" severity>=ERROR jsonPayload.message=\"PLATFORM_CORE_01_ESCLIENT_SEARCH_ERROR:\" timestamp>=\"2026-11-23T16:59:00Z\" timestamp<=\"2026-11-23T17:03:00Z\"",
      "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-reporting-api\" severity>=ERROR jsonPayload.message:\"search_phase_execution_exception\" timestamp>=\"2026-11-23T17:00:00Z\" timestamp<=\"2026-11-23T17:01:00Z\"",
      "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-reporting-api\" severity>=ERROR jsonPayload.message:\"getEmailAggregateData\" timestamp>=\"2026-11-23T17:00:00Z\" timestamp<=\"2026-11-23T17:04:00Z\"",
      "resource.type=\"k8s_container\" resource.labels.container_name=\"crm-conversations-reporting-worker\" severity>=ERROR timestamp>=\"2026-11-23T17:00:00Z\" timestamp<=\"2026-11-23T17:03:00Z\"",
      "resource.type=\"k8s_container\" resource.labels.container_name=\"crm-conversations-reporting-worker\" severity>=ERROR jsonPayload.message:\"rejected\" timestamp>=\"2026-11-23T17:23:00Z\" timestamp<=\"2026-11-23T17:26:00Z\""
    ]
  },
  "sources": {
    "agent_failure": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-reporting-api%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3A%22getEmailAggregateData%22;timeRange=2026-11-23T17%3A00%3A00Z%2F2026-11-23T17%3A04%3A00Z?project=highlevel-backend",
    "worker_rejections": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-reporting-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3A%22rejected%22;timeRange=2026-11-23T17%3A23%3A00Z%2F2026-11-23T17%3A26%3A00Z?project=highlevel-backend",
    "domain_metrics": "https://prod.grafana.leadconnectorhq.com/d/aws-es-high-cpu-node?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-reporting-prod&var-clientId=696084496265&var-nodeId=All&from=1795451400000&to=1795455300000",
    "aws_discussion": "https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795412903681709?thread_ts=1795115427.534199"
  },
  "access": "Validated existing Grafana and Slack sessions; read-only CloudWatch queries through Grafana, gcloud logs, Kubernetes configuration and local source inspection."
}

## Evidence

### Dashboards (incident window)
- [P95 latency — conversations-reporting-api agent aggregation; repeated elevation](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795451867000&to=1795455467000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-reporting-api&var-method=POST&var-route=%2Fconversations-reporting%2Femails%2Faggregate%2Fagent&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&viewPanel=17)
- [Traffic by response status — conversations-reporting-api agent aggregation; brief 5xx spike](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795451867000&to=1795455467000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-reporting-api&var-method=POST&var-route=%2Fconversations-reporting%2Femails%2Faggregate%2Fagent&var-pod_name=All&viewPanel=1)
- [Grafana dashboard](https://prod.grafana.leadconnectorhq.com/d/aws-es-high-cpu-node?orgId=1&var-datasource=4PeY_YyVz&var-region=default&var-domainName=conversations-reporting-prod&var-clientId=696084496265&var-nodeId=All&from=1795451400000&to=1795455300000)

### Related incidents
- [Concurrent reporting backlog and suspected OpenSearch slowdown — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795451063168569)
- [Team shares AWS diagnosis of OpenSearch disk throughput limits — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795412903681709?thread_ts=1795115427.534199)
- [Export change reverted to investigate increased disk reads — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795444258397449?thread_ts=1795115427.534199)

## Investigation Timeline

**Deep dive passes**: 2
**Worker calls**: 12

**Total worker time**: 1039.2s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 16.3s | ✓ |
| grafana-worker | worker | 109.3s | ✓ |
| kubectl-worker | worker | 108.2s | ✓ |
| log-explorer-worker | worker | 279.7s | ✓ |
| slack-worker | worker | 63.0s | ✓ |
| triage | worker | 15.5s | ✓ |
| triage_llm_ranking | decision | 42.0s | ✓ |
| deep_dive_pass_1 | worker | 362.8s | ✓ |
| follow_up_check | decision | 14.5s | ✓ |
| deep_dive_pass_2 | worker | 9.0s | ✗ Codex exited 1: here you looked.

## Output Format
Return a JSON object with your deep dive findings, log queries used, and any log explorer URLs.
## Lessons from Past Investigations

• For HeapOutOfMemory alerts: resolve the workload to its source repo (via workload_repos table), find the worker entry point, and analyze the message processing code for memory patterns — unbounded Promise.all(), in-memory buffering, large payload accumulation, flowControl/maxMessages config. Do NOT rely solely on log error correlation (e.g. ECONNREFUSED) without verifying the code path actually causes heap pressure.

2026-11-23T17:58:18.532984Z ERROR rmcp::transport::worker: worker quit with fatal: Transport channel closed, when AuthRequired(AuthRequiredError { www_authenticate_header: "Bearer realm=\"OAuth\", resource_metadata=\"https://mcp.clickhouse.cloud/.well-known/oauth-protected-resource/clickstack\"" })
2026-11-23T17:58:19.046912Z ERROR rmcp::transport::worker: worker quit with fatal: Transport channel closed, when AuthRequired(AuthRequiredError { www_authenticate_header: "Bearer realm=\"OAuth\", resource_metadata=\"https://mcp.clickhouse.cloud/.well-known/oauth-protected-resource/clickstack\"" })
2026-11-23T17:58:19.181066Z ERROR rmcp::transport::worker: worker quit with fatal: Transport channel closed, when AuthRequired(AuthRequiredError { www_authenticate_header: "Bearer realm=\"MCP Server\", error=\"invalid_token\", error_description=\"Bearer token required\", resource_metadata=\"https://mcp.clickup.com/.well-known/oauth-protected-resource/mcp\"" })
hook: SessionStart
hook: SessionStart
hook: SessionStart Completed
hook: SessionStart Completed
hook: UserPromptSubmit
hook: UserPromptSubmit
hook: UserPromptSubmit Completed
hook: UserPromptSubmit Completed
ERROR: You hit your spend cap set by the owner of your workspace. Ask an owner to increase your spend cap to continue.
ERROR: You hit your spend cap set by the owner of your workspace. Ask an owner to increase your spend cap to continue.
 |
| follow_up_check | decision | 10.1s | ✗ Codex exited 1: a and Slack sessions; read-only CloudWatch queries through Grafana, gcloud logs, Kubernetes configuration and local source inspection.\"\n}","logQueries":[],"logExplorerUrls":[]},"passCount":2}

Question:
Based on the deep dive findings:
1. Is the triage direction still correct, or does the evidence point elsewhere?
2. Do you need more data to determine the root cause?

Answer YES if you need more data (and specify what — e.g., source code, past incidents, dependency graph).
Answer NO if you have enough evidence.
If the direction was wrong, say YES and specify the correct direction.

2026-11-23T17:58:27.543249Z ERROR rmcp::transport::worker: worker quit with fatal: Transport channel closed, when AuthRequired(AuthRequiredError { www_authenticate_header: "Bearer realm=\"OAuth\", resource_metadata=\"https://mcp.clickhouse.cloud/.well-known/oauth-protected-resource/clickstack\"" })
2026-11-23T17:58:28.144662Z ERROR rmcp::transport::worker: worker quit with fatal: Transport channel closed, when AuthRequired(AuthRequiredError { www_authenticate_header: "Bearer realm=\"OAuth\", resource_metadata=\"https://mcp.clickhouse.cloud/.well-known/oauth-protected-resource/clickstack\"" })
2026-11-23T17:58:28.203985Z ERROR rmcp::transport::worker: worker quit with fatal: Transport channel closed, when AuthRequired(AuthRequiredError { www_authenticate_header: "Bearer realm=\"MCP Server\", error=\"invalid_token\", error_description=\"Bearer token required\", resource_metadata=\"https://mcp.clickup.com/.well-known/oauth-protected-resource/mcp\"" })
hook: SessionStart
hook: SessionStart
hook: SessionStart Completed
hook: SessionStart Completed
hook: UserPromptSubmit
hook: UserPromptSubmit
hook: UserPromptSubmit Completed
hook: UserPromptSubmit Completed
ERROR: You hit your spend cap set by the owner of your workspace. Ask an owner to increase your spend cap to continue.
ERROR: You hit your spend cap set by the owner of your workspace. Ask an owner to increase your spend cap to continue.
 |
| cross_validation | worker | 8.7s | ✗ Codex exited 1: in #dev or #jenkins-alerts for 20–23 Nov. The rollback is reported in the incident discussion.\",\n  \"confidence\": \"Low for root-cause attribution: Slack context only; metrics, logs and deployment state were not independently verified.\",\n  \"access\": \"Existing Slack session validated with auth.test; read-only queries only.\"\n}"
    }
  }
}

## Output Format (JSON only)
```json
{
  "sources": [{ "name": "...", "finding": "...", "agrees": true/false }],
  "confidence": "high|medium|low",
  "discrepancies": ["..."],
  "rootCauseSummary": "...",
  "rootCauseCategory": "..."
}
```
2026-11-23T17:58:56.156564Z ERROR rmcp::transport::worker: worker quit with fatal: Transport channel closed, when AuthRequired(AuthRequiredError { www_authenticate_header: "Bearer realm=\"OAuth\", resource_metadata=\"https://mcp.clickhouse.cloud/.well-known/oauth-protected-resource/clickstack\"" })
2026-11-23T17:58:56.870366Z ERROR rmcp::transport::worker: worker quit with fatal: Transport channel closed, when AuthRequired(AuthRequiredError { www_authenticate_header: "Bearer realm=\"OAuth\", resource_metadata=\"https://mcp.clickhouse.cloud/.well-known/oauth-protected-resource/clickstack\"" })
2026-11-23T17:58:56.979304Z ERROR rmcp::transport::worker: worker quit with fatal: Transport channel closed, when AuthRequired(AuthRequiredError { www_authenticate_header: "Bearer realm=\"MCP Server\", error=\"invalid_token\", error_description=\"Bearer token required\", resource_metadata=\"https://mcp.clickup.com/.well-known/oauth-protected-resource/mcp\"" })
hook: SessionStart
hook: SessionStart
hook: SessionStart Completed
hook: SessionStart Completed
hook: UserPromptSubmit
hook: UserPromptSubmit
hook: UserPromptSubmit Completed
hook: UserPromptSubmit Completed
ERROR: You hit your spend cap set by the owner of your workspace. Ask an owner to increase your spend cap to continue.
ERROR: You hit your spend cap set by the owner of your workspace. Ask an owner to increase your spend cap to continue.
 |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IHF693VKVKTCM|#140757 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&alert_id=1891525268083367598|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-reporting-api` route `/conversations-reporting/emails/aggregate/agent` [POST] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 10.82s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&from=now-30m&to=now&var-datasource=ber8nnhvgsjy8f&var-container=conversations-reporting-api&var-method=POST&var-route=/conversations-reporting/emails/aggregate/agent&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&var-latencyThreshold=5|here> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IHF693VKVKTCM|#140757 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&amp;alert_id=1891525268083367598|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-reporting-api` route `/conversations-reporting/emails/aggregate/agent` [POST] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 10.82s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-30m&amp;to=now&amp;var-datasource=ber8nnhvgsjy8f&amp;var-container=conversations-reporting-api&amp;var-method=POST&amp;var-route=/conversations-reporting/emails/aggregate/agent&amp;var-pod_name=All&amp;var-minLatency=0&amp;var-maxLatency=100000&amp;var-latencyThreshold=5|here>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-23T17:59:04.185Z*