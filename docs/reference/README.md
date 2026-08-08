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
| Verified timings (Python) | provision 1m25s · deploy 2m3s · eval 3m15s · teardown 2m11s |
| Verified timings (C#) | provision 1m43s · deploy 3m1s · teardown 1m45s |

---

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
