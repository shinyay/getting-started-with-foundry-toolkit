# 🚚 Deploy modes reference

Foundry Toolkit can deploy an agent from source code, from a Dockerfile-built image, or from a
container image you already built somewhere else.

---

## Evidence used for this page

| Evidence | What was read |
|---|---|
| Captured `azd ai agent init --help` | deploy-mode, code settings and `--image` help text |
| Captured Bicep eject | `acr.bicep`, `acr-pull-role-assignment.bicep`, `resources.bicep`, `main.parameters.json` |
| Captured generated sample | `azure.yaml`, `Dockerfile`, `.azdignore`, `.dockerignore`, `requirements.txt` |
| Captured init environments | Bicep/Terraform `.azure/agent-framework-agent-basic-responses-dev/.env` files written by `azd ai agent init` |
| Live C# environment | `live/env-csharp.txt`, created via `azd env new` before provision/deploy |
| Existing docs | [`azure-yaml.md`](./azure-yaml.md), [`environment-variables.md`](./environment-variables.md), [`azd-cli.md`](./azd-cli.md), [`../guide-cli/README.md`](../guide-cli/README.md) |

> [!NOTE]
> Code mode was verified by a live Azure run of the C# sample. Container mode was **not** run
> live for this page; any claim about resources it creates is labelled as an inference from
> ejected Bicep or CLI help.

---

## Quick decision table

| Choose this mode | When | You provide | Toolkit creates ACR? | Main trade-off |
|---|---|---|---:|---|
| **Code mode** | Most Python/.NET agents; fastest path; no custom OS image needed | Source, dependencies, runtime and entry point | ❌ No | Less image-level control, but fewer resources and faster iteration. |
| **Container mode** | Native packages, custom base image, system tools, exact OS image, heavy build steps | Source plus `Dockerfile` | ✅ Inferred from ejected Bicep when ACR is included | More control, but registry cost and slower build/push loop. |
| **BYO-image** | CI already builds images; enterprise registry workflow; pinned digest/release promotion | Pre-built image URL | ❌ Help says ACR setup is skipped | Maximum control, but you own image build, patching and registry access. |

> [!TIP]
> Start with **code mode** unless you can name the thing that requires a container image.

---

## The three modes at a glance

```mermaid
flowchart TB
    S["Agent project"] --> M{"Deployment mode"}
    M -->|code| C1["ZIP source"]
    C1 --> C2["Foundry remote build"]
    C2 --> C3["No ACR"]
    M -->|container| D1["Dockerfile"]
    D1 --> D2["Build image"]
    D2 --> D3["Push to ACR"]
    M -->|--image| B1["Pre-built image URL"]
    B1 --> B2["No scaffolded Dockerfile or ACR setup"]
    C3 --> R["Foundry Agent Service"]
    D3 --> R
    B2 --> R
```

---

## Code mode

Code mode is selected by `codeConfiguration:` in `azure.yaml` and by init flags such as
`--deploy-mode code`, `--runtime` and `--entry-point`.

<details open>
<summary>✅ Verified generated `azure.yaml` excerpt</summary>

```yaml
services:
  agent-framework-agent-basic-responses:
    host: azure.ai.agent
    project: src/agent-framework-agent-basic-responses
    language: python
    codeConfiguration:
      runtime: python_3_13
      entryPoint: main.py
    uses:
      - ai-project
```
</details>

The reference manifest page documents the supported shape:

| Field | Verified values documented in this repo | Meaning |
|---|---|---|
| `runtime` | `python_3_13`, `python_3_14`, `dotnet_10` | Host runtime token. |
| `entryPoint` | `main.py`, `MyAgent.dll` | File the host executes. |
| `dependencyResolution` | `remote_build`, `bundled` | Whether Azure resolves dependencies or you upload bundled dependencies. |

The captured manifest did not include `dependencyResolution`, but existing docs show
`remote_build` as the default/golden-path value.

### Code mode resources

