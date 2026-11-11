# Alert Investigation: PubSubUnackedMessages

**Investigation ID**: inv-2026-11-11-41336695
**Workload**: conversations-providers-deliver-twilio-events
**Container**: conversations-providers-deliver-twilio-events
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-11T14:24:32.438Z
**Status**: delivering

## Root Cause

**Summary**: A verified publisher surge coincided with slower processing, temporarily exceeding consumer throughput. The backlog peaked at 19:48 IST (14:18 UTC) and recovered to 199 messages by 19:56 IST (14:26 UTC). Metrics and logs corroborate accumulation with processing slowdown; the surge's origin and the slowdown's initiating cause remain unresolved.
**Confidence**: medium
**Category**: publisher_surge_with_concurrent_processing_slowdown

## Triage

**Primary Anomaly**: error_spike
**Details**: Confirmed main-subscription backlog peaked at 113,073 messages at 14:18 UTC and recovered to 199 by 14:26. ERROR volume reached 300/min versus a pre-14:00 median of 6/min; sampled logs show downstream 503s, connection resets, and 30–120-second processing delays. These establish degraded processing, not its causal contribution. Acknowledgements continued, sent/ack was 1.01, and no restarts were detected. Publisher ingress remains unverified.
**Anomalous**: Yes
**Deep Dive Direction**: YES — the triage direction remains supported: demand exceeded throughput alongside processing slowdown, but the initiating cause is unresolved. Early slow events on existing consuming pods weaken startup delay as the explanation.

Need actual dependency-call durations and errors, per-pod throttling and processing rates, and newly scaled subscriber readiness around 13:55–14:10 UTC, plus verification that inspected source matches the deployed image. These would distinguish demand pressure from dependency-induced or CPU-related capacity loss.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| Cloud Monitoring publisher and subscription metrics | Publish operations peaked at 48,195/min, 8.37 times the immediate baseline. Backlog subsequently peaked at 113,073 messages. Acknowledgements continued and sent/ack was 1.01, supporting temporary demand exceeding throughput without sustained retry amplification. | ✓ |
| Application processing metrics and GCP logs | Mean processing duration increased from 512 ms to 6,019 ms; logs independently show 30–120-second processing delays. This corroborates concurrent slowdown, but does not establish whether CPU pressure or downstream failures caused it. | ✓ |
| Kubernetes | Ready containers increased from 65 to 130, with no detected restarts or overlapping rollout. This weakens crash and deployment hypotheses; Ready status without probes does not prove subscriber readiness. | ✓ |

### Discrepancies
- Publisher surge is verified, but its business source and possible publisher-side duplication remain unknown.
- Dependency 503s existed before the surge, and ERROR volume peaked after the backlog peak. No correlated call durations establish dependency failure as the initiating cause.
- Per-pod CPU throttling and processing rates were not verified because Grafana authentication failed; CPU-induced capacity loss remains unproven.
- Nack measurements were inconsistently described as rates and a gauge. The malformed retry query cannot exclude poison messages; ten first-attempt samples are insufficient.
- Repeated subscription measurements share the same underlying Cloud Monitoring source. Cross-service impact, deployed-source provenance, and recovery beyond the supplied snapshots remain unverified.

## Deep Dive Findings

