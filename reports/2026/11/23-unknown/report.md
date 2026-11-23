# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-23-25f881e3
**Workload**: CRM-conversations
**Container**: CRM-conversations
**Cluster**: servers-usc1-prd-cluster
**Channel**: slack
**Detected**: 2026-11-23T11:15:41.929Z
**Status**: delivering

## Root Cause

**Summary**: A low-volume Outlook delivery latency spike is corroborated by metrics and two approximately 69-second failed delivery attempts. Microsoft Graph mailbox-move failures are associated with those attempts, but evidence does not establish whether upstream waiting, SDK retries or subsequent authentication caused the delay. Root cause remains unconfirmed.
**Confidence**: low
**Category**: undetermined

## Triage

**Primary Anomaly**: none
**Details**: Latency is the primary anomaly, outside the supplied enum: POST /leadconnector/outlook/deliver reached histogram-capped p95 of 30s, with approximately 2 of 13 requests exceeding 30s. Traffic and average resources were not elevated; no restarts were observed. Historical candidates match misparsed CRM-conversations/Unknown labels, not this incident. The 16:27 IST CORS error peak coincided with scale-out and precedes the latency rise at 16:40 IST; causation is unproven.
**Anomalous**: Yes
**Deep Dive Direction**: YES — The triage direction remains correct: both ~69-second requests encountered Microsoft Graph mailbox-move errors, but the delay mechanism is unconfirmed.

Need Graph per-attempt durations, upstream statuses, Retry-After values and actual retry timings, plus token-acquisition completion and lookup/hydration/lock/refresh durations to distinguish upstream waiting, SDK backoff and authentication delay.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | Route latency rose to histogram-capped p95 of 30s around 16:40 IST; approximately 2 of 13 requests exceeded 30s. Traffic and average resource usage were not elevated. | ✓ |
| Application logs | Two delivery attempts failed after approximately 69s at 16:38:05 and 16:40:08 IST, consistent with the latency anomaly. Initial token hydration completed within 13ms; Graph attempt and subsequent authentication durations are unavailable. | ✓ |
| Kubernetes | Zero restarts and 30/30 current pods ready support the absence of a restart-driven incident. Scale-out at 16:27 IST preceded the slow requests but has no demonstrated causal connection. | ✓ |
| Slack | Confirms alert acknowledgement and an Outlook error-handling deployment approximately five hours earlier; provides no independent diagnosis of the delay. | ✗ |

### Discrepancies
- Metrics and logs corroborate slow requests, not the exact delay mechanism. Mailbox-move errors were reported in an earlier pass but their nested details were not reverified in the supplied deep dive.
- The 30s percentile is histogram-capped, not a maximum request duration; approximately 69s attempts are compatible with it.
- Fast hydration argues against initial token-store delay but does not exclude later token refresh, authentication, Graph waiting or retry backoff. Local source correspondence to the deployed image is unverified.
- CORS errors peaked at 16:27 IST, before the latency rise; no causal connection is established. Cross-container infrastructure checks were not performed.
- Falsification remains incomplete: per-attempt Graph timing and authentication completion are needed to test upstream-delay attribution. Fleet averages cannot exclude individual-pod or event-loop stalls; a shared trace ID prevents unique request correlation by trace alone.

## Deep Dive Findings

