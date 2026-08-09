# 🧪 Lab 08 — Observability: find a real trace

> ⏱️ **30 min** · 📋 **Requires:** [Lab 03](03-deploy.md) · 💰 **~$0.05 + App Insights ingestion** · ☁️ **Creates 3 Azure resources**

> ✅ **Verified end-to-end on live Azure — 2026-08-09.** The telemetry rows, the KQL results and
> the deploy failure on this page are all real captured output
> (`azd 1.30.0`, `azure.ai.agents 1.0.0-beta.9`, `eastus2`).

## What you'll learn

`azd ai agent invoke` prints a **Trace ID** on every call. Until now that was a number you
ignored. This lab turns it into something you can query.

By the end you will be able to:

- Attach Application Insights to a Foundry project and prove telemetry arrives.
- Query real agent requests with KQL, and read the three columns that matter.
- **Avoid the deploy-breaking mistake** that the obvious approach leads to.
- Say precisely what the platform traces for you, and what it does not.

> [!IMPORTANT]
> **The most valuable thing on this page is the thing that failed.** The intuitive way to wire up
> App Insights — setting the connection string as an agent environment variable — **breaks your
> deploy with HTTP 400.** We tried it, it failed, and step 2 is the correct route.

## Steps

### 1. Deploy any agent and capture a Trace ID

Use the agent from [Lab 03](03-deploy.md), or any agent you already have running.

```bash
azd ai agent invoke <agent-name> "What is 2+2?"
```

✅ Expected — note the `Trace ID` line:

```text
Trace ID:     7013379fc62ca98207441df3a356da8b
[agent-framework-agent-basic-responses] 4
Server responded in 7.524s (first byte: 1.205s)
```

Right now that ID goes nowhere. There is no App Insights resource to hold it.

### 2. Create App Insights and connect it to the project

> [!CAUTION]
> ❌ **Do NOT do this** — it is the natural first attempt and it fails:
> ```yaml
> environmentVariables:
>   - name: APPLICATIONINSIGHTS_CONNECTION_STRING   # ← deploy fails, HTTP 400
>     value: ${APPLICATIONINSIGHTS_CONNECTION_STRING}
> ```
> ✅ Verified failure text:
> ```text
> RESPONSE 400: 400 Bad Request
> ERROR CODE: invalid_payload
> "Environment variable 'APPLICATIONINSIGHTS_CONNECTION_STRING' is reserved for platform
>  use. All FOUNDRY_* and AGENT_* variables are reserved per container-image-spec.
>  Please remove and re-try."
> ```
> The variable is **reserved**. Telemetry is wired at the *project* level, not the agent level.
> See [environment variables](../reference/environment-variables.md).

The correct route — create the resource, then attach it as a **project connection**:

```bash
az monitor app-insights component create \
  --app appi-mylab --location eastus2 \
  --resource-group <your-rg> --application-type web \
  --query "{name:name, connectionString:connectionString}"
```

✅ Expected:

```json
{
  "connectionString": "InstrumentationKey=959e06c0-…;IngestionEndpoint=https://eastus2-3.in.applicationinsights.azure.com/;LiveEndpoint=…;ApplicationId=…",
  "name": "appi-p2verify"
}
```

Then attach it to the Foundry account as an `AppInsights` connection:

```bash
az cognitiveservices account connection create \
  --name <cog-account-name> --resource-group <your-rg> \
  --connection-name appinsights-conn \
  --category AppInsights --auth-type ApiKey \
  --target "<the App Insights resource ID>"
```

✅ Expected — the connection object, with the resource wired into `metadata.ResourceId`:

```jsonc
{
  "group": "ServicesAndApps",
  "isDefault": true,
  "isSharedToAll": true,
  "metadata": {
    "ApiType": "Azure",
    "ResourceId": "/subscriptions/…/providers/microsoft.insights/components/appi-p2verify"
  },
  "type": "Microsoft.CognitiveServices/accounts/connections"
}
```

`"isDefault": true` is what makes the project actually emit to it.

### 3. Generate traffic

One invoke is not enough to be sure — a single row could be a coincidence. Send three:

```bash
for i in 1 2 3; do azd ai agent invoke <agent-name> "What is 2+2?"; done
```

✅ Expected — three responses, three distinct Trace IDs:

```text
Trace ID:     7013379fc62ca98207441df3a356da8b
[agent-framework-agent-basic-responses] 4
Server responded in 7.524s (first byte: 1.205s)
Trace ID:     42bebbc72cd745527d44196db9de022b
[agent-framework-agent-basic-responses] 4
Server responded in 6.959s (first byte: 1.282s)
Trace ID:     5fedeffb5e4e95bb1a630cb28156127d
[agent-framework-agent-basic-responses] 4
Server responded in 7.620s (first byte: 1.128s)
```

> [!NOTE]
> **Telemetry is not instant.** Ingestion lag is typically 1–3 minutes. An empty query result
> immediately after invoking means "wait", not "broken" — the mistake here is reconfiguring
> something that was already working.

### 4. Query it — count first

Always confirm *something* arrived before writing a real query.

