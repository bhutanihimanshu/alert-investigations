# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-23-81dad01b
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: web-users-servers-usc1-prd-cluster
**Channel**: slack
**Detected**: 2026-11-23T12:37:42.582Z
**Status**: delivering

## Root Cause

**Summary**: Firestore wait directly explains one slow snippets write. Independently observed database availability errors overlap that request, supporting a transient shared Firestore disruption as a contributor to the latency alert. The backend trigger, request-level retries and exact histogram amplification remain unconfirmed.
**Confidence**: medium
**Category**: dependency_latency_firestore

## Triage

**Primary Anomaly**: none
**Details**: The anomaly is latency, which is absent from the allowed categories: snippets-api POST /snippets/:locationId p95 reached 32s during 18:01–18:06:30 IST and recovered to 74ms by 18:08. Metrics report one request exceeding 10s in the preceding hour, without a concurrent traffic surge, CPU spike, 5xx increase or restarts. JWT warnings do not establish causation. Historical matches largely reflect the incorrect CRM-conversations routing label; the earlier live-chat Firestore incident is a dependency lead, not a confirmed match.
**Anomalous**: Yes
**Deep Dive Direction**: YES — the Firestore direction remains correct. Shared slowdown is supported, but its underlying trigger is unconfirmed.

Need request-level Firestore RPC spans with retry attempts, statuses and server timing, correlated database telemetry, and image-to-commit provenance to verify deployed source.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| Request logs | A successful 201 response completed at 18:00:56 IST in 15.565s; templates.set consumed 15.558s, directly confirming Firestore wait dominated this request. | ✓ |
| Firestore telemetry | Commit UNAVAILABLE responses occurred during 17:59–18:01 IST, overlapping the slow request. Read failures also increased, supporting shared disruption without proving this write failed or retried. | ✓ |
| Grafana | Route p95 peaked at 32s and recovered by 18:08 IST. One request exceeded 10s in the preceding hour; no concurrent traffic surge, CPU spike, 5xx increase or restarts was reported. | ✓ |
| Kubernetes | Zero restarts and a rollout dating to November 12 weaken crash and deployment hypotheses. Current resource measurements cannot exclude alert-time pressure. | ✓ |
| Slack | Reports of cross-service Firestore delays support the dependency hypothesis, but contain no independently correlated snippets request. | ✓ |

### Discrepancies
- The 32s histogram p95 differs from the logged 15.565s request. Histogram buckets and the exact alert expression remain unverified; 32s is not a confirmed individual request duration.
- The request completed before the 18:01–18:06:30 metric elevation. Rolling-window effects are plausible but not verified.
- Database aggregates cannot attribute UNAVAILABLE responses or retry attempts to this specific write; Cloud Trace lookup was blocked by quota errors.
- Falsification checks weaken quota and contention explanations: Commit RESOURCE_EXHAUSTED and DEADLINE_EXCEEDED were zero, and ABORTED stayed within the local baseline. Seven-day database baseline and event-loop evidence are missing.
- JWT warnings peaked later and lack a causal link. The merged Outlook investigation concerns a different workload.

## Deep Dive Findings

