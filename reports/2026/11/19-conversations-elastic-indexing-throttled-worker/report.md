# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-19-ac6b72e2
**Workload**: conversations-elastic-indexing-throttled-worker
**Container**: conversations-elastic-indexing-throttled-worker
**Cluster**: workers-us-central-production-cluster
**Channel**: slack
**Detected**: 2026-11-19T10:07:48.629Z
**Status**: delivering

## Root Cause

**Summary**: Root cause remains unconfirmed. Elevated CPU and throttling occurred on a worker constrained to one replica with a substantial backlog. Incoming indexing demand exceeding that capacity is plausible, but no two independent sources establish the initiating trigger or connect lookup failures causally to CPU usage.
**Confidence**: low
**Category**: undetermined

## Triage

**Primary Anomaly**: cpu_high
**Details**: CPU peaked at 1.007 cores versus a seven-day baseline mean of 0.263; throttling peaked at 23.7%. CPU fell below the warning threshold by 15:45 IST. Memory, pod count and restarts remained stable. HPA was capped at one replica with 65,209 undelivered messages in its snapshot. Opportunity lookup warnings peaked earlier at 15:23–15:24 IST; targeted samples showed 'No Living connections', but causation of the CPU spike is unproven. No historical candidate establishes a matching RCA.
**Anomalous**: Yes
**Deep Dive Direction**: YES — Triage direction remains correct: incoming indexing volume overwhelming the capped worker is the leading hypothesis; connection failures’ CPU impact remains unproven.

Need producer/tenant/action aggregates and event IDs to check upstream duplication, exact contact-lookup failure rates, deployed source/configuration and connectivity evidence, and incident-time CPU profiling. Check downstream Elasticsearch capacity before recommending scaling.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | CPU peaked at 1.007 cores with 23.7% throttled periods; memory, replicas and restarts remained stable. Confirms CPU pressure, not its trigger. | ✓ |
| Kubernetes | HPA was capped at one replica and ScalingLimited, with 65,209 undelivered messages in its snapshot. Supports constrained processing capacity; no recent rollout or restart. | ✓ |
| GCP logs and local source | Opportunity lookup failures are confirmed. Local code catches these failures and continues indexing; no CPU profile or complete failure-rate denominator connects them to CPU pressure. | ✓ |
| Elasticsearch metrics | Observed conversations-search metrics show no broad saturation: maximum node CPU 39.78%, write queue 1 and zero write-rejection increase. Contact-client connectivity remains unverified. | ✓ |
| Slack | Confirms the CPU alert but supplies no human diagnosis. Changes to the main indexing subscription have no established causal link to this worker. | ✓ |

### Discrepancies
- CPU pressure and the replica constraint are corroborated, but increased incoming volume and its initiating producer are unverified.
- Warnings peaked at 15:23–15:24 IST, before the 15:37:48 IST alert; alignment with the CPU peak is not established.
- Sample counts cannot establish total lookup failures, tenant dominance or duplication. One-pod concentration is expected with one replica.
- Cross-container correlation was not performed; the reported false flag does not exclude infrastructure involvement.
- Falsification checks weaken deployment, restart and broad Elasticsearch-saturation explanations. Distinguishing workload volume from expensive error handling still requires aligned ingress/ack rates, event identities and CPU profiling.

## Deep Dive Findings

