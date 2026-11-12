# Alert Investigation: PubSubUnackedMessages

**Investigation ID**: inv-2026-11-12-eb036bd4
**Workload**: elastic-indexing-conversations-events-sub
**Container**: elastic-indexing-conversations-events-sub
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-12T17:56:32.189Z
**Status**: delivering

## Root Cause

**Summary**: A real backlog developed despite healthy, active consumers and subsequently drained substantially. Elevated deadline expirations and acknowledgement warnings favor exactly-once acknowledgement or lease-management degradation. Forwarder saturation remains a hypothesis requiring incident-specific GCP diagnostics and deployed subscriber evidence.
**Confidence**: low
**Category**: unconfirmed_pubsub_acknowledgement_or_lease_degradation

## Triage

**Primary Anomaly**: none
**Details**: Confirmed backlog anomaly outside the listed categories: 12,013 to 1,399,022 messages, with 9,730 expired ack deadlines per five minutes versus baseline p95 379. All 90 pods were ready with zero restarts; processing and Elasticsearch latency were normal. Backlog fell to 48,546 by 17:57 UTC, indicating recovery underway. Candidates 1–2 partially match healthy consumers and subsequent drain, but a triggering publish surge or lock-induced throughput ceiling is unproven. Missing deliveryAttempt fields prevent excluding poison messages.
**Anomalous**: Yes
**Deep Dive Direction**: YES — Triage remains valid, but evidence favors exactly-once acknowledgement/lease-management degradation over a publish surge or lock contention. Project quota exhaustion is not supported; root cause remains unconfirmed.

Need GCP forwarder allocation, acknowledgement-persistence latency/errors and tuner changes; subscriber ack/modack and lease-extension diagnostics; direct comparison with ticket 341482; and retry-index message identities/errors to explain residual failures.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana metrics | Backlog peaked at 1,399,022; expired ack deadlines reached 9,730 per five minutes versus baseline p95 379. Processing and Elasticsearch latency remained normal. Supports acknowledgement/lease degradation, but does not establish its cause. | ✓ |
| Kubernetes | 90/90 pods were ready with zero restarts and low resource usage. Latest rollout preceded the alert by approximately 4.5 hours. Independently argues against pod disruption or resource exhaustion. | ✓ |
| Application logs | No alert-correlated error spike or explicit acknowledgement failures were found. Sampled lock and lookup warnings do not establish causality; missing deliveryAttempt fields leave retries unresolved. | ✗ |
| Local SDK source | Acknowledgement and modack response errors can be suppressed and retried internally, explaining possible log silence. This establishes a plausible mechanism, not incident-specific occurrence; deployed version equivalence is unverified. | ✓ |
| Deep-dive metric reanalysis | Saved samples contain substantial failure_ack_id warnings. These support acknowledgement-path trouble but reuse monitoring evidence and are not independent corroboration. | ✓ |

### Discrepancies
- No two independent sources establish the complete causal chain from forwarder saturation to acknowledgement failures and backlog growth.
- Ticket 341482 and incident-specific forwarder allocation, persistence latency, and tuner changes were not independently verified.
- Five-minute bucket maxima suggest continuing backlog while one-minute observations show decline to 48,546 by 17:57 UTC; aggregation and observation timing explain the apparent disagreement. Complete clearance is unverified.
- Absolute nack activity was substantial but below baseline; it does not establish an incident-specific retry storm. Missing retry identities prevent excluding residual poison messages.
- Falsification checks weaken pod disruption, processing slowdown, and sustained retry amplification. Publish surge and project quota exhaustion are unsupported; acknowledgement degradation remains unconfirmed.

## Deep Dive Findings

