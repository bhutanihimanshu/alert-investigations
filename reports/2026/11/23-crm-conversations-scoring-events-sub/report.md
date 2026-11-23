# Alert Investigation: PubSubUnackedMessages

**Investigation ID**: inv-2026-11-23-f35829b6
**Workload**: crm-conversations-scoring-events-sub
**Container**: crm-conversations-scoring-events-sub
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-23T15:18:23.229Z
**Status**: delivering

## Root Cause

**Summary**: A transient increase in published traffic most likely exceeded effective consumer throughput, producing a short-lived backlog on crm-conversations-scoring-events-sub. Sustained acknowledgements, sent/ack ratio 1.0009, low nacks and zero recorded restarts argue against a sustained retry storm or consumer crash. Backlog recovered before alert receipt; the upstream reason for the traffic increase remains unverified.
**Confidence**: medium
**Category**: transient_traffic_surge

## Triage

**Primary Anomaly**: traffic_spike
**Details**: Topic publish operations rose from 1.46M to 3.02M per five minutes after 20:30 IST. Backlog peaked at 97,085 at 20:44 IST and fell below 10,000 by 20:46 IST, before alert receipt. HPA reached its 70-pod ceiling; acknowledgements remained high, sent/ack was 1.0009, and recorded restarts were zero. Strong pattern match to inv-2026-09-30-f02dc591. Sparse errors are not established contributors; missing delivery-attempt fields and dead-letter series prevent claiming zero redeliveries or dead-letter activity.
**Anomalous**: Yes
**Deep Dive Direction**: shortcut_verification: Verify aligned topic publish and acknowledgement rates around 20:30–20:46 IST to confirm transient inflow exceeding throughput, then confirm sustained recovery and alert resolution. Identify the publisher only if recurrence warrants further investigation; do not attribute recovery solely to HPA because it reached maximum replicas before the backlog peak.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| Cloud Monitoring | Publishing increased after 20:30 IST and exceeded acknowledgements during initial backlog growth and before the 97,085 peak at 20:44. Acknowledgements remained active; backlog stayed below 10,000 from 20:46 through 21:21. | ✓ |
| Kubernetes | HPA scaled to its 70-pod maximum by 20:35 IST. Retained pods showed zero restarts; the latest deployment completed approximately 4h40m before alert receipt. This supports active consumers responding to load, not a demonstrated crash-driven backlog. | ✓ |
| Application logs and handler source | Sparse processing errors and 14 slow events were observed, but no causal link to backlog growth was established. Caught CONTACT-processing errors preserve success=true, weakening a sustained error-driven nack-loop explanation. | ✓ |

### Discrepancies
- Grafana and direct Cloud Monitoring Pub/Sub queries share underlying telemetry; they are not independent corroboration.
- Publish operations and acknowledgements are distinct counters, and the 20:45 publish sample is missing; exact backlog accounting is unavailable.
- HPA reached maximum nine minutes before the backlog peak; recovery cannot be attributed solely to scaling.
- Missing delivery-attempt fields and dead-letter series do not establish zero redelivery or dead-letter activity. Current pod resource snapshots do not establish incident-time resource usage.
- Metric recovery is confirmed in supplied findings, but OnCall remained unresolved. The publisher responsible for increased volume was not identified.

## Deep Dive Findings

