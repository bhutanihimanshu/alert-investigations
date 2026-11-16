# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-16-791f2cd8
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: crm-messages-search
**Channel**: slack
**Detected**: 2026-11-16T18:22:06.045Z
**Status**: delivering

## Root Cause

**Summary**: The multi-node low-storage condition and sharp decline are supported by Grafana and related Slack reports. Shard relocation is a possible contributor, but no evidence establishes the initiating cause or causal chain. The node reduction happened after the main decline. Further AWS domain-event, shard-allocation, per-index storage, and CloudTrail evidence is needed.
**Confidence**: low
**Category**: Unconfirmed AWS OpenSearch storage depletion

## Triage

**Primary Anomaly**: none
**Details**: FreeStorageSpace on node S5hgPL_GRWiO-d5sR3SpwQ is 98,842 MB, near its 7-day low and well below the 7-day hourly median of 479,871 MB. Other nodes also show low free storage. Slack reports earlier storage alerts and shard redistribution after reducing nodes from 90 to 80, but causality is unverified. The closest historical investigation also left the cause unknown. Kubernetes and GCP logs did not provide corroborating evidence.
**Anomalous**: Yes
**Deep Dive Direction**: YES. The triage direction is still correct: the sharp storage decline overlapped with shard relocations, but the node reduction and metrics deployment happened later and aren’t established as causes. To determine root cause, get AWS domain events and shard allocation details, per-index storage, and CloudTrail changes for the alert window.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana CloudWatch metrics | Free storage on the alerting node fell sharply and remained near its 7-day low; other nodes also had low free storage. Shard relocations peaked during the initial decline, but had largely subsided by the alert. | ✓ |
| Slack alert and team discussion | Multiple nodes had storage alerts. The team reported node reduction and shard redistribution, but the discussion does not establish these as the cause. | ✓ |
| Kubernetes and GCP logs | No corroborating pod state or application log evidence was obtained; the Kubernetes context was unavailable and GCP queries timed out. | ✗ |

### Discrepancies
- The alert value is 99,086.73 MB; Grafana reported 98,842 MB near 18:20 UTC and 104,371 MB at 18:25 UTC.
- The earlier sharp decline preceded the node-count reduction, so that reduction does not explain its onset.
- Relocations overlapped with the decline, but available metrics do not identify the shards or quantify their storage impact.
- AWS domain events, shard allocation, per-index storage, and CloudTrail changes were unavailable; GCP log queries timed out.

## Deep Dive Findings

