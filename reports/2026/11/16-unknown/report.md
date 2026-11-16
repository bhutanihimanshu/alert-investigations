# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-16-bd40001d
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: conversations-search
**Channel**: slack
**Detected**: 2026-11-16T08:54:13.149Z
**Status**: delivering

## Root Cause

**Summary**: The Yellow transition and concurrent OpenSearch timeouts are supported by multiple sources, but the initiating cause is unconfirmed. A node-membership or shard-allocation event and workload-related latency remain hypotheses. Shard allocation details and AWS domain events are needed to distinguish them.
**Confidence**: low
**Category**: Unconfirmed OpenSearch cluster health event

## Triage

**Primary Anomaly**: none
**Details**: The conversations-search OpenSearch cluster transitioned from its usual Green status to Yellow near 08:54 UTC. Slack reports a second concurrent Yellow alert and later recovery; the cause remains unconfirmed. No alert-window CPU or JVM samples were available, Kubernetes pod data could not be retrieved, and the queried application logs returned no errors. A past mapping-related Yellow incident is not confirmed by current evidence.
**Anomalous**: Yes
**Deep Dive Direction**: YES. The triage direction is still correct: investigate node membership, shard allocation and unassigned replicas, plus AWS/OpenSearch domain events and availability-zone signals. Yellow recovered, and the brief node-count decrease coincided with it; indexing timeouts overlapped but don’t establish the cause.

More data is needed to determine root cause: shard allocation state and replica identities, AWS/OpenSearch domain events or health signals, and relevant indexing/workload metrics. Verify the indexing worker’s dependency usage before recommending fixes based on its timeouts.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | The conversations-search cluster transitioned from its usual Green status to Yellow around 08:49 UTC. The cluster-status series ended at 08:54 UTC; CPU and JVM samples were unavailable during the alert window. | ✓ |
| GCP logs and source review | Multiple indexing-worker pods logged 30-second OpenSearch request timeouts from 08:44 UTC onward. The worker uses OpenSearch bulk and get operations, confirming impact, but the timeouts do not establish what caused Yellow. | ✓ |
| Slack | A second Yellow alert fired for the same cluster, and a later update reported recovery. The cause remained unconfirmed; unassigned replicas were only a possibility. | ✓ |
| Kubernetes and AWS | Pod state, shard allocation details, AWS domain events, and availability-zone signals were unavailable, leaving the initiating event unverified. | ✓ |

### Discrepancies
- Indexing-worker timeouts were observed before the reported Yellow samples, but available evidence cannot determine whether they preceded the underlying cluster event or contributed to it.
- Slack reports recovery, while the queried Grafana cluster-status series ended at 08:54 UTC and does not independently verify recovery timing.

## Deep Dive Findings

{
  "alert": {
    "name": "AWS ES Cluster Health Yellow",
    "cluster": "conversations-search",
    "alert_time_utc": "2026-11-16T08:54:13Z"
  },
  "findings": [
    {
      "finding": "Indexing-worker OpenSearch request timeouts were already occurring before the observed Yellow interval.",
      "evidence": "Read-only GCP log queries found 30-second Request Timeout errors from multiple indexing-worker pods between 08:44 and 08:48:59 UTC, before the reported Yellow samples around 08:49 UTC. This establishes early workload impact, but not what initiated it.",
      "confidence": "medium",
      "source_urls": [
        "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-elastic-indexing-worker%22%0Atimestamp%3E%3D%222026-11-16T08%3A44%3A00Z%22%0Atimestamp%3C%3D%222026-11-16T08%3A49%3A00Z%22%0Aseverity%3E%3DERROR%0AjsonPayload.error.errorMessage%3D%22Request%20Timeout%20after%2030000ms%22;timeRange=2026-11-16T08%3A44%3A00Z%2F2026-11-16T08%3A49%3A00Z?project=highlevel-backend",
        "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-clientId=696084496265&var-datasource=4PeY_YyVz&from=1794817453000&to=1794821053000&viewPanel=59"
      ]
    },
    {
      "finding": "The worker uses Elasticsearch bulk and get operations, so these timeouts are consistent with OpenSearch impact; they do not prove the worker caused Yellow.",
      "evidence": "The prior source review verified those dependency calls. Timeouts were present before the observed Yellow samples and continued afterward; request volume, shard allocation, and domain events were unavailable to distinguish a shared workload trigger from cluster degradation.",
      "confidence": "medium",
      "source_urls": [
        "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-elastic-indexing-worker%22%0Atimestamp%3E%3D%222026-11-16T08%3A44%3A00Z%22%0Atimestamp%3C%3D%222026-11-16T09%3A10%3A00Z%22%0Aseverity%3E%3DERROR;timeRange=2026-11-16T08%3A44%3A00Z%2F2026-11-16T09%3A10%3A00Z?project=highlevel-backend",
        "https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794819257672559?thread_ts=1794819253198779"
      ]
    },
    {
      "finding": "The initiating cluster event remains unconfirmed.",
      "evidence": "The supplied metrics show a brief node-count dip concurrent with Yellow and subsequent recovery; master reachability remained 1. No shard identities, unassigned-replica details, AWS domain events, or availability-zone signals were retrieved. AWS CLI was unavailable, and the browser access host was unavailable in this executor.",
      "confidence": "high",
      "source_urls": [
        "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-clientId=696084496265&var-datasource=4PeY_YyVz&from=1794818340000&to=1794820200000&viewPanel=6"
      ]
    },
    {
      "finding": "The current evidence does not support the historical mapping-error explanation.",
      "evidence": "The targeted mapping/mapper/strict_dynamic error query returned zero entries for the indexing worker in the alert window.",
      "confidence": "medium",
      "source_urls": []
    }
  ],
  "root_cause": {
    "status": "unconfirmed",
    "assessment": "Worker timeouts predated the observed Yellow samples, making them evidence of concurrent OpenSearch or workload impact rather than proof of cause. The node-count decrease and Yellow recovery are consistent with a transient membership or shard-allocation event, but the available data cannot identify the initiating event.",
    "hypotheses_not_confirmed": [
      "Node departure or shard reallocation temporarily left replicas unassigned.",
      "Indexing workload or another source of cluster latency contributed to the transition."
    ],
    "next_evidence_needed": [
      "OpenSearch cluster health, shard allocation explain, and unassigned replica identities for approximately 08:44–09:00 UTC.",
      "AWS OpenSearch domain events, CloudTrail changes, and availability-zone or AWS Health signals for the same interval.",
      "Indexing rate, request latency, and worker error counts before and during Yellow."
    ]
  },
  "log_queries_used": [
    {
      "purpose": "Check whether indexing timeouts preceded the observed Yellow interval",
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"conversations-elastic-indexing-worker\" AND timestamp>=\"2026-11-16T08:44:00Z\" AND timestamp<=\"2026-11-16T08:49:00Z\" AND severity>=ERROR AND jsonPayload.error.errorMessage=\"Request Timeout after 30000ms\"",
      "result": "Projected samples showed 30-second request timeouts across multiple worker pods from 08:44:00 through 08:48:59 UTC.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-elastic-indexing-worker%22%0Atimestamp%3E%3D%222026-11-16T08%3A44%3A00Z%22%0Atimestamp%3C%3D%222026-11-16T08%3A49%3A00Z%22%0Aseverity%3E%3DERROR%0AjsonPayload.error.errorMessage%3D%22Request%20Timeout%20after%2030000ms%22;timeRange=2026-11-16T08%3A44%3A00Z%2F2026-11-16T08%3A49%3A00Z?project=highlevel-backend"
    },
    {
      "purpose": "Inspect indexing-worker errors across the broader alert window",
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"conversations-elastic-indexing-worker\" AND timestamp>=\"2026-11-16T08:44:00Z\" AND timestamp<=\"2026-11-16T09:10:00Z\" AND severity>=ERROR",
      "result": "Prior projected results showed 30-second bulk update/preparation timeouts through 09:00:56 UTC; the 08:54 sample also contained the nested error field `jsonPayload.error.errorMessage`.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-elastic-indexing-worker%22%0Atimestamp%3E%3D%222026-11-16T08%3A44%3A00Z%22%0Atimestamp%3C%3D%222026-11-16T09%3A10%3A00Z%22%0Aseverity%3E%3DERROR;timeRange=2026-11-16T08%3A44%3A00Z%2F2026-11-16T09%3A10%3A00Z?project=highlevel-backend"
    }
  ],
  "limitations": [
    "Shard allocation state and unassigned replica identities were not retrieved.",
    "AWS domain events, CloudTrail, AWS Health, and availability-zone signals were not verified.",
    "No alert-window CPU or JVM samples were available; Kubernetes pod state was unavailable.",
    "The before-Yellow timeout check used limited samples and does not establish the start or rate of the timeout pattern."
  ]
}

## Evidence

### Dashboards (incident window)
- [Cluster yellow status — conversations-search (yellow began in final two 5-minute samples)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-clientId=696084496265&var-datasource=4PeY_YyVz&from=1794817453000&to=1794821053000&viewPanel=59)
- [CPU by node — conversations-search (7-day baseline; no samples in alert window)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-clientId=696084496265&var-datasource=4PeY_YyVz&from=1794216253000&to=1794821053000&viewPanel=66)
- [Aws Elasticsearch — panel 6](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-domainName=conversations-search&var-clientId=696084496265&var-datasource=4PeY_YyVz&from=1794818340000&to=1794820200000&viewPanel=6)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22CRM-conversations%22%0AjsonPayload.message%3D~%22yellow%7Ccluster%7Copensearch%7Celastic%22%0Aseverity%3E%3DERROR;timeRange=2026-11-16T08%3A49%3A13Z%2F2026-11-16T08%3A59%3A13Z?project=highlevel-backend)

### Related incidents
- [Concurrent conversations-search Yellow alert #139960 in #alerts-database](https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794819253198779?thread_ts=1794819253198779)
- [RCA reports recovery and no observed consumer errors or slow queries; cause remains unconfirmed — 16 Nov](https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794819257672559?thread_ts=1794819253198779)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 851.0s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.2s | ✓ |
| grafana-worker | worker | 98.2s | ✓ |
| kubectl-worker | worker | 131.5s | ✓ |
| log-explorer-worker | worker | 67.8s | ✓ |
| slack-worker | worker | 62.8s | ✓ |
| triage | worker | 12.9s | ✓ |
| triage_llm_ranking | decision | 34.4s | ✓ |
| deep_dive_pass_1 | worker | 162.6s | ✓ |
| follow_up_check | decision | 15.0s | ✓ |
| deep_dive_pass_2 | worker | 105.8s | ✓ |
| follow_up_check | decision | 11.0s | ✓ |
| deep_dive_pass_3 | worker | 120.2s | ✓ |
| cross_validation | worker | 17.7s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I7RGLA1ZEESL2|#139959 AWS ES Cluster Health Yellow>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#cluster-status-red--yellow|here> AlertValues: Cluster: conversations-search *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I7RGLA1ZEESL2|#139959 AWS ES Cluster Health Yellow>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#cluster-status-red--yellow|here> AlertValues: Cluster: conversations-search
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-16T09:06:42.582Z*