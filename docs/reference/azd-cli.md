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
  version     Prints the version of the application
```

> [!IMPORTANT]
> **`agent` is one of four `azd ai` namespaces.** This page documents `azd ai agent`, but the
> toolkit is broader — and each namespace is a **separate extension** that azd offers to
> install on first use:
>
> | Namespace | Extension | What it does |
> |---|---|---|
> | `azd ai agent` | `azure.ai.agents` | Scaffold, run, deploy, evaluate agents — this page |
> | `azd ai project` | `azure.ai.projects` | Foundry project + model-deployment management |
> | `azd ai inspector` | `azure.ai.inspector` | The local Agent Inspector UI (`:8087`) |
> | `azd ai toolbox` | `azure.ai.toolboxes` | Toolboxes: versioned bundles of tools behind one MCP endpoint |
>
> Typing a command from a namespace you don't have installed does **not** fail — azd prints
> `Command 'ai toolbox' was not found, but there's an available extension that provides it`
> and offers to install it. Convenient interactively; a hard stop in CI, where you should
> install extensions explicitly. See the [ecosystem map](./ecosystem.md).

### `azd ai toolbox` at a glance

| Command | Purpose |
|---|---|
| `toolbox create <name> --from-file <path>` | Create a toolbox and its **initial version** from JSON/YAML |
| `toolbox show <name>` | Show a version, **including its computed MCP endpoint** |
| `toolbox list` / `versions` | List toolboxes / inspect versions |
| `toolbox publish` | Set the **default version** |
| `toolbox connection` / `skill` | Manage connection-backed tools / skill references |
| `toolbox delete` | Delete a toolbox or a single version |

The shape matters: a toolbox is **versioned and immutable**, and `create` makes a version
rather than a mutable object. Changing tools produces a *new* version; nothing switches over
until you `publish` it. That is the same deploy-then-promote discipline agents themselves use,
applied to tools. Worked example: [`samples/python/03-mcp-toolbox`](../../samples/python/03-mcp-toolbox/).

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