{
  "alert": {
    "type": "AWS ES Free Storage Space",
    "domain": "crm-messages-search",
    "node_id": "S5hgPL_GRWiO-d5sR3SpwQ",
    "fired_at_utc": "2026-11-16T18:22:06Z",
    "alert_value_mb": 99086.73
  },
  "findings": [
    {
      "finding": "The target node’s free storage fell from 475,066 MB at 00:00 UTC to about 106,500 MB by 09:10 UTC. It then declined more gradually, reaching 98,842 MB at 18:20 before recovering to 104,371 MB at 18:25. The alert was a brief threshold crossing during a longer low-storage condition.",
      "confidence": "high",
      "source_url": "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=S5hgPL_GRWiO-d5sR3SpwQ&from=1794787200000&to=1794854100000&viewPanel=39"
    },
    {
      "finding": "The storage decline overlapped with shard relocation activity: Shards.relocating peaked at 160 at 05:45 UTC, fell to 18 by 08:25 and 3 by 12:55, and was 2 around the alert. This timing supports relocation as a possible contributor, but does not show which shards moved or how much disk they consumed.",
      "confidence": "medium",
      "source_url": "https://us-east-1.console.aws.amazon.com/cloudwatch/deeplink.js?region=us-east-1#metricsV2:graph=%7B%22view%22%3A%22timeSeries%22%2C%22stacked%22%3Afalse%2C%22title%22%3A%22Shards.relocating%20%5Cu2014%20crm-messages-search%22%2C%22start%22%3A%222026-11-16T00%3A00%3A00Z%22%2C%22end%22%3A%222026-11-16T18%3A35%3A00Z%22%2C%22region%22%3A%22us-east-1%22%2C%22metrics%22%3A%5B%5B%22AWS/ES%22%2C%22Shards.relocating%22%2C%22ClientId%22%2C%22696084496265%22%2C%22DomainName%22%2C%22crm-messages-search%22%2C%7B%22stat%22%3A%22Maximum%22%2C%22period%22%3A300%7D%5D%5D%7D"
    },
    {
      "finding": "The domain’s Nodes metric remained at 93 through 08:25, then dropped to 83 by 12:55 and stayed there through 18:30. Since the major storage decline was already underway before the node-count reduction, that reduction does not explain the initial drop. It also occurred after relocations had largely subsided.",
      "confidence": "medium",
      "source_url": "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=All&from=1794787200000&to=1794854100000&viewPanel=6"
    },
    {
      "finding": "At 18:20 UTC, two other returned node series had less free storage than the alert node: about 34,690 MB and 49,146 MB. The condition was therefore not unique to the alerting node. The query returned 80 node series while Nodes reported 83, so the series do not establish complete active-node coverage.",
      "confidence": "medium",
      "source_url": "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=All&from=1794787200000&to=1794854100000&viewPanel=21"
    },
    {
      "finding": "IndexingRate was about 1,109 at the 05:45 relocation peak and 8,563 at 18:20; its observed peak was 16,300 at 14:35. These are domain-level rates and do not identify which indices or nodes consumed storage. Earlier same-day investigation samples reported indexing and search rates below their seven-day medians, which does not establish a sustained workload surge as the trigger.",
      "confidence": "low",
      "source_url": "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=All&from=1794787200000&to=1794854100000&viewPanel=20"
    },
    {
      "finding": "The previously cited Grafana URL using viewPanel=21 for shard relocations is mislabeled: dashboard metadata identifies panel 21 as “Free storage space by Node.” Shards.relocating was queried separately through the Grafana CloudWatch datasource; the dashboard has no panel for that metric.",
      "confidence": "high",
      "source_url": "https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=All&from=1794787200000&to=1794854100000&viewPanel=21"
    }
  ],
  "root_cause": {
    "status": "unconfirmed",
    "assessment": "Multi-node low storage and a large early-day decline are confirmed. Shard relocations coincided with that decline, but available evidence does not establish whether relocation, index growth, or an AWS-side event initiated it. The later node-count reduction occurred after most of the decline and relocation peak; the reported metrics deployment was also after the sharp decline.",
    "confidence": "low",
    "disconfirming_evidence": "Relocations had fallen from 160 to 2 by the alert, while storage remained low and continued a slower decline. The node-count reduction followed the main storage drop. IndexingRate is domain-wide and cannot attribute disk changes to a particular node or index.",
    "recommended_next_evidence": "Correlate AWS OpenSearch domain events and CloudTrail changes with timestamped shard-to-node allocation, per-index store-size deltas, and producer or backfill throughput."
  },
  "metrics_queries_used": [
    {
      "source": "Grafana CloudWatch datasource, AWS/ES",
      "metrics": [
        "FreeStorageSpace by NodeId",
        "Shards.relocating",
        "Nodes",
        "IndexingRate"
      ],
      "window_utc": "2026-11-16T00:00:00Z to 2026-11-16T18:35:00Z",
      "period": "300 seconds"
    }
  ],
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=~\"(conversations-elastic-indexing-worker|conversations-elastic-indexing-messages-worker|conversations-message-search-backfill-conversation-worker)\" AND timestamp>=\"2026-11-16T18:17:00Z\" AND timestamp<=\"2026-11-16T18:27:00Z\" AND severity>=ERROR AND jsonPayload.message:*",
      "result": "The gcloud logging read request produced no output before timing out after about 60 seconds; it was interrupted. No result can be inferred.",
      "url": "https://console.cloud.google.com/logs/query?project=highlevel-backend&query=resource.type%3D%22k8s_container%22%20AND%20resource.labels.container_name%3D~%22%28conversations-elastic-indexing-worker%7Cconversations-elastic-indexing-messages-worker%7Cconversations-message-search-backfill-conversation-worker%29%22%20AND%20timestamp%3E%3D%222026-11-16T18%3A17%3A00Z%22%20AND%20timestamp%3C%3D%222026-11-16T18%3A27%3A00Z%22%20AND%20severity%3E%3DERROR%20AND%20jsonPayload.message%3A%2A&timeRange=2026-11-16T18%3A17%3A00Z%2F2026-11-16T18%3A27%3A00Z"
    }
  ],
  "log_explorer_urls": [
    "https://console.cloud.google.com/logs/query?project=highlevel-backend&query=resource.type%3D%22k8s_container%22%20AND%20resource.labels.container_name%3D~%22%28conversations-elastic-indexing-worker%7Cconversations-elastic-indexing-messages-worker%7Cconversations-message-search-backfill-conversation-worker%29%22%20AND%20timestamp%3E%3D%222026-11-16T18%3A17%3A00Z%22%20AND%20timestamp%3C%3D%222026-11-16T18%3A27%3A00Z%22%20AND%20severity%3E%3DERROR%20AND%20jsonPayload.message%3A%2A&timeRange=2026-11-16T18%3A17%3A00Z%2F2026-11-16T18%3A27%3A00Z"
  ],
  "limitations": [
    "AWS domain events, shard allocation details, per-index storage, and CloudTrail changes were not retrieved. The local AWS CLI and boto3 were unavailable, no AWS credential environment was present, and browser access was unavailable.",
    "The GCP log histogram timed out in the prior pass; the targeted application ERROR query in this pass also timed out, so application logs did not corroborate the alert.",
    "Grafana CloudWatch metrics were accessible and queried read-only; those aggregate metrics cannot identify the causal shard, index, or AWS configuration event."
  ],
  "related_slack_urls": [
    "https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794853326045929?thread_ts=1794853326.045929",
    "https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794819724914529?thread_ts=1794819724.914529",
    "https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794825127209359?thread_ts=1794822724.434039"
  ]
}

