# 🧪 Lab 07 — Container mode: watch the image get built

> ⏱️ **25 min** · 📋 **Requires:** [Lab 03](03-deploy.md) · 💰 **~$0.10 + a Premium ACR** · ☁️ **Creates 3 Azure resources**

> ✅ **Verified end-to-end on live Azure — 2026-08-09.** Every timing, resource name shape,
> SKU and command output on this page was captured from a real run
> (`azd 1.30.0`, `azure.ai.agents 1.0.0-beta.9`, `eastus2`). Nothing here is illustrative.

## What you'll learn

Every lab so far deployed with `language: python` and never mentioned a container. That was a
convenience: **Foundry runs your agent in a container either way.** In *code mode* the platform
builds the image for you and hides it. In *container mode* the image becomes yours — you own the
`Dockerfile`, and an Azure Container Registry appears in your resource group to hold it.

By the end you will be able to:

- Read `azure.yaml` and say which deploy mode a project uses, in one line.
- Point at the exact resource, connection and role assignment that container mode adds.
- Explain why container mode has a **standing daily cost** that code mode does not.
- Decide which mode a given project should use.

Everything else — provision, deploy, invoke, down — is identical. That is the point.

## The one line that decides everything

```yaml
services:
  my-agent:
    project: src/my-agent
    host: azure.ai.agent
    language: docker          # ← container mode. `python` or `dotnet` here = code mode.
    docker:
      remoteBuild: true       # ← build in Azure, not on your laptop
```

`language: docker` is the switch. Nothing else in the manifest changes mode.

> [!NOTE]
> **`remoteBuild: true` means you do not need Docker installed.** The build runs in Azure
> Container Registry Tasks. This is how the run on this page was done — in a container with no
> Docker daemon available.

## Steps

### 1. Scaffold a container-mode sample

```bash
mkdir -p ~/foundry-labs/container && cd ~/foundry-labs/container
azd ai agent init -m agent-framework-agent-basic-responses --no-prompt
```

> [!IMPORTANT]
> ✅ Verified: `azd ai agent init -m <sample>` creates a **subdirectory** named after the sample.
> You land one level up from your project. `cd` into it before anything else:
> ```bash
> cd agent-framework-agent-basic-responses
> ```

Confirm the mode before you spend anything:

```bash
grep -A2 "language:" azure.yaml
```

✅ Expected — this is the real output from the verified run:

```yaml
        language: docker
        docker:
            remoteBuild: true
```

### 2. Look at the environment *before* provisioning

This is the comparison that makes the lab worth doing.

```bash
azd env set AZURE_SUBSCRIPTION_ID <your-sub-id>
azd env set AZURE_LOCATION eastus2
azd env get-values | sort
```

✅ Expected — 8 values, and **no registry anywhere**:

```text
AI_AGENT_PENDING_PROVISION="project"
AZD_AGENT_SKIP_ACR="false"
AZD_RESOURCE_TOKEN_SALT="9ca61abd"
AZURE_ENV_NAME="agent-framework-agent-basic-responses-dev"
AZURE_LOCATION="eastus2"
AZURE_RESOURCE_GROUP="rg-agent-framework-agent-basic-responses-dev-9ca61abd"
AZURE_SUBSCRIPTION_ID="e3e0bed3-d145-4f35-8f8e-328d8f55215a"
USE_EXISTING_AI_PROJECT="false"
```

`AZD_AGENT_SKIP_ACR="false"` is already present, **before** provisioning. azd has read the
manifest and decided a registry is needed. Nothing has been created yet.

### 3. Provision

```bash
time azd provision --no-prompt
```

✅ Expected — measured **2m39s** on the verified run:

```text
Foundry deployment complete
SUCCESS: Your application was provisioned in Azure in 2 minutes 39 seconds.
```

### 4. Diff the environment

```bash
azd env get-values | sort
```

Three new values appear that code mode never produces:

| Variable | Verified value shape |
|---|---|
| `AZURE_CONTAINER_REGISTRY_ENDPOINT` | `cr6kkz3uyx7e75m.azurecr.io` |
| `AZURE_CONTAINER_REGISTRY_RESOURCE_ID` | `…/Microsoft.ContainerRegistry/registries/cr6kkz3uyx7e75m` |
| `AZURE_AI_PROJECT_ACR_CONNECTION_NAME` | `cr6kkz3uyx7e75m-conn` |

The registry name is `cr` + the same resource token as the account (`cog-6kkz3uyx7e75m`), so the
two are visibly paired.

### 5. Deploy — this is where the image is built

```bash
time azd deploy --no-prompt
```

✅ Expected — measured **2m40s**, with the agent-status polling loop visible:

```text
  my-agent: Deploying (Creating agent) [1m28s]
  my-agent: Deploying (Deploying hosted agent) [1m28s]
  my-agent: Deploying (Waiting for agent to become active) [2m5s]
  my-agent: Deploying (Polling agent status (1/30)) [2m15s]
  my-agent: Deploying (Polling agent status (2/30)) [2m29s]
  my-agent: Deploying (Polling agent status (3/30)) [2m39s]
  my-agent: Deploying (Registering agent environment variables) [2m39s]
  my-agent: Done [2m40s]
SUCCESS: Your application was deployed to Azure in 2 minutes 40 seconds.
```

> [!NOTE]
> **`Polling agent status (n/30)`** is the container starting up. 30 polls is the ceiling — an
> agent that never becomes active fails here rather than hanging forever.

### 6. Prove the image exists

This is the payoff — in code mode there is nothing to look at.

```bash
az acr repository list --name <registry-name> -o table
```

✅ Expected — one repository, named `<project>/<env-qualified-agent-name>`:

```text
Result
-----------------------------------------------------------------------
agent-framework-agent-basic-responses/agent-framework-agent-basic-responses-agent-framework-agent-basic-responses-dev
```

And its tags:

```bash
az acr repository show-tags --name <registry-name> --repository <repo> -o tsv
```

✅ Expected — **the tag is a deploy timestamp, not `latest`**:

```text
azd-deploy-1786234162
```

> [!TIP]
> Each `azd deploy` adds a new `azd-deploy-<unix-seconds>` tag. This is your deploy history, and
> it is also why an ACR left running slowly accumulates storage cost. There is no automatic
> cleanup.

### 7. Invoke — identical to every other lab

```bash
azd ai agent invoke <agent-name> "Reply with exactly: container mode works"
```

✅ Expected — measured **11.399s** (first byte 4.928s) on a cold container:

```text
[agent-framework-agent-basic-responses] container mode works
Server responded in 11.399s (first byte: 4.928s)
```

### 8. Tear down

```bash
azd down --force --purge
```

✅ Expected — measured **2m5s**:

```text
Purging soft-deleted Cognitive Services account cog-6kkz3uyx7e75m...
SUCCESS: Your application was removed from Azure in 2 minutes 5 seconds.
```

## ✅ Checkpoint

You have finished this lab when all five are true:

| # | Check | Verified expected result |
|---|---|---|
| 1 | `grep "language:" azure.yaml` | `language: docker` |
| 2 | `az resource list -g <rg> --query "[].{Name:name,Sku:sku.name}" -o table` | includes a registry at **Premium** |
| 3 | `az acr repository list` | exactly one repository |
| 4 | `az acr repository show-tags` | a tag matching `azd-deploy-<digits>` |
| 5 | `azd ai agent invoke` | output begins `[<agent-name>]` |

The full verified resource list — note the SKU column:

```text
Name                                                Sku
--------------------------------------------------  -------
cog-6kkz3uyx7e75m                                   S0
cog-6kkz3uyx7e75m/agent-framework-agent-basic-resp
cr6kkz3uyx7e75m                                     Premium
```

> [!CAUTION]
> ✅ **Verified: the ACR is created at `Premium`, the most expensive SKU.** You did not choose
> this and azd does not ask. Premium bills a **fixed daily rate regardless of whether you push or
> pull anything** — so a forgotten container-mode environment costs money while doing nothing.
> This is the single strongest reason to run `azd down --force --purge` after this lab.
> See [cost](../reference/cost.md).

### How the agent is allowed to pull the image

Two objects you never asked for make this work. Both are worth seeing once, because when a
container-mode deploy fails, one of them is usually why.

**The connection** — how the project knows where the registry is:

```jsonc
{
  "name": "cr6kkz3uyx7e75m-conn",
  "properties": {
    "authType": "ManagedIdentity",         // ← no passwords anywhere
    "category": "ContainerRegistry",
    "target": "cr6kkz3uyx7e75m.azurecr.io"
  }
}
```

**The role assignment** — how it is permitted to pull:

```json
[{ "principalId": "7bf1c0e4-…", "role": "AcrPull", "type": "ServicePrincipal" }]
```

Read them together: the **project's managed identity holds `AcrPull` on the registry**. There is
no admin user, no username/password, and no key in your environment. If you ever bring your own
registry, these two objects are exactly what you must recreate.

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| `docker: command not found` | Local build attempted. | Set `docker: remoteBuild: true` — then no daemon is needed. |
| No ACR after `azd provision` | Not actually container mode. | `grep "language:" azure.yaml` must say `docker`. |
| `azd env get-values` shows no registry vars | Provision did not complete. | Re-run `azd provision --no-prompt`. |
| Deploy stops at `Polling agent status (30/30)` | The container never became healthy. | Test the image locally, or check `azd ai agent monitor`. |
| `az acr repository list` → auth error | You have no data-plane role on the registry. | `az acr login --name <registry>`; the *agent* uses `AcrPull` via managed identity, you do not inherit it. |
| Cost appears the day after you finish | ACR was not deleted. | `azd down --force --purge`, then `az group list -o table` to confirm zero. |

## ✏️ Exercise

Your team wants to switch an existing code-mode agent to container mode so they can install a
system package (`libgraphviz`) that `pip` cannot provide.

1. Which single line in `azure.yaml` changes?
2. Name the three environment variables that will appear after the next `azd provision`.
3. Your finance owner asks what the recurring cost impact is, in one sentence.

<details>
<summary>Solution</summary>

1. **`language: python` → `language: docker`**, plus a `docker: remoteBuild: true` block so no
   local daemon is required. You also now need a `Dockerfile` in the service's `project:` dir —
   that is where `apt-get install libgraphviz-dev` goes, which is the whole reason for the switch.

2. `AZURE_CONTAINER_REGISTRY_ENDPOINT`, `AZURE_CONTAINER_REGISTRY_RESOURCE_ID`,
   `AZURE_AI_PROJECT_ACR_CONNECTION_NAME`.

3. *"We gain a **Premium**-SKU Azure Container Registry that bills daily whether or not we deploy,
   plus storage that grows by one image tag per deploy with no automatic cleanup."*

   The subtlety worth flagging to your team: the cost is **per environment**, so a
   dev/staging/prod split triples it.
</details>

## → Next

- 👉 [Lab 08 — Observability](08-observability.md) — see what the container is actually doing.
- 👉 [Deploy modes](../reference/deploy-modes.md) — code vs container vs BYO image, side by side.
- 👉 [Cost](../reference/cost.md) — the Premium-ACR line item in context.
- 👉 [What Azure creates](../learn/06-what-azure-creates.md) — the resource footprint, conceptually.

---

<sub>[← Evaluate](06-evaluate.md) · [🧪 Tutorial index](README.md) · [Observability →](08-observability.md)</sub>
