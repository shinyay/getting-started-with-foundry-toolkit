# 🚀 Lab 03 — Deploy and clean up

> ⏱️ **20 min** · 📋 **Requires:** [Lab 02](02-first-agent.md) · 💰 **~$0.02** · ☁️ **Uses the 2 Azure resources from Lab 02**

Deploy the local agent to Foundry, verify the hosted version, then destroy everything safely.

> [!NOTE]
> **Your agent will be called `agent-framework-agent-basic-responses`, not `hello-world`.**
> The verified blocks on this page were captured from this repository's own
> [`samples/python/01-hello-world`](../../samples/python/01-hello-world/), whose `azure.yaml`
> sets `name: hello-world`. [Lab 02](02-first-agent.md) instead initialises the *catalog*
> sample, which keeps its own name — and `init` never renames it. The commands, field names,
> ordering and timings below are unaffected; only the agent name differs. Re-run the whole page
> against the catalog sample and the name is the single substitution to make.

## What you'll learn

- Deploy a hosted agent version to Foundry.
- Inspect the deployed definition and managed identity.
- Invoke the cloud endpoint and use the Trace ID.
- Tear down with `--purge` so the Cognitive Services name is reusable.

## Steps

### 1. `deploy` — push to Foundry

```bash
azd deploy --no-prompt
```

<details open>
<summary>✅ Verified output — 2 min 21 s</summary>

```text
  hello-world: Packaging (Packaging code)
  hello-world: Publishing
  ai-project: Done [4s]
  hello-world: Deploying (Deploying hosted agent (code deploy)) [13s]
  hello-world: Deploying (Creating agent) [13s]
  hello-world: Deploying (Waiting for agent to become active) [47s]
  hello-world: Deploying (Polling agent status (1/30)) [57s]
  …
  hello-world: Deploying (Registering agent environment variables) [2m21s]
  hello-world: Done [2m21s]

- Agent playground (portal): https://ai.azure.com/nextgen/r/…/build/agents/…?version=1
- Agent endpoint (responses): https://cog-….services.ai.azure.com/api/projects/…/agents/…/endpoint/protocols/openai/responses?api-version=v1

SUCCESS: Your application was deployed to Azure in 2 minutes 21 seconds.
```
</details>

Deploy also injects **five** per-service env vars back into your environment:

```text
AGENT_AGENT_FRAMEWORK_AGENT_BASIC_RESPONSES_ENDPOINT
AGENT_AGENT_FRAMEWORK_AGENT_BASIC_RESPONSES_NAME
AGENT_AGENT_FRAMEWORK_AGENT_BASIC_RESPONSES_PROJECT_ENDPOINT
AGENT_AGENT_FRAMEWORK_AGENT_BASIC_RESPONSES_RESPONSES_ENDPOINT
AGENT_AGENT_FRAMEWORK_AGENT_BASIC_RESPONSES_VERSION
```

The pattern is `AGENT_<SERVICE_NAME_UPPERCASED>_<FIELD>`, with `-` replaced by `_`. These are
not decoration: `azd ai agent eval` resolves the agent it is about to evaluate from `_NAME` and
`_VERSION`, and reports which variable it read.

### 2. `show` — inspect what landed

```bash
azd ai agent show
```

<details open>
<summary>✅ Verified output — captured 2026-08-08</summary>

```text
FIELD                            VALUE
-----                            -----
ID                               hello-world:1
Name                             hello-world
Version                          1
Status                           active
Description                      A minimal Agent Framework agent hosted by Foundry.
Created At                       2026-08-08T08:04:38Z
Agent GUID                       0d1d9283-4140-4af1-913f-7af55b59ef29
Instance Identity Principal ID   60cc05ac-adbe-419f-b507-596392098a76
Instance Identity Client ID      60cc05ac-adbe-419f-b507-596392098a76
Blueprint Principal ID           9d1c3fb4-2988-4c66-abeb-a3ae47833276
Blueprint Client ID              c0b23f85-770b-4513-a1a4-870aa232e4b0
Blueprint Reference Type         ManagedAgentIdentityBlueprint
Blueprint Reference ID           hello-world-0d1d9
Metadata[enableVnextExperience]  true
Playground URL                   https://ai.azure.com/nextgen/r/…,cog-56mzb54ouruu6,rdpy/build/agents/hello-world/build?version=1
Endpoint (responses)             https://cog-56mzb54ouruu6.services.ai.azure.com/api/projects/rdpy/agents/hello-world/endpoint/protocols/openai/responses?api-version=v1
```
</details>

