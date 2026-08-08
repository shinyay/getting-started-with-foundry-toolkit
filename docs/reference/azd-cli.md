# ⚙️ `azd ai agent` CLI reference

Captured from `azd 1.30.0` / `azure.ai.agents 1.0.0-beta.9`.

```bash
azd ai agent --help
```

```text
Ship agents with Microsoft Foundry from your terminal. (Preview)

Available Commands:
  code        Manage agent source code. (Preview)
  delete      Delete a hosted agent.
  doctor      Diagnose problems with an azd ai agent project.
  endpoint    Manage agent endpoint and card configuration.
  eval        Create and run quick evals for an agent.
  files       Manage files in a hosted agent session.
  init        Initialize a new AI agent project. (Preview)
  invoke      Send a message to your agent.
  monitor     Monitor logs from a hosted agent.
  optimize    Evaluate and optimize AI agents.
  run         Run your agent locally for development.
  sample      Browse the curated catalog of agent samples and azd templates.
  sessions    Manage sessions for a hosted agent endpoint.
  show        Show the status of a hosted agent.
```

## Global flags

| Flag | Meaning |
|---|---|
| `-C, --cwd` | run as if in another directory |
| `--debug` | verbose diagnostics |
| `-e, --environment` | select azd environment |
| `--no-prompt` | non-interactive; fails if a value can't be resolved |
| `-o, --output` | `default` \| `json` |

---

## `init`

```text
agent init [<path>] [-m <manifest pointer>] [--src <source directory>]
```

| Flag | Notes |
|---|---|
| `-m, --manifest` | path/URI to an agent manifest **or** a sample's unified `azure.yaml` |
| `-s, --src` | init from local code (default target `src/<agent-id>`) |
| `--agent-name` | Foundry agent identity. **Ignored when adopting a sample's `azure.yaml`** |
| `--deploy-mode` | `code` \| `container` (default `code` for Python/.NET with `--no-prompt`) |
| `--runtime` | `python_3_13`, `python_3_14`, `dotnet_10` — required with `code` + `--no-prompt` |
| `--entry-point` | e.g. `app.py`, `MyAgent.dll` — required with `code` + `--no-prompt` |
| `--dep-resolution` | `remote_build` (default) \| `bundled` |
| `--protocol` | `responses`, `invocations`, `invocations_ws`, `activity` (repeatable) |
| `--model` | model to deploy (default `gpt-5.4-mini`) |
| `-d, --model-deployment` | reuse an existing deployment; **takes precedence over `--model`** |
| `-p, --project-id` | use an existing Foundry project resource ID |
| `--image` | BYO prebuilt image; requires `--agent-name`; incompatible with `--deploy-mode code` |
| `--infra[=bicep\|terraform]` | eject IaC into `./infra/` |
| `--force` | overwrite an in-tree manifest without prompting |

```bash
# from the catalog
azd ai agent init -m "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/responses/01-basic/azure.yaml"

# CI/CD, fully non-interactive
azd ai agent init --no-prompt --project-id "<resource-id>" \
  --deploy-mode code --runtime python_3_13 --entry-point app.py

# from your own code
azd ai agent init --src ./src/my-agent --agent-name my-unique-agent

# bring your own image
azd ai agent init --no-prompt --agent-name my-agent --image myacr.azurecr.io/agents/my-agent:v1
```

`init` also generates **`.agentignore`** (`.gitignore` syntax) controlling what enters the
code-deploy ZIP.

---

## `run`

| Flag | Default | Notes |
|---|---|---|
| `-p, --port` | `8088` | agent server port |
| `--inspector-port` | `8087` | Agent Inspector UI port |
| `--no-client` | | don't auto-open Inspector/Playground |
| `-c, --start-command` | | override `azure.yaml` + auto-detection |
| `--channel` | `emulator` | M365 Agents Playground, `activity` agents only |

```bash
azd ai agent run                 # opens the Inspector
azd ai agent run --no-client     # headless
azd ai agent run -p 9000
```

Readiness line: `Running on http://0.0.0.0:8088`.

---

## `invoke`

