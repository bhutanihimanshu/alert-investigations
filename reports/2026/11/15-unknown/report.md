# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-15-4c1bbc7a
**Workload**: crm-leadconnector-standard-production
**Container**: crm-leadconnector-standard-production
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-15T17:45:25.257Z
**Status**: delivering

## Root Cause

**Summary**: Undetermined. Slack confirms a downscale from R60 to R50, but the cited metrics are for the wrong time window and CPU could not be validated. Recheck Atlas CPU, memory, requests, connections, and event history around 17:45Z.
**Confidence**: low
**Category**: MongoDB autoscaling; cause unverified

## Triage

**Primary Anomaly**: none
**Details**: The alert reports sustained low CPU and post-scale memory, triggering a MongoDB downscale from R60 to R50. Grafana shows memory at 45.6%, requests at 1,771/sec, and connections at 3,298—each below its seven-day average. No matching container ERROR/WARNING logs were found. Kubernetes state and CPU percentage could not be independently verified. Historical candidates do not match this MongoDB autoscaling event.
**Anomalous**: No
**Deep Dive Direction**: YES — The triage direction is still correct: verify Atlas CPU, memory, and event history around both scaling events. The available metrics suggest demand had fallen, but they don’t confirm the tier changes or the autoscaler’s decision.

YES — More data is needed to determine root cause: Atlas event history and reliable CPU metrics around both scaling events. The cause of the earlier request surge also remains unestablished.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Slack alert and thread | The alert reports a downscale from R60 to R50 at 17:45Z after sustained low CPU and post-scale memory. A reply confirms the cluster was downscaled. | ✓ |
| Grafana | The cited request, connection, and memory values are below seven-day averages, but the metrics were read around 12:15Z, not the 17:45Z alert time. CPU could not be reliably verified. | ✗ |
| GCP logs | No matching ERROR or WARNING entries were returned around 17:45Z. This does not establish Atlas metrics or autoscaler decisions. | ✗ |
| Kubernetes | No matching workload was found in the available workers-cluster context; Kubernetes state and events remain unverified. | ✗ |
| Earlier same-day upscale investigation | The earlier R50-to-R60 window showed elevated MongoDB load and Facebook worker activity, but did not establish the surge trigger or Atlas scaling decisions. | ✗ |

### Discrepancies
- The Grafana metrics cited for the downscale are around 12:15Z, approximately 5.5 hours before the 17:45Z alert.
- Slack confirms the downscale event, but available evidence does not independently verify the CPU and memory conditions or the autoscaler's decision.
- The prior elevated-load measurements concern the earlier upscale window and cannot establish the later downscale cause.

## Deep Dive Findings

