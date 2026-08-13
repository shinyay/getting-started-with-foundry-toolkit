# 🛠️ Lab 01 — Set up your machine

> ⏱️ **15 min** · 📋 **Requires:** Azure subscription · 💰 **$0** · ☁️ **Creates 0 Azure resources**

Go from a clean machine to a working toolchain: the right `azd`, five extensions, both
sign-ins, and `azd ai agent doctor` running.

## What you'll learn

- Install the `azd` version that can read current Foundry Toolkit manifests.
- Install the agent, inspector, projects and toolbox extensions.
- Sign in with both CLIs the labs use.
- Read `doctor` output — its three groups, its cascade of skips, and its exit codes.

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
> See [troubleshooting → version skew](../reference/troubleshooting.md#1-version-skew--the-one-that-bites-everyone).

Note that you do **not** need a local Python 3.13/3.14. `azd ai agent run` provisions its own
interpreter via `uv` (observed: it downloaded CPython 3.14.3 automatically).

> [!NOTE]
> **Command blocks in these labs assume `bash` or `zsh`.** Two substitutions cover `fish`:
>
> | These labs write | In `fish` |
> |---|---|
> | `echo $?` | `echo $status` |
> | `VAR=value azd …` | `env VAR=value azd …` |
>
> If you run **VS Code Insiders**, `code` is `code-insiders` in every command below.

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
azd version 1.30.0 (commit eea6db684821093daabd8bf357b6c9b636168abf) (stable)
```

The trailing `(stable)` is the release channel. If yours says anything else you are on a
pre-release build, which is worth knowing before you report odd behaviour.

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
<summary>✅ Verified output — captured 2026-08-14, everything already current (each <code>Upgrading …</code> line is shown once; see the note)</summary>

```text
Upgrade azd extensions (azd extension upgrade)
Upgrades the specified extensions on the local machine.

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

> [!NOTE]
> **Each `Upgrading <extension>` line is printed once or twice, and which ones double varies
> between runs.** Three consecutive runs on an unchanged machine emitted 8, 10 and 8 of those
> lines. They are ordinary output, not a redrawn progress line — the capture contains no
> cursor-movement escape, so every one of them stays in your scrollback. The block above shows
> each once. What matters is the `(-) Skipped:` line per extension, the `5 skipped` total and
> `SUCCESS`.

> [!NOTE]
> That is what a **terminal** shows. `azd` prints a live `Upgrading <extension>` line and
> overwrites it in place with the result, so you never see it settle. Redirect the same command
> to a file and those progress lines survive — twice each — because a file has no cursor to
> rewind. Neither form is wrong; do not be surprised when CI logs look busier than your screen.

> [!IMPORTANT]
> ✅ **Verified: there are five extensions, not four.** `azure.ai.connections` is easy to miss
> because nothing in the getting-started flow mentions it. Confirm yours with
> `azd extension list --installed`:
>
> ```text
> ID                     NAME                             STATUS       INSTALLED      LATEST         SOURCE
> ─────────────────────────────────────────────────────────────────────────────────────────────────────────
> azure.ai.agents        Foundry agents (Beta)            Up to date   1.0.0-beta.9   1.0.0-beta.9   azd
> azure.ai.connections   Foundry Connections (Beta)       Up to date   1.0.0-beta.4   1.0.0-beta.4   azd
> azure.ai.inspector     Foundry Agent Inspector (Beta)   Up to date   1.0.0-beta.3   1.0.0-beta.3   azd
> azure.ai.projects      Foundry Projects (Beta)          Up to date   1.0.0-beta.5   1.0.0-beta.5   azd
> azure.ai.toolboxes     Foundry Toolboxes (Beta)         Up to date   1.0.0-beta.5   1.0.0-beta.5   azd
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

Already signed in? Check without re-authenticating:

```bash
az account show --query "{sub:name, user:user.name}" -o table
azd auth login --check-status
```

```text
Sub                                       User
----------------------------------------  -------------------------
MCAPS-…                                   you@example.com

Logged in to Azure as you@example.com
```

Lab 02 asks for your **subscription ID**. You never have to read it — [Lab 02 § 3](02-first-agent.md#3-env--point-at-your-subscription)
pipes it straight into `azd`. To print it on its own:

```bash
az account show --query id -o tsv
```

> [!WARNING]
> **`-o table` drops columns whose key is `id` or `type` — capitalise the key.**
> `--query "{sub:name, id:id}" -o table` renders **only** `Sub`; `--query "{sub:name, Id:id}"`
> renders both. Verified 2026-08-12 by asking for the same value under three names —
> `{name:name, id:id, foo:id, type:name}` printed `Name` and `Foo` and silently dropped the
> other two. You meet the same behaviour with `type` in
> [Lab 02 § 5](02-first-agent.md#5-provision--create-azure-resources).

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

`azd ai agent doctor` is the best debugging tool in the toolkit: **13 checks** across local
config, authentication and remote state, each failure naming the exact env var, role or
command that fixes it.

**It reads a project, and you do not have one yet.** `azd ai agent init` in
[Lab 02](02-first-agent.md) creates the first one. So this section teaches you to *read*
`doctor`; there are seven states you will actually see, each in a different place:

| Output | Where you can actually get it |
|---|---|
| `1 passed, 1 failed, 11 skipped` — outside any project | the [Checkpoint](#-checkpoint) at the end of this lab |
| `4 passed, 1 failed, 8 skipped` — project, but no environment selected | [Lab 02 § 4](02-first-agent.md#4-doctor--check-before-you-spend-money) |
| `6 passed, 1 failed, 6 skipped` — scaffolded, before `provision`, **with `--local-only`** | [Lab 02 § 4](02-first-agent.md#4-doctor--check-before-you-spend-money) |
| `7 passed, 1 failed, 5 skipped` — the same point **without** `--local-only` | [Lab 04 § 3](04-add-tools.md#3-provision-and-set-the-model-name) |
| `9 passed, 1 failed, 3 skipped` — endpoint set, project unreachable | [Lab 03 troubleshooting](03-deploy.md#-if-that-didnt-work) |
| `10 passed, 1 failed, 2 skipped` — provisioned, before `deploy` | [Lab 02 § 5](02-first-agent.md#5-provision--create-azure-resources) |
| `11 passed, 0 failed, 2 skipped` — all green | [Lab 03 § 4](03-deploy.md#4-doctor--check-local-and-remote-state) |

> [!TIP]
> **Two of those rows are the same moment in the lifecycle.** The only difference is
> `--local-only`, which turns the five network checks into skips. Before you compare your
> counts against this table, check which flag you used.

How to read any `doctor` run:

1. **13 checks in three groups** — Local (8), Authentication (1), Remote (4).
   `--local-only` skips the five network ones.
2. **Every failure carries a `fix:` line with the exact command.** That is what makes `doctor`
   better than reading a stack trace.
3. **Skips come in three kinds, and the reason is always in the line.** *Not applicable*
   (`no toolbox resources declared…`), *cascading from a failure above* (`see check
   local.project-endpoint-set`), and *missing prerequisite* (`AZURE_AI_PROJECT_ID is not set in
   the current azd environment.`). Only the second kind clears itself when you fix the topmost
   `(x)`, so always fix that first — the Checkpoint below shows **one** failure producing
   **11** skips.
4. **A red `(x)` is not always wrong.** Before `azd provision`, `FOUNDRY_PROJECT_ENDPOINT`
   cannot exist. "Ready" means the Local group is clean *apart from the checks that depend on
   resources you have not created yet*.

Run it whenever anything is odd — it is faster than reading an error.

Exit codes matter in automation:

| Exit code | Meaning |
|---|---|
| `0` | At least one check passed and none failed. |
| `1` | At least one check failed. |
| `2` | All checks were skipped — dangerous in CI because nothing was evaluated. |

### 8. Region and quota

The samples default to **`gpt-5.4-mini`**, `GlobalStandard`, capacity `10`. Decide your region
now — you will set it in
[Lab 02 § 3](02-first-agent.md#3-env--point-at-your-subscription).

| Variable | Controls | Set it when |
|---|---|---|
| `AZURE_LOCATION` | where the Foundry project is created | always |
| `AZURE_AI_DEPLOYMENTS_LOCATION` | where the **model** is deployed | you have no model quota in `AZURE_LOCATION` |

`AZURE_AI_DEPLOYMENTS_LOCATION` exists so you can keep the project near you while placing the
model where you actually have quota. If provisioning fails on capacity, change that one
variable rather than moving the whole project.

> [!WARNING]
> **Do not try to set these yet.** `azd env set` writes into an azd environment, and the first
> environment is created by `azd ai agent init` in Lab 02. Run it before that and it refuses:
>
> ```text
> ERROR: no project exists; to create a new project, run `azd init`
> ```
>
> ✅ Verified 2026-08-12 on two machines — exit code `1`, and nothing is written to disk.
> **Ignore the CLI's advice to run `azd init`.** A bare `azd init` builds an `azd` project with
> no agent service in it. Lab 02 uses `azd ai agent init`, which is a different command.

## ✅ Checkpoint

Lab 01 creates **no Azure resources and no project**, so the thing to verify here is the
toolchain, not an agent. Three commands:

```bash
azd version                                              # ≥ 1.27.1
azd extension list --installed                           # five extensions, all "Up to date"
az account show --query "{sub:name, user:user.name}" -o table   # signed in, correct subscription
```

> [!CAUTION]
> **Do not paste bare `az account show` output into an issue, a PR or a chat.** It prints your
> tenant ID, subscription ID, tenant display name and tenant domain. The `--query` above shows
> what this checkpoint needs and nothing more. If you are asked for a subscription ID, send it
> deliberately — never as a by-product of a status check.

Then confirm `doctor` itself is reachable. Run it from a directory with no project in it —
which is where you are at the end of this lab:

```bash
azd ai agent doctor --local-only
```

<details open>
<summary>✅ Verified output — captured 2026-08-12, outside any project</summary>

```text
azd ai agent doctor

Local
   (✓) azd extension reachable
   (x) azure.yaml present and parseable
       Failed to get project config: rpc error: code = Unknown desc = no project exists; to create a new project, run `azd init`
       fix: Run from a directory containing `azure.yaml`, or initialize one with `azd init`.
   (-) azd environment selected -- skipped (azure.yaml check failed)
   (-) agent service in azure.yaml -- skipped (azure.yaml check failed)
   (-) FOUNDRY_PROJECT_ENDPOINT set -- skipped (environment check failed or skipped)
   (-) agent definition valid (per service) -- skipped (no agent services detected or upstream check blocked)
   (-) manual env vars set -- skipped (agent definition check failed or skipped)
   (-) Manifest toolboxes have endpoint env vars set -- skipped (no azd environment is selected (see check `local.environment-selected`).)

Authentication
   (-) authentication -- skipped (remote check excluded by --local-only)

Remote
   (-) Foundry project endpoint reachable -- skipped (remote check excluded by --local-only)
   (-) Developer has required role on Foundry project -- skipped (remote check excluded by --local-only)
   (-) Hosted agents are active -- skipped (remote check excluded by --local-only)
   (-) Manifest connections exist on Foundry project -- skipped (remote check excluded by --local-only)

1 passed, 1 failed, 11 skipped

To fix, run these commands in order:

  1. azd ai agent init  -- scaffold or refresh the agent project

Then re-run `azd ai agent doctor` to verify.
```
</details>

> [!IMPORTANT]
> **`1 passed, 1 failed, 11 skipped` is the correct result here — this lab does not end
> all-green.** The one failure is the absence of a project, and you have not created one yet;
> `azd ai agent init` in [Lab 02](02-first-agent.md) is what clears it. Note the cascade again:
> 11 skips from **one** real problem.
>
> `doctor` only reaches all-green once an agent is deployed, which is the end of
> [Lab 03](03-deploy.md#4-doctor--check-local-and-remote-state). If you want it green now,
> nothing is wrong with your machine — you are simply two labs early.

> [!TIP]
> The CLI contradicts itself in that output: the error body says to run `azd init`, while the
> `fix:` line says `azd ai agent init`. **Follow the `fix:` line.** A bare `azd init` scaffolds
> an `azd` project with no agent service in it, and `doctor` will still fail on the next check.

If you see something else, jump to *If that didn't work* below.

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| `must contain 'template' field` | `azd` is older than 1.27.1 and silently installed an old agent extension. | Upgrade `azd`, then `azd extension upgrade --all`. |
| `azd ai agent` is not recognized | The extension is not installed or not reachable. | Run `azd extension install azure.ai.agents`. |
| Auth checks fail | `az login` or `azd auth login` is missing, or the wrong subscription is selected. | Run both sign-ins and `az account set --subscription <id>`. |
| `doctor` exits `2` | Every check was skipped. | Run it inside an initialized project after Lab 02 has provisioned resources. |
| `1 passed, 1 failed, 11 skipped` | **Nothing is wrong.** You are not inside an agent project yet. | Expected at the end of this lab. `azd ai agent init` in [Lab 02](02-first-agent.md) clears it. |
| `doctor` is not all-green | Also expected. It cannot be until an agent is deployed. | Continue to [Lab 02](02-first-agent.md); all-green arrives in [Lab 03](03-deploy.md#4-doctor--check-local-and-remote-state). |
| `azd env set` → `ERROR: no project exists` | `azd env set` needs an azd environment, and none exists yet. | Expected in this lab. `azd ai agent init` in [Lab 02](02-first-agent.md) creates it; set your variables there. |

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
