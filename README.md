# Alert Investigations

Automated production alert investigation reports with Grafana dashboard screenshots and GCP log evidence.

## Structure

```
reports/
  YYYY/
    MM/
      DD-<alert-slug>/
        report.md              # Concise report (shareable)
        report-verbose.md      # Full investigation with all evidence
        screenshots/           # Grafana + GCP screenshots
```

## How it works

1. Alerts fire in Slack (`#alerts-crm-conversations`, `#alerts-crm`)
2. Automation detects new alerts every 5 minutes
3. Cursor agent investigates end-to-end (Grafana metrics, GCP logs, kubectl, past investigations)
4. Reports are committed here with screenshots
5. GitHub links are posted in Slack DM for review

## Reports

| Date | Alert | Service | Report |
|------|-------|---------|--------|
| 2026-03-22 | PodRestartsAboveThreshold | conversations-bulk-internal-api | [report](reports/2026/03/22-pod-restarts-conversations-bulk-internal-api/report.md) |
| 2026-03-19 | PodRestartsAboveThreshold | conversations-bulk-internal-api | [report](reports/2026/03/19-pod-restarts-conversations-bulk-internal-api/report.md) |
