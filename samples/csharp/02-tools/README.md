# 02 · Tools (C#)

> ⏱️ **20 min** · 📋 **Requires:** [01 · Hello World](../01-hello-world/) · 💰 **token + hosted-agent compute** · ☁️ **Creates 2 Azure resources**

Add a local C# function so the agent can act as a Seattle hotel concierge.

## What you'll learn

- Expose a C# method as an Agent Framework tool with `AIFunctionFactory.Create`.
- See how tool names and descriptions guide model tool choice.
- Compare C# tool declarations with Python `@tool` declarations.
- Keep local env and Docker ignore files aligned with the Python rung.

## Steps

### 1. Open this sample

```bash
cd samples/csharp/02-tools
```

### 2. Inspect the only real change from 01

```csharp
AIAgent agent = new AIProjectClient(projectEndpoint, new DefaultAzureCredential())
    .AsAIAgent(
        model: deployment,
        instructions: """
            You are a helpful Seattle hotel concierge assistant.
            Use the available tools to help customers find hotels in Seattle.
            Provide detailed information about available hotels when asked.
            """,
        name: "local-tools",
        description: "A hotel concierge assistant with local function tools",
        tools:
        [
            AIFunctionFactory.Create(
                GetAvailableHotels,
                "GetAvailableHotels",
                "Gets a list of available hotels in Seattle with details about amenities and pricing.")
        ]);

static string GetAvailableHotels(string? checkInDate = null, string? checkOutDate = null, int? guests = null)
{
    // returns hotel data as a string
}
```

| | Python equivalent |
|---|---|
| method group | decorated function |
| explicit tool name | function `__name__` |
| explicit description | docstring |

The description is prompt engineering. The model reads it to decide whether to call the tool.

### 3. Provision and run locally

```bash
azd env set AZURE_SUBSCRIPTION_ID <id>
azd env set AZURE_LOCATION eastus2
azd provision
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
azd ai agent run --no-client
```

In a second terminal:

```bash
azd ai agent invoke --local "I need a hotel in Seattle for 2 guests next weekend."
azd ai agent invoke --local "What is 2+2?"
```

Run without `--no-client` to watch the call/result timeline in Agent Inspector.

### 4. Deploy and monitor

```bash
azd deploy
azd ai agent invoke "Find me a hotel in Seattle."
azd ai agent monitor -f
azd down --force --purge
```

Tools run inside your container. Nothing is registered with Foundry as a separate tool resource.

## ✅ Checkpoint

From this directory, verify that the manifest points at a real source folder:

```bash
python3 -c "import os,yaml; d=yaml.safe_load(open('azure.yaml')); s=d['services']['local-tools']; print(s['project'], os.path.isdir(s['project']))"
```

```text
src/local-tools True
```

If you see something else, jump to *If that didn't work* below.

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| Hotel prompt does not use the tool. | Tool description is the model's routing hint. | Ask for Seattle hotel availability and keep the description specific. |
| Local `dotnet run` cannot find env values. | `.env` was not created from `.env.example`. | Copy `src/local-tools/.env.example` to `.env` and fill local-only values. |
| `project` path check prints `False`. | Manifest and source folder are out of sync. | Restore `project: src/local-tools` or the folder name. |

## ✏️ Exercise

Predict whether `azd ai agent invoke --local "What is 2+2?"` should call `GetAvailableHotels`.

<details>
<summary>Solution</summary>

It should not call the tool. The prompt does not ask about hotels, Seattle, guests, or dates, so
the model can answer directly.
</details>

## Provenance

This sample is **adapted from upstream**, not invented here. Exact mapping so you can diff it:

| | |
|---|---|
| **Upstream path** | [`csharp/hosted-agents/agent-framework/local-tools`](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/csharp/hosted-agents/agent-framework/local-tools) |
| **Upstream source dir** | `src/local-tools` |
| **Source dir here** | `src/local-tools` |
| **Deviations** | `azure.yaml` reindented/reordered; directory name unchanged from upstream. |

Everything under `src/` other than `azure.yaml` is **byte-identical** to upstream output.
Regenerate the original at any time with `azd ai agent init -m "<upstream azure.yaml URL>"`.

## → Next

[03 · MCP tools](../03-mcp-toolbox/)
