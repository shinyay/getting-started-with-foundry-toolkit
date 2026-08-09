# 02 · Tools

> ⏱️ **20 min** · 📋 **Requires:** [01 · Hello World](../01-hello-world/) · 💰 **token + hosted-agent compute** · ☁️ **Creates 2 Azure resources**

Add a local Python function so the agent can do something instead of only chatting.

## What you'll learn

- Expose an ordinary Python function as an Agent Framework tool.
- Understand why docstrings and parameter descriptions are prompt surface.
- Compare a prompt that should call a tool with one that should not.
- Deploy without registering tools separately in Foundry.

## Steps

### 1. Open this sample

```bash
cd samples/python/02-tools
```

### 2. Inspect the only real change from 01

```python
@tool(approval_mode="never_require")
def get_weather(
    location: Annotated[str, Field(description="The location to get the weather for.")],
) -> str:
    """Get the weather for a given location."""
    conditions = ["sunny", "cloudy", "rainy", "stormy"]
    return f"The weather in {location} is {conditions[randint(0, 3)]} with a high of {randint(10, 30)}°C."

agent = Agent(
    client=client,
    instructions="You are a friendly assistant. Keep your answers brief.",
    tools=[get_weather],
    default_options={"store": False},
)
```

| Source | Becomes |
|---|---|
| function name | the tool name |
| docstring | the tool description the model reads |
| `Annotated[..., Field(description=...)]` | the parameter schema description |
| return type | the result contract |

`approval_mode="never_require"` is appropriate here because the tool is safe and read-only.
Use approval for anything that writes, spends money, or sends messages.

### 3. Provision and run locally

```bash
azd env set AZURE_SUBSCRIPTION_ID <your-subscription-id>
azd env set AZURE_LOCATION eastus2
azd provision
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
azd ai agent run --no-client
```

### 4. Invoke prompts that should and should not use the tool

```bash
azd ai agent invoke --local "What's the weather in Tokyo?"
azd ai agent invoke --local "What is 2+2?"
```

Run without `--no-client` if you want Agent Inspector to show the call/result timeline.

### 5. Deploy and monitor

```bash
azd deploy
azd ai agent invoke "What's the weather in Osaka?"
azd ai agent monitor -f
azd down --force --purge
```

Tools run inside your container. Nothing is registered with Foundry as a separate tool resource.

## ✅ Checkpoint

From this directory, verify that the manifest points at a real source folder:

```bash
python3 -c "import os,yaml; d=yaml.safe_load(open('azure.yaml')); s=d['services']['tools-agent']; print(s['project'], os.path.isdir(s['project']))"
```

```text
src/tools-agent True
```

If you see something else, jump to *If that didn't work* below.

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| Weather prompts never use the tool. | The tool name/docstring/parameter descriptions are the model's tool prompt. | Keep descriptions specific and ask for weather in a location. |
| `KeyError: 'AZURE_AI_MODEL_DEPLOYMENT_NAME'` | The model env var was not set. | Run `azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini`. |
| Inspector does not open. | You started with `--no-client`. | Run `azd ai agent run` without `--no-client`. |

## ✏️ Exercise

Predict whether `azd ai agent invoke --local "What is 2+2?"` should call `get_weather`.

<details>
<summary>Solution</summary>

It should not call the tool. The prompt does not mention weather or a location, so the model can
answer directly.
</details>

## Provenance

This sample is **adapted from upstream**, not invented here. Exact mapping so you can diff it:

| | |
|---|---|
| **Upstream path** | [`python/hosted-agents/agent-framework/responses/02-tools`](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/python/hosted-agents/agent-framework/responses/02-tools) |
| **Upstream source dir** | `src/agent-framework-agent-with-local-tools-responses` |
| **Source dir here** | `src/tools-agent` |
| **Deviations** | `azure.yaml` reindented/reordered and renamed; source unchanged. |

Everything under `src/` other than `azure.yaml` is **byte-identical** to upstream output.
Regenerate the original at any time with `azd ai agent init -m "<upstream azure.yaml URL>"`.

## → Next

[03 · MCP & Foundry Toolbox](../03-mcp-toolbox/)
