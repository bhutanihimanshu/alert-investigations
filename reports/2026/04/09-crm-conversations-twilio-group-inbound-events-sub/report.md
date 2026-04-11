# Pubsub Oldest Unacked Age Investigation -- crm-conversations-twilio-group-inbound-events-sub -- 2026-04-09

**Author:** Himanshu Bhutani | **Status:** Self-resolved (messages went to dead letter)

## Summary

| Field | Value |
|-------|-------|
| Alert | #115225 Pubsub Oldest Unacked Messages age above 30mins |
| Service | crm-conversations-twilio-group-inbound-events-worker |
| Subscription | crm-conversations-twilio-group-inbound-events-sub |
| Fired | 10:14 IST (04:44 UTC) on 2026-04-09 |
| Duration | ~2 hours (04:07 -- 06:00 UTC) |
| Impact | 4 messages went to dead letter; low-volume subscription (~2.8 msgs/min) |

## Root Cause

A single poison message (ID: `IM9987a193f7a3493aa7750aa868385486`) failed repeatedly with `HttpException: Message not found` at `handleDelieveryUpdate`, causing a nack-retry cycle (804 nacks vs 415 successful acks in the window). The message was never acked, its age grew linearly to 3,684s (61 min), and it eventually went to dead letter along with 3 other stuck messages. The subscription self-recovered after dead-lettering.

**Secondary issue:** The worker has a permanent Redis ECONNREFUSED condition (~10,788 errors/hour, 24/7) because no Redis sidecar is configured in the deployment values -- the worker defaults to `127.0.0.1:6379`. This does NOT cause message processing failures by itself (the code catches the error) but represents a configuration debt.

## Proof

<details>
<summary>[Cloud Monitoring] Oldest unacked age: linear growth 0s -> 3,684s from 09:37 to 10:38 IST</summary>

> **Verify:** The oldest unacked message age grew linearly at ~1s/s from 09:37 IST (04:07 UTC) to a peak of 3,684s at 10:38 IST (05:08 UTC). Linear growth = a message stuck and never processed. No plateau or step pattern (which would indicate processing stalls). After dead-lettering, the value returned to 0.

```
Time (UTC)     Age(s)
04:07          3s
04:08          64s
04:10          184s
04:14          424s    <-- oldest unacked starts exceeding thresholds
04:30          1,384s
04:44          2,104s  <-- ALERT FIRES
05:08          3,684s  <-- PEAK (then dead-lettered, recovery)
```

Cloud Monitoring query: `pubsub.googleapis.com/subscription/oldest_unacked_message_age` for `crm-conversations-twilio-group-inbound-events-sub`
</details>

<details>
<summary>[Grafana] Worker Detailed View: 804 nacks vs 415 sent, avg ack latency 266ms</summary>

> **Verify:** The Subscription Stats gauge row shows Total Nack Requests = 804 (red), which is nearly 2x the Total Sent Messages = 415. This confirms a poison message being nacked and redelivered repeatedly. The Oldest Unacked Message chart shows the linear rise to ~1.11 hours.

![Worker Detailed View](screenshots/001-worker-detailed-view-subscription-stats-gauges-sent-nack-exp.png)

[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a04e5483-eb8c-47ef-8198-30147926964c/worker-detailed-view?orgId=1&var-subscriptionId=crm-conversations-twilio-group-inbound-events-sub&from=1775705400000&to=1775714400000)
</details>

<details>
<summary>[GCP Logs] HttpException: Message not found for id:IM9987a193f7a3493aa7750aa868385486 -- 100+ retries starting 09:35 IST</summary>

> **Verify:** A single message ID (`IM9987a193f7a3493aa7750aa868385486`) appears 100+ times in ERROR logs. The error occurs at `handleDelieveryUpdate` (line 788 in the worker). Every retry fails with the same exception, and the message is nacked back to PubSub, creating a retry storm.

```
resource.type="k8s_container"
resource.labels.container_name="crm-conversations-twilio-group-inbound-events-worker"
severity>=ERROR
-jsonPayload.message=~"error connecting redis"
-jsonPayload.message=~"reconnecting to redis"
jsonPayload.message=~"HttpException"
timestamp>="2026-04-09T04:00:00Z"
timestamp<="2026-04-09T05:30:00Z"
```

First error: `2026-04-09T04:05:40Z` (09:35:40 IST)
Stack trace: `HttpException: Message not found for id:IM9987a193f7a3493aa7750aa868385486 at handleDelieveryUpdate (crm-conversations-twilio-group-inbound-events-worker.js:788)`
</details>

<details>
<summary>[Cloud Monitoring] Only 4 undelivered messages -- low-volume subscription, not a backlog crisis</summary>

> **Verify:** `num_undelivered_messages` peaked at just 5. This is NOT a high-volume backlog. The alert fired because a few messages were stuck for a long time, not because many messages were queued.

```
Time (UTC)     Undelivered
03:30-04:05    0
04:10-05:05    4
05:08          0 (recovered after dead-lettering)
```

Dead letter count: 4 messages total
</details>

<details>
<summary>[Cloud Monitoring + Grafana] Pods stable at 3, zero restarts -- worker NOT down</summary>

> **Verify:** The earlier Slack thread investigation stated "consumer workload is completely down and has no running pods." This is incorrect. Pod count was stable at 3 throughout, with zero restarts. The worker WAS running and processing messages -- it just couldn't process this specific poison message.

Pod restarts: 0 (confirmed via `kube_pod_container_status_restarts_total`)
Pod count: 3 (stable, confirmed via `kube_pod_info`)
</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| **High** | Fix the poison message handling: `handleDelieveryUpdate` should catch `HttpException: Message not found` and ack the message (set `success = true`) instead of letting it nack. A deleted/missing message will never succeed on retry. | CRM Conversations |
| **Medium** | Add `REDIS_HOST` env var to the deployment values YAML, or configure a Redis sidecar. The persistent Redis ECONNREFUSED on localhost is a config debt producing ~10.8k errors/hour of noise. | CRM Conversations |
| **Low** | Consider adding a force-ack after N retries for this worker to prevent future poison message scenarios. | CRM Conversations |

## Links

- [Verbose report](report-verbose.md)
- [Worker Detailed View](https://prod.grafana.leadconnectorhq.com/d/a04e5483-eb8c-47ef-8198-30147926964c/worker-detailed-view?orgId=1&var-subscriptionId=crm-conversations-twilio-group-inbound-events-sub&from=1775705400000&to=1775714400000)
- [GCP PubSub Console](https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-twilio-group-inbound-events-sub?project=highlevel-backend)
- [Grafana OnCall Alert](https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IHUSNYCJ5SSNG)
