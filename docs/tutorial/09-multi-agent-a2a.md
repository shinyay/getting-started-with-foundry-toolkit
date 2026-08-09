# 🧪 Lab 09 — Agent-to-agent (A2A) delegation

> ⏱️ **60 min** · 📋 **Requires:** [Lab 05](05-mcp-toolbox.md) · 💰 **two projects running at once** · ☁️ **Creates 4 Azure resources across 2 resource groups**

Deploy two agents in two separate Foundry projects and make one delegate to the other.

> ⚠️ **Read this before you start.** This is the one lab in the repo that **does not reach a
> green finish**. We ran it end-to-end on live Azure on 2026-08-09 with the toolkit's own
> Python sample: everything up to the final delegation is verified working, and the last step
> reproducibly fails. We are shipping it because the failure is *instructive* and because
> three of the defects it exposes will bite you in unrelated projects.
>
> If you want a lab that finishes green, do [Lab 05](05-mcp-toolbox.md) instead.

## What you'll learn

- What an **agent card** actually contains — we fetched a real one.
- How a `RemoteA2A` connection + `a2a_preview` toolbox fit together.
- 🔴 That **an unset `${VAR}` in `azure.yaml` expands to empty silently**.
- 🔴 That **a broken agent returns HTTP 200 with an empty body** and `invoke` exits **0**.
- 🔴 That `azd ai agent monitor` — not `invoke` — is where the truth lives.

---

## The shape

```text
caller project                          executor project
┌─────────────────────────┐             ┌──────────────────────────┐
│ caller agent            │             │ executor agent           │
│   └─ Toolbox (MCP)      │             │   (math expert)          │
│        └─ a2a_preview   │──── A2A ───▶│   incoming A2A enabled   │
│             └─ RemoteA2A│             │   + agent card published │
│                connection             └──────────────────────────┘
└─────────────────────────┘
```

Two **separate `azd` projects**, so two resource groups and two Foundry accounts. That is the
point: A2A is a boundary between independently owned agents, not a function call.

---

## Steps

### 1. Deploy the executor

```bash
mkdir -p a2a/executor && cd a2a/executor
azd ai agent init -m "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/a2a/01-delegation/executor/azure.yaml" --no-prompt
cd agent-framework-a2a-executor-responses
```

> [!NOTE]
> ✅ Verified: `azd ai agent init -m` creates a **subdirectory** named after the sample. You do
> not `cd` into a directory you made — you `cd` into the one it made.

```bash
azd env set AZURE_SUBSCRIPTION_ID <sub-id>
azd env set AZURE_LOCATION eastus2
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
azd provision    # measured 1m24s
azd deploy       # measured 2m26s
```

Confirm it works **on its own** before adding any A2A:

```bash
azd ai agent invoke agent-framework-a2a-executor-responses "What is 4871 multiplied by 293?"
```

```text
[agent-framework-a2a-executor-responses] 1,427,203 — I multiplied 4871 by 293 by
using 4871×300 − 4871×7.
Server responded in 14.786s (first byte: 7.958s)
```

### 2. Enable incoming A2A

The sample ships `scripts/setup-a2a.sh`, which PATCHes the agent to publish an `agent_card`
and add `a2a` to its protocols.

> [!WARNING]
> 🔴 **The script fails as shipped.** It reads `../.env`, but `azd` writes environment values
> to `.azure/<env>/.env`. Running it bare:
>
> ```text
> Error: FOUNDRY_PROJECT_ENDPOINT is not set (expected in .../scripts/../.env).
> ```
>
> Supply the value from `azd` instead:

```bash
EP=$(azd env get-value FOUNDRY_PROJECT_ENDPOINT)
cd src/agent-framework-a2a-executor-responses/scripts
FOUNDRY_PROJECT_ENDPOINT="$EP" ./setup-a2a.sh
```

```text
Enabling incoming A2A on agent 'agent-framework-a2a-executor-responses'...
done.

Incoming A2A enabled.
  A2A endpoint:  https://…/agents/…/endpoint/protocols/a2a
  Agent card:    https://…/agents/…/endpoint/protocols/a2a/agentCard/v0.3
```

### 3. Read the agent card — this is the interesting bit

```bash
TOK=$(az account get-access-token --resource https://ai.azure.com --query accessToken -o tsv)
curl -sS -H "Authorization: Bearer $TOK" "<agent-card-url>" | python3 -m json.tool
```

<details>
<summary>✅ Verified live agent card</summary>