{
  "workload": "crm-conversations-scoring-events-sub",
  "category": "transient_traffic_surge",
  "confidence": "high",
  "finding": "Aligned publishing and acknowledgement metrics support transient inflow exceeding consumer throughput. Backlog recovered, but OnCall resolution is not confirmed.",
  "aligned_metrics": {
    "date": "2026-11-23",
    "timezone": "Asia/Kolkata",
    "method": "Sum of five aligned 60-second samples; publish operations and acknowledgements are distinct counters, not exact backlog accounting.",
    "windows": [
      {"ist": "20:25–20:29", "publish_operations": 1492302, "acknowledgements": 1498263},
      {"ist": "20:30–20:34", "publish_operations": 2611094, "acknowledgements": 2402709},
      {"ist": "20:35–20:39", "publish_operations": 3573072, "acknowledgements": 3620855},
      {"ist": "20:40–20:44", "publish_operations": 3825830, "acknowledgements": 3721320}
    ]
  },
  "recovery": {
    "peak_backlog": 97085,
    "peak_time_ist": "20:44",
    "below_threshold_since_ist": "20:46",
    "maximum_backlog_after_recovery": 6846,
    "latest_backlog": 2926,
    "latest_sample_ist": "21:21",
    "oncall_acknowledged": true,
    "oncall_resolved": false
  },
  "causal_assessment": [
    "Publishing exceeded acknowledgements during initial growth and immediately before the backlog peak; acknowledgements remained active.",
    "HPA reached 70 replicas at 20:35, nine minutes before the peak. Recovery cannot be attributed solely to scaling.",
    "The 20:45 publish sample is missing, preventing exact comparison during the sharpest drain minute.",
    "Supplied pod and log evidence argues against a crash or sustained retry storm. Sparse processing errors are not established backlog contributors.",
    "Historical investigation inv-2026-09-30-f02dc591 records the same transient surge pattern; its zero-redelivery findings do not establish zero redelivery today."
  ],
  "log_queries_used": [
    {
      "source": "Supplied Layer 1 findings; existing results reviewed, not rerun",
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"crm-conversations-scoring-events-worker\" AND severity>=ERROR AND jsonPayload.message=\"Error while processing event\" AND timestamp>=\"2026-11-23T14:59:00Z\" AND timestamp<=\"2026-11-23T15:19:00Z\"",
      "finding": "21 errors: 15 contact-not-found, 5 undefined Email errors, 1 datastore timeout; no demonstrated causal link to backlog."
    }
  ],
  "access": "Read-only Cloud Monitoring via existing gcloud authentication and Grafana/OnCall via validated existing session.",
  "limitations": [
    "Missing delivery-attempt fields and dead-letter series prevent claiming zero redeliveries or dead-letter activity.",
    "Specific publisher identity was not traced in this targeted verification."
  ],
  "recommendation": "No immediate capacity change supported. Track recurrence and trace publisher identity if repeated bursts cause sustained threshold breaches.",
  "urls": {
    "alert": "https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IZD4SJB91G3S5",
    "logs": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-scoring-events-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D%22Error%20while%20processing%20event%22;timeRange=2026-11-23T14%3A59%3A00Z%2F2026-11-23T15%3A19%3A00Z?project=highlevel-backend"
  }
}

## Evidence

### Dashboards (incident window)
- [Unacked messages — scoring subscription: 97k spike clears below 10k](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-scoring-events-sub&from=1795445302000&to=1795448718000&viewPanel=1)
- [Acknowledgements — scoring subscription: throughput remains high](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-scoring-events-sub&from=1795445302000&to=1795448718000&viewPanel=3)
- [Nack requests — scoring subscription: no retry storm](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=crm-conversations-scoring-events-sub&from=1795445302000&to=1795448718000&viewPanel=6)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-scoring-events-worker%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D%22Error%20while%20processing%20event%22;timeRange=2026-11-23T14%3A59%3A00Z%2F2026-11-23T15%3A19%3A00Z?project=highlevel-backend)

## Investigation Timeline

**Deep dive passes**: 1
**Worker calls**: 9

**Total worker time**: 777.6s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 16.5s | ✓ |
| grafana-worker | worker | 102.9s | ✓ |
| kubectl-worker | worker | 147.2s | ✓ |
| log-explorer-worker | worker | 302.1s | ✓ |
| triage | worker | 16.1s | ✓ |
| triage_llm_ranking | decision | 32.8s | ✓ |
| deep_dive_pass_1 | worker | 128.2s | ✓ |
| follow_up_check | decision | 12.4s | ✓ |
| cross_validation | worker | 19.4s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IZD4SJB91G3S5|#140747 Pubsub Unacked Messages above 10k >* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/a869c751-a88d-4f06-b984-c4413e22edc3/view?orgId=1|source>*) AlertValues: Doc for this alert: http://platform.docs/alerts/pubsub-unacked-messages/ Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-scoring-events-sub Unacked Messages: 78518 GCP Link: https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-scoring-events-sub?project=highlevel-backend *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IZD4SJB91G3S5|#140747 Pubsub Unacked Messages above 10k >* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/a869c751-a88d-4f06-b984-c4413e22edc3/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-scoring-events-sub Unacked Messages: 78518 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-scoring-events-sub?project=highlevel-backend>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-23T15:53:08.405Z*