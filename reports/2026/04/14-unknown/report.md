# Alert Investigation: PubSubOldestUnackedAge

**Investigation ID**: inv-2026-04-14-9700b6fe
**Workload**: crm-conversations-mailgun-inbound-events-sub
**Container**: crm-conversations-mailgun-inbound-events-sub
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-04-14T08:55:08.520Z
**Status**: delivering

## Root Cause

**Summary**: Bad deployment (ReplicaSet 55d4656cb) at 08:09 UTC introduced a Firestore double-initialization bug at main.js:61. All new pods crash deterministically on startup within ~10s. All 3 old pods (ReplicaSet 5945fb6d8b) were killed during the rolling update between 08:12-08:15 UTC. Zero healthy consumers from 08:16 onward → PubSub messages accumulate linearly → oldest unacked age reached 2,124s when alert fired at 08:55 UTC. No recovery observed — pods still in CrashLoopBackOff at 09:07+. Recommended action: rollback to ReplicaSet 5945fb6d8b and fix the Firestore settings() call ordering before redeploying.
**Confidence**: high
**Category**: bad_deployment_crash_loop

## Triage

**Primary Anomaly**: error_spike
**Details**: Bad deployment (ReplicaSet 55d4656cb) rolled out at ~08:09 UTC. All 4 new pods crash immediately on startup with 'Firestore has already been initialized. You can only call settings() once' (main.js:61). Old pods (ReplicaSet 5945fb6d8b) were killed during rollout, leaving zero healthy consumers. CrashLoopBackOff since 08:12 UTC → PubSub messages accumulate unacked → alert fires at 08:55 UTC with 2124s (~35min) oldest unacked age. No SLOW_EVENT, no deliveryAttempt retries, no infrastructure errors (Redis/Mongo/OOM) — purely an application-level Firestore double-initialization bug in the new build.
**Anomalous**: Yes
**Deep Dive Direction**: Validate whether this matches known pattern: "Deployment-Triggered Alert". Investigate using Layer 1 evidence — do NOT assume the match is correct. Check for alternative explanations.

## Cross-Validation

**Confidence**: high

| Source | Finding | Agrees |
|--------|---------|--------|
| GCP kubectl + pod events | New ReplicaSet 55d4656cb rolled out at 08:09 UTC. All 4 new pods crash immediately on startup with 'Firestore has already been initialized. You can only call settings() once' (main.js:61). Old pods (5945fb6d8b) killed during rolling update. CrashLoopBackOff from 08:13 onward. Zero healthy consumers from ~08:16 UTC. | ✓ |
| GCP application logs | ERROR logs confirm crash at main.js:61 → @google-cloud/firestore/build/src/index.js:554. No SLOW_EVENT, no deliveryAttempt > 1, no infrastructure errors (Redis/Mongo/OOM). Crash is deterministic on every startup attempt. | ✓ |
| Cloud Monitoring PubSub metrics | Ack rate was 470-540/min before deployment, dropped to 0/min from 08:16 onward. Oldest unacked age climbed linearly at ~62s/min from 08:16, reaching 2,436s at alert time. No nack spike (no consumer alive to nack). | ✓ |
| Deep dive validation | Confirmed all 3 old pods killed (p725d at 08:12:48, ncqhj at 08:12:55, 6p2kv at 08:14:59). Disproved triage hypothesis that ncqhj/6p2kv might still be running. Sharp transition from healthy to zero-consumer at 08:15-08:16. No competing signals. | ✓ |
| Negative evidence checks | No OOMKilled, no ECONNREFUSED, no ETIMEDOUT, no MongoDB timeout, no SLOW_EVENT, no poison messages, no retry amplification. All alternative explanations ruled out. | ✓ |

### Discrepancies
- Triage initially hypothesized old pods ncqhj and 6p2kv might still be running — deep dive confirmed both were killed at 08:12:55 and 08:14:59 respectively
- Grafana agent hit session expiry and max turns — PubSub metric validation was completed via deep dive instead

## Deep Dive Findings



The deep dive validation is complete. Here are the structured findings:

