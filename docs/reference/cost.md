# 💰 Cost

> **Captured live on 2026-08-08** — resource inventory and SKUs are from a real
> `azd provision` + `azd deploy` of
> [`samples/csharp/01-hello-world`](../../samples/csharp/01-hello-world/) in `eastus2`,
> subsequently destroyed.
>
> [!WARNING]
> **No prices appear on this page.** Azure pricing varies by region, currency, agreement and
> date, and a number written here would be wrong within weeks. This page tells you exactly
> **what meters you turned on** — go to the
> [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) for what they
> cost *you*, *today*.

---

## What actually gets created

After a complete `provision` + `deploy` in the default (code) mode:

```bash
az resource list -g rg-<env> --query "[].{type:type,name:name}" -o table
```

```text
Name
----------------------
cog-m3ln646lhfgcu
cog-m3ln646lhfgcu/rdcs
```

**Two resources.** One `Microsoft.CognitiveServices` account (`kind: AIServices`, `sku: S0`)
and one project nested inside it. That is the entire footprint.

Notably absent — each one a meter you did **not** turn on:

| Not created | Confirmed by |
|---|---|
| Container Registry | `AZURE_CONTAINER_REGISTRY_ENDPOINT=""` |
| Application Insights | no `APPLICATIONINSIGHTS_CONNECTION_STRING` in `azd env get-values` |
| Storage / Cosmos DB / AI Search | `ENABLE_CAPABILITY_HOST="false"` |
| VNet, private endpoints | `AZURE_FOUNDRY_NETWORK_MODE="none"` |

This is the single most reassuring fact about getting started: **the default footprint is
small, and the expensive things are opt-in.**

---

## The meters you are on

### 1. Model inference — the one that matters

The dominant cost, and it is **per token**, not per hour. The captured run auto-deployed:

```json
{
  "name": "gpt-5.4-mini",
  "model":  { "name": "gpt-5.4-mini", "format": "OpenAI", "version": "2026-03-17" },
  "sku":    { "name": "GlobalStandard", "capacity": 10 }
}
```

Two fields drive the bill:

- **`GlobalStandard`** is *pay-as-you-go per token*. You are **not** renting capacity — an idle
  deployment on GlobalStandard bills nothing for sitting there. This is why leaving a
  provisioned environment overnight is usually cheap and leaving a *chatty* one is not.
  Provisioned-throughput SKUs behave the opposite way: reserved and billed whether used or not.
- **`capacity: 10`** on GlobalStandard is a **rate limit** (tokens-per-minute quota), not a
  spend commitment. Raising it raises your ceiling, not your bill.

> [!TIP]
> `gpt-5.4-mini` is chosen by the toolkit for a reason. Staying on a `-mini` class model while
> learning is the highest-leverage cost decision available — often an order of magnitude
> cheaper per token than a frontier model, and indistinguishable for "hello world".

### 2. The account itself

`S0` on `AIServices` is the standard pay-as-you-go tier. The account is a container; the
consumption rides on the model deployments inside it.

### 3. Hosted agent compute

Your agent runs in a Foundry-managed container. The captured deployment took **2m6s** to reach
`active` and then stayed running to serve requests — meaning **a deployed agent is a running
thing, not a function that sleeps**. It is the cost that accrues while you are not looking.

`container.resources` in [`azure.yaml`](./azure-yaml.md#containerresources) (`cpu`, `memory`)
is the lever here. Bigger containers cost more.

---

## The one number worth knowing

```text
azd provision   1m43s      → 2 resources
azd deploy      3m01s      → agent active (2m6s of that = agent activation)
```

Under **five minutes** from nothing to a working cloud agent, and about the same to destroy it.
That short cycle is itself a cost control: there is no reason to leave an environment standing
"because rebuilding is a hassle". It isn't.

---

## Keeping it near zero

```bash
azd down --force --purge
```

> [!IMPORTANT]
> **`--purge` is not optional.** Cognitive Services accounts are **soft-deleted** by default:
> without `--purge` the account lingers in a recoverable state, keeps its name reserved, and
> counts against your quota — so the *next* `azd provision` can fail with a name conflict for a
> resource you believe you deleted. `--force` skips the prompt; `--purge` is what actually
> removes it.

Verify, don't assume:

```bash
az group exists -n rg-<env>                                    # false
az cognitiveservices account list-deleted -o table             # should not list yours
```

Checklist:

| Habit | Why |
|---|---|
| `azd down --force --purge` when done for the day | GlobalStandard is cheap idle, compute is not |
| Stay on `-mini` models while learning | Largest single lever |
| Don't add App Insights until you need it | Ingestion + retention are separate meters |
| Prefer code mode over container mode | Avoids ACR entirely ([deploy modes](./deploy-modes.md)) |
| Set a budget alert on the subscription | The only control that works while you sleep |
| One `azd` environment per experiment | Makes `down` genuinely complete |

```bash
az consumption budget create --budget-name foundry-learning \
  --amount 50 --time-grain Monthly --category Cost
```

---

## Where cost surprises come from

| Surprise | Cause | Avoided by |
|---|---|---|
| Bill after "I deleted it" | Soft-deleted account | `--purge` |
| Steady cost with no traffic | Agent container running | `azd down` |
| Cost jumped 10× | Switched off a `-mini` model | Check the deployment SKU |
| Unexpected storage line | Enabled capability host / BYO storage | `ENABLE_CAPABILITY_HOST` |
| Networking charges | Private endpoints | `AZURE_FOUNDRY_NETWORK_MODE` |

---

## See also

- 👉 [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) — real numbers
- 👉 [`deploy-modes.md`](./deploy-modes.md) — which mode adds ACR
- 👉 [`observability.md`](./observability.md) — App Insights as an added meter
- 👉 [`environment-variables.md`](./environment-variables.md) — the flags quoted above