```bash
az monitor app-insights query --app <app-insights-name> --resource-group <rg> \
  --analytics-query "union * | where timestamp > ago(30m) | summarize count() by itemType" \
  -o table
```

✅ Expected — exactly the three calls you made:

```text
  request          3
```

### 5. Read the actual requests

```bash
az monitor app-insights query --app <app-insights-name> --resource-group <rg> \
  --analytics-query "requests | where timestamp > ago(30m) | project timestamp, name, resultCode, duration, cloud_RoleName" \
  -o table
```

✅ Expected — the verified output:

```text
timestamp                     | name          | resultCode | duration   | cloud_RoleName
2026-08-09T00:27:15.4946734Z  | invoke_agent  | 0          | 5824.1601  | agentsv2
2026-08-09T00:27:29.198329Z   | invoke_agent  | 0          | 6628.5839  | agentsv2
2026-08-09T00:26:57.9832888Z  | invoke_agent  | 0          | 6536.2622  | agentsv2
```

Four things to read off this table — they are not obvious:

| Observation | What it means |
|---|---|
| `name` is **`invoke_agent`** | The platform traces the *invocation*, not your Python functions. Your own spans need instrumentation you add. |
| `resultCode` is **`0`**, not `200` | This is not an HTTP status. Do not write `where resultCode == 200` — you will get zero rows forever. |
| `cloud_RoleName` is **`agentsv2`** | The emitter is the Foundry **platform**, not your container. That is why the env var is reserved. |
| `duration` ≈ **5824–6629 ms** vs CLI-reported 6.9–7.6 s | The ~1 s gap is client-side round trip. Server duration is the honest latency number. |

## ✅ Checkpoint

| # | Check | Verified expected result |
|---|---|---|
| 1 | `azd ai agent invoke` | prints a `Trace ID:` line |
| 2 | connection `isDefault` | `true` |
| 3 | `summarize count() by itemType` | `request` count equals your invoke count |
| 4 | `requests \| project name` | `invoke_agent` |
| 5 | `cloud_RoleName` | `agentsv2` |

If check 3 returns nothing, wait three minutes and re-run **before** changing any configuration.

### What is *not* traced

Being precise about the boundary is the practical value here:

| Traced automatically ✅ | Not traced unless you add it ❌ |
|---|---|
| The `invoke_agent` request | Individual tool/function calls inside your agent |
| End-to-end server duration | Per-step timing within a turn |
| Result code and role name | Prompt and completion content |
| Trace ID correlation | Token counts per call |
| | Your own log lines |

For anything in the right column you instrument your own code with OpenTelemetry. The platform
gives you the outer span; the inside is yours.

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| `azd deploy` fails HTTP 400 `invalid_payload` | You set `APPLICATIONINSIGHTS_CONNECTION_STRING` as an agent env var. | Remove it. Use a **project connection** (step 2). |
| Query returns nothing, minutes after invoking | Ingestion lag. | Wait 1–3 min, re-run the count query first. |
| Query returns nothing, ever | Connection not `isDefault`, or wrong App Insights resource. | `az cognitiveservices account connection show --name <acct> --connection-name <conn>` |
| `where resultCode == 200` → 0 rows | `resultCode` is `0` for success here. | Use `resultCode == 0`, or drop the filter. |
| `az monitor app-insights` not found | The CLI extension is missing. | `az extension add --name application-insights` |
| `No module named 'rpds.rpds'` warning | Cosmetic CLI extension noise. | ✅ Verified harmless — the command still returns correct JSON. |

## ✏️ Exercise

Your agent calls a slow external API inside a tool. Users report ~8-second responses. You open
App Insights and see one `invoke_agent` row at `duration: 7800`.

1. Does this tell you the external API is the problem?
2. What would you have to change to find out?
3. Why can you not answer this with `cloud_RoleName` filtering?

<details>
<summary>Solution</summary>

1. **No.** ✅ Verified: the platform emits a *single* `invoke_agent` span covering the whole turn.
   7800 ms could be the model, your tool, the external API, or container cold start — the row
   cannot distinguish them.

2. **Instrument your own code.** Add OpenTelemetry spans around the external call inside your
   agent, so the tool call becomes a child span with its own duration. The platform's outer span
   stays; you are adding the inside.

3. `cloud_RoleName` is **`agentsv2`** — the platform's own role, identical for every agent. It
   identifies the *emitter*, not your application, so it cannot separate one code path from
   another. Filtering on it narrows nothing.

   The general lesson: **platform telemetry tells you *that* a turn was slow, never *why*.**
</details>

## → Next

- 👉 [Lab 09 — Multi-agent A2A](09-multi-agent-a2a.md) — the one lab that does not finish green.
- 👉 [Observability reference](../reference/observability.md) — connection shapes and KQL.
- 👉 [Environment variables](../reference/environment-variables.md) — the full reserved list.
- 👉 [Troubleshooting](../reference/troubleshooting.md) — every verified failure mode.

---

<sub>[← Container mode](07-container-mode.md) · [🧪 Tutorial index](README.md) · [Multi-agent A2A →](09-multi-agent-a2a.md)</sub>