{
  "finding": "Root cause remains undetermined. A timestamp discrepancy undermines the previous metric comparison: the alert text says 5:45:21 PM and its Slack timestamp is 17:45:25Z, but the Grafana metrics cited for the downscale were read around 12:15Z—a 5.5-hour offset. Those lower memory, request, and connection values do not verify conditions at the apparent alert time. The earlier R50→R60 investigation is consistent with a load surge, with Facebook worker activity and elevated MongoDB metrics, but the surge trigger and Atlas scaling decisions remain unconfirmed.",
  "confidence": "Low for the downscale cause; medium that Facebook worker activity materially contributed to the earlier measured surge.",
  "evidence": [
    {
      "source": "Alert and Slack timestamps",
      "result": "The alert body reports 5:45:21 PM on Nov 15, while its Slack message timestamp is 17:45:25Z. The prior downscale Grafana links cover 12:15Z, not 17:45Z. The reported metrics therefore do not establish Atlas conditions at the alert time.",
      "urls": [
        "https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794764725266849?thread_ts=1794764725.266849"
      ]
    },
    {
      "source": "Earlier R50→R60 investigation",
      "result": "The recorded 04:23–05:23Z window showed one host at 95.5% CPU, cluster requests around 5,491–5,831/sec, and MongoDB operation p95 around 1.2s. The Facebook worker was the largest measured application contributor. This supports elevated load during that measured window, but not the trigger or Atlas autoscaler’s decision.",
      "urls": [
        "https://prod.grafana.leadconnectorhq.com/d/_s7Pjkb7z/dedicated-atlas-clusters-hardware-metrics?orgId=1&var-Datasource=ber8nnhvgsjy8f&var-job=highlevel-prod-mongo-metrics&var-group_id=60803ad4b9f71f2ec1288526&var-cl_name=crm-leadconnector-standard-production&var-rs_nm=%24__all&var-host=%24__all&var-process_port=%24__all&var-interval=1m&from=1794716580000&to=1794720180000&viewPanel=44",
        "https://prod.grafana.leadconnectorhq.com/d/a019c5e8-473e-4431-806c-5704cb544110/mongo-observability?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=%24__all&var-team=platform&var-sub_team=%24__all&var-app=crm-integrations-facebook-worker&var-mongo_host_name=crm-leadconnector-stand-shard-00-00-pri.hfkhk.mongodb.net&var-mongo_collection=%24__all&var-mongo_method_name=%24__all&from=1794716580000&to=1794720180000&viewPanel=20",
        "https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794718405497649?thread_ts=1794718405.497649"
      ]
    },
    {
      "source": "GCP logs",
      "result": "A targeted query found repeated Facebook lead-processing messages from one worker pod around 23:55Z on Nov 14. This confirms activity at that time, but does not identify the triggering event or establish a causal link to the measured MongoDB surge. The earlier downscale-window ERROR/WARNING checks found no entries for the MongoDB cluster name; this does not verify Atlas metrics or autoscaling events.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-integrations-facebook-worker%22%0Atimestamp%3E%3D%222026-11-14T23%3A00%3A00Z%22%0Atimestamp%3C%3D%222026-11-15T00%3A00%3A00Z%22%0AjsonPayload.message%3A%22Contact%20Obj%20created%20for%20lead%22;timeRange=2026-11-14T23%3A00%3A00Z%2F2026-11-15T00%3A00%3A00Z?project=highlevel-backend"
    },
    {
      "source": "Atlas and Kubernetes access",
      "result": "Atlas event history and reliable CPU around both scaling events remain unavailable in the prior checks. Kubernetes checks used a workers-cluster context and found no matching MongoDB workload, so they cannot establish Atlas tier changes or cause."
    }
  ],
  "log_queries": [
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"crm-integrations-facebook-worker\" AND timestamp>=\"2026-11-14T23:00:00Z\" AND timestamp<=\"2026-11-15T00:00:00Z\" AND jsonPayload.message:\"Contact Obj created for lead\"",
      "result": "Returned 10 matching lead-processing messages from one pod, spanning at least 23:55:13–23:55:27Z. This confirms processing activity only; it does not establish the surge trigger.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-integrations-facebook-worker%22%0Atimestamp%3E%3D%222026-11-14T23%3A00%3A00Z%22%0Atimestamp%3C%3D%222026-11-15T00%3A00%3A00Z%22%0AjsonPayload.message%3A%22Contact%20Obj%20created%20for%20lead%22;timeRange=2026-11-14T23%3A00%3A00Z%2F2026-11-15T00%3A00%3A00Z?project=highlevel-backend"
    }
  ],
  "limitations": [
    "Recheck Atlas CPU, memory, requests, connections, and event history using the correct downscale window around 17:45Z on Nov 15; the cited 12:15Z metrics are offset by 5.5 hours.",
    "The available evidence does not identify what initiated the earlier request surge or confirm the autoscaler’s evaluation and tier changes.",
    "Kubernetes state and container logs cannot independently confirm Atlas cluster operations."
  ]
}

## Evidence

