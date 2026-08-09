# 04 · Evaluation

> ⏱️ **35 min** (most of it waiting) · 📋 **Requires:** [03 · MCP & Foundry Toolbox](../03-mcp-toolbox/) · 💰 **token + hosted-agent compute + eval-model calls** · ☁️ **Creates 2 Azure resources**

Deploy a target agent, generate an evaluation suite, run it, and read the score.

> ✅ **Verified end-to-end on live Azure 2026-08-09.** Every command, path, timing and number
> on this page came from that run. Where the run contradicted our earlier documentation, this
> page follows the run.

## What you'll learn

- Where eval files actually live — and why putting them in the repo root silently breaks.
- Why `eval.yaml` is **not portable between projects**.
- That `eval generate` and `eval run` are two separate, differently-priced steps.
- How to read a failing score without panicking about it.

---

## The layout that works

This is the single most common way to get stuck. `azd ai agent eval` resolves **every** path
relative to the service's `project:` directory in `azure.yaml` — *not* the directory you run
the command from, and *not* the repo root.

```text
04-eval/
├── azure.yaml                 ← services.hello-world.project: src/hello-world
└── src/hello-world/           ← ⬅ the "agent project root". Everything eval lives HERE.
    ├── main.py
    ├── requirements.txt
    ├── eval.yaml              ← ⬅ not in the repo root
    ├── datasets/smoke-core/
    ├── evaluators/smoke-core/
    └── .agent_configs/baseline/
```

The CLI states this itself when it starts:

```text
(✓) Project:        /…/04-eval/src/hello-world (azure.yaml service "hello-world" project path)
    Eval config:    /…/04-eval/src/hello-world/eval.yaml
    Baseline:       /…/04-eval/src/hello-world/.agent_configs/baseline/metadata.yaml
```

> [!CAUTION]
> **`--config` does not rescue a wrong layout.** A *relative* `--config` is still resolved
> against the project directory, so `--config eval.yaml` from the repo root fails with the
> identical error. Only an absolute path escapes it — and then the dataset and evaluator
> `local_uri` paths still resolve against the project directory, so it only moves the failure.
>
> ```text
> ERROR: failed to read eval config "/…/src/hello-world/eval.yaml":
>   open /…/src/hello-world/eval.yaml: no such file or directory
> ```

---

## Steps

### 1. Deploy the target agent

Evaluation measures a **deployed** agent, so it has to exist first.

```bash
cd samples/python/04-eval
azd env new eval-demo
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
azd provision       # 1m34s measured
azd deploy          # 2m41s measured
```

### 2. Generate the suite

The checked-in `eval.yaml`, dataset and evaluator are the **real artifacts from our verified
run**, kept so you can read them before spending anything. But they name assets registered in
*our* project, which does not exist any more.

> [!IMPORTANT]
> **`eval.yaml` is not portable.** `local_uri` is a *download cache*, not an upload source.
> `eval run` does **not** register local files. Pointing a fresh project at a borrowed
> `eval.yaml` fails:
>
> ```text
> RESPONSE 404: UserError
> "The evaluator smoke-core was not found"   messageFormat: "The evaluator {id} was not found"
> ```
>
> The dataset and evaluator must be registered **in the project you are evaluating against**.
> `eval generate` is what registers them.

```bash
azd ai agent eval generate --no-prompt \
  --gen-instruction "A minimal Agent Framework agent hosted by Foundry that answers questions about itself." \
  --eval-model gpt-5.4-mini --max-samples 15
```

<details>
<summary>✅ Verified output (8m51s)</summary>

```text
   Dataset:    smoke-core (1.0)
               /…/src/hello-world/datasets/smoke-core
   Evaluator:  smoke-core (1)
               /…/src/hello-world/evaluators/smoke-core/rubric_dimensions.json

   Evaluator dimensions (7):
     Weight  Dimension
     ──────  ─────────
         10  identity_fidelity
          6  scope_bound_self_knowledge
          6  cross_turn_consistency
          5  uncertainty_calibration
          5  general_quality
          4  unsupported_metadata_fabrication
          2  concise_direct_response
```
</details>

**This is the expensive step, and it is the one nobody budgets for.** It took **8m51s** —
more than double the eval run itself — because a model writes 15 test cases *and* invents a
seven-dimension rubric from your one-line instruction. You pay eval-model tokens for all of it.

Note the rubric is **generated from your instruction**, so it is not stable: a different
`--gen-instruction` produces different dimensions. It is a starting point to edit, not a
standard to measure against.

### 3. Run it

```bash
azd ai agent eval run --no-prompt
```

<details>
<summary>✅ Verified output (3m43s)</summary>

