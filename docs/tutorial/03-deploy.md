# 🚀 Lab 03 — Deploy and clean up

> ⏱️ **20 min** · 📋 **Requires:** [Lab 02](02-first-agent.md) · 💰 **~$0.02** · ☁️ **Uses the 2 Azure resources from Lab 02**

Deploy the local agent to Foundry, verify the hosted version, then destroy everything safely.

> [!NOTE]
> **Your agent will be called `agent-framework-agent-basic-responses`, not `hello-world`.**
> The verified blocks on this page were captured from this repository's own
> [`samples/python/01-hello-world`](../../samples/python/01-hello-world/), whose `azure.yaml`
> sets `name: hello-world`. [Lab 02](02-first-agent.md) instead initialises the *catalog*
> sample, which keeps its own name — and `init` never renames it. The commands, field names,
> and ordering below are unaffected; timings vary by run. Substitute four strings, three of
> which come from your **azd environment name** rather than from the sample:
>
> | On this page | Where yours comes from |
> |---|---|
> | agent `hello-world` | your `azure.yaml` service name |
> | account `cog-56mzb54ouruu6` | random, new on every `provision` |
> | project `rdpy` | your azd environment name, cut to 32 characters |
> | group `rg-rdpy` | `rg-` + your azd environment name |
>
> That is why this page's project is a four-letter string and Lab 02's is 32 characters long:
> the environment was named `rdpy` when this was captured. See
> [Lab 02 § 5](02-first-agent.md#5-provision--create-azure-resources).
>
> A 2026-08-16 canary re-walk used Lab 02's catalog agent on `azd` 1.31.1 and the current
> extension row. Deploy, `show`, remote invoke, Trace ID correlation and the all-green
> `doctor` shape all reproduced; current timings live in
> [reference → Verified runs](../reference/README.md#current-row-canary--labs-0104).

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
<summary>✅ Verified output — 2 min 4 s, captured through a pty 2026-08-12</summary>

```text
Deploying services (azd deploy)


  Service                                Status        Duration
  ─────────────────────────────────────  ────────────  ──────────
  ● ai-project                             Done          3s
  ● agent-framework-agent-basic-responses  Done          2m4s
- Agent playground (portal): https://ai.azure.com/nextgen/r/…,rg-…,,cog-…,…/build/agents/…/build?version=1
- Agent endpoint (responses): https://cog-….services.ai.azure.com/api/projects/…/agents/…/endpoint/protocols/openai/responses?api-version=v1
  For information on invoking the agent, see https://aka.ms/azd-agents-invoke

Set up an evaluation suite to measure quality and impact in one step with azd ai agent eval generate

Next:
  azd ai agent show agent-framework-agent-basic-responses
  verify it's running

  azd ai agent invoke agent-framework-agent-basic-responses '<payload>'
  test the deployment


SUCCESS: Your application was deployed to Azure in 2 minutes 4 seconds.
You can view the resources created under the resource group rg-… in Azure Portal:
https://portal.azure.com/#@/resource/subscriptions/<your-subscription-id>/resourceGroups/rg-…/overview
```
</details>

> [!NOTE]
> **That table is live — the `Duration` column counts up and the row is rewritten in place.**
> While it runs the header reads `Service · Phase · Elapsed · Detail`; the final frame above is
> the summary azd leaves behind. Somewhere that cannot render cursor movement you get one line
> per step instead (`hello-world: Deploying (Polling agent status (1/30)) [57s]` and so on).
> Neither form is wrong; they are the same run rendered for different destinations. A failure
> renders in the same table, with `✗` and `Failed`.
>
> A redirect always produces the per-step form — but **a pty does not always produce the
> table.** The same command captured through `script -qec '…' /dev/null` from a process with no
> controlling terminal produced per-step lines, and still did so after the pty was given an
> explicit size with `stty rows 40 cols 200`. Why is not established. If you are capturing to
> compare against this block, confirm the capture actually contains escape sequences before
> trusting it.

> [!CAUTION]
> **Three of those lines carry identifiers.** The playground URL embeds a tenant identifier
> plus your group, account and project names; the final portal URL embeds your **subscription
> ID**. They are elided above for that reason. Do not paste this block into an issue unedited.

The `Next:` block, the `aka.ms` line and the `eval generate` suggestion only appear on a
terminal. See [`AGENTS.md`](../../AGENTS.md) for why this page captures through a pty.

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
`_VERSION`, and **names the variable it read** — run it and it tells you, before doing anything
else:

<details open>
<summary>✅ Verified output — the resolution header of <code>azd ai agent eval generate</code>, 2026-08-12</summary>

```text
Detected eval target:
  (✓) Service:        agent-framework-agent-basic-responses (azure.yaml)
  (✓) Agent:          agent-framework-agent-basic-responses (AGENT_AGENT_FRAMEWORK_AGENT_BASIC_RESPONSES_NAME)
  (✓) Version:        1 (AGENT_AGENT_FRAMEWORK_AGENT_BASIC_RESPONSES_VERSION)
  (✓) Kind:           hosted (azure.yaml (inline))
  (✓) Endpoint:       https://cog-….services.ai.azure.com/api/projects/… (FOUNDRY_PROJECT_ENDPOINT)
```
</details>

The parenthesis after each value is its **source**, which makes this the fastest way to find
out why a command is targeting the wrong agent.

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

Next:
  azd ai agent show agent-framework-agent-basic-responses
  confirm the deployed agent is healthy

  azd ai agent monitor --follow
  stream live logs from the agent
```
</details>

Note the **Trace ID** — remote invocations are traced automatically. Paste it into the portal
to see the span tree, or `grep` for it in the container's own logs: the same value appears
there as `x-request-id` and `trace-id` on the request that served this call.

Live logs — **`--follow` is what streams**:

```bash
azd ai agent monitor --follow
```

> [!WARNING]
> **Bare `azd ai agent monitor` does not stream.** It fetches the last 50 lines
> (`--tail`, default 50) and exits `0`. Measured: `timeout 25 … azd ai agent monitor` returned
> after a few seconds with 53 lines. The CLI's own `Next:` block above recommends the
> `--follow` form, and `--help` says so too: *"Use `--follow` to stream logs in real-time, or
> omit it to fetch recent logs and exit."*

> [!TIP]
> **The container log is where local and cloud visibly differ.** On your laptop
> [Lab 02 § 6](02-first-agent.md#6-run--the-local-loop) shows `DefaultAzureCredential` falling
> back to `AzureCliCredential`. In the hosted agent the same code logs
> `DefaultAzureCredential acquired a token from ManagedIdentityCredential` — the per-agent
> identity from § 2 doing its job. See [identity model](../learn/07-identity-model.md).
>
> Two more things the log shows that nothing else does: the agent calls the *same* model
> endpoint you called locally (`POST …/api/projects/…/openai/v1/responses → 200`), and a
> harmless `WARNING … Content type 'usage' is not supported yet. This is usually safe to
> ignore.`

> [!WARNING]
> Current unpinned Python images may also print `Failed to set up A365 OpenAI Agents
> instrumentation` followed by `ModuleNotFoundError: No module named 'agents'`. The same
> 2026-08-16 session immediately logged `Tracing configured successfully`, acquired its
> managed-identity token and completed the request with HTTP 200. This is an optional
> instrumentation warning; it does not mean this sample requires an Agent365-enabled tenant.
> See
> [troubleshooting § 21](../reference/troubleshooting.md#21-agent365-and-a365-messages-in-hosted-logs).

### 4. `doctor` — check local and remote state

```bash
azd ai agent doctor
```

Checks local config, authentication *and* remote state: endpoint reachability, your RBAC role,
and whether hosted agents are enabled. Run it first whenever anything is odd.

**This is the first point in the tutorial where `doctor` can go all-green.** Every earlier run
has a legitimate `(x)` in it: no project until [Lab 02](02-first-agent.md)'s `init`, and no
`FOUNDRY_PROJECT_ENDPOINT` until its `provision`.

<details open>
<summary>✅ Verified output — all green, after deploy, captured through a pty 2026-08-12</summary>

```text
azd ai agent doctor

Local
   (✓) azd extension reachable
   (✓) azure.yaml present and parseable
   (✓) azd environment selected
   (✓) agent service in azure.yaml
   (✓) FOUNDRY_PROJECT_ENDPOINT set
   (✓) agent definition valid (per service)
   (✓) manual env vars set
   (-) Manifest toolboxes have endpoint env vars set -- skipped (no toolbox resources declared in any service's agent.manifest.yaml.)

Authentication
   (✓) authentication

Remote
   (✓) Foundry project endpoint reachable
   (✓) Developer has required role on Foundry project
   (✓) Hosted agents are active
   (-) Manifest connections exist on Foundry project -- skipped (no connection resources declared in any service's agent.manifest.yaml.)

11 passed, 0 failed, 2 skipped

Next:
  azd ai agent show agent-framework-agent-basic-responses
  verify it's running

  azd ai agent invoke agent-framework-agent-basic-responses '<payload>'
  test the deployment
```
</details>

Each `(-)` states its own reason in the parenthesis, and here both say the same thing: this
sample declares no toolboxes and no connections, so there is nothing to check.

> [!WARNING]
> **A skip does not always mean "not applicable".** It can also mean "could not be evaluated,
> because something upstream failed" — and the parenthesis is how you tell the two apart. Break
> the project endpoint and the same two checks skip with a very different reason:
>
> ```text
> (-) Hosted agents are active -- skipped (Foundry endpoint did not respond (see check `remote.foundry-endpoint`).)
> ```
>
> Note that it names the upstream check. Always read the reason, and fix the topmost `(x)`
> first — see [Lab 01 § 7](01-setup.md#7-verify-everything), where one failure produces eleven
> skips.

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

> [!WARNING]
> **The command can stop waiting while its server jobs continue.** On 2026-08-16 evaluator
> generation finished in 32 seconds, but dataset generation reached the CLI's 11m16s polling
> limit. The command exited successfully, kept the job IDs and told the reader to run
> `azd ai agent eval run`; that command resumed the same dataset for another 1m20s before
> starting the evaluation. Do not delete the generated state or start over unless the CLI
> explicitly reports a failed job. See
> [troubleshooting § 7](../reference/troubleshooting.md#7-eval-generate-refuses-to-run).

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

The generated file above is the 2026-08-09 shape. The 2026-08-16 run also wrote
`agent.version: "1"`.

Generation behavior is version-sensitive. On the current row, the 15-row dataset and rubric
followed the supplied five-question **generation instruction**, not the agent description.
Each row is still a test *intent*, not a hard-coded string:

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
> **9 of 15 is the real result for this captured run, not a fixed pass target.** Its generated
> rubric grades identity fidelity while the sample says only *"You are a friendly assistant"*.
> The 2026-08-16 catalog-agent run generated different assets and finished **13 of 15** after
> the resume described above. Compare the same saved suite across agent changes; comparing two
> newly generated suites measures both the agent and a moving test.

With the generated suite saved, you now have a **repeatable number** to move. Full walkthrough:
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
<summary>✅ Verified output — 2 min 4 s, captured through a pty 2026-08-12</summary>

```text
Deleting all resources and deployed code on Azure (azd down)
Local application code is not deleted when running 'azd down'.


SUCCESS: Your application was removed from Azure in 2 minutes 4 seconds.
```
</details>

> [!NOTE]
> **That really is the whole output.** While it runs, `azd` rewrites a progress line in place
> — `Listing Cognitive Services accounts in rg-…`, `Deleting model deployment …`,
> `Purging soft-deleted Cognitive Services account …` — and none of it survives to the end.
> A capture that cannot render cursor movement keeps every one of those lines; one measured
> teardown wrote `Deleting resource group … (this can take several minutes)` **37** times.
> Teardown timing varies widely: 1 m 46 s, 2 m 4 s, 2 m 21 s, 2 m 40 s and 3 m 51 s across five
> measured runs. The current catalog-agent canary added **1 m 56 s**.

Teardown is not finished because the command said `SUCCESS`. Verify it:

```bash
az group exists -n <your-resource-group>
```

```text
false
```

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


SUCCESS: Your application was removed from Azure in 2 minutes 4 seconds.
```

Then confirm the group is actually gone — `SUCCESS` is the command's opinion, `false` is Azure's:

```bash
az group exists -n <your-resource-group>
```

```text
false
```

If you see something else, jump to *If that didn't work* below.

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| `show` does not show `status: active` | Deployment is still activating or failed. | Wait a minute, run `azd ai agent doctor`, then inspect deployment output. |
| `deploy` fails with `RESPONSE 404 … "Project not found"` | The project exists in ARM but the Foundry data plane will not serve it. Ignore the error's own hint — the agent definition is fine. | See the box below. |
| Remote invoke fails but local invoke worked | Cloud identity/config differs from your laptop. | Use `azd ai agent show --output json` and check `instance_identity` and env vars. |
| You tried `azd ai agent logs` | That command does not exist. | Use `azd ai agent monitor`. |
| `monitor` prints and exits instead of streaming | You omitted `--follow`. | `azd ai agent monitor --follow`. |
| Re-provision fails with a name conflict after teardown | You ran `azd down` without `--purge`. | Purge the soft-deleted Cognitive Services account or wait for retention to expire. |

> [!WARNING]
> **`SUCCESS: Your application was provisioned` does not guarantee a usable project.** Observed
> 2026-08-12: `azd provision` reported success twice, ARM reported `provisioningState:
> Succeeded` for the account *and* the project, the model deployment existed and
> `properties.endpoints["AI Foundry API"]` matched `FOUNDRY_PROJECT_ENDPOINT` exactly — and the
> data plane still answered `404 NotFound / "Project not found"` for over thirty minutes.
>
> `doctor` diagnoses this correctly and is worth running first:
>
> ```text
> (x) Foundry project endpoint reachable
>     Foundry returned HTTP 404 (endpoint is wrong or the project no longer exists).
>
> 9 passed, 1 failed, 3 skipped
> ```
>
> **But its suggested fix did not work here.** It offers `azd provision` to "create the missing
> Foundry resources"; nothing was missing, and re-running it changed nothing. What did work was
> `azd down --force --purge` followed by provisioning into a **new azd environment**. The cause
> is not known, so do not read one into it.

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
