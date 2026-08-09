# 🧾 `azure.yaml` reference

The single contract between your code and Foundry. Everything `azd` does is derived from it.

---

## Field schema

The annotated example below shows one *valid* manifest. This table is the **schema** — what
each field accepts, whether you may omit it, and what happens if you do. Read this when you
are writing a manifest by hand; read the example when you want to see one whole.

> Types: `str`, `int`, `bool`, `map`, `list`. **Req** = required.
> Enum values are the complete accepted set unless marked *(open)*.

### Top level

| Field | Type | Req | Default | Accepted values / notes |
|---|---|---|---|---|
| `name` | `str` | ✅ | — | Project name. Lowercase, hyphens. Also seeds resource names. |
| `metadata.template` | `str` | ⬜ | — | Telemetry/provenance tag, e.g. `agent-framework-basic@1.0`. Free-form *(open)*. |
| `requiredVersions.azd` | `str` | ⬜ | unconstrained | semver range, e.g. `>= 1.30.0`. See [below](#requiredversions--read-this-one). |
| `infra.provider` | `str` | ⬜ | `microsoft.foundry` | `microsoft.foundry` \| `bicep` \| `terraform` |
| `services` | `map` | ✅ | — | Keyed by **service name**. One entry per agent. |

### `services.<name>`

| Field | Type | Req | Default | Accepted values / notes |
|---|---|---|---|---|
| `host` | `str` | ✅ | — | `ai.agent` for hosted agents. The key that makes it a Foundry agent at all. |
| `project` | `str` | ✅ | — | **Path to the source directory**, relative to `azure.yaml`. Must resolve or deploy fails. |
| `language` | `str` | ✅ | — | `python` \| `dotnetCsharp` \| `typescript` \| `java` — **short forms** (≠ `runtime`) |
| `name` | `str` | ⬜ | the service key | Remote agent name. Diverges from the key only if you set it. |
| `description` | `str` | ⬜ | — | Shown in the portal and the agent card. |
| `config` | `map` | ⬜ | — | Agent config block — see below. |

### `services.<name>.config`

| Field | Type | Req | Default | Accepted values / notes |
|---|---|---|---|---|
| `runtime` | `str` | ⬜ | inferred from `language` | **Full tokens**: `python_3_13`, `python_3_12`, `dotnet_9_0` … Not interchangeable with `language`. |
| `protocols` | `list` | ⬜ | `[responses]` | `responses` \| `invocations` \| `activity` \| `invocations_ws` |
| `instructions` | `str` | ⬜ | — | System prompt, for prompt-style config. |
| `model` | `str` | ⬜ | — | Model deployment name. |
| `environmentVariables` | `list` | ⬜ | `[]` | `- name: X` + `value: ${Y}`. **Never literal secrets.** |
| `codeConfiguration` | `map` | ⬜ | — | See [`codeConfiguration`](#codeconfiguration). |
| `container` | `map` | ⬜ | — | See [`container.resources`](#containerresources). |
| `instance_identity` | `str`/`map` | ⬜ | shared identity | Per-agent managed identity. **snake_case** — remote-side field. |
| `toolboxes` | `list` | ⬜ | `[]` | Toolbox references; each needs its endpoint env var set. |

### `codeConfiguration`

| Field | Type | Req | Default | Accepted values / notes |
|---|---|---|---|---|
| `entryPoint` | `str` | ⬜ | language default | e.g. `main.py`. |
| `dependencyResolution` | `str` | ⬜ | `remote_build` | `remote_build` \| `local` — where dependencies are resolved. |

### `container.resources`

| Field | Type | Req | Default | Accepted values / notes |
|---|---|---|---|---|
| `cpu` | `str`/`num` | ⬜ | platform default | e.g. `"1"`. |
| `memory` | `str` | ⬜ | platform default | e.g. `"2Gi"`. |

> [!IMPORTANT]
> The two fields most likely to break you are both **paths and tokens, not values**:
> `services.<name>.project` must resolve on disk (a renamed directory silently breaks deploy),
> and `language` vs `config.runtime` use **different vocabularies**. Everything else has a
> forgiving default; these two do not.

---

## Complete annotated example

This is the **verified** manifest produced by `azd ai agent init` from the basic Python sample,
with every field explained.

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/Azure/azure-dev/main/schemas/v1.0/azure.yaml.json

requiredVersions:                       # ← guards against version skew (see below)
    extensions:
        azure.ai.agents: '>=1.0.0-beta.4'

name: agent-framework-agent-basic-responses   # project name

services:
    # ─────────────── the agent ───────────────
    agent-framework-agent-basic-responses:
        project: src/agent-framework-agent-basic-responses   # path to source
        host: azure.ai.agent                                 # ← marks this a hosted agent
        language: python                                     # python | dotnet
        uses:
            - ai-project                                     # ← link to the project service

        codeConfiguration:                # only for --deploy-mode code
            dependencyResolution: remote_build   # remote_build | bundled
            entryPoint: main.py
            runtime: python_3_13                 # python_3_13 | python_3_14 | dotnet_10

        container:
            resources:
                cpu: "0.5"                # "0.25" | "0.5" | "1" | "2"
                memory: 1Gi               # 0.5Gi  | 1Gi   | 2Gi | 4Gi

        description: A basic Agent Framework agent hosted by Foundry.

        environmentVariables:             # injected into the running agent
            - name: AZURE_AI_MODEL_DEPLOYMENT_NAME
              value: ${AZURE_AI_MODEL_DEPLOYMENT_NAME}     # ← from azd env

        kind: hosted                      # hosted | prompt
        name: agent-framework-agent-basic-responses   # ← the Foundry agent identity

        metadata:
            tags:
                - Agent Framework
                - Responses Protocol

        protocols:
            - protocol: responses
              version: 2.0.0

    # ─────────────── the Foundry project + models ───────────────
    ai-project:
        host: azure.ai.project
        deployments:
            - name: gpt-5.4-mini
              model:
                  format: OpenAI
                  name: gpt-5.4-mini
                  version: "2026-03-17"
              sku:
                  name: GlobalStandard    # GlobalStandard | Standard | DataZoneStandard
                  capacity: 10            # ×1000 TPM

infra:
    provider: microsoft.foundry           # managed Foundry provisioning (no Bicep on disk)
```

---

## The two service kinds

| | `host: azure.ai.agent` | `host: azure.ai.project` |
|---|---|---|
| Represents | your agent | the Foundry account + project + model deployments |
| Creates | an agent version | `Microsoft.CognitiveServices/accounts` + `/projects` |
| Linked by | `uses: [ai-project]` | — |
| Required? | yes | yes, unless you bring an existing project via `--project-id` |

> A file **without** any `host: azure.ai.agent` service is not an agent project, and
> `azd ai agent` commands will refuse to run.

---

## `requiredVersions` — read this one

```yaml
requiredVersions:
    extensions:
        azure.ai.agents: '>=1.0.0-beta.4'
```

Samples **self-declare** the minimum extension they need. This is the mechanism that surfaces
version skew — and the reason an old extension fails with a confusing message rather than a
helpful one. If you hand-write a manifest, keep this block.

---

## camelCase local, snake_case remote

The **file** is camelCase. The **deployed API** is snake_case. Both are correct; do not
"fix" one to match the other.

| `azure.yaml` (local) | `azd ai agent show` (remote) |
|---|---|
| `codeConfiguration` | `code_configuration` |
| `entryPoint: main.py` | `entry_point: ["python","main.py"]` |
| `dependencyResolution` | `dependency_resolution` |
| `environmentVariables` (list) | `environment_variables` (map) |
| `protocols` | `protocol_versions` |
| `container.resources.cpu` | `cpu` |
| `container.resources.memory` | `memory` |

---

## `codeConfiguration`

Only meaningful with `--deploy-mode code`.

| Field | Values | Meaning |
|---|---|---|
| `runtime` | `python_3_13`, `python_3_14`, `dotnet_10` | **full token** — bare `python` is invalid |
| `entryPoint` | `main.py`, `MyAgent.dll` | file the host executes |
| `dependencyResolution` | `remote_build` *(default)* | Azure installs deps from `requirements.txt` |
| | `bundled` | you ship pre-resolved deps in the ZIP |

`remote_build` keeps uploads tiny and is right for almost everyone. Use `bundled` for private
package feeds or air-gapped builds.

---

## `container.resources`

Only three tiers are accepted:

| cpu | memory |
|---|---|
| `"0.25"` | `0.5Gi` |
| `"0.5"` | `1Gi` |
| `"1"` | `2Gi` |
| `"2"` | `4Gi` |

> `cpu` must be a **quoted string**. An unquoted `0.5` is a YAML float and will be rejected.

---

## `environmentVariables` — and what *not* to put there

```yaml
environmentVariables:
    - name: MY_FEATURE_FLAG
      value: "true"
    - name: AZURE_AI_MODEL_DEPLOYMENT_NAME
      value: ${AZURE_AI_MODEL_DEPLOYMENT_NAME}     # ${} interpolates from azd env
```

> [!WARNING]
> **Never declare `AGENT_*` or `FOUNDRY_*` variables here.** The runtime injects them
> (`FOUNDRY_PROJECT_ENDPOINT`, `AGENT_<SERVICE>_NAME`, `AGENT_<SERVICE>_VERSION`, …).
> Declaring them yourself shadows the real values and produces baffling failures.

> [!CAUTION]
> **Never put secrets in `azure.yaml`** — it is committed. Use:
> ```bash
> azd env set MY_API_KEY <value>     # → .azure/<env>/.env, gitignored
> ```
> then reference `${MY_API_KEY}`.

---

## `infra.provider`

| Value | Behaviour |
|---|---|
| `microsoft.foundry` | **default.** azd generates the ARM deployment; nothing on disk |
| `bicep` | you own `./infra/*.bicep` |
| `terraform` | you own `./infra/*.tf` |

To take control, *eject*:

```bash
azd ai agent init --infra            # eject Bicep
azd ai agent init --infra=terraform  # eject Terraform
```

Start managed. Eject only when you need custom networking, private endpoints or policy.

---

## `protocols`

```yaml
protocols:
  - protocol: responses      # responses | invocations | invocations_ws | activity
    version: 2.0.0
```

Multiple protocols are allowed. See [concepts §4](../learn/README.md#4-the-protocol-is-the-contract).

---

## Legacy `agent.yaml` / `agent.manifest.yaml`

Older samples ship an **AgentManifest** (`agent.manifest.yaml`, requires a `template:` field)
instead of a unified `azure.yaml`. `azd ai agent init -m` accepts both: given a manifest it
*generates* an `azure.yaml` for you. `agent.manifest.yaml` is a **seed** — it is not kept on
disk after init.

A standalone `agent.yaml` still works but emits a migration warning. New work should use the
unified `azure.yaml`.

---

## Companion files

| File | Purpose |
|---|---|
| `.agentignore` | excludes files from the **code-deploy ZIP** (`.gitignore` syntax) — generated by `init` |
| `.azdignore` | excludes files from azd packaging generally |
| `.dockerignore` | container mode only |
| `.env.example` | documents required runtime vars; copy to `.env` for bare `python main.py` |
| `Dockerfile` | container mode only — unused in code mode |

---

## Validate

```bash
azd ai agent doctor     # includes "azure.yaml present and parseable"
python3 -c "import yaml;yaml.safe_load(open('azure.yaml'))"
```

The `$schema` comment on line 1 also gives you completion and inline validation in VS Code
with the YAML extension.
