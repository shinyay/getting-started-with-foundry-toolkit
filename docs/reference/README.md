# 📖 Reference

Lookup tables. Read the [guides](../guide-cli/README.md) first; come back here for details.

Twelve pages, grouped by what you are trying to do.

### 🧭 Start here

| Page | Use it for |
|---|---|
| [`glossary.md`](glossary.md) | **every term in one place** — read this if a word is unfamiliar ⭐ |
| [`ecosystem.md`](ecosystem.md) | the four products, their repos, and which docs to trust |
| [`troubleshooting.md`](troubleshooting.md) | 16 real failures with real fixes ⭐ |

### 📐 The contract — what you write

| Page | Use it for |
|---|---|
| [`azure-yaml.md`](azure-yaml.md) | **field schema** + every field annotated |
| [`environment-variables.md`](environment-variables.md) | what's set by whom, and what never to declare |
| [`azd-cli.md`](azd-cli.md) | all **40** `azd ai agent` commands + the other three `azd ai` namespaces |

### ☁️ The platform — what Azure does

| Page | Use it for |
|---|---|
| [`deploy-modes.md`](deploy-modes.md) | code vs container vs BYO-image, and when ACR appears |
| [`infrastructure.md`](infrastructure.md) | ejecting to Bicep or Terraform, and what lands |
| [`identity-and-rbac.md`](identity-and-rbac.md) | the **three identities**, and why local≠cloud 🔐 |

### 🏭 Running it for real

| Page | Use it for |
|---|---|
| [`observability.md`](observability.md) | `monitor`, logs, and adding tracing |
| [`cost.md`](cost.md) | what actually bills, and keeping it near zero 💰 |
| [`sample-catalog.md`](sample-catalog.md) | all 34 upstream samples (21 Python + 13 C#) |

---

## Fast facts

| | |
|---|---|
| Minimum `azd` | **1.27.1** (this guide verified on 1.30.0) |
| Extensions | `azure.ai.agents` `1.0.0-beta.9`, `azure.ai.inspector` `1.0.0-beta.3`, `azure.ai.projects` `1.0.0-beta.5`, `azure.ai.toolboxes` (Beta) |
| VS Code extension ID | `ms-windows-ai-studio.windows-ai-studio` |
| Ports | **8088** agent (`--port`) · **8087** Inspector UI (`--inspector-port`) · **5679** debugpy (AI Toolkit) |
| Default model | `gpt-5.4-mini`, `GlobalStandard`, capacity 10 |
| Runtimes | `python_3_13`, `python_3_14`, `dotnet_10` |
| Protocols | `responses` ⭐, `invocations`, `invocations_ws`, `activity` |
| CPU/memory tiers | `0.25/0.5Gi`, `0.5/1Gi`, `1/2Gi`, `2/4Gi` |
| Resources created | 1 CognitiveServices account + 1 project |
| Verified timings | see the table below |

---

## Verified runs

Every number here came from a real run against live Azure, then destroyed. Nothing is estimated.

| | Python `01-hello-world` | C# `01-hello-world` | Python (earlier run) |
|---|---|---|---|
| Date | 2026-08-08 | 2026-08-08 | earlier session |
| `azd provision` | **1m20s** | **1m43s** | 1m25s |
| `azd deploy` | **2m21s** | **3m1s** | 2m3s |
| First `invoke` | **14.242s** (first byte 7.357s) | **6.877s** (first byte 3.622s) | — |
| `azd ai agent eval` | — | — | 3m15s |
| `azd down --force --purge` | **1m46s** | **1m45s** | 2m11s |
| Resources created | **2** | **2** | 2 |
| RG-scope role assignments | **0** | **0** | — |

What to take from this:

- **Two resources, every time.** Language and run do not change the footprint — one
  `Microsoft.CognitiveServices` account plus one nested project. See [cost](cost.md).
- **Under 5 minutes, cold to serving.** Both languages. Rebuilding an environment is cheap, so
  there is no reason to leave one running.
- **Timings vary ±20% run to run.** The Python run was *faster* to provision and deploy but
  *slower* on first invoke — with a single sample each, that is noise, not a language ranking.
  Treat these as orders of magnitude, not benchmarks.
- **First invoke is always the slow one** — cold start plus a managed-identity token fetch. See
  [observability](observability.md).
- **Zero role assignments, both runs.** The golden path runs on *inherited* permissions. See
  [identity & RBAC](identity-and-rbac.md).


## The commands you'll actually type

```bash
azd ai agent init -m "<manifestUrl>"        # scaffold
azd env set AZURE_SUBSCRIPTION_ID <id>
azd env set AZURE_LOCATION eastus2
azd provision
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini   # ⚠️ always needed
azd ai agent run --no-client                # terminal 1
azd ai agent invoke --local "hello"         # terminal 2
azd deploy
azd ai agent invoke "hello"
azd ai agent doctor                         # when anything is wrong
azd down --force --purge                    # ⚠️ always finish here
```

---

## Top 3 gotchas

1. **`azd` < 1.27.1** → extension silently downgrades → `must contain 'template' field`.
2. **`AZURE_AI_MODEL_DEPLOYMENT_NAME` is never set by `provision`** → `RuntimeError` on first run.
3. **`azd down` without `--purge`** → soft-deleted account holds the name for 48 h.

All three are covered in [troubleshooting](troubleshooting.md).