Real output from a live deploy (`2026-08-08`, C# hello-world):

```text
FIELD                            VALUE
-----                            -----
ID                               hello-world:1
Name                             hello-world
Version                          1
Status                           active
Created At                       2026-08-08T07:45:06Z
Agent GUID                       4e7e0e00-af5c-462b-a40b-09657fc5e964
Instance Identity Principal ID   af65ac56-3c8b-4fc9-ba35-9cb1a37e52da
Instance Identity Client ID      af65ac56-3c8b-4fc9-ba35-9cb1a37e52da
Blueprint Principal ID           68c06f0f-7eee-48d0-8d6b-ddeb81f5c1bb
Blueprint Client ID              6371fa08-38ff-483f-aadf-98edb2ecb0af
Blueprint Reference Type         ManagedAgentIdentityBlueprint
Blueprint Reference ID           hello-world-4e7e0
Playground URL                   https://ai.azure.com/nextgen/r/...
Endpoint (responses)             https://<account>.services.ai.azure.com/api/projects/<proj>/agents/hello-world/endpoint/protocols/openai/responses?api-version=v1
```

This is the **only** command that shows the agent's **instance identity** — the principal your
container actually authenticates as, and the one you must grant roles to. `ID` is
`<name>:<version>`; `Status` must read `active` before an invoke will succeed. See
[identity & RBAC](./identity-and-rbac.md).

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

13 checks across Local / Authentication / Remote. Run it first for any problem.

| Flag | Purpose |
|---|---|
| `--local-only` | Skip remote (network-dependent) checks. Useful offline, behind a proxy, or for fast triage. |
| `--unredacted` | Show raw principal IDs, scope ARNs, and UPNs. **Redacted by default** — pass this only when you need the real IDs, and never paste the output into an issue unredacted. |

### Exit codes — the reason `doctor` is CI-usable

`doctor` is not just a human report; it exits meaningfully, so it works as a pipeline gate:

| Code | Meaning |
|---|---|
| `0` | At least one check passed and **none failed** |
| `1` | **Any** check failed |
| `2` | **All** checks were skipped (usually: nothing to check — wrong directory, or `--local-only` with everything remote) |

Note the asymmetry: `2` is *not* "worse than 1". It means *nothing was actually evaluated*,
which is arguably more dangerous in CI than a clean failure — a naive `if doctor; then` would
treat it as non-zero and stop, but a `[ $? -eq 1 ]` check would sail past it. Prefer:

```bash
azd ai agent doctor --local-only
case $? in
  0) echo "healthy" ;;
  1) echo "check failed"; exit 1 ;;
  2) echo "nothing was checked — wrong cwd?"; exit 1 ;;
esac
```

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

## The commands most guides never mention

The five-line table this section used to be was the single biggest gap in this reference.
`azd ai agent` exposes **15 top-level subcommands and 40 invocable commands**; the golden path
uses about a third of them. The rest are where the day-2 work lives — and three of these
families (`sessions`, `files`, `optimize`) have **no equivalent in the GUI at all**.

### `sessions` — the conversation is a real, addressable resource

A hosted agent does not just answer requests; it holds **sessions**, and a session has its own
lifetime, its own filesystem, and its own ID. This is what makes stateful agents debuggable.

| Command | Purpose |
|---|---|
| `sessions create [agent] [version]` | Open a session, optionally pinned to a **specific agent version** |
| `sessions list` | List sessions for an agent |
| `sessions show` | Details of one session |
| `sessions stop` | Stop a running session |
| `sessions delete` | Delete it |

```bash
azd ai agent sessions create my-agent 3          # pin the session to version 3
azd ai agent sessions create --session-id my-session
```

> [!TIP]
> Pinning a session to a version is the cleanest way to **A/B two agent versions** by hand:
> create one session on `3`, one on `4`, and invoke both. No redeploy, no traffic-splitting.

### `files` — a session has a filesystem

| Command | Purpose |
|---|---|
| `files upload [agent] [file]` | Push a file into the session |
| `files download [file]` | Pull one out |
| `files list` / `files stat` | List / metadata |
| `files mkdir` | Create a directory |
| `files delete` | Remove a file or directory |

```bash
azd ai agent files upload ./data/input.csv --target-path /data/input.csv
azd ai agent files download /data/output.csv --target-path ./output.csv
```

Most flags **auto-detect**: the agent is inferred when only one exists, and `--session-id`
defaults to the **last invoke session** — so `upload` then `invoke` then `download` works with
no IDs typed at all. That default is convenient and a trap: if you invoked something else in
between, "the last session" is not the one you think.

### `code download` — pull deployed source back out

| Flag | Purpose |
|---|---|
| `--version` | Download a specific version instead of latest |
| `--zip` | Save as a zip without extracting |
| `-d, --dest` | Destination (default `./<agent-name>/`) |

```bash
azd ai agent code download my-agent --version 3
azd ai agent code download my-agent --zip --dest ./backup
```

This is the recovery hatch for the classic "what is *actually* running in prod?" question, and
a genuine answer to configuration drift — you can diff deployed source against your working
tree. Marked **Preview**.

### `optimize` — machine-improved instructions

`optimize` runs a job that generates candidate agent instructions, scores them with evaluators,
and lets you apply or deploy the winner.

| Command | Purpose |
|---|---|
| `optimize` | Submit an optimization job |
| `optimize status` / `list` | Check / list runs |
| `optimize apply` | Apply the winning candidate **locally**, into your azd project |
| `optimize deploy` | Deploy a winning candidate as a **new agent version** |
| `optimize cancel` | Cancel a running job |

| Key flag | Purpose |
|---|---|
| `-m, --eval-model` | Model used for evaluation (**required**) |
| `--optimize-model` | Model doing the optimization reasoning — gpt-5 family recommended (**required**) |
| `--evaluator` | Repeatable; built-in or custom evaluator name |
| `--max-candidates` | Candidates to generate (default 5) |
| `--no-wait` | Submit and return immediately |

> [!IMPORTANT]
> `optimize apply` writes into your project — the improved prompt lands in your **source tree**
> so it can be reviewed and committed like any other change. That is the important design
> choice: optimization output is a **pull request**, not a hidden server-side mutation.

### `endpoint` — change the agent card without redeploying

| Command | Purpose |
|---|---|
| `endpoint show` | Current endpoint + agent-card configuration |
| `endpoint update [name] [--force]` | Update endpoint/card **without deploying a new version** |

`--force` skips confirmation for **breaking changes** — meaning this command can break clients,
which is why it asks. Relevant to A2A, where the agent card is how other agents discover you
(see [multi-agent](../learn/10-multi-agent.md)).

### `delete`

Deletes a hosted agent. Distinct from `azd down`, which destroys the whole environment's
infrastructure.

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
