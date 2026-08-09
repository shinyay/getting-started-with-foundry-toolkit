# 🧪 手を動かす — Tutorial

> **Ten labs, in order.** Each one is self-contained, checkpointed, and ends with a working
> thing you can point at. Everything here was run against live Azure and the output you see is
> the output we got.

New to the toolkit? Read **[📘 Learn](../learn/README.md)** first — 20 minutes there saves an
hour of confused debugging here. In a hurry? Skip straight to [Lab 01](01-setup.md).

## Pick a route

You almost certainly do not want all ten labs today. Pick the one that matches why you are here.

### ⚡ Quick win — "show me it works" · ~1 hour · ~$0.05

| | Lab | You get |
|---|---|---|
| 1 | [01 — Setup](01-setup.md) | tools installed, `doctor` green |
| 2 | [02 — Your first agent](02-first-agent.md) | an agent answering **on your laptop** |
| 3 | [03 — Deploy](03-deploy.md) | the same agent live on Azure, then destroyed |

**Stop here and you already know the whole lifecycle.** Everything after this is depth.

### 🌗 Practitioner — "I'm going to build something" · ~3 hours · ~$0.30

Quick win, plus:

| | Lab | You get |
|---|---|---|
| 4 | [04 — Add tools](04-add-tools.md) | an agent that *does* things, not just talks |
| 5 | [05 — MCP toolbox](05-mcp-toolbox.md) | external tools over MCP |
| 6 | [06 — Evaluate](06-evaluate.md) | a **measured** quality score, not a vibe |

Recommended reading alongside: [Learn 01–06](../learn/README.md).

### 🏭 Production — "this has to survive contact with a team" · ~6 hours · ~$1

Practitioner, plus:

| | Lab | You get |
|---|---|---|
| 7 | [07 — Container mode](07-container-mode.md) | you own the image; the ACR appears |
| 8 | [08 — Observability](08-observability.md) | a real trace, queried with KQL |
| 9 | [09 — Multi-agent A2A](09-multi-agent-a2a.md) | agent-to-agent — ⚠️ **and where it breaks** |
| 10 | [10 — CI/CD](10-cicd.md) | the pipeline, and the traps that make CI lie to you |

Recommended reading alongside: [Learn 07–10](../learn/README.md) and the
[reference](../reference/README.md) deep-dives.

## Alternative routes

These replace labs 02–03 rather than adding to them.

| Route | When to take it |
|---|---|
| [🖱️ VS Code GUI](alt-vscode.md) | You would rather click than type. Same Azure resources underneath. |
| [🧩 Prompt agents](alt-prompt-agents.md) | Your agent is instructions + hosted tools, with no code of your own. |

## All labs at a glance

| Lab | ⏱️ | Requires | 💰 | Live-verified |
|---|---|---|---|---|
| [01 — Setup](01-setup.md) | 20 min | — | free | ✅ |
| [02 — Your first agent](02-first-agent.md) | 30 min | 01 | ~$0.01 | ✅ |
| [03 — Deploy](03-deploy.md) | 30 min | 02 | ~$0.02 | ✅ |
| [04 — Add tools](04-add-tools.md) | 30 min | 03 | ~$0.02 | ✅ |
| [05 — MCP toolbox](05-mcp-toolbox.md) | 35 min | 04 | ~$0.03 | ✅ |
| [06 — Evaluate](06-evaluate.md) | 45 min | 03 | ~$0.20 | ✅ |
| [07 — Container mode](07-container-mode.md) | 25 min | 03 | ~$0.10 + Premium ACR | ✅ |
| [08 — Observability](08-observability.md) | 30 min | 03 | ~$0.05 | ✅ |
| [09 — Multi-agent A2A](09-multi-agent-a2a.md) | 60 min | 03 | ~$0.10 | ⚠️ **partial — by design** |
| [10 — CI/CD](10-cicd.md) | 45 min | 03 | per pipeline run | ➖ workflow not executed |

Costs are rough guides for a single pass at the region and model used here. See
[cost](../reference/cost.md) for the real arithmetic.

## Before your first lab

```bash
azd version
az login && azd auth login
azd ai agent doctor
```

Full prerequisites, including the version floor and every install path:
**[Lab 01 — Setup](01-setup.md)**.

> [!IMPORTANT]
> **Tear down after every lab.** `azd down --force --purge`. The `--purge` matters: without it a
> soft-deleted Foundry account keeps its name reserved and the next lab collides with it. Lab 07
> in particular leaves a **Premium**-SKU registry billing daily if you skip this.

## How to read a lab

Every lab has the same shape, so you can skim it before committing:

| Section | What it is for |
|---|---|
| `> ⏱️ · 📋 · 💰 · ☁️` | Decide whether to start, before you start |
| **What you'll learn** | The mental model, not just the commands |
| **Steps** | Copy-pasteable, in order |
| **✅ Checkpoint** | Concrete expected output — you are done when this matches |
| **🔧 If that didn't work** | The 3–6 failures specific to *this* lab |
| **✏️ Exercise** | Predict something, then check yourself |
| **→ Next** | Where to go |

## What "verified" means here

A block marked ✅ **verified** was captured from a real run against live Azure on the date
stated. If we could not verify something, the page says so in those words — see
[Lab 09](09-multi-agent-a2a.md), which documents a defect we could **not** work around after four
attempts. A labelled gap is honest; an unlabelled guess would not be.

## → Next

- 👉 [Lab 01 — Setup](01-setup.md) — start here.
- 👉 [📘 Learn](../learn/README.md) — the concepts behind these labs.
- 👉 [📖 Reference](../reference/README.md) — look things up later.
- 👉 [🧪 Samples](../../samples/README.md) — the code these labs run.
