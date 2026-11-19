# Alert Investigation: 5XXPercentagePerAPI

**Investigation ID**: inv-2026-11-19-09f85318
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-19T15:49:35.501Z
**Status**: delivering

## Root Cause

**Summary**: The supported proximate cause is grammar-engine stream failures, with fewer three-second timeouts, surfacing as route-level 5xx responses during the anomalous interval. Metrics independently establish impact and logs/source support the failure mechanism. The underlying provider or SDK failure reason remains unknown; neither an actual upstream HTTP 503 nor A/B rollout causality is established.
**Confidence**: medium
**Category**: grammar_engine_failure_underlying_cause_unconfirmed

## Triage

**Primary Anomaly**: error_spike
**Details**: POST /conversations/grammar/check on conversations-api recorded 34/54 requests failing in five minutes (62.96%, versus 0.384% seven-day baseline). Logs independently show 40 upstream LLM HTTP 503 failures and five approximately three-second timeouts across 42 pods and five locations. Fleet-average resources remained within historical ranges. Historical candidates match only the broad downstream-failure/low-volume pattern, not this route or causal signature; no RCA shortcut is justified.
**Anomalous**: Yes
**Deep Dive Direction**: YES — Triage direction remains correct: provider-stream errors dominate, separate from three-second timeouts. The logged 503 may be SDK-inferred; upstream HTTP 503 and A/B rollout causality remain unproven.

Need sanitized provider codes/types, original response status, upstream request IDs and retry traces; deployed image/dependency versions; and Statsig allocation/audit timestamps correlated with traffic and failure onset.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | Grammar-check recorded approximately 34 failures among 54 requests in five minutes, versus a 0.384% seven-day baseline. Fleet-average CPU and memory remained within historical ranges. | ✓ |
| Application logs and source inspection | Logs show 40 AI_StreamProviderError failures and five approximately three-second timeouts across 42 pods. Inspected local code maps these paths to HTTP 503 and 504, supporting the mechanism behind the route's 5xx increase. | ✓ |
| Deployment audit | The sampled failing pods share a ReplicaSet whose configured image was deployed over five hours before the alert. This weakens an immediate code-rollout hypothesis but does not exclude a latent defect. | ✓ |
| Slack | An A/B launch announcement preceded the alert by 26 minutes, but no diagnosis, allocation records or exposure evidence establishes rollout causality. | ✗ |
| Kubernetes | Affected-cluster access timed out; pod events, resource limits and HPA state could not independently corroborate the findings. | ✗ |

### Discrepancies
- Initial findings describe upstream HTTP 503 responses; deeper inspection establishes only logged errorStatus=503, which may be SDK-inferred.
- Metric counts and log counts cover different windows and are not request-correlated; they cannot be reconciled one-to-one.
- Local source supports the error-to-response mechanism, but deployed dependency versions and exact deployed source parity remain unverified.
- The launch announcement is temporal context only: whitelist access can bypass experiment allocation.
- Normal fleet averages and zero observed restarts weaken broad resource-pressure hypotheses but do not exclude pod outliers or event-loop blocking; cross-channel infrastructure correlation was incomplete.

## Deep Dive Findings