{
  "workload": "conversations-providers-deliver-twilio-events",
  "window": "2026-11-11T13:55:00Z/2026-11-11T14:25:00Z",
  "conclusion": {
    "classification": "Verified publisher surge with concurrent processing slowdown",
    "confidence": "High for ingress increase; low for attribution of capacity loss",
    "summary": "Publisher ingress is now verified independently of subscription deliveries. Processing slowdown and CPU pressure overlap the surge, while dependency errors already existed beforehand. Their individual contributions to the backlog remain unquantified."
  },
  "findings": {
    "publisher_ingress": {
      "source": "Fresh Cloud Monitoring query: topic/send_message_operation_count, successful operations, 60-second ALIGN_SUM",
      "baseline_median_per_minute": 5755,
      "baseline_interval": "13:50–14:00 UTC",
      "samples": {
        "14:00": 5845,
        "14:01": 14401,
        "14:02": 19349,
        "14:09": 48195
      },
      "peak_to_baseline_ratio": 8.37,
      "limitations": "Some minute samples were absent. These are publish-operation counts, not unique business-event counts; publisher-side duplication was not assessed."
    },
    "processing_and_cpu": {
      "source": "Re-examined prior-pass metric files",
      "mean_processing_ms": {
        "14:00": 512,
        "14:01": 2174,
        "14:07": 6019
      },
      "interpretation": "Processing duration increased alongside ingress. Cached aggregate throttling and maximum CPU series cannot establish which pods lost throughput or whether CPU pressure initiated the slowdown.",
      "per_pod_verification": "Blocked: fresh Grafana authentication returned HTTP 401; native browser inventory reported browser access host unavailable."
    },
    "dependency_errors": {
      "source": "Fresh bounded ERROR query",
      "events": [
        {
          "timestamp": "2026-11-11T13:55:17.147041320Z",
          "finding": "Company-wallet has-funds endpoint returned HTTP 503."
        },
        {
          "timestamp": "2026-11-11T13:56:20.275689125Z",
          "finding": "SMS compliance-config endpoint returned HTTP 503."
        },
        {
          "timestamp": "2026-11-11T13:56:20.276717185Z",
          "finding": "The same pod logged a Pub/Sub processing failure with upstream connection termination approximately 1 ms later; message identity was unavailable."
        },
        {
          "timestamp": "2026-11-11T13:57:54.713639497Z",
          "finding": "Persona compliance endpoint returned HTTP 503."
        }
      ],
      "duration_result": "All five returned entries lacked the requested duration, durationMs, elapsedMs, responseTime, processingTimeMs and httpRequest.latency fields.",
      "causal_boundary": "Errors precede the surge. Their presence alone does not explain the throughput change, and adjacent same-pod logs do not prove they belong to one message."
    },
    "scaled_subscribers": {
      "source": "Cached Kubernetes snapshot and fresh targeted log query",
      "ready_counts": {
        "14:00": 65,
        "14:06": 106,
        "14:08": 126,
        "14:10": 130
      },
      "sample": "Pod ending 2bq9n was created at 14:05:01 and marked Ready at 14:05:05 UTC.",
      "consumption_check": "No SLOW_EVENT entries returned for that pod during 14:05–14:10 UTC.",
      "limitation": "No readiness probe is configured. Ready status and absence of slow-event logs do not establish subscriber connection time or processing rate."
    },
    "deployed_source": {
      "source": "Fresh Kubernetes deployment and container registry queries",
      "revision": "226",
      "image_digest": "sha256:cca845e068785efe5bbf78d03be4c54445a0132b96769da356c7a70050997455",
      "deployment_pr_label": "35737",
      "registry_build_details": [],
      "source_matches_image": "Unverified: digest confirmed, but registry returned no build provenance linking it to the inspected local source."
    }
  },
  "log_queries_used": [
    {
      "filter": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"conversations-providers-deliver-twilio-worker\" AND timestamp>=\"2026-11-11T13:55:00Z\" AND timestamp<=\"2026-11-11T14:05:00Z\" AND severity>=ERROR AND jsonPayload.message=~\"503|upstream\"",
      "limit": 5,
      "order": "ascending",
      "result": "Five entries; no projected call-duration fields present."
    },
    {
      "filter": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"conversations-providers-deliver-twilio-worker\" AND resource.labels.pod_name=\"conversations-providers-deliver-twilio-worker-5d74b665c7-2bq9n\" AND timestamp>=\"2026-11-11T14:05:00Z\" AND timestamp<=\"2026-11-11T14:10:00Z\" AND severity=WARNING AND jsonPayload.message=\"[SLOW_EVENT]\"",
      "limit": 5,
      "order": "ascending",
      "result": "Zero entries; subscriber readiness remains unverified."
    }
  ],
  "log_explorer_urls": [
    "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-providers-deliver-twilio-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22503%7Cupstream%22;timeRange=2026-11-11T13%3A55%3A00Z%2F2026-11-11T14%3A05%3A00Z?project=highlevel-backend"
  ],
  "remaining_evidence": [
    "Caller-scoped dependency spans with actual elapsed durations and reliable message correlation.",
    "Per-pod CPU throttling, completed-message rates and first successful consumption timestamps.",
    "Build provenance linking the deployed digest to source and dependency versions."
  ],
  "execution": {
    "access": "Local gcloud and kubectl credentials; cached prior-pass metrics; stored Grafana session validation.",
    "parallel_agent_attempt": "Failed with a thread-store error.",
    "production_changes": false,
    "messages_posted": false,
    "orchestrator_launched": false
  }
}

