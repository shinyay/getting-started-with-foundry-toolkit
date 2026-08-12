# 🧪 Lab 06 — Measure quality with an eval run

> ⏱️ **40 min** (≈25 min of it waiting on jobs) · 📋 **Requires:** [Lab 03](03-deploy.md) · 💰 **eval-model tokens — the priciest lab here** · ☁️ **Creates 2 Azure resources + 1 dataset + 1 evaluator**

Turn "it seems to work" into a number you can compare against.

> ✅ **Verified end-to-end on live Azure 2026-08-09.** Timings, paths, errors and scores below
> are from that run, then torn down. Where it contradicted our earlier docs, this page follows
> the run.

> [!NOTE]
> **Your agent will be called `agent-framework-agent-basic-responses`, not `hello-world`.**
> Like [Lab 03](03-deploy.md), the captures on this page come from this repository's own
> [`samples/python/01-hello-world`](../../samples/python/01-hello-world/), while
> [Lab 02](02-first-agent.md) initialises the catalog sample and keeps its name. Commands and
> field names are unaffected — substitute the agent name. Scores will differ on every run by
> design; see the note on the rubric below.

## What you'll learn

- That evaluation is **two commands with very different costs**, not one.
- Where eval files must live — the single most common way to get stuck.
- Why an `eval.yaml` you copied from somewhere else **cannot work**.
- How to read a bad first score correctly.

---

## The mental model

```text
azd ai agent eval generate     ← model WRITES a dataset + a rubric, registers them in the project
        ↓                        ~9 min · expensive · you do this once
azd ai agent eval run          ← agent is invoked per case, judge scores each one
        ↓                        ~4 min · repeat after every change
azd ai agent eval show         ← list past runs
```

Two things follow from this that catch people out:

1. **`generate` registers project-scoped assets.** The dataset and evaluator become resources
   *inside your Foundry project*. `run` only references them by name and version.
2. **So `eval.yaml` is not portable.** Copying one from another project — or from this
   repo — gives you a file full of names that do not exist where you are pointing it.

---

## Steps

### 1. Deploy something to measure

Evaluation invokes a **deployed** agent. Use the sample that is set up for it:

```bash
cd samples/python/04-eval
azd env new eval-demo
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini
azd provision      # measured 1m34s
azd deploy         # measured 2m41s
```

### 2. Understand the layout before you spend anything

`azd ai agent eval` resolves **every** path against the service's `project:` directory from
`azure.yaml` — not your shell's cwd, not the repo root.

```text
04-eval/
├── azure.yaml               services.hello-world.project: src/hello-world
└── src/hello-world/         ⬅ the "agent project root"
    ├── main.py
    ├── eval.yaml            ⬅ HERE, not in the repo root
    ├── datasets/
    ├── evaluators/
    └── .agent_configs/baseline/
```

The CLI announces its resolution on every invocation — read these three lines rather than
guessing:

```text
(✓) Project:        /…/04-eval/src/hello-world (azure.yaml service "hello-world" project path)
    Eval config:    /…/04-eval/src/hello-world/eval.yaml
    Baseline:       /…/04-eval/src/hello-world/.agent_configs/baseline/metadata.yaml
```

> [!CAUTION]
> **`--config` does not rescue a wrong layout.** A relative `--config` is *also* resolved
> against the project directory, so `--config eval.yaml` from the repo root produces the
> identical error. An absolute path gets past config-reading, but `local_uri` for the dataset
> and evaluator still resolve against the project directory — so it only moves the failure one
> step later.

### 3. Generate the suite — and expect to wait

```bash
azd ai agent eval generate --no-prompt \
  --gen-instruction "A minimal Agent Framework agent hosted by Foundry that answers questions about itself." \
  --eval-model gpt-5.4-mini --max-samples 15
```

<details>
<summary>✅ Verified output — 8m51s</summary>

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

   Portal:
     Dataset:   https://ai.azure.com/…/build/data/datasets/smoke-core/1.0
     Evaluator: https://ai.azure.com/…/build/evaluations/catalog/smoke-core/1
```
</details>

**8m51s — longer than provision, deploy and the eval run combined.** Budget for it. A model is
writing 15 test cases *and* deriving a seven-dimension weighted rubric from your one sentence,
and you pay eval-model tokens for all of it.

> [!IMPORTANT]
> **The rubric is generated from your `--gen-instruction`, so it is not a standard.** A
> different sentence yields different dimensions and weights. Treat the first generation as a
> draft to read and edit (`azd ai agent eval update`), not as an objective yardstick.

Two files now exist that did not before — read them, they are the whole substance of the lab:

| File | What it is |
|---|---|
| `datasets/smoke-core/smoke-core_dg.jsonl` | 15 rows of `{id, description, query, candidate_response}` |
| `evaluators/smoke-core/rubric_dimensions.json` | the weighted dimensions the judge applies |

### 4. Run it

```bash
azd ai agent eval run --no-prompt
```

<details>
<summary>✅ Verified output — 3m43s</summary>

```text
Eval run started
   Eval: eval_d201f33b88ff458fb5561a32b0e00bf2
   Run:  evalrun_8506c73aa76540f4a31fec68feca06ef
   Report: https://ai.azure.com/…/evaluations/eval_…/run/evalrun_…

  (✓) Done  Eval run  (3m 43s)

