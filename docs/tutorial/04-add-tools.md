# 🧰 Lab 04 — Add local tools

> ⏱️ **30 min** · 📋 **Requires:** [Lab 03](03-deploy.md) · 💰 **~$0.02** · ☁️ **Creates 2 Azure resources**

Turn the basic agent into an agent that can call your code as a tool.

## What you'll learn

- Expose a Python function or C# method as an agent tool.
- Write tool descriptions the model can use to decide when to call the tool.
- Compare prompts that need a tool with prompts the model can answer directly.
- Deploy the same local tool code to Foundry.

## Steps

### 1. Scaffold the tools sample

Choose one language.

Python:

```bash
mkdir 02-tools && cd 02-tools
azd ai agent init -m "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/responses/02-tools/azure.yaml"
```

C#:

```bash
mkdir 02-tools && cd 02-tools
azd ai agent init -m "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/csharp/hosted-agents/agent-framework/local-tools/azure.yaml"
```

The local samples in this repo are here if you want to inspect them before scaffolding:
[Python](../../samples/python/02-tools/) and [C#](../../samples/csharp/02-tools/).

> [!NOTE]
> **Both tracks have now been walked against live Azure — Python three times, C# once, on
> 2026-08-14.** The C# walk covered §§ 1–4, 6 and 7: it provisioned, ran locally, deployed,
> invoked and tore down successfully, and what it found is folded in below, including **two
> failures that happen only in C#**. It did **not** cover § 5, whose C# fragment is still
> illustrative. Blocks are labelled with the track they were captured from; where the same
> bytes came out of both, the summary says so. §§ 3, 6 and 7 are language-agnostic — § 3's
> `doctor` output came out **byte-identical on both tracks**.

> [!IMPORTANT]
> **`init` nests a folder named after the sample, and every later command must run inside it.**
> The Python command above produced
> `02-tools/agent-framework-agent-with-local-tools-responses/` — not `02-tools/`. The C# one
> produced `02-tools/local-tools/`. **The two names come from different places**: the Python
> folder is named after the sample directory upstream, the C# folder after the `name:` field
> in the sample's own `azure.yaml`. Do not derive one from the other — read the
> `Copying template code from local path to:` line in your own output, then:
>
> ```bash
> cd <the folder that line names>
> ls azure.yaml
> ```
>
> Skip this and § 3 fails with `ERROR: no project exists; to create a new project, run
> 'azd init'` — verified by running `azd provision` from `02-tools/`. Same behaviour as
> [Lab 02 § 2](02-first-agent.md#2-init--scaffold), which explains it in full.

Unlike Lab 02, this command has no `--no-prompt`, so `init` **asks five questions** before it
finishes. The first one has no equivalent anywhere in Lab 02:

<details>
<summary>✅ Verified output — the interactive tail of <code>init</code>, 2026-08-13 (subscription name and ID elided; the list bodies each question scrolls through are omitted)</summary>

```text
Installing required extensions...
  (-) Skipped: Installing azure.ai.agents extension (version 1.0.0-beta.9 already installed)
? Select a Foundry project to host your agent and any models or tools it uses.: Create a new Foundry project
Select an Azure subscription to provision your agent and Foundry project resources.
? Select a tenant:  3. All tenants
? Select subscription: <your-subscription-name> (<your-subscription-id>)
Select an Azure location. This determines which models are available and where your Foundry project resources will be deployed.
? Select location: (US) East US 2 (eastus2)

Model deployment 'gpt-5.4-mini' is defined in the azure.yaml:
  Model: gpt-5.4-mini (OpenAI), version 2026-03-17
  SKU: GlobalStandard, capacity 10

? How would you like to proceed?: Deploy as specified in azure.yaml

Adopted the sample's azure.yaml as the project manifest at azure.yaml.

Next:
  cd agent-framework-agent-with-local-tools-responses
  enter your new project folder

  azd provision
  set up your Foundry project, models, and connections

  azd deploy
  when ready to deploy to Azure
```

</details>

Answer **Create a new Foundry project** to follow this lab; **Use an existing Foundry project**
reuses the one Lab 03 left behind, if you kept it. Answer **Deploy as specified in azure.yaml**
to the fifth; the other two options are **Choose a different model** and **Skip this model
entirely (remove from azure.yaml)**.

> [!IMPORTANT]
> **The fifth question sets `AZURE_AI_MODEL_DEPLOYMENT_NAME` for you**, which is why § 3 below
> does not ask you to. Lab 02 has to set it by hand only because `--no-prompt` skips this
> question entirely. Same toolchain, opposite advice — the flag is the reason.

> [!WARNING]
> **The subscription picker can offer you one that Azure then refuses.** Choosing
> `MCAPS-…-71560-2024-…` produced `RESPONSE 404 / SubscriptionNotFound` on the very next step;
> the account's usable subscription was the near-identically named `MCAPS-…-118084-2025-…`.
> azd's own `Suggestion:` — set `AZURE_LOCATION` — does not recover it, because the wrong
> subscription is already written to the environment. `3. All tenants` is not itself the
> hazard: a second run picked `All tenants` and reached the right subscription without
> incident. **Read the whole name, not the prefix.** Why a listed subscription is unusable was
> not established. If you are already stuck, see [*If that didn't work*](#-if-that-didnt-work).

The `Next:` block is azd telling you what this page's § 3 assumes: **`cd` into the folder
`init` created.** `init` also writes a random 8-character salt, `AZD_RESOURCE_TOKEN_SALT`, and
derives the resource-group name from it **before contacting Azure at all** — see
[Lab 02 § 5](02-first-agent.md#5-provision--create-azure-resources).

> [!TIP]
> **To tell a finished `init` from an interrupted one, look for
> `AZURE_AI_MODEL_DEPLOYMENT_NAME` in `azd env get-values`** — only the fifth question writes
> it. Do not count the values: a run of these five questions leaves 11, an interrupted one 5,
> and an `init` that completed *without prompting* — azd skips the questions when it decides
> the terminal is not interactive — leaves 6, with a different set of keys again.

### 2. Inspect the tool definition

Python exposes a normal function with `@tool`:

```python
from agent_framework import Agent, tool
from pydantic import Field
from typing_extensions import Annotated


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

C# passes the method, tool name and description explicitly:

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

static string GetAvailableHotels(
    string? checkInDate = null,
    string? checkOutDate = null,
    int? guests = null)
{
    // …returns hotel data as a string
}
```

> [!CAUTION]
> **The C# sample looks like it defaults to `gpt-4o`. It does not.** `Program.cs` reads
> `Environment.GetEnvironmentVariable("AZURE_AI_MODEL_DEPLOYMENT_NAME") ?? "gpt-4o"`, but
> `azure.yaml` declares that variable as `value: ${AZURE_AI_MODEL_DEPLOYMENT_NAME}`, and azd
> expands an unset variable to an **empty string** — which is not `null`, so `??` never fires.
> Verified on 2026-08-14: the process dies with
> `System.ArgumentException: Argument is whitespace (Parameter 'model')`. The fallback in the
> source is unreachable under azd, so § 3's check applies to C# too.

How the model learns what the tool does:

| Source | Python | C# |
|---|---|---|
| Tool name | function name | explicit string |
| Tool description | docstring | explicit description |
| Parameters | type annotations + `Field(description=…)` | method signature |
| Result | return value | return value |

> [!TIP]
> The docstring, description and parameter descriptions are **prompt engineering**, not just
> comments. Vague descriptions are the most common cause of "the agent won't use my tool".

### 3. Provision and set the model name

From inside the folder `init` created:

```bash
azd ai agent doctor
azd provision
```

`doctor` before `provision` gives a state that is **not** the one Lab 02 shows, because Lab 02
runs it with `--local-only`:

<details>
<summary>✅ Verified output — scaffolded, subscription and location set, before <code>provision</code>, 2026-08-13 (Python) and byte-identical on the C# walk, 2026-08-14 — all 30 lines</summary>

```text
azd ai agent doctor

Local
   (✓) azd extension reachable
   (✓) azure.yaml present and parseable
   (✓) azd environment selected
   (✓) agent service in azure.yaml
   (x) FOUNDRY_PROJECT_ENDPOINT set
       FOUNDRY_PROJECT_ENDPOINT is not set in the current azd environment
       fix: Run `azd provision` to create the Foundry project, or `azd env set FOUNDRY_PROJECT_ENDPOINT <https://...>` to point at an existing one.
   (✓) agent definition valid (per service)
   (✓) manual env vars set
   (-) Manifest toolboxes have endpoint env vars set -- skipped (no toolbox resources declared in any service's agent.manifest.yaml.)

Authentication
   (✓) authentication

Remote
   (-) Foundry project endpoint reachable -- skipped (FOUNDRY_PROJECT_ENDPOINT is not set (see check `local.project-endpoint-set`).)
   (-) Developer has required role on Foundry project -- skipped (AZURE_AI_PROJECT_ID is not set in the current azd environment.)
   (-) Hosted agents are active -- skipped (Foundry endpoint did not respond (see check `remote.foundry-endpoint`).)
   (-) Manifest connections exist on Foundry project -- skipped (Foundry project endpoint unreachable (see check `remote.foundry-endpoint`).)

7 passed, 1 failed, 5 skipped

To fix:

  Review the fix: notes above for each failed check.

Then re-run `azd ai agent doctor` to verify.
```

</details>

> [!NOTE]
> **Look at the third Remote skip.** `Developer has required role on Foundry project` is skipped
> because `AZURE_AI_PROJECT_ID is not set` — a missing prerequisite, not a cascade from the
> failure above it. The other three skips *do* cascade. See
> [Lab 01 § 7](01-setup.md#7-verify-everything).

`provision` took **1 min 21 s** here and produced the same output shape as
[Lab 02 § 5](02-first-agent.md#5-provision--create-azure-resources) — nothing new to read. A
second run measured 1 min 16 s. Unlike `init` and `invoke`, `provision` prints **no** `Next:`
block. Two things are worth checking against your own run:

```bash
azd env get-values | grep -E 'FOUNDRY_PROJECT_ENDPOINT|AZURE_RESOURCE_GROUP'
```

```text
AZURE_RESOURCE_GROUP="rg-agent-framework-agent-with-local-tools-responses-dev-<salt>"
FOUNDRY_PROJECT_ENDPOINT="https://cog-<random>.services.ai.azure.com/api/projects/agent-framework-agent-with-local"
```

The environment name here is 51 characters; the project is `agent-framework-agent-with-local`
— its first **32**. That was predicted before the run and matched exactly on two separate
runs, which is the third confirmation of the naming rule in
[Lab 02 § 5](02-first-agent.md#5-provision--create-azure-resources). The C# walk tested the
other side of the rule for the first time: its environment is `local-tools-dev`, **15**
characters, and the project came back as `local-tools-dev` in full. The rule truncates; it
does not pad, hash or rename.

**Whether you must set `AZURE_AI_MODEL_DEPLOYMENT_NAME` depends on how `init` went.** This
lab does not pass `--no-prompt`, so § 1's fifth question normally sets it and `provision`
never does. But if `init` could not prompt — it prints
`Continuing because --no-prompt was specified` and lists the values it skipped — then nothing
set it and § 4 will fail. Check rather than assume:

```bash
azd env get-values | grep MODEL_DEPLOYMENT
```

If that prints nothing, set it before going on:

```bash
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
```

> [!WARNING]
> **`doctor` will not catch it if that value is missing.** Run with the variable unset,
> `doctor` still reports `(✓) manual env vars set` and the same `7 passed, 1 failed,
> 5 skipped` — the failure surfaces later, in § 4, when the agent process starts. The `grep`
> above is the check that works.

### 4. Run locally

```bash
azd ai agent run --no-client
```

> [!CAUTION]
> **C# only — start the server like this instead if you are not on an Azure VM:**
>
> ```bash
> AZURE_TOKEN_CREDENTIALS=dev azd ai agent run --no-client
> ```
>
> .NET's `DefaultAzureCredential` tries the instance metadata service first and treats its
> **timeout** as a fatal error rather than "this credential is unavailable", so on a laptop or
> under WSL it stops there instead of falling through to your `az login`. Verified on
> 2026-08-14: the first invoke hung for 100 s, the second failed outright with
> `ManagedIdentityCredential authentication failed`. Python's credential chain does fall
> through, which is why only this track needs the variable — and only locally. The deployed
> agent has a real managed identity and is unaffected.

In a second terminal, ask for something that needs the tool.

Python:

```bash
azd ai agent invoke --local "What's the weather in Tokyo?"
```

C#:

```bash
azd ai agent invoke --local "I need a hotel in Seattle for 2 guests next weekend."
```

<details>
<summary>✅ Verified output — Python, 2026-08-13 (weather is random; wording varies)</summary>

```text
Target:       localhost:8088 (local)
Message:      "What's the weather in Tokyo?"
Session:      13c126ec-06a4-44e2-941d-ee9f6a36a77a
Conversation: 547c5785-1e44-4462-b347-34df0304c1f3

[local] Tokyo is cloudy with a high of 14°C.

Server responded in 11.863s (first byte: 11.863s)

Next:
  azd deploy
  deploy the agent to Azure

  azd ai agent monitor --follow
  view logs after deploying
```

</details>

C#, same question, on the walk that verified this track:

<details>
<summary>✅ Verified output — C#, 2026-08-14 (the remaining four hotels and the trailing <code>Next:</code> block are elided)</summary>

```text
Target:       localhost:8088 (local)
Message:      "I need a hotel in Seattle for 2 guests next weekend."
Session:      61e17e6a-2b2a-47aa-a066-88aa08383204
Conversation: 3fe03f8c-3cc9-43e4-aa6c-a7e472dc0d11

[local] Here are some great Seattle hotel options for **2 guests next weekend**:

### Top options
- **The Grand Seattle** — Downtown Seattle  
  **$289/night** | **4.7/5** | 12 rooms available  
  Amenities: Free WiFi, Pool, Spa, Restaurant, Fitness Center
…
Server responded in 9.020s (first byte: 9.016s)
```

</details>

> [!IMPORTANT]
> **Nothing in that output says a tool ran.** There is no tool line, no marker, no timing
> breakdown — a tool-backed answer and an invented answer are the same shape. The only
> client-side hint is latency: 11.863 s here against 1.848 s for a question that needed no tool.
> Latency is a hint, not proof.

The proof is in the **first** terminal, the one running `azd ai agent run`:

<details>
<summary>✅ Verified output — the tool call as it appears in the <code>run</code> log, 2026-08-13 (the OpenTelemetry span dumps that sit between these lines are omitted — see the note below)</summary>

```text
2026-08-13 21:49:18,656 INFO agent_framework: {'role': 'assistant', 'parts': [{'type': 'tool_call', 'id': 'call_qZlqhTYT5Y0uWBIHSaHHcNsd', 'name': 'get_weather', 'arguments': '{"location":"Tokyo"}'}]}
2026-08-13 21:49:18,658 INFO agent_framework: Function name: get_weather
2026-08-13 21:49:18,659 INFO agent_framework: Function get_weather succeeded.
2026-08-13 21:49:18,660 INFO agent_framework: Function duration: 0.001342s
2026-08-13 21:49:18,670 INFO agent_framework: {'role': 'tool', 'parts': [{'type': 'tool_call_response', 'id': 'call_qZlqhTYT5Y0uWBIHSaHHcNsd', 'response': 'The weather in Tokyo is cloudy with a high of 14°C.'}]}
```

</details>

`Function duration: 0.001342s`. The tool is **not** what makes the request slow; the model
round-trips are. That is worth remembering before you optimise a tool.

> [!NOTE]
> **Those five lines are not adjacent in the log.** Between them sit two OpenTelemetry span
> dumps — `chat gpt-5.4-mini` (47 lines) and `execute_tool get_weather` (39 lines). The
> millisecond timestamps are the thread to pull: `,656` is the model asking, `,658`–`,660`
> is your Python running, `,670` is the answer going back.

The same log carries the exact JSON the model was shown, which is § 2's table as data rather
than as a claim:

<details>
<summary>✅ Verified output — the <code>gen_ai.tool.definitions</code> span attribute, 2026-08-13</summary>

```text
        "gen_ai.tool.definitions": "[{\"type\": \"function\", \"name\": \"get_weather\", \"description\": \"Get the weather for a given location.\", \"parameters\": {\"properties\": {\"location\": {\"description\": \"The location to get the weather for.\", \"title\": \"Location\", \"type\": \"string\"}}, \"required\": [\"location\"], \"title\": \"get_weather_input\", \"type\": \"object\"}}]",
```

</details>

It arrives doubly escaped because an OpenTelemetry attribute holds a *string*, so the whole
schema is JSON inside JSON. Decoded — this block is derived from the one above, not a separate
capture — it reads:

```json
[{"type": "function", "name": "get_weather", "description": "Get the weather for a given location.", "parameters": {"properties": {"location": {"description": "The location to get the weather for.", "title": "Location", "type": "string"}}, "required": ["location"], "title": "get_weather_input", "type": "object"}}]
```

Your docstring is the `description`. Your `Field(description=…)` is the parameter
`description`. Nothing else about your function reaches the model.

> [!NOTE]
> **This visibility is Python-only.** The C# agent server logs nothing equivalent: on the
> 2026-08-14 walk its `run` log contained no `gen_ai.tool.definitions` attribute and never
> named `GetAvailableHotels`, while the Python log emitted the attribute three times. The
> arguments to `AIFunctionFactory.Create` do reach the model — the tool was called and
> answered — but in C# you cannot read back what the model was shown. Where the
> troubleshooting table below says "check `gen_ai.tool.definitions`", that step has no C#
> counterpart; compare behaviour instead.

Then ask for something that should not use the tool:

```bash
azd ai agent invoke --local "What is 2+2?"
```

> [!WARNING]
> **Consecutive invokes reuse the same session — the second question is not asked in
> isolation.** Both invokes above printed
> `Session: 13c126ec-…` and `Conversation: 547c5785-…`, and the server log shows the whole
> first exchange replayed as input to the second. `azd ai agent invoke --help` states it:
> *"Sessions are persisted per-agent — consecutive invokes reuse the same session
> automatically."* To ask a genuinely fresh question:
>
> ```bash
> azd ai agent invoke --local --new-session --new-conversation "What is 2+2?"
> ```
>
> It also costs you: every invoke re-sends the accumulated history.

Across the whole local `run` — three invokes, seven model turns — the log recorded
`finish_reasons: ["tool_calls"]` **once** and `["stop"]` six times, and
`Function name: get_weather` appeared exactly once. The arithmetic question never reached the
tool, with or without a clean session.

Run without `--no-client` to watch the call/result timeline in the Agent Inspector.
The Inspector view was not verified for this lab.

### 5. Add your own tool

Python — put the function beside `get_weather`, above `main()`, and add it to the `tools=[…]`
list already there:

```python
@tool(approval_mode="never_require")
def get_stock_price(
    symbol: Annotated[str, Field(description="Ticker symbol, e.g. MSFT.")],
) -> str:
    """Look up the latest closing price for a stock ticker."""
    return f"{symbol} closed at $432.10."

agent = Agent(client=client, instructions="…", tools=[get_weather, get_stock_price], …)
```

C#:

```csharp
static string GetRoomRate(string hotelName, int nights)
    => $"{hotelName}: $210/night × {nights} nights = ${210 * nights}.";

tools:
[
    AIFunctionFactory.Create(GetAvailableHotels, "GetAvailableHotels", "…"),
    AIFunctionFactory.Create(GetRoomRate, "GetRoomRate",
        "Calculates the total room rate for a hotel and number of nights.")
]
```

> [!IMPORTANT]
> **Restart `azd ai agent run` — it does not reload.** `azd ai agent run --help` describes it
> as starting the server *"in the foreground. Press Ctrl+C to stop"*, and offers no watch or
> reload flag. Edit while it is running and you will keep testing the old code.

> [!NOTE]
> **The C# fragment above is illustrative — it has not been run.** The 2026-08-14 C# walk
> covered §§ 1–4, 6 and 7, but stopped short of adding a second tool, so unlike the Python
> addition this one is written from the sample's shape rather than from a run. It is a
> fragment, not a compilable file: `tools:` is the object-initialiser property, not a
> statement. The Python path below is the one with captured output behind it.

Then ask for the new tool. The Python addition above was applied verbatim and run:

```bash
azd ai agent invoke --local --new-session --new-conversation "What did MSFT close at?"
```

<details>
<summary>✅ Verified output — Python, after adding <code>get_stock_price</code>, 2026-08-14 (the <code>Next:</code> block that follows is elided)</summary>

```text
Target:       localhost:8088 (local)
Message:      "What did MSFT close at?"
Session:      d47a5f5f-8e5f-4ad4-bcc3-971f1b5677ce
Conversation: 6184936d-3921-4af5-aa8c-0aabfd2074a9

[local] MSFT closed at **$432.10**.

Server responded in 5.085s (first byte: 5.085s)
```

</details>

The `run` terminal shows the same five-line shape as § 4, with the new name, and
`gen_ai.tool.definitions` now carries **both** tools — which is the check that your function
actually reached the model:

<details>
<summary>✅ Verified output — the <code>run</code> log after adding the second tool, 2026-08-14 (the span dumps between these lines are omitted, as in § 4)</summary>

```text
2026-08-14 07:15:00,572 INFO agent_framework: {'role': 'assistant', 'parts': [{'type': 'tool_call', 'id': 'call_byGM4AOeKBs8rKOY9CpGqo6f', 'name': 'get_stock_price', 'arguments': '{"symbol":"MSFT"}'}]}
2026-08-14 07:15:00,574 INFO agent_framework: Function name: get_stock_price
2026-08-14 07:15:00,575 INFO agent_framework: Function get_stock_price succeeded.
```

</details>

Rules that matter:

1. **Return a string** or something trivially serializable — it goes back into the prompt.
2. **Raise or throw informative exceptions.** The model sees the error and can retry or explain.
3. **Keep tools fast.** Every call is inside the user's latency budget.
4. Use human approval for anything that writes, spends money or sends messages.

### 6. Deploy it

Stop the local `azd ai agent run` first — Ctrl+C in that terminal. Nothing below needs it, and
leaving it running holds port 8088.

```bash
azd deploy
azd ai agent invoke "What's the weather in Osaka?"
azd ai agent monitor
```

Tools run **inside your container** — same code, same process. Nothing is registered with
Foundry, so there is no extra tool deployment step. `azd ai agent show --output json` is the
evidence: the deployed definition has **no `tools` key anywhere in it**. Check it on your own
deployment with

```bash
azd ai agent show <your-agent-name> --output json | grep -c '"tools"'
```

which returns `0`. What the `definition` object does carry is only how to build and run your
container — this list is derived from that JSON, not printed by azd in this form:

```text
code_configuration, container_protocol_versions, cpu, environment_variables, kind, memory, protocol_versions
```

Deploy took **2 min 22 s**, **2 min 6 s** and **2 min 12 s** across three runs. Redeploying
unchanged code afterwards took **12 s**, **14 s** and **15 s**, and left `Version` at `1`.

> [!WARNING]
> **A redeploy immediately after the first deploy can fail with `Project not found` — and the
> project is fine.** One of three runs produced this 9 s after starting, against a project that
> `az resource list`, `azd ai agent show` and a working remote `invoke` all confirmed existed:
>
> ```text
> RESPONSE 404: 404 Not Found
> ERROR CODE: NotFound
> {
>   "error": {
>     "code": "NotFound",
>     "message": "Project not found"
>   }
> }
> ```
>
> Re-running `azd deploy` unchanged succeeded in 14 s. **Retry before you diagnose.** The
> other two runs redeployed first time, so this is intermittent and its cause was not
> established.

The remote invoke prints two things the local one does not — a server-assigned session and a
`Trace ID`:

<details>
<summary>✅ Verified output — remote invoke against the deployed agent, 2026-08-13</summary>

```text
Agent:        agent-framework-agent-with-local-tools-responses (remote)
Message:      "What's the weather in Osaka?"
Session:      (new -- server will assign)
Conversation: conv_56b6651cf227725400QjrYnYbX6n2xE2m5hyXchhGCsalU5L6m

Session:      842b965678e9e133aea23fc53ca1a35e0256d1fed1f79ffd17d1725f9461671 (assigned by server)
Trace ID:     06398239f93b448e3e5043c212a09aff
[agent-framework-agent-with-local-tools-responses] Osaka is cloudy and about 13°C today.

Server responded in 22.053s (first byte: 7.722s)

Next:
  azd ai agent show agent-framework-agent-with-local-tools-responses
  confirm the deployed agent is healthy

  azd ai agent monitor --follow
  stream live logs from the agent
```

</details>

The tool call is invisible here too. Remotely there is no `run` terminal to read, so the
evidence moves to `monitor` — and the `Trace ID` above is how you find it:

<details>
<summary>✅ Verified output — the same tool call, in <code>azd ai agent monitor</code>, 2026-08-13</summary>

```text
22:00:19  stderr   2026-08-13 13:00:19,961 INFO agent_framework: {'role': 'assistant', 'parts': [{'type': 'tool_call', 'id': 'call_kIrF5aUqa0S3HCZsFbb8Bi05', 'name': 'get_weather', 'arguments': '{"location":"Osaka"}'}]}
22:00:19  stderr   2026-08-13 13:00:19,962 INFO agent_framework: Function name: get_weather
22:00:19  stderr   2026-08-13 13:00:19,962 INFO agent_framework: Function get_weather succeeded.
22:00:19  stderr   2026-08-13 13:00:19,962 INFO agent_framework: Function duration: 0.000173s
22:00:19  stderr   2026-08-13 13:00:19,966 INFO agent_framework: {'role': 'tool', 'parts': [{'type': 'tool_call_response', 'id': 'call_kIrF5aUqa0S3HCZsFbb8Bi05', 'response': 'The weather in Osaka is cloudy with a high of 13°C.'}]}
22:00:26  stderr   2026-08-13 13:00:26,063 INFO azure.ai.agentserver: Inbound POST /responses completed with status 200 in 15158.4ms (x-request-id: 06398239f93b448e3e5043c212a09aff, x-ms-client-request-id: 86debe2f-c0ef-4789-bade-a25b75dbd26d, trace-id: 06398239f93b448e3e5043c212a09aff)
```

</details>

The `trace-id` on the last line is the `Trace ID` the client printed. That is the join.

> [!NOTE]
> **Every line carries two clocks and they are nine hours apart.** `22:00:19` is `monitor`
> stamping the line in your local timezone; `13:00:19` is the container logging in UTC. The
> offset is your own, and across midnight UTC the two even disagree on the **date** — a later
> run logged `07:20:23` beside a container stamp of `2026-08-13 22:20:23`. `monitor`'s stamp
> carries no date at all, so there is nothing on the line to warn you. Match on the
> `trace-id`, not on the time.

> [!NOTE]
> **Two numbers here do not agree, and both are correct.** The client reported 22.053 s; the
> container reported 15158.4 ms for the request it served. The ~7 s difference is the same
> unexplained gap measured in [Lab 03 § 3](03-deploy.md#3-invoke--call-the-deployed-agent).
> Part of the 7.722 s to first byte is a container cold start — `monitor` shows the process
> booting *after* the invoke was issued.

Add `-f` to follow the log live instead of dumping what has accumulated:

```bash
azd ai agent monitor -f
```

### 7. Clean up

```bash
azd down --force --purge
az group exists -n <your-resource-group>
az cognitiveservices account list-deleted --query "[].name" -o json
```

<details>
<summary>✅ Verified output — 3 min 51 s, 2026-08-13 (the repeated progress lines are collapsed to one each — see the note; resource names shortened to <code>rg-…</code> / <code>cog-…</code>)</summary>

```text
Deleting all resources and deployed code on Azure (azd down)
Local application code is not deleted when running 'azd down'.

Listing Cognitive Services accounts in rg-…
Deleting model deployment gpt-5.4-mini on cog-…
Deleting resource group rg-… (this can take several minutes)
Purging soft-deleted Cognitive Services account cog-…

SUCCESS: Your application was removed from Azure in 3 minutes 51 seconds.
```

</details>

<details>
<summary>✅ Verified output — <code>az group exists</code> after the purge, 2026-08-13</summary>

```text
false
```

</details>

`SUCCESS` is not confirmation — see
[Lab 03 § 6](03-deploy.md#6-down--destroy-everything-with-purge). Neither, on its own, is
`false`.

> [!CAUTION]
> **`azd down --force --purge` can delete the resource group and still leave the account
> soft-deleted.** A second run of this lab produced no `SUCCESS` line at all:
>
> ```text
> ERROR: deleting infrastructure: error deleting Azure resources: provisioning destroy failed: cognitive_account_purge: DELETE https://management.azure.com/subscriptions/<your-subscription-id>/providers/Microsoft.CognitiveServices/locations/eastus2/resourceGroups/rg-…/deletedAccounts/cog-…
> RESPONSE 409: 409 Conflict
> ERROR CODE: RequestConflict
> ```
>
> The message is *"the resource entity provisioning state is not terminal"* — `azd` reached
> the purge before Azure had finished deleting. At that moment `az group exists` returned
> **`false`** while `az cognitiveservices account list-deleted` still listed the account. The
> group check passes and the purge has failed. That is why the third command above exists.
>
> Recovery is to wait and purge by hand; it succeeded about three minutes later:
>
> ```bash
> az cognitiveservices account purge -n cog-<random> -g rg-<your-resource-group> -l eastus2
> ```
>
> A soft-deleted account holds its name and its quota. Leaving one behind is why a later
> `provision` can fail for reasons that look unrelated.

Both checks must agree — `false` **and** an empty list:

<details>
<summary>✅ Verified output — after the manual purge succeeded, 2026-08-13</summary>

```text
group exists: false
soft-deleted accounts: 0
[]
```

</details>

## ✅ Checkpoint

**Check this at the end of § 4, not here** — by the time you have run § 7 the Foundry project
is gone and the local agent cannot start. With `azd ai agent run --no-client` up in the first
terminal, this is what the second one should show:

```bash
azd ai agent invoke --local "What's the weather in Tokyo?"
```

<details>
<summary>✅ Verified output — Python, 2026-08-13 (weather is random, so wording and temperature vary)</summary>

```text
Target:       localhost:8088 (local)
Message:      "What's the weather in Tokyo?"
Session:      13c126ec-06a4-44e2-941d-ee9f6a36a77a
Conversation: 547c5785-1e44-4462-b347-34df0304c1f3

[local] Tokyo is cloudy with a high of 14°C.

Server responded in 11.863s (first byte: 11.863s)
```

</details>

The `Next:` block that follows it is terminal-only and elided here.

An answer alone is not the checkpoint — [§ 4](#4-run-locally) explains why a made-up answer
looks identical. The lab has worked only if the **first** terminal also logged
`Function name: get_weather`. And, as in [Lab 03](03-deploy.md#-checkpoint), it is finished
only once § 7's two teardown checks both agree.

If you see something else, jump to *If that didn't work* below.

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| `ERROR: no project exists; to create a new project, run 'azd init'` | You are in the folder you made with `mkdir`, not the one `init` nested inside it. | `cd` into the folder named in `init`'s `Copying template code from local path to:` line. See [§ 1](#1-scaffold-the-tools-sample). |
| `RESPONSE 404: SubscriptionNotFound` right after picking a subscription during `init` | The subscription the picker offered is not usable — check you did not pick a near-identically named sibling. Mechanism not established. | `azd env set AZURE_SUBSCRIPTION_ID <id>`, `azd env set AZURE_TENANT_ID <id>` and `azd env set AZURE_LOCATION <region>`. Setting only `AZURE_LOCATION`, as azd suggests, is not enough. |
| `RESPONSE 404: NotFound / "Project not found"` from `azd deploy`, on a project that exists | Intermittent; seen once on an immediate redeploy. Cause not established. | Re-run `azd deploy` unchanged. It succeeded in 14 s. Confirm first with `az resource list -g <rg>` — if the project is listed, nothing is broken. See [§ 6](#6-deploy-it). |
| `azd down` ends in `409 RequestConflict / provisioning state is not terminal` | `azd` reached the purge before Azure finished deleting. The group is gone; the account is soft-deleted. | Wait a few minutes, then `az cognitiveservices account purge -n cog-<random> -g rg-<name> -l <region>`. Verify with `az cognitiveservices account list-deleted` — `az group exists` returns `false` even while this is unresolved. See [§ 7](#7-clean-up). |
| Your new tool from § 5 is ignored, and the log never names it | `azd ai agent run` starts the server in the foreground and has no reload flag, so it is still running the code you had when you started it. | Ctrl+C and re-run `azd ai agent run --no-client`. Confirm with `gen_ai.tool.definitions` — it should now list both tools. See [§ 5](#5-add-your-own-tool). |
| The agent never uses your tool | The name/description does not tell the model when to call it. | Make the docstring or C# description specific and action-oriented. Check what the model actually received in `gen_ai.tool.definitions` — see [§ 4](#4-run-locally). |
| `ValueError: Model is required. Set via 'model' parameter or 'FOUNDRY_MODEL' environment variable.` | Nothing set `AZURE_AI_MODEL_DEPLOYMENT_NAME` — § 1's fifth question did not run, and `azd provision` never sets it. `azure.yaml` maps the variable through as `value: ${AZURE_AI_MODEL_DEPLOYMENT_NAME}`, so the container receives an **empty string** rather than nothing, which is why this is a `ValueError` from the client and not a `KeyError`. `doctor` does not catch it. | Run `azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini`, then `azd ai agent run` again. |
| **C#** — `System.ArgumentException: Argument is whitespace (Parameter 'model')`, thrown from `AsAIAgent` at `Program.cs` line 17 | The same empty string as the row above, reaching C# instead of Python. The sample's `?? "gpt-4o"` cannot save you: `??` tests for `null`, and an empty string is not null. Verified 2026-08-14. | Identical fix: `azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini`, then re-run. See [§ 2](#2-inspect-the-tool-definition). |
| **C#** — `ManagedIdentityCredential authentication failed: All Managed Identity sources are unavailable … IMDSv2 probe failed` on every local `invoke` | Only on a machine with no Azure Instance Metadata Service — a laptop, or WSL. .NET's `DefaultAzureCredential` treats the IMDS **timeout** as a fatal error rather than "credential unavailable", so it stops instead of falling through to your `az login`. Python's does not, which is why only this track breaks. Costs 100 s before it fails. Deployed agents are unaffected: they have a real managed identity. | Restrict the chain to developer credentials for the local run only: `AZURE_TOKEN_CREDENTIALS=dev azd ai agent run --no-client`. Verified 2026-08-14 — the same invoke then answered in 9.020 s. |
| A harmless question like `2+2` calls the tool | The tool description is too broad — or the previous question is still in the session. | Narrow when the tool should be used, and retest with `--new-session --new-conversation`. |
| You cannot tell whether the tool ran | `invoke` never shows tool calls. | Read the `azd ai agent run` terminal locally, or `azd ai agent monitor` after deploying. |
| Tool works locally but not after deploy | You changed source but did not redeploy, or cloud identity differs. | Run `azd deploy`; check `azd ai agent show --output json`. |

Everything else: [troubleshooting](../reference/troubleshooting.md).

## ✏️ Exercise

Do this **while you still have § 4's setup running**, before § 7 tears it down. Change only
the tool description/docstring, not the tool code, and **restart `azd ai agent run`** — it
does not reload ([§ 5](#5-add-your-own-tool)). Predict whether this prompt should call the
tool: `What is 2+2?` Then run it and compare — in a clean session, or the previous question is
still in context:

```bash
azd ai agent invoke --local --new-session --new-conversation "What is 2+2?"
```

<details>
<summary>Solution</summary>

A good tool description should keep `What is 2+2?` inside the model, with no tool call. If the
agent calls the tool, the description is too broad; describe the external fact the tool returns
(weather, hotel availability, stock price) and not generic answering.

Measured with the sample's own description, unchanged: `What is 2+2?` produced
`finish_reasons: ["stop"]` and no `Function name:` line, both with the weather question still
in the session and in a clean `--new-session --new-conversation` run. That held on three
separate walks. **The edited-description half of this exercise has not been measured** — the
prediction above is reasoning, not a captured result.
</details>

## → Next

[Lab 05 — Add a Foundry Toolbox](05-mcp-toolbox.md)

---

<sub>[← Deploy](03-deploy.md) · [🧪 Tutorial index](README.md) · [MCP toolbox →](05-mcp-toolbox.md)</sub>
