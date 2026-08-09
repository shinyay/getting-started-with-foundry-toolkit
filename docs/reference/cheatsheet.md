# 🗂️ Cheatsheet

> **What this is:** a one-page quick-reference for flags, commands, and variables.
> **What this is not:** a tutorial or explanation — follow links for depth.
> For the full reference index see [README.md](README.md).

---

## The six verbs

| Step | Command | What it does |
|------|---------|--------------|
| 1 | `azd ai agent init -m ./azure.yaml` | Scaffold project from a sample or manifest |
| 2 | `azd provision` | Create Azure resources (Foundry project, model deployment) |
| 3 | `azd ai agent run` | Run agent locally with Inspector UI |
| 4 | `azd deploy` | Deploy agent to Foundry hosted runtime |
| 5 | `azd ai agent invoke "Hello!"` | Send a message to the deployed agent |
| 6 | `azd ai agent eval run` | Execute an evaluation run |

Details → [azd-cli.md](azd-cli.md)

---

## Subcommands (`azd ai agent …`)

The complete list — 15 subcommands, ✅ verified from `azd ai agent --help`.

| Subcommand | Purpose | Notes |
|------------|---------|-------|
| `init` | Initialize project | `-m <sample>` creates a **subdirectory** named after the sample |
| `run` | Local dev server | Opens Inspector automatically; `--no-client` to suppress |
| `invoke` | Send message (local or remote) | `-l/--local` for local target |
| `monitor` | Stream logs from a hosted agent | A session ID is required — auto-resolved from your last invocation, or pass `--session-id`. `--follow` to tail |
| `show` | Status of a hosted agent | Prints a **table, not JSON**, and exposes two identities — see [identity-and-rbac.md](identity-and-rbac.md) |
| `delete` | Remove a hosted agent or one version | `--version` deletes one version; `--force` for active sessions |
| `doctor` | Diagnose project problems | **13 checks** in 3 groups; a failure turns later checks into skips |
| `eval` | Create and run quick evals | `run` · `generate` · `list` · `show` · `update` |
| `code` | Manage agent source code | `download` |
| `endpoint` | Manage endpoint and agent-card config | `show` · `update` — changes the card without redeploying |
| `files` | Manage files inside a hosted session | `upload` · `download` · `list` · `stat` · `mkdir` · `delete` |
| `sessions` | Manage hosted sessions | `create` · `list` · `show` · `stop` · `delete` |
| `optimize` | Evaluate and optimize agents | `apply` · `deploy` · `list` · `status` · `cancel` |
| `sample` | Browse the curated sample catalog | `list` · `show` — a **curated subset**, not a full index |
| `version` | Print the extension version | |

> [!CAUTION]
> **Does not exist:**
> - `azd ai agent list` — ERROR: unknown command. There is no list verb; `show` reports on
>   the agent in the current project, not on every agent in the Foundry project.
> - `azd ai agent logs` — ERROR: unknown command (the command is `monitor`)

> [!NOTE]
> **Toolboxes are a separate command tree** — `azd ai toolbox`, from the `azure.ai.toolboxes`
> extension, not `azd ai agent`. Its verbs are `create` · `show` · `list` · `publish` ·
> `delete` · `versions` · `connection` · `skill`. That extension installs itself **on first
> use, mid-command**, which is why CI should pre-install it.

---

## Flags you will reach for