```text
Eval run started
   Eval: eval_d201f33b88ff458fb5561a32b0e00bf2
   Run:  evalrun_8506c73aa76540f4a31fec68feca06ef
   Report: https://ai.azure.com/nextgen/r/…/evaluations/eval_…/run/evalrun_…

  (✓) Done  Eval run  (3m 43s)

Name:       smoke-core
Status:     Completed
Agent:      hello-world v1
Results:    15 total, 9 passed, 6 failed, 0 errored

Per-criteria results:
  smoke-core: 9 passed, 6 failed, 0 errored
```
</details>

> [!NOTE]
> **`eval run` edits your `eval.yaml` in place.** It printed
> `Updated eval.yaml with current environment values` and added `agent.version: "1"`, pinning
> the run to the version that was deployed. Expect a dirty working tree afterwards — that
> diff is the CLI recording *what was actually measured*.

### 4. Read the result honestly

**9 of 15.** A hello-world agent fails 40% of a suite generated for it. That is the correct
and useful outcome, and it is worth sitting with:

- The rubric was generated from the sentence *"answers questions about itself"*, so it checks
  `identity_fidelity` and `unsupported_metadata_fabrication` — but `main.py` only says
  *"You are a friendly assistant. Keep your answers brief."* The agent was never told who it
  is, so it improvises, and the judge marks that as fabrication.
- **The gap between the two instructions is the finding.** The number is not a grade on the
  toolkit; it is a measurement of a real mismatch between what the agent was told and what it
  is being judged on.

That is the entire point of the exercise: a first eval score is a **starting baseline**, not a
verdict. Chase the delta after a change, never the absolute number.

```bash
azd ai agent eval show --no-prompt
```

```text
Eval:       eval_d201f33b88ff458fb5561a32b0e00bf2
Runs:       1
Recent runs:
  Run ID                                    Status     Passed  Failed  Created
  evalrun_8506c73aa76540f4a31fec68feca06ef  Completed  9/15    6       2026-08-09 01:03:15 UTC
```

### 5. Clean up

```bash
azd down --force --purge     # 2m53s measured
```

---

## ✅ Checkpoint

Confirm the layout is right **before** spending anything, from `samples/python/04-eval`:

```bash
python3 -c "
import os,yaml
proj=yaml.safe_load(open('azure.yaml'))['services']['hello-world']['project']
e=yaml.safe_load(open(os.path.join(proj,'eval.yaml')))
print('eval.yaml in project dir :', os.path.exists(os.path.join(proj,'eval.yaml')))
print('dataset resolves         :', os.path.isdir(os.path.join(proj,e['dataset']['local_uri'])))
print('evaluator resolves       :', os.path.exists(os.path.join(proj,e['evaluators'][0]['local_uri'])))
"
```

```text
eval.yaml in project dir : True
dataset resolves         : True
evaluator resolves       : True
```

Every path is joined to `project`, exactly as the CLI does it. If any line prints `False`, the
CLI will fail the same way.

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| `failed to read eval config ".../src/hello-world/eval.yaml"` | `eval.yaml` is in the repo root. | Move it into the service's `project:` directory. A relative `--config` will not help. |
| `The evaluator smoke-core was not found` (404) | You reused an `eval.yaml` from another project. `local_uri` does not upload. | Run `eval generate` in *this* project to register the assets. |
| `AzureDeveloperCLICredential: signal: killed` | Transient token-acquisition failure, not a config problem. | Re-run. It cleared on retry every time. `azd auth token -o json` first if it repeats. |
| `eval generate` seems hung | It is not — it is generating 15 cases and a rubric. | Measured **8m51s**. Use `--no-wait` if you need the terminal back. |
| Your `eval.yaml` changed after a run | Expected. `eval run` writes back `agent.version`. | Commit it — it records what was measured. |

## ✏️ Exercise

`main.py` says *"You are a friendly assistant. Keep your answers brief."* The rubric's
heaviest dimension (weight 10) is `identity_fidelity`. Without spending anything, predict:
would editing `main.py` alone raise the score on the **existing** suite?

<details>
<summary>Solution</summary>

**Yes — and this is the one change that legitimately moves the number.** The suite is already
registered in the project, so re-running measures the *same* 15 cases against a new agent
version. Editing instructions → `azd deploy` → `eval run` is a genuine A/B on a fixed
yardstick.

Two traps:
1. `eval run` pinned `agent.version: "1"`. After redeploying you are on v2 — update or remove
   that field, or you will re-measure the old version and conclude your change did nothing.
2. Re-running `eval generate` instead would produce a **different rubric and dataset**, so the
   scores would not be comparable. Change one side at a time.
</details>

## → Next

📖 [Reference](../../../docs/reference/) · or the guided version of this lab:
[Tutorial 06 · Evaluate](../../../docs/tutorial/06-evaluate.md)
