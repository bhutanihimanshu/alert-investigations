# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-09-f88f2013
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-09T11:33:30.106Z
**Status**: delivering

## Root Cause

**Summary**: Strongly supported migration-induced monitor mismatch: the /videos/ prefix was redirected to the Go service while monitoring retained the Node health path, causing persistent probe 404s. Timing, route/configuration evidence and team observations independently support the causal chain. The conclusion would be undermined if deployed routing differed from the merged diff or the deployed Go service supported /videos/health; neither deployed-state detail is fully verified. This establishes a monitoring failure, not customer-facing outage or absence of customer impact.
**Confidence**: high
**Category**: migration_health_check_configuration_mismatch

## Triage

**Primary Anomaly**: error_spike
**Details**: Synthetic /videos/health probes changed from HTTP 200 to persistent 404 at 06:25:18 UTC on November 9; this does not establish a customer API error spike. Resource usage remains within baseline, with four available replicas and no restarts in the last hour. Slack reports traffic migration without a health-monitor update, matching candidates inv-2026-11-09-d9096254 and inv-2026-11-09-ae89bdf4. Current routing and customer impact remain unverified; empty logs and unavailable kubectl checks are inconclusive.
**Anomalous**: Yes
**Deep Dive Direction**: YES — The triage direction remains correct: evidence supports a migration/monitor mismatch, with the replacement probe now healthy. To confirm root cause, obtain the ingress patch diff and monitor configuration at the 06:25 UTC onset, establishing which route changed and why `/videos/health` began returning 404.

## Cross-Validation

**Confidence**: high

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana probe telemetry | /videos/health changed from HTTP 200 to persistent 404 at 06:25:18 UTC, supporting a health-path failure rather than a demonstrated customer API error spike. | ✓ |
| Ingress PR #702 and GCP audit logs | The merged diff redirects /videos/ to automation-next-conversations-api without rewriting. Successful ingress patches occurred 29–30 seconds before probe failure; audit payloads do not establish the exact deployed diff. | ✓ |
| Monitor configuration and application source | Historical monitoring retained /videos/health. Node registers that endpoint, while inspected Go source exposes /f1/conversations/health instead. This explains the 404 mechanism, although deployed source revisions were not verified. | ✓ |
| Slack team context | The team independently reported traffic migration without updating the health endpoint. Its claim of no customer impact remains unverified. | ✓ |

### Discrepancies
- Exact deployed ingress and monitor payloads and Go image revision remain unavailable.
- Healthy original replicas and normal current resources weaken an exhaustion hypothesis but do not establish destination-service health at onset.
- Empty application logs and unavailable kubectl checks are inconclusive; log queries covered the later recurrence rather than the initial failure.
- Replacement-probe recovery is reported through prior-pass evidence; original-path compatibility, customer GIF generation and current OnCall resolution remain unverified.

## Deep Dive Findings

