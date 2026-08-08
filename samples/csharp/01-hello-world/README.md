# 01 · Hello World (C#)

> **New idea:** a hosted agent is just an HTTP server that speaks a protocol.

```bash
mkdir 01-hello && cd 01-hello
azd ai agent init -m "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/csharp/hosted-agents/agent-framework/hello-world/azure.yaml"
```

---

## The code

```csharp
using Azure.AI.AgentServer.Core;
using Azure.AI.Projects;
using Azure.Identity;
using DotNetEnv;
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.Foundry.Hosting;

Env.NoClobber().TraversePath().Load();          // ① local .env, never clobbering real env

var projectEndpoint = new Uri(Environment.GetEnvironmentVariable("FOUNDRY_PROJECT_ENDPOINT")
    ?? throw new InvalidOperationException("FOUNDRY_PROJECT_ENDPOINT environment variable is not set."));

var deployment = Environment.GetEnvironmentVariable("AZURE_AI_MODEL_DEPLOYMENT_NAME")
    ?? throw new InvalidOperationException("AZURE_AI_MODEL_DEPLOYMENT_NAME environment variable is not set.");

AIAgent agent = new AIProjectClient(projectEndpoint, new DefaultAzureCredential())   // ②
    .AsAIAgent(
        model: deployment,
        instructions: "You are a helpful AI assistant. Be concise and informative.", // ③
        name: "hello-world",
        description: "A minimal Hello World agent using the Agent Framework");

var builder = AgentHost.CreateBuilder(args);                                        // ④
builder.Services.AddFoundryResponses(agent);
builder.RegisterProtocol("responses", endpoints => endpoints.MapFoundryResponses());

var app = builder.Build();
app.Run();
```

| | Why it matters |
|---|---|
| ① `Env.NoClobber().TraversePath()` | reads `.env` for local dev but **never overwrites** real env vars, so hosted containers keep injected values |
| ② `DefaultAzureCredential` | `az login` locally, **managed identity** in Azure — same code both places |
| ③ `instructions` | the entire prompt |
| ④ `AgentHost.CreateBuilder` | Kestrel on **8088**, `GET /readiness`, OpenTelemetry, `x-platform-server` header |

**No `ResponseHandler` subclass.** The framework manages the LLM call, conversation history and
response lifecycle.

### Multi-turn

The framework calls `GetHistoryAsync()` internally. Pass `previous_response_id` from one
response into the next request to continue a conversation.

| Environment | History storage |
|---|---|
| local | in-process — **lost on restart** |
| hosted (`FOUNDRY_HOSTING_ENVIRONMENT` set) | durable server-side |

---

## `azure.yaml` essentials

```yaml
language: csharp
codeConfiguration:
    runtime: dotnet_10
    entryPoint: hello-world.dll      # ← the ASSEMBLY, not Program.cs
```

---

## Run it

```bash
azd env set AZURE_SUBSCRIPTION_ID <id>
azd env set AZURE_LOCATION eastus2
azd provision
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini     # ⚠️ required
azd ai agent run --no-client
```

```bash
azd ai agent invoke --local "What is Microsoft Foundry?"
```

Raw HTTP works too:

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What is Microsoft Foundry?", "stream": false}' | jq .

# follow-up, using the id from the previous response
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Can you summarize that?", "previous_response_id": "<id>", "stream": false}' | jq .
```

Or plain `dotnet run` after copying `.env.example` → `.env`.

---

## Expected warning

```text
[WARNING] APPLICATIONINSIGHTS_CONNECTION_STRING not set — traces will not be sent to
Application Insights.
```

Harmless locally. **Do not** declare that variable in your manifest — it is auto-injected in
hosted containers, and declaring it shadows the real value.

---

## Deploy

```bash
azd deploy
azd ai agent invoke "What is Microsoft Foundry?"
azd ai agent show --output json
azd down --force --purge
```

---

👉 Next: [02 · Tools](../02-tools/)
