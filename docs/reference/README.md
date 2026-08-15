# 📖 Reference

Lookup tables. Read the [guides](../tutorial/02-first-agent.md) first; come back here for details.

Fifteen pages, grouped by what you are trying to do.

### ⚡ In a hurry

| Page | Use it for |
|---|---|
| [`cheatsheet.md`](cheatsheet.md) | **one screen** — every command, flag, port and variable ⭐ |
| [`faq.md`](faq.md) | **a direct answer** to one specific question ⭐ |

### 🧭 Start here

| Page | Use it for |
|---|---|
| [`glossary.md`](glossary.md) | **every term in one place** — read this if a word is unfamiliar ⭐ |
| [`ecosystem.md`](ecosystem.md) | the four products, their repos, and which docs to trust |
| [`troubleshooting.md`](troubleshooting.md) | 16 real failures with real fixes ⭐ |

### 📐 The contract — what you write

| Page | Use it for |
|---|---|
| [`azure-yaml.md`](azure-yaml.md) | **field schema** + every field annotated |
| [`environment-variables.md`](environment-variables.md) | what's set by whom, and what never to declare |
| [`azd-cli.md`](azd-cli.md) | all **40** `azd ai agent` commands + the other three `azd ai` namespaces |

### ☁️ The platform — what Azure does

| Page | Use it for |
|---|---|
| [`deploy-modes.md`](deploy-modes.md) | code vs container vs BYO-image, and when ACR appears |
| [`infrastructure.md`](infrastructure.md) | ejecting to Bicep or Terraform, and what lands |
| [`identity-and-rbac.md`](identity-and-rbac.md) | the **three identities**, and why local≠cloud 🔐 |

### 🏭 Running it for real

