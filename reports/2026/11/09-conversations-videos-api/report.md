# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-09-d9096254
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-09T07:32:45.107Z
**Status**: delivering

## Root Cause

**Summary**: Traffic migration left the internal health monitor targeting /videos/health, which current ingress forwards without rewriting to a destination that returns 404 for that path. Independent probe metrics confirm persistent failure, while /f1/conversations/health returns 200. Complete the monitor migration and verify successful deployed probes and alert resolution; preserve a compatible original route if required.
**Confidence**: high
**Category**: health_monitor_configuration_mismatch

## Triage

**Primary Anomaly**: error_spike
**Details**: External /videos/health changed from HTTP 200 to persistent 404 at 06:25:30 UTC (11:55:30 IST), coinciding with old-workload traffic dropping to zero. CPU, memory and pod health remained normal. Slack reports traffic migration without updating health checks. Candidate inv-2026-11-09-ae89bdf4 closely matches, but its precise ingress mechanism is not independently verified by Layer 1. Empty application logs do not disprove routing failure; their windows missed onset.
**Anomalous**: Yes
**Deep Dive Direction**: Targeted verification of the migration hypothesis: inspect live /videos/ ingress destination and URI rewrite, verify destination health routes, and compare public /videos/health with /f1/conversations/health. Correlate routing changes with 06:25:30 UTC. Confirm internal monitor update and probe recovery; Betterstack resolution on a different URL does not prove original-endpoint recovery. Treat the same-day historical RCA as a lead, not independent corroboration.

## Cross-Validation

**Confidence**: high

| Source | Finding | Agrees |
|--------|---------|--------|
| Live ingress configuration and HTTP responses | Ingress forwards /videos/health unchanged to automation-next-conversations-api. Public requests returned 404 for that path and 200 for /f1/conversations/health, supporting a health-path mismatch. | ✓ |
| Grafana probe and traffic metrics | The original endpoint changed from 200 to 404 at 06:25:30 UTC as old-workload traffic declined to zero. Its probe still failed at 07:38:38 UTC. | ✓ |
| GCP audit logs | Successful ingress patches occurred 41–42 seconds before the first failing probe. Timing supports the migration hypothesis, but missing patch payloads prevent identifying the precise causal change. | ✓ |
| Slack responder context | Responders reported traffic migration without a corresponding health-monitor update. The internal monitor update remained unconfirmed. | ✓ |
| Kubernetes and resource metrics | Healthy old-workload pods, zero restarts and normal resource usage argue against a crash or resource-pressure explanation. Their probes check / rather than the failing external path. | ✓ |

### Discrepancies
- Confidence is high for the current routing/monitor-path mismatch, but medium for attribution to the exact historical ingress patch.
- Empty application-log queries missed failure onset and do not disprove routing failure.
- Local source was not matched to the deployed revision; live responses establish endpoint behavior but not the exact backend handler.
- Betterstack resolved against a different URL; this does not establish original-endpoint or internal-monitor recovery.
- Customer impact was reported absent in Slack but not independently measured. The later pipeline completion cannot explain the earlier failure onset.

## Deep Dive Findings