✅ Verified by the live C# sample run: code mode produced exactly two Azure resources after
`azd provision` + `azd deploy`.

```text
Name
----------------------
cog-m3ln646lhfgcu
cog-m3ln646lhfgcu/rdcs
```

The same live environment dump showed `AZURE_CONTAINER_REGISTRY_ENDPOINT=""`,
`AZURE_CONTAINER_REGISTRY_RESOURCE_ID=""`, `AZURE_AI_PROJECT_ACR_CONNECTION_NAME=""`,
`ENABLE_CAPABILITY_HOST="false"`, `ENABLE_HOSTED_AGENTS="true"`, and
`AZURE_FOUNDRY_NETWORK_MODE="none"`.

The init-time environments add one more subtle fact. Both captured Bicep and Terraform init
runs wrote `AZD_AGENT_SKIP_ACR="true"` before any provision happened:

```text
AI_AGENT_PENDING_PROVISION="project"
AZD_AGENT_SKIP_ACR="true"
AZD_RESOURCE_TOKEN_SALT="3bb69824"
AZURE_ENV_NAME="agent-framework-agent-basic-responses-dev"
AZURE_RESOURCE_GROUP="rg-agent-framework-agent-basic-responses-dev-3bb69824"
USE_EXISTING_AI_PROJECT="false"
```

> [!NOTE]
> `AZD_AGENT_SKIP_ACR="true"` is written by `azd ai agent init`, not by `azd provision`.
> The live C# run was bootstrapped with `azd env new` against an already-scaffolded in-repo
> sample, so `AZD_AGENT_SKIP_ACR` was absent there — but provision still created no ACR
> (`AZURE_CONTAINER_REGISTRY_ENDPOINT=""`, exactly two resources). Absence of this marker does
> **not** mean ACR will be created.
>
> Practical consequence: `azd env new` and `azd ai agent init` do not produce identical starting
> states. Init also wrote `AZD_RESOURCE_TOKEN_SALT`, `AI_AGENT_PENDING_PROVISION`, and
> `USE_EXISTING_AI_PROJECT`; a CI environment bootstrapped differently from a laptop can start
> from different azd variables even when both paths work.

| Resource | Created? | Evidence |
|---|---:|---|
| Foundry / AIServices account | ✅ | Live resource list plus `AZURE_AI_ACCOUNT_NAME="cog-m3ln646lhfgcu"`; account kind/sku verified as `AIServices`/`S0` in sibling cost evidence. |
| Foundry project | ✅ | Live resource list plus `AZURE_AI_PROJECT_NAME="rdcs"`. |
| Azure Container Registry | ❌ | Init `.env` wrote `AZD_AGENT_SKIP_ACR="true"`; the live env lacked that marker but still had `AZURE_CONTAINER_REGISTRY_ENDPOINT=""` and no registry in the resource list. |
| Application Insights | ❌ | No App Insights resource in the live resource list; sibling observability page confirms no `APPLICATIONINSIGHTS_CONNECTION_STRING`. |
| Storage / Cosmos DB / AI Search / VNet | ❌ for the basic run | Exactly two resources were listed; `ENABLE_CAPABILITY_HOST="false"` and `AZURE_FOUNDRY_NETWORK_MODE="none"`. |

> [!NOTE]
> The agent itself is deployed later by `azd deploy` as a project child/version, not as a
> separate top-level Azure resource in the provision list.

### Why code mode is usually first

| Area | Code mode behaviour |
|---|---|
| Build time | Usually fastest for simple agents because you upload source rather than building/pushing an image yourself. |
| Cold start | Platform builds the runtime image; you give up some image-level optimisations. |
| Control | Runtime and dependency strategy are declarative, not a full Dockerfile. |
| Cost | No registry cost for the basic path. |
| Operations | Fewer moving parts: account, project, agent deployment. |

---

## Container mode