```json
{
  "deep_dive_result": {
    "root_cause": "BAD_DEPLOYMENT_CRASH_LOOP",
    "known_pattern_match": "Deployment-Triggered Alert",
    "pattern_match_validated": true,
    "confidence": "high",
    "summary": "New deployment (ReplicaSet 55d4656cb) rolled out at 08:09 UTC with a Firestore double-initialization bug in main.js:61. All 3 new pods crash immediately on startup (~10s after container start). All 3 old pods (ReplicaSet 5945fb6d8b) were killed during rolling update. Zero healthy consumers from 08:16 UTC onward → PubSub messages accumulate linearly → alert fires at 08:55 UTC.",
    "causal_chain": [
      "08:09:44 — New ReplicaSet 55d4656cb begins rollout",
      "08:12:47-54 — New pods start main container",
      "08:12:48-14:59 — All 3 old pods killed during rolling update (p725d, ncqhj, 6p2kv)",
      "08:12:57 — First new pod crashes: 'Firestore has already been initialized. You can only call settings() once' at main.js:61",
      "08:13:16-19 — All new pods enter CrashLoopBackOff",
      "08:16:00 — Zero acks. Zero consumers. Unacked age begins linear climb (~62s/min)",
      "08:55:00 — Alert fires at ~2,436s oldest unacked age"
    ],
    "evidence": {
      "deployment_confirmed": true,
      "all_old_pods_killed": true,
      "old_pods_killed": ["5945fb6d8b-p725d (08:12:48)", "5945fb6d8b-ncqhj (08:12:55)", "5945fb6d8b-6p2kv (08:14:59)"],
      "all_new_pods_crash": true,
      "new_pods_crashing": ["55d4656cb-6x8tg", "55d4656cb-tgf6w", "55d4656cb-w8xmd"],
      "crash_error": "Error: Firestore has already been initialized. You can only call settings() once",
      "crash_location": "/app/dist/apps/conversations/main.js:61:19 → @google-cloud/firestore/build/src/index.js:554",
      "ack_rate_before": "470-540/min (healthy)",
      "ack_rate_after": "0/min from 08:16 onward (zero consumers)",
      "unacked_age_growth": "Linear ~62s/min from 08:16, reached 2,436s at alert time, 3,935s by 09:20",
      "no_recovery_observed": true
    },
    "alternative_explanations_checked": {
      "infrastructure_errors": "None — no ECONNREFUSED, ETIMEDOUT, MongoDB timeout, OOM",
      "slow_events": "None — no SLOW_EVENT logs (workers never ran long enough)",
      "poison_messages": "None — no deliveryAttempt > 1 (no consumer alive to process)",
      "pubsub_side_issue": "Ruled out — acks were healthy (470-540/min) right up until pods died",
      "partial_outage": "Ruled out — ALL old pods killed, ALL new pods crash deterministically",
      "gradual_degradation": "Ruled out — sharp transition from healthy to zero-consumer at 08:15-08:16"
    },
    "disproof_attempts": {
      "old_pods_survived": "DISPROVEN — triage hypothesized ncqhj and 6p2kv might still be running; logs confirm both were killed at 08:12:55 and 08:14:59 respectively",
      "some_acks_after_crash": "DISPROVEN — ack count drops to 143 at 08:15 (partial draining of 6p2kv), then zero from 08:16"
    },
    "log_queries_used": [
      "resource.type=\"k8s_pod\" AND resource.labels.pod_name=~\"crm-conversations-mailgun-inbound-worker-55d4656cb\" (deployment events)",
      "resource.type=\"k8s_container\" AND textPayload=~\"Firestore has already been initialized\" AND severity>=ERROR (crash logs)",
      "resource.type=\"k8s_pod\" AND resource.labels.pod_name=~\"crm-conversations-mailgun-inbound-worker-5945fb6d8b\" AND jsonPayload.reason=~\"Killing\" (old pod termination)",
      "Cloud Monitoring: ack_message_count for crm-conversations-mailgun-inbound-events-sub",
      "Cloud Monitoring: oldest_unacked_message_age for crm-conversations-mailgun-inbound-events-sub"
    ],
    "log_explorer_urls": {
      "firestore_crash_errors": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-mailgun-inbound-worker%22%0AtextPayload%3D~%22Firestore%20has%20already%20been%20initialized%22%0Aseverity%3E%3DERROR;timeRange=2026-04-14T08%3A10%3A00Z%2F2026-04-14T09%3A10%3A00Z?project=highlevel-backend",
      "pod_crashloop_events": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.pod_name%3D~%22crm-conversations-mailgun-inbound-worker-55d4656cb%22%0AjsonPayload.reason%3D%22BackOff%22;timeRange=2026-04-14T08%3A10%3A00Z%2F2026-04-14T09%3A10%3A00Z?project=highlevel-backend",
      "old_pod_termination": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_pod%22%0Aresource.labels.pod_name%3D~%22crm-conversations-mailgun-inbound-worker-5945fb6d8b%22%0AjsonPayload.reason%3D%22Killing%22;timeRange=2026-04-14T08%3A10%3A00Z%2F2026-04-14T09%3A30%3A00Z?project=highlevel-backend"
    },
    "recommended_action": "Rollback deployment to previous ReplicaSet 5945fb6d8b. The new image (sha256:78a2d31d) has a Firestore double-initialization bug at main.js:61 — settings() called after Firestore already initialized. Fix the code before re-deploying."
  }
}
```

## Investigation Timeline

**Deep dive passes**: 1
**Worker calls**: 8

**Total worker time**: 1882.9s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 13.0s | ✓ |
| grafana-worker | worker | 208.5s | ✓ |
| gcp-worker | worker | 1187.8s | ✓ |
| triage | worker | 17.9s | ✓ |
| triage_llm_ranking | decision | 56.0s | ✓ |
| deep_dive_pass_1 | worker | 356.9s | ✓ |
| follow_up_check | decision | 17.8s | ✓ |
| cross_validation | worker | 24.9s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IRN6RMXLRQ41F|#115417 Pubsub Oldest Unacked Messages age above 30mins>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/demqv0uteacjkf/view?orgId=1|source>*) AlertValues: Doc for this alert: http://platform.docs/alerts/pubsub-unacked-age/ Hey, looks like the age of the Oldest Unacked message in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-mailgun-inbound-events-sub Age of the Oldest Unacked Messages(seconds): 2124 GCP Link: https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-mailgun-inbound-events-sub?project=highlevel-backend *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IRN6RMXLRQ41F|#115417 Pubsub Oldest Unacked Messages age above 30mins>* via CRM-conversations (*<https://prod.grafana.leadconnectorhq.com/alerting/grafana/demqv0uteacjkf/view?orgId=1|source>*) AlertValues: Doc for this alert: <http://platform.docs/alerts/pubsub-unacked-age/> Hey, looks like the age of the Oldest Unacked message in the below subscription has gone above the configured threshold. :chart_with_upwards_trend: Subscription: crm-conversations-mailgun-inbound-events-sub Age of the Oldest Unacked Messages(seconds): 2124 GCP Link: <https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-mailgun-inbound-events-sub?project=highlevel-backend>
```

---
*Generated by Alert Investigation Orchestrator at 2026-04-14T09:23:03.094Z*