{
  "root_cause": {
    "assessment": "Traffic migration left the internal monitor targeting an unsupported health path. Live ingress forwards /videos/health unchanged to automation-next-conversations-api, which returns 404; /f1/conversations/health returns 200.",
    "confidence": "High for the current routing/path mismatch; medium for attribution to the precise historical configuration change."
  },
  "findings": [
    {
      "finding": "Ingress destination and rewrite verified",
      "evidence": "In both servers-usc1-prd-cluster and web-users-servers-usc1-prd-cluster, /videos/ delegates to automation-next-conversations-api virtual services. Destination is automation-next-conversations-api-service.default.svc.cluster.local:6050. Neither the matching parent route nor destination delegate specifies a URI rewrite."
    },
    {
      "finding": "Destination health-path mismatch verified",
      "evidence": "Local Go source registers /health, /healthz and /f1/conversations/health, but not /videos/health. Public requests at 13:07:29–30 IST (07:37:29–30 UTC) returned 404 with 'Cannot GET /videos/health' for the original URL and 200 OK for the replacement.",
      "urls": [
        "https://services.leadconnectorhq.com/videos/health",
        "https://services.leadconnectorhq.com/f1/conversations/health"
      ],
      "limitation": "Local source was not matched to the deployed image revision; live HTTP responses independently confirm the endpoint behavior."
    },
    {
      "finding": "Routing changes closely precede failure onset",
      "evidence": "GCP audit logs show three successful patches to web-users cluster admin/routes-ingress-production at 11:54:47.670–49.634 IST (06:24:47.670–49.634 UTC). Independently queried probe metrics show HTTP 200 through 11:55:15 IST and 404 from 11:55:30 IST (06:25:30 UTC).",
      "limitation": "Audit results did not expose the changed route payload, so these timestamps do not independently prove which patch migrated /videos/."
    },
    {
      "finding": "Internal monitor recovery is not confirmed",
      "evidence": "At 13:08:38 IST (07:38:38 UTC), blackbox metrics still targeted /videos/health with probe_http_status_code=404 and probe_success=0. No matching replacement-target series was returned. The fetched Slack thread still only states 'Updating it', without completion confirmation.",
      "url": "https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794209649313529?thread_ts=1794209565.107049",
      "limitation": "Direct platform ConfigMap inspection failed because the Kubernetes API was unreachable. Betterstack resolution against another URL does not establish original-endpoint recovery."
    }
  ],
  "recommended_actions": [
    "Complete the internal monitor change to /f1/conversations/health and verify its deployed target, successful probe samples, and alert resolution.",
    "If /videos/health remains a supported external contract, restore an explicit compatible health route.",
    "Include health-monitor migration and probe verification in future traffic cutovers."
  ],
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_cluster\" protoPayload.resourceName:\"virtualservices\" timestamp>=\"2026-11-09T06:20:00Z\" timestamp<=\"2026-11-09T06:30:00Z\"",
      "limit": 5,
      "purpose": "Discover routing audit events."
    },
    {
      "query": "resource.type=\"k8s_cluster\" protoPayload.resourceName:\"virtualservices/routes-ingress-production\" timestamp>=\"2026-11-09T06:20:00Z\" timestamp<=\"2026-11-09T06:28:00Z\"",
      "limit": 5,
      "purpose": "Identify ingress patch timing."
    },
    {
      "query": "resource.type=\"k8s_cluster\" resource.labels.cluster_name=\"web-users-servers-usc1-prd-cluster\" protoPayload.resourceName=\"networking.istio.io/v1beta1/namespaces/admin/virtualservices/routes-ingress-production\" timestamp>=\"2026-11-09T06:24:00Z\" timestamp<=\"2026-11-09T06:26:00Z\"",
      "limit": 5,
      "result": "Three patches; each reported status.code=0.",
      "log_explorer_url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0Aresource.labels.cluster_name%3D%22web-users-servers-usc1-prd-cluster%22%0AprotoPayload.resourceName%3D%22networking.istio.io%2Fv1beta1%2Fnamespaces%2Fadmin%2Fvirtualservices%2Froutes-ingress-production%22%0Atimestamp%3E%3D%222026-11-09T06%3A24%3A00Z%22%0Atimestamp%3C%3D%222026-11-09T06%3A26%3A00Z%22;timeRange=2026-11-09T06%3A24%3A00Z%2F2026-11-09T06%3A26%3A00Z?project=highlevel-backend"
    }
  ],
  "access_method": "Existing kubectl/gcloud credentials and validated Slack/Grafana stored browser sessions; read-only requests.",
  "limitations": [
    "Customer impact was not independently measured.",
    "The same-day historical RCA was not used as independent corroboration.",
    "Parallel delegation was attempted but failed because the executor could not load the subagent thread."
  ]
}

## Evidence

### Dashboards (incident window)
- [Memory — crm-conversations-videos-api: working set within seven-day range](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=All&var-container=crm-conversations-videos-api&var-pod_name=All&var-gcpProject=highlevel-backend&from=1794204000000&to=1794211200000&viewPanel=30)
- [Pod creation — crm-conversations-videos-api: current pods predate failure onset](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=All&var-container=crm-conversations-videos-api&var-pod_name=All&var-gcpProject=highlevel-backend&from=1794204000000&to=1794211200000&viewPanel=52)

### Related incidents
- [Current alert: internal health-check update in progress — 9 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794209649313529?thread_ts=1794209565.107049)
- [Team RCA: traffic migrated, health endpoint not updated — 9 Nov](https://gohighlevel.slack.com/archives/CMNNHMQUU/p1794207634430789?thread_ts=1794205565.261719)
- [Earlier identical HealthCheckDown #139298 — 9 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794205665112629)
- [Betterstack cleared after 16 minutes with a different checked URL — 9 Nov](https://gohighlevel.slack.com/archives/CMNNHMQUU/p1794206574477309)

## Investigation Timeline

**Deep dive passes**: 1
**Worker calls**: 10

**Total worker time**: 580.1s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 16.4s | ✓ |
| grafana-worker | worker | 86.4s | ✓ |
| kubectl-worker | worker | 81.9s | ✓ |
| log-explorer-worker | worker | 65.6s | ✓ |
| slack-worker | worker | 62.4s | ✓ |
| triage | worker | 17.0s | ✓ |
| triage_llm_ranking | decision | 73.7s | ✓ |
| deep_dive_pass_1 | worker | 146.0s | ✓ |
| follow_up_check | decision | 11.6s | ✓ |
| cross_validation | worker | 19.1s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IBCACFHD2N6R3|#139303 HealthCheckDown>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=18240233953195705161&amp;alert_id=11383604584845527343|source>*) summary: Health Check Endpoint Down threshold: 1 description: Health check endpoint - `<https://services.leadconnectorhq.com/videos/health>` for the app `crm-conversations-videos-api` is down for the last 1 minute *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IBCACFHD2N6R3|#139303 HealthCheckDown>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=18240233953195705161&amp;alert_id=11383604584845527343|source>*) summary: Health Check Endpoint Down threshold: 1 description: Health check endpoint - `<https://services.leadconnectorhq.com/videos/health>` for the app `crm-conversations-videos-api` is down for the last 1 minute
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-09T07:40:03.479Z*