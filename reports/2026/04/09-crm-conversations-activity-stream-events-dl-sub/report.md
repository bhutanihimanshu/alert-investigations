# Pubsub Unacked Messages — crm-conversations-activity-stream-events-dl-sub — 2026-04-09

**Author:** Himanshu Bhutani | **Status:** Open (no DL worker deployed)

## Summary

| Field | Value |
|-------|-------|
| Alert | #115224 Pubsub Unacked Messages above 500 |
| Service | crm-conversations-activity-stream-events-dl-sub |
| Fired | 08:28 IST (02:58 UTC) on 2026-04-09 |
| Duration | Ongoing (no consumer exists for the DL subscription) |
| Impact | 89,535 dead-lettered messages permanently stuck — activity stream events lost for affected contacts |

## Root Cause

The dead-letter subscription `crm-conversations-activity-stream-events-dl-sub` has **no consumer worker deployed** (`crm-conversations-activity-stream-events-dl-worker` does not exist). Messages that exhaust 10 delivery attempts on the parent subscription `crm-conversations-activity-stream-events-sub` are forwarded to the DL topic, but nothing processes them. A traffic surge on 2026-04-08 between 22:30-01:30 IST (17:00-20:00 UTC) caused massive Redis lock contention and expired ack deadlines, dead-lettering ~89,300 messages in ~2.5 hours.

## Proof

<details>
<summary>[Cloud Monitoring] DL sub backlog jumped from 217 to 89,524 in 2.5 hours on Apr 8</summary>

> **Verify:** Undelivered messages were stable at ~200-260 for a week, then spiked from 217 at 22:58 IST to 89,524 by 01:50 IST on Apr 9.

Key data points:
| Time (IST) | Undelivered Messages | Delta |
|---|---|---|
| Apr 8, 22:30 | 217 | baseline |
| Apr 8, 23:28 | 414 | +197 |
| Apr 8, 23:35 | 1,756 | +1,342 |
| Apr 9, 00:15 | 43,092 | +41,336 |
| Apr 9, 01:20 | 88,366 | +45,274 |
| Apr 9, 01:50 | 89,524 | plateaued |

</details>

<details>
<summary>[Kubernetes] No DL worker deployment exists — zero pods processing the DL subscription</summary>

> **Verify:** `kubectl get deployment -n default | grep activity-stream` shows only the parent worker (5 pods), not a DL worker.

```
crm-conversations-activity-stream-worker    5/5    5    5    2y128d
```

No deployment matching `crm-conversations-activity-stream-events-dl-worker` exists.

</details>

<details>
<summary>[Cloud Monitoring] Parent sub had 57,510 expired ack deadlines during the spike</summary>

> **Verify:** Expired ack deadlines peaked at 12,719 in a single 5-min window at 23:40 IST. Zero nacks — workers were processing but too slowly, causing ack deadline expiry (600s).

Peak expired ack deadline windows:
| Time (IST) | Expired Deadlines |
|---|---|
| 23:40 (Apr 8) | 12,719 |
| 23:10 (Apr 8) | 5,507 |
| 00:20 (Apr 9) | 4,464 |
| 00:25 (Apr 9) | 3,716 |

Total: 57,510 expired deadlines. Zero nacks — messages were held too long by lock contention.

</details>

<details>
<summary>[GCP Logs] 70,901 Redis lock conflict warnings during the spike window</summary>

> **Verify:** Cloud Monitoring `log_entry_count` for WARNING severity shows 70,901 entries in a 15-minute burst at 23:30-23:40 IST.

```
resource.type="k8s_container"
resource.labels.container_name="crm-conversations-activity-stream-worker"
jsonPayload.message=~"Redis lock conflict"
severity>=WARNING
```

ERROR-level logs show `Failed to acquire lock for contactId` and `error connecting redis {"errno":-111,"code":"ECONNREFUSED","syscall":"connect","address":"127.0.0.1","port":6379}`.

</details>

<details>
<summary>[Grafana] Workers Health Overview — DL sub backlog visualization</summary>

> **Verify:** The DL subscription shows a flat high line at ~89k undelivered messages.

![Workers Health Overview](screenshots/001-workers-health-overview-unacked-messages-dl-sub-highlighted.png)
[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc/conversations-workers-health-overview?orgId=1&var-subscriptionId=crm-conversations-activity-stream-events-dl-sub&from=1775664000000&to=1775707200000)

</details>

<details>
<summary>[Grafana] Parent subscription — healthy ack rate, backlog peaked at 18,801</summary>

> **Verify:** Parent subscription was processing (2.9M acks in the window) but backlog grew temporarily to 18,801 during the lock conflict storm.

![Worker Detailed View — Parent Sub](screenshots/003-worker-detailed-view-parent-subscription-activity-stream-eve.png)
[Open in Grafana](https://prod.grafana.leadconnectorhq.com/d/a04e5483-eb8c-47ef-8198-30147926964c/worker-detailed-view?orgId=1&var-subscriptionId=crm-conversations-activity-stream-events-sub&from=1775664000000&to=1775707200000)

</details>

## Action Items

| Priority | Action | Owner |
|----------|--------|-------|
| **High** | Deploy a DL worker for `crm-conversations-activity-stream-events-dl-sub` or purge the 89k messages and disable the dead-letter policy if DL processing is not needed | Core team / CRM Conversations |
| **Medium** | Investigate the traffic surge trigger on Apr 8 17:00-20:00 UTC — check events worker for bulk operations that generated the lock conflicts | Core team |
| **Medium** | Review Redis lock retry configuration for this worker — high retry counts amplify slot-blocking during contention | CRM Conversations |
| **Low** | Add monitoring for DL subscription backlogs with a separate alert threshold | Platform |

## Links

- [Verbose report](report-verbose.md)
- [Workers Health Overview — DL Sub](https://prod.grafana.leadconnectorhq.com/d/ffex8olsxa9kwc/conversations-workers-health-overview?orgId=1&var-subscriptionId=crm-conversations-activity-stream-events-dl-sub&from=1775664000000&to=1775707200000)
- [Worker Detailed View — Parent Sub](https://prod.grafana.leadconnectorhq.com/d/a04e5483-eb8c-47ef-8198-30147926964c/worker-detailed-view?orgId=1&var-subscriptionId=crm-conversations-activity-stream-events-sub&from=1775664000000&to=1775707200000)
- [GCP PubSub Console — DL Sub](https://console.cloud.google.com/cloudpubsub/subscription/detail/crm-conversations-activity-stream-events-dl-sub?project=highlevel-backend)
- [Grafana OnCall Alert Group](https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IE149FU7B1WJ7)