## Evidence

### Dashboards (incident window)
- [Unacked messages — conversations-providers-deliver-twilio-events: 113k peak followed by drainage](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=conversations-providers-deliver-twilio-events&from=1794405240000&to=1794407400000&viewPanel=1)
- [Acknowledgements — conversations-providers-deliver-twilio-events: catch-up throughput](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=conversations-providers-deliver-twilio-events&from=1794405240000&to=1794407400000&viewPanel=3)
- [Nack requests — conversations-providers-deliver-twilio-events: below seven-day p95](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=conversations-providers-deliver-twilio-events&from=1794405240000&to=1794407400000&viewPanel=6)
- [Expired ack deadlines — conversations-providers-deliver-twilio-events: brief recovery-phase spike](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=conversations-providers-deliver-twilio-events&from=1794405240000&to=1794407400000&viewPanel=7)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-providers-deliver-twilio-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22503%7Cupstream%20connect%20error%7CECONNREFUSED%7CDEADLINE_EXCEEDED%7CMongo.%2Atimeout%22;timeRange=2026-11-11T14%3A15%3A00Z%2F2026-11-11T14%3A25%3A00Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-providers-deliver-twilio-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D~%22503%7Cupstream%22;timeRange=2026-11-11T13%3A55%3A00Z%2F2026-11-11T14%3A05%3A00Z?project=highlevel-backend)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 12

**Total worker time**: 1331.1s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 15.8s | ✓ |
| grafana-worker | worker | 108.8s | ✓ |
| kubectl-worker | worker | 134.7s | ✓ |
| log-explorer-worker | worker | 308.8s | ✓ |
| triage | worker | 14.0s | ✓ |
| triage_llm_ranking | decision | 62.9s | ✓ |
| deep_dive_pass_1 | worker | 265.0s | ✓ |
| follow_up_check | decision | 11.0s | ✓ |
| deep_dive_pass_2 | worker | 166.6s | ✓ |
| follow_up_check | decision | 11.7s | ✓ |
| deep_dive_pass_3 | worker | 213.6s | ✓ |
| cross_validation | worker | 18.3s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IG46LR2KQBIKA|#139582 Pubsub Unacked Messages above 50k >* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/a6cdb79b-c9b7-4122-8f0e-09c7e58ebcd1/view?orgId=1|source>*) AlertValues: Doc for this alert: http://platform.docs/alerts/pubsub-unacked-messages/ Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: conversations-providers-deliver-twilio-events Unacked Messages: 75073 GCP Link: https://console.cloud.google.com/cloudpubsub/subscription/detail/conversations-providers-deliver-twilio-events?project=highlevel-backend *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IG46LR2KQBIKA|#139582 Pubsub Unacked Messages above 50k >* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/a6cdb79b-c9b7-4122-8f0e-09c7e58ebcd1/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: conversations-providers-deliver-twilio-events Unacked Messages: 75073 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/conversations-providers-deliver-twilio-events?project=highlevel-backend>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-11T14:44:38.995Z*