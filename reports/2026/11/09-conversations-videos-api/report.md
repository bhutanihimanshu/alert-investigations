# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-09-ae89bdf4
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-09T06:27:45.112Z
**Status**: delivering

## Root Cause

**Summary**: High-confidence migration regression: production ingress moved the entire /videos/ prefix to the Go conversations API without rewriting the URI or preserving the existing /videos/health endpoint. Deployment timing, live routing, source definitions, and independent probe/HTTP observations support the causal chain to HealthCheckDown. Restore health-route compatibility and verify the public endpoint after deployment.
**Confidence**: high
**Category**: ingress_migration_missing_health_route

## Triage

**Primary Anomaly**: error_spike
**Details**: The public /videos/health endpoint returned HTTP 404 ('Cannot GET /videos/health') around 06:25–06:29 UTC, corroborated by probe metrics and a direct HTTP check. All four crm-conversations-videos-api pods were Ready with zero restarts and low resource usage; Kubernetes probes check a different path. No historical RCA matches: retrieval used the generic CRM-conversations integration label. Missing application logs do not establish absence of errors.
**Anomalous**: Yes
**Deep Dive Direction**: Verify production ingress PR #702 deployment timing and the actual backend/rewrite for /videos/. Check /videos/health support on automation-next-conversations-api and compare public versus backend responses. Production QA was requested immediately before the failure, making migration-related routing or route omission the leading unconfirmed hypothesis. Verify GIF endpoint impact separately; healthy original pods do not establish external recovery.

## Cross-Validation

**Confidence**: high

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana probe metrics and public HTTP checks | Both independently confirm /videos/health returns 404. The endpoint remained failing at 06:33:14 UTC. | ✓ |
| Live Kubernetes ingress and GCP audit logs | The /videos/ prefix now routes unchanged to automation-next-conversations-api. Configuration application at 06:24:47–06:24:49 UTC precedes the first reported failure around 06:25 UTC; the second cluster followed around 06:27:47 UTC. | ✓ |
| Node and Go route definitions | Node automatically registers /videos/health; inspected Go health registrations omit it. This explains why moving the entire prefix produces a health-route 404. | ✓ |
| Slack migration context and GitHub PR #702 | The documented migration and merged ingress change corroborate the routing mechanism. Production QA was requested immediately before the failure. | ✓ |
| Falsification checks | Public Go health returns 200 and /videos/gif reaches validation with 400, arguing against complete destination-service failure. Grafana and Kubernetes independently show healthy original pods, whose probes check a different path. | ✓ |

### Discrepancies
- The original workload's healthy pods and 200 responses do not contradict the external 404: ingress now targets another service.
- The inspected Go source was not mapped to the deployed image, and direct backend comparison was denied. The exact deployed route omission remains inferred.
- Audit timestamps establish configuration application, not exact proxy convergence.
- Missing original-service logs and interrupted Go log queries provide no evidence of error absence.
- Successful GIF generation, customer impact, and endpoint recovery remain unverified.

## Deep Dive Findings

{
  "summary": "High-confidence health-route regression caused by the /videos/ ingress migration. Production now forwards /videos/health unchanged to automation-next-conversations-api, whose health-route registrations omit that path.",
  "confidence": "high",
  "findings": [
    {
      "finding": "Production ingress PR #702 merged at 11:53:08 IST (06:23:08 UTC), replacing crm-conversations-videos-api with automation-next-conversations-api for the entire /videos/ prefix.",
      "source": "https://github.com/GoHighLevel/platform-shared-changes/pull/702"
    },
    {
      "finding": "Live Kubernetes configuration confirms the new destination in both serving clusters, with no URI rewrite at either the parent route or delegate. The destination is automation-next-conversations-api-service.default.svc.cluster.local:6050.",
      "deployment_timing": {
        "web_users_cluster_argocd_apply_utc": "2026-11-09T06:24:48Z",
        "servers_cluster_argocd_apply_utc": "2026-11-09T06:27:47Z"
      },
      "corroboration": "GCP audit logs independently show VirtualService patches at 06:24:47–06:24:49 UTC and 06:27:45–06:27:48 UTC. These are configuration-application times, not exact proxy-convergence times."
    },
    {
      "finding": "The original Node bootstrap automatically registers /<route>/health, and the videos configuration uses route=videos. The Go health registrations include /health and /f1/conversations/health but omit /videos/health. PR #702 described GIF as the only route, overlooking this shared-bootstrap endpoint.",
      "sources": [
        "https://github.com/GoHighLevel/marketplace-backend/blob/main/common/main.ts",
        "https://github.com/GoHighLevel/automation-next-apps-backend/blob/main/apps/conversations/api/routes/health.go"
      ]
    },
    {
      "finding": "Public checks corroborate a specific missing route rather than complete Go-service unavailability.",
      "checks": [
        {
          "url": "https://services.leadconnectorhq.com/videos/health",
          "observed_at_utc": "2026-11-09T06:33:14.947491173Z",
          "status": 404,
          "response": "Cannot GET /videos/health"
        },
        {
          "url": "https://services.leadconnectorhq.com/f1/conversations/health",
          "status": 200,
          "response": "OK"
        },
        {
          "url": "https://services.leadconnectorhq.com/videos/gif",
          "status": 400,
          "response": "VIDEOS_LOCATION_ID_REQUIRED",
          "interpretation": "The GIF route reaches its validation handler. Successful GIF generation and customer impact remain unverified."
        }
      ]
    }
  ],
  "causal_chain": "Ingress migration moved both GIF and the existing monitored health URL to Go. GIF was implemented, but the Node bootstrap's /videos/health endpoint was not carried over. Unrewritten health requests therefore return 404 despite healthy original pods.",
  "limitations": [
    "Direct backend comparison was blocked: Kubernetes denied pods/portforward permission for both services.",
    "Source inspection used current main and local Node source; the deployed image was not mapped to an exact source commit.",
    "The targeted application-error query and a follow-up audit-query verification were interrupted after prolonged execution without results; neither establishes absence of errors.",
    "No GIF generation was attempted because it writes an asset. No production changes or messages were made."
  ],
  "recommended_actions": [
    "Restore /videos/health compatibility in the Go service; alternatively, explicitly route that health URL to an existing health handler.",
    "Add migration coverage for shared-bootstrap endpoints and verify the public health URL after deployment.",
    "Run authorized production GIF QA separately before declaring the migration fully healthy."
  ],
  "log_queries_used": [
    {
      "purpose": "Ingress deployment audit",
      "query": "resource.type=\"k8s_cluster\" protoPayload.resourceName:\"virtualservices\" protoPayload.resourceName:\"routes-ingress-production\" timestamp>=\"2026-11-09T06:22:00Z\" timestamp<=\"2026-11-09T06:29:00Z\"",
      "result": "12 matching audit entries returned.",
      "log_explorer_url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%0AprotoPayload.resourceName%3A%22virtualservices%22%0AprotoPayload.resourceName%3A%22routes-ingress-production%22;timeRange=2026-11-09T06%3A22%3A00Z%2F2026-11-09T06%3A29%3A00Z?project=highlevel-backend"
    },
    {
      "purpose": "Targeted Go application errors",
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"automation-next-conversations-api\" severity>=ERROR timestamp>=\"2026-11-09T06:24:00Z\" timestamp<=\"2026-11-09T06:30:00Z\" (\"/videos/health\" OR \"/videos/gif\")",
      "result": "Interrupted; no conclusion."
    },
    {
      "purpose": "Audit verification",
      "query": "resource.type=\"k8s_cluster\" resource.labels.cluster_name=\"web-users-servers-usc1-prd-cluster\" protoPayload.resourceName=\"networking.istio.io/v1beta1/namespaces/admin/virtualservices/routes-ingress-production\" timestamp>=\"2026-11-09T06:24:47Z\" timestamp<=\"2026-11-09T06:24:51Z\"",
      "result": "Interrupted; initial audit results remain the available log evidence."
    }
  ],
  "access": "Existing local Kubernetes, gcloud, GitHub CLI and validated Slack sessions; public HTTP checks via curl."
}

