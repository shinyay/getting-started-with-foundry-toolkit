# Repository guide for coding agents

This repository is **a documentation + samples repo** for the **Microsoft Foundry Toolkit**.
It is *not* the toolkit itself. Nothing here is a published package; every sample is meant to
be read, copied and run by a human learner.

## What lives where

The repo is organised as **three modes**, and the directory name announces which mode you
are in. Content that does the wrong job for its layer belongs in a different layer.

| Path | Mode | Purpose |
|---|---|---|
| `docs/learn/` | 📘 **read** | The mental model. **No command the reader is expected to type.** |
| `docs/tutorial/` | 🧪 **do** | Sequential, checkpointed labs. Minimal prose; link to `learn/` for *why*. |
| `docs/reference/` | 📖 **look up** | Lookup tables: schema, CLI surface, env vars, ports, troubleshooting. No narrative. |
| `samples/python/` | 🧪 **do** | Progressive ladder 01 → 04, Python. Clone-and-run. |
| `samples/csharp/` | 🧪 **do** | The same ladder in C#, 01 → 03 (step 04 is language-agnostic). |

## Rules when editing this repo

1. **Never invent CLI output.** Every fenced block labelled *verified* was captured from a
   real run against real Azure. If you change a command, re-run it or mark the block as
   illustrative.
2. **Respect the layer.** A command the reader types does not belong in `learn/`. An
   explanation of *why* does not belong in `tutorial/` — link to `learn/` instead. A
   narrative walkthrough does not belong in `reference/`.
3. **Keep the tracks symmetric.** A capability documented in the CLI tutorial should have an
   `alt-vscode.md` counterpart or an explicit "not available in the GUI" note.
4. **Docs are English-only.** Do not add translated copies.
5. **Samples must stay runnable standalone.** Each sample folder carries its own `README.md`
   and its own `azure.yaml`; do not factor shared files up a level.
6. **Pin nothing you cannot verify.** Version numbers in prose must match the verified
   toolchain recorded in `docs/tutorial/01-setup.md` and `docs/reference/ecosystem.md`.
7. **One fact, one home.** Commands may be repeated — that is what makes a lab followable.
   Explanations, version numbers and measured timings may not; they live in `reference/`
   and everything else links to them.

## Microsoft Foundry Skill

This repo's subject matter is covered by the **Microsoft Foundry Skill**. Install it for
guided deployment, evaluation and troubleshooting workflows:

```bash
npx skills add https://github.com/microsoft/azure-skills --skill microsoft-foundry
```

- **Copilot CLI**: `/plugin marketplace add microsoft/azure-skills` then `/plugin install azure@azure-skills`
- **Claude Code**: `/plugin install azure@claude-plugins-official`

Then ask naturally, e.g. `Use the Microsoft Foundry Skill to deploy this agent.`

When running Foundry commands on behalf of a user, set the telemetry marker inline
(never persist it):

```bash
AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd ai agent show
```

## Validation before committing

```bash
# YAML sanity for every sample manifest
find samples -name azure.yaml -exec python3 -c "import sys,yaml;yaml.safe_load(open(sys.argv[1]))" {} \;
```