{
  "workload": "conversations-elastic-indexing-throttled-worker",
  "assessment": "Incoming indexing work remains the leading CPU-pressure hypothesis. This pass found no downstream Elasticsearch saturation during the incident. The initiating producer action, upstream duplication rate and CPU contribution of contact failures remain unproven.",
  "root_cause_confirmed": false,
  "confidence": "medium",
  "new_findings": {
    "downstream_elasticsearch": {
      "domain": "conversations-search",
      "window_ist": "2026-11-19 14:30–16:00",
      "source": "Grafana CloudWatch datasource; 60-second domain and node metrics",
      "observations": {
        "maximum_node_cpu_percent": 39.78,
        "maximum_jvm_memory_pressure_percent": 71.2,
        "maximum_indexing_latency_ms": 0.731,
        "maximum_write_queue": 1,
        "write_rejection_counter_increase": 0,
        "minimum_node_free_storage_mib": 121787.113,
        "domain_nodes": 36,
        "active_shards": 643,
        "unassigned_shards": 0,
        "relocating_shards": 0,
        "red_or_yellow_status": false
      },
      "limitations": "CPU returned 32 node series; other node metrics returned 33. These measurements argue against incident-time saturation but do not establish shard-level spare throughput or a safe replica target."
    },
    "lookup_observability": {
      "finding": "Available worker ORM metrics contained conversation get/bulk series, but no contact lookup series. Local executeWithLatencyTracking records metrics only after successful operations; rejected calls bypass recording.",
      "implication": "Absent contact metrics do not prove absent attempts. An exact lookup failure percentage cannot be calculated from these metrics.",
      "verification": "Five re-fetched opportunity-seed warning entries contained 'No Living connections' and locationId Z8IqJkllOBbwvlvfSWtt.",
      "failure_count": null,
      "failure_percentage": null
    },
    "producer_attribution": {
      "finding": "Worker counters expose processing title and field/change type, without tenant, producer or action labels.",
      "alternative_checked": "Local resync code publishes conversation snapshots with isResyncEvent=true and normally no before state. Such events can generate field-create increments without new business-object creation.",
      "log_result": "A tenant-scoped elastic-indexing-resync-worker query for model CONVERSATION during 15:00–15:40 IST returned zero entries.",
      "limitations": "This narrow negative result does not exclude other producers, earlier resync activity or different logging paths. Producer aggregates and event IDs remain unavailable."
    },
    "profiling": {
      "finding": "The existing Pyroscope datasource returned no matching worker service name for the incident window.",
      "limitation": "No incident-time CPU profile was retrieved; CPU cost cannot be assigned to lookup failures."
    },
    "deployed_configuration": {
      "finding": "Live deployment remains revision 81 with the previously supplied image digest, no explicit ELASTICSEARCH_CRM_CONTACTS_URL and no envFrom entries.",
      "limitation": "Image-level configuration, deployed source revision and pod-level DNS/TCP behavior remain unverified. Missing deployment environment wiring alone does not prove an empty endpoint.",
      "local_source_revision": "70a5360b863"
    }
  },
  "blockers": [
    "GCP denied logging.buckets.list, preventing discovery of Log Analytics buckets for server-side aggregates.",
    "Available metrics lack producer identities and a complete contact-lookup attempt denominator.",
    "Parallel agent startup failed in this executor; checks continued directly."
  ],
  "recommended_next_action": "Obtain producer-side event IDs with location/action/isResyncEvent aggregates and verify the deployed contact-client configuration. Incident-time Elasticsearch measurements support reconsidering the cap, but refresh capacity and shard distribution before choosing a replica increase.",
  "log_queries_used": [
    {
      "purpose": "Revalidate contact lookup failures",
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-elastic-indexing-throttled-worker\" severity=WARNING jsonPayload.message=\"[ConversationOpportunitySeed] Failed to resolve opportunities for a new conversation\" timestamp>=\"2026-11-19T09:48:48Z\" timestamp<=\"2026-11-19T09:58:48Z\"",
      "limit": 5,
      "result": "Five matching entries."
    },
    {
      "purpose": "Check tenant-specific conversation resync",
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"elastic-indexing-resync-worker\" jsonPayload.locationId=\"Z8IqJkllOBbwvlvfSWtt\" jsonPayload.model=\"CONVERSATION\" timestamp>=\"2026-11-19T09:30:00Z\" timestamp<=\"2026-11-19T10:10:00Z\"",
      "limit": 5,
      "result": "Zero entries."
    }
  ],
  "source_urls": {
    "elasticsearch_capacity": "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=All&from=1795078800000&to=1795084200000",
    "lookup_failures": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-elastic-indexing-throttled-worker%22%0Aseverity%3DWARNING%0AjsonPayload.message%3D%22%5BConversationOpportunitySeed%5D%20Failed%20to%20resolve%20opportunities%20for%20a%20new%20conversation%22;timeRange=2026-11-19T09%3A48%3A48Z%2F2026-11-19T09%3A58%3A48Z?project=highlevel-backend"
  },
  "access": "Validated stored Grafana session, existing gcloud authentication and explicit production kubectl context. No production changes, messages or additional orchestrator investigations."
}

## Evidence