{
  "status": "shared_firestore_disruption_supported_trigger_unconfirmed",
  "workload": "snippets-api",
  "summary": "New database telemetry shows transient UNAVAILABLE responses overlapping the slow snippets write. Image-to-commit provenance is now resolved. Request-level retries and the underlying Firestore trigger remain unconfirmed.",
  "findings": {
    "request": {
      "request_id": "6ffee1ea-ff10-4bcf-a144-124f33134d4d",
      "completion_ist": "2026-11-23T18:00:56.464+05:30",
      "http_status": 201,
      "duration_ms": 15565,
      "firestore_duration_ms": 15557.515,
      "operation": "templates.set",
      "document_count": 1,
      "database": "(default)",
      "assessment": "Reverified: approximately 99.95% of logged request duration was spent awaiting Firestore."
    },
    "database_telemetry": {
      "metric": "firestore.googleapis.com/api/request_count",
      "scope": "highlevel-backend, database=(default), location=nam5",
      "aggregation": "60-second ALIGN_SUM",
      "unavailable_responses": [
        {
          "interval_ist": "17:59–18:00",
          "Commit": 56,
          "BatchGetDocuments": 227,
          "RunQuery": 98
        },
        {
          "interval_ist": "18:00–18:01",
          "Commit": 17,
          "BatchGetDocuments": 68,
          "RunQuery": 29
        }
      ],
      "comparison": "These three methods recorded zero UNAVAILABLE responses in the other queried minute buckets during 17:55–18:07 IST.",
      "additional_evidence": "BatchGetDocuments DEADLINE_EXCEEDED increased from 21–51/min during 17:55–17:59 to 531 and 591 during 18:00–18:02 IST.",
      "counterevidence": "Commit RESOURCE_EXHAUSTED and DEADLINE_EXCEEDED remained zero throughout the queried window. Commit ABORTED was 185–204/min during 17:59–18:01, within the preceding 162–224/min range.",
      "interpretation": "Supports a transient shared Firestore availability/latency disruption. Does not establish quota exhaustion, a contention spike, or retries on this individual write."
    },
    "request_tracing": {
      "correction": "The UUID in the log trace field equals x-request-id. It is not the request's B3 trace ID.",
      "b3_trace_id": "b6fab27a061b3e9ccafff14d8b2c4d47",
      "b3_span_id": "cafff14d8b2c4d47",
      "b3_sampled": "0",
      "log_span_id_present": false,
      "cloud_trace_results": "Normalized UUID lookup returned 404. Both window searches and the subsequent actual B3 trace lookup returned 429 read-quota errors.",
      "assessment": "Incoming B3 sampling was disabled; this does not prove every instrumentation path was unsampled. No RPC attempts, statuses or server timing were retrieved."
    },
    "image_provenance": {
      "image_index_digest": "sha256:afbf48ba7ecbf62c06fafafc9b6a54da11a69de584a8e6ebf3e9d271c827e775",
      "source_revision": "a536a87b18b374445ee703fd755107fcc55dd666",
      "source_repository": "https://github.com/GoHighLevel/marketplace-backend",
      "evidence": "OCI index links its amd64 image to a SLSA v1 attestation containing vcs:revision and vcs:source. The revision also exists in local Git history.",
      "image_created_ist": "2026-11-12T17:59:48.667+05:30",
      "correction": "Previous manifest 404 was caused by an incompatible Accept media type; accepting OCI media types succeeded.",
      "qualification": "Registry-attested revision identified; attestation signature and build-context cleanliness were not independently verified.",
      "source_url": "https://github.com/GoHighLevel/marketplace-backend/commit/a536a87b18b374445ee703fd755107fcc55dd666"
    }
  },
  "confidence": {
    "firestore_wait_dominated_request": "high",
    "shared_firestore_disruption_contributed": "medium",
    "individual_write_retried": "unknown",
    "underlying_backend_trigger": "unknown"
  },
  "next_action": "Use the verified database error window and B3 identifiers for a platform/GCP support investigation of backend availability. Request-level retry or server timing evidence is still required before prescribing retry or capacity changes.",
  "limitations": [
    "Cloud Trace read-quota errors prevented lookup of the actual B3 trace.",
    "No seven-day database baseline was collected; rarity beyond the local window is unknown.",
    "Database aggregates cannot associate an UNAVAILABLE response with this specific snippets write.",
    "Live deployment tracing settings were not retrieved because the attempted Kubernetes endpoint timed out."
  ],
  "log_queries_used": [
    {
      "query": "trace=\"projects/highlevel-backend/traces/6ffee1ea-ff10-4bcf-a144-124f33134d4d\" timestamp>=\"2026-11-23T12:30:00Z\" timestamp<=\"2026-11-23T12:31:00Z\"",
      "limit": 5,
      "result": "Three entries; verified response duration, Firestore duration and tracing headers."
    }
  ],
  "metric_query_used": {
    "filter": "metric.type=\"firestore.googleapis.com/api/request_count\" AND resource.labels.database_id=\"(default)\"",
    "start": "2026-11-23T12:25:00Z",
    "end": "2026-11-23T12:37:00Z",
    "alignment_period": "60s",
    "aligner": "ALIGN_SUM",
    "pagination_complete": true
  },
  "log_explorer_urls": [
    {
      "name": "Request and Firestore timing",
      "url": "https://console.cloud.google.com/logs/query;query=trace%3D%22projects%2Fhighlevel-backend%2Ftraces%2F6ffee1ea-ff10-4bcf-a144-124f33134d4d%22;timeRange=2026-11-23T12%3A30%3A00Z%2F2026-11-23T12%3A31%3A00Z?project=highlevel-backend"
    }
  ],
  "access": "Existing local gcloud authentication, read-only Google APIs, registry metadata and local source inspection. No production changes or messages posted."
}

