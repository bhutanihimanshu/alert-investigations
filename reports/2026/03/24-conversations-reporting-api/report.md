# BaseService503HighCount — conversations-reporting-api — 2026-03-24 (Slack) / automation pickup 2026-03-25 IST

| Field | Value |
| --- | --- |
| OnCall group | [IPE1Y4NALIV39](https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IPE1Y4NALIV39) (#113439) |
| Slack | [Thread](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1774321935773169) (`C097UPY34QJ`, `thread_ts` 1774321935.773169) |
| Alert type | BaseService503HighCount |
| Workload | conversations-reporting-api |
| Threshold / observed | 5000 / **31021** (from Grafana alert text in Slack) |
| Slack message time (UTC) | 2026-03-24T03:12:15.773Z |

## Root cause (technical)

**Not determined in this run.** Grafana API returned 401 (session expired); targeted GCP log reads for the alert window did not return within the investigation timeout. The Slack thread contains acknowledgements and reminders only—no engineering RCA.

**Working hypothesis (low confidence):** The same automation poll that ingested this alert also ingested multiple other firing alerts (pod restarts, HTTP latency, health check). That pattern is consistent with a **short-lived platform or dependency incident** affecting several workloads; `conversations-reporting-api` would then show elevated outbound 503s in BaseService metrics. This is **not** confirmed against metrics or logs here.

For validated technical causes on this service in the past, see evidence-only pointers in the verbose report (no claim they apply to this event).

## Automation / coverage gap (high confidence)

**Yes — BaseService503 is in the automation allowlist** (`extract_alert_type` maps `baseservice503*` → `BaseService503HighCount` in `check-slack-alerts.sh`).

**Why there was no dedicated report:** On `2026-03-25` ~15:12 IST, the poller saw **seven** new alerts in one batch (`BURST_THRESHOLD=3`), so **burst mode** ran. It logged `Investigating grouped alerts: podrestartsabovethreshold (3 alerts)` and `Launching Cursor agent for investigation...` for that **first** type group only.

**What went wrong next:** In `check-slack-alerts.log` there is **no** matching `Agent investigation complete` line after that launch (unlike other investigation runs on Mar 16 and Mar 27). The following poll at 15:17 IST still shows **`daily count: 0/15`**, which implies `increment_daily_count` (called only after `run_investigation` returns) **never ran** for that burst. The burst `while` loop therefore **never advanced** to the next alert types (HTTPLatencyPerApp, BaseService503, etc.). Slack cursors and `mark_seen` had already advanced for all seven messages, so those alerts **did not re-queue** for a later pass.

A second failure mode also exists in code: even if the first agent had finished quickly, **`MAX_INVESTIGATIONS_PER_POLL=3`** would only allow three type-groups per poll; with four distinct types in the batch, **one type would still be dropped** in a fully successful run.

## Follow-up

1. **Refresh Grafana session** and inspect [BaseService observability](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d44d366cf29b/baseservice-3a-observability) with `destination=conversations-reporting-api` for 2026-03-24 ~02:00–05:00 UTC to see which downstream services contributed 503s.
2. **GCP** — pod events and app logs for `conversations-reporting-api` in the same window (`fieldPath` on probe failures distinguishes app vs `istio-proxy`).
3. **Automation** — (a) ensure a stuck or killed agent does not strand the rest of a burst (timeout, async queue, or per-type subprocess); (b) prioritize or second-pass **BaseService503 / 5xx** when `MAX_INVESTIGATIONS_PER_POLL` would drop them; (c) optional **multi-type single prompt** for the whole burst.

## Evidence index

1. Slack `conversations.replies` for parent `ts=1774321935.773169` — full alert text, thread content.
2. `~/.cursor/automations/logs/check-slack-alerts.log` — burst + grouped investigation line.
3. `~/.cursor/automations/tasks/check-slack-alerts.sh` — `BURST_THRESHOLD`, `MAX_INVESTIGATIONS_PER_POLL`, burst grouping logic.