| Flag | Applies to | Meaning |
|------|-----------|---------|
| `--no-prompt` | All | Non-interactive; fails if a required value is missing |
| `-e, --environment` | All | Pick the azd environment explicitly |
| `-l, --local` | `invoke` | Target localhost instead of Foundry |
| `-p, --port <n>` | `run` | Agent server port (default 8088) |
| `--port <n>` | `invoke` | Local server port — **no `-p` shorthand here**, see the caution below |
| `--inspector-port <n>` | `run` | Inspector UI port (default 8087) |
| `--no-client` | `run` | Don't open Inspector/Playground browser tab |
| `-c, --start-command` | `run` | Override the startup command from `azure.yaml` |
| `--local-only` | `doctor` | Skip remote checks (offline-friendly) |
| `--unredacted` | `doctor` | Show raw principal IDs, scope ARNs and UPNs |
| `--no-wait` | `eval run` | Start the run, return immediately |
| `--config <f>` | `eval run` | Eval config YAML (default `eval.yaml`) |
| `--force` | `init` | Overwrite an input manifest already inside the generated `src` tree |
| `--force` | `delete` | Delete even if the agent has active sessions |
| `-m, --manifest` | `init` | Path or URI to an agent manifest, or a sample's `azure.yaml` |
| `--infra[=bicep\|terraform]` | `init` | Eject IaC from `azure.yaml` into `./infra/` |
| `--version <n>` | `invoke` | Invoke a specific agent version |
| `--version <n>` | `delete` | Delete one version only; the agent itself survives |
| `-f, --input-file` | `invoke` | Send file contents as the request body |
| `-p, --protocol` | `invoke` | `responses` (default), `invocations`, or `a2a` (a2a is remote-only) |
| `--new-conversation` | `invoke` | Discard saved conversation history |
| `--new-session` | `invoke` | Discard saved session, start fresh |
| `-t, --timeout <s>` | `invoke` | Request timeout in seconds, `0` for none (default 1800) |
| `--purge` | `azd down` | Hard-delete (see [Teardown](#teardown)) |

> [!CAUTION]
> **`-p` means two different things.** On `run` it is `--port`. On `invoke` it is `--protocol`.
> `azd ai agent invoke -p 8088` does not set a port — it sets the protocol to `8088` and fails.
> On `invoke`, spell `--port` in full. (✅ verified from `--help` on both commands.)

Details → [azd-cli.md](azd-cli.md)

---

## Environment variables

### Set for you by `azd provision`

| Variable | Example value |
|----------|--------------|
| `FOUNDRY_PROJECT_ENDPOINT` | `https://<account>.services.ai.azure.com/api/projects/<name>` |
| `AZURE_AI_PROJECT_ID` | Full ARM resource ID |
| `AZURE_OPENAI_ENDPOINT` | `https://<account>.openai.azure.com/` |
| `AZURE_RESOURCE_GROUP` | `rg-<env>` |
| `AZURE_SUBSCRIPTION_ID` | Subscription GUID |
| `AZURE_LOCATION` | e.g. `eastus2` |

### Set for you only after `azd deploy`

| Variable | Example value |
|----------|--------------|
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | `gpt-5.4-mini` |
| `AGENT_<NAME>_ENDPOINT` | Agent version URL |
| `AGENT_<NAME>_RESPONSES_ENDPOINT` | Full responses protocol URL |

> [!CAUTION]
> `AZURE_AI_MODEL_DEPLOYMENT_NAME` is **never set by provision** — it appears only after deploy.
> If your agent code reads it at local-dev time, you must `azd env set` it yourself.

> [!CAUTION]
> `AZURE_AI_PROJECT_ENDPOINT` **does not exist**. The correct key is `FOUNDRY_PROJECT_ENDPOINT`.

> [!CAUTION]
> **Reserved prefixes:** `FOUNDRY_*`, `AGENT_*`, `APPLICATIONINSIGHTS_CONNECTION_STRING` —
> declaring any of these in `azure.yaml` → `environmentVariables` causes deploy to fail with HTTP 400.

Details → [environment-variables.md](environment-variables.md)

---

## Ports

| Port | Service |
|------|---------|
| 8088 | Agent server (local) |
| 8087 | Inspector UI |
| 5679 | debugpy (Python debug attach) |

---

## `azure.yaml` — fields you type most

```yaml
name: my-project                  # azd project name
services:
  my-agent:
    project: ./src/my-agent       # path to agent source
    host: azure.ai.agent          # Foundry hosted runtime
    language: python              # python | dotnetCsharp | typescript | java
    docker:
      remoteBuild: true           # build image in ACR (container mode)
    uses:                         # wire Foundry connections
      - ai-project
    environmentVariables:         # injected at runtime
      MY_VAR: ${AZURE_ENV_NAME}
```

| Field | One-liner |
|-------|-----------|
| `project:` | Relative path to agent code directory |
| `host:` | Must be `azure.ai.agent` for Foundry hosting |
| `language:` | Short forms: `python`, `dotnetCsharp`, `typescript`, `java` — **not** the same vocabulary as `runtime` (`python_3_13`, `dotnet_10`, …) |
| `docker.remoteBuild:` | `true` → ACR Tasks builds the image, so you do **not** need Docker installed |
| `uses:` | Connects the service to Foundry project resources |
| `environmentVariables:` | Key-value pairs injected into the runtime container — reserved names fail with HTTP 400 |

Details → [azure-yaml.md](azure-yaml.md)

---

## Verified timings

Timings depend on region, SKU, and network. Canonical measurements live in one place:

→ [README.md#verified-runs](README.md#verified-runs)

---

## Failure signals worth memorising

| Symptom | Cause | Fix |
|---------|-------|-----|
| `invoke` exits 0 but response is empty | `invoke` **always exits 0** regardless of agent output | Never gate CI on exit code — assert on output content |
| Provision/deploy succeeds but variable is blank at runtime | Unset `${VAR}` in `azure.yaml` **silently expands to an empty string** — neither provision nor deploy warns | `azd env get-values` before you deploy; nothing downstream will tell you |
| HTTP 400 on deploy | `environmentVariables` uses a reserved name (`FOUNDRY_*`, `AGENT_*`, `APPLICATIONINSIGHTS_CONNECTION_STRING`) | Remove or rename the variable |
| `ERROR: unknown command "logs"` | Command does not exist | Use `azd ai agent monitor` |
| `ERROR: unknown command "list"` | Command does not exist | Use `azd ai agent show` |

Details → [troubleshooting.md](troubleshooting.md)

---

## Teardown

```bash
azd down --force --purge
```

Without `--purge` the Cognitive Services account is **soft-deleted** — it keeps its globally-unique name reserved (48 h) and blocks re-provisioning under the same name.

Details → [infrastructure.md](infrastructure.md)

---

*Versions are intentionally not repeated here — see [Fast facts](README.md#fast-facts) for the canonical source.*
