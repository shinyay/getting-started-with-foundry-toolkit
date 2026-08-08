# 03 · MCP & Foundry Toolbox

> **New idea:** tools you didn't write, running outside your container.

Steps 01–02 kept everything inside your process. Real agents borrow capability — search, code
execution, business systems — from **external tool servers**.

---

## Two ways to get external tools

| | **Foundry Toolbox** *(this sample)* | **MCP server** |
|---|---|---|
| What | a Foundry-managed bundle of tools | any Model Context Protocol server |
| Discovery | resolved from the project at runtime | you supply the endpoint/command |
| Auth | your agent's managed identity | you configure it |
| Transport | Foundry | `stdio` or HTTP+SSE |
| Good for | Azure-native capability | anything in the MCP ecosystem |

> [!WARNING]
> **Broken catalog entry (verified 2026-08-08).** `azd ai agent sample list --language python`
> advertises a Python MCP sample at
> `…/agent-framework/responses/03-mcp/azure.yaml`, but that path **404s** — the directory no
> longer exists upstream. The C# equivalent
> (`csharp/hosted-agents/agent-framework/mcp-tools/azure.yaml`) **is** live.
> This sample therefore uses **Foundry Toolbox** (`04-foundry-toolbox`, verified 200) for the
> Python track, and points the C# track at the working MCP sample.

---

## Scaffold it

```bash
mkdir 03-toolbox && cd 03-toolbox
azd ai agent init -m "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/responses/04-foundry-toolbox/azure.yaml"
```

---

## The code

Note it becomes **async** — external tools need a connection lifecycle.

```python
import asyncio
from agent_framework_foundry_hosting import FoundryToolbox, ResponsesHostServer

async def main():
    credential = DefaultAzureCredential()

    # Resolves its endpoint from TOOLBOX_ENDPOINT, or from
    # FOUNDRY_PROJECT_ENDPOINT + TOOLBOX_NAME. Authenticates every request with
    # the credential and forwards the platform per-request call-id.
    toolbox = FoundryToolbox(credential)

    client = FoundryChatClient(
        project_endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"],
        model=os.environ["AZURE_AI_MODEL_DEPLOYMENT_NAME"],
        credential=credential,
    )

    agent = Agent(
        client=client,
        instructions="You are a friendly assistant. Keep your answers brief.",
        tools=toolbox,                   # ← a whole toolbox, not a list of functions
        default_options={"store": False},
    )

    await ResponsesHostServer(agent).run_async()

if __name__ == "__main__":
    asyncio.run(main())
```

| Change vs 02 | Why |
|---|---|
| `async def main()` + `run_async()` | tool connections are I/O-bound |
| `tools=toolbox` (not a list) | the toolbox enumerates its own tools |
| shared `credential` | one identity for both the model and the tools |

The host **connects the toolbox on first use and closes it at shutdown** — you don't manage
the lifecycle.

---

## Configuration

`azure.yaml` adds one variable:

```yaml
environmentVariables:
    - name: AZURE_AI_MODEL_DEPLOYMENT_NAME
      value: ${AZURE_AI_MODEL_DEPLOYMENT_NAME}
    - name: TOOLBOX_ENDPOINT
      value: ${TOOLBOX_ENDPOINT}
```

```bash
azd env set TOOLBOX_ENDPOINT https://<your-toolbox-endpoint>
```

If you omit it, `FoundryToolbox` falls back to `FOUNDRY_PROJECT_ENDPOINT` + `TOOLBOX_NAME`.

Check your wiring before running:

```bash
azd ai agent doctor
```

`doctor` has a dedicated check — *"Manifest toolboxes have endpoint env vars set"* — which
reports **skipped** when no toolbox is declared, and **failed** when one is declared but
unconfigured.

---

## Connecting an MCP server instead

MCP servers are reachable two ways:

| Transport | Use when |
|---|---|
| `stdio` | the server is a local command (`npx …`, `uv run …`) |
| HTTP + SSE | the server is remote |

In **VS Code Agent Builder** this is a first-class UI: browse a featured MCP catalog, connect an
existing server, scaffold a new one in Python/TypeScript (then `F5` → *Debug in Agent Builder*),
or **reuse tools already installed in VS Code** (creates a `VSCode Tools` MCP entry).
See [guide-gui §4](../../../docs/guide-gui/README.md#4-agent-builder--prompt-agents).

MCP servers need Node (`npm install -g npx`) and/or Python (`uv` recommended).

---

## Run & deploy

```bash
azd provision
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
azd ai agent run --no-client
azd ai agent invoke --local "Search for the latest Microsoft Foundry release notes."
azd deploy
```

> [!IMPORTANT]
> External tools need **permissions**. The deployed agent uses its own managed identity
> (`instance_identity.principal_id` from `azd ai agent show --output json`), *not* your user
> account. If a tool works locally but fails once deployed, grant that principal the role it
> needs.

---

## Clean up

```bash
azd down --force --purge
```

---

👉 Next: [04 · Evaluation](../04-eval/) — is it actually any good?