| Page | Use it for |
|---|---|
| [`observability.md`](observability.md) | `monitor`, logs, and adding tracing |
| [`cost.md`](cost.md) | what actually bills, and keeping it near zero 💰 |
| [`multi-agent.md`](multi-agent.md) | A2A details, agent cards, trust boundaries and what is *not* verified |
| [`sample-catalog.md`](sample-catalog.md) | all 37 upstream samples (24 Python + 13 C#) |

---

## Fast facts

> 🏷️ **This table is the canonical home for every version, port, tier and runtime in this repo.**
> Other pages may *use* these values in commands, but they must not *redefine* them. If a number
> here disagrees with a number elsewhere, this table wins — and the other page is a bug.
> CI enforces the version rows automatically.

| | |
|---|---|
| Minimum `azd` | **1.27.1** (this guide verified on 1.30.0) |
| Extensions (5) | `azure.ai.agents` `1.0.0-beta.9`, `azure.ai.connections` `1.0.0-beta.4`, `azure.ai.inspector` `1.0.0-beta.3`, `azure.ai.projects` `1.0.0-beta.5`, `azure.ai.toolboxes` `1.0.0-beta.5` |
| VS Code extension ID | `ms-windows-ai-studio.windows-ai-studio` |
| Ports | **8088** agent (`--port`) · **8087** Inspector UI (`--inspector-port`) · **5679** debugpy (AI Toolkit) |
| Default model | `gpt-5.4-mini`, `GlobalStandard`, capacity 10 |
| Runtimes | `python_3_13`, `python_3_14`, `dotnet_10` |
| Protocols | `responses` ⭐, `invocations`, `invocations_ws`, `activity` |
| CPU/memory tiers | `0.25/0.5Gi`, `0.5/1Gi`, `1/2Gi`, `2/4Gi` |
| Resources created | 1 CognitiveServices account + 1 project |
| Verified timings | see the table below |

---

## Newer than the verified toolchain

> 🏷️ **This section is the one home for toolchain drift.** Every other page states the
> toolchain it was *verified on* and must not be edited to claim a newer one. CI check 7
> enforces that, and exempts only this file.

Measured **2026-08-15**. The labs have **not** been re-walked, so every ✅ Verified block
elsewhere in this repository still describes the row it was captured on.

| | Verified on | Installed when measured |
|---|---|---|
| `azd` | 1.30.0 | **1.31.1** |
| agents extension | `1.0.0-beta.9` | **`1.0.0-beta.10`** |
| projects extension | `1.0.0-beta.5` | **`1.0.0-beta.6`** |
| what a clean install brings | 5 extensions | **3** — see below |

### The dependency set shrank

Installing `azure.ai.agents` into a scratch `AZD_CONFIG_DIR` on 2026-08-15 brought
`azure.ai.inspector` and `azure.ai.projects` and nothing else.

| Extension | Verified row | Today |
|---|---|---|
| `azure.ai.inspector` | dependency | dependency |
| `azure.ai.projects` | dependency | dependency |
| `azure.ai.connections` | dependency | **not installed** |
| `azure.ai.toolboxes` | on first use | not re-tested |

No upstream changelog mentions this. Whether `azure.ai.connections` returns on demand — the
way `azure.ai.toolboxes` does — is **not verified**, and it is the single most likely way a
lab breaks on the newer row.

A machine that has been used for other work may carry more than either number.
`azure.ai.routines`, `azure.ai.skills` and `microsoft.foundry` were all present on the
authoring machine and none of them came from installing `azure.ai.agents`.

### What actually changed

The 49 `azd ai …` captures in [`evidence/help/`](../../evidence/) were re-taken on each
step and diffed, rather than assumed:

| Step | Files changed of 49 | Meaning |
|---|---:|---|
| `azd` 1.30.0 → 1.31.1, extensions unchanged | **0** | the agent command surface lives entirely in the extension; upgrading core `azd` moves none of it |
| agents `beta.9` → `beta.10` | **3** | 21 lines, listed below |

| Capture | Change |
|---|---|
| `_root.txt` | two new subcommands: `azd ai agent pack` and `azd ai agent publish`, which build and publish a Teams app package for an activity agent |
| `init.txt` | `--infra` no longer ejects into `./infra/` wholesale — existing infrastructure is preserved and Foundry files are generated as a separate `infra/foundry` layer |
| `monitor.txt` | `monitor` now works **outside** an azd project, resolving the endpoint from `azd ai project set` or `FOUNDRY_PROJECT_ENDPOINT` |

`azure.ai.projects` moved as a **dependency** of the agents extension, not on its own — it
reported *No update available* when reached by name in the same run.

### What this means for a page you are reading

| Page states | Status |
|---|---|
| a flag, default or subcommand | still backed by `evidence/help/`, which deliberately stays on the verified row |
| an `--infra` outcome, or the `azd ai agent` subcommand list | **known stale** — see the table above |
| a timing, resource name or agent reply | unaffected; those are run outputs, not CLI surface |

One command was renamed in `azd` 1.31.0: `azd extension upgrade` became
`azd extension update`, with the old name kept as a working but **undiscoverable** alias.
[Lab 01](../tutorial/01-setup.md#3-install-the-agent-extension) was re-captured on 1.31.1
because it is the page whose job is to describe what you install *today*.

---

## Verified runs

Every number here came from a real run against live Azure, then destroyed. Nothing is estimated.

> **Where each number comes from.** `provision`, `deploy` and `down` are `azd`'s own
> `SUCCESS: … in N minutes M seconds` line. `invoke` is the CLI's `Server responded in …` line.
> `eval run` is its `(✓) Done  Eval run  (3m 43s)` line. `eval generate` prints **two** timers
> rather than a total — `Dataset generation (8m 35s)` plus `Evaluator generation (16 seconds)`,
> summed here as 8m51s.

| | Python `01-hello-world` | C# `01-hello-world` | Python `04-eval` | Python catalog `01-basic` | Python catalog `02-tools` | Python catalog `02-tools` re-run | Python catalog `02-tools` 3rd | C# catalog `02-tools` | C# catalog `02-tools` 2nd |
|---|---|---|---|---|---|---|---|---|---|
| Date | 2026-08-08 | 2026-08-08 | **2026-08-09** | **2026-08-12** | **2026-08-13** | **2026-08-13** (2nd) | **2026-08-14** (3rd) | **2026-08-14** (C#) | **2026-08-14** (C# 2nd) |
| `azd provision` | **1m20s** | **1m43s** | **1m34s** | **1m24s** | **1m21s** | **1m16s** | **1m23s** | **1m39s** | **1m24s** |
| `azd deploy` | **2m21s** | **3m1s** | **2m41s** | **2m3s** | **2m22s** (redeploy of unchanged code: **12s**) | **2m6s** (redeploy: 404, retry **14s**) | **2m12s** (redeploy: **15s**) | **2m24s** | **2m13s** |
| First `invoke` | **14.242s** (first byte 7.357s) | **6.877s** (first byte 3.622s) | **14.442s** (first byte 7.935s) | **14.995s** (first byte 7.830s) | **22.053s** (first byte 7.722s) | **20.555s** (first byte 7.296s) | **21.231s** (first byte 8.683s) | **9.587s** (first byte 3.960s) | **6.152s** (first byte 3.882s) |
| First `invoke --local` | — | — | — | **9.518s** (first byte 9.517s) | **11.863s** (first byte 11.863s) | **6.915s** (first byte 6.915s) | **9.196s** (first byte 9.196s) | **9.020s** (first byte 9.016s) | **9.115s** (first byte 9.114s) |
| `azd ai agent eval generate` | — | — | **8m51s** | — | — | — | — | — | — |
| `azd ai agent eval run` | — | — | **3m43s** → 15 cases, 9 passed, 6 failed | — | — | — | — | — | — |
| `azd down --force --purge` | **1m46s** | **1m45s** | **2m53s** | **1m46s** | **3m51s** | **failed (409)**, purged by hand | **1m45s** | **2m36s** | **4m1s** |
| Resources created | **2** | **2** | **2** | **2** | **2** | **2** | **2** | not measured | not measured |
| RG-scope role assignments | **0** | **0** | **0** | **0** | not measured | not measured | not measured | not measured | not measured |

The 2026-08-12 column is a **reproducibility re-run** on a byte-identical toolchain (`azd 1.30.0`,
all five extensions unchanged), following [Lab 02](../tutorial/02-first-agent.md) and
[Lab 03](../tutorial/03-deploy.md) exactly as written rather than using this repository's own
samples. Every timing landed within noise of the original. It also closed the repo's last
uncaptured tutorial block — `invoke --local` — and corrected four claims; see the
[changelog](../../CHANGELOG.md). Labs 04–10 were **not** re-run.

> [!IMPORTANT]
> **That re-run was captured through a redirect, and a redirect hides things.** A later
> hands-on walk of both labs in an interactive terminal found that `azd deploy` draws a
> rewritten table where a redirect emits per-step lines, that `azd down`'s progress lines never
> survive to the end of a terminal session, and that `Next:` blocks are printed only to a tty.
> The re-run above could not have caught any of that, because it compared a redirect against a
> redirect. **Capture through a pty — `script -qec '<command>' /dev/null` — before promoting a
> block to verified.** A pty is necessary but **not sufficient**: the same command captured
> that way from a process with no controlling terminal still produced the per-step form. Count
> the escape sequences in the capture (`grep -c $'\033' <file>`) — the good `deploy` capture
> had 514, the degraded one 9.
>
> **A third walk sharpened this.** Run from an agent with a real pty — `isatty()` true on all
> three descriptors, a controlling terminal, an 80×24-plus window — `azd ai agent init` still
> announced *"Continuing because `--no-prompt` was specified"* and asked nothing, with no such
> flag on the command line. Under `script -qec` too. **Interactive output cannot be captured
> that way at all**, only non-interactive output, and only some of that faithfully: `invoke`
> matched a human's capture escape-for-escape (5 and 5), while `provision` did not (8 against
> 39).
>
> **A fourth walk made it per-command.** Counting cursor-movement escapes
> (`grep -o $'\033\[[0-9]*[ABKJ]' <file> | wc -l`) across a human-driven walk and two
> agent-driven ones shows the split is not a matter of degree — it is a property of the
> command. `doctor`, `show` and `invoke` draw no live UI, and their captures are **identical**
> either way; `init` (1463 → 0), `provision` (488 → 0), `deploy` (768 → 0) and `down` (665 → 0)
> lose theirs entirely. **Captures of those four from an agent must not back a verified
> block**; the other three may.
>
> **A pty is still not a faithful transcript for `monitor`.** Its output is streamed through
> the caller's terminal width, so long lines are broken **mid-token** — a 2026-08-14 capture
> split `traceparent:` across two lines as `traceparen` / `t: ,`. It also interleaves its own
> `status` lines out of chronological order, printing a `15:51:05` line among `15:48:17` ones.
> Neither is an artefact you may silently repair: quote the wrap as captured, or say in the
> `<summary>` that lines were rejoined.

The 2026-08-13 column is the [Lab 04](../tutorial/04-add-tools.md) walk — the same lifecycle
with a tool-calling agent, run interactively and captured through a pty. Its `invoke` numbers
are the slowest in the table because the request makes two model round-trips instead of one,
and the first one paid a container cold start. The tool itself took **0.000173 s**.

The **re-run** column is that same lab walked again from an empty directory on the same day
and the same toolchain, to check the page rather than the product. Every timing landed within
noise, and the tool-call evidence — the `gen_ai.tool.definitions` attribute, the
`finish_reasons` distribution, the span-dump structure — reproduced exactly. Two things did
not, and both are now documented in the lab: an `azd deploy` of unchanged code failed once
with `404 / Project not found` against a project that demonstrably existed, and
`azd down --force --purge` failed at the purge step with `409 RequestConflict`, deleting the
resource group but leaving the account soft-deleted. **`az group exists` returned `false`
while that was still true**, which is why the lab now checks
`az cognitiveservices account list-deleted` as well.

The **3rd** column is a third walk of the same lab, driven entirely by an agent rather than a
human at a keyboard. It reproduced every timing and every tool-call artefact, and closed the
lab's last unverified instruction — § 5's *add your own tool*, which had shipped as a snippet
nobody had run. It also failed in an instructive way: **`azd` would not prompt at all**, so
§ 1's five interactive questions could not be re-verified from it.

The **C#** column is [Lab 04](../tutorial/04-add-tools.md)'s other track, walked for the first
time on 2026-08-14. Until then the page presented both tracks side by side while every verified
block on it came from Python. Provision, deploy, invoke and teardown all landed within the
Python spread, so the lifecycle itself is language-neutral — but three behaviours are not, and
the lab now records each:

- **`DefaultAzureCredential` is not portable.** .NET treats an IMDS **timeout** as fatal
  instead of "credential unavailable", so a local run on a machine with no instance metadata
  service — a laptop, or WSL — spends 100 s and then fails with
  `ManagedIdentityCredential authentication failed`, never reaching the `az login` Python falls
  through to. `AZURE_TOKEN_CREDENTIALS=dev` fixes it; deployed agents, which have a real
  managed identity, were never affected.
- **The sample's `?? "gpt-4o"` default is dead code.** `azure.yaml` expands an unset
  `AZURE_AI_MODEL_DEPLOYMENT_NAME` to an empty string rather than to nothing, and `??` tests
  for null.
- **Tool definitions are invisible.** The C# server logs no `gen_ai.tool.definitions`, the
  attribute the Python track uses to prove what the model was shown.

Its `Resources created` and role-assignment rows read *not measured* because `az resource list`
was not run on that walk; teardown was confirmed with `az group exists` and
`az cognitiveservices account list-deleted` instead.

**Teardown timing is the one number you should not trust to a single run.** Provision and
deploy have held within a few seconds across every run in this table; the nine completed
`azd down --force --purge` runs recorded on this page span **1m45s to 4m1s** — the slowest is
more than twice the fastest — and one more did not complete at all, failing at the purge step
with `409 RequestConflict`. It waits on resource-group deletion and a Cognitive Services purge,
neither of which is under `azd`'s control. Budget for four minutes, and confirm with
`az group exists` **and** `az cognitiveservices account list-deleted` rather than trusting the
`SUCCESS` line.

### Blocks whose capture file was not kept

A 2026-08-14 audit re-parsed every ✅ Verified block in Labs 01–04 against the retained capture
files. Labs 01 and 04 reconcile completely. **Six blocks in Labs 02 and 03 do not**, and the
reason in every case is that the capture was never saved to a file, not that the output is in
doubt:

| Page | Block | Why it does not reconcile |
|---|---|---|
| [Lab 02](../tutorial/02-first-agent.md) § 2 `init` | scaffold + `Next:` | The saved capture is a **redirect**, so it lacks the `Next:` block entirely and carries spinner lines. Corroborated instead by the Lab 04 pty capture, which shows the identical non-prompting `Next:` layout. |
| [Lab 02](../tutorial/02-first-agent.md) § 6 `run` | startup log | Matches the saved capture line for line **except the timestamps, the PID and the hostname** — so the page quotes a second, unsaved session of the same command. |
| [Lab 02](../tutorial/02-first-agent.md) § 5 `doctor` | `10 passed, 1 failed, 2 skipped` | No capture retained. The arithmetic is consistent with every other `doctor` run (13 checks). |
| [Lab 03](../tutorial/03-deploy.md) § 2 `show` | `hello-world:1` | Captured 2026-08-08, before captures were kept. |
| [Lab 03](../tutorial/03-deploy.md) § 3 `invoke` | remote reply | Same run, same reason. |
| [Lab 03](../tutorial/03-deploy.md) § 5 `eval` | `3m43s` run | Captured 2026-08-09, same reason. |

These are **not** relabelled illustrative: each was captured from a real run, and saying
otherwise would be its own false claim. What is missing is the artefact that would let a
reader re-check it. Closing the gap needs a billed Azure run of Labs 02 and 03 with
`script -qec` captures kept alongside the existing ones.

> [!TIP]
> The audit also found that the checker used to enforce all this had been reading
> `<details>` but not `<details open>`, and so had silently skipped **16 of the 30** blocks it
> was believed to cover. Labs 02 and 03 use the `open` form throughout, which is why the gap
> above went unseen. **A tool that reports nothing is not the same as a tool that reports
> success.****

Container mode was measured separately — it is the outlier:

| | Code mode (default) | Container mode |
|---|---|---|
| `azd provision` | **1m20s** | **2m39s** — roughly double |
| `azd deploy` | 2m21s | **2m40s** |
| First `invoke` | 14.242s | **11.399s** |
| `azd down --force --purge` | 1m46s | **2m5s** |
| Resources created | **2** | **3** (adds a **Premium** ACR) |
| Role assignments | 0 | **1** — `AcrPull`, at *ACR* scope |

What to take from this:

- **Two resources, every time — unless you choose container mode.** Language and run do not
  change the footprint; the deploy mode does. See [cost](cost.md) and
  [deploy modes](deploy-modes.md).
- **Under 5 minutes, cold to serving.** Both languages. Rebuilding an environment is cheap, so
  there is no reason to leave one running.
- **Evaluation is not a 5-minute add-on.** `eval generate` alone took **8m51s** — longer than
  provision, deploy and the eval run *combined*. Budget for it. See [Lab 06](../tutorial/06-evaluate.md).
- **Timings vary ±20% run to run.** The Python run was *faster* to provision and deploy but
  *slower* on first invoke — with a single sample each, that is noise, not a language ranking.
  Treat these as orders of magnitude, not benchmarks.
- **First invoke is always the slow one** — cold start plus a managed-identity token fetch. See
  [observability](observability.md).
- **Zero role assignments, both runs.** The golden path runs on *inherited* permissions. See
  [identity & RBAC](identity-and-rbac.md).


## The commands you'll actually type

```bash
azd ai agent init -m "<manifestUrl>"        # scaffold
azd env set AZURE_SUBSCRIPTION_ID <id>
azd env set AZURE_LOCATION eastus2
azd provision
azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME gpt-5.4-mini   # ⚠️ always needed
azd ai agent run --no-client                # terminal 1
azd ai agent invoke --local "hello"         # terminal 2
azd deploy
azd ai agent invoke "hello"
azd ai agent doctor                         # when anything is wrong
azd down --force --purge                    # ⚠️ always finish here
```

---

## Top 3 gotchas

1. **`azd` < 1.27.1** → extension silently downgrades → `must contain 'template' field`.
2. **`AZURE_AI_MODEL_DEPLOYMENT_NAME` is never set by `provision`** → `RuntimeError` on first run.
3. **`azd down` without `--purge`** → soft-deleted account holds the name for 48 h.

All three are covered in [troubleshooting](troubleshooting.md).
