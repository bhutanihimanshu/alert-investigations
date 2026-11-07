> ⚠️ **PARTIAL INVESTIGATION** — the orchestrator hit its critical-path budget and force-delivered before completing all phases.
> Budget hit in state: `targeted_deep_dive` at 2026-11-07T12:43:33.387Z.
> Critical-path duration: 3479983 ms.
> Treat conclusions below as preliminary — workers may have been mid-query when aborted.

---
# Alert Investigation: CloudTasksQueueDepth

**Investigation ID**: inv-2026-11-07-f6c94ce3
**Workload**: conversations-messages-schedule-events
**Container**: conversations-messages-schedule-events
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-07T10:33:30.779Z
**Status**: delivering

## Triage

**Primary Anomaly**: none
**Details**: Queue depth reached 409671, 2.4% above threshold and 25.2% above its seven-day mean, with sustained growth. No creation surge or elevated execution drift was established. Depth includes future scheduled tasks, so overdue processing remains unconfirmed. Pod and log checks used the queue name rather than a verified serving workload; empty results do not establish health. The historical slot-blocking candidate has no supporting evidence.
**Anomalous**: Yes
**Deep Dive Direction**: YES — The triage direction remains correct: distinguish future scheduled inventory from overdue processing. Creations exceeding successful attempts and deletions support inventory accumulation, but do not establish its underlying cause or customer impact.

Need timestamped queue-wide task metadata (`target`, `createTime`, `scheduleTime`) to measure overdue count/age over time and separate delivery versus extension creation cohorts, plus reconciliation of flow totals against observed depth growth.

## Deep Dive Findings

{
  "workload": "conversations-messages-schedule-events",
  "root_cause_confirmed": false,
  "assessment": "Queue-specific metrics support inventory accumulation: successful creations exceed successful executions and explicit deletions. Future scheduling contributes to inventory, but historical overdue counts and delivery-versus-extension creation cohorts remain unavailable.",
  "confidence": "Medium for inventory accumulation; low for its underlying business cause or customer impact.",
  "findings": {
    "alert_hour": {
      "window_utc": "2026-11-07T10:34:00Z/2026-11-07T11:34:00Z",
      "successful_create_task_calls": 1871,
      "successful_task_attempts": 693,
      "unavailable_task_attempts": 0,
      "successful_delete_task_calls": 59,
      "creations_minus_successful_attempts_and_deletions": 1119,
      "depth_start": 408565,
      "depth_end": 409651,
      "observed_depth_growth": 1086,
      "unreconciled_difference": 33
    },
    "seven_days": {
      "window_utc": "2026-10-31T11:34:00Z/2026-11-07T11:34:00Z",
      "successful_create_task_calls": 1354450,
      "successful_task_attempts": 1098667,
      "unavailable_task_attempts": 6,
      "successful_delete_task_calls": 125867,
      "creations_minus_successful_attempts_and_deletions": 129916,
      "depth_start": 268488,
      "depth_end": 409651,
      "observed_depth_growth": 141163,
      "unreconciled_difference": 11247,
      "interpretation": "The flow totals broadly explain growth and strongly oppose a retry storm. They do not provide an exact inventory reconciliation; the discrepancy remains unexplained."
    },
    "completion_semantics": "Successful attempts are queue-specific Cloud Tasks response_code=ok outcomes, improving on prior route-level response counts. They do not prove downstream message delivery.",
    "current_queue_statistics": {
      "observed_at_utc": "2026-11-07T11:59:06Z",
      "state": "RUNNING",
      "estimated_task_count": 409962,
      "oldest_estimated_arrival_time": "2026-11-07T11:59:59Z",
      "executions_last_minute": 14,
      "effective_execution_rate_limit_per_second": 500,
      "limitation": "The API defines oldestEstimatedArrivalTime as the nearest estimated future scheduled execution, not the oldest overdue task. Executions include successful and unsuccessful outcomes. Neither field establishes zero overdue work."
    },
    "creation_cohorts": {
      "available": false,
      "evidence": [
        "The scheduling source imports @platform-core/cloudtask and targets /conversations/messages/deliver.",
        "The installed client substitutes an extension task for schedules beyond 28 days, retaining the same queue.",
        "cloudtask_creation_total labels only queue_name; it cannot distinguish delivery tasks, extension tasks, or scheduling horizons.",
        "Native Cloud Tasks metrics expose API method and response code, but no target-route or scheduling-cohort labels."
      ],
      "source_files": [
        "marketplace-backend/apps/conversations/src/models/firestore/message.ts:3121",
        "marketplace-backend/node_modules/@platform-core/cloudtask/dist/cloudTask.js:235",
        "marketplace-backend/node_modules/@platform-core/cloudtask/dist/cloudTask.js:325",
        "marketplace-backend/node_modules/@platform-core/cloudtask/dist/observability.js:8"
      ]
    }
  },
  "log_queries_used": [
    {
      "query": "resource.type=\"cloud_tasks_queue\" resource.labels.queue_id=\"conversations-messages-schedule-events\" timestamp>=\"2026-11-07T10:34:00Z\" timestamp<=\"2026-11-07T11:34:00Z\"",
      "limit": 5,
      "result": "Cloud Logging API returned no entries and no next-page token. This does not establish absence of errors.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22cloud_tasks_queue%22%20resource.labels.queue_id%3D%22conversations-messages-schedule-events%22;timeRange=2026-11-07T10%3A34%3A00Z%2F2026-11-07T11%3A34%3A00Z?project=highlevel-backend"
    }
  ],
  "sources": [
    "https://console.cloud.google.com/cloudtasks/queue/us-central1/conversations-messages-schedule-events?project=highlevel-backend",
    "https://monitoring.googleapis.com/v3/projects/highlevel-backend/metricDescriptors/cloudtasks.googleapis.com/queue/task_attempt_count",
    "https://cloudtasks.googleapis.com/$discovery/rest?version=v2beta3"
  ],
  "remaining_gaps": [
    "Historical queue-wide overdue count and age: no such metric appeared in the native Cloud Tasks metric catalog; no historical inventory census was available.",
    "Creation cohorts by target, creation time, and scheduled time: unavailable in the inspected counters and queue logs.",
    "The underlying reason for increased scheduled inventory and the flow-versus-depth reconciliation discrepancy."
  ],
  "recommended_next_action": "Collect timestamped task metadata snapshots grouped by target, createTime, and scheduleTime to measure overdue inventory and creation cohorts separately. Reconcile creations, successful attempts, deletions, and depth before attributing growth to processing capacity.",
  "access": "Read-only GCP APIs authenticated through gcloud, local source inspection, and read-only SQLite. No production changes, messages, or additional orchestrator investigation."
}