## Evidence

### Dashboards (incident window)
- [Memory — crm-conversations-videos-api (stable usage across serving clusters)](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=All&var-container=crm-conversations-videos-api&var-pod_name=All&var-gcpProject=highlevel-backend&from=1794203864875&to=1794205751664&viewPanel=30)
- [Pod timeline — crm-conversations-videos-api (current pods predate alert)](https://prod.grafana.leadconnectorhq.com/d/a4859d4a-1e0a-4ae3-b9b2-d04d366cf29b?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=All&var-container=crm-conversations-videos-api&var-pod_name=All&var-gcpProject=highlevel-backend&from=1794203864875&to=1794205751664&viewPanel=52)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-conversations-videos-api%22%0Aseverity%3E%3DWARNING%0A%28jsonPayload.message%3D~%22%28%3Fi%29health%7Ctimeout%7Cerror%7Cfailed%22%20OR%20textPayload%3D~%22%28%3Fi%29health%7Ctimeout%7Cerror%7Cfailed%22%29;timeRange=2026-11-09T06%3A22%3A00Z%2F2026-11-09T06%3A33%3A00Z?project=highlevel-backend)

### Related incidents
- [Videos health check returns 404; war room requested — 9 Nov](https://gohighlevel.slack.com/archives/CMNNHMQUU/p1794205565261719)
- [Proposed /videos/ ingress migration to Go service](https://gohighlevel.slack.com/archives/C03D923DH8F/p1793683205717489)
- [Videos migration prod QA requested immediately before health failure](https://gohighlevel.slack.com/archives/C096N87SBFW/p1794205485923179?thread_ts=1793653175.956119)

## Investigation Timeline

**Deep dive passes**: 1
**Worker calls**: 10

**Total worker time**: 661.8s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 10.5s | ✓ |
| grafana-worker | worker | 83.0s | ✓ |
| kubectl-worker | worker | 106.1s | ✓ |
| log-explorer-worker | worker | 73.6s | ✓ |
| slack-worker | worker | 72.0s | ✓ |
| triage | worker | 13.6s | ✓ |
| triage_llm_ranking | decision | 68.0s | ✓ |
| deep_dive_pass_1 | worker | 205.3s | ✓ |
| follow_up_check | decision | 10.6s | ✓ |
| cross_validation | worker | 19.2s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IIM3NVB2D3HHF|#139298 HealthCheckDown>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=18240233953195705161&amp;alert_id=11383604584845527343|source>*) summary: Health Check Endpoint Down threshold: 1 description: Health check endpoint - `<https://services.leadconnectorhq.com/videos/health>` for the app `crm-conversations-videos-api` is down for the last 1 minute *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IIM3NVB2D3HHF|#139298 HealthCheckDown>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=18240233953195705161&amp;alert_id=11383604584845527343|source>*) summary: Health Check Endpoint Down threshold: 1 description: Health check endpoint - `<https://services.leadconnectorhq.com/videos/health>` for the app `crm-conversations-videos-api` is down for the last 1 minute
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-09T06:35:19.016Z*