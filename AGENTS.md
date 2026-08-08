# Repository guide for coding agents

This repository is **a documentation + samples repo** for the **Microsoft Foundry Toolkit**.
It is *not* the toolkit itself. Nothing here is a published package; every sample is meant to
be read, copied and run by a human learner.

## What lives where

| Path | Purpose |
|---|---|
| `docs/concepts/` | Vendor-neutral explanation of the mental model. No commands. |
| `docs/setup/` | Install + auth + first-run checks. |
| `docs/guide-cli/` | The `azd ai agent` track — the primary, most capable path. |
| `docs/guide-gui/` | The VS Code Foundry Toolkit extension track. |
| `docs/reference/` | Lookup tables: schema, CLI surface, env vars, ports, troubleshooting. |
| `samples/python/` | Progressive ladder 01 → 04, Python. |
| `samples/csharp/` | The same ladder in C#. |

## Rules when editing this repo

1. **Never invent CLI output.** Every fenced block labelled *verified* was captured from a
   real run against real Azure. If you change a command, re-run it or mark the block as
   illustrative.
2. **Keep the two tracks symmetric.** A capability documented in `guide-cli` should have a
   `guide-gui` counterpart or an explicit "not available in the GUI" note.
3. **Docs are English-only.** Do not add translated copies.
4. **Samples must stay runnable standalone.** Each sample folder carries its own `README.md`
   and its own `azure.yaml`; do not factor shared files up a level.
5. **Pin nothing you cannot verify.** Version numbers in prose must match the verified
   toolchain recorded in `docs/setup/README.md` and `docs/reference/ecosystem.md`.

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