{
  "workload": "elastic-indexing-conversations-events-sub",
  "root_cause": {
    "finding": "Exactly-once acknowledgement or lease-management degradation remains the leading hypothesis. Forwarder saturation is not confirmed.",
    "confidence": "medium",
    "alternatives": "Supplied evidence argues against pod disruption, processing latency regression, sustained retry amplification, and project quota exhaustion. Residual poison messages remain possible."
  },
  "new_findings": [
    {
      "finding": "Completed the previously incomplete retry-log search.",
      "evidence": "Two pages searched for 18:00–18:10 UTC; zero matching entries and no remaining continuation token.",
      "limitation": "Missing deliveryAttempt fields mean this does not exclude retries."
    },
    {
      "finding": "Application log silence does not establish acknowledgement success.",
      "evidence": "Local BatchBaseWorker v2.24 calls msg.ack(). Its installed @google-cloud/pubsub 4.11.0 catches and suppresses acknowledgement response-promise errors on this path. modAck() similarly suppresses response errors.",
      "limitation": "Deployed package equivalence was not verified."
    },
    {
      "finding": "Local SDK retries transient acknowledgement failures internally.",
      "evidence": "message-queues.js applies a ten-minute retry-duration limit; ModAckQueue additionally stops retrying when retryCount reaches three.",
      "interpretation": "This provides a plausible mechanism for hidden acknowledgement failures and lease expiry, but does not prove it occurred during this incident."
    },
    {
      "finding": "Reanalysis of saved pass-2 metrics distinguishes warning reasons.",
      "window_utc": "2026-11-12T17:25:00Z/2026-11-12T18:15:00Z",
      "available_sample_totals": {
        "failure_ack_id": {"ack": 572414, "modack": 183439},
        "invalid_ack": {"ack": 29, "modack": 529},
        "internal": {"ack": 52, "modack": 8}
      },
      "limitation": "These are warning counts from previously collected samples, not unique messages or newly refreshed measurements."
    }
  ],
  "ticket_341482_comparison": {
    "matching_supplied_features": "Healthy consumers, suspended streams, acknowledgement warnings, and subsequent recovery.",
    "verification": "Ticket content was not independently retrieved. Targeted local searches found references in this investigation's state/debug files, not an independent support record.",
    "missing": "Incident-specific forwarder allocation, durable acknowledgement-persistence latency/errors, and tuner changes during 17:25–17:55 UTC."
  },
  "residual_retry_index": {
    "result": "Read-only Elasticsearch search through the existing Grafana datasource API returned HTTP 401.",
    "fallback": "Native browser discovery failed because the browser access host was unavailable.",
    "source_observation": "Local tracking code records conversation id, locationId and deliveryAttempt for failed messages meeting the threshold; it does not record the failure reason.",
    "conclusion": "Message identities and their failure causes remain unresolved."
  },
  "log_queries_used": [
    {
      "window_utc": "2026-11-12T17:44:00Z/2026-11-12T17:54:00Z",
      "container": "conversations-elastic-indexing-worker",
      "filter": "(jsonPayload.message=~\"(?i)TRANSIENT_FAILURE_ACK_ID|UNAVAILABLE|ack.*fail|modack|acknowledg\" OR textPayload=~\"(?i)TRANSIENT_FAILURE_ACK_ID|UNAVAILABLE|ack.*fail|modack|acknowledg\")",
      "result": "Zero matches; pagination complete."
    },
    {
      "window_utc": "2026-11-12T18:00:00Z/2026-11-12T18:10:00Z",
      "container": "conversations-elastic-indexing-worker",
      "filter": "(jsonPayload.message=~\"deliveryAttempt:[2-9]\" OR jsonPayload.deliveryAttempt>1 OR jsonPayload.metadata.meta.deliveryAttempt>1)",
      "result": "Zero matches across two pages; pagination complete."
    }
  ],
  "log_explorer_urls": [],
  "next_evidence_needed": [
    "GCP support diagnostics comparing this incident directly with ticket 341482.",
    "Deployed SDK version and subscriber acknowledgement/modack debug diagnostics.",
    "Authenticated retry-index results followed by identity-specific error correlation."
  ],
  "execution": {
    "access": "Existing gcloud-derived authentication for Logging REST; local source and saved metric artifacts.",
    "delegation": "Subagent creation failed.",
    "production_changes": false,
    "messages_posted": false,
    "orchestrator_launched": false
  }
}

## Evidence

### Dashboards (incident window)
- [Unacked messages — elastic-indexing-conversations-events-sub: backlog surge](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=elastic-indexing-conversations-events-sub&from=1794504391904&to=1794507991904&viewPanel=1)
- [Ack throughput — elastic-indexing-conversations-events-sub: consumers remain active](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=elastic-indexing-conversations-events-sub&from=1794504391904&to=1794507991904&viewPanel=3)
- [Expired ack deadlines — elastic-indexing-conversations-events-sub: timeout surge](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc?orgId=1&var-datasource=LI7N5rfVz&var-projectId=highlevel-backend&var-subscriptionId=elastic-indexing-conversations-events-sub&from=1794504391904&to=1794507991904&viewPanel=7)
- [Batch processing latency — conversations-elastic-indexing-worker: no p95 regression](https://prod.grafana.leadconnectorhq.com/d/df0gi4rll78jka?orgId=1&var-app=conversations-elastic-indexing-worker&from=1794504391904&to=1794507991904&viewPanel=7)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-elastic-indexing-worker%22%0Aseverity%3DWARNING%0AjsonPayload.message%3D~%22Resource%20locked%20error%7CFailed%20to%20acquire%20lock%7CError%20acquiring%20locks%7CFailed%20to%20resolve%20opportunities%22;timeRange=2026-11-12T17%3A47%3A00Z%2F2026-11-12T17%3A57%3A00Z?project=highlevel-backend)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 12

**Total worker time**: 1000.8s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 19.2s | ✓ |
| grafana-worker | worker | 149.5s | ✓ |
| kubectl-worker | worker | 86.9s | ✓ |
| log-explorer-worker | worker | 204.6s | ✓ |
| triage | worker | 14.1s | ✓ |
| triage_llm_ranking | decision | 30.2s | ✓ |
| deep_dive_pass_1 | worker | 236.7s | ✓ |
| follow_up_check | decision | 11.8s | ✓ |
| deep_dive_pass_2 | worker | 104.1s | ✓ |
| follow_up_check | decision | 11.7s | ✓ |
| deep_dive_pass_3 | worker | 114.0s | ✓ |
| cross_validation | worker | 17.9s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IGWT63P6SIGZ2|#139710 Pubsub Unacked Messages above 500k >* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/ecf97300-ce66-4685-b538-0fb55cdc5a75/view?orgId=1|source>*) AlertValues: Doc for this alert: http://platform.docs/alerts/pubsub-unacked-messages/ Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: elastic-indexing-conversations-events-sub Unacked Messages: 1085108 GCP Link: https://console.cloud.google.com/cloudpubsub/subscription/detail/elastic-indexing-conversations-events-sub?project=highlevel-backend *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IGWT63P6SIGZ2|#139710 Pubsub Unacked Messages above 500k >* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/ecf97300-ce66-4685-b538-0fb55cdc5a75/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-messages/> Hey, looks like the Unacked messages in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: elastic-indexing-conversations-events-sub Unacked Messages: 1085108 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/elastic-indexing-conversations-events-sub?project=highlevel-backend>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-12T18:10:06.393Z*