# ❓ Frequently Asked Questions

> Short answers to the questions newcomers actually ask — especially where the
> intuitive answer is wrong. Each answer links to the page that owns the detail.

---

## Getting started

### How many azd extensions do I need?

**Five**, not four. The easy-to-miss one is `azure.ai.connections`. Additionally,
`azure.ai.toolboxes` installs itself on first use (mid-command), which can hang CI
if network access is restricted. See the full list in
[Setup](../tutorial/01-setup.md).

### What does `azd ai agent doctor` actually check?

It runs **13 checks** grouped into Local (8), Authentication (1), and Remote (4).
A failed check turns every subsequent check into a **skip**, so always fix the
topmost `(x)` first. It has no GUI equivalent.
See [Troubleshooting § doctor](troubleshooting.md).

### What does `must contain 'template' field` mean?

Your YAML is fine — your **azd version is too old**. The `azd` version gates the
extension version, which gates the manifest format the extension can parse. The
extension may silently downgrade with only a warning.
Full explanation: [Troubleshooting § version skew](troubleshooting.md#1-version-skew--the-one-that-bites-everyone).

### Where do I find version numbers and timing benchmarks?

They live in one canonical place — [Fast facts](README.md#fast-facts) and
[Verified runs](README.md#verified-runs). They are not duplicated elsewhere
because CI enforces version consistency.

---

## Cost & cleanup

### How many Azure resources does code-mode create?

Exactly **2**. Container mode adds a **third: an ACR at Premium SKU**, billed
daily whether or not you deploy. Each deploy adds an image tag with no automatic
cleanup. Details: [Cost](cost.md) and [Deploy modes](deploy-modes.md).

### Why does `azd down` not let me re-provision with the same name?

Without the `--purge` flag, the Cognitive Services account is **soft-deleted** —
it keeps its name reserved and blocks re-provisioning until you purge it.
See [Cost § cleanup](cost.md).

---

## Deploying

### Why is my agent broken even though `azd provision` and `azd deploy` both succeeded?

`AZURE_AI_MODEL_DEPLOYMENT_NAME` is **never set for you** by `azd provision`.
An unset `${VAR}` in `azure.yaml` silently expands to an empty string — both
provision and deploy still succeed, and the failure only appears at runtime.
See [Environment variables](environment-variables.md).

### What is the correct endpoint variable name?

`FOUNDRY_PROJECT_ENDPOINT`. The name **`AZURE_AI_PROJECT_ENDPOINT` does not exist** —
using it gives you an empty value. See [Environment variables](environment-variables.md).

### Which env-var names are reserved?

`APPLICATIONINSIGHTS_CONNECTION_STRING` and all `FOUNDRY_*` / `AGENT_*` variables
are **reserved by the platform**. Declaring one as an agent env var fails the deploy
with HTTP 400 `invalid_payload`. Telemetry is wired at the project level instead.
See [Environment variables](environment-variables.md) and [Observability](observability.md).

### Does deploying again create a second agent?

No. Deploying with the same name creates a new **version** (`:2`). Each deployed
agent gets its own managed identity. See [Identity & RBAC](identity-and-rbac.md).

### Do I need Docker installed for container mode?

Not if you set `remoteBuild: true` — the build runs in ACR Tasks.
See [Deploy modes](deploy-modes.md).

### What does `azd ai agent init -m <sample>` produce?

It creates a **subdirectory** named after the sample, not files in the current
directory. See [Tutorial § first agent](../tutorial/02-first-agent.md).

---

## Debugging

### `azd ai agent invoke` returned success — can I trust that?

**No.** `invoke` returns HTTP 200 with an **empty body and exit code 0** when the
agent is broken. Never gate CI on its exit code; assert on the output content
instead. See [Troubleshooting](troubleshooting.md).

> [!CAUTION]
> This is the single most dangerous false-positive in the toolkit.

### How do I list my deployed agents?

`azd ai agent list` does **not exist**. Use the Azure AI Foundry portal or the
SDK. See [CLI reference](azd-cli.md).

### How do I view agent logs?

`azd ai agent logs` does **not exist**. The command is `azd ai agent monitor`.
See [Observability](observability.md).

### I see a scary `169.254.169.254` traceback — is something wrong?

No. That is `DefaultAzureCredential` probing the IMDS endpoint, which does not
exist on a laptop. It is harmless and expected.
See [Troubleshooting](troubleshooting.md).

### `curl http://localhost:8088/` returns 404 — is the agent broken?

No — there is **no root route**. 404 is the correct response for `/`.
See [Troubleshooting](troubleshooting.md).

### How do I query App Insights for agent invocations?

Look in the `requests` table for `name = invoke_agent`, `resultCode = 0` (not 200),
`cloud_RoleName = agentsv2`. The platform traces the invocation only — your own
tool calls need your own instrumentation.
See [Observability](observability.md).

---

## Evaluation

### Where must `eval.yaml` live?

Inside the service's `project:` directory (e.g. `src/<agent>/`), **not** the
sample root. A relative `--config` path cannot rescue a wrong layout.
See [Tutorial § evaluate](../tutorial/06-evaluate.md).

### Can I copy `eval.yaml` between projects?

**No.** `local_uri` is a download cache, not an upload source. Reusing an
`eval.yaml` from another project gives `404 … evaluator not found`. Only
`eval generate` registers the assets.
See [Tutorial § evaluate](../tutorial/06-evaluate.md).

### Why is `eval generate` so slow?

Because it does two jobs, and it prints a timer for each: dataset generation, then
evaluator generation. Together they take roughly **2.4×** an `eval run`. It is the step
nobody budgets time for — see [Verified runs](README.md#verified-runs) for the measured
numbers.

### My eval scored 6 failures — did I break something?

Probably not. The verified run scored **15 total, 9 passed, 6 failed** — and
6 failures is the *correct* result. The generated rubric grades identity fidelity,
while the sample's system prompt is only "You are a friendly assistant".
See [Tutorial § evaluate](../tutorial/06-evaluate.md).

---

## Multi-agent (A2A)

### Does A2A work end-to-end with the toolkit today?

Not fully. A2A was verified up to delegation, but delegation itself could **not**
be made to work — `azd provision` drops the manifest's `audience` field on a
`RemoteA2A` connection. The tutorial lab is deliberately the one that does not
finish green. See [Multi-agent A2A tutorial](../tutorial/09-multi-agent-a2a.md)
and [Multi-agent reference](multi-agent.md).

### Are all upstream A2A samples available via `azd ai agent sample list`?

No. Python, C#, and LangGraph A2A samples all exist upstream, but
`azd ai agent sample list` is a **curated subset**, not a full index.
See [Sample catalog](sample-catalog.md).

---

## Docs & ecosystem

### Do the VS Code docs cover `azd`?

No. The VS Code documentation covers the GUI only — `azd` appears **0 times**
across all 16 pages. See [Ecosystem](ecosystem.md).

### Where is the canonical source for version and timing data?

[Reference README § Fast facts](README.md#fast-facts) and
[Reference README § Verified runs](README.md#verified-runs). Those are the
single sources of truth enforced by CI.
