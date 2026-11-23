# Alert Investigation: Unknown

**Investigation ID**: inv-2026-11-23-669fcd5f
**Workload**: highlevel-backend
**Container**: highlevel-backend
**Cluster**: N/A
**Channel**: slack
**Detected**: 2026-11-23T15:54:54.114Z
**Status**: delivering

## Root Cause

**Summary**: A widespread error spike and elevated Firestore RPC failures are supported independently. Firestore-related waiting is a plausible contributor, but provider degradation, client retry amplification and application contention remain unresolved. The later rollout cannot explain onset. Provider diagnostics and matched RPC traces at 19:09, 19:48 and 20:26 IST are needed to distinguish these causes.
**Confidence**: low
**Category**: undetermined

## Triage

**Primary Anomaly**: error_spike
**Details**: At 19:09 IST, 242 containers had >=100 ERRORs and >5x their preceding median, indicating a widespread event. Firestore DEADLINE_EXCEEDED and ABORTED RPCs were elevated during the broader incident window, but exact temporal and request-level causation remains unverified. Historical candidates inv-2026-11-23-dc656f50 and inv-2026-11-23-de589c50 support Firestore waiting as a related hypothesis, not a confirmed RCA match. highlevel-backend identifies the project, not a workload; pod health remains unassessed.
**Anomalous**: Yes
**Deep Dive Direction**: YES — the Firestore direction remains appropriate, but provider degradation versus client retry amplification, scheduling delays, or application contention is unresolved.

Need DoiT/GCP diagnostics for 19:09, 19:48 and 20:26 IST; incident traces showing RPC attempts, retries/backoff, callers and document keys; and deployed image-to-commit mappings for pre-incident and 19:41 rollout comparisons.

## Cross-Validation

**Confidence**: low

| Source | Finding | Agrees |
|--------|---------|--------|
| Grafana | Firestore DEADLINE_EXCEEDED and ABORTED RPCs exceeded observed baselines, supporting dependency distress. Thirty-minute sampling cannot establish correlation with individual incident peaks. | ✓ |
| Cloud Logging histogram | 242 containers had simultaneous ERROR spikes at 19:09 IST. This independently supports widespread impact, but sampled errors do not identify Firestore as the cause. | ✓ |
| Slack | Platform reports Firestore stalls and multiple database latency alerts. The reported 60-second read followed by a 9-ms retry lacks independently retrieved spans and incident timing. | ✓ |
| Kubernetes ReplicaSets | The 19:41 rollout followed the 19:09 onset, ruling it out as the initial trigger. Its contribution to later peaks remains untested. | ✓ |
| Pod health checks | Checks targeted the project identifier rather than a workload; resource pressure, restarts and scheduling delays remain unassessed. | ✗ |

### Discrepancies
- Multiple sources support a widespread incident, but no two independently establish the causal chain from Firestore RPC failures to application impact.
- conversations-api sampled HTTP 5xx remained within baseline; reported send-message failures were not verified in that workload's metrics.
- Cross-container evidence is project-wide, not cluster-specific; ai-employees-api sample counts are not incident-wide error totals.
- Timed-out log queries provide no evidence for or against matching errors. RESOURCE_EXHAUSTED being zero at 20:30 does not exclude earlier throttling.
- Falsification remains incomplete: incident-specific traces, caller/document-key attribution, retry amplification and pod health are missing. No provider diagnosis was recovered.

## Deep Dive Findings