Container mode uses a Dockerfile. The captured basic Python sample includes a Dockerfile even
though its manifest is code-mode shaped; the existing [`azure-yaml.md`](./azure-yaml.md) says the
Dockerfile is **container mode only — unused in code mode**. That means the file's presence is
not evidence that the verified code-mode live run used a container image.

<details open>
<summary>✅ Verified generated Dockerfile</summary>

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY . user_agent/
WORKDIR /app/user_agent

RUN if [ -f requirements.txt ]; then \
        pip install -r requirements.txt; \
    else \
        echo "No requirements.txt found"; \
    fi

EXPOSE 8088

# Precompile Python bytecode at build time so cold starts don't pay for it.
# These base images ship almost none of the standard library precompiled
# (~2% of stdlib .py files, on both Debian-slim and Azure Linux), so without
# this every new container recompiles hundreds of stdlib modules on first
# import. Dependencies are best-effort (`|| true`) so a stray unparsable file
# vendored by a third-party package can't fail the build; application code is
# compiled strictly so a syntax error surfaces at build time rather than at
# container start.
# PYTHONDONTWRITEBYTECODE is cleared for these commands only: it blocks
# bytecode *writes*, not *reads*, so images that set it still benefit.
RUN PYTHONDONTWRITEBYTECODE= python -m compileall -q $(python -c "import sysconfig as s; print(s.get_paths()['stdlib'], s.get_paths()['purelib'])") || true; \
    PYTHONDONTWRITEBYTECODE= python -m compileall -q .

CMD ["python", "main.py"]
```
</details>

### What the Dockerfile is doing

| Line / block | Purpose |
|---|---|
| `FROM python:3.12-slim` | Uses a slim Python base image for the container. |
| `COPY . user_agent/` | Copies the service source into the image. |
| `pip install -r requirements.txt` | Installs dependencies when the file exists. |
| `EXPOSE 8088` | Matches the local/runtime server port used by the agent host. |
| `compileall` for stdlib and site-packages | Best-effort precompile to reduce first-import work. |
| strict `compileall -q .` | Fails the build on syntax errors in application code. |
| `CMD ["python", "main.py"]` | Starts the generated agent. |

> [!TIP]
> The bytecode precompile is a useful cold-start optimisation. It moves work from container
> startup to image build time.

### Container mode resources

> [!NOTE]
> Inferred from the Bicep template, not observed in a live run. Container mode was not provisioned
> for this page, so the exact live resource count is unknown.

The captured Bicep shows what appears when `includeAcr` is true.

| Resource | Created? | Evidence |
|---|---:|---|
| Foundry / AIServices account | ✅ | `resources.bicep` creates `Microsoft.CognitiveServices/accounts@2025-06-01`. |
| Foundry project | ✅ | `resources.bicep` creates the nested `projects` resource. |
| Model deployments | ✅ when declared | `resources.bicep` loops over `deployments`. |
| Azure Container Registry | ✅ when `includeAcr` is true | `acr.bicep` creates `Microsoft.ContainerRegistry/registries@2023-07-01`. |
| ACR project connection | ✅ when `includeAcr` is true | `acr.bicep` creates a project `connections` child with `category: ContainerRegistry`. |
| AcrPull role assignment | ✅ when `includeAcr` is true | `acr.bicep` grants role id `7f951dda-4ed3-4680-a7ca-43fe172d538d`. |

<details>
<summary>✅ Verified ACR-related Bicep excerpts</summary>

```bicep
@description('Include an Azure Container Registry. Set true when any agent uses docker:.')
param includeAcr bool = false
```

```bicep
module acr 'acr.bicep' = if (includeAcr) {
  name: 'acr'
  params: {
    location: location
    tags: tags
    name: '${abbrs.containerRegistryRegistries}${resourceToken}'
    foundryAccountName: foundryAccount.name
    foundryProjectName: foundryAccount::project.name
    foundryProjectPrincipalId: foundryAccount::project.identity.principalId
    enableNetworkIsolation: enableNetworkIsolation
  }
}
```

```bicep
resource registry 'Microsoft.ContainerRegistry/registries@2023-07-01' = {
  name: name
  location: location
  tags: tags
  sku: {
    name: 'Premium'
  }
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    adminUserEnabled: false
    publicNetworkAccess: enableNetworkIsolation ? 'Disabled' : 'Enabled'
    zoneRedundancy: 'Disabled'
  }
}
```
</details>

### Container mode trade-offs

| Area | Container mode behaviour |
|---|---|
| Build time | Slower loop: build image, push image, then deploy. |
| Cold start | Best control: choose base image, precompile, install native packages. |
| Control | Highest control over OS packages, Python/.NET patch level, build tooling and entry command. |
| Cost | ACR can add registry cost. Captured Bicep uses Premium SKU when it creates ACR. |
| Operations | More moving parts: image build, registry access, AcrPull, connection metadata. |

---

## BYO-image mode

BYO-image mode is selected with `azd ai agent init --image <image-url>`.

<details open>
<summary>✅ Verified `--image` help text</summary>

```text
--image string              Pre-built container image URL (e.g., 'myacr.azurecr.io/agent:v1'). When set without --manifest, skips template/language selection, code scaffolding, Dockerfile generation, and ACR setup, and requires --agent-name. Incompatible with --deploy-mode code.
```
</details>

The help text also includes this example:

```bash
azd ai agent init --no-prompt --agent-name my-agent \
  --image myacr.azurecr.io/agents/my-agent:v1
