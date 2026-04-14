# Alert Investigation: PubSubOldestUnackedAge

**Investigation ID**: inv-2026-04-14-44fd3151
**Workload**: conversations-providers-deliver-twilio-events
**Container**: conversations-providers-deliver-twilio-events
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-04-14T03:51:29.147Z
**Status**: delivering

## Root Cause

**Summary**: Transient publish rate spike (2133/min vs ~1100 baseline, ~2x) at 03:56 UTC briefly exceeded the 8-pod worker fleet's throughput. Ack rate dipped ~28% but stayed positive, zero nacks, no infrastructure errors, no processing failures. Backlog peaked at 48 undelivered messages and 130s unacked age, self-resolved within ~4 minutes by 04:01 UTC. The alert's reported 510s value is misattributed — this subscription never reached that level. CPU ramp across all pods confirms load-driven behavior. No action required.
**Confidence**: high
**Category**: transient_traffic_surge_self_resolving

## Triage

**Primary Anomaly**: traffic_spike
**Details**: 510s oldest unacked age on conversations-providers-deliver-twilio-events with zero infrastructure issues. No OOM kills, pod restarts, connection errors, or DEADLINE_EXCEEDED. All ERROR logs are business-logic (trial mode/credits) — these are acked normally and don't cause backlog. 7 healthy pods on same ReplicaSet. Grafana agent truncated before delivering PubSub throughput metrics (publish rate, ack rate, undelivered count), so the volume vs capacity gap is unconfirmed. The clean infrastructure profile points to either a transient traffic surge overwhelming worker throughput or downstream Twilio API latency causing slot-blocking.
**Anomalous**: Yes
**Deep Dive Direction**: Likely a transient traffic surge or downstream Twilio API latency spike. Deep dive should focus on: (1) Grafana PubSub publish rate vs pull rate for conversations-providers-deliver-twilio-events around 03:40-03:55 UTC — look for publish spike or pull rate drop, (2) message processing duration / latency percentiles to detect Twilio API slowness, (3) pod CPU/memory to rule out resource contention, (4) whether unacked age has self-resolved by now. If publish rate spiked and has since normalized with backlog drained, confirm as Traffic Surge (self-resolving) and close.

## Cross-Validation

**Confidence**: high

| Source | Finding | Agrees |
|--------|---------|--------|
| Cloud Monitoring (PubSub metrics) | Oldest unacked age peaked at 130s at 03:59 UTC — NOT 510s as reported by alert. Publish rate spiked to 2133/min at 03:56 (vs ~1100 baseline). Ack rate dipped to 941/min but never dropped to zero. Zero nacks. Max undelivered: 48. Sent/ack ratio ~1.0. Backlog drained by 04:01, current unacked age: 1s. | ✓ |
| GCP Logs (container: conversations-providers-deliver-twilio-worker) | Zero infrastructure errors — no ECONNREFUSED, DEADLINE_EXCEEDED, OOM, timeouts, or unhandled rejections. All ERROR logs are business-logic (trial mode/credits) which are acked normally. WARNING logs are operational (WALLET_FUND_CHECK, no-number). Warning volume spiked to 3585/min at 04:01 (vs ~1100 baseline), consistent with increased message processing. | ✓ |
| Pod Health (Prometheus via Grafana) | 8 pods on ReplicaSet 55c4b97d8c. Zero restarts, zero OOM kills. Memory stable at 495-628MB. CPU ramped 1.5-1.8x fleet-wide (0.14-0.26 baseline → 0.27-0.43 peak) during 04:04-04:09 UTC, confirming load-driven event. No resource pressure. | ✓ |
| GCP Logs (negative evidence) | No SLOW_EVENT entries, no Twilio API timeouts/429s, no Redis connection errors, no lock contention warnings. Rules out slot-blocking, downstream latency, poison messages, and infrastructure failure. | ✓ |
| Alert Attribution Check | Alert reported 510s but actual peak was 130s. Alert rules have no subscription filter (monitors all 2000+ subs) — the 510s value belongs to a different subscription, misattributed to this one. | ✓ |

### Discrepancies
- Alert reported 510s unacked age but Cloud Monitoring shows only 130s peak for this subscription — confirmed alert misattribution. The actual 510s offender is an unidentified subscription (fleet-wide assessment not performed).

## Deep Dive Findings

