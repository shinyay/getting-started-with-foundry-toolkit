# 🗺️ Ecosystem map — repos, products, and which docs to trust

Where everything lives, and how much to believe it.

---

## The four products

| # | Product | Identifier | Source available? |
|---|---|---|---|
| **A** | VS Code extension | `ms-windows-ai-studio.windows-ai-studio` | ❌ closed-source |
| **B** | `azd ai agent` CLI | azd extension `azure.ai.agents` | ❌ closed-source |
| **C** | Foundry Skill | `microsoft/azure-skills` → `microsoft-foundry` | ✅ open (187 files) |
| **D** | Foundry Canvas | Copilot **App** plugin | bundled in `foundry-toolkit` |

> [!IMPORTANT]
> **`github.com/microsoft/foundry-toolkit` contains no toolkit source code.** It is a
> docs / changelog / issue-tracker / sample-distribution repo. Do not go there looking for
> the implementation.

---

## Repositories

| Repo | What's actually in it |
|---|---|
| [`microsoft/foundry-toolkit`](https://github.com/microsoft/foundry-toolkit) | `WHATS_NEW.md` ⭐, issues, `doc/`, `samples/hosted-agent/sample-catalog.json`, `devpack-installer/`, the `microsoft-foundry` Canvas plugin |
| [`microsoft-foundry/foundry-samples`](https://github.com/microsoft-foundry/foundry-samples) | the 34 samples `azd ai agent sample list` serves |
| [`microsoft/azure-skills`](https://github.com/microsoft/azure-skills) | the `microsoft-foundry` Copilot skill |
| [`microsoft/agent-framework`](https://github.com/microsoft/agent-framework) | the SDK the samples build on |
| [`Azure/azure-dev`](https://github.com/Azure/azure-dev) | `azd` core + the `azure.yaml` JSON schema |

### Inside `microsoft/foundry-toolkit`

| Path | Trust |
|---|---|
| `WHATS_NEW.md` | ⭐ **the only continuously-maintained source.** 59 versions of real release notes |
| `doc/` | ⚠️ **13 of 17 files are marked "outdated and no longer maintained"** |
| `PRODUCT.md` / `DESIGN.md` | ⚠️ these spec the *changelog website*, not the product |
| `samples/hosted-agent/sample-catalog.json` | ✅ the 52-template catalog behind the GUI wizard |
| `devpack-installer/` | ❓ **Foundry DevPack** — 8 preview releases, purpose undocumented anywhere |

---

## Documentation sources, ranked

| Rank | Source | Covers | Caveat |
|---:|---|---|---|
| 1 | `azd ai agent --help` | CLI | always matches your installed version |
| 2 | `WHATS_NEW.md` | everything | changelog format, no tutorials |
| 3 | [Microsoft Learn — Foundry agents](https://learn.microsoft.com/azure/ai-foundry/agents/) | service + CLI | |
| 4 | The `microsoft-foundry` skill | CLI workflows | designed for agents, not humans |
| 5 | [VS Code docs](https://code.visualstudio.com/docs/intelligentapps/overview) | **GUI only** | ⚠️ `azd` appears **0 times** across all 16 pages |

### The `azd` documentation gap

Grepping all 16 VS Code doc pages for `azd` / "Azure Developer CLI" returns **zero hits**.
The entire CLI-first path — `init`/`run`/`invoke`/`eval`/`optimize`, the `host: azure.ai.agent`
contract, `provision`/`deploy` — is undocumented there. Yet that is exactly what the Foundry
Skill drives and what the Canvas plugin shells out to.

**Two parallel product surfaces; only one is documented on that site.** This repository exists
largely to fill that gap.

---

## VS Code docs staleness map

| Trust | Pages |
|---|---|
| ✅ | `hosted-agents`, `agentbuilder`, `agent-inspector`, `playground`, `evaluation`, `tracing`, `finetune`, `modelconversion`, `profiling` |
| ⚠️ | `overview` (sidebar ~2 versions behind), `models` (GitHub Models), `tool-catalog` (preview-gated), `copilot-tools` (broken link) |
| ❌ | `create-agents` (old 2-template wizard, contradicts `hosted-agents`), `faq` (dead repos; claims container support is "in progress" — it shipped) |
| ⚫ | `finetune-legacy` (explicitly historical) |

### Verified broken links

| Page | Broken | Correct |
|---|---|---|
| `copilot-tools` | `itemName=ms-ai.vscode-ai-toolkit` → **404** | `ms-windows-ai-studio.windows-ai-studio` |
| `faq` | `AI-Mou/windows-ai-studio` | dead |
| `faq` | `microsoft/vscode-ai-toolkit` | dead |

### Verified broken sample link

`azd ai agent sample list --language python` advertises
`…/agent-framework/responses/03-mcp/azure.yaml` → **404**. The directory no longer exists
upstream. The C# `mcp-tools` sample is live.

---

## ⚠️ GitHub Models is retired

`WHATS_NEW.md` v1.6.7 (5 Aug 2026) removed GitHub Models from the Model Catalog, playground,
model comparison, Prompt Builder and evaluations, following service retirement.

**But five doc pages still present it as the free on-ramp:** `overview`, `models`, `tracing`,
`copilot-tools`, `faq`.

Today's no-Azure options: **Foundry Local**, **Ollama**, **ONNX**, or a custom endpoint via
`windowsaistudio.remoteInfereneEndpoints` (the misspelling is in the product).

---

## The Foundry Skill

```bash
npx skills add https://github.com/microsoft/azure-skills --skill microsoft-foundry
```

Published benchmark, identical task, Sonnet 4.6:

| | Time | Credits |
|---|---|---|
| Without skill | 33 min 20 s | 410 |
| **With skill** | **10 min 30 s** | **100** |
| | **−69%** | **−76%** |

The VS Code extension **auto-installs** this same skill (plus
`microsoft-foundry-agent-framework-code-gen`). The extension and the CLI share one skill —
that is the real integration point between the two tracks.

---

## Maturity

| Component | Status |
|---|---|
| VS Code extension | GA-ish, docs partly stale |
| `azd ai agent` | **Preview** — moves fastest, breaks most |
| Hosted agents | **Preview** |
| Workflows / Routines | **Preview** |
| Foundry Skill | Preview |
| Foundry Canvas | Early preview |
| Foundry DevPack | 8 preview releases, purpose undocumented |

> [!NOTE]
> Preview means the manifest format has already changed once (agent manifest → unified
> `azure.yaml`). Pin your `azd` and extension versions in CI, and re-verify after upgrades.

---

## Getting help

```bash
azd ai agent doctor          # try this first
azd ai agent --help
azd version && azd extension list --installed --output json
```

Issues: <https://github.com/microsoft/foundry-toolkit/issues> (~93 open — search first).
