# 📊 Observability

> **Captured live on 2026-08-08** against `azd 1.30.0` / `azure.ai.agents 1.0.0-beta.9`,
> from a real deploy + invoke of
> [`samples/csharp/01-hello-world`](../../samples/csharp/01-hello-world/).
> Log excerpts are real output, trimmed for width.

The headline fact, because it will save you an hour:

> [!IMPORTANT]
> **The default `azd ai agent` flow provisions no Application Insights.** After a full
> `provision` + `deploy`, `azd env get-values` contains **no** `APPLICATIONINSIGHTS_CONNECTION_STRING`,
> no OTel variables, nothing. Verified on the captured run. Your observability out of the box
> is **`azd ai agent monitor`** — a live log stream — and nothing else.
>
> Guidance that tells you to "query your agent's traces in App Insights" is describing a
> resource you have to add yourself.

---

## `azd ai agent monitor` — what you actually get

```bash
azd ai agent monitor <agent-name>
```

| Flag | Default | Purpose |
|---|---|---|
| `-f, --follow` | off | Stream in real time |
| `-t, --type` | `console` | `console` (stdout/stderr) or `system` (container events) |
| `-l, --tail` | `50` | Trailing lines, **1–300** |
| `-s, --session-id` | last | Stream one session ([sessions](./azd-cli.md#sessions--the-conversation-is-a-real-addressable-resource)) |
| `--raw` | off | Raw SSE stream, unformatted |
| `--utc` | off | UTC instead of local timestamps |
| `--user-identity` | — | Sets `x-agent-user-id` / `x-ms-user-identity` |

Real output:

```text
16:46:59  stdout   info: Azure.Identity[24]
16:46:59  stdout         MSAL 4.84.2.0 MSAL.NetCore .NET 10.0.9 Linux
                         [FindAccessTokenAsync] Discovered 0 access tokens in cache using
                         partition key: af65ac56-...-9cb1a37e52da_managed_identity_AppTokenCache
16:46:59  stdout         [LogMetricsFromAuthResult] Cache Refresh Reason: NoCachedAccessToken
16:46:59  stdout         [LogMetricsFromAuthResult] DurationTotalInMs: 309
16:46:59  stdout         TokenEndpoint: ****
16:48:53  status   Successfully connected to container
```

Read three things out of that:

- **`status` is a third stream** alongside `stdout`/`stderr`. `Successfully connected to
  container` is the CLI's own connection event, not your app's.
- **Secrets are redacted at the log layer** — `TokenEndpoint: ****`. Foundry is scrubbing this,
  not the SDK. Don't rely on it for *your* secrets; it is not a general-purpose scrubber.
- **The runtime tells you what it is** — `.NET 10.0.9 Linux`, `MSAL 4.84.2.0`. Useful when a
  bug is version-specific and `azure.yaml` only said `dotnetCsharp`.

> [!NOTE]
> **`--type system` is not obviously different.** On the captured run, `--type system` returned
> substantially the same content as `--type console`. Documented as observed; do not assume
> `system` gives you container lifecycle events that `console` hides.

> [!WARNING]
> **`--tail` maxes out at 300 lines.** There is no `--since`, no time range, and no query
> language. `monitor` is a *tail*, not a log store. Anything older than the buffer is gone
> unless you exported it. This is the strongest practical argument for adding App Insights
> before you go to production.

### `logs` is not a command

```bash
azd ai agent logs hello-world
ERROR: unknown command "logs" for "agent"
```

Verified. The command is `monitor`.

---

## Timing signal from `invoke`

`azd ai agent invoke` prints a timing line, which is the cheapest latency telemetry available:

```text
Server responded in 6.877s (first byte: 3.622s)
```

Both numbers matter. **First byte 3.622s** is time-to-first-token — what a user perceives.
**Total 6.877s** is generation length. If first byte is high, look at cold start, token
acquisition (that 309 ms MSAL fetch), and model queueing; if the *gap* is large, the model is
simply producing a lot of tokens. Optimising the wrong one wastes a day.

---

## Adding real tracing

> ✅ **Verified 2026-08-09** against a live run. This section previously documented the
> wrong method — see the warning below, which is kept because the wrong method is the
> obvious one and you will probably try it.

### ⚠️ The way that does not work

The intuitive approach — passing the connection string to the agent as an environment
variable — is **rejected by the platform**:

```yaml
# DO NOT DO THIS — azd deploy fails
environmentVariables:
  - name: APPLICATIONINSIGHTS_CONNECTION_STRING
    value: ${APPLICATIONINSIGHTS_CONNECTION_STRING}
```

```text
RESPONSE 400: 400 Bad Request
ERROR CODE: invalid_payload

{
  "error": {
    "code": "invalid_payload",
    "message": "Environment variable 'APPLICATIONINSIGHTS_CONNECTION_STRING' is reserved for
                platform use. All FOUNDRY_* and AGENT_* variables are reserved per
                container-image-spec. Please remove and re-try.",
    "param": "environment_variables[0]",
    "type": "invalid_request_error"
  }
}
```

Two things to take from that message:

1. `APPLICATIONINSIGHTS_CONNECTION_STRING` is **reserved** — the platform injects it itself.
2. **Every `FOUNDRY_*` and `AGENT_*` variable is reserved too.** Declaring any of them in
   `environmentVariables` fails the deploy. See
   [environment variables](./environment-variables.md).

### The way that works

Tracing is wired at the **project level**, as a connection on the Foundry account — not per
agent. Create the resource:

```bash
az monitor app-insights component create \
  --app appi-<name> --location <region> --resource-group rg-<env> \
  --application-type web --query connectionString -o tsv
```

Then attach it as an `AppInsights` connection on the Cognitive Services account:

```bash
AIID=$(az monitor app-insights component show --app appi-<name> -g rg-<env> --query id -o tsv)
az rest --method put \
  --url "https://management.azure.com/subscriptions/<sub>/resourceGroups/rg-<env>/providers/Microsoft.CognitiveServices/accounts/<account>/connections/appi-conn?api-version=2025-06-01" \
  --body '{"properties":{"category":"AppInsights","authType":"ApiKey","group":"Azure",
           "target":"'"$AIID"'","isSharedToAll":true,
           "credentials":{"key":"<connection string>"},
           "metadata":{"ApiType":"Azure","ResourceId":"'"$AIID"'"}}}'
```

The response comes back with `"isDefault": true` and `"group": "ServicesAndApps"`.

> [!IMPORTANT]
> **No redeploy is needed, and no code change is needed.** The sample agent contains no
> OpenTelemetry setup at all — telemetry began flowing as soon as the connection existed.
> This is why the environment-variable approach was never necessary.

### What actually arrives

Three `invoke` calls, then a query 90 seconds later:

```text
timestamp                     | name         | resultCode | duration  | cloud_RoleName
2026-08-09T00:26:57.9832888Z  | invoke_agent | 0          | 6536.2622 | agentsv2
2026-08-09T00:27:15.4946734Z  | invoke_agent | 0          | 5824.1601 | agentsv2
2026-08-09T00:27:29.198329Z   | invoke_agent | 0          | 6628.5839 | agentsv2
```

Three invokes, three `requests` rows. Note what is **not** there:

| Table | Rows after 3 invokes | Meaning |
|---|---|---|
| `requests` | **3** | one per invoke, named `invoke_agent`, role `agentsv2` |
| `dependencies` | 0 | the model call is not surfaced as a dependency |
| `traces` | 0 | no log-level telemetry from this sample |
| `customEvents` | 0 | populated by **evaluation** runs, not by invokes |
| `exceptions` | 0 | nothing failed |

> [!NOTE]
> Out of the box you get **request-level** telemetry, not distributed tracing. `duration` is
> server-side and excludes the client round trip — compare 6536 ms here with the 7.5 s the
> `invoke` command reported. For span-level detail inside your agent you still have to
> instrument your own code with OpenTelemetry.

`customEvents` is the useful Foundry-specific table once you start running evaluations —
eval results land there, which is what lets you correlate a score back to the exact response
that produced it. It stays empty until an eval runs.

---

## A practical ladder

| Stage | Tool | Cost |
|---|---|---|
| "Did it start?" | `azd ai agent monitor -f` | free |
| "Why is it slow?" | `invoke` timing line | free |
| "Which version answered?" | `azd ai agent show` + pinned sessions | free |
| "What happened last Tuesday?" | App Insights | billed |
| "Is quality regressing?" | `azd ai agent eval run` + `--trace-days` | billed |

`eval generate --trace-days` builds an evaluation dataset **from production traces**, which
only works once traces exist — another reason to wire App Insights before you need it.

---

## See also

- 👉 [`azd-cli.md`](./azd-cli.md) — full `monitor` and `sessions` surface
- 👉 [`identity-and-rbac.md`](./identity-and-rbac.md) — decoding the MSAL lines above
- 👉 [`troubleshooting.md`](./troubleshooting.md) — symptom index
- 👉 [`cost.md`](./cost.md) — what App Insights adds to the bill
