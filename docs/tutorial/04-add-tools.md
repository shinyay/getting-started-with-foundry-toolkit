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

```bash
azd provision
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
```

Same gotcha as Lab 02: `AZURE_AI_MODEL_DEPLOYMENT_NAME` is never set by `azd provision`.

### 4. Run locally

```bash
azd ai agent run --no-client
```

In a second terminal, ask for something that needs the tool.

Python:

```bash
azd ai agent invoke --local "What's the weather in Tokyo?"
```

C#:

```bash
azd ai agent invoke --local "I need a hotel in Seattle for 2 guests next weekend."
```

Then ask for something that should not use the tool:

```bash
azd ai agent invoke --local "What is 2+2?"
```

Run without `--no-client` to watch the call/result timeline in the Agent Inspector.

### 5. Add your own tool

Python:

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

Rules that matter:

1. **Return a string** or something trivially serializable — it goes back into the prompt.
2. **Raise or throw informative exceptions.** The model sees the error and can retry or explain.
3. **Keep tools fast.** Every call is inside the user's latency budget.
4. Use human approval for anything that writes, spends money or sends messages.

### 6. Deploy it

```bash
azd deploy
azd ai agent invoke "What's the weather in Osaka?"
azd ai agent monitor -f
```

Tools run **inside your container** — same code, same process. Nothing is registered with
Foundry, so there is no extra tool deployment step.

### 7. Clean up

```bash
azd down --force --purge
```

## ✅ Checkpoint

You should now be able to run a local invoke that asks for the tool:

```bash
azd ai agent invoke --local "What's the weather in Tokyo?"
```

<!-- illustrative: weather is random, so exact wording and temperature vary. -->
```text
Target:       localhost:8088 (local)
Message:      "What's the weather in Tokyo?"
Session:      …
Conversation: …

[local] The weather in Tokyo is cloudy with a high of 24°C.

Server responded in …s (first byte: …s)
```

If you see something else, jump to *If that didn't work* below.

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| The agent never uses your tool | The name/description does not tell the model when to call it. | Make the docstring or C# description specific and action-oriented. |
| `RuntimeError: Model deployment name is not configured.` | `azd provision` did not set `AZURE_AI_MODEL_DEPLOYMENT_NAME`. | Run `azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini`. |
| A harmless question like `2+2` calls the tool | The tool description is too broad. | Narrow when the tool should be used. |
| Tool works locally but not after deploy | You changed source but did not redeploy, or cloud identity differs. | Run `azd deploy`; check `azd ai agent show --output json`. |

Everything else: [troubleshooting](../reference/troubleshooting.md).

## ✏️ Exercise

Change only the tool description/docstring, not the tool code. Predict whether this prompt
should call the tool: `What is 2+2?` Then run it and compare.

<details>
<summary>Solution</summary>

A good tool description should keep `What is 2+2?` inside the model, with no tool call. If the
agent calls the tool, the description is too broad; describe the external fact the tool returns
(weather, hotel availability, stock price) and not generic answering.
</details>

## → Next

[Lab 05 — Add a Foundry Toolbox](05-mcp-toolbox.md)

---

<sub>[← Deploy](03-deploy.md) · [🧪 Tutorial index](README.md) · [MCP toolbox →](05-mcp-toolbox.md)</sub>