```json
{
    "name": "agent-framework-a2a-executor-responses",
    "description": "A math expert that performs arithmetic operations and explains the steps.",
    "url": "https://…/endpoint/protocols/a2a",
    "version": "1.0",
    "protocolVersion": "0.3",
    "capabilities": {
        "streaming": false,
        "pushNotifications": false,
        "stateTransitionHistory": false,
        "extensions": []
    },
    "defaultInputModes": ["text"],
    "defaultOutputModes": ["text"],
    "skills": [
        {
            "id": "arithmetic",
            "name": "Arithmetic and math expert",
            "description": "Performs arithmetic operations … and returns concise numeric answers.",
            "tags": [],
            "examples": []
        }
    ],
    "supportsAuthenticatedExtendedCard": false,
    "additionalInterfaces": [
        { "transport": "HTTP+JSON", "url": "https://…/endpoint/protocols/a2a" }
    ],
    "preferredTransport": "JSONRPC"
}
```
</details>

Worth noticing: the card advertises `preferredTransport: JSONRPC` while
`additionalInterfaces` offers `HTTP+JSON`; `streaming` is **false**, so a caller cannot expect
token-by-token output from a delegated call. **The card is the contract** — everything the
caller knows about the executor comes from this document.

### 4. Deploy the caller

```bash
cd ../../../..   # back to a2a/
mkdir -p caller && cd caller
azd ai agent init -m "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/a2a/01-delegation/caller/azure.yaml" --no-prompt
cd agent-framework-a2a-caller-responses

azd env set AZURE_SUBSCRIPTION_ID <sub-id>
azd env set AZURE_LOCATION eastus2
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
azd env set a2a_executor_endpoint "<the A2A endpoint from step 2>"
azd env set TOOLBOX_NAME a2a-delegation-tools      # ⬅ do not skip this, see below
azd provision    # measured 1m30s
azd deploy       # measured 2m22s
```

> [!CAUTION]
> 🔴 **`azd env set TOOLBOX_NAME` is not optional, and nothing tells you that.**
>
> The caller's `azure.yaml` declares `TOOLBOX_NAME: ${TOOLBOX_NAME}`. If the variable is unset,
> **azd substitutes an empty string without warning**, and the container builds this URL:
>
> ```text
> POST …/toolboxes//mcp?api-version=v1   →   HTTP 405 Method Not Allowed
>                    ↑↑ empty toolbox name
> ```
>
> The correct value is the toolbox service key from `azure.yaml` — `a2a-delegation-tools`. It
> is also in the sample's `.env.example`, which is the only place it is written down.
>
> **The general lesson matters more than this one variable:** any `${VAR}` in `azure.yaml` that
> you have not set becomes an empty string in your container. Check with
> `azd env get-values` before deploying.

Confirm the connection was created — note it points **into the other project**:

```bash
az cognitiveservices account connection list --name <account> -g <rg> \
  --query "[].{name:name,category:properties.category,auth:properties.authType}" -o table
```

```text
Name             Category    Auth
---------------  ----------  --------------
math-expert-a2a  RemoteA2A   UserEntraToken
```

### 5. Invoke — and learn to distrust `invoke`

```bash
azd ai agent invoke agent-framework-a2a-caller-responses "What is 4871 multiplied by 293? Show the steps."
```

```text
Session:      24b123a0…  (assigned by server)
Trace ID:     30f2707e12f8bc3d4555543be7c540f3
Server responded in 9.010s (first byte: 8.294s)
```

Read that carefully. **There is no agent line.** No answer, no error — and:

```bash
echo $?    # → 0
```

> [!CAUTION]
> 🔴 **This is the single most important thing in this lab.** A hosted agent that throws
> during tool initialisation still returns **HTTP 200 with an empty body**, and
> `azd ai agent invoke` **exits 0**. A CI pipeline that checks the exit code sees a pass.
>
> **An empty agent line is a failure.** Treat it as one.

### 6. Find the real error

```bash
azd ai agent monitor agent-framework-a2a-caller-responses
```

> [!NOTE]
> The command is `monitor`. There is no `azd ai agent logs` — it returns
> `ERROR: unknown command "logs" for "agent"`.

```text
mcp.shared.exceptions.McpError: tools/list failed for 1 tool source(s), succeeded for 0
{"errors":[{"name":"math-expert-a2a","type":"a2a_preview","error":{
  "code":"CONNECTION_FAILED",
  "message":"Connection response for UserIdentity connection '…/connections/math-expert-a2a'
             is missing 'Audience'/'TokenAudience'. Token exchange cannot proceed
             without an audience."}}]}
```

---

## 🛑 Where this stops — and why we are telling you

The caller's `azure.yaml` **does** declare the audience:

```yaml
  math-expert-a2a:
    host: azure.ai.connection
    category: RemoteA2A
    authType: UserEntraToken
    audience: https://ai.azure.com      # ⬅ declared here
```

But the connection `azd provision` actually created contains only:

```json
"metadata": { "AgentCardPath": "/agentCard/v0.3" }
```

🔴 **`azd provision` silently drops `audience` when creating a `RemoteA2A` connection.**
That is a defect in the manifest-to-connection mapping, not a mistake in the sample.

We tried to repair it by hand and **could not**. For the record, all of these were applied
successfully and none changed the runtime error:

| Attempt | Result |
|---|---|
| `Audience` + `TokenAudience` in `metadata`, **account** scope | persisted ✅ · error unchanged |
| same at **project** scope (the scope named in the error) | persisted ✅ · error unchanged |
| top-level `audience` property on `properties` | persisted ✅ · error unchanged |
| redeploy the caller after each patch | ✅ · error unchanged |

**Most likely cause:** `authType: UserEntraToken` means the toolbox forwards **the calling
user's** Entra token to the executor. `azd ai agent invoke` does not appear to supply a token
the executor accepts across two separate Foundry projects, so the exchange has no audience to
target and fails before `tools/list` returns.

**Status: A2A delegation is not verified end-to-end.** Everything up to `tools/list` is.
We would rather ship an honest dead end than a walkthrough that quietly skips the part that
does not work. If you get this working, the fix belongs in this table.

---

## ✅ Checkpoint

Because the final step does not pass, the checkpoint is what you *can* prove. All three should
hold:

```bash
# 1. the executor answers directly
azd ai agent invoke agent-framework-a2a-executor-responses "What is 2+2?"
#    → an agent line with an answer

# 2. the agent card is published
curl -sS -H "Authorization: Bearer $TOK" "<card-url>" | python3 -c "import json,sys; c=json.load(sys.stdin); print(c['protocolVersion'], [s['id'] for s in c['skills']])"
#    → 0.3 ['arithmetic']

# 3. the toolbox URL has a name in it — check BEFORE deploying
azd env get-value TOOLBOX_NAME
#    → a2a-delegation-tools   (an error here means you'd have shipped toolboxes//mcp)
```

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| `Error: FOUNDRY_PROJECT_ENDPOINT is not set` | `setup-a2a.sh` reads `../.env`; azd writes `.azure/<env>/.env`. | `FOUNDRY_PROJECT_ENDPOINT=$(azd env get-value FOUNDRY_PROJECT_ENDPOINT) ./setup-a2a.sh` |
| `HTTP 405` on `toolboxes//mcp` | `TOOLBOX_NAME` unset → empty substitution. | `azd env set TOOLBOX_NAME a2a-delegation-tools`, redeploy. |
| Empty response, `invoke` exits 0 | The agent threw during tool init. | `azd ai agent monitor <agent>`. Never trust the exit code alone. |
| `CONNECTION_FAILED … missing 'Audience'` | `azd provision` dropped the manifest's `audience`. | ⚠️ No known fix — see the table above. |
| `unknown command "logs"` | It's `monitor`. | `azd ai agent monitor <agent>` |
| Two resource groups linger | Two projects, two `azd` environments. | `azd down --force --purge` in **both** directories. |

## ✏️ Exercise

You inherit an agent whose `azure.yaml` contains `API_BASE: ${API_BASE}`, and `API_BASE` was
never set. Deploy succeeds, `invoke` exits 0, output is empty. Which of these finds it fastest?

- **A.** Re-read `azure.yaml`
- **B.** `azd env get-values`
- **C.** `azd ai agent monitor <agent>`
- **D.** `azd ai agent doctor`

<details>
<summary>Solution</summary>

**C, then B.**

`monitor` shows the *symptom in the request itself* — a malformed URL or a 4xx containing the
empty value — which tells you which variable is wrong. Then `azd env get-values` confirms it is
missing.

- **A** is where the trap is *defined*, not where it is *visible*. `${API_BASE}` looks correct.
- **B** alone shows a long list with no indication of which entry matters.
- **D** validates project structure, not runtime values.

The habit worth keeping: after any deploy that returns an empty body, go straight to
`monitor`. `invoke` exiting 0 tells you the HTTP call succeeded — nothing more.
</details>

## → Next

🧪 [Lab 10 — CI/CD](10-cicd.md) · 📘 or the concepts behind this:
[Multi-agent patterns](../learn/10-multi-agent.md)

---

<sub>[← Observability](08-observability.md) · [🧪 Tutorial index](README.md) · [CI/CD →](10-cicd.md)</sub>