{
  "workload": "leadconnector-api",
  "route": "POST /leadconnector/outlook/deliver",
  "conclusion": "New evidence argues against slow initial token lookup/hydration: both requests logged KV2_READ_OK within 13ms of token-acquisition start. The remaining approximately 69 seconds cannot be separated into Graph request time, retry backoff or subsequent authentication work.",
  "new_findings": [
    {
      "pod_suffix": "zv2xf",
      "timestamps_ist": {
        "token_start": "2026-11-23T16:36:55.738+05:30",
        "kv2_read_ok": "2026-11-23T16:36:55.750+05:30",
        "delivery_failure": "2026-11-23T16:38:05.001+05:30"
      },
      "token_start_to_kv2_read_ok_ms": 12.02,
      "kv2_read_ok_to_failure_seconds": 69.251
    },
    {
      "pod_suffix": "4rj8v",
      "timestamps_ist": {
        "token_start": "2026-11-23T16:38:59.388+05:30",
        "kv2_read_ok": "2026-11-23T16:38:59.398+05:30",
        "delivery_failure": "2026-11-23T16:40:08.795+05:30"
      },
      "token_start_to_kv2_read_ok_ms": 10.232,
      "kv2_read_ok_to_failure_seconds": 69.396
    }
  ],
  "interpretation": {
    "source_confirmation": "Local twoWaySyncTokenStore.ts emits KV2_READ_OK after assigning hydrated tokens and applying expiry. getAccessToken checks expiry afterward.",
    "authentication": "No matching Outlook expiry, refresh, lock or token-completion message appeared in the filtered results. This weakens an authentication-delay hypothesis but does not conclusively exclude it.",
    "graph": "Local SDK retries HTTP 429, 503 and 504 and honors Retry-After. Actual upstream statuses, attempt durations and sleep intervals remain unavailable.",
    "prior_evidence": "Mailbox-move error details remain supported by the supplied previous pass; this pass reconfirmed failure timestamps, not those nested error details."
  },
  "confidence": {
    "fast_initial_hydration": "medium",
    "exact_delay_mechanism": "unconfirmed"
  },
  "limitations": [
    "Local source correspondence to the deployed image remains unverified.",
    "Both requests share one trace identifier; timestamps and pods distinguish them.",
    "No per-attempt Graph spans or complete authentication duration measurements were recovered."
  ],
  "needed_evidence": "Distinct delivery-attempt IDs; Graph attempt start/end, status, Retry-After and actual retry sleep; token-acquisition completion and lookup/hydration/lock/refresh durations. Exclude tokens and email content.",
  "log_queries_used": {
    "base": "resource.type=\"k8s_container\" resource.labels.container_name=\"leadconnector-api\" resource.labels.cluster_name=\"servers-usc1-prd-cluster\" timestamp>=\"2026-11-23T11:06:50Z\" timestamp<=\"2026-11-23T11:10:15Z\"",
    "failure_filter": "jsonPayload.message=\"[OUTLOOK_DELIVER_REPLY_FAILED]\"",
    "timing_filter": "(resource.labels.pod_name=\"leadconnector-api-5d589d5c66-zv2xf\" OR resource.labels.pod_name=\"leadconnector-api-5d589d5c66-4rj8v\") (jsonPayload.message=~\"[Tt]oken|[Rr]etry|[Gg]raph|OUTLOOK_DELIVER|[Mm]ail delivery|[Ll]ock|KV2\" OR jsonPayload.message:\"TFYDe1F33wsDEzePC1KN\")",
    "results": "Failure query returned 2 records with limit=5; timing query returned 16 records with limit=80."
  },
  "log_explorer_urls": [
    "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22leadconnector-api%22%0Aresource.labels.cluster_name%3D%22servers-usc1-prd-cluster%22%0AjsonPayload.message%3D%22%5BOUTLOOK_DELIVER_REPLY_FAILED%5D%22;timeRange=2026-11-23T11%3A06%3A50Z%2F2026-11-23T11%3A10%3A15Z?project=highlevel-backend"
  ],
  "access": "Existing authenticated gcloud CLI and local source; read-only. No production changes or messages posted."
}

## Evidence