### Dashboards (incident window)
- [CPU — conversations-elastic-indexing-throttled-worker: peak near one core, then decline](https://prod.grafana.leadconnectorhq.com/d/iRRI_ay4k?orgId=1&var-datasource=ber8nnhvgsjy8f&var-team=crm&var-subTeam=conversations&var-cluster=workers-us-central-production-cluster&var-container=conversations-elastic-indexing-throttled-worker&var-pod_name=All&from=1795081068000&to=1795084668000&viewPanel=16)
- [Memory — conversations-elastic-indexing-throttled-worker: stable during CPU spike](https://prod.grafana.leadconnectorhq.com/d/iRRI_ay4k?orgId=1&var-datasource=ber8nnhvgsjy8f&var-team=crm&var-subTeam=conversations&var-cluster=workers-us-central-production-cluster&var-container=conversations-elastic-indexing-throttled-worker&var-pod_name=All&from=1795081068000&to=1795084668000&viewPanel=40)
- [Aws Elasticsearch](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=conversations-search&var-clientId=696084496265&var-nodeId=All&from=1795078800000&to=1795084200000)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-elastic-indexing-throttled-worker%22%0Aresource.labels.cluster_name%3D%22workers-us-central-production-cluster%22%0Aseverity%3DWARNING%0AjsonPayload.message%3D%22%5BConversationOpportunitySeed%5D%20Failed%20to%20resolve%20opportunities%20for%20a%20new%20conversation%22;timeRange=2026-11-19T09%3A48%3A48Z%2F2026-11-19T09%3A58%3A48Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-elastic-indexing-throttled-worker%22%0Aseverity%3DWARNING%0AjsonPayload.message%3D%22%5BConversationOpportunitySeed%5D%20Failed%20to%20resolve%20opportunities%20for%20a%20new%20conversation%22;timeRange=2026-11-19T09%3A48%3A48Z%2F2026-11-19T09%3A58%3A48Z?project=highlevel-backend)

### Related incidents
- [Team reports Pub/Sub delivery changes and backlog improvement — 19 Nov](https://gohighlevel.slack.com/archives/C096N87SBFW/p1795080056684749)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 1662.6s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.4s | ✓ |
| grafana-worker | worker | 137.7s | ✓ |
| kubectl-worker | worker | 116.0s | ✓ |
| log-explorer-worker | worker | 350.9s | ✓ |
| slack-worker | worker | 81.0s | ✓ |
| triage | worker | 14.5s | ✓ |
| triage_llm_ranking | decision | 25.8s | ✓ |
| deep_dive_pass_1 | worker | 298.0s | ✓ |
| follow_up_check | decision | 10.1s | ✓ |
| deep_dive_pass_2 | worker | 257.7s | ✓ |
| follow_up_check | decision | 11.5s | ✓ |
| deep_dive_pass_3 | worker | 329.8s | ✓ |
| cross_validation | worker | 18.3s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/ICTGMRJTXTRAB|#140288 CPUUsageAboveWarning>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=14158143755521728932&amp;alert_id=15811060861379992709|source>*) summary: Wokrload CPU Usage Above Warning Threshold dashboard: <https://prod.grafana.leadconnectorhq.com/d/iRRI_ay4k/crm-master?orgId=1&amp;viewPanel=16&amp;var-team=crm&amp;var-cluster=workers-us-central-production-cluster&amp;var-subTeam=conversations&amp;var-container=conversations-elastic-indexing-throttled-worker&amp;from=now-180m&amp;to=now|here> threshold: 85% description: Workload: conversations-elastic-indexing-throttled-worker Cluster: workers-us-central-production-cluster current_value: 92.05% *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/ICTGMRJTXTRAB|#140288 CPUUsageAboveWarning>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=14158143755521728932&amp;alert_id=15811060861379992709|source>*) summary: Wokrload CPU Usage Above Warning Threshold dashboard: <https://prod.grafana.leadconnectorhq.com/d/iRRI_ay4k/crm-master?orgId=1&amp;viewPanel=16&amp;var-team=crm&amp;var-cluster=workers-us-central-production-cluster&amp;var-subTeam=conversations&amp;var-container=conversations-elastic-indexing-throttled-worker&amp;from=now-180m&amp;to=now|here> threshold: 85% description: Workload: conversations-elastic-indexing-throttled-worker Cluster: workers-us-central-production-cluster current_value: 92.05%
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-19T10:35:07.795Z*