{
  "status": "partially_resolved",
  "confidence": "medium",
  "summary": "Identified the image configured for the failing pods' ReplicaSet. Provider failure cause, deployed dependency versions and Statsig rollout causality remain unverified.",
  "findings": {
    "deployment": {
      "image": "gcr.io/highlevel-backend/marketplace-conversations@sha256:52910415f769f3883a1acb987758ebc623cfe510614c39225418c6301536ef3f",
      "replicaset": "conversations-api-66555f8c64",
      "evidence": "Deployment audit request and response contain this digest at 2026-11-19 16:01:30 IST. ReplicaSet audit records confirm the same digest at 16:02–16:03 IST. All five sampled failing pods share this ReplicaSet prefix.",
      "causality": "The latest matching non-scale deployment patch was over five hours before the alert; retrieved records do not support an immediate code-rollout trigger.",
      "limitations": "Registry lookup confirmed the digest but returned no build details. Installed dependency versions and individual pod runtime imageIDs remain unverified."
    },
    "provider_errors": {
      "verified": "Five logs rechecked during 21:12–21:17 IST contain four AI_StreamProviderError failures lasting 559–656ms with errorStatus=503, and one TimeoutError lasting 3006ms.",
      "missing_fields": [
        "Sanitized provider code/type",
        "Original upstream HTTP response status",
        "Upstream request ID",
        "Retry attempt timestamps"
      ],
      "source_confirmation": "Local modelErrorLogFields retains only error.name and error.statusCode. The sampled logs contain none of the requested additional fields.",
      "conclusion": "The logged 503 remains insufficient to establish an actual upstream HTTP 503 response. The fast stream failures remain distinct from the three-second timeout."
    },
    "statsig": {
      "source_configuration": {
        "whitelist_gate": "crm-ai-conv-composer-copilot",
        "experiment": "crm-ai-conv",
        "parameter": "enabled"
      },
      "important_distinction": "Local source short-circuits experiment evaluation when the whitelist gate passes. Grammar traffic therefore cannot automatically be attributed to experiment allocation.",
      "access": "No Statsig, Langfuse or native browser tool was available in this executor. No allocation or audit records were retrieved.",
      "conclusion": "Announcement timing remains correlation; allocation changes, exposure timestamps and affected cohort membership are unverified."
    }
  },
  "next_evidence_needed": [
    "Build manifest or SBOM for the identified digest to establish installed SDK versions.",
    "Sanitized provider or Langfuse observations containing original status, provider code/type, upstream request ID and attempt timestamps.",
    "Statsig audit and exposure records for both the whitelist gate and experiment, correlated with route failures."
  ],
  "log_queries_used": [
    {
      "purpose": "Exact-workload deployment history",
      "query": "resource.type=\"k8s_cluster\" resource.labels.cluster_name=\"web-users-servers-usc1-prd-cluster\" protoPayload.resourceName=~\"/deployments/conversations-api$\" NOT protoPayload.methodName:\"scale\" timestamp>=\"2026-11-18T00:00:00Z\" timestamp<=\"2026-11-19T15:53:00Z\"",
      "limit": 5,
      "result": "Four matching patches; latest at 10:31:30 UTC. A subsequent one-entry template projection recovered the image digest."
    },
    {
      "purpose": "Link failing ReplicaSet to image",
      "query": "resource.type=\"k8s_cluster\" resource.labels.cluster_name=\"web-users-servers-usc1-prd-cluster\" protoPayload.resourceName=~\"/replicasets/conversations-api-66555f8c64$\" timestamp>=\"2026-11-19T10:25:00Z\" timestamp<=\"2026-11-19T10:40:00Z\"",
      "limit": 5,
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_cluster%22%20resource.labels.cluster_name%3D%22web-users-servers-usc1-prd-cluster%22%20protoPayload.resourceName%3D~%22%2Freplicasets%2Fconversations-api-66555f8c64%24%22;timeRange=2026-11-19T10%3A25%3A00Z%2F2026-11-19T10%3A40%3A00Z?project=highlevel-backend"
    },
    {
      "purpose": "Reconfirm error types and available telemetry",
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-api\" resource.labels.cluster_name=\"web-users-servers-usc1-prd-cluster\" severity=WARNING jsonPayload.message=\"[GRAMMAR-CHECK] engine failed\" timestamp>=\"2026-11-19T15:42:00Z\" timestamp<=\"2026-11-19T15:47:00Z\"",
      "limit": 5,
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%20resource.labels.container_name%3D%22conversations-api%22%20resource.labels.cluster_name%3D%22web-users-servers-usc1-prd-cluster%22%20severity%3DWARNING%20jsonPayload.message%3D%22%5BGRAMMAR-CHECK%5D%20engine%20failed%22;timeRange=2026-11-19T15%3A42%3A00Z%2F2026-11-19T15%3A47%3A00Z?project=highlevel-backend"
    }
  ],
  "access": "Existing gcloud authentication and read-only local source inspection. Delegation failed to initialize. No production changes, messages or additional orchestrator investigations."
}

## Evidence

