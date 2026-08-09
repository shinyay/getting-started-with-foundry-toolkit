# 🛠️ Lab 01 — Set up your machine

> ⏱️ **15 min** · 📋 **Requires:** Azure subscription · 💰 **$0** · ☁️ **Creates 0 Azure resources**

Go from a clean machine to `azd ai agent doctor` reporting all green.

## What you'll learn

- Install the `azd` version that can read current Foundry Toolkit manifests.
- Install the agent, inspector, projects and toolbox extensions.
- Sign in with both CLIs the labs use.
- Run `doctor` and understand its exit codes.

## Steps

### 1. Prerequisites

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

### 2. Install `azd`

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

### 3. Install the agent extension

The agent commands ship as an `azd` **extension**, not in core `azd`.

```bash
azd extension install azure.ai.agents
```

Then immediately bring everything to latest:

```bash
azd extension upgrade --all
```

<details>
<summary>✅ Verified output — captured 2026-08-09, everything already current</summary>

```text
Upgrading azure.ai.agents extension
  (-) Skipped: Upgrading azure.ai.agents extension (No upgrade available)
Upgrading azure.ai.connections extension
  (-) Skipped: Upgrading azure.ai.connections extension (No upgrade available)
Upgrading azure.ai.inspector extension
  (-) Skipped: Upgrading azure.ai.inspector extension (No upgrade available)
Upgrading azure.ai.projects extension
  (-) Skipped: Upgrading azure.ai.projects extension (No upgrade available)
Upgrading azure.ai.toolboxes extension
  (-) Skipped: Upgrading azure.ai.toolboxes extension (No upgrade available)

  5 skipped

SUCCESS: Extensions upgraded successfully
```
</details>

> [!IMPORTANT]
> ✅ **Verified: there are five extensions, not four.** `azure.ai.connections` is easy to miss
> because nothing in the getting-started flow mentions it. Confirm yours with
> `azd extension list --installed`:
>
> ```text
> ID                     NAME                             STATUS       INSTALLED      LATEST
> azure.ai.agents        Foundry agents (Beta)            Up to date   1.0.0-beta.9   1.0.0-beta.9
> azure.ai.connections   Foundry Connections (Beta)       Up to date   1.0.0-beta.4   1.0.0-beta.4
> azure.ai.inspector     Foundry Agent Inspector (Beta)   Up to date   1.0.0-beta.3   1.0.0-beta.3
> azure.ai.projects      Foundry Projects (Beta)          Up to date   1.0.0-beta.5   1.0.0-beta.5
> azure.ai.toolboxes     Foundry Toolboxes (Beta)         Up to date   1.0.0-beta.5   1.0.0-beta.5
> ```

Confirm what you actually have:

```bash
azd extension list --installed
```

