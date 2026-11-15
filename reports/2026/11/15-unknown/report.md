# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-15-a6aa7eba
**Workload**: crm-leadconnector-standard-production
**Container**: crm-leadconnector-standard-production
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-15T04:53:25.497Z
**Status**: delivering

## Root Cause

**Summary**: Root cause is undetermined. The Facebook worker was a major measured MongoDB contributor and processed leads during the surge, but the evidence does not establish that marketplace-modules-fb-missing-leads-analysis-cron initiated the load.
**Confidence**: medium
**Category**: MongoDB request-rate surge; trigger unconfirmed

## Triage

**Primary Anomaly**: traffic_spike
**Details**: MongoDB request rate rose to about 5,500–5,800/s from a pre-event median of 1,368/s; crm-integrations-facebook-worker was the largest observed contributor, at about 3,000–3,500/s versus 29.7/s baseline. One host reached 95.5% CPU and Mongo operation p95 latency rose to about 1.2s. Slack reports the marketplace-modules-fb-missing-leads-analysis-cron as a possible cause, but this is not independently verified. Historical candidates do not establish a matching MongoDB autoscale RCA.
**Anomalous**: Yes
**Deep Dive Direction**: YES — the triage direction is still correct: investigate the Facebook worker and correlate its workload with the reported cron. The evidence does not establish the cron as the trigger or point to a different cause.

More data is needed: cron execution times, Facebook worker message volume and MongoDB operations by event type around those runs, plus Atlas scale-up history and post-scale CPU, request-rate, and latency metrics.

## Cross-Validation

**Confidence**: medium

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana MongoDB metrics | During the alert window, cluster request rate rose to about 5,500–5,800/s, one host reached 95.5% CPU, and operation p95 latency reached about 1.2s. The Facebook worker was the largest observed application contributor, at about 3,000–3,500 requests/s versus a 29.7/s baseline. | ✓ |
| GCP Facebook worker logs | Two lead-processing log entries from one worker pod occurred during the surge. They confirm activity but do not identify the triggering event or link it to the named cron. | ✓ |
| Facebook worker source | The worker's lead-sync path performs MongoDB reads and contact processing, making it a plausible contributor; the available evidence does not tie the logged work to that path or cron. | ✓ |
| Slack context | A nearby thread attributes the database activity to marketplace-modules-fb-missing-leads-analysis-cron, but this remains a team-reported hypothesis without independent verification. | ✗ |
| GCP Kubernetes/log checks | The MongoDB cluster is not a Kubernetes workload in the checked context, and the container log query for that cluster name returned no entries; these checks do not explain the autoscaling trigger. | ✗ |

### Discrepancies
- Metrics and worker logs support elevated MongoDB load and Facebook lead processing during the surge, but do not establish that the named cron triggered it.
- The nearby Slack CPU alert reports 38.5 against a threshold of 75; it is a distinct metric from the MongoDB host CPU above 90%.
- Cron run history, worker event volume and MongoDB operations by event type, and Atlas scale-up history were not verified.

## Deep Dive Findings

{
  "finding": "The Facebook worker was processing Facebook leads during the MongoDB surge, strengthening the case that it contributed to load. The available evidence still does not show that marketplace-modules-fb-missing-leads-analysis-cron triggered the activity; root cause remains undetermined.",
  "evidence": [
    {
      "source": "GCP worker logs",
      "result": "Two `Facebook: Contact Obj created for lead` entries came from the same worker pod at 05:19:13 and 05:19:15 UTC, within the 04:23–05:23 UTC surge window. They confirm lead processing at that time, but do not identify the event type or cron that initiated it.",
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"crm-integrations-facebook-worker\" AND timestamp>=\"2026-11-15T05:15:00Z\" AND timestamp<=\"2026-11-15T05:22:00Z\" AND jsonPayload.message=\"Facebook: Contact Obj created for lead\"",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-integrations-facebook-worker%22%0Atimestamp%3E%3D%222026-11-15T05%3A15%3A00Z%22%0Atimestamp%3C%3D%222026-11-15T05%3A22%3A00Z%22%0AjsonPayload.message%3D%22Facebook%3A%20Contact%20Obj%20created%20for%20lead%22?project=highlevel-backend"
    },
    {
      "source": "Facebook worker source",
      "result": "The 90-day lead-sync path reads FacebookPages and FacebookForms, then processes leads sequentially through contact lookup/create and downstream publishing. This is a plausible source of MongoDB operations, but the logs do not tie the observed leads to that path or to the named cron."
    },
    {
      "source": "Grafana metrics supplied in the investigation state",
      "result": "The worker was the largest observed application contributor at roughly 3,010–3,459 MongoDB requests/s versus a 29.7/s pre-event median. Cluster request rate was about 5,491–5,831/s; one host reached 95.5% CPU and operation p95 reached about 1.2s.",
      "url": "https://prod.grafana.leadconnectorhq.com/d/a019c5e8-473e-4431-806c-5704cb544110/mongo-observability?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=%24__all&var-team=platform&var-sub_team=%24__all&var-app=crm-integrations-facebook-worker&var-mongo_host_name=crm-leadconnector-stand-shard-00-00-pri.hfkhk.mongodb.net&var-mongo_collection=%24__all&var-mongo_method_name=%24__all&from=1794716580000&to=1794720180000&viewPanel=20"
    }
  ],
  "disconfirming_or_missing": [
    "The two lead-processing log entries do not reveal their triggering event type, and no cron execution time was verified.",
    "No per-event worker volume or MongoDB operation breakdown was available to connect the cron to the request surge.",
    "Atlas scale-up history and post-scale CPU, request-rate, and latency trends remain unverified."
  ],
  "log_queries": [
    {
      "query": "resource.type=\"k8s_container\" AND resource.labels.container_name=\"crm-integrations-facebook-worker\" AND timestamp>=\"2026-11-15T05:15:00Z\" AND timestamp<=\"2026-11-15T05:22:00Z\" AND jsonPayload.message=\"Facebook: Contact Obj created for lead\"",
      "result": "Returned two lead-processing entries at 05:19:13 and 05:19:15 UTC from one worker pod.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-integrations-facebook-worker%22%0Atimestamp%3E%3D%222026-11-15T05%3A15%3A00Z%22%0Atimestamp%3C%3D%222026-11-15T05%3A22%3A00Z%22%0AjsonPayload.message%3D%22Facebook%3A%20Contact%20Obj%20created%20for%20lead%22?project=highlevel-backend"
    }
  ],
  "conclusion": {
    "root_cause": "Undetermined. Facebook lead processing is confirmed during the surge, and the worker is a major measured MongoDB contributor; the named cron as trigger is unconfirmed.",
    "confidence": "Medium that the worker materially contributed to load; low that the named cron caused the surge.",
    "evidence_needed": [
      "Cron run history and timestamps around 04:23–05:23 UTC.",
      "Worker message counts and MongoDB operations grouped by event type around those runs.",
      "Atlas scale-up event history and post-scale CPU, request-rate, and latency metrics."
    ]
  }
}