```

### BYO-image resources

> [!NOTE]
> Inferred from CLI help and normal project provisioning, not observed in a live BYO-image run.
> The help text directly supports the Dockerfile/ACR rows; the account/project rows assume you
> are provisioning a new project rather than passing an existing one.

| Resource | Created by toolkit? | Evidence / confidence |
|---|---:|---|
| Foundry / AIServices account | ✅ if provisioning a new project | Inferred from normal project service provisioning, not live-observed for BYO-image. |
| Foundry project | ✅ if provisioning a new project | Inferred from normal project service provisioning, not live-observed for BYO-image. |
| Dockerfile | ❌ | Help text says Dockerfile generation is skipped. |
| ACR setup | ❌ | Help text says ACR setup is skipped. |
| Your registry/image | ❌ | You bring the image URL; creation is outside this init path. |

> [!CAUTION]
> BYO-image does not mean "no registry exists". It means the toolkit does not scaffold or set up
> one for you in this path. Your image still has to be reachable by Foundry at deploy/run time.

### BYO-image trade-offs

| Area | BYO-image behaviour |
|---|---|
| Build time | Fast from azd's perspective because the image already exists. |
| Cold start | Fully controlled by your image and build pipeline. |
| Control | Maximum control, including pinned base images and digest-based promotion. |
| Cost | Registry cost belongs to the registry you bring; toolkit skips its own ACR setup. |
| Operations | You own image patching, vulnerability scanning, authentication and retention. |

---

## `.azdignore`, `.dockerignore`, and what gets packaged

The captured generated source included both ignore files:

| File | Verified content | Used by |
|---|---|---|
| `.azdignore` | `.env.example` | azd packaging for the service source. |
| `.dockerignore` | `.venv`, `__pycache__`, `*.pyc`, `*.pyo`, `*.pyd`, `.Python`, `.env` | Docker build context in container mode. |

The CLI help says init also generates a default `.agentignore` for code-deploy ZIP packaging.
The captured project root `.gitignore` contained `.azure`; a root `.agentignore` was not present
in the captured Bicep project I inspected.

---

## Resource comparison

| Resource / artifact | Code mode | Container mode | BYO-image |
|---|---:|---:|---:|
| `codeConfiguration` in `azure.yaml` | ✅ | ❌ normally | ❌ |
| Source ZIP upload | ✅ | ❌ | ❌ |
| Dockerfile used | ❌ | ✅ | ❌ generated by toolkit |
| Pre-built image URL | ❌ | ❌ | ✅ |
| Foundry account | ✅ verified | ✅ inferred | ✅ inferred if not using existing project |
| Foundry project | ✅ verified | ✅ inferred | ✅ inferred if not using existing project |
| ACR created by toolkit | ❌ verified for basic code run | ✅ inferred when `includeAcr` is true | ❌ per help text |
| ACR connection | ❌ verified for basic code run | ✅ inferred from Bicep | ❌ setup skipped by init path |
| Registry cost | ❌ for basic code run | ✅ inferred if toolkit creates ACR | External to toolkit |
| Best for | First deployment and most agents | Custom runtime/image needs | Enterprise image pipelines |

---

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Expecting Dockerfile changes to affect code mode | Deploy ignores your Dockerfile | Use container mode or BYO-image. |
| Treating `AZD_AGENT_SKIP_ACR` as a provision output | Marker present after `init`, absent after `azd env new` | It is an init-time marker. Check `AZURE_CONTAINER_REGISTRY_ENDPOINT` and the resource list to verify whether ACR exists. |
| Expecting ACR in code mode | `AZURE_CONTAINER_REGISTRY_ENDPOINT=""` | That is expected for the verified basic code run; no ACR was created. |
| Using `--image` with `--deploy-mode code` | CLI rejects the combination | Pick BYO-image or code mode, not both. |
| Assuming BYO-image creates a registry | No generated ACR setup | Create/manage the registry yourself or use container mode. |
| Forgetting runtime/entry point in non-interactive code init | Init cannot resolve required values | Pass `--runtime` and `--entry-point`. |

---

## Command examples

<details open>
<summary>Illustrative commands</summary>

```bash
# Code mode: source upload, no ACR for the basic path
azd ai agent init --no-prompt \
  --deploy-mode code \
  --runtime python_3_13 \
  --entry-point main.py