## Evidence

### Dashboards (incident window)
- [Task creation — conversations-messages-schedule-events (1820 in the last hour)](https://prod.grafana.leadconnectorhq.com/d/f20df355-ca85-4671-a625-70256c88853d?orgId=1&var-app=All&var-queueName=conversations-messages-schedule-events&var-Team=All&var-subTeam=All&var-container=All&from=1794047640000&to=1794051240000&viewPanel=101)
- [Execution ETA drift — conversations-messages-schedule-events (p95 49.24 seconds)](https://prod.grafana.leadconnectorhq.com/d/f20df355-ca85-4671-a625-70256c88853d?orgId=1&var-app=All&var-queueName=conversations-messages-schedule-events&var-Team=All&var-subTeam=All&var-container=All&from=1794047640000&to=1794051240000&viewPanel=702)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22conversations-messages-schedule-events%22%0Aseverity%3E%3DERROR;timeRange=2026-11-07T10%3A28%3A30Z%2F2026-11-07T10%3A38%3A30Z?project=highlevel-backend)

### Related incidents
- [Earlier same-queue alert: 408072 tasks, marked automatically resolved — 7 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794017262786409)
- [Conversations production pipeline 1628 reported success — 6 Nov](https://gohighlevel.slack.com/archives/G01FL9T6UGL/p1793959930801729)
- [Conversations production pipeline 1629 reported success — 6 Nov](https://gohighlevel.slack.com/archives/G01FL9T6UGL/p1793961858028129)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 12

**Total worker time**: 1995.2s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 9.9s | ✓ |
| grafana-worker | worker | 119.2s | ✓ |
| kubectl-worker | worker | 141.9s | ✓ |
| log-explorer-worker | worker | 271.4s | ✓ |
| slack-worker | worker | 54.2s | ✓ |
| triage | worker | 12.9s | ✓ |
| triage_llm_ranking | decision | 44.9s | ✓ |
| deep_dive_pass_1 | worker | 374.2s | ✓ |
| follow_up_check | decision | 10.0s | ✓ |
| deep_dive_pass_2 | worker | 156.3s | ✓ |
| follow_up_check | decision | 10.0s | ✓ |
| deep_dive_pass_3 | worker | 790.2s | ✗ Codex request aborted |

## Alert Context

```
*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IZHYISJPYX3MM|#139150 CloudTasks Queue Depth > 400000>* via CRM-conversations 
AlertValues:  Hey, looks like the tasks in the below queue have gone above 

Cloudtask Queue: conversations-messages-schedule-events
Tasks in the Queue: 409636 :warning:

GCP Link to the queue: https://console.cloud.google.com/cloudtasks/queue/us-central1/conversations-messages-schedule-events/metrics?project=highlevel-backend

If you want to update the threshold of this queue, please inform infra team.

  :package: Showing the last alert only out of 3 total. Visit <https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IZHYISJPYX3MM|the plugin page> to see them all.
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-07T12:43:33.392Z*