## Evidence

### Dashboards (incident window)
- [P95 latency — snippets-api POST /snippets/:locationId; inspect the transient spike (panel aggregates clusters)](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795433880000&to=1795438080000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=snippets-api&var-method=POST&var-route=%2Fsnippets%2F%3AlocationId&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&viewPanel=17)
- [Request rate — snippets-api POST /snippets/:locationId; low traffic during the latency spike](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795433880000&to=1795438080000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=snippets-api&var-method=POST&var-route=%2Fsnippets%2F%3AlocationId&var-pod_name=All&viewPanel=1)

### Logs
- [Request and Firestore timing](https://console.cloud.google.com/logs/query;query=trace%3D%22projects%2Fhighlevel-backend%2Ftraces%2F6ffee1ea-ff10-4bcf-a144-124f33134d4d%22;timeRange=2026-11-23T12%3A30%3A00Z%2F2026-11-23T12%3A31%3A00Z?project=highlevel-backend)

### Related incidents
- [Merge target investigates Outlook OAuth, not snippets — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795436773172769)
- [Team reports cross-service Firestore delays — 23 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1795434619320709)
- [Conversations isAutoReply deployment and production sanity — 23 Nov](https://gohighlevel.slack.com/archives/C013CG4N7U0/p1795423225028339)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 1716.5s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 17.9s | ✓ |
| grafana-worker | worker | 127.7s | ✓ |
| kubectl-worker | worker | 125.5s | ✓ |
| log-explorer-worker | worker | 372.7s | ✓ |
| slack-worker | worker | 81.1s | ✓ |
| triage | worker | 19.3s | ✓ |
| triage_llm_ranking | decision | 47.8s | ✓ |
| deep_dive_pass_1 | worker | 332.3s | ✓ |
| follow_up_check | decision | 15.1s | ✓ |
| deep_dive_pass_2 | worker | 276.9s | ✓ |
| follow_up_check | decision | 15.9s | ✓ |
| deep_dive_pass_3 | worker | 260.7s | ✓ |
| cross_validation | worker | 23.4s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I3XTJVMVXRPEI|#140721 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&alert_id=8578529744200094170|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `snippets-api` route `/snippets/:locationId` [POST] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 32s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&from=now-30m&to=now&var-datasource=ber8nnhvgsjy8f&var-container=snippets-api&var-method=POST&var-route=/snippets/:locationId&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&var-latencyThreshold=5|here> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/I3XTJVMVXRPEI|#140721 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&amp;alert_id=8578529744200094170|source>*) cluster: web-users-servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `snippets-api` route `/snippets/:locationId` [POST] on web-users-servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 32s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-30m&amp;to=now&amp;var-datasource=ber8nnhvgsjy8f&amp;var-container=snippets-api&amp;var-method=POST&amp;var-route=/snippets/:locationId&amp;var-pod_name=All&amp;var-minLatency=0&amp;var-maxLatency=100000&amp;var-latencyThreshold=5|here>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-23T13:35:18.789Z*