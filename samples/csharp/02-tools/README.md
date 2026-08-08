# 02 · Tools (C#)

> **New idea:** the agent stops just talking and starts *doing*.

A hotel concierge that looks up availability with a local function.

```bash
mkdir 02-tools && cd 02-tools
azd ai agent init -m "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/csharp/hosted-agents/agent-framework/local-tools/azure.yaml"
```

---

## The only real change from 01

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
                GetAvailableHotels,                       // ① the method
                "GetAvailableHotels",                     // ② tool name
                "Gets a list of available hotels in Seattle with details about amenities and pricing.")  // ③
        ]);

static string GetAvailableHotels(
    string? checkInDate = null,
    string? checkOutDate = null,
    int? guests = null)
{
    // …returns hotel data as a string
}
```

| | Python equivalent |
|---|---|
| ① method group | the decorated function |
| ② explicit name | function `__name__` |
| ③ explicit description | the **docstring** |

C# passes the description explicitly rather than reading a docstring — but it plays the same
role. **It is prompt engineering.** The model reads it to decide whether to call the tool.

Parameters are inferred from the signature; optional params (`string?`, `int?`) become optional
in the generated schema.

---

## Run it

```bash
azd provision
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
azd ai agent run --no-client
```

```bash
azd ai agent invoke --local "I need a hotel in Seattle for 2 guests next weekend."
azd ai agent invoke --local "What is 2+2?"      # no tool call — compare the two
```

Run without `--no-client` to watch the call/result timeline in the Agent Inspector.

---

## Adding your own tool

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

Rules:

1. **Return a string** (or something trivially serialisable) — it re-enters the prompt.
2. **Throw informative exceptions.** The model sees them and can recover.
3. **Keep tools fast** — they sit inside the user's latency budget.
4. **`static`** unless the tool genuinely needs instance state.

---

## Deploy

```bash
azd deploy
azd ai agent invoke "Find me a hotel in Seattle."
azd ai agent monitor -f
azd down --force --purge
```

Tools run **inside your container** — nothing is registered with Foundry.

---

👉 Next: [03 · MCP tools](../03-mcp-toolbox/)