## Evidence

### Dashboards (incident window)
- [Mongo host CPU — crm-leadconnector-standard-production (alert window)](https://prod.grafana.leadconnectorhq.com/d/_s7Pjkb7z/dedicated-atlas-clusters-hardware-metrics?orgId=1&var-Datasource=ber8nnhvgsjy8f&var-job=highlevel-prod-mongo-metrics&var-group_id=60803ad4b9f71f2ec1288526&var-cl_name=crm-leadconnector-standard-production&var-rs_nm=%24__all&var-host=%24__all&var-process_port=%24__all&var-interval=1m&from=1794716580000&to=1794720180000&viewPanel=44)
- [Mongo operation volume and latency — crm-leadconnector host (alert window)](https://prod.grafana.leadconnectorhq.com/d/a019c5e8-473e-4431-806c-5704cb544110/mongo-observability?orgId=1&var-datasource=ber8nnhvgsjy8f&var-cluster=%24__all&var-team=platform&var-sub_team=%24__all&var-app=crm-integrations-facebook-worker&var-mongo_host_name=crm-leadconnector-stand-shard-00-00-pri.hfkhk.mongodb.net&var-mongo_collection=%24__all&var-mongo_method_name=%24__all&from=1794716580000&to=1794720180000&viewPanel=20)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-leadconnector-standard-production%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3A(%22*%22)
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22crm-integrations-facebook-worker%22%0Atimestamp%3E%3D%222026-11-15T05%3A15%3A00Z%22%0Atimestamp%3C%3D%222026-11-15T05%3A22%3A00Z%22%0AjsonPayload.message%3D%22Facebook%3A%20Contact%20Obj%20created%20for%20lead%22?project=highlevel-backend)

### Related incidents
- [MongoDB R50→R60 autoscale; team investigates cron activity](https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1794718405535809)
- [Nearby CPU alert; reported cron activity as the cause](https://gohighlevel.slack.com/archives/C097UPY34QJ/p1794719085517459?thread_ts=1794719085.517459)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 807.1s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.6s | ✓ |
| grafana-worker | worker | 195.8s | ✓ |
| kubectl-worker | worker | 115.2s | ✓ |
| log-explorer-worker | worker | 72.3s | ✓ |
| slack-worker | worker | 52.2s | ✓ |
| triage | worker | 13.4s | ✓ |
| triage_llm_ranking | decision | 19.7s | ✓ |
| deep_dive_pass_1 | worker | 73.3s | ✓ |
| follow_up_check | decision | 9.7s | ✓ |
| deep_dive_pass_2 | worker | 43.8s | ✓ |
| follow_up_check | decision | 9.9s | ✓ |
| deep_dive_pass_3 | worker | 170.7s | ✓ |
| cross_validation | worker | 19.4s | ✓ |

## Alert Context

```
||CLUSTER - crm-leadconnector-standard-production, METRIC - Cluster is Autoscaling => Upscale 📊 *MongoDB Auto-Scale Alert* 🏷️ *Cluster:* crm-leadconnector-standard-production 📈 *Tier Change:* Base Tier: R50 -> New Tier: R60 💰 *New Rate:* $2.59/hr ⏰ *Time:* 11/15/2026, 4:53:22 AM 📝 *Reason:* crm-leadconnector-standard-production has exceeded 90% CPU usage for the past 10 minutes as of 11:53 PM EST on 2026/11/14. CLUSTER - crm-leadconnector-standard-production, METRIC - Cluster is Autoscaling =&gt; Upscale :bar_chart: *MongoDB Auto-Scale Alert* :label: *Cluster:* crm-leadconnector-standard-production :chart_with_upwards_trend: *Tier Change:* Base Tier: R50 -&gt; New Tier: R60 :moneybag: *New Rate:* $2.59/hr :alarm_clock: *Time:* 11/15/2026, 4:53:22 AM :memo: *Reason:* crm-leadconnector-standard-production has exceeded 90% CPU usage for the past 10 minutes as of 11:53 PM EST on 2026/11/14.
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-15T05:36:54.354Z*