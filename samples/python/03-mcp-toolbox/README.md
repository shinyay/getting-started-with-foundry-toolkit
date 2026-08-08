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

But where does that endpoint come from? **You have to create the toolbox first** — it is a
real Azure Foundry resource, not something `azd provision` makes for you. That is what the
vendored `src/toolbox-agent/toolbox.yaml` is for.

### Create the toolbox

`toolbox.yaml` declares the tools that sit behind one endpoint: `web_search`,
`code_interpreter`, and four MCP servers. Create it, then read back its endpoint:

```bash
cd src/toolbox-agent
azd ai toolbox create agent-tools --from-file ./toolbox.yaml
azd ai toolbox show agent-tools                 # prints the computed MCP endpoint
azd env set TOOLBOX_ENDPOINT "<endpoint from show>"
```

> [!IMPORTANT]
> `azd ai toolbox` lives in a **separate extension** (`azure.ai.toolboxes`) from
> `azd ai agent` (`azure.ai.agents`). The first invocation prompts to install it:
>
> ```text
> Command 'ai toolbox' was not found, but there's an available extension that provides it
> Id: azure.ai.toolboxes   Name: Foundry Toolboxes (Beta)
> ```
>
> See the [ecosystem map](../../../docs/reference/ecosystem.md) — `azd ai` has **four**
> namespaces (`agent`, `inspector`, `project`, `toolbox`).

> [!NOTE]
> Toolbox versions are **immutable**. Adding or removing a connection creates a *new* version
> and never changes which version is the default; promote one with
> `azd ai toolbox publish <toolbox> <version>`. `azd ai toolbox versions` inspects them.

Three of the six tools in `toolbox.yaml` reference a **project connection** by name
(`ghmcppat`, `langmcpconn`, `foundrymcpconn`). List what your project actually has:

```bash
./scripts/list-foundry-connectors.sh        # or .ps1 on Windows
```

> [!IMPORTANT]
> **`create` does not validate those connections — verified live on 2026-08-08.** Running it
> against a project with none of them still succeeded:
>
> ```text
> toolbox create: resolved project endpoint https://cog-<token>.services.ai.azure.com/api/projects/rdpy (source=azdEnv)
> Created toolbox agent-tools at version 1.
> Endpoint: https://cog-<token>.services.ai.azure.com/api/projects/rdpy/toolboxes/agent-tools/versions/1/mcp?api-version=v1
> exit_code=0
> ```
>
> So **exit code 0 from `toolbox create` is not proof your tools work.** Creation *packages a
> definition*; the connections are resolved later, when an agent actually calls a tool. A broken
> reference fails at **invoke time**, not at create time — which is a much worse place to find it.
>
> Two practical consequences:
> - Don't gate CI on `toolbox create` succeeding. It nearly always will.
> - Verify wiring with `azd ai agent doctor` (it has a dedicated toolbox-endpoint check) and by
>   actually invoking the agent.

Note the endpoint shape — `/toolboxes/<name>/versions/<n>/mcp` — it is **version-pinned**. That
is the immutability above made concrete: consumers address a specific version, so publishing a
new one cannot silently change behaviour underneath a running agent.

For a first run, trim `toolbox.yaml` to the tools that need no connection at all —
`web_search`, `code_interpreter`, and the no-auth `noauth_mcp` server. That is enough to see the
concept work end to end, and it removes the failure mode above entirely.

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

---

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
