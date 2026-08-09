# 01 · Hello World (C#)

> ⏱️ **15 min** · 📋 **Requires:** [setup](../../../docs/tutorial/01-setup.md) · 💰 **token + hosted-agent compute** · ☁️ **Creates 2 Azure resources**

Deploy the smallest C# Agent Framework agent and prove it responds through the Foundry host.

> [!NOTE]
> **Verified end-to-end on 2026-08-08** against live Azure (`eastus2`, `azd 1.30.0`):
> `provision` 1m43s → 2 resources · `deploy` 3m1s → agent `active` · `invoke` responded in
> 6.877s (first byte 3.622s) · `azd down --force --purge` 1m45s, verified back to zero.

## What you'll learn

- Run a checked-in C# `azure.yaml` sample.
- Map `dotnet_10` and `entryPoint: hello-world.dll` to the built assembly.
- Use `Env.NoClobber().TraversePath()` for local `.env` without shadowing hosted variables.
- Clean up the two Azure resources the sample creates.

## Steps

### 1. Open this sample

```bash
cd samples/csharp/01-hello-world
```

### 2. Inspect the agent shape

```csharp
Env.NoClobber().TraversePath().Load();

var projectEndpoint = new Uri(Environment.GetEnvironmentVariable("FOUNDRY_PROJECT_ENDPOINT")
    ?? throw new InvalidOperationException("FOUNDRY_PROJECT_ENDPOINT environment variable is not set."));

var deployment = Environment.GetEnvironmentVariable("AZURE_AI_MODEL_DEPLOYMENT_NAME")
    ?? throw new InvalidOperationException("AZURE_AI_MODEL_DEPLOYMENT_NAME environment variable is not set.");

AIAgent agent = new AIProjectClient(projectEndpoint, new DefaultAzureCredential())
    .AsAIAgent(
        model: deployment,
        instructions: "You are a helpful AI assistant. Be concise and informative.",
        name: "hello-world",
        description: "A minimal Hello World agent using the Agent Framework");

var builder = AgentHost.CreateBuilder(args);
builder.Services.AddFoundryResponses(agent);
builder.RegisterProtocol("responses", endpoints => endpoints.MapFoundryResponses());
```

| | Why it matters |
|---|---|
| `Env.NoClobber().TraversePath()` | reads `.env` locally but does not overwrite hosted env vars |
| `DefaultAzureCredential` | `az login` locally, managed identity in Azure |
| `instructions` | the entire prompt |
| `AgentHost.CreateBuilder` | Kestrel on 8088, readiness, telemetry, Responses protocol |

### 3. Check the manifest entry point

```yaml
language: csharp
codeConfiguration:
  runtime: dotnet_10
  entryPoint: hello-world.dll
```

The entry point is the assembly, not `Program.cs`.

### 4. Provision, run, deploy, and clean up

```bash
azd env set AZURE_SUBSCRIPTION_ID <id>
azd env set AZURE_LOCATION eastus2
azd provision
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
azd ai agent run --no-client
```

In a second terminal:

```bash
azd ai agent invoke --local "What is Microsoft Foundry?"
```

Raw HTTP works too after the local server is running:

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What is Microsoft Foundry?", "stream": false}' | jq .
```

Deploy and remove resources:

```bash
azd deploy
azd ai agent invoke "What is Microsoft Foundry?"
azd ai agent show --output json
azd down --force --purge
```

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
| `AZURE_AI_MODEL_DEPLOYMENT_NAME environment variable is not set.` | The model env var was not set after provision. | Run `azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini`. |
| Warning about `APPLICATIONINSIGHTS_CONNECTION_STRING`. | Telemetry is not configured locally. | Harmless locally; do not declare it in `azure.yaml`. |
| `project` path check prints `False`. | Manifest and source folder are out of sync. | Restore `project: src/hello-world` or the folder name. |

## ✏️ Exercise

Predict what happens if `entryPoint` is changed from `hello-world.dll` to `Program.cs`.

<details>
<summary>Solution</summary>

The host will look for an executable assembly named `Program.cs`, which is not what `dotnet
publish` produces. The entry point must be the built DLL.
</details>

## Provenance

This sample is **adapted from upstream**, not invented here. Exact mapping so you can diff it:

| | |
|---|---|
| **Upstream path** | [`csharp/hosted-agents/agent-framework/hello-world`](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/csharp/hosted-agents/agent-framework/hello-world) |
| **Upstream source dir** | `src/hello-world-dotnet-agent-framework` |
| **Source dir here** | `src/hello-world` |
| **Deviations** | `azure.yaml` `name`, service key and `project` all renamed to match the shortened directory. **This rename is exactly what broke the manifest** until it was fixed — see the note below. |

Everything under `src/` other than `azure.yaml` is **byte-identical** to upstream output.
Regenerate the original at any time with `azd ai agent init -m "<upstream azure.yaml URL>"`.

## → Next

[02 · Tools](../02-tools/)