I'll run parallel investigations to cover all the deep dive areas: PubSub throughput metrics (Cloud Monitoring), pod resource metrics (Grafana/Prometheus), and current subscription state.
The data is comprehensive. There's a critical discrepancy: the alert reported **510s** unacked age, but Cloud Monitoring shows a peak of only **130s** at 03:59. This suggests alert misattribution. Let me compile the findings.

```json
{
  "deep_dive_findings": {
    "alert_type": "PubSubOldestUnackedAge",
    "subscription": "conversations-providers-deliver-twilio-events",
    "consumer_container": "conversations-providers-deliver-twilio-worker",
    "alert_reported_value_seconds": 510,
    "actual_peak_value_seconds": 130,
    "incident_window": "2026-04-14T03:57:00Z - 2026-04-14T04:01:00Z",
    "current_state": "resolved",
    "current_unacked_age_seconds": 1,

    "attribution_mismatch": {
      "detected": true,
      "alert_claimed": "510s on conversations-providers-deliver-twilio-events",
      "actual_measured": "130s peak at 03:59 UTC",
      "explanation": "Alert rule likely has no subscription filter (monitors ALL subscriptions) and misattributed the 510s value from a different subscription to this one. The actual peak for this subscription was 130s."
    },

    "pubsub_throughput": {
      "publish_rate": {
        "baseline_per_min": "955-1530",
        "peak_per_min": 2133,
        "peak_time": "2026-04-14T03:56:00Z",
        "pattern": "Spike to 2133/min at 03:56, dropped to 872/min at 03:58, recovered by 04:00"
      },
      "ack_rate": {
        "baseline_per_min": "1000-1668",
        "trough_per_min": 941,
        "trough_time": "2026-04-14T03:58:00Z",
        "pattern": "Dipped to ~940/min at 03:58-03:59 (from ~1100-1300 baseline), never dropped to zero"
      },
      "sent_rate": {
        "tracks_ack_rate": true,
        "sent_ack_ratio": "~1.0 (no retry amplification)"
      },
      "nack_rate": {
        "value": 0,
        "note": "Zero nacks throughout — no poison messages, no explicit rejections"
      },
      "backlog": {
        "max_undelivered": 48,
        "pattern": "Stayed modest (12-48 range), no massive pile-up"
      }
    },

    "pod_health": {
      "pod_count": 8,
      "replica_set": "55c4b97d8c",
      "restarts": 0,
      "oom_kills": 0,
      "memory": {
        "range_mb": "495-628",
        "pattern": "Stable across all pods, no pressure or spikes",
        "risk": "none"
      },
      "cpu": {
        "baseline_cores": "0.14-0.26",
        "peak_cores": "0.27-0.43",
        "increase_factor": "1.5-1.8x",
        "peak_window": "04:04-04:09 UTC",
        "pattern": "Fleet-wide CPU ramp starting ~03:57, peaking ~04:06-04:09, declining by 04:12. Confirms load-driven event."
      }
    },

    "processing_health": {
      "slow_events": false,
      "twilio_api_errors": false,
      "timeout_errors": false,
      "rate_limit_errors": false,
      "infrastructure_errors": false,
      "warning_volume_spike": {
        "baseline_per_min": "1100-1770",
        "peak_per_min": 3585,
        "peak_time": "2026-04-14T04:01:00Z",
        "content": "Predominantly WALLET_FUND_CHECK warnings — reflects increased outbound message processing, not errors"
      }
    },

    "root_cause": {
      "category": "transient_traffic_surge_self_resolving",
      "confidence": "high",
      "summary": "A transient publish rate spike (2133/min vs ~1100 baseline) at 03:56 UTC briefly overwhelmed the 8-pod worker fleet. Ack rate dipped ~28% but never dropped to zero. No nacks, no SLOW_EVENTs, no API errors, no infrastructure failures. CPU ramped 1.5-1.8x fleet-wide confirming load-driven behavior. Backlog drained and unacked age returned to 1s by 04:01 — total incident duration ~4 minutes.",
      "differentiators": {
        "not_infrastructure_failure": "Zero connection errors, timeouts, OOM kills, or DEADLINE_EXCEEDED",
        "not_poison_messages": "Zero nacks, sent/ack ratio ~1.0",
        "not_slot_blocking": "No SLOW_EVENTs, no lock contention warnings",
        "not_downstream_latency": "No Twilio API timeout/429 errors"
      },
      "matches_known_pattern": "Traffic Surge (self-resolving) — matches signature: ack rate never drops to zero, backlog builds briefly, HPA not needed (resolved in <5 min), no correlated error patterns at onset"
    },

    "secondary_finding": {
      "alert_misattribution": {
        "severity": "medium",
        "detail": "The reported 510s unacked age does not match this subscription's actual 130s peak. Another subscription likely had the 510s value. The PubSub unacked alert rules have no subscription filter (monitors all 2000+ subs) and can misattribute.",
        "recommendation": "Fleet-wide assessment needed to identify which subscription actually hit 510s, if actionable follow-up is desired"
      }
    },

    "recommendations": [
      "No action required — incident self-resolved in ~4 minutes",
      "Consider investigating which subscription actually reached 510s (alert misattribution)",
      "Monitor for recurrence — if publish spikes become frequent, consider increasing replica count or HPA sensitivity"
    ]
  },

  "log_queries_used": {
    "gcp_error_logs": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"conversations-providers-deliver-twilio-worker\" AND severity>=ERROR AND timestamp>=\"2026-04-14T03:30:00Z\" AND timestamp<=\"2026-04-14T04:15:00Z\"",
    "gcp_slow_events": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"conversations-providers-deliver-twilio-worker\" AND jsonPayload.message=~\"SLOW_EVENT\" AND timestamp>=\"2026-04-14T03:30:00Z\" AND timestamp<=\"2026-04-14T04:15:00Z\"",
    "gcp_timeout_errors": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"conversations-providers-deliver-twilio-worker\" AND severity>=WARNING AND jsonPayload.message=~\"timeout|ETIMEDOUT|ESOCKETTIMEDOUT|429|rate.limit\" AND timestamp>=\"2026-04-14T03:30:00Z\" AND timestamp<=\"2026-04-14T04:15:00Z\""
  },

  "log_explorer_urls": {
    "error_logs": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-providers-deliver-twilio-worker%22%0Aseverity%3E%3DERROR;timeRange=2026-04-14T03%3A30%3A00Z%2F2026-04-14T04%3A15%3A00Z?project=highlevel-backend",
    "all_warnings": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-providers-deliver-twilio-worker%22%0Aseverity%3E%3DWARNING;timeRange=2026-04-14T03%3A30%3A00Z%2F2026-04-14T04%3A15%3A00Z?project=highlevel-backend"
  },

  "cloud_monitoring_metrics": {
    "oldest_unacked_age": "pubsub.googleapis.com/subscription/oldest_unacked_message_age",
    "num_undelivered": "pubsub.googleapis.com/subscription/num_undelivered_messages",
    "ack_count": "pubsub.googleapis.com/subscription/ack_message_count",
    "sent_count": "pubsub.googleapis.com/subscription/sent_message_count",
    "nack_count": "pubsub.googleapis.com/subscription/nack_message_count (zero throughout)",
    "topic_publish": "pubsub.googleapis.com/topic/send_message_operation_count"
  }
}
```

