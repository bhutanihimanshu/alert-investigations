# PubSub Oldest Unacked Age — crm-conversations-sla-reporting-events-sub — 2026-03-25

**Author:** Himanshu Bhutani | **Status:** Draining (self-recovering)

## Summary

| Field | Value |
|-------|-------|
| Alert | #113552 Pubsub Oldest Unacked Messages age above 30mins |
| Service | crm-conversations-sla-reporting-worker |
| Subscription | crm-conversations-sla-reporting-events-sub |
| Cluster | workers-us-central-production-cluster |
| Fired | 04:11 IST (22:41 UTC) on 2026-03-25 |
| Peak oldest unacked | ~103 min (6,177s) at ~03:35 IST (22:05 UTC) |
| Peak backlog | ~52,343 undelivered messages at ~03:30 IST (22:00 UTC) |
| Duration | Backlog building since ~19:00 IST Mar 24 (13:30 UTC); draining by alert time |
| Impact | SLA reporting data delayed; no user-facing impact (internal analytics pipeline) |

## Root Cause

**Validation failures in `handleOutboundCleared`** — outbound "cleared SLA" messages are missing required `sla.cleared_*` fields (specifically `hasInboundId: false`), causing the worker to throw errors and nack those messages. PubSub retries them (up to 10 attempts with exponential backoff 10s→600s), consuming worker capacity and inflating the backlog. Successfully processed messages are acked normally (sent/ack ratio ~1.0x), but the error rate (~100-280 errors/5min) combined with retry backoff keeps the oldest unacked age elevated.

## What Happened

1. **~19:00 IST Mar 24** — Oldest unacked age began climbing above the ~7 min baseline, suggesting a new class of messages started arriving that the worker couldn't process.
2. **~21:00 IST Mar 24** — Backlog crossed 50k undelivered; oldest unacked age reached ~100 min. Worker ack rate steady at ~5k/5min but insufficient to outpace incoming + retried messages.
3. **04:11 IST Mar 25** — Alert fired (#113552). Oldest unacked at 3,511s (~58 min). Backlog already draining (52k → 35k by alert time).
4. **Post-alert** — Backlog continued draining as failed messages exhausted their 10 delivery attempts and moved to the dead-letter topic (`crm-conversations-sla-reporting-events-dl`).

## Proof

<details>
<summary>[Cloud Monitoring] Oldest unacked age peaked at 103 min (~6,177s) at 03:35 IST</summary>

> **Verify:** `oldest_unacked_message_age` metric shows sustained elevation above 1,800s (30 min threshold) from ~19:00 IST Mar 24 through the alert window.

| Time (IST) | Oldest Unacked (s) | Minutes |
|---|---|---|
| 03:30 (22:00 UTC) | 6,044 | 101 |
| 03:35 (22:05 UTC) | 6,177 | 103 |
| 03:40 (22:10 UTC) | 6,156 | 103 |
| 04:10 (22:40 UTC) | 5,876 | 98 |
| 04:30 (23:00 UTC) | 4,673 | 78 |
| 05:00 (23:30 UTC) | 4,061 | 68 |

[Open in GCP Console](https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-sla-reporting-events-sub?project=highlevel-backend)
</details>

<details>
<summary>[Cloud Monitoring] Backlog draining — 52k → 21k undelivered over 90 min</summary>

> **Verify:** `num_undelivered_messages` shows a steady decline from 52,343 at 03:30 IST to 21,435 at 05:00 IST, confirming workers ARE processing but can't keep up with incoming + retried messages.

| Time (IST) | Undelivered |
|---|---|
| 03:30 (22:00 UTC) | 52,343 |
| 04:10 (22:40 UTC) | 35,483 |
| 04:30 (23:00 UTC) | 22,704 |
| 05:00 (23:30 UTC) | 21,435 |

2-week trend: 0 undelivered through Mar 22, 1,386 on Mar 23, 78,446 peak on Mar 24 — backlog is recent.
</details>

<details>
<summary>[Cloud Monitoring] Sent/ack ratio ~1.0x — no retry amplification</summary>

> **Verify:** `sent_message_count` vs `ack_message_count` ratio stays between 0.94x and 1.06x, ruling out nack-based retry storms.

| Time (IST) | Sent | Acked | Ratio |
|---|---|---|---|
| 04:00 (22:30 UTC) | 6,817 | 6,490 | 1.05x |
| 04:10 (22:40 UTC) | 6,852 | 6,677 | 1.03x |
| 04:35 (23:05 UTC) | 4,757 | 5,067 | 0.94x |
| 04:45 (23:15 UTC) | 4,853 | 4,880 | 0.99x |

</details>

<details>
<summary>[GCP Logs] Consistent ERROR: "Missing required sla.cleared_* fields on outbound message"</summary>

> **Verify:** All ERROR logs from the worker show the same validation failure in `handleOutboundCleared`, with `hasInboundId: false` in metadata.

```
resource.type="k8s_container"
resource.labels.container_name="crm-conversations-sla-reporting-worker"
severity>=ERROR
timestamp>="2026-03-24T22:00:00Z"
timestamp<="2026-03-24T23:30:00Z"
```

Error distribution (30-entry sample):
- 15× `SLA Reporting: batch message failed`
- 15× `SLA Reporting: Missing required sla.cleared_* fields (MB code bug?)`

Error volume: ~100-280 errors per 5 minutes throughout the incident window, peaking at ~22:40 UTC.

Sample log entry metadata:
```json
{
  "hasInboundId": false,
  "hasChannel": true,
  "hasInboundAt": true,
  "hasOverdueAt": true,
  "messageId": "Sb9L80kfGhu4wDnMDrnS"
}
```

[Open in GCP Log Explorer](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-sla-reporting-worker%22%0Aseverity%3E%3DERROR;timeRange=2026-03-24T22:00:00Z%2F2026-03-24T23:30:00Z?project=highlevel-backend)
</details>

<details>
<summary>[Code] Worker nacks failed messages — retry with exponential backoff up to 600s</summary>

> **Verify:** The worker's `processBatch` only adds successfully processed messages to `successAckIds`. Failed messages are NOT added, causing PubSub to retry them (subscription config: min backoff 10s, max backoff 600s, max 10 delivery attempts before dead-letter).

The production-deployed code has `handleOutboundCleared` logic (visible in compiled JS at line 608/696) that validates `sla.cleared_*` fields on outbound messages. When validation fails (`hasInboundId: false`), the message is not acked and PubSub retries it.

Subscription retry config:
- `minimumBackoff: 10s`
- `maximumBackoff: 600s`
- `maxDeliveryAttempts: 10`
- Dead-letter topic: `crm-conversations-sla-reporting-events-dl`

</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| **High** | Fix `handleOutboundCleared` to handle outbound messages missing `inboundId` — either skip gracefully (ack without processing) or populate the field from an alternative source | CRM Conversations team |
| **Medium** | Add the production deployment YAML (`values.crm-conversations-sla-reporting-worker.yaml`) to the `deployments/production/` directory in the repo — currently only staging exists, creating a config drift | CRM Conversations team |
| **Low** | Consider force-acking messages that fail validation after fewer retries (currently retries 10 times with up to 600s backoff before dead-lettering) — validation errors won't self-heal on retry | CRM Conversations team |

## Links

- [Verbose report](report-verbose.md)
- [GCP PubSub Console](https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-sla-reporting-events-sub?project=highlevel-backend)
- [GCP Log Explorer — ERROR logs](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-sla-reporting-worker%22%0Aseverity%3E%3DERROR;timeRange=2026-03-24T22:00:00Z%2F2026-03-24T23:30:00Z?project=highlevel-backend)
