# 02 · Tools

> **New idea:** the agent stops just talking and starts *doing*.

A tool is an ordinary Python function. The `@tool` decorator exposes it to the model; the
Agent Framework handles schema generation, the call loop and result plumbing.

---

## Scaffold it

```bash
mkdir 02-tools && cd 02-tools
azd ai agent init -m "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/responses/02-tools/azure.yaml"
```

---

## The only real change from 01

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
    tools=[get_weather],          # ← the whole difference
    default_options={"store": False},
)
```

### How the model learns what the tool does

| Source | Becomes |
|---|---|
| function **name** | the tool name |
| **docstring** | the tool description — *the model reads this to decide when to call it* |
| `Annotated[str, Field(description=…)]` | the parameter's JSON-schema description |
| return type | the result contract |

> [!TIP]
> The docstring and `Field(description=…)` are **prompt engineering**, not documentation.
> Vague descriptions are the most common cause of "the agent won't use my tool".

### `approval_mode`

| Value | Behaviour |
|---|---|
| `"never_require"` | call it silently — right for safe, read-only lookups |
| *(default)* | the host may require human approval before executing |

Use approval for anything that writes, spends money or sends messages.

---

## Run it

```bash
azd provision
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
azd ai agent run --no-client
```

```bash
azd ai agent invoke --local "What's the weather in Tokyo?"
```

The agent will call `get_weather("Tokyo")` and phrase the result. Compare:

```bash
azd ai agent invoke --local "What is 2+2?"        # no tool call — the model just answers
```

Watch the tool call happen:

```bash
azd ai agent run          # (no --no-client) → Agent Inspector shows the call/result timeline
```

---

## Adding your own tool

```python
@tool(approval_mode="never_require")
def get_stock_price(
    symbol: Annotated[str, Field(description="Ticker symbol, e.g. MSFT.")],
) -> str:
    """Look up the latest closing price for a stock ticker."""
    return f"{symbol} closed at $432.10."

agent = Agent(client=client, instructions="…", tools=[get_weather, get_stock_price], …)
```

Three rules that matter:

1. **Return a string** (or something trivially serialisable) — it goes back into the prompt.
2. **Raise informative exceptions.** The model sees the error and can retry or explain.
3. **Keep tools fast.** Every call is inside the user's latency budget.

---

## Deploy

```bash
azd deploy
azd ai agent invoke "What's the weather in Osaka?"
azd ai agent monitor -f        # watch the tool call server-side
```

Tools run **inside your container** — same code, same process. Nothing is registered with
Foundry, so there is no extra deployment step.

---

## Local tools vs catalog tools

| | Local tools *(this sample)* | Catalog / Toolbox tools *(next)* |
|---|---|---|
| Defined in | your source | Foundry / MCP servers |
| Runs in | your container | external service |
| Version with | your code | independently |
| Best for | business logic, private APIs | search, code interpreter, shared capabilities |

---

## Clean up

```bash
azd down --force --purge
```

---

👉 Next: [03 · MCP & Foundry Toolbox](../03-mcp-toolbox/) — tools you didn't write.