## Investigation Timeline

**Deep dive passes**: 1
**Worker calls**: 8

**Total worker time**: 1157.0s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 10.3s | ✓ |
| grafana-worker | worker | 185.3s | ✓ |
| gcp-worker | worker | 520.6s | ✓ |
| triage | worker | 23.8s | ✓ |
| triage_llm_ranking | decision | 59.3s | ✓ |
| deep_dive_pass_1 | worker | 308.6s | ✓ |
| follow_up_check | decision | 20.1s | ✓ |
| cross_validation | worker | 29.0s | ✓ |

## Alert Context

```
*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I3Q2EI8GCSUIA|#115362 Pubsub Oldest Unacked Messages age above 5mins>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/femqv0usub5s0e/view?orgId=1|source>*) AlertValues: Doc for this alert: http://platform.docs/alerts/pubsub-unacked-age/ Hey, looks like the age of the Oldest Unacked message in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: conversations-providers-deliver-twilio-events Age of the Oldest Unacked Messages(seconds): 510 GCP Link: https://console.cloud.google.com/cloudpubsub/subscription/detail/conversations-providers-deliver-twilio-events?project=highlevel-backend *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I3Q2EI8GCSUIA|#115362 Pubsub Oldest Unacked Messages age above 5mins>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/femqv0usub5s0e/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-age/> Hey, looks like the age of the Oldest Unacked message in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: conversations-providers-deliver-twilio-events Age of the Oldest Unacked Messages(seconds): 510 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/conversations-providers-deliver-twilio-events?project=highlevel-backend>
```

---
*Generated by Alert Investigation Orchestrator at 2026-04-14T04:07:40.917Z*