### Dashboards (incident window)
- [Outlook delivery latency — leadconnector-api: p95 reaches 30s at 16:40 IST](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795430741000&to=1795434341000&var-datasource=ber8nnhvgsjy8f&var-cluster=servers-usc1-prd-cluster&var-container=leadconnector-api&var-method=POST&var-route=%2Fleadconnector%2Foutlook%2Fdeliver&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&viewPanel=17)
- [Outlook delivery traffic — leadconnector-api: low volume and response classes](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795430741000&to=1795434341000&var-datasource=ber8nnhvgsjy8f&var-cluster=servers-usc1-prd-cluster&var-container=leadconnector-api&var-method=POST&var-route=%2Fleadconnector%2Foutlook%2Fdeliver&var-pod_name=All&viewPanel=1)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22leadconnector-api%22%0Aresource.labels.cluster_name%3D%22servers-usc1-prd-cluster%22%0Aseverity%3E%3DERROR%0A-jsonPayload.message%3D~%22Failed%20to%20configure%20CORS%20for%20storage%20buckets%22;timeRange=2026-11-23T10%3A52%3A00Z%2F2026-11-23T11%3A05%3A00Z?project=highlevel-backend)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22leadconnector-api%22%0Aresource.labels.cluster_name%3D%22servers-usc1-prd-cluster%22%0AjsonPayload.message%3D%22%5BOUTLOOK_DELIVER_REPLY_FAILED%5D%22;timeRange=2026-11-23T11%3A06%3A50Z%2F2026-11-23T11%3A10%3A15Z?project=highlevel-backend)

### Related incidents
- [Outlook error handling change and production promotion — 23 Nov](https://gohighlevel.slack.com/archives/C096N87SBFW/p1795064245350139)
- [Gmail delivery fix deployed and production sanity passed — 23 Nov](https://gohighlevel.slack.com/archives/C096N87SBFW/p1795411885150389)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 1442.3s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 16.6s | ✓ |
| grafana-worker | worker | 87.8s | ✓ |
| kubectl-worker | worker | 210.0s | ✓ |
| log-explorer-worker | worker | 361.4s | ✓ |
| slack-worker | worker | 69.5s | ✓ |
| triage | worker | 18.4s | ✓ |
| triage_llm_ranking | decision | 42.8s | ✓ |
| deep_dive_pass_1 | worker | 276.1s | ✓ |
| follow_up_check | decision | 16.5s | ✓ |
| deep_dive_pass_2 | worker | 156.6s | ✓ |
| follow_up_check | decision | 13.1s | ✓ |
| deep_dive_pass_3 | worker | 152.6s | ✓ |
| cross_validation | worker | 21.0s | ✓ |

## Alert Context

```
||*<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IXTQ3FQYDIJBP|#140711 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&alert_id=15042120747186146505|source>*) cluster: servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `leadconnector-api` route `/leadconnector/outlook/deliver` [POST] on servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 30s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&from=now-30m&to=now&var-datasource=ber8nnhvgsjy8f&var-container=leadconnector-api&var-method=POST&var-route=/leadconnector/outlook/deliver&var-pod_name=All&var-minLatency=0&var-maxLatency=100000&var-latencyThreshold=5|here> *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/IXTQ3FQYDIJBP|#140711 HTTPLatencyPerSubTeam>* via CRM-conversations (*<https://vmalert.platform.prd.msgsndr.net/vmalert/alert?group_id=7041113084036845597&amp;alert_id=15042120747186146505|source>*) cluster: servers-usc1-prd-cluster summary: Application Latency Above Threshold threshold: 10s description: Latency for application - `leadconnector-api` route `/leadconnector/outlook/deliver` [POST] on servers-usc1-prd-cluster is above threshold for the last 5 minutes current_value: 30s dashboard_link: <https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f/api-requests-overview?orgId=1&amp;from=now-30m&amp;to=now&amp;var-datasource=ber8nnhvgsjy8f&amp;var-container=leadconnector-api&amp;var-method=POST&amp;var-route=/leadconnector/outlook/deliver&amp;var-pod_name=All&amp;var-minLatency=0&amp;var-maxLatency=100000&amp;var-latencyThreshold=5|here>
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-23T11:37:14.257Z*