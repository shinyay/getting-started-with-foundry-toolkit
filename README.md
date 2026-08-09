# Getting Started with Microsoft Foundry Toolkit

> **Ship your first AI agent to Azure in about ten minutes — then understand every command you just ran.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://gist.githubusercontent.com/shinyay/56e54ee4c0e22db8211e05e70a63247e/raw/f3ac65a05ed8c8ea70b653875ccac0c6dbc10ba1/LICENSE)
[![azd](https://img.shields.io/badge/azd-%E2%89%A5%201.27.1-0078D4)](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)
[![Python](https://img.shields.io/badge/Python-3.13%20%7C%203.14-3776AB?logo=python&logoColor=white)](samples/python/)
[![.NET](https://img.shields.io/badge/.NET-10-512BD4?logo=dotnet&logoColor=white)](samples/csharp/)
[![Verified](https://img.shields.io/badge/verified-against%20live%20Azure-success)](docs/tutorial/02-first-agent.md)

A hands-on guide to the **Microsoft Foundry Toolkit** covering **both** the `azd ai agent` CLI
and the VS Code extension, with a 4-step **Python** sample ladder and **C#** code rungs through step 03. Every command and
every output block was captured from a **real run against real Azure** — not copied from docs.

> [!NOTE]
> Verified **2026-08-09** against `azd 1.30.0` · `azure.ai.agents 1.0.0-beta.9` ·
> `azure.ai.toolboxes 1.0.0-beta.5` · `azure.ai.inspector 1.0.0-beta.3` · `azure.ai.projects 1.0.0-beta.5`.
> Hosted agents and `azd ai agent` are in **preview**; details shift between versions.

---

## 🚀 Quick Start

```bash
# 0. Prerequisites — the version floor is not optional
#    Don't have azd yet? See docs/tutorial/01-setup.md for Linux/Windows/macOS install.
#    macOS w/ Homebrew:
brew update && brew upgrade azd     # need >= 1.27.1
#    Linux / WSL:  curl -fsSL https://aka.ms/install-azd.sh | bash
#    Windows:      winget install microsoft.azd
azd extension upgrade --all
az login && azd auth login

# 1. Scaffold
mkdir my-agent && cd my-agent
azd ai agent init -m "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/responses/01-basic/azure.yaml"

# 2. Provision  (~1m25s, creates 2 Azure resources)
azd env set AZURE_SUBSCRIPTION_ID <your-subscription-id>
azd env set AZURE_LOCATION eastus2
azd provision
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini   # ⚠️ provision does NOT set this

# 3. Run locally  → http://localhost:8088
azd ai agent run --no-client

# 4. Talk to it (second terminal)
azd ai agent invoke --local "What is Microsoft Foundry?"

# 5. Deploy  (~2m3s)
azd deploy && azd ai agent invoke "What is Microsoft Foundry?"

# 6. ⚠️ Always clean up
azd down --force --purge
```

---

## 💡 Why this repo exists

The Foundry Toolkit is **four different products sharing one name**, documented in four
different places — and the most capable one is barely documented at all.

| | Product | Where it's documented |
|---|---|---|
| **A** | VS Code extension | [VS Code docs](https://code.visualstudio.com/docs/intelligentapps/overview) (16 pages) |
| **B** | **`azd ai agent` CLI** | ⚠️ **`azd` appears 0 times across all 16 doc pages** |
| **C** | `microsoft-foundry` Copilot skill | a skill repo, written for agents not humans |
| **D** | Foundry Canvas | a bundled Copilot App plugin |

This repo documents **A and B equally**, from verified runs, and maps the ecosystem so you know
[which official pages to trust](docs/reference/ecosystem.md#vs-code-docs-staleness-map).

---

## 🧭 Three ways to use this repo

This guide is built around one idea: **読む → 手を動かす → 後で引く**
(*understand it → do it → look it up later*). Each mode is a separate directory, so you always
know which one you are in.

| | Mode | What it is | Start here |
|---|---|---|---|
| 📘 | **[Learn](docs/learn/README.md)** | 10 pages of mental model. **No commands to type.** | [What is the toolkit?](docs/learn/01-what-is-the-toolkit.md) |
| 🧪 | **[Tutorial](docs/tutorial/README.md)** ⭐ | 10 checkpointed labs. Every output verified live. | [Lab 01 — Setup](docs/tutorial/01-setup.md) |
| 📖 | **[Reference](docs/reference/README.md)** | 15 pages to look things up mid-task. | [Cheatsheet](docs/reference/cheatsheet.md) · [FAQ](docs/reference/faq.md) |

### Pick a route through the labs

| Route | Time | Cost | You end up with |
|---|---|---|---|
| ⚡ **Quick win** — labs 01→03 | ~1 h | ~$0.05 | an agent live on Azure, then destroyed |
| 🌗 **Practitioner** — + labs 04–06 | ~3 h | ~$0.30 | tools, MCP, and a **measured** quality score |
| 🏭 **Production** — + labs 07–10 | ~6 h | ~$1 | containers, tracing, A2A, CI/CD |

→ Full route details and the lab index: **[docs/tutorial/README.md](docs/tutorial/README.md)**

### Or jump straight to a question

| I want to… | Go here |
|---|---|
| …get one answer to one question | [❓ FAQ](docs/reference/faq.md) ⭐ |
| …keep one page open in a second tab | [🗂️ Cheatsheet](docs/reference/cheatsheet.md) ⭐ |
| …not know what a word means | [📚 Glossary](docs/reference/glossary.md) ⭐ |
| …fix an error I'm staring at | [🔧 Troubleshooting](docs/reference/troubleshooting.md) ⭐ |
| …build in VS Code instead of the terminal | [🖱️ VS Code route](docs/tutorial/alt-vscode.md) |
| …build an agent with no code | [💬 Prompt agents](docs/tutorial/alt-prompt-agents.md) |
| …learn by running code | [🧪 Samples ladder](samples/README.md) |
| …look up a manifest field | [🧾 `azure.yaml` reference](docs/reference/azure-yaml.md) |
| …look up a CLI flag | [⚙️ CLI reference](docs/reference/azd-cli.md) |
| …fix a 403 that only happens in the cloud | [🔐 Identity & RBAC](docs/reference/identity-and-rbac.md) ⭐ |
| …split one agent into several | [🕸️ Multi-agent](docs/reference/multi-agent.md) |
| …know what it costs | [💰 Cost](docs/reference/cost.md) |
| …know which official docs to believe | [🗺️ Ecosystem map](docs/reference/ecosystem.md) |

---

## ✨ What's inside

```text
docs/
├── learn/              📘 read — 10 pages of mental model, each with a self-check
├── tutorial/           🧪 do — 10 checkpointed labs, every output verified ⭐
└── reference/          📖 look up — 15 pages · cheatsheet · FAQ · manifest schema
                        identity · cost · observability · deploy modes · glossary
samples/
├── python/       01-hello-world → 02-tools → 03-mcp-toolbox → 04-eval
└── csharp/       01 → 03 (step 04 is CLI-level and language-agnostic)
```

### The sample ladder

```mermaid
flowchart LR
    A["<b>01</b> hello-world<br/><i>it responds</i>"] --> B["<b>02</b> tools<br/><i>it acts</i>"]
    B --> C["<b>03</b> mcp-toolbox<br/><i>it borrows<br/>capability</i>"]
    C --> D["<b>04</b> eval<br/><i>it is measured</i>"]
```

---

## 🏗️ How it works

Six verbs, whichever track you pick:

```mermaid
flowchart LR
    I["1 · init"] --> P["2 · provision"] --> R["3 · run"] --> D["4 · deploy"] --> V["5 · invoke"] --> E["6 · eval"]
    E -.->|iterate| R
```

| Verb | CLI | VS Code |
|---|---|---|
| init | `azd ai agent init` | **Create Agent** wizard |
| provision | `azd provision` | Foundry Project Setup |
| run | `azd ai agent run` → agent **:8088** + Inspector UI **:8087** | `F5` → Agent Inspector (debugpy **:5679**) |
| deploy | `azd deploy` | **Deploy Hosted Agent** wizard |
| invoke | `azd ai agent invoke` | Agent Playground |
| eval | `azd ai agent eval` | **Evaluation** view |

<details>
<summary><b>What actually gets created in Azure</b> (reassuringly little)</summary>

```text
Name                                                Sku
--------------------------------------------------  -------
cog-6kkz3uyx7e75m                                   S0
cog-6kkz3uyx7e75m/<project>
```

That's it — no ACR, no Key Vault, no storage account, unless you opt into container mode, which
adds a **third** line at the **Premium** SKU:

```text
cr6kkz3uyx7e75m                                     Premium
```
Each deployed agent gets **its own managed identity**, and agents are versioned by name
(`my-agent:1`, `my-agent:2`, …).
</details>

---

## 🔧 Top 3 things that will break

<details open>
<summary><b>1. Your <code>azd</code> is too old</b> — the #1 first-run failure</summary>

```text
ERROR: … must contain 'template' field
```

Nothing is wrong with your YAML. `azd` gates the extension version, which gates the manifest
format it can parse. Worse, the extension **silently downgrades** with only a warning:

```text
WARNING: 1.0.0-beta.9 is incompatible with azd 1.25.6 (requires ">=1.27.1"),
installing 0.1.41-preview instead
```

```bash
brew update && brew upgrade azd     # brew update is required — a stale tap capped at 1.26.0
azd extension upgrade --all
```
</details>

<details>
<summary><b>2. <code>AZURE_AI_MODEL_DEPLOYMENT_NAME</code> is never set for you</b></summary>

`azure.yaml` interpolates it and the code requires it, but `azd provision` only writes the JSON
blob `AI_PROJECT_DEPLOYMENTS`.

```bash
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
```
</details>

<details>
<summary><b>3. The scary <code>169.254.169.254</code> traceback is harmless</b></summary>

`DefaultAzureCredential` probing the Azure Instance Metadata Service, which doesn't exist on
your laptop. It falls back to `az login`. Ignore it.
</details>

→ [21 sections of real, captured failures](docs/reference/troubleshooting.md)

> [!TIP]
> **`azd ai agent doctor` is the best tool in the toolkit.** 13 checks across local config,
> authentication and remote state — it names the exact variable or role that's missing.
> There is no GUI equivalent.

---

## 📖 Verified timings & cost

Measured on real runs, `eastus2`, `gpt-5.4-mini`. Your numbers will differ — the point is the
*order of magnitude*, and which commands are unexpectedly expensive.

| Step | Time | Note |
|---|---|---|
| `azd provision` | **1 m 34 s** | creates 2 resources |
| `azd deploy` (code mode) | **2 m 41 s** | |
| `azd provision` (container mode) | **2 m 39 s** | + a **Premium** ACR appears |
| `azd deploy` (container mode) | **2 m 40 s** | builds and pushes the image |
| `azd ai agent invoke` | **14.4 s** | cold; ~7 s warm |
| `azd ai agent eval generate` | **≥ 8 m 51 s** | ⚠️ the slowest command in the toolkit |
| `azd ai agent eval run` | **3 m 51 s** | 15 cases → **9 passed, 6 failed** |
| `azd down --force --purge` | **2 m 53 s** | |

Times marked from `azd`'s own `SUCCESS: … in N minutes M seconds` line, except `eval generate`
(azd prints no total — this is the last observed progress tick) and `eval run` (wall clock).

> [!NOTE]
> **`eval run` reporting 6 failures is the correct result, not a broken sample.** The generated
> rubric grades identity fidelity; the sample's instructions are just *"You are a friendly
> assistant"*. The gap between the two **is** the lesson — see [Lab 06](docs/tutorial/06-evaluate.md).

Cost is token-based plus hosted-agent compute (`0.5 vCPU / 1Gi` in these samples). Full
breakdown, including the standing daily cost container mode adds: [💰 Cost](docs/reference/cost.md).

> [!CAUTION]
> Always finish with **`azd down --force --purge`**. Without `--purge`, the Cognitive Services
> account is soft-deleted for 48 hours and keeps its name, blocking re-provisioning.

---

## 📚 References

| Source | Trust |
|---|---|
| `azd ai agent --help` | ⭐ always matches your installed version |
| [`WHATS_NEW.md`](https://github.com/microsoft/foundry-toolkit/blob/main/WHATS_NEW.md) | ⭐ the only continuously-maintained source |
| [Microsoft Learn — Foundry agents](https://learn.microsoft.com/azure/ai-foundry/agents/) | ✅ service + CLI |
| [VS Code docs](https://code.visualstudio.com/docs/intelligentapps/overview) | ⚠️ GUI only; several stale pages |
| [`microsoft-foundry` skill](https://github.com/microsoft/azure-skills) | ✅ CLI workflows |
| [`foundry-samples`](https://github.com/microsoft-foundry/foundry-samples) | ✅ the 34 samples |

> [!TIP]
> Install the **Microsoft Foundry Skill** for your coding agent. Microsoft's published
> benchmark on an identical task: **33 min → 10.5 min** (−69%) and **410 → 100 credits** (−76%).
> ```bash
> npx skills add https://github.com/microsoft/azure-skills --skill microsoft-foundry
> ```

---

## 🤝 Contributing

Found a bug, or has the preview moved on? [Open an issue](https://github.com/shinyay/getting-started-with-foundry-toolkit/issues/new).

Because this guide's value is that its output is **real**, please re-run the command and paste
the actual output when correcting a verified block.

---

## ⭐ Support

If this project helps you, please consider:
- ⭐ Starring this repository
- 🐛 [Reporting issues](https://github.com/shinyay/getting-started-with-foundry-toolkit/issues/new)
- 📢 Sharing with others

---

## Licence

Released under the [MIT license](https://gist.githubusercontent.com/shinyay/56e54ee4c0e22db8211e05e70a63247e/raw/f3ac65a05ed8c8ea70b653875ccac0c6dbc10ba1/LICENSE)

## Author

- github: <https://github.com/shinyay>
- bluesky: <https://bsky.app/profile/yanashin.bsky.social>
- twitter: <https://twitter.com/yanashin18618>
- mastodon: <https://mastodon.social/@yanashin>
- linkedin: <https://www.linkedin.com/in/yanashin/>