| Flag | Notes |
|---|---|
| `-l, --local` | target `localhost` instead of Foundry |
| `--port` | local port (default `8088`) |
| `-p, --protocol` | `responses` (default), `invocations`, `a2a` (**remote only**) |
| `--agent-endpoint` | full URL — invoke **without an azd project**; protocol inferred |
| `-f, --input-file` | send a file's contents as the body |
| `-s, --session-id` / `--conversation-id` | explicit overrides |
| `--new-session` / `--new-conversation` | discard the saved one |
| `--client-header` | repeatable `x-client-*` header; other names rejected; not supported with `a2a` |
| `--user-identity` | `x-agent-user-id` (local) / `x-ms-user-identity` (remote) |
| `--call-id` | `x-agent-foundry-call-id`, local only |
| `-t, --timeout` | seconds, default `1800`, `0` = none |

```bash
azd ai agent invoke --local "hello"
azd ai agent invoke "hello"
azd ai agent invoke -f ./payload.json
azd ai agent invoke --agent-endpoint "https://<acct>.services.ai.azure.com/api/projects/<proj>/agents/<name>/endpoint/protocols/openai/responses?api-version=v1" "hello"
```

Sessions and conversations are **persisted between calls**, so multi-turn works by default.
Use `--new-session` to start clean.

---

## `show`

No flags beyond globals. Use `--output json` for the full definition.

```bash
azd ai agent show
azd ai agent show --output json
```

> There is no `azd ai agent list`.

---

## `monitor`

| Flag | Default | Notes |
|---|---|---|
| `-f, --follow` | | stream live |
| `-l, --tail` | `50` | 1–300 trailing lines |
| `-t, --type` | `console` | `console` (stdout/stderr) \| `system` (container events) |
| `-s, --session-id` | | logs for one session |
| `--raw` | | raw SSE, no formatting |
| `--utc` | | UTC timestamps |

```bash
azd ai agent monitor -f
azd ai agent monitor -t system -l 200
```

---

## `doctor`

13 checks across Local / Authentication / Remote. No flags. Run it first for any problem.

---

## `eval`

| Subcommand | Purpose |
|---|---|
| `generate` | create a local eval suite for a deployed agent |
| `run` | execute `eval.yaml` |
| `list` | evaluations in the project |
| `show` | an eval definition, run history, or run details |
| `update` | push edited evaluators/datasets |

`generate` **requires** one of: `--gen-instruction`, `--gen-instruction-file`, `--config`, or
both `--dataset` and `--evaluators`.

```bash
azd ai agent eval generate --gen-instruction "Generate 5 factual questions about X."
azd ai agent eval run
azd ai agent eval show
```

Artifacts: `eval.yaml`, `datasets/<name>/*.jsonl`, `evaluators/<name>/rubric_dimensions.json`,
`.agent_configs/baseline/metadata.yaml`.

---

## `optimize`

Automatically improves agent instructions against evaluators.

| Flag | Notes |
|---|---|
| `-a, --agent` | service name from `azure.yaml`, or Foundry agent name |
| `-m, --eval-model` | **required** — model used for scoring |
| `--optimize-model` | **required** — reasoning model; gpt-5 family recommended |
| `--evaluator` | repeatable; required unless set in `--config` |
| `-d, --dataset` | local file or registered dataset |
| `--max-candidates` | ≥1, default `5` |
| `-c, --config` | YAML config instead of flags |
| `--no-wait` | submit and return |
| `--poll-interval` | seconds, default `10` |
| `-p, --project-endpoint` | Foundry project URL |

> Requires the **`responses`** protocol.

---

## `sample`

```bash
azd ai agent sample list --language python
azd ai agent sample list --language dotnetCsharp --output json
```

> `--language` uses short forms (`python`, `dotnetCsharp`); `--runtime` uses full tokens
> (`python_3_13`). They are not interchangeable.

---

## Other subcommands

| Command | Purpose |
|---|---|
| `code` | manage agent source code (preview) |
| `endpoint` | agent endpoint + agent-card configuration |
| `sessions` | list/manage sessions on a hosted endpoint |
| `files` | manage files inside a hosted session |
| `delete` | delete a hosted agent |

---

## Core `azd` commands you also need

| Command | Purpose |
|---|---|
| `azd auth login` | sign in |
| `azd env set K V` | write to `.azure/<env>/.env` |
| `azd env get-values` | dump environment |
| `azd provision` | create Azure resources |
| `azd deploy` | package + push the agent |
| `azd down --force --purge` | destroy everything (**always `--purge`**) |
| `azd extension upgrade --all` | keep extensions in sync |
