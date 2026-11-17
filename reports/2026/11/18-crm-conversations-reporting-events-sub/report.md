# Alert Investigation: PubSubUnackedMessages

**Investigation ID**: inv-2026-11-17-a6e8891e
**Workload**: crm-conversations-reporting-events-sub
**Container**: crm-conversations-reporting-events-sub
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-17T19:14:30.919Z
**Status**: delivering

## Root Cause

**Summary**: OpenSearch write-queue saturation rejected reporting-worker writes during the growing Pub/Sub backlog. Independent application logs and throughput/latency metrics support this causal contribution, while local source explains how failed writes remain unacknowledged. The trigger for saturation is unknown; traffic surge, poison messages and quantified retry amplification are not established. Next, verify the affected OpenSearch domain and incident-time shard, disk, merge and configuration history.
**Confidence**: medium
**Category**: opensearch_write_saturation

## Triage

**Primary Anomaly**: error_spike
**Details**: OpenSearch HTTP 429 rejected_execution_exception indicates write saturation: 49/50 sampled errors showed all 8 write threads active and approximately 10,030 queued tasks against 10,000 capacity. ERROR volume peaked at 19,527 entries/min at 00:39:30 IST. Subscription backlog attribution was confirmed at 12,823,920 messages and subsequently reached 15,311,662 despite 300 pods and zero observed restarts. Elevated processing latency and expired ack deadlines support processing degradation; exact nack rates require metric-semantics validation. No historical candidate establishes an exact RCA match: ClickHouse capacity cases involve a different dependency, and traffic-surge cases lack this rejection storm.
**Anomalous**: Yes
**Deep Dive Direction**: YES — Triage direction remains correct: OpenSearch write saturation is the confirmed bottleneck; its initiating cause remains unresolved. A traffic surge is unsupported.

Need verified endpoint-to-domain/node mapping; incident-time shard placement, disk latency, merges, indexing pressure and relocations; maintenance/configuration history before 00:09 IST; and deployed source/dependency provenance plus Pub/Sub message-ID redelivery evidence to quantify retry amplification.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| GCP application logs | At 00:39:30 IST, ERROR volume peaked at 19,527/min. Of 50 sampled errors, 49 showed OpenSearch HTTP 429 write rejections, with 8 active threads and approximately 10,030 queued tasks against 10,000 capacity. This directly establishes rejected writes, not the initiating cause. | ✓ |
| Grafana and Cloud Monitoring | Backlog attribution was verified at 12,823,920 messages and subsequently reached 15,311,662. Elevated batch-processing latency and expired ack deadlines corroborate processing degradation during the rejection window. Metrics showed 300 pods and zero restart increases. | ✓ |
| Local worker source | Failed OpenSearch items are excluded from successAckIds, supporting the mechanism from rejected writes to unacknowledged work. Deployed source and dependency provenance remain unverified. | ✓ |
| Falsification checks | Continued acknowledgements and zero observed restart increases argue against a complete consumer outage or crash loop. Missing delivery-attempt and message-ID evidence cannot rule out redelivery. Publisher throughput was not verified, so a traffic surge is unsupported. | ✓ |

### Discrepancies
- OpenSearch saturation is corroborated as a processing bottleneck; its initiating cause and contribution relative to incoming traffic remain unresolved.
- The reported 2,640.9 nacks/second is unvalidated: nack_requests is a GAUGE with cumulative-count semantics and observed decreases. Do not use this rate to quantify retry amplification.
- The 49/50 rejection proportion and single-pod concentration describe a bounded sample, not fleet-wide distributions.
- Kubernetes reads timed out; empty deployment and event arrays do not establish absence of changes. Pod-health conclusions rely on metrics.
- Grafana subsequently returned HTTP 401. Incident-time OpenSearch node metrics, endpoint mapping, maintenance history and deployed-code provenance remain unavailable.

## Deep Dive Findings

