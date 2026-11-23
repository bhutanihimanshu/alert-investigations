# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-23-5c8b513c
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: web-users-servers-usc1-prd-cluster
**Channel**: slack
**Detected**: 2026-11-23T14:17:41.901Z
**Status**: delivering

## Root Cause

**Summary**: Unconfirmed cause of elevated POST /conversations/messages/relations latency. Shared dependency or application waiting remains plausible, but Redis, Firestore and rollout attribution lack request-level causal evidence.
**Confidence**: low
**Category**: unknown

## Triage

**Primary Anomaly**: error_spike
**Details**: Primary symptom is route latency: POST /conversations/messages/relations p99 reached 49s versus a 1.85s seven-day mean, at low traffic. Container ERROR volume also rose to 129/min versus 28–62/min; sampled errors include Redis/rate-counter timeouts, but none identify this route. Firestore-related historical candidates are plausible leads, not confirmed matches. Revision 247 overlaps onset; later scheduling failures cannot explain onset. Current healthy pods do not establish incident-time health.
**Anomalous**: Yes
**Deep Dive Direction**: YES — Shared dependency/application waiting remains the correct direction; root cause is unconfirmed.

Need slow and normal request spans from the node-local collector’s storage destination to separate Firestore RPC/retries, IAM/cache/Redis waits, and Objects/Associations timing. Also need the revision-246 middleware artifact and initialization-state evidence to verify pre-channel configuration waits and OAuth counter bypass.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | Route p99 reached 49s at low traffic. Confirms latency, but average CPU and memory do not establish the cause or exclude individual pod pressure. | ✓ |
| GCP logs | Redis/cache errors occurred near detection, but sampled errors do not identify the alerted route. No request-level link establishes that these errors caused its latency. | ✗ |
| Kubernetes | Revision 247 and CPU-driven scaling overlap detection. Later scheduling failures cannot explain onset; current healthy pods cannot establish incident-time health. | ✗ |
| Slack | Other routes reportedly experienced Firestore waits, but the linked export incident concerns a different workload. This does not independently confirm Firestore involvement in relations latency. | ✗ |
| Deep dive | No dependency spans were recovered. Local middleware bypasses OAuth counter checks for APP requests and increments counters asynchronously, weakening direct OAuth-counter attribution; incident-version behavior remains unverified. | ✗ |

### Discrepancies
- The primary verified anomaly is route latency, not route-specific errors; no two independent sources establish its causal mechanism.
- The error histogram spans clusters, while latency is cluster-scoped. Sampled error counts are not population totals.
- Low request volume makes p99 sensitive to individual requests; historical spikes exceeded this incident's peak.
- Rollout timing establishes correlation only. No revision comparison or trace proves a deployment regression.
- Falsification remains incomplete: compare slow and normal request spans for Firestore, IAM/cache, initialization and Objects/Associations waits, then verify revision-246 middleware behavior. Timed-out queries provide no negative evidence.

## Deep Dive Findings