{
  "status": "partial",
  "root_cause_confidence": "low",
  "summary": "Provider degradation versus client retry amplification remains unresolved. No provider diagnosis or incident-specific RPC attempt traces were recovered.",
  "findings": [
    {
      "finding": "The refreshed platform thread still describes a planned DoiT escalation.",
      "evidence": "All thread replies were fetched; no case ID or provider response appeared. The linked diagnostic artifact redirects to Google sign-in.",
      "source": "https://gohighlevel.slack.com/archives/C07JM1DA9K3/p1795435595404449",
      "limitation": "This does not establish whether a support case exists elsewhere."
    },
    {
      "finding": "A reported trace demonstrates a stalled read followed by a fast retry, but does not explain the three incident peaks.",
      "evidence": "The platform thread describes trace cc41b14f9002c5b7493e042a607d0b72: BatchGetDocuments failed with UNAVAILABLE after 60,004 ms; the retry completed in 9 ms.",
      "source": "https://gohighlevel.slack.com/archives/C07JM1DA9K3/p1795435595404449",
      "limitation": "Secondhand trace evidence; timestamp, document keys and raw spans were not independently retrieved."
    },
    {
      "finding": "Live ReplicaSets reconfirm the 19:41 rollout occurred after the 19:09 onset.",
      "cluster": "web-users-servers-usc1-prd-cluster",
      "image_repository": "gcr.io/highlevel-backend/marketplace-conversations",
      "pre_incident": {
        "created_ist": "2026-11-23 15:31:30",
        "conversations_api_revision": 246,
        "digest": "sha256:c42c9fd2b99c38be377ba3a395d7f68a458400533b493264dca5500b1e09fbe0"
      },
      "later_rollout": {
        "created_ist": "2026-11-23 19:41:41",
        "conversations_api_revision": 247,
        "digest": "sha256:2fa114d861a3e326a656cb6f52f2fe1085b4163c2e73fc9b7b4e57b5b5cfd6b9"
      },
      "limitation": "Registry tags were empty for the earlier digest and green/latest for the later digest. Neither supplied a source commit. Later amplification remains untested."
    }
  ],
  "log_queries_used": [
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-api\" severity>=ERROR timestamp>=\"2026-11-23T13:38:00Z\" timestamp<\"2026-11-23T13:40:00Z\"",
      "limit": 5,
      "result": "Timed out after 45 seconds.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%20resource.labels.container_name%3D%22conversations-api%22%20severity%3E%3DERROR;timeRange=2026-11-23T13%3A38%3A00Z%2F2026-11-23T13%3A40%3A00Z?project=highlevel-backend"
    },
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-isolated-api\" severity>=ERROR timestamp>=\"2026-11-23T14:17:30Z\" timestamp<\"2026-11-23T14:18:30Z\" \"RESOURCE_EXHAUSTED\"",
      "limit": 5,
      "result": "Timed out after 35 seconds.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%20resource.labels.container_name%3D%22conversations-isolated-api%22%20severity%3E%3DERROR%20%22RESOURCE_EXHAUSTED%22;timeRange=2026-11-23T14%3A17%3A30Z%2F2026-11-23T14%3A18%3A30Z?project=highlevel-backend"
    },
    {
      "query": "resource.type=\"k8s_container\" resource.labels.container_name=\"conversations-isolated-api\" severity>=ERROR timestamp>=\"2026-11-23T14:55:30Z\" timestamp<\"2026-11-23T14:56:30Z\" \"ABORTED\"",
      "limit": 5,
      "result": "Timed out after 35 seconds.",
      "url": "https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%20resource.labels.container_name%3D%22conversations-isolated-api%22%20severity%3E%3DERROR%20%22ABORTED%22;timeRange=2026-11-23T14%3A55%3A30Z%2F2026-11-23T14%3A56%3A30Z?project=highlevel-backend"
    }
  ],
  "limitations": [
    "Query timeouts are unavailable evidence, not zero matching errors; linked queries remain result-unverified.",
    "No incident-specific retry counts, backoff, caller/document-key attribution or scheduling-delay evidence was recovered.",
    "Parallel delegation failed because the executor could not load the child thread context."
  ],
  "required_evidence": [
    "DoiT/GCP backend diagnostics for 19:09, 19:48 and 20:26 IST separately.",
    "Matched slow and normal RPC traces containing attempts, backoff, caller, pod, database and document keys.",
    "Build provenance linking both deployed digests to commits."
  ],
  "access": "Validated existing Slack session; local gcloud and kubectl credentials. Read-only; no messages posted or production changes made."
}

## Evidence

