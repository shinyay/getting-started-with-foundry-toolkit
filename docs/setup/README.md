# 🛠️ Setup

> Goal: from a clean machine to `azd ai agent doctor` reporting all green.

---

## 1. Prerequisites

| Tool | Minimum | Why | Check |
|---|---|---|---|
| **Azure Developer CLI (`azd`)** | **`1.27.1`** ⚠️ | Runs the whole agent lifecycle | `azd version` |
| **Azure CLI (`az`)** | 2.60+ | Sign-in, resource inspection | `az version` |
| **Python** | 3.10+ *(any)* | Only to have *a* Python; azd fetches its own | `python3 --version` |
| **.NET SDK** | 10.0+ | C# samples only | `dotnet --version` |
| **Docker** | any recent | Only for `--deploy-mode container` | `docker --version` |
| **VS Code** | 1.90+ | GUI track only | `code --version` |
| **An Azure subscription** | — | With permission to create Cognitive Services | `az account show` |

> [!WARNING]
> **The `azd` version is the single most important prerequisite.**
> `azd` gates which `azure.ai.agents` extension version you may install, and the extension
> version gates which manifest format it can parse. An `azd` older than **1.27.1** will
> silently install extension `0.1.x`, which cannot read today's sample manifests, and you
> will get a confusing `must contain 'template' field` error.
> See [troubleshooting → version skew](../reference/troubleshooting.md#1-version-skew-the-one-that-bites-everyone).

Note that you do **not** need a local Python 3.13/3.14. `azd ai agent run` provisions its own
interpreter via `uv` (observed: it downloaded CPython 3.14.3 automatically).

---

## 2. Install `azd`

<details>
<summary><b>macOS / Linux (Homebrew)</b></summary>

```bash
brew update          # ← do not skip; a stale tap will offer an old azd
brew install azd     # or: brew upgrade azd
```

`brew update` matters. On a stale tap the newest offered version was `1.26.0` — below the
`1.27.1` floor — and only after `brew update` did `1.30.0` appear.
</details>

<details>
<summary><b>Linux (install script)</b></summary>

```bash
curl -fsSL https://aka.ms/install-azd.sh | bash
```
</details>

<details>
<summary><b>Windows (winget)</b></summary>

```powershell
winget install microsoft.azd
```
</details>

Verify:

```bash
azd version
```

```text
azd version 1.30.0 (commit …)
```

---

## 3. Install the agent extension

The agent commands ship as an `azd` **extension**, not in core `azd`.

```bash
azd extension install azure.ai.agents
```

Then immediately bring everything to latest:

```bash
azd extension upgrade --all
```

<details>
<summary>✅ Verified output</summary>

```text
(✓) Done: Upgraded azure.ai.inspector dependency (0.0.1-preview → 1.0.0-beta.3)
  1 upgraded (1 dependency updated), 1 skipped
SUCCESS: Extensions upgraded successfully
```
</details>

Confirm the extensions — `azure.ai.agents` pulls in three:

```bash
azd extension list --installed --output json
```

| Extension | Role |
|---|---|
| `azure.ai.agents` | the `azd ai agent` command tree |
| `azure.ai.inspector` | local Agent Inspector UI |
| `azure.ai.projects` | Foundry project/model management (pulled in as a dependency) |

> [!IMPORTANT]
> **A fourth extension installs itself later.** `azure.ai.toolboxes` (Beta) is *not* part of
> the `azure.ai.agents` bundle. The first time you run a `azd ai toolbox …` command, azd says:
>
> ```text
> Command 'ai toolbox' was not found, but there's an available extension that provides it
> Id: azure.ai.toolboxes   Name: Foundry Toolboxes (Beta)
> ```
>
> …and offers to install it, so your extension list silently becomes **four**. That is fine
> interactively but a failure in CI, where nothing can answer the prompt. Install it up front
> if your pipeline touches toolboxes:
>
> ```bash
> azd extension install azure.ai.toolboxes
> ```

> [!NOTE]
> Some sample READMEs tell you to run `azd ext install microsoft.foundry`. That is an alias
> for the same bundle. `azd extension install azure.ai.agents` is the explicit form.

---

## 4. Sign in

Two different CLIs, two different sign-ins. You need both.

```bash
az login                      # used by DefaultAzureCredential at runtime
azd auth login                # used by azd for provisioning/deploying
```

Check:

```bash
az account show --query "{sub:name, id:id, user:user.name}" -o table
```

If you have several subscriptions, pin one:

```bash
az account set --subscription <subscription-id>
```

---

## 5. Install the VS Code extension *(GUI track)*

Marketplace ID — **copy this exactly**:

```text
ms-windows-ai-studio.windows-ai-studio
```

```bash
code --install-extension ms-windows-ai-studio.windows-ai-studio
```

> [!CAUTION]
> The official docs page *Use Copilot tools* links to `itemName=ms-ai.vscode-ai-toolkit`,
> which **404s**. That ID does not exist. Use the one above.

The extension needs the **.NET Runtime**; it installs it on first activation, which can take
a minute or two on a cold start.

For Python debugging in the GUI track, also install:

```bash
code --install-extension ms-python.python
```

---

## 6. Optional but recommended: the Foundry Skill

If you use a coding agent (Copilot CLI, Claude Code), install the **Microsoft Foundry Skill**.
Microsoft's published benchmark on an identical task: **33 min 20 s → 10 min 30 s**
(−69% time) and **410 → 100 credits** (−76% cost).

```bash
npx skills add https://github.com/microsoft/azure-skills --skill microsoft-foundry
```

Or as a plugin:

```text
Copilot CLI:  /plugin marketplace add microsoft/azure-skills
              /plugin install azure@azure-skills
Claude Code:  /plugin install azure@claude-plugins-official
```

---

## 7. Verify everything

Inside any initialised agent project:

```bash
azd ai agent doctor
```

<details>
<summary>✅ Verified output — this is what "ready" looks like</summary>

```text
azd ai agent doctor

Local
   (✓) azd extension reachable
   (✓) azure.yaml present and parseable
   (✓) azd environment selected
   (✓) agent service in azure.yaml
   (✓) FOUNDRY_PROJECT_ENDPOINT set
   (✓) agent definition valid (per service)
   (✓) manual env vars set
   (-) Manifest toolboxes have endpoint env vars set -- skipped

Authentication
   (✓) authentication

Remote
   (✓) Foundry project endpoint reachable
   (✓) Developer has required role on Foundry project
   (✓) Hosted agents are active
   (-) Manifest connections exist on Foundry project -- skipped

11 passed, 0 failed, 2 skipped
```
</details>

`doctor` is the best debugging tool in the toolkit. Run it whenever anything is odd — it
checks local config, auth **and** remote state, and names the exact env var or role that is
missing.

---

## 8. Region and quota

The samples default to **`gpt-5.4-mini`**, `GlobalStandard`, capacity `10`.

```bash
azd env set AZURE_LOCATION eastus2
azd env set AZURE_AI_DEPLOYMENTS_LOCATION eastus2   # optional: models in a different region
```

`AZURE_AI_DEPLOYMENTS_LOCATION` exists so you can keep the project near you while placing the
model where you actually have quota. If provisioning fails on capacity, change that one
variable rather than moving the whole project.

---

## Next

👉 [The CLI golden path](../guide-cli/README.md) — build, run and deploy your first agent.
