# BaseService503HighCount — conversations-reporting-api — verbose evidence log

## 1. Slack (validated session)

`auth.test` succeeded for workspace HighLevel before fetching thread.

**Parent message (bot — Grafana Alerts):**

- Text includes: `#113439 BaseService503HighCount`, CRM-conversations routing, `current_value: 31021`, `threshold: 5000`, workload **conversations-reporting-api**, doc link `http://platform.docs/alerts/base-service-destination-503-alert/`, dashboard link to BaseService observability.
- Edited at `1774327308` (Grafana/on-call UI edit; original post `ts` **1774321935.773169**).
- Acknowledgement attachment: Balaji Venkatesh.

**Thread replies (substantive):**

- Repeated Grafana bot reminders that the alert group remained acknowledged.
- Human: request to treat 503s as critical and explain cause (no technical RCA in thread as of fetched state).

Permalink provided by operator: `https://gohighlevel.slack.com/archives/C097UPY34QJ/p1774321935773169`

## 2. Timestamp interpretation

`1774321935.773169` → **2026-03-24T03:12:15.773Z** (UTC) → **2026-03-24 08:42:15 IST**.

The local automation log timestamps are in **IST** and show the **poll** that first enqueued this message, not the Grafana evaluation instant.

## 3. Automation log (local file)

File: `~/.cursor/automations/logs/check-slack-alerts.log`

Relevant sequence (from log inspection):

- `[2026-03-25 15:12:27] NEW ALERT ... #113439 BaseService503HighCount` (among several other NEW ALERT lines in the same second).
- `[2026-03-25 15:12:28] Burst detected (7 alerts > threshold 3), grouping by alert type`
- `#dev` context fetch (activity found).
- `[2026-03-25 15:12:30] Investigating grouped alerts: podrestartsabovethreshold (3 alerts)`
- `[2026-03-25 15:12:30] Launching Cursor agent for investigation...`
- **Missing:** `Agent investigation complete for #alerts-crm-conversations alert` (present after other investigation runs on Mar 16 / Mar 27 in the same file).
- `[2026-03-25 15:17:30] Starting Slack alert check (daily count: 0/15)` — next poll; daily counter unchanged.

**Investigation log artifacts:** `~/.cursor/automations/logs/investigations/` (current listing) contains no `2026-03-25-*.log`, consistent with the Mar 25 burst not finishing the investigation pipeline for that launch.

## 4. Script logic (why BaseService503 was skipped)

File: `~/.cursor/automations/tasks/check-slack-alerts.sh`

- `BURST_THRESHOLD=3` — burst path when more than three alerts in one poll.
- `MAX_INVESTIGATIONS_PER_POLL=3` — hard cap per channel poll.
- Burst branch builds `unique_types` via `cut -f1 | sort | uniq -c | sort -rn | awk '{print $2}'` (alert type field, descending frequency).
- Iterates types in that order; **stops when `investigations_this_poll >= MAX_INVESTIGATIONS_PER_POLL`**.

**Observed for this incident (log + state):** Only the **first** type group (`podrestartsabovethreshold`) reached `run_investigation`. There is **no** trailing `Agent investigation complete` log line for that launch, and the next scheduler tick still showed **`daily count: 0`**, so `increment_daily_count` never ran — the burst loop **did not proceed** to later types (HTTPLatencyPerApp, `baseservice503`, `unknown`, etc.). Phase 1 had already **`mark_seen`** all seven Slack messages and advanced the channel cursor, so those alerts **cannot re-enter** the queue on a later poll.

**Structural risk (even if the first agent had returned cleanly):** The same batch contained **four** distinct normalized types (approximate counts):

| Approx. type key | Count in batch |
| --- | ---: |
| podrestartsabovethreshold | 3 |
| httplatencyperapp | 2 |
| unknown (HealthCheckDown not in regex) | 1 |
| baseservice503 (from `BaseService503` substring in `BaseService503HighCount`) | 1 |

With `MAX_INVESTIGATIONS_PER_POLL=3`, **at least one type group would still be skipped** after three successful investigations.

**Cooldown note:** Elsewhere the same log shows `On cooldown (5xx/conversations-reporting-api)` for different alert text — a separate code path. This thread’s skip is **burst + non-returning first `run_investigation`**, compounded by **per-poll cap** if the loop had continued.

## 5. Metrics / logs attempted in this run

| Source | Result |
| --- | --- |
| Grafana `GET /api/org` | **401** — `session.token.rotate` / Unauthorized |
| `gcloud logging read` (narrow timestamp + pod filter) | **No usable output** within timeout (command remained running in background; treat as **not verified**) |

Therefore **no** Prometheus series or log lines are cited as proof of downstream service names, probe `fieldPath`, or traffic shape for **this** alert.

## 6. Repository docs (historical context only — not this incident)

These are **not** evidence for #113439; they show known patterns for the same workload name:

- `docs/public/conversations/base-service-503-reporting-api-2026-03-06.md` — istiod KEDA oscillation / sidecar 503 symptom chain (later nuance: also verify app vs proxy `fieldPath`).
- `docs/public/conversations/conversations-reporting-api-memory-pressure-2026-03-10.md` — memory pressure pattern.

Use them only as a checklist when Grafana/GCP are available.

## 7. Investigations DB

`sqlite3 ~/.cursor/data/investigations.db` — no row keyed to this Slack `slack_message_ts` or OnCall id was required for this report; prior rows reference older manual report slugs (e.g. 2026-03-06) with different alert ids.

## 8. Suggested automation improvements (actionable)

1. **Priority queue:** After burst detection, sort type groups so `BaseService503HighCount` (and/or `Api5xxSpike`) cannot be starved by higher-count lower-severity types.
2. **Second pass:** If `unique_types` has more types than `MAX_INVESTIGATIONS_PER_POLL`, schedule remaining types on the next poll without requiring a new Slack message (or raise cap for burst days).
3. **Bundled prompt:** Single investigation with “multi-symptom incident” including **all** alert texts in the batch, not only the dominant type (trade-off: noisier prompts).

---

*Generated as part of a manual follow-up investigation; technical RCA pending Grafana/GCP access.*