### Dashboards (incident window)
- [HTTP traffic — conversations-api, alert cluster: elevated but within baseline range](https://prod.grafana.leadconnectorhq.com/d/d2db17da-530c-43f3-9273-c0fd664c591f?orgId=1&from=1795429800000&to=1795446600000&var-datasource=ber8nnhvgsjy8f&var-cluster=web-users-servers-usc1-prd-cluster&var-container=conversations-api&var-method=All&var-route=All&var-pod_name=All&viewPanel=1)

### Logs
- [Log Explorer query](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.container_name%3D%22ai-employees-api%22%0Aseverity%3E%3DERROR%0AjsonPayload.message%3D%22Error%20at%20redirectAIreply%22;timeRange=2026-11-23T13%3A34%3A00Z%2F2026-11-23T13%3A44%3A00Z?project=highlevel-backend)

### Related incidents
- [Platform reports default-database server-side stalls; DoiT escalation planned — 23 Nov](https://gohighlevel.slack.com/archives/C07JM1DA9K3/p1795435595404449)
- [Concurrent database latency burns and service-call anomaly — 23 Nov](https://gohighlevel.slack.com/archives/C09JSAXKKG8/p1795439766606339)
- [Conversations isAutoReply fix promoted; production sanity reported complete — 23 Nov](https://gohighlevel.slack.com/archives/C013CG4N7U0/p1795423225028339)
- [RCS status-handling deployment explicitly deferred — 23 Nov](https://gohighlevel.slack.com/archives/C013CG4N7U0/p1795439106765839)

## Investigation Timeline

**Deep dive passes**: 3
**Worker calls**: 13

**Total worker time**: 1365.5s

| Step | Type | Duration | Status |
|------|------|----------|--------|
| session_validation | decision | 11.5s | ✓ |
| grafana-worker | worker | 144.3s | ✓ |
| kubectl-worker | worker | 87.7s | ✓ |
| log-explorer-worker | worker | 366.5s | ✓ |
| slack-worker | worker | 67.8s | ✓ |
| triage | worker | 15.7s | ✓ |
| triage_llm_ranking | decision | 29.6s | ✓ |
| deep_dive_pass_1 | worker | 277.4s | ✓ |
| follow_up_check | decision | 13.4s | ✓ |
| deep_dive_pass_2 | worker | 170.6s | ✓ |
| follow_up_check | decision | 12.0s | ✓ |
| deep_dive_pass_3 | worker | 148.7s | ✓ |
| cross_validation | worker | 20.4s | ✓ |

## Alert Context

```
||<@U03D8V5L371> <!subteam^S09KD3KTP9R> <!subteam^S0AGFL102TA> We are seeing increased firestore latency which is leading to a lot of APIs being impacted with the increased latency. Can we check this as a P0 in war room. Several Instances of *<https://prod.grafana.leadconnectorhq.com/a/grafana-oncall-app/alert-groups/II2UP6X25K9JS| HTTPLatencyPerSubTeam>* alerts Initial Findings - ```PFirestore (default, highlevel-backend) has been slow since ~16:35 IST and it is hitting several teams, not only conversations. Impact (conversations APIs, 16:00-20:40 IST vs Friday same hours): - Requests over 10s: ~880k vs ~154k. Over 30s: 234k vs 25k. Traffic is only 18% higher. - Send message (POST /conversations/messages): 185k over 10s vs 17k; 5xx 3,024 vs 448. Why we think it is Firestore: - Firestore DEADLINE_EXCEEDED: ~390k vs ~48k on Friday, mostly BatchGetDocuments. - Slow requests sit inside Firestore calls for 16-75s (Balaji's traces). - #alerts-database shows Firestore latency burning for automation, platform-iam, leadgen-onboarding and crm-leadconnector. - No Google Cloud incident is posted. Load leads: (default) reads 1.31x normal since 03:30 IST (automation-workflows-start-worker, opportunities-upsert-api, crm-integrations-notifications-events-worker); opportunities-api calls 1.82x since 13:30 IST.``` We are seeing increased firestore latency which is leading to a lot of APIs being impacted with the increased latency. Can we check this as a P0 in war room. Several Instances of HTTPLatencyPerSubTeam alerts Initial Findings - PFirestore (default, highlevel-backend) has been slow since ~16:35 IST and it is hitting several teams, not only conversations. Impact (conversations APIs, 16:00-20:40 IST vs Friday same hours): - Requests over 10s: ~880k vs ~154k. Over 30s: 234k vs 25k. Traffic is only 18% higher. - Send message (POST /conversations/messages): 185k over 10s vs 17k; 5xx 3,024 vs 448. Why we think it is Firestore: - Firestore DEADLINE_EXCEEDED: ~390k vs ~48k on Friday, mostly BatchGetDocuments. - Slow requests sit inside Firestore calls for 16-75s (Balaji's traces). - #alerts-database shows Firestore latency burning for automation, platform-iam, leadgen-onboarding and crm-leadconnector. - No Google Cloud incident is posted. Load leads: (default) reads 1.31x normal since 03:30 IST (automation-workflows-start-worker, opportunities-upsert-api, crm-integrations-notifications-events-worker); opportunities-api calls 1.82x since 13:30 IST.
```

---
*Generated by Alert Investigation Orchestrator [Codex] at 2026-11-23T17:38:20.426Z*