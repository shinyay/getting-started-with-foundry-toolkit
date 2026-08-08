# 01 · Hello World

> **New idea:** a hosted agent is just an HTTP server that speaks a protocol.

Everything else in this ladder is a variation on this file.

---

## Scaffold it

```bash
mkdir 01-hello && cd 01-hello
azd ai agent init -m "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/responses/01-basic/azure.yaml"
```

Or copy this folder.

---

## The code

```python
from agent_framework import Agent
from agent_framework.foundry import FoundryChatClient
from agent_framework_foundry_hosting import ResponsesHostServer
from azure.identity import DefaultAzureCredential
from dotenv import load_dotenv

load_dotenv()

def main():
    model_name = os.getenv("AZURE_AI_MODEL_DEPLOYMENT_NAME") or os.getenv("FOUNDRY_MODEL_NAME")

    client = FoundryChatClient(                          # ① talk to Foundry
        project_endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"],
        model=model_name,
        credential=DefaultAzureCredential(),             # ② no keys anywhere
    )

    agent = Agent(                                       # ③ the agent itself
        client=client,
        instructions="You are a friendly assistant. Keep your answers brief.",
        default_options={"store": False},                # ④ host manages history
    )

    ResponsesHostServer(agent).run()                     # ⑤ serve on :8088
```

| | Why it matters |
|---|---|
| ① `FoundryChatClient` | reads `FOUNDRY_PROJECT_ENDPOINT`, which Foundry injects at runtime |
| ② `DefaultAzureCredential` | your `az login` locally, the agent's **managed identity** in Azure — the same code works in both |
| ③ `instructions` | the entire "prompt" of this agent |
| ④ `store: False` | the hosting layer owns conversation history, so the model service doesn't need to |
| ⑤ `ResponsesHostServer` | wraps the agent in an OpenAI **Responses**-compatible HTTP server |

**There is no Foundry-specific request handling in your code.** You write an agent; the host
turns it into a service.

---

## Run it

```bash
azd env set AZURE_SUBSCRIPTION_ID <your-subscription-id>
azd env set AZURE_LOCATION eastus2
azd provision                                        # ~1m25s
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini   # ⚠️ required, see below
azd ai agent run --no-client
```

Wait for:

```text
Running on http://0.0.0.0:8088 (CTRL + C to quit)
```

In a second terminal:

```bash
azd ai agent invoke --local "In one short sentence, what is Microsoft Foundry?"
```

```text
[local] Microsoft Foundry is Microsoft's platform for building, customizing, and
deploying AI apps and agents using foundation models.

Server responded in 9.966s
```

---

## ⚠️ Two things that will trip you up

**1. `AZURE_AI_MODEL_DEPLOYMENT_NAME` is not set by `provision`.** Without it:

```text
RuntimeError: Model deployment name is not configured.
```

```bash
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
```

**2. The `169.254.169.254` traceback is harmless.** `DefaultAzureCredential` probes the Azure
Instance Metadata Service, which doesn't exist on your laptop. It falls back to `az login`.
Ignore it.

---

## Deploy

```bash
azd deploy                                    # ~2m3s
azd ai agent invoke "What is Microsoft Foundry?"
azd ai agent show --output json
```

`show` reveals what Foundry actually stored:

```json
{
  "id": "hello-world:1",
  "definition": {
    "code_configuration": { "entry_point": ["python","main.py"], "runtime": "python_3_13" },
    "cpu": "0.5", "memory": "1Gi"
  },
  "status": "active",
  "instance_identity": { "principal_id": "2debe4d4-…" }
}
```

Note `:1` — redeploying the same `name:` makes `:2`, not a second agent.

---

## Clean up

```bash
azd down --force --purge
```

---

## What changed vs. an ordinary script

| Ordinary script | This agent |
|---|---|
| you call the model | the **host** calls your agent |
| you manage history | `store: False` — host manages it |
| keys in `.env` | managed identity, no keys |
| runs where you run it | runs in Foundry, versioned, traced |

---

👉 Next: [02 · Tools](../02-tools/) — give the agent something to *do*.