Name:       smoke-core
Status:     Completed
Agent:      hello-world v1
Results:    15 total, 9 passed, 6 failed, 0 errored
```
</details>

> [!NOTE]
> **`eval run` rewrites your `eval.yaml`.** It logs `Updated eval.yaml with current environment
> values` and adds `agent.version: "1"`. A dirty working tree afterwards is correct behaviour —
> the CLI is recording which version was actually measured.

### 5. Clean up

```bash
azd down --force --purge      # measured 2m53s
```

---

## ✅ Checkpoint

Before spending anything, prove the layout resolves the way the CLI will resolve it. From
`samples/python/04-eval`:

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

You should see:

```text
eval.yaml in project dir : True
dataset resolves         : True
evaluator resolves       : True
```

And after a real run:

```bash
azd ai agent eval show --no-prompt
```

```text
Runs:       1
Recent runs:
  Run ID                                    Status     Passed  Failed  Created
  evalrun_8506c73aa76540f4a31fec68feca06ef  Completed  9/15    6       2026-08-09 01:03:15 UTC
```

## 🔧 If that didn't work

| Symptom | Cause | Fix |
|---|---|---|
| `failed to read eval config ".../src/hello-world/eval.yaml"` | `eval.yaml` sits in the repo root. | Move it into the service's `project:` directory. A relative `--config` will not help. |
| `RESPONSE 404 UserError · "The evaluator smoke-core was not found"` | You reused an `eval.yaml` from another project. `local_uri` is a download cache, **not** an upload source. | Run `eval generate` in *this* project so the assets get registered. |
| `AzureDeveloperCLICredential: signal: killed` | Transient token-acquisition failure — unrelated to your config. | Re-run; it cleared on retry every time. Run `azd auth token -o json` first if it persists. |
| `generate` looks hung | It isn't. | Measured **8m51s**. Use `--no-wait` to get the terminal back and poll later. |
| Score is worse than you expected | Usually a real mismatch, not a bug — see below. | Compare the agent's instructions to the rubric's heaviest dimension. |
| `eval.yaml` shows up modified in `git status` | `eval run` writes back `agent.version`. | Expected. Commit it. |

---

## How to read a bad first score

Our run scored **9/15 — a 40% failure rate on a hello-world agent.** That is the honest result,
and diagnosing it is more instructive than a green run:

- The rubric's heaviest dimension (weight 10) is `identity_fidelity`, because the suite was
  generated from *"…answers questions about itself"*.
- But `main.py` says only *"You are a friendly assistant. Keep your answers brief."*
- The agent was **never told who it is**, so it improvises — and
  `unsupported_metadata_fabrication` marks it down for exactly that.

**The gap between those two sentences is the finding.** The number is not a grade on your
agent or on the toolkit; it measures the distance between what the agent was told and what it
is being judged on. A first score is a **baseline**. Chase the delta after a change, never the
absolute value.

## ✏️ Exercise

Without spending anything, decide which of these two changes lets you legitimately claim your
agent improved:

- **A.** Edit `main.py`'s instructions, `azd deploy`, then `azd ai agent eval run`.
- **B.** Re-run `azd ai agent eval generate` with a better `--gen-instruction`, then run.

<details>
<summary>Solution</summary>

**A.** The suite stays fixed, so re-running measures the same 15 cases against a new agent
version — a real A/B on a stable yardstick.

**B is measuring with a different ruler.** `generate` produces a new dataset *and* a new
rubric, so the two scores are not comparable. That may be the right thing to do, but you
cannot call the difference an improvement.

⚠️ **The trap inside A:** `eval run` pinned `agent.version: "1"`. After redeploying you are on
v2, so unless you update or remove that field you will re-measure the **old** version and
conclude your change did nothing.
</details>

## → Next

🧪 [Lab 07 — Container mode](07-container-mode.md) · 📖 or look things up in
[Reference](../reference/README.md)

---

<sub>[← MCP toolbox](05-mcp-toolbox.md) · [🧪 Tutorial index](README.md) · [Container mode →](07-container-mode.md)</sub>