### Dashboards (incident window)
- [5xx percentage — conversations-api grammar check, sustained failures around 21:19 IST](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795101575209&to=1795103604327&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-api&var-method=POST&var-route=%2Fconversations%2Fgrammar%2Fcheck&var-pod_name=All&viewPanel=14)
- [Request volume by status — conversations-api grammar check, low-volume error burst](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795101575209&to=1795103604327&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-api&var-method=POST&var-route=%2Fconversations%2Fgrammar%2Fcheck&var-pod_name=All&viewPanel=1)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%20resource.labels.container_name%3D%22conversations-api%22%20severity%3DWARNING%20jsonPayload.message%3D%22%5BGRAMMAR-CHECK%5D%20engine%20failed%22;timeRange=2026-11-19T15%3A44%3A00Z%2F2026-11-19T15%3A52%3A45.747668027Z?project=highlevel-backend)

### Related incidents
- [Grammar Checker A/B launch announced 26 minutes before alert — 19 Nov](https://gohighlevel.slack.com/archives/G01FCDXTH4Z/p1795101799715659)
- [Conversations search-fix deployment and default promotion — 18 Nov](https://gohighlevel.slack.com/archives/C013CG4N7U0/p1794981674465429)
- [Message-list Go canary announced at 1% — 19 Nov](https://gohighlevel.slack.com/archives/C096N87SBFW/p1795083793657419)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 938.2s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 28.8s | ✓ |
| grafana-worker | worker | 106.5s | ✓ |
| kubectl-worker | worker | 87.3s | ✓ |
| log-explorer-worker | worker | 105.9s | ✓ |
| slack-worker | worker | 102.4s | ✓ |
| triage | worker | 22.6s | ✓ |
| triage_llm_ranking | decision | 38.1s | ✓ |
| deep_dive_pass_1 | worker | 116.4s | ✓ |
| follow_up_check | decision | 13.6s | ✓ |
| deep_dive_pass_2 | worker | 172.3s | ✓ |
| follow_up_check | decision | 12.9s | ✓ |
| deep_dive_pass_3 | worker | 109.3s | ✓ |
| cross_validation | worker | 22.0s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IG35J3KK4LC83|#140317 5XXPercentagePerAPI>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=10727950922855835787&amp;alert_id=17425008581343681945|source>*) app: conversations-api route: /conversations/grammar/check method: POST summary: 5XX Error Rate Above Threshold - Per API logs_link: <https://hyperdx.platform.prd.msgsndr.net/dashboards/6abc04ea06e1c8259f0f477e?source=6aa77741acd7321c4deda7ba&amp;where=container_name%253A%2522conversations-api%2522%2520AND%2520http_status%253A%255B500%2520TO%2520%252A%255D&amp;select=timestamp%252C%2520deployment_name%252C%2520log_level%252Clog&amp;whereLanguage=lucene&amp;filters=%255B%255D&amp;orderBy=timestamp%2520DESC&amp;from=&amp;to=&amp;isLive=true&amp;liveInterval=1800000&amp;refreshFrequency=10000&amp;mode=results&amp;denoise=false&amp;granularity=auto|here> threshold: 1% current_value: 72.72727272727272 dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-15m&amp;to=now&amp;viewPanel=14&amp;var-container=conversations-api|here> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IG35J3KK4LC83|#140317 5XXPercentagePerAPI>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=10727950922855835787&amp;alert_id=17425008581343681945|source>*) app: conversations-api route: /conversations/grammar/check method: POST summary: 5XX Error Rate Above Threshold - Per API logs_link: <https://hyperdx.platform.prd.msgsndr.net/dashboards/6abc04ea06e1c8259f0f477e?source=6aa77741acd7321c4deda7ba&amp;where=container_name%253A%2522conversations-api%2522%2520AND%2520http_status%253A%255B500%2520TO%2520%252A%255D&amp;select=timestamp%252C%2520deployment_name%252C%2520log_level%252Clog&amp;whereLanguage=lucene&amp;filters=%255B%255D&amp;orderBy=timestamp%2520DESC&amp;from=&amp;to=&amp;isLive=true&amp;liveInterval=1800000&amp;refreshFrequency=10000&amp;mode=results&amp;denoise=false&amp;granularity=auto|here> threshold: 1% current_value: 72.72727272727272 dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-15m&amp;to=now&amp;viewPanel=14&amp;var-container=conversations-api|here>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-19T16:02:16.832Z*