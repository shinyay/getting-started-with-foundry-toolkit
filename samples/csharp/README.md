# 🔷 C# samples

The same ladder as [Python](../python/), using **Microsoft Agent Framework for .NET**.

| # | Sample | New idea | Upstream |
|---|---|---|---|
| [01](01-hello-world/) | Hello World | agent = HTTP server | `csharp/…/agent-framework/hello-world` |
| [02](02-tools/) | Local tools | `AIFunctionFactory.Create` | `csharp/…/agent-framework/local-tools` |
| [03](03-mcp-toolbox/) | MCP tools | client-side **and** server-side MCP | `csharp/…/agent-framework/mcp-tools` |
| [04](../python/04-eval/) | Evaluation | language-agnostic — see the Python page | — |

`azd ai agent eval` is CLI-level and identical for both languages, so step 04 is not duplicated.

---

## Python ↔ C# differences

| | Python | C# |
|---|---|---|
| Runtime token | `python_3_13` | **`dotnet_10`** |
| Entry point | `main.py` | **`<assembly>.dll`** (e.g. `hello-world.dll`) |
| Dependencies | `requirements.txt` | `.csproj` `PackageReference` |
| Client | `FoundryChatClient` | `AIProjectClient(...).AsAIAgent(...)` |
| Host | `ResponsesHostServer(agent).run()` | `AgentHost.CreateBuilder(args)` + `AddFoundryResponses` |
| Tool declaration | `@tool` decorator | `AIFunctionFactory.Create(fn, name, description)` |
| `.env` loading | `load_dotenv()` | `Env.NoClobber().TraversePath().Load()` |
| `language:` in `azure.yaml` | `python` | `csharp` |
| Upstream samples | 21 | 13 |

Everything else — `azure.yaml`, `azd` commands, ports, protocols, eval — is identical.

---

## Packages

```xml
<PackageReference Include="Microsoft.Agents.AI.Foundry.Hosting" Version="1.15.0-preview.260722.1" />
<PackageReference Include="Azure.AI.Projects" Version="2.1.0-beta.4" />
<PackageReference Include="DotNetEnv" Version="3.1.1" />
```

Plus, for MCP:

```xml
<PackageReference Include="ModelContextProtocol" Version="1.2.0" />
<PackageReference Include="Microsoft.Extensions.AI.OpenAI" Version="10.6.0" />
```

Target framework is `net10.0`, SDK `Microsoft.NET.Sdk.Web`. The samples set
`<NoWarn>$(NoWarn);NU1903;NU1605</NoWarn>` to silence audit/downgrade noise from preview feeds.

---

## `entryPoint` is the DLL, not the project

```yaml
codeConfiguration:
    runtime: dotnet_10
    entryPoint: hello-world.dll        # ← assembly name from <AssemblyName>/.csproj filename
```

A common mistake is writing `Program.cs` or the project name. It must be the built **assembly**.

---

## What `AgentHost.CreateBuilder()` gives you

Per the sample's own comments, it auto-configures:

- Kestrel on port **8088** (or `PORT`)
- `GET /readiness` health probe
- OpenTelemetry traces and metrics
- the `x-platform-server` response header

So `Program.cs` stays about the agent, not the plumbing.

---

## Getting started

```bash
mkdir my-agent && cd my-agent
azd ai agent init -m "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/csharp/hosted-agents/agent-framework/hello-world/azure.yaml"
```

Then follow the [CLI golden path](../../docs/guide-cli/README.md).

---

## Full C# catalog

13 samples including Text Search RAG, Translation Workflow, Human-in-the-Loop, background and
note-taking agents → [sample catalog](../../docs/reference/sample-catalog.md#-c-13).
