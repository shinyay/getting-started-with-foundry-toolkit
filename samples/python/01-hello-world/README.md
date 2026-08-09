# 01 · Hello World

> ⏱️ **15 min** · 📋 **Requires:** [setup](../../../docs/tutorial/01-setup.md) · 💰 **token + hosted-agent compute** · ☁️ **Creates 2 Azure resources**

Deploy the smallest Python Agent Framework agent and prove it responds through the Foundry host.

> [!NOTE]
> **Verified end-to-end on 2026-08-08** against live Azure (`eastus2`, `azd 1.30.0`):
> `provision` 1m20s → 2 resources · `deploy` 2m21s → agent `active` · `invoke` responded in
> 14.242s (first byte 7.357s) · `azd down --force --purge` 1m46s, verified back to zero.

## What you'll learn

- Run a checked-in `azure.yaml` sample instead of scaffolding a new project.
- Identify the three pieces of a Responses-hosted Python agent: client, agent, host server.
- Set the model deployment variable that `azd provision` does not set for you.
- Clean up the two Azure resources the sample creates.

## Steps

### 1. Open this sample

```bash
cd samples/python/01-hello-world
```

### 2. Inspect the agent shape

```python
client = FoundryChatClient(
    project_endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"],
    model=model_name,
    credential=DefaultAzureCredential(),
)

agent = Agent(
    client=client,
    instructions="You are a friendly assistant. Keep your answers brief.",
    default_options={"store": False},
)

ResponsesHostServer(agent).run()
```

| | Why it matters |
|---|---|
| `FoundryChatClient` | reads `FOUNDRY_PROJECT_ENDPOINT`, which Foundry injects at runtime |
| `DefaultAzureCredential` | your `az login` locally, the agent's managed identity in Azure |
| `instructions` | the entire prompt of this agent |
| `store: False` | the hosting layer owns conversation history |
| `ResponsesHostServer` | wraps the agent in an OpenAI Responses-compatible HTTP server |

### 3. Provision and run locally

```bash
azd env set AZURE_SUBSCRIPTION_ID <your-subscription-id>
azd env set AZURE_LOCATION eastus2
azd provision
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
azd ai agent run --no-client
```

The local server listens on port 8088.

### 4. Invoke it from a second terminal

```bash
azd ai agent invoke --local "In one short sentence, what is Microsoft Foundry?"
```

Expected shape — ⚠️ we captured the **remote** invoke below, not this local one, so the timing
here is a placeholder rather than a recording:

```text
[hello-world] Microsoft Foundry is Microsoft's AI platform for building, customizing,
and deploying AI apps and agents.

Server responded in <n>s (first byte: <n>s)
```

### 5. Deploy, inspect, and clean up

```bash
azd deploy
azd ai agent invoke "What is Microsoft Foundry?"
azd ai agent show --output json
azd down --force --purge
```

`show` reveals that Foundry stored a versioned agent such as `hello-world:1`; redeploying the
same `name:` creates a new version, not a second logical agent.

## ✅ Checkpoint

From this directory, verify that the manifest points at a real source folder:

```bash
python3 -c "import os,yaml; d=yaml.safe_load(open('azure.yaml')); s=d['services']['hello-world']; print(s['project'], os.path.isdir(s['project']))"
```

```text
src/hello-world True
```

If you see something else, jump to *If that didn't work* below.

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| `RuntimeError: Model deployment name is not configured.` | `AZURE_AI_MODEL_DEPLOYMENT_NAME` was not set after provision. | Run `azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini`. |
| `DefaultAzureCredential` mentions `169.254.169.254`. | Local credential probing checked the Azure metadata endpoint. | Ignore it if the command then falls back to `az login`. |
| `project` path check prints `False`. | `azure.yaml` and `src/` are out of sync. | Restore `project: src/hello-world` or rename the folder back. |

## ✏️ Exercise

Before running it, predict what the checkpoint command prints if you rename `src/hello-world`
to `src/renamed-agent` without editing `azure.yaml`.

<details>
<summary>Solution</summary>

It prints `src/hello-world False`. `azd` reads the `project:` path from `azure.yaml`; it does not
search for a replacement directory.
</details>

## Provenance

This sample is **adapted from upstream**, not invented here. Exact mapping so you can diff it:

| | |
|---|---|
| **Upstream path** | [`python/hosted-agents/agent-framework/responses/01-basic`](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/python/hosted-agents/agent-framework/responses/01-basic) |
| **Upstream source dir** | `src/agent-framework-agent-basic-responses` |
| **Source dir here** | `src/hello-world` |
| **Deviations** | `azure.yaml` was reindented, keys reordered, `name` changed to `hello-world`, `description` reworded, and `codeConfiguration.dependencyResolution` + `infra.provider` **added**. |

Everything under `src/` other than `azure.yaml` is **byte-identical** to upstream output.
Regenerate the original at any time with `azd ai agent init -m "<upstream azure.yaml URL>"`.

## → Next

[02 · Tools](../02-tools/)