{
  "workload": "crm-conversations-reporting-events-sub",
  "pass": 3,
  "conclusion": "OpenSearch write saturation remains the confirmed processing bottleneck from prior evidence. This pass did not establish its initiating cause, verify endpoint-to-domain mapping, or quantify retry amplification.",
  "confidence": {
    "processing_bottleneck": "high",
    "initiating_cause": "low"
  },
  "new_findings": {
    "nack_metric_semantics": {
      "verified_live": true,
      "metric": "pubsub.googleapis.com/subscription/nack_requests",
      "kind": "GAUGE",
      "description": "Cumulative count of messages that were nacked. One message nacked multiple times will be counted multiple times.",
      "sample_period_seconds": 60,
      "ingestion_delay_seconds": 360,
      "cached_incident_series": "84 points with 46 decreases between consecutive points.",
      "interpretation": "Neither summing these values nor treating them as a conventional monotonic counter establishes nack throughput. The previously reported 2640.9 nacks/second remains unvalidated."
    },
    "message_identity": {
      "local_source": "The reporting worker's explicit Pub/Sub messageId logging block is commented out. Its locally installed synchronous-pull dependency nacks unsuccessful messages by setting acknowledgement deadlines to zero.",
      "live_log_check": "No entries matched the five queried message-ID field paths during 00:35:30–00:45:30 IST.",
      "limitation": "This establishes an evidence gap, not absence of redelivery. Exact deployed source and dependency contents remain unverified.",
      "retry_amplification_fraction": null
    },
    "early_rejections": {
      "window_ist": "2026-11-18 00:05:00–00:15:00",
      "matching_entries": 0,
      "interpretation": "The exact rejected-execution message filter returned no entries in this window. It does not establish when saturation began or negate the later confirmed rejection storm."
    },
    "provenance": {
      "image_manifest_lookup": "Direct GCR manifest lookup for the supplied deployed digest returned HTTP 404.",
      "result": "No image configuration labels or deployed dependency provenance could be retrieved."
    }
  },
  "access_limitations": [
    "Grafana session revalidation returned HTTP 401; incident-time CloudWatch node metrics remain unavailable.",
    "No native AWS/OpenSearch connector or AWS CLI was available. The prior Kubernetes secret-read denial remains unresolved.",
    "Parallel-agent launch failed because its thread context could not be loaded.",
    "Local Terraform and source history cannot establish applied production configuration or deployed code."
  ],
  "remaining_evidence_needed": [
    "Authorized endpoint-to-domain mapping and node identity for 817397e44e2bfe92700e08758331ccf9.",
    "Incident-time shard placement, disk latency, merges, indexing pressure and relocations.",
    "OpenSearch maintenance and configuration audit history preceding 2026-11-18 00:09 IST.",
    "Image-to-build provenance and Pub/Sub message-ID-level delivery records."
  ],
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"crm-conversations-reporting-worker\" severity>=ERROR jsonPayload.message=~\"rejected execution\" timestamp>=\"2026-11-17T18:35:00Z\" timestamp<=\"2026-11-17T18:45:00Z\"",
      "limit": 5,
      "order": "ascending",
      "result_count": 0
    },
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"crm-conversations-reporting-worker\" severity>=INFO (jsonPayload.messageId:* OR jsonPayload.metadata.messageId:* OR jsonPayload.meta.id:* OR jsonPayload.metadata.meta.id:* OR jsonPayload.message.messageId:*) timestamp>=\"2026-11-17T19:05:30Z\" timestamp<=\"2026-11-17T19:15:30Z\"",
      "limit": 20,
      "result_count": 0
    }
  ],
  "log_explorer_urls": [
    {
      "evidence": "Confirmed rejection storm; URL validated in the supplied prior investigation, not revalidated this pass.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-reporting-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22rejected%20execution%22;timeRange=2026-11-17T19%3A05%3A30Z%2F2026-11-17T19%3A15%3A30Z?project=highlevel-backend"
    }
  ],
  "access": "Existing local gcloud authentication and Grafana session validation; read-only. No messages posted, production changes made, or additional orchestrator investigation launched."
}

## Evidence

### Dashboards (incident window)
- [Unacked messages — reporting-events-sub: backlog grows past 14 million](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794939563000&to=1794943163000&viewPanel=1)
- [Ack throughput — reporting-events-sub: declining acknowledgements](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794939563000&to=1794943163000&viewPanel=3)
- [Expired ack deadlines — reporting-events-sub: timeout surge](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-reporting-events-sub&from=1794939563000&to=1794943163000&viewPanel=7)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-reporting-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22rejected%20execution%22;timeRange=2026-11-17T19%3A05%3A30Z%2F2026-11-17T19%3A15%3A30Z?project=highlevel-backend)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 12

**Total worker time**: 1344.2s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 16.8s | ✓ |
| grafana-worker | worker | 117.8s | ✓ |
| kubectl-worker | worker | 92.9s | ✓ |
| log-explorer-worker | worker | 376.7s | ✓ |
| triage | worker | 14.1s | ✓ |
| triage_llm_ranking | decision | 20.2s | ✓ |
| deep_dive_pass_1 | worker | 329.0s | ✓ |
| follow_up_check | decision | 8.8s | ✓ |
| deep_dive_pass_2 | worker | 162.9s | ✓ |
| follow_up_check | decision | 9.2s | ✓ |
| deep_dive_pass_3 | worker | 176.8s | ✓ |
| cross_validation | worker | 19.0s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I3VDHD33WTV5J|#140135 Pubsub Unacked Messages above 5million>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/e86b2cb5-9fa9-4384-b29b-2eaf16b57717/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-reporting-events-sub Unacked Messages: 12823920 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-reporting-events-sub?project=highlevel-backend> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I3VDHD33WTV5J|#140135 Pubsub Unacked Messages above 5million>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/e86b2cb5-9fa9-4384-b29b-2eaf16b57717/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-reporting-events-sub Unacked Messages: 12823920 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-reporting-events-sub?project=highlevel-backend>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-17T19:37:21.965Z*