### Dashboards (incident window)
- [Host memory used — crm-leadconnector-standard-production (about 46% at alert; 7-day average about 85%)](https://prod.grafana.leadconnectorhq.com/d/_s7Pjkb7z/dedicated-atlas-clusters-hardware-metrics?orgId=1&var-Datasource=ber8nnhvgsjy8f&var-job=highlevel-prod-mongo-metrics&var-group_id=60803ad4b9f71f2ec1288526&var-cl_name=crm-leadconnector-standard-production&from=1794743100000&to=1794746700000&viewPanel=13)
- [MongoDB request rate — crm-leadconnector-standard-production (below 7-day average at alert)](https://prod.grafana.leadconnectorhq.com/d/W0lo7Gx7z/dedicated-atlas-clusters-mongo-metrics?orgId=1&var-group_id=60803ad4b9f71f2ec1288526&var-cl_name=crm-leadconnector-standard-production&from=1794743100000&to=1794746700000&viewPanel=88)
- [MongoDB connections — crm-leadconnector-standard-production (below 7-day average at alert)](https://prod.grafana.leadconnectorhq.com/d/W0lo7Gx7z/dedicated-atlas-clusters-mongo-metrics?orgId=1&var-group_id=60803ad4b9f71f2ec1288526&var-cl_name=crm-leadconnector-standard-production&from=1794743100000&to=1794746700000&viewPanel=81)
- [Dedicated Atlas Clusters Hardware Metrics — panel 44](https://prod.grafana.leadconnectorhq.com/d/_s7Pjkb7z/dedicated-atlas-clusters-hardware-metrics?orgId=1&var-Datasource=ber8nnhvgsjy8f&var-job=highlevel-prod-mongo-metrics&var-group_id=60803ad4b9f71f2ec1288526&var-cl_name=crm-leadconnector-standard-production&var-rs_nm=%24__all&var-host=%24__all&var-process_port=%24__all&var-interval=1m&from=1794716580000&to=1794720180000&viewPanel=44)

### Related incidents
- [Downscale confirmed — 15 Nov](https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794764725266849?thread_ts=1794764725.266849)
- [Earlier same-day upscale — 15 Nov](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794718405497649?thread_ts=1794718405.497649)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 769.5s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 12.8s | ✓ |
| grafana-worker | worker | 140.7s | ✓ |
| kubectl-worker | worker | 110.7s | ✓ |
| log-explorer-worker | worker | 65.0s | ✓ |
| slack-worker | worker | 56.2s | ✓ |
| triage | worker | 14.5s | ✓ |
| triage_llm_ranking | decision | 21.8s | ✓ |
| deep_dive_pass_1 | worker | 148.9s | ✓ |
| follow_up_check | decision | 13.0s | ✓ |
| deep_dive_pass_2 | worker | 46.1s | ✓ |
| follow_up_check | decision | 11.5s | ✓ |
| deep_dive_pass_3 | worker | 110.2s | ✓ |
| cross_validation | worker | 18.2s | ✓ |

## Alert Context

```
||CLUSTER - crm-leadconnector-standard-production, METRIC - Cluster is Autoscaling =&gt; Downscale :bar_chart: *MongoDB Auto-Scale Alert* :label: *Cluster:* crm-leadconnector-standard-production :chart_with_downwards_trend: *Tier Change:* Base Tier: R60 -&gt; New Tier: R50 :moneybag: *New Rate:* $1.31/hr :alarm_clock: *Time:* 11/15/2026, 5:45:21 PM :memo: *Reason:* crm-leadconnector-standard-production has fallen below 45% CPU usage for the past 4 hours and the post-scaling memory usage of your cluster has remained below 60% for the past 4 hours as of 12:44 PM E... CLUSTER - crm-leadconnector-standard-production, METRIC - Cluster is Autoscaling =&gt; Downscale :bar_chart: *MongoDB Auto-Scale Alert* :label: *Cluster:* crm-leadconnector-standard-production :chart_with_downwards_trend: *Tier Change:* Base Tier: R60 -&gt; New Tier: R50 :moneybag: *New Rate:* $1.31/hr :alarm_clock: *Time:* 11/15/2026, 5:45:21 PM :memo: *Reason:* crm-leadconnector-standard-production has fallen below 45% CPU usage for the past 4 hours and the post-scaling memory usage of your cluster has remained below 60% for the past 4 hours as of 12:44 PM E...
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-15T17:55:13.007Z*