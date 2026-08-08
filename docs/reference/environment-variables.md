# 🔑 Environment variables

Two different stores, and knowing which is which prevents most configuration confusion.

| Store | Path | Committed? | Set with |
|---|---|---|---|
| **azd environment** | `.azure/<env-name>/.env` | ❌ gitignored | `azd env set K V` |
| **Agent runtime** | injected by Foundry into the container | n/a | `environmentVariables` in `azure.yaml` |

```bash
azd env get-values            # dump everything
azd env set KEY value         # write
azd env list                  # list environments
azd env select <name>         # switch
```

---

## You must set these

| Variable | Example | Notes |
|---|---|---|
| `AZURE_SUBSCRIPTION_ID` | `e3e0bed3-…` | required before `provision` |
| `AZURE_LOCATION` | `eastus2` | project region |
| **`AZURE_AI_MODEL_DEPLOYMENT_NAME`** | `gpt-5.4-mini` | ⚠️ **not set for you** — see below |
| `AZURE_AI_DEPLOYMENTS_LOCATION` | `swedencentral` | optional; place models where you have quota |

> [!WARNING]
> **`AZURE_AI_MODEL_DEPLOYMENT_NAME` is the #1 first-run failure.**
> `azure.yaml` interpolates `${AZURE_AI_MODEL_DEPLOYMENT_NAME}` and the sample code requires it,
> but `azd provision` only writes the JSON blob `AI_PROJECT_DEPLOYMENTS`. Set it manually:
> ```bash
> azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
> ```

---

## Written by `azd provision`

Verified dump after a successful provision:

| Variable | Example | Meaning |
|---|---|---|
| **`FOUNDRY_PROJECT_ENDPOINT`** | `https://cog-xxx.services.ai.azure.com/api/projects/<proj>` | ⭐ the endpoint your code uses |
| `AZURE_AI_PROJECT_ID` | `/subscriptions/…/projects/<proj>` | full ARM resource id |
| `AZURE_AI_PROJECT_NAME` | `agent-framework-agent-basic-resp` | project name (truncated to 32 chars) |
| `AZURE_AI_ACCOUNT_NAME` | `cog-czn5ugi4jtvzs` | Cognitive Services account |
| `AZURE_OPENAI_ENDPOINT` | `https://cog-xxx.openai.azure.com/` | OpenAI-compatible endpoint |
| `AI_PROJECT_DEPLOYMENTS` | `[{"name":"gpt-5.4-mini",…}]` | JSON array of deployments |
| `AZURE_RESOURCE_GROUP` | `rg-…-b9dd475c` | |
| `AZURE_TENANT_ID` | `16b3c013-…` | |
| `AZURE_ENV_NAME` | `…-dev` | azd environment name |
| `ENABLE_HOSTED_AGENTS` | `true` | |
| `ENABLE_CAPABILITY_HOST` | `false` | |
| `AZURE_FOUNDRY_NETWORK_MODE` | `none` | `none` \| managed networking |
| `AZURE_FOUNDRY_MANAGED_ISOLATION_MODE` | `` | set for isolated networking |
| `AZD_AGENT_SKIP_ACR` | `true` | `true` in code deploy mode — no registry created |
| `AZD_RESOURCE_TOKEN_SALT` | `b9dd475c` | salt for generated resource names |
| `USE_EXISTING_AI_PROJECT` | `false` | `true` when you passed `--project-id` |
| `AI_AGENT_PENDING_PROVISION` | `""` | `"project"` before provisioning; cleared after |
| `AZURE_CONTAINER_REGISTRY_ENDPOINT` | `""` | populated only in container mode |
| `AZURE_CONTAINER_REGISTRY_RESOURCE_ID` | `""` | " |
| `AZURE_AI_PROJECT_ACR_CONNECTION_NAME` | `""` | " |
| `AZURE_AI_PROJECT_CONNECTION_NAMES` | `""` | comma list of project connections |

---

## Written by `azd deploy`

Per-service, uppercased with `-` → `_`:

```text
AGENT_<SERVICE_NAME>_NAME
AGENT_<SERVICE_NAME>_VERSION
```

Example for service `agent-framework-agent-basic-responses`:

```text
AGENT_AGENT_FRAMEWORK_AGENT_BASIC_RESPONSES_NAME     = agent-framework-agent-basic-responses
AGENT_AGENT_FRAMEWORK_AGENT_BASIC_RESPONSES_VERSION  = 1
```

---

## Injected into the running agent

Available inside the container **without** declaring them:

| Variable | Meaning |
|---|---|
| `FOUNDRY_PROJECT_ENDPOINT` | project endpoint |
| `FOUNDRY_MODEL_NAME` | model name (alternative to `AZURE_AI_MODEL_DEPLOYMENT_NAME`) |
| `AGENT_*` | agent identity/version |

> [!CAUTION]
> **Never declare `AGENT_*` or `FOUNDRY_*` in `environmentVariables`.** You will shadow the
> runtime's real values and get failures that look like nothing is wrong.

---

## Your own variables

```yaml
# azure.yaml
environmentVariables:
    - name: MY_FEATURE_FLAG
      value: "true"
    - name: MY_API_KEY
      value: ${MY_API_KEY}        # ← resolved from azd env at deploy time
```

```bash
azd env set MY_API_KEY sk-...      # → .azure/<env>/.env, gitignored
```

Secrets go in the azd environment, **never** literal in `azure.yaml`.

---

## Local-only development

For running `python main.py` directly (without `azd ai agent run`), copy the example file:

```bash
cd src/<project>
cp .env.example .env
```

```text
FOUNDRY_PROJECT_ENDPOINT="https://cog-xxx.services.ai.azure.com/api/projects/<proj>"
AZURE_AI_MODEL_DEPLOYMENT_NAME="gpt-5.4-mini"
```

`main.py` calls `load_dotenv()`, so this is picked up automatically. `.env` is gitignored.

---

## Telemetry marker

When a coding agent drives these commands, Microsoft asks for:

```bash
AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd ai agent show
```

Set it **inline**, never persisted — it identifies skill-driven traffic.

---

## Ports

| Port | Used by |
|---|---|
| **8088** | `azd ai agent run` — the agent server |
| **8087** | Agent Inspector UI (`--inspector-port`) |
| **5679** | `debugpy` (VS Code `F5` track) |

Override with `azd ai agent run -p 9000 --inspector-port 9001`.
