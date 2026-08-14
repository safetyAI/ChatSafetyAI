# Chatbot Azure Container Apps diagnostics

This guide sets up the diagnostics needed to investigate:

- random R Shiny greyouts / WebSocket disconnects;
- container restarts and replica replacements;
- health-probe failures;
- KEDA scale-in / scale-out behavior;
- unexpected HTTP / Envoy errors;
- CPU / memory pressure.

---

## 1) Configure Azure Monitor diagnostics

From the main deployment guide [azure_MRE_commands_client.md](https://github.com/safetyAI/ChatSafetyAI/blob/main/azure_MRE_commands_client.md), we already create:
- the Container Apps environment
- the Log Analytics workspace
- the chatbot Container App

```bash
# Replace the values below with your deployment-specific names
RESOURCE_GROUP=csai-mre
ENV_NAME=csai-acaenv-mre
WORKSPACE_NAME=csai-mre-logs
CHATBOT_APP=csai-mre-chatbot

ENV_DIAGNOSTIC_SETTING_NAME="${ENV_NAME}-diagnostics"
APP_DIAGNOSTIC_SETTING_NAME="${CHATBOT_APP}-metrics"
```

### a) Resolve the existing Azure resource IDs

```bash
WORKSPACE_RESOURCE_ID=$(az monitor log-analytics workspace show \
  --resource-group "$RESOURCE_GROUP" \
  --workspace-name "$WORKSPACE_NAME" \
  --query id \
  --output tsv)

WORKSPACE_GUID=$(az monitor log-analytics workspace show \
  --resource-group "$RESOURCE_GROUP" \
  --workspace-name "$WORKSPACE_NAME" \
  --query customerId \
  --output tsv)

ENV_ID=$(az containerapp env show \
  --name "$ENV_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query id \
  --output tsv)

CHATBOT_APP_ID=$(az containerapp show \
  --name "$CHATBOT_APP" \
  --resource-group "$RESOURCE_GROUP" \
  --query id \
  --output tsv)
```

### b) Inspect the current logging / diagnostic configuration before making any change, to avoid redundant diagnostic settings.

```bash
# the commands below are read-only
echo "=== Current Container Apps environment logging destination ==="

az containerapp env show \
  --name "$ENV_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query properties.appLogsConfiguration.destination \
  --output tsv

echo "=== Existing environment diagnostic settings ==="

az monitor diagnostic-settings list \
  --resource "$ENV_ID" \
  --query "[].{Name:name,Workspace:workspaceId}" \
  --output table

echo "=== Existing chatbot diagnostic settings ==="

az monitor diagnostic-settings list \
  --resource "$CHATBOT_APP_ID" \
  --query "[].{Name:name,Workspace:workspaceId}" \
  --output table
```

### c) Switch the Container Apps environment logging destination to Azure Monitor

Register the Azure Monitor provider:

```bash
az provider register --namespace Microsoft.Insights --wait
```

`ContainerAppHTTPLogs` are exposed through Azure Monitor diagnostic settings at the **Container Apps environment** level.

```bash
az containerapp env update \
  --name "$ENV_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --logs-destination azure-monitor
```

> **Note:** Azure CLI may print a message saying that "Azure Monitor must be
> set up manually." This is expected. Setting `--logs-destination azure-monitor`
> only enables Azure Monitor as the environment logging mode; the Diagnostic
> Settings created in the following steps define the actual Log Analytics
> destination.


Portal equivalent:

**Container Apps Environment → Monitoring → Logging options → Logs destination = Azure Monitor**

After changing the destination in the portal, refresh the environment page if **Diagnostic settings** does not immediately appear.

### d) Enable environment-level logs

This diagnostic setting sends the Container Apps environment logs to the existing Log Analytics workspace.

```bash
az monitor diagnostic-settings create \
  --name "$ENV_DIAGNOSTIC_SETTING_NAME" \
  --resource "$ENV_ID" \
  --workspace "$WORKSPACE_RESOURCE_ID" \
  --export-to-resource-specific true \
  --logs '[{"categoryGroup":"allLogs","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

The important resource-specific tables are:

- `ContainerAppHTTPLogs` — Envoy ingress HTTP / WebSocket activity;
- `ContainerAppSystemLogs` — KEDA, Envoy, health probes, revisions, replicas, container lifecycle, etc.;
- `ContainerAppConsoleLogs` — application `stdout` / `stderr`.

Verify the environment diagnostic setting:

```bash
az monitor diagnostic-settings show \
  --name "$ENV_DIAGNOSTIC_SETTING_NAME" \
  --resource "$ENV_ID" \
  --output jsonc
```

### e) Export chatbot-level metrics

Metrics such as replica count, restart count, requests, CPU, and memory belong to the individual Container App resource.

```bash
az monitor diagnostic-settings create \
  --name "$APP_DIAGNOSTIC_SETTING_NAME" \
  --resource "$CHATBOT_APP_ID" \
  --workspace "$WORKSPACE_RESOURCE_ID" \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

Verify:

```bash
az monitor diagnostic-settings show \
  --name "$APP_DIAGNOSTIC_SETTING_NAME" \
  --resource "$CHATBOT_APP_ID" \
  --output jsonc
```

You can also list all settings attached to the chatbot:

```bash
az monitor diagnostic-settings list \
  --resource "$CHATBOT_APP_ID" \
  --output table
```

Allow several minutes for newly enabled logs / metrics to appear.

> If `AzureMetrics` is initially reported as an unknown table, first verify that the **chatbot-level** `AllMetrics` diagnostic setting exists, then wait several minutes.
> The same metrics can always be inspected immediately from **Container App → Monitoring → Metrics** or through `az monitor metrics list`.

Example direct CLI metrics check:

```bash
az monitor metrics list \
  --resource "$CHATBOT_APP_ID" \
  --metrics Replicas RestartCount Requests UsageNanoCores WorkingSetBytes \
  --interval 1m \
  --offset 1h \
  --output jsonc
```

---

## 2) Save the diagnostic queries in Log Analytics

Open:

**Azure Portal → Log Analytics workspace → Logs → KQL mode**

The queries below deliberately contain **no `take` / `limit` clause**. They return all matching records inside the time range selected in the Log Analytics UI, subject only to Azure's normal result-display limits.

For routine inspection, use **Last 1 hour** or **Last 24 hours**.

For an incident, record the exact time in **UTC** and use a **Custom** range covering approximately 5–10 minutes before and after the greyout.

Do not merge these four signals into one large query. HTTP, platform, application, and metrics data answer different questions and are easier to interpret separately.

Replace "csai-mre-chatbot" with your deployment-specific name in what follows (we cannot use variables as KQL queries are meant to the copied and pasted in the Azure Portal interface directly). 

---

### Query 1: HTTP / WebSocket / Envoy diagnostics

**Saved-query name:** `CSAI - HTTP and WebSocket diagnostics`

**Use when:** the UI greys out, keep-alive requests start failing, a WebSocket disconnects, or an ingress timeout / reset is suspected.

```kusto
ContainerAppHTTPLogs
| where ContainerAppName == "csai-mre-chatbot"
| project
    TimeGenerated,
    StartTime,
    Method,
    Path,
    StatusCode,
    RequestDuration,
    ResponseCodeDetails,
    ResponseFlags,
    ConnectionId,
    RevisionName,
    ReplicaName,
    UpstreamHost,
    UpstreamRequestAttemptCount
| order by TimeGenerated desc
```

Look for:

- the last successful keep-alive `200`;
- sudden `500`, `502`, `503`, or `504` responses;
- `stream_idle_timeout`;
- connection / upstream reset flags;
- WebSocket-related disconnect records;
- unusually long request duration;
- a change in `RevisionName` or `ReplicaName`.

Do **not** filter the successful keep-alive requests out during an incident investigation. The exact point where they stop returning `200` is useful evidence.

---

### Query 2: Azure platform / KEDA / replica / health-probe diagnostics

**Saved-query name:** `CSAI - Platform and replica diagnostics`

**Use when:** investigating replica scale events, container restarts, health-probe failures, revision changes, or other Azure-side lifecycle behavior.

```kusto
ContainerAppSystemLogs
| where ContainerAppName == "csai-mre-chatbot"
| project
    TimeGenerated,
    Type,
    EventSource,
    OperationName,
    Reason,
    RevisionName,
    ReplicaName,
    Count,
    Log
| order by TimeGenerated desc
```

Look for:

- `ProbeFailed`;
- startup, readiness, and liveness failures;
- `ContainerCreated` / `ContainerStarted`;
- termination / stop events;
- KEDA scaler events;
- replica creation or removal;
- revision activation / deactivation;
- image pulls occurring unexpectedly during an established session;
- the same replica starting its container again.

A particularly useful restart signature is:

- same `RevisionName`;
- same `ReplicaName`;
- another `ContainerCreated` / `ContainerStarted` event;
- an incremented `Count`.

That indicates the **container restarted inside the existing replica**, even though the replica name did not change.

> Do not over-filter this query when the root cause is unknown. A narrow probe-only query can hide the lifecycle event that actually explains the incident.

---

### Query 3: R Shiny application console diagnostics

**Saved-query name:** `CSAI - R Shiny console diagnostics`

**Use when:** correlating Azure-side events with what the R process itself was doing.

```kusto
ContainerAppConsoleLogs
| where ContainerAppName == "csai-mre-chatbot"
| project
    TimeGenerated,
    Stream,
    RevisionName,
    ContainerGroupName,
    ContainerName,
    Log
| order by TimeGenerated desc
```

Look for:

- R / Shiny errors or exceptions;
- long synchronous / blocking operations;
- application logs abruptly stopping and then starting again;
- application processing continuing after the browser disconnected;
- whether logs before and after the incident came from the same replica.

`ContainerGroupName` identifies the pod / replica that emitted the application log and can be correlated with `ReplicaName` in the system / HTTP logs.

---

### Query 4: scaling / restart / CPU / memory metrics

**Saved-query name:** `CSAI - Scaling and resource metrics`

**Use when:** investigating unexpected scaling, restart counts, CPU pressure, memory pressure, or request-volume anomalies.

```kusto
AzureMetrics
| where tolower(_ResourceId) endswith "/containerapps/csai-mre-chatbot"
| where MetricName in (
    "Replicas",
    "RestartCount",
    "Requests",
    "UsageNanoCores",
    "WorkingSetBytes",
    "CpuPercentage",
    "MemoryPercentage"
)
| project
    TimeGenerated,
    MetricName,
    Average,
    Minimum,
    Maximum,
    Total,
    UnitName
| order by TimeGenerated desc
```

The most useful metrics are:

- `Replicas` — active replica count;
- `RestartCount` — cumulative restart count per replica;
- `Requests` — processed HTTP requests;
- `UsageNanoCores` / `CpuPercentage` — CPU usage;
- `WorkingSetBytes` / `MemoryPercentage` — memory usage.

Container Apps metrics have one-minute granularity, so use them to identify trends / spikes around an incident rather than expecting second-level precision.

If the `AzureMetrics` table is not yet available, use:

**Container App → Monitoring → Metrics**

or:

```bash
az monitor metrics list \
  --resource "$CHATBOT_APP_ID" \
  --metrics Replicas RestartCount Requests UsageNanoCores WorkingSetBytes \
  --interval 1m \
  --offset 1h \
  --output jsonc
```

---

## 3) Save the queries so they can be reused

For each of the four queries:

1. Run the query in **Log Analytics → Logs → KQL mode**.
2. Select **Save → Save as query**.
3. Keep **Save to the default query pack** unless there's already a dedicated query pack.
4. Use category: `Containers`.
5. Create/selecct label `CSAI Diagnostics`
6. Save the queries with these names:
   - `CSAI - HTTP and WebSocket diagnostics`
   - `CSAI - Platform and replica diagnostics`
   - `CSAI - R Shiny console diagnostics`
   - `CSAI - Scaling and resource metrics`

Saved queries can then be reopened from the **Queries** interface.
Make sure that default query pack is loaded (can be enabled in the Queries hub window)

Users need **Log Analytics Contributor** to save / edit queries and **Log Analytics Reader** to view and run saved queries.

---

## 4) Use the built-in Azure Container Apps diagnostic detectors

The built-in detector is important because it can correlate health-probe failures with actual container restarts even when the ordinary system-log table does not expose the full causal chain.

Open:

**Container App → Diagnose and solve problems → Availability and Performance → Health Probe Failures**

Review:

1. **Probe failures restarted container(s)**  
   If Azure explicitly reports that probe failures caused container restarts and impacted availability, this is causal platform evidence, not merely background probe noise.

2. **Health probe configuration per revision**  
   Select the revision involved in the incident and record:
   - probe type: Startup / Readiness / Liveness;
   - protocol;
   - port;
   - timeout;
   - period;
   - initial delay;
   - failure threshold.

3. **Health Probe Failures by revision / message**  
   Distinguish startup, readiness, and liveness failures.

4. **Container restarts and lifecycle correlation**  
   Check whether probe failures correlate with container restart / termination events around the incident.

Interpretation:

- **Startup probe failures after a container has already restarted** are usually evidence of the replacement process starting up; they do not by themselves prove what triggered the preceding restart.
- **Readiness failures** can remove a replica from traffic without necessarily restarting it.
- **Liveness failures** are especially important for an established session because repeated liveness failures can cause the platform to restart the container.
- If Azure's detector explicitly says that probe failures caused container restarts, treat those restarts as real availability-impacting platform events.

Microsoft recommends using this detector specifically for diagnosing probe failures and reviewing their per-revision configuration.

---

## 5) Greyout investigation workflow

When a user reports or reproduces a greyout:

1. Record the exact timestamp in **UTC**.
2. Record what the browser console showed:
   - last successful keep-alive;
   - first failing keep-alive;
   - HTTP status;
   - WebSocket close / error if visible.
3. Set the Log Analytics time range to approximately **±5–10 minutes** around the incident.
4. Run **HTTP / WebSocket diagnostics**.
5. Run **Platform and replica diagnostics**.
6. Run **R Shiny console diagnostics**.
7. Run **Scaling and resource metrics** if `AzureMetrics` is available; otherwise inspect the Container App **Metrics** blade.
8. Open **Diagnose and solve problems → Availability and Performance → Health Probe Failures** and inspect the same time window / revision.
9. Correlate:
   - timestamp;
   - revision;
   - replica;
   - container restart;
   - probe failures;
   - browser keep-alive status;
   - CPU / memory;
   - application logs.

Typical interpretations:

- `200` keep-alives suddenly become immediate `500`s and the same replica shows a second `ContainerCreated` / `ContainerStarted` sequence → strong evidence of a container restart.
- `ReplicaName` changes → replica replacement / scale event.
- `RestartCount` increases → container restart.
- Azure Health Probe Failures detector says probes caused restart(s) → probe failures are directly impacting availability.
- WebSocket / ingress reset occurs while the same container continues logging normally → connection / ingress / network issue.
- `504` + `stream_idle_timeout` after ~240 seconds while application logs continue → long-running request / blocked application process; ingress closed the connection but the container may still be running.
- CPU / memory reaches the configured limit immediately before a restart → investigate resource pressure / OOM / throttling.
- replica count drops around the incident → investigate KEDA scale-in.

### Important R Shiny interpretation

A grey Shiny UI means the browser lost its active Shiny connection. It does **not** prove that the failure originated inside application code.

External events such as:

- a container restart;
- a replica termination / replacement;
- an ingress / WebSocket reset;
- a network interruption;

can all break the Shiny WebSocket and therefore produce the same disconnected / greyed-out UI.

---

## 6) Diagnose excessive replica scale-out

When a single CSAI session unexpectedly causes many replicas to start.
The goal is to determine whether the browser-side keep-alive requests, combined with the HTTP scaler setting `concurrentRequests=1`, are generating enough HTTP activity to trigger unnecessary KEDA scale-out.

Start from the minimum replica count and open **exactly one chatbot session**.

Run two tests:

- **Test A:** keep-alive JavaScript enabled.
- **Test B:** keep-alive JavaScript temporarily disabled.

For each test, leave the session open for approximately 2–3 minutes and use the same Log Analytics time window.

### Measure HTTP traffic in 15-second windows

Run in **Log Analytics workspace -> Logs -> KQL mode**

```kusto
ContainerAppHTTPLogs
| where ContainerAppName == "csai-mre-chatbot"
| summarize
    TotalRequests = count(),
    KeepaliveGETs = countif(Method == "GET" and Path == "/"),
    AvgDurationMs = avg(RequestDuration),
    MaxDurationMs = max(RequestDuration),
    ReplicasServingTraffic = dcount(ReplicaName)
    by bin(TimeGenerated, 15s)
| extend RequestsPerSecond = round(TotalRequests / 15.0, 2)
| order by TimeGenerated asc
```

This shows how much HTTP traffic one session generates and how much of it comes from the `GET /` keep-alive.

With a 1-second keep-alive interval, approximately 15 `GET /` requests per 15-second window are expected while the application is responsive.

### Identify which HTTP endpoints generate the traffic

```kusto
ContainerAppHTTPLogs
| where ContainerAppName == "csai-mre-chatbot"
| summarize
    Requests = count(),
    AvgDurationMs = avg(RequestDuration),
    MaxDurationMs = max(RequestDuration)
    by bin(TimeGenerated, 15s), Method, Path
| order by TimeGenerated asc, Requests desc
```

Use this to determine whether the traffic is primarily the keep-alive `GET /` requests or whether Shiny, authentication, static resources, or another endpoint is generating additional HTTP requests.

### Inspect KEDA scaling activity

```kusto
ContainerAppSystemLogs
| where ContainerAppName == "csai-mre-chatbot"
| where EventSource == "KEDA"
| project
    TimeGenerated,
    Type,
    Reason,
    RevisionName,
    Log
| order by TimeGenerated asc
```

Use this to correlate HTTP activity with KEDA scale-out / scale-in activity.

### Inspect replica count

If `AzureMetrics` is available:

```kusto
AzureMetrics
| where tolower(_ResourceId) endswith "/containerapps/csai-mre-chatbot"
| where MetricName in ("Replicas", "Requests")
| project
    TimeGenerated,
    MetricName,
    Average,
    Minimum,
    Maximum,
    Total
| order by TimeGenerated asc
```

Alternatively, use:

**Container App → Monitoring → Metrics**

and display:

- `Replicas`
- `Requests`

at one-minute granularity.

### Interpretation

Compare Test A and Test B.

If:

- one session with keep-alive enabled causes heavy scale-out;
- the HTTP logs show that most traffic consists of `GET /` keep-alives;
- KEDA scale-out occurs at the same time;
- and disabling the keep-alive prevents or greatly reduces the scale-out;

then there is strong evidence that the keep-alive traffic combined with `concurrentRequests=1` is driving excessive scaling.

If both tests produce similar scale-out, inspect the endpoint breakdown to identify the other HTTP traffic responsible.

> The HTTP-log queries show request volume, but do not perfectly reconstruct KEDA's internal concurrency metric. The controlled A/B test is therefore the most useful evidence for determining whether the keep-alive mechanism is causing the scale-out.

## 7) Azure Container Apps built-in diagnostics

In addition to Log Analytics, Azure Container Apps provides built-in diagnostic reports that can correlate platform events such as health-probe failures and container restarts.

Open: **Azure Portal -> Container App -> Diagnose and solve problems -> Availability and Performance**

****

### Health Probe Failures

Open: **Availability and Performance -> Health Probe Failures**

Use this when investigating random greyouts, container restarts, or health-probe warnings.

Select the revision involved in the incident and review:

- whether Azure reports that probe failures caused container restarts;
- Startup, Readiness, and Liveness probe failures;
- probe configuration for the revision;
- failure timestamps;
- container restart / lifecycle correlation.

Important interpretation:

- **Startup** failures often occur while a newly started/restarted container is coming online.
- **Readiness** failures can temporarily remove a replica from traffic.
- **Liveness** failures can cause the platform to restart the container.
- If Azure explicitly reports that probe failures restarted containers and impacted availability, treat this as platform evidence rather than harmless probe noise.

### Container Exit Events

Open: **Availability and Performance -> Container Exit Events**

Use this when a container appears to have restarted or disappeared.

Select the affected revision and inspect:

- container exit timestamps;
- exit codes;
- reported termination reasons;
- repeated exits/restarts.

Correlate these timestamps with:

- `ContainerAppSystemLogs`;
- `ContainerAppHTTPLogs`;
- `ContainerAppConsoleLogs`;
- browser keep-alive / WebSocket errors;
- replica/restart metrics.

> The built-in diagnostic reports are especially useful when Log Analytics shows that a container restarted but does not expose the event that originally triggered the restart.

## 8) Relevant Microsoft documentation

- Azure Container Apps log storage and monitoring options:  
  https://learn.microsoft.com/azure/container-apps/log-options
- Azure Container Apps health probes:  
  https://learn.microsoft.com/azure/container-apps/health-probes
- Troubleshoot health probe failures:  
  https://learn.microsoft.com/azure/container-apps/troubleshoot-health-probe-failures
- Azure Container Apps metrics:  
  https://learn.microsoft.com/azure/container-apps/metrics
- Save queries in Log Analytics:  
  https://learn.microsoft.com/azure/azure-monitor/logs/save-query