# Container mode: use the service Dockerfile and registry-backed deployment
azd ai agent init --no-prompt \
  --deploy-mode container

# BYO-image: point at a pre-built image and skip scaffolded Dockerfile/ACR setup
azd ai agent init --no-prompt \
  --agent-name my-agent \
  --image myacr.azurecr.io/agents/my-agent:v1
```
</details>

---

## How this relates to `azd provision` and `azd deploy`

| Phase | Code mode | Container mode | BYO-image |
|---|---|---|---|
| `azd provision` | Creates account/project/model infrastructure; no ACR in verified basic run. | Inferred: creates account/project/model plus ACR pieces when ACR is included. | Inferred: creates account/project/model if not using an existing project; registry is external. |
| `azd deploy` | Uploads source/package and creates a new agent version. | Inferred: builds/pushes image, then creates a new agent version. | Inferred from `--image` help: uses the provided image URL to create a new agent version. |
| `azd down` | Deletes provisioned Azure resources for the environment. | Inferred: also removes registry resources if they were provisioned. | Inferred: does not own the external image registry. |

> [!NOTE]
> Inferred from help and ejected templates, not observed in a live run for container or BYO-image.
> The exact remote deploy implementation is preview behaviour. Verify with
> [`azd ai agent show --output json`](./azd-cli.md#show) and the diagnostics in
> [`troubleshooting.md`](./troubleshooting.md) for your project.

---

## Next

- 👉 [`infrastructure.md`](./infrastructure.md) — ejected Bicep/Terraform files and ACR modules
- 👉 [`azure-yaml.md`](./azure-yaml.md) — `codeConfiguration`, `container.resources`, `infra.provider`
- 👉 [`azd-cli.md`](./azd-cli.md) — `init`, `--deploy-mode` and `--image` flags
- 👉 [`environment-variables.md`](./environment-variables.md) — code-mode environment values, init markers and ACR outputs
- 👉 [`../guide-cli/README.md`](../guide-cli/README.md) — verified CLI walkthrough
- 👉 [`troubleshooting.md`](./troubleshooting.md) — provision/deploy diagnostics