> [!NOTE]
> **`show` defaults to a table, but JSON is available.** `-o json` is a documented flag
> (`azd ai agent show --help` lists `-o, --output string  The output format (supported: json,
> table) (default "table")`), and it returns the same record with machine-readable field names —
> `instance_identity.principal_id`, `definition.environment_variables`, and so on. Use it
> whenever you want to pipe into `jq`:
>
> ```bash
> azd ai agent show --output json | jq -r '.instance_identity.principal_id'
> ```
>
> The JSON also carries the deploy defaults the table omits: `cpu: "0.5"`, `memory: "1Gi"`,
> `dependency_resolution: "remote_build"`, `runtime: "python_3_13"` and the negotiated
> `protocol_versions`.

> [!IMPORTANT]
> **The hosted runtime is not the interpreter you ran locally.** The deployed agent reports a
> different Python from the one `uv` fetched for `azd ai agent run`, so code that passes locally
> can still fail in the cloud. Details and the check command:
> [troubleshooting §10](../reference/troubleshooting.md#related-the-hosted-runtime-is-a-different-python-from-the-local-one).

Four things this proves:

1. **Auto-versioning.** `ID` is `hello-world:1`. Deploy again with the same `name:` and you get
   `:2` — a new *version*, not a second agent. See [versioning](../learn/08-versioning.md).
2. **Per-agent managed identity.** `Instance Identity Principal ID` is the principal you grant
   roles to when the agent needs an Azure resource. It is created per agent, not per project —
   which is why a 403 in the cloud is almost never a 403 on your laptop.
3. **There are two identities here, not one.** `Instance Identity` is the agent's own; the
   `Blueprint` pair belongs to the platform template that stamped it. Grant roles to the
   *instance* identity. See [identity model](../learn/07-identity-model.md).
4. **`Status: active` is the deploy's real success signal** — not the exit code of `azd deploy`.

> `azd ai agent list` does **not** exist. Use `show`, or the portal.

### 3. `invoke` — call the deployed agent

Same command as Lab 02, but no `--local`:

```bash
azd ai agent invoke "What is Microsoft Foundry?"
```

<details open>
<summary>✅ Verified output</summary>

```text
Agent:        hello-world (remote)
Message:      "What is Microsoft Foundry?"
Session:      (new -- server will assign)
Conversation: conv_eaca5918799f539a00b3TD0QQe5s60buMoiKD3UmhppXTeLIb0

Session:      537709a17b73653baa1318ad6343500cac9adaf379aa74256ecfc7148c0c1d6 (assigned by server)
Trace ID:     ca6fc70ab022ca7bfcb0042304cce482
[hello-world] Microsoft Foundry is Microsoft’s AI platform for building, customizing, and deploying AI apps and agents, especially with large language models. It brings together model access, tools, orchestration, evaluation, and deployment in one place.

In practice, it lets you:
- choose and use foundation models,
- build chatbots and AI agents,
- connect AI to your data and APIs,
- test, monitor, and deploy AI solutions.

It’s closely related to Azure AI and is aimed at helping teams move from prototypes to production.

Server responded in 14.242s (first byte: 7.357s)
```
</details>

Note the **Trace ID** — remote invocations are traced automatically; paste it into the portal
to see the span tree.

Live logs:

```bash
azd ai agent monitor
```

### 4. `doctor` — check local and remote state

```bash
azd ai agent doctor
```

Checks local config, authentication *and* remote state: endpoint reachability, your RBAC role,
and whether hosted agents are enabled. Run it first whenever anything is odd. Full green output
is in [Lab 01](01-setup.md#-checkpoint) — the block in
[§7](01-setup.md#7-verify-everything) is the `--local-only` run, which is *meant* to show one
failure before you provision.

> [!TIP]
> **`10 passed, 0 failed, 3 skipped` is usually a flake, not a problem.** The remote
> *"Hosted agents are active"* probe intermittently fails to get a token and is then counted as
> **skipped** rather than failed, so `doctor` still exits `0`. Re-run before investigating:
> [troubleshooting §7d](../reference/troubleshooting.md#7d-azuredeveloperclicredential-signal-killed).

Exit codes:

| Exit code | Meaning |
|---|---|
| `0` | At least one check passed and none failed. |
| `1` | Any check failed. |
| `2` | All checks skipped — dangerous in CI because nothing was evaluated. |

### 5. Optional verified content: `eval`

Evaluation is not the main path for this lab, but this verified block existed in the original
CLI guide and is retained here until it is rehomed to reference material.

```bash
azd ai agent eval generate --no-prompt \
  --gen-instruction "Generate 5 short factual questions a developer new to Microsoft Foundry would ask."
```

One of `--gen-instruction`, `--gen-instruction-file`, `--config`, or both `--dataset` and
`--evaluators` is **required** — a bare `eval generate` errors out.

Generated artifacts:

```text
src/<project>/
├── eval.yaml
├── datasets/smoke-core/smoke-core_dg.jsonl     ← LLM-authored test cases
├── evaluators/smoke-core/rubric_dimensions.json
└── .agent_configs/baseline/metadata.yaml
```

```yaml
# eval.yaml
name: smoke-core
agent:
    name: agent-framework-agent-basic-responses
    kind: hosted
    config: .agent_configs/baseline/metadata.yaml
dataset:
    name: smoke-core
    version: "1.0"
    local_uri: datasets/smoke-core
evaluators:
    - name: smoke-core
      version: "1"
      local_uri: evaluators/smoke-core/rubric_dimensions.json
options:
    eval_model: gpt-5.4-mini
max_samples: 15
```

The dataset is **LLM-authored from your agent's own description** — each row is a test
*intent*, not a hard-coded string:

```json
{
  "id": 1,
  "description": "Test whether the assistant answers a very short identity question at the
   correct level of specificity, using only what is supported by the assistant spec…"
}
```

Then run it:

```bash
azd ai agent eval run --no-prompt
```

<details open>
<summary>✅ Verified output — 3m 43s (captured 2026-08-09)</summary>

```text
Resolving eval context...
  Reading project configuration...
  Detecting agent service...
  Resolving Foundry project endpoint...
  Updated eval.yaml with current environment values
Eval run started
   Eval: eval_d201f33b88ff458fb5561a32b0e00bf2
   Run:  evalrun_8506c73aa76540f4a31fec68feca06ef
   Report: https://ai.azure.com/nextgen/r/…/build/evaluations/eval_…/run/evalrun_…
  (✓) Done  Eval run  (3m 43s)

Eval:       eval_d201f33b88ff458fb5561a32b0e00bf2
Run:        evalrun_8506c73aa76540f4a31fec68feca06ef
Name:       smoke-core
Status:     Completed
Created:    2026-08-09 01:03:15 UTC
Agent:      hello-world v1

Results:    15 total, 9 passed, 6 failed, 0 errored

Per-criteria results:
  smoke-core: 9 passed, 6 failed, 0 errored
```
</details>

> [!IMPORTANT]
> **9 of 15 is the real result, and 6 failures is not a broken sample.** The rubric `eval generate`
> wrote grades identity fidelity; the sample's instruction is only *"You are a friendly assistant"*.
> The gap between the rubric and the instruction **is** the finding. Chasing a green score by
> weakening the rubric would defeat the purpose.

The point is that you now have a **repeatable number** to move. Full walkthrough:
[Lab 06 — Evaluate](06-evaluate.md).

```bash
azd ai agent eval list      # history
azd ai agent eval show      # details of a run
azd ai agent eval update    # push edited evaluators/datasets
```

> [!TIP]
> `eval run` prints a **portal Report URL**. That page shows per-case pass/fail with the
> rubric's reasoning — far more useful than the terminal summary when you are trying to
> improve a score.

### 6. `down` — destroy everything with purge

```bash
azd down --force --purge
```

> [!CAUTION]
> **Do not omit `--purge`.** Cognitive Services accounts are soft-deleted for 48 hours by
> default, and the name stays reserved. Without `--purge`, a re-provision into the same name
> can fail even though the resource group is gone.

<details open>
<summary>✅ Verified output — 1 min 46 s</summary>

```text
Deleting all resources and deployed code on Azure (azd down)
Local application code is not deleted when running 'azd down'.

Listing Cognitive Services accounts in rg-rdpy...
Deleting model deployment gpt-5.4-mini on cog-56mzb54ouruu6...
Deleting resource group rg-rdpy...
Deleting resource group rg-rdpy (this can take several minutes)
…
Purging soft-deleted Cognitive Services account cog-56mzb54ouruu6...

SUCCESS: Your application was removed from Azure in 1 minute 46 seconds.
```
</details>


### 7. Command cheat sheet

| Command | Purpose |
|---|---|
| `azd ai agent init` | scaffold from sample / manifest / local src |
| `azd ai agent sample` | browse the catalog |
| `azd provision` | create the Foundry account + project |
| `azd ai agent run` | local server on `:8088` |
| `azd ai agent invoke [--local]` | send a message |
| `azd deploy` | package + push + version |
| `azd ai agent show` | deployed agent status/definition |
| `azd ai agent monitor` | stream logs |
| `azd ai agent doctor` | 13-point diagnostic |
| `azd ai agent eval` | generate / run / list / show / update |
| `azd ai agent optimize` | evaluate-and-improve instructions |
| `azd ai agent code` | manage agent source |
| `azd ai agent endpoint` | endpoint + agent-card config |
| `azd ai agent sessions` | session management |
| `azd ai agent files` | files in a hosted session |
| `azd ai agent delete` | delete a hosted agent |
| `azd down --force --purge` | destroy everything |

Full flag-by-flag surface: [azd-cli.md](../reference/azd-cli.md).

## ✅ Checkpoint

The lab is complete only after teardown purges the soft-deleted Cognitive Services account:

```bash
azd down --force --purge
```

```text
Deleting all resources and deployed code on Azure (azd down)
Local application code is not deleted when running 'azd down'.

Listing Cognitive Services accounts in rg-rdpy...
Deleting model deployment gpt-5.4-mini on cog-56mzb54ouruu6...
Deleting resource group rg-rdpy...
Deleting resource group rg-rdpy (this can take several minutes)
…
Purging soft-deleted Cognitive Services account cog-56mzb54ouruu6...

SUCCESS: Your application was removed from Azure in 1 minute 46 seconds.
```

If you see something else, jump to *If that didn't work* below.

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| `show` does not show `status: active` | Deployment is still activating or failed. | Wait a minute, run `azd ai agent doctor`, then inspect deployment output. |
| Remote invoke fails but local invoke worked | Cloud identity/config differs from your laptop. | Use `azd ai agent show --output json` and check `instance_identity` and env vars. |
| You tried `azd ai agent logs` | That command does not exist. | Use `azd ai agent monitor`. |
| Re-provision fails with a name conflict after teardown | You ran `azd down` without `--purge`. | Purge the soft-deleted Cognitive Services account or wait for retention to expire. |

Everything else: [troubleshooting](../reference/troubleshooting.md).

## ✏️ Exercise

Before you run `azd down`, predict what is lost and what stays on disk.

<details>
<summary>Solution</summary>

`azd down --force --purge` deletes the Azure resources, hosted agent, model deployment and the
soft-deleted Cognitive Services account reservation. Your local project files are not deleted.
That is why the verified output says: `Local application code is not deleted when running 'azd down'.`
</details>

## → Next

[Lab 04 — Add local tools](04-add-tools.md)

---

<sub>[← Your first agent](02-first-agent.md) · [🧪 Tutorial index](README.md) · [Add tools →](04-add-tools.md)</sub>