{
  "status": "partial",
  "root_cause": "Unconfirmed. No request-level dependency spans were recovered.",
  "confidence": "Low for causal attribution.",
  "findings": [
    {
      "finding": "The live node-local collector forwards traces to the platform collector.",
      "evidence": "ConfigMap opentelemetry-collector-web-prd/opentelemetry-collector-web-users-servers-prd-agent configures the traces pipeline to export through otlphttp to https://otel-traces-http.platform.prd.msgsndr.net:443.",
      "limitation": "Current configuration establishes routing, not successful export during the incident."
    },
    {
      "finding": "Uptrace is the configured downstream trace destination in repository configuration.",
      "evidence": "helm-overrides/gke-platform-prd-usc1/otel-traces-collector/custom-values.yaml configures the traces pipeline with otlp/uptrace and debug exporters.",
      "limitation": "Live platform-cluster verification failed with EOF connecting to its private API. Uptrace storage and incident trace availability remain unverified."
    },
    {
      "finding": "Local middleware weakens OAuth counter timeouts as the explanation for the slow APP request.",
      "evidence": "Locally installed @platform-core/ratelimit 4.1.2 bypasses counter checks for non-OAUTH channels. Counter increments are also asynchronous rather than awaited.",
      "limitation": "Revision-246 package contents were not recovered; this is local-code evidence only."
    },
    {
      "finding": "Pre-channel initialization remains a possible wait, but is narrower than a recurring per-request Redis wait.",
      "evidence": "Local getConfigManagerInstance assigns its singleton before awaiting initialization; subsequent calls return that instance immediately. The initial caller awaits Redis initialization before the channel check. Resolved @platform-core/dynamic-config 1.1.0 uses a five-second connection timer, but API fallback can extend the wait.",
      "limitation": "No incident-time initialization state or duration was recovered. This does not establish that initialization delayed the sampled request."
    },
    {
      "finding": "Revision-246 artifact verification remains blocked.",
      "evidence": "Live ReplicaSet conversations-api-656f54bb95 confirms the supplied image digest. Registry manifest retrieval returned HTTP 404, including after bearer-token authentication; the gcloud artifact lookup was interrupted after returning no output.",
      "limitation": "These failures do not prove the image is absent."
    }
  ],
  "log_queries_used": [
    {
      "purpose": "Collector export errors",
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"otel-traces-collector-platform-prd\" timestamp>=\"2026-11-23T14:09:00Z\" timestamp<=\"2026-11-23T14:12:00Z\" severity>=ERROR",
      "limit": 5,
      "result": "Interrupted without results; not evidence of zero errors."
    },
    {
      "purpose": "Middleware initialization failures",
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-api\" resource.labels.cluster_name=\"web-users-servers-usc1-prd-cluster\" timestamp>=\"2026-11-23T14:09:00Z\" timestamp<=\"2026-11-23T14:12:00Z\" jsonPayload.message=~\"Failed to initialize ConfigManager|Failed to initialize dynamic config listener\"",
      "limit": 5,
      "result": "Timed out after 35 seconds; initialization state remains unknown.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.labels.container_name%3D%22conversations-api%22%0Aresource.labels.cluster_name%3D%22web-users-servers-usc1-prd-cluster%22%0Aresource.type%3D%22k8s_container%22%0AjsonPayload.message%3D~%22Failed%20to%20initialize%20ConfigManager%7CFailed%20to%20initialize%20dynamic%20config%20listener%22;timeRange=2026-11-23T14%3A09%3A00Z%2F2026-11-23T14%3A12%3A00Z?project=highlevel-backend",
      "url_validation": "Filters and time range match the attempted query; matching entries could not be verified."
    }
  ],
  "required_evidence": [
    "Uptrace spans for slow trace candidate 1d2487117050d08194df6ab77e1a342c, correlated with request d9b7475f-a283-474e-a2bd-3a0cf44cf701.",
    "A normal request's actual exported trace ID, correlated with request 4a1f96a9-c500-4d79-bd97-2ff8698fcb64.",
    "Revision-246 ratelimit and dynamic-config package contents, plus initialization timing on the slow request's pod."
  ],
  "access": "Existing gcloud authentication and explicit kubectl contexts; read-only. Subagent launch failed because executor thread context was unavailable. No messages, production changes, or additional investigations were launched."
}

## Evidence

### Dashboards (incident window)
- [P99 latency — conversations-api relations route; intermittent spikes, saved series hidden](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795441661000&to=1795446900000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-api&var-method=POST&var-route=%2Fconversations%2Fmessages%2Frelations&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&var-latencyThreshold=10&viewPanel=11)
- [Request rate — conversations-api relations route; low-volume successful traffic](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795441661000&to=1795446900000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-api&var-method=POST&var-route=%2Fconversations%2Fmessages%2Frelations&var-pod_name=All&viewPanel=1)

### Related incidents
- [Export latency with reported Firestore waits; cause unresolved — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795441064502709)
- [Cross-route latency discussion and Firestore escalation — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795433181325409)
- [isAutoReply fix promoted; production sanity completed — 23 Nov](https://gohighlevel.slack.com/archives/C013CG4N7U0/p1795423225028339)
- [Inbound-email detection change; production build shared — 23 Nov](https://gohighlevel.slack.com/archives/C096N87SBFW/p1795414948575729)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 1733.8s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.9s | ✓ |
| grafana-worker | worker | 114.1s | ✓ |
| kubectl-worker | worker | 113.8s | ✓ |
| log-explorer-worker | worker | 348.2s | ✓ |
| slack-worker | worker | 82.7s | ✓ |
| triage | worker | 16.8s | ✓ |
| triage_llm_ranking | decision | 44.7s | ✓ |
| deep_dive_pass_1 | worker | 443.6s | ✓ |
| follow_up_check | decision | 14.4s | ✓ |
| deep_dive_pass_2 | worker | 303.8s | ✓ |
| follow_up_check | decision | 15.6s | ✓ |
| deep_dive_pass_3 | worker | 204.3s | ✓ |
| cross_validation | worker | 19.9s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I2FSU1HF1SZCP|#140735 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&alert_id=15287468713337867209|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-api` route `/conversations/messages/relations` [POST] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 22s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&from=now-30m&to=now&var-datasource=ber8nnhvgsjy8f&var-container=conversations-api&var-method=POST&var-route=/conversations/messages/relations&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&var-latencyThreshold=5|here> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I2FSU1HF1SZCP|#140735 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&amp;alert_id=15287468713337867209|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `conversations-api` route `/conversations/messages/relations` [POST] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 22s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-30m&amp;to=now&amp;var-datasource=ber8nnhvgsjy8f&amp;var-container=conversations-api&amp;var-method=POST&amp;var-route=/conversations/messages/relations&amp;var-pod_name=All&amp;var-minLatency=0&amp;var-maxLatency=100000&amp;var-latencyThreshold=5|here>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-23T15:36:46.929Z*