{
  "summary": "Retrieved the ingress PR diff and historical monitor configuration. Evidence strongly supports a migration-induced health-path mismatch: /videos/ moved to the Go service while monitoring retained /videos/health.",
  "confidence": "high for the migration/monitor mismatch; exact deployed patch payload remains unavailable",
  "findings": [
    {
      "finding": "The ingress migration changed the entire /videos/ prefix destination.",
      "evidence": "PR #702 merged at 11:53:08 IST (06:23:08 UTC). Its one-line diff replaces crm-conversations-videos-api with automation-next-conversations-api while retaining prefix /videos/. The PR explicitly describes forwarding without rewriting.",
      "why": "The GIF endpoint had been ported to Go so the Node deployment could be retired. The PR described GIF as the only route, overlooking the bootstrap-provided health endpoint.",
      "source": "https://github.com/GoHighLevel/platform-shared-changes/pull/702"
    },
    {
      "finding": "Successful ingress patches immediately preceded probe failure.",
      "evidence": "Fresh audit queries returned successful patches at 06:24:47.670, 06:24:48.856 and 06:24:49.634 UTC, approximately 29–30 seconds before the supplied 06:25:18 UTC failure onset.",
      "limitation": "Audit records omit request and response payloads. The Git diff establishes the intended change; audit timing alone cannot prove its deployed contents."
    },
    {
      "finding": "Historical monitoring configuration still targeted the original health path.",
      "evidence": "The latest file revision preceding onset, b7703b66cefadb4d608aa93016cae6f87f4048b0, contains target https://services.leadconnectorhq.com/videos/health with ghl_app=crm-conversations-videos-api.",
      "source": "https://github.com/GoHighLevel/platform-infra-helm-charts/blob/b7703b66cefadb4d608aa93016cae6f87f4048b0/helm-templates/victoria-metrics-agent/configmap/platform-scrape-prd.yaml#L772",
      "limitation": "This verifies historical repository configuration, not a captured ConfigMap from 06:25 UTC. Supplied probe telemetry independently confirms the active target."
    },
    {
      "finding": "Source inspection explains why the migrated health request returned 404.",
      "evidence": "Node common/main.ts registers /${options.route}/health, and videos uses route='videos'. Inspected Go health registrations include /f1/conversations/health but omit /videos/health.",
      "limitation": "Local source was not matched to the deployed image. Earlier investigation records independently observed unchanged ingress forwarding and HTTP 404 versus HTTP 200 on the two paths; those observations were not repeated in this pass."
    },
    {
      "finding": "A later commit explicitly corrected the monitor target.",
      "evidence": "Commit f765caeb84729d5f3d4684b8112ff01bc20f4861, timestamped 07:38:45 UTC, replaces /videos/health with /f1/conversations/health and updates ghl_app to automation-next-conversations-api. Fresh audit queries confirm successful vmagent ConfigMap patches at 11:38:27–36 UTC; supplied prior-pass metrics show subsequent replacement-probe success.",
      "source": "https://github.com/GoHighLevel/platform-infra-helm-charts/commit/f765caeb84729d5f3d4684b8112ff01bc20f4861",
      "limitation": "Commit time is not deployment time. ConfigMap audit payloads remain unavailable."
    }
  ],
  "falsification": "Supplied healthy original pods, normal resource usage, and successful replacement health probes argue against resource exhaustion or complete destination-service failure. They do not establish successful customer GIF generation.",
  "remaining_unknowns": [
    "Exact deployed ingress and monitor payloads at onset.",
    "Deployed Go image-to-source revision mapping.",
    "Customer GIF-generation impact and OnCall resolution."
  ],
  "recommended_actions": [
    "Include bootstrap health routes and monitor cutover verification in migration checks.",
    "Restore /videos/health compatibility if it remains a supported contract.",
    "Verify customer GIF generation separately before asserting no customer impact."
  ],
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_cluster\" resource.labels.cluster_name=\"web-users-servers-usc1-prd-cluster\" protoPayload.resourceName=\"networking.istio.io/v1beta1/namespaces/admin/virtualservices/routes-ingress-production\" timestamp>=\"2026-11-09T06:24:00Z\" timestamp<=\"2026-11-09T06:26:00Z\"",
      "limit": 5,
      "result": "Three successful patches; request/response payloads absent.",
      "log_explorer_url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%20resource.labels.cluster_name%3D%22web-users-servers-usc1-prd-cluster%22%20protoPayload.resourceName%3D%22networking.istio.io/v1beta1/namespaces/admin/virtualservices/routes-ingress-production%22;timeRange=2026-11-09T06%3A24%3A00Z%2F2026-11-09T06%3A26%3A00Z?project=highlevel-backend"
    },
    {
      "query": "resource.type=\"k8s_cluster\" resource.labels.cluster_name=\"gke-platform-prd-usc1\" protoPayload.resourceName:\"configmaps\" protoPayload.resourceName:\"agent\" timestamp>=\"2026-11-09T11:38:00Z\" timestamp<=\"2026-11-09T11:39:00Z\"",
      "limit": 5,
      "result": "Five successful ConfigMap patches; request/response payloads absent.",
      "log_explorer_url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%20resource.labels.cluster_name%3D%22gke-platform-prd-usc1%22%20protoPayload.resourceName%3A%22configmaps%22%20protoPayload.resourceName%3A%22agent%22;timeRange=2026-11-09T11%3A38%3A00Z%2F2026-11-09T11%3A39%3A00Z?project=highlevel-backend"
    }
  ],
  "access": "Read-only local gcloud and GitHub CLI credentials. GitHub connector access failed; subagent delegation failed to load its thread. No production changes or messages sent."
}

## Evidence

### Dashboards (incident window)
- [Memory — crm-conversations-videos-api (current usage within seven-day range)](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=All&var-container=crm-conversations-videos-api&var-pod_name=All&from=1794204000000&to=1794225600000&viewPanel=30)
- [Pod creation — crm-conversations-videos-api (current pods predate incident)](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=All&var-container=crm-conversations-videos-api&var-pod_name=All&from=1794204000000&to=1794225600000&viewPanel=52)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-videos-api%22%0Aseverity%3E%3DWARNING;timeRange=2026-11-09T11%3A28%3A30Z%2F2026-11-09T11%3A38%3A30Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%20resource.labels.cluster_name%3D%22web-users-servers-usc1-prd-cluster%22%20protoPayload.resourceName%3D%22networking.istio.io/v1beta1/namespaces/admin/virtualservices/routes-ingress-production%22;timeRange=2026-11-09T06%3A24%3A00Z%2F2026-11-09T06%3A26%3A00Z?project=highlevel-backend)

### Related incidents
- [Team attributes health-check failure to traffic migration — 9 Nov](https://gohighlevel.slack.com/archives/CMNNHMQUU/p1794207634430789)
- [Earlier identical alert: internal monitor update still awaiting confirmation — 9 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794209565107049)

## Investigation Timeline

**Deep dive passes**: 2
**Worker calls**: 12

**Total worker time**: 737.4s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 1.7s | ✓ |
| grafana-worker | worker | 98.1s | ✓ |
| kubectl-worker | worker | 94.1s | ✓ |
| log-explorer-worker | worker | 60.6s | ✓ |
| slack-worker | worker | 59.4s | ✓ |
| triage | worker | 14.1s | ✓ |
| triage_llm_ranking | decision | 74.9s | ✓ |
| deep_dive_pass_1 | worker | 157.7s | ✓ |
| follow_up_check | decision | 12.7s | ✓ |
| deep_dive_pass_2 | worker | 135.2s | ✓ |
| follow_up_check | decision | 11.4s | ✓ |
| cross_validation | worker | 17.6s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IQVSZH96LK9NU|#139313 HealthCheckDown>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=18240233953195705161&amp;alert_id=11383604584845527343|source>*) summary: Health Check Endpoint Down threshold: 1 description: Health check endpoint - `<https://services.leadconnectorhq.com/videos/health>` for the app `crm-conversations-videos-api` is down for the last 1 minute *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IQVSZH96LK9NU|#139313 HealthCheckDown>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=18240233953195705161&amp;alert_id=11383604584845527343|source>*) summary: Health Check Endpoint Down threshold: 1 description: Health check endpoint - `<https://services.leadconnectorhq.com/videos/health>` for the app `crm-conversations-videos-api` is down for the last 1 minute
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-09T11:45:24.109Z*