## Evidence

### Dashboards (incident window)
- [Free storage — crm-messages-search node S5hgPL_GRWiO-d5sR3SpwQ (current near 7-day low)](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&viewPanel=39&var-domainName=crm-messages-search&var-nodeId=S5hgPL_GRWiO-d5sR3SpwQ&from=1794248889000&to=1794853689000)
- [Aws Elasticsearch — panel 39](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=S5hgPL_GRWiO-d5sR3SpwQ&from=1794787200000&to=1794854100000&viewPanel=39)
- [Aws Elasticsearch — panel 6](https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&var-datasource=4PeY_YyVz&var-region=us-east-1&var-domainName=crm-messages-search&var-clientId=696084496265&var-nodeId=All&from=1794787200000&to=1794854100000&viewPanel=6)

### Related incidents
- [Message-search metrics deployment — green passed, prod promotion requested](https://gohighlevel.slack.com/archives/C096N87SBFW/p1794819112828199?thread_ts=1794819112.828199)
- [Earlier crm-messages-search storage alerts and investigation](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794819724914529?thread_ts=1794819724.914529)
- [Team discussion: node reduction from 90 to 80 and shard redistribution](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794825127209359?thread_ts=1794822724.434039)

## Investigation Timeline

**Deep dive passes**: 2
**Worker calls**: 12

**Total worker time**: 986.2s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 16.5s | ✓ |
| grafana-worker | worker | 76.1s | ✓ |
| kubectl-worker | worker | 79.6s | ✓ |
| log-explorer-worker | worker | 77.5s | ✓ |
| slack-worker | worker | 77.5s | ✓ |
| triage | worker | 15.0s | ✓ |
| triage_llm_ranking | decision | 33.4s | ✓ |
| deep_dive_pass_1 | worker | 212.0s | ✓ |
| follow_up_check | decision | 9.6s | ✓ |
| deep_dive_pass_2 | worker | 366.4s | ✓ |
| follow_up_check | decision | 9.4s | ✓ |
| cross_validation | worker | 13.2s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/ISR8UEA91WKPA|#140032 AWS ES Free Storage Space>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#free-storage|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having low storage space. Please check the details below, Cluster: crm-messages-search Node Id: S5hgPL_GRWiO-d5sR3SpwQ Storage Space: 99086.73 MB Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=39&amp;var-domainName=crm-messages-search&amp;var-nodeId=S5hgPL_GRWiO-d5sR3SpwQ> Help Doc : <http://platform.docs/data/faq/elasticsearch/elasticsearch-common-errors/#aws-es-free-storage-space-via-platform-critical> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/ISR8UEA91WKPA|#140032 AWS ES Free Storage Space>* via CRM-conversations runbook: <https://platform.docs/alerts/runbooks/opensearch/alert-response-operational/#free-storage|here> AlertValues: Hey, looks like one of the nodes in an AWS ES cluster is having low storage space. Please check the details below, Cluster: crm-messages-search Node Id: S5hgPL_GRWiO-d5sR3SpwQ Storage Space: 99086.73 MB Dashboard Link: <https://prod.grafana.leadconnectorhq.com/d/eCN_s8yWk/aws-elasticsearch?orgId=1&amp;viewPanel=39&amp;var-domainName=crm-messages-search&amp;var-nodeId=S5hgPL_GRWiO-d5sR3SpwQ> Help Doc : <http://platform.docs/data/faq/elasticsearch/elasticsearch-common-errors/#aws-es-free-storage-space-via-platform-critical>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-16T18:39:34.009Z*