✅ Verified 2026-08-09 — **five** extensions, versions as pinned in
[reference → Fast facts](../reference/README.md#fast-facts):

| Extension | Role | How it got there |
|---|---|---|
| `azure.ai.agents` | the `azd ai agent` command tree | you installed it |
| `azure.ai.connections` | Foundry connections (used by A2A / MCP wiring) | dependency |
| `azure.ai.inspector` | local Agent Inspector UI | dependency |
| `azure.ai.projects` | Foundry project / model management | dependency |
| `azure.ai.toolboxes` | `azd ai toolbox …` | ⚠️ installs **on first use**, see below |

> [!IMPORTANT]
> **`azure.ai.toolboxes` installs itself the first time you need it**, mid-command. It is *not*
> part of the `azure.ai.agents` bundle, so the first `azd ai toolbox …` prompts:
>
> ```text
> Command 'ai toolbox' was not found, but there's an available extension that provides it
> Id: azure.ai.toolboxes   Name: Foundry Toolboxes (Beta)
> ```
>
> Fine interactively; **a hang in CI**, where nothing can answer the prompt. Install it up front
> if your pipeline touches toolboxes:
>
> ```bash
> azd extension install azure.ai.toolboxes
> ```

> [!NOTE]
> Some sample READMEs tell you to run `azd ext install microsoft.foundry`. That is an alias
> for the same bundle. `azd extension install azure.ai.agents` is the explicit form.

### 4. Sign in

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

### 5. Install the VS Code extension *(GUI track)*

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

### 6. Optional but recommended: the Foundry Skill

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

### 7. Verify everything

Inside any initialised agent project:

```bash
azd ai agent doctor
```

<details open>
<summary>✅ Verified output — captured 2026-08-09, before provisioning</summary>

```text
azd ai agent doctor

Local
   (✓) azd extension reachable
   (✓) azure.yaml present and parseable
   (✓) azd environment selected
   (✓) agent service in azure.yaml
   (x) FOUNDRY_PROJECT_ENDPOINT set
       FOUNDRY_PROJECT_ENDPOINT is not set in the current azd environment
       fix: Run `azd provision` to create the Foundry project, or `azd env set
       FOUNDRY_PROJECT_ENDPOINT <https://...>` to point at an existing one.
   (✓) agent definition valid (per service)
   (✓) manual env vars set
   (-) Manifest toolboxes have endpoint env vars set -- skipped (no toolbox resources
       declared in any service's agent.manifest.yaml.)

Authentication
   (-) authentication -- skipped (remote check excluded by --local-only)

Remote
   (-) Foundry project endpoint reachable -- skipped (remote check excluded by --local-only)
   (-) Developer has required role on Foundry project -- skipped (remote check excluded by --local-only)
   (-) Hosted agents are active -- skipped (remote check excluded by --local-only)
   (-) Manifest connections exist on Foundry project -- skipped (remote check excluded by --local-only)

6 passed, 1 failed, 6 skipped

To fix:

  Review the fix: notes above for each failed check.

Then re-run `azd ai agent doctor` to verify.
```
</details>

**Read this output, don't just scan it.** Three things it teaches:

1. **13 checks in three groups** — Local (8), Authentication (1), Remote (4). `--local-only`
   skips the 5 network ones, which is why 6 are skipped here.
2. **Every failure carries a `fix:` line with the exact command.** That is what makes `doctor`
   better than reading an error.
3. **A red `(x)` before you provision is correct.** `FOUNDRY_PROJECT_ENDPOINT` cannot exist yet —
   `azd provision` writes it. You are "ready" when the *Local* group is clean apart from that one.

<details>
<summary>What it looks like with no environment selected at all</summary>

```text
   (x) azd environment selected
       Failed to get current environment: rpc error: code = Unknown desc = default environment not found
       fix: Create one with `azd env new <name>` or select an existing one with `azd env select <name>`.
   (-) FOUNDRY_PROJECT_ENDPOINT set -- skipped (environment check failed or skipped)
   (-) manual env vars set -- skipped (no azd environment selected (cannot resolve agent.yaml variables))

4 passed, 1 failed, 8 skipped

To fix, run these commands in order:

  1. azd env new  -- create or select an azd environment
```

✅ Verified. Note the **cascade**: one failed check turns later checks into *skips*, not
failures. Always fix the topmost `(x)` first — the ones below it may not be real problems.
</details>

`doctor` is the best debugging tool in the toolkit. Run it whenever anything is odd — it
checks local config, auth **and** remote state, and names the exact env var or role that is
missing.

Exit codes matter in automation:

| Exit code | Meaning |
|---|---|
| `0` | At least one check passed and none failed. |
| `1` | At least one check failed. |
| `2` | All checks were skipped — dangerous in CI because nothing was evaluated. |

### 8. Region and quota

The samples default to **`gpt-5.4-mini`**, `GlobalStandard`, capacity `10`.

```bash
azd env set AZURE_LOCATION eastus2
azd env set AZURE_AI_DEPLOYMENTS_LOCATION eastus2   # optional: models in a different region
```

`AZURE_AI_DEPLOYMENTS_LOCATION` exists so you can keep the project near you while placing the
model where you actually have quota. If provisioning fails on capacity, change that one
variable rather than moving the whole project.

## ✅ Checkpoint

You should now be able to run this and see this inside any initialised agent project:

```bash
azd ai agent doctor
```

```text
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

If you see something else, jump to *If that didn't work* below.

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| `must contain 'template' field` | `azd` is older than 1.27.1 and silently installed an old agent extension. | Upgrade `azd`, then `azd extension upgrade --all`. |
| `azd ai agent` is not recognized | The extension is not installed or not reachable. | Run `azd extension install azure.ai.agents`. |
| Auth checks fail | `az login` or `azd auth login` is missing, or the wrong subscription is selected. | Run both sign-ins and `az account set --subscription <id>`. |
| `doctor` exits `2` | Every check was skipped. | Run it inside an initialized project after Lab 02 has provisioned resources. |

Everything else: [troubleshooting](../reference/troubleshooting.md).

## ✏️ Exercise

Predict the exit code before you run `azd ai agent doctor` in CI. If every check is skipped,
should the job pass?

<details>
<summary>Solution</summary>

No. `doctor` exits `2` when all checks are skipped. Treat that as a CI failure because no real
local, authentication or remote state was evaluated.
</details>

## → Next

[Lab 02 — Run your first agent locally](02-first-agent.md)

---

<sub>[← 🧪 Tutorial index](README.md) · [🧪 Tutorial index](README.md) · [Your first agent →](02-first-agent.md)</sub>
