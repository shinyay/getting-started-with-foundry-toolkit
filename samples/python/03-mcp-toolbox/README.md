# 03 · MCP & Foundry Toolbox

> ⏱️ **25 min** · 📋 **Requires:** [02 · Tools](../02-tools/) · 💰 **token + hosted-agent compute** · ☁️ **Creates 2 Azure resources; toolbox setup is separate**

Connect the agent to external tools through Foundry Toolbox instead of functions in your process.

> [!WARNING]
> **Broken catalog entry (verified 2026-08-08).** `azd ai agent sample list --language python`
> advertises a Python MCP sample at
> `…/agent-framework/responses/03-mcp/azure.yaml`, but that path **404s**. The C# equivalent
> is live. This Python rung therefore uses Foundry Toolbox (`04-foundry-toolbox`, verified 200)
> and the C# rung covers direct MCP.

## What you'll learn

- Distinguish local tools from external toolbox/MCP tools.
- Configure the `TOOLBOX_ENDPOINT` that this sample needs.
- Understand why toolbox creation is not proof that every connection works.
- Check toolbox wiring before deploying.

## Steps

### 1. Open this sample

```bash
cd samples/python/03-mcp-toolbox
```

### 2. Inspect the async toolbox shape

```python
credential = DefaultAzureCredential()
toolbox = FoundryToolbox(credential)

client = FoundryChatClient(
    project_endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"],
    model=os.environ["AZURE_AI_MODEL_DEPLOYMENT_NAME"],
    credential=credential,
)

agent = Agent(
    client=client,
    instructions="You are a friendly assistant. Keep your answers brief.",
    tools=toolbox,
    default_options={"store": False},
)

await ResponsesHostServer(agent).run_async()
```

| Change vs 02 | Why |
|---|---|
| `async def main()` + `run_async()` | tool connections are I/O-bound |
| `tools=toolbox` | the toolbox enumerates its own tools |
| shared `credential` | one identity for model and tools |

### 3. Create or point at a toolbox

`azure.yaml` declares the endpoint as an env var:

```yaml
environmentVariables:
  - name: AZURE_AI_MODEL_DEPLOYMENT_NAME
    value: ${AZURE_AI_MODEL_DEPLOYMENT_NAME}
  - name: TOOLBOX_ENDPOINT
    value: ${TOOLBOX_ENDPOINT}
```

Create the toolbox from the vendored definition, then copy the endpoint into the azd env:

```bash
cd src/toolbox-agent
azd ai toolbox create agent-tools --from-file ./toolbox.yaml
azd ai toolbox show agent-tools
azd env set TOOLBOX_ENDPOINT "<endpoint from show>"
```

> [!IMPORTANT]
> `azd ai toolbox` is provided by the separate `azure.ai.toolboxes` extension. The first
> invocation can prompt to install it.

> [!IMPORTANT]
> **`create` does not validate project connections — verified live on 2026-08-08.** Running it
> against a project with none of the named connections still succeeded:
>
> ```text
> toolbox create: resolved project endpoint https://cog-<token>.services.ai.azure.com/api/projects/rdpy (source=azdEnv)
> Created toolbox agent-tools at version 1.
> Endpoint: https://cog-<token>.services.ai.azure.com/api/projects/rdpy/toolboxes/agent-tools/versions/1/mcp?api-version=v1
> exit_code=0
> ```
>
> Exit code 0 means the definition was packaged. Broken connections fail later, when the agent
> calls a tool.

For a first run, trim `toolbox.yaml` to tools that need no connection: `web_search`,
`code_interpreter`, and `noauth_mcp`.

### 4. Provision, check, run, and deploy

```bash
azd env set AZURE_SUBSCRIPTION_ID <your-subscription-id>
azd env set AZURE_LOCATION eastus2
azd provision
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
azd ai agent doctor
azd ai agent run --no-client
azd ai agent invoke --local "Search for the latest Microsoft Foundry release notes."
azd deploy
azd down --force --purge
```

External tools need permissions. The deployed agent uses its own managed identity from
`azd ai agent show --output json`, not your user account.

## ✅ Checkpoint

From this directory, verify that the manifest points at a real source folder:

```bash
python3 -c "import os,yaml; d=yaml.safe_load(open('azure.yaml')); s=d['services']['toolbox-agent']; print(s['project'], os.path.isdir(s['project']))"
```

```text
src/toolbox-agent True
```

If you see something else, jump to *If that didn't work* below.

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| `doctor` reports the toolbox endpoint env var missing. | `TOOLBOX_ENDPOINT` is declared but not set in the azd env. | Run `azd env set TOOLBOX_ENDPOINT "<endpoint>"`. |
| `toolbox create` succeeds but calls fail. | Connection names are resolved at invoke time, not create time. | Trim to no-connection tools or create the missing project connections. |
| Local works but hosted fails. | The hosted agent has a different managed identity. | Grant permissions to the agent principal from `azd ai agent show --output json`. |

## ✏️ Exercise

Predict what `azd ai agent doctor` reports for the toolbox-endpoint check if `TOOLBOX_ENDPOINT`
is not set.

<details>
<summary>Solution</summary>

It should fail the toolbox endpoint check. The manifest declares `TOOLBOX_ENDPOINT`, so the
sample is not wired until the azd environment contains that variable.
</details>

## Provenance

This sample is **adapted from upstream**, not invented here. Exact mapping so you can diff it:

| | |
|---|---|
| **Upstream path** | [`python/hosted-agents/agent-framework/responses/04-foundry-toolbox`](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/python/hosted-agents/agent-framework/responses/04-foundry-toolbox) |
| **Upstream source dir** | `src/agent-framework-agent-with-foundry-toolbox-responses` |
| **Source dir here** | `src/toolbox-agent` |
| **Deviations** | `azure.yaml` renamed and reordered. Directory numbered `03` here because upstream `03-mcp` **does not exist** — see [sample catalog](../../../docs/reference/sample-catalog.md). |

Everything under `src/` other than `azure.yaml` is **byte-identical** to upstream output.
Regenerate the original at any time with `azd ai agent init -m "<upstream azure.yaml URL>"`.

## → Next

[04 · Evaluation](../04-eval/)
