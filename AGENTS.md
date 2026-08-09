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

CI runs **nine** checks (`.github/workflows/validate.yml`). Run them all locally first —
none of them need Azure, network access or credentials:

```bash
python3 - <<'PY'
import re, subprocess, sys, yaml
steps = yaml.safe_load(open('.github/workflows/validate.yml'))['jobs']['validate']['steps']
fail = 0
for s in steps:
    if not s.get('name', '').startswith('Check'):
        continue
    m = re.search(r"python - <<'PY'\n(.*?)\nPY\s*$", s['run'], re.S)
    if not m:
        continue
    r = subprocess.run([sys.executable, '-c', m.group(1)], capture_output=True, text=True)
    tail = [l for l in r.stdout.strip().splitlines() if not l.startswith('::error')]
    fail += r.returncode != 0
    print(f"[{'PASS' if r.returncode == 0 else 'FAIL'}] {s['name']}: {tail[-1] if tail else r.stderr[:200]}")
    if r.returncode:
        print("\n".join(l for l in r.stdout.splitlines() if l.startswith('::error')))
sys.exit(1 if fail else 0)
PY
```

| # | Check | Why it exists |
|---|---|---|
| 1 | Relative links resolve | |
| 2 | Every YAML parses | |
| 3 | `azure.yaml` `project:` paths exist | caught a shipped-broken C# sample |
| 4 | Tutorial labs carry the lab skeleton | |
| 5 | No doc page is orphaned from `README.md` | |
| 6 | Eval assets live in the service `project:` dir | caught an unrunnable eval sample |
| 7 | Version claims match `docs/reference/README.md` | one version in 14 files is 14 places to drift |
| 8 | The three-layer contract holds | rule 2 above, made executable |
| 9 | Every `#anchor` matches a real heading | caught 59 dangling anchors that check 1 could not see |
