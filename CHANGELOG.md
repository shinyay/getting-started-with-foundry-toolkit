# Changelog

All notable changes to this repository are recorded here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## Versioning policy

This repo uses **CalVer**, not SemVer: `vYYYY.0M.PATCH`.

SemVer answers *"did the API break?"* — this repo has no API. The only question a reader
actually has is *"how stale is this?"*, and a date answers it directly. That matters more
here than in most repos, because everything is pinned to a **preview** toolchain: the `azd`
extensions are at `1.0.0-beta.*`, and every timing was measured on a specific day.

- Another release in the same month → bump `PATCH` (`2026.08.1`).
- A new month → new `YYYY.0M.0`.
- A release states the toolchain it was verified against. If your installed versions differ
  from the ones in the release notes, trust your local `--help` over this repo — that is
  [rule 9 of living with preview](docs/learn/09-living-with-preview.md).

---

## [Unreleased]

### Added

- **`evidence/` — 49 verbatim `--help` captures (212 KB).** The baseline rule 1 is checked
  against now lives in the repo instead of in a throwaway working directory, so a
  contributor can diff a changed command against what was actually observed. Includes four
  captures that are *negative* evidence — proof that `azd ai agent deploy`, `env`, `logs`
  and `provision` do **not** exist, which is the mistake a reader arriving from `azd` core
  is most likely to make.

### Verified

- **Toolchain re-checked 2026-08-12.** `azd` 1.30.0 and all five extensions still report
  *Up to date*, so `v2026.08.0` remains accurate — no re-cut needed.
- **A second exit-code false positive.** `azd ai agent sample show --help` exits **0** and
  silently prints the parent help, while `azd ai agent deploy --help` exits **1**. An
  unknown *nested* subcommand therefore succeeds where an unknown top-level one fails.
  Recorded in the [FAQ](docs/reference/faq.md), which previously described the `invoke`
  case as if it were isolated.

### Fixed

- **Removed the agent's local capture directory from two "verified" logs.** The Bicep and
  Terraform ejection excerpts in [infrastructure](docs/reference/infrastructure.md) carried
  an absolute session-state path. It was genuine `azd` output, not a fabrication, but it is
  meaningless to a reader — shortened to `<work-dir>` and labelled as shortened, so the
  block stays honest about no longer being character-for-character verbatim.
- **CI moved off Node 20 actions.** `actions/checkout` and `actions/setup-python` were
  being force-run on Node 24 with a deprecation warning; when Node 20 is removed the nine
  validation checks would have stopped running, which is this repo's only guarantee that
  its documentation still holds. Both bumped to `@v7`.

---

## [2026.08.0] — 2026-08-09

First tagged release. It covers all work to date, since there was no prior release to
compare against.

**What this repo is:** a getting-started guide for the Microsoft Foundry Toolkit
(`azd ai agent` CLI and the AI Toolkit VS Code extension), organised as three modes —
読む (learn) → 手を動かす (hands-on) → 後で引く (reference).

**The rule that makes it different:** every fenced block labelled *verified* was captured
from a real run against real Azure. Nothing is written from memory. Where something could
not be verified, the page says so rather than guessing.

### Added

- **📘 Learn — 10 pages + index.** The mental model, with **no command the reader types**.
  Covers the four products wearing one name, prompt vs hosted agents, the six lifecycle
  verbs, protocols, where code runs, the Azure footprint, the identity model, versioning,
  living with preview, and multi-agent patterns. Every page ends with a
  **✅ Check your understanding** — three questions answerable with no Azure and no spending.
- **🧪 Tutorial — 10 checkpointed labs + 2 alternative routes + index.** Setup, first agent,
  deploy, tools, MCP toolbox, evaluation, container mode, observability, multi-agent A2A and
  CI/CD. Each lab carries a time/cost/prereq banner, a `✅ Checkpoint`, a
  `🔧 If that didn't work` section and an `✏️ Exercise`. `alt-vscode.md` and
  `alt-prompt-agents.md` are alternatives to labs 02–03, not extra labs.
- **📖 Reference — 15 pages + index.** Cheatsheet, FAQ, glossary, troubleshooting,
  ecosystem map, `azure.yaml` schema, environment variables, `azd` CLI surface, deploy modes,
  infrastructure, identity & RBAC, observability, cost, multi-agent and the sample catalog.
- **🐍🔷 Sample ladders.** Python `01-hello-world → 02-tools → 03-mcp-toolbox → 04-eval` and
  C# `01 → 03` (step 04 is CLI-level and language-agnostic). Each sample is standalone: its
  own `README.md` and its own `azure.yaml`.
- **9 CI checks** (`.github/workflows/validate.yml`), none of which need Azure, network
  access or credentials. See *How this stays honest* below.
- **`AGENTS.md`** — the contributor contract, including the two rules everything else hangs
  off: *never invent CLI output*, and *respect the layer*.

### Verified

Measured against live Azure on 2026-08-08 and 2026-08-09, then destroyed. The canonical
tables live in [`docs/reference/README.md`](docs/reference/README.md) — this is a summary,
and that file wins any disagreement.

| Toolchain | Version |
|---|---|
| `azd` | **1.30.0** (minimum supported: 1.27.1) |
| `azure.ai.agents` | `1.0.0-beta.9` |
| `azure.ai.connections` | `1.0.0-beta.4` |
| `azure.ai.inspector` | `1.0.0-beta.3` |
| `azure.ai.projects` | `1.0.0-beta.5` |
| `azure.ai.toolboxes` | `1.0.0-beta.5` |

Headline measurements:

- **2 Azure resources** in the default code mode; **3** in container mode, where the third is
  an ACR at **Premium** SKU that bills daily whether or not you deploy.
- **Under 5 minutes** from nothing to a serving agent, in both Python and C#.
- **0** resource-group-scope role assignments in code mode; **1** (`AcrPull`, at ACR scope)
  in container mode.
- A full evaluation cycle scored **15 cases, 9 passed, 6 failed** — and 6 failures is the
  *correct* result, because the generated rubric grades identity fidelity while the sample's
  instruction is only "You are a friendly assistant".

### Fixed

Findings from three review passes and three fabrication audits, all remediated:

- **69 rubber-duck review findings** (11 P0, 34 P1, 24 P2).
- **10 fabricated claims** found by audit and corrected against captured evidence —
  including two blocks labelled "✅ Verified output" that had never been run, invented
  session UUIDs, an invented resource name, and an invented eval result. Where no evidence
  existed, the block was relabelled honestly rather than deleted.
- **59 dangling heading anchors** left by the three-mode restructure. The *files* still
  resolved, so the link checker stayed green while readers silently landed at the top of the
  wrong page. All repaired, and CI check 9 now blocks a recurrence.
- Several CLI details that were wrong in a way that would waste a reader's time — most
  sharply, `-p` means `--port` on `azd ai agent run` but `--protocol` on
  `azd ai agent invoke`, so `invoke -p 8088` sets the protocol to `8088` and fails.

### Known gaps

Stated up front rather than quietly omitted:

- **A2A delegation does not work.** Two agents deploy and the agent card is fetchable, but
  `azd provision` drops the manifest's `audience` on a `RemoteA2A` connection and the
  delegating call fails. Four workarounds were attempted and none succeeded.
  [Lab 09](docs/tutorial/09-multi-agent-a2a.md) is deliberately the one lab that does not
  finish green.
- **BYO-image deploy mode is documented from inference, not a live run.** See
  [deploy modes](docs/reference/deploy-modes.md).
- **`azd ai agent invoke` exits 0 even on an empty response.** This is the product's
  behaviour, not a repo defect, but it is the single most dangerous false positive in the
  toolkit — never gate CI on its exit code.

### How this stays honest

Structure that isn't enforced decays, so nine checks run on every push:

| # | Check | Why it exists |
|---|---|---|
| 1 | Relative links resolve | |
| 2 | Every YAML parses | |
| 3 | `azure.yaml` `project:` paths exist | caught a shipped-broken C# sample |
| 4 | Tutorial labs carry the lab skeleton | |
| 5 | No doc page is orphaned from `README.md` | |
| 6 | Eval assets live in the service `project:` dir | caught an unrunnable eval sample |
| 7 | Version claims match the canonical table | one version in 14 files is 14 places to drift |
| 8 | The three-layer contract holds | no reader-typed command in `learn/`; no lab without something to type |
| 9 | Every `#anchor` matches a real heading | caught 59 dead links that check 1 could not see |

---

## How this repo got here

| Phase | Commits | What happened |
|---|---|---|
| **1 · Build** | [`54c439a`](https://github.com/shinyay/getting-started-with-foundry-toolkit/commit/54c439a) | Research the upstream toolkit and the VS Code docs, then write the first complete guide, reference set and sample ladders. |
| **1 · Review** | [`c612262`](https://github.com/shinyay/getting-started-with-foundry-toolkit/commit/c612262), [`88dda95`](https://github.com/shinyay/getting-started-with-foundry-toolkit/commit/88dda95) | Rubber-duck review for correctness, depth and clarity; remediate 69 findings; re-audit the audit and validate the Python path live. |
| **2 · Restructure** | [`0178dc2`](https://github.com/shinyay/getting-started-with-foundry-toolkit/commit/0178dc2) | Split into the three modes, add four live-verified labs, add CI checks 5–7, and replace every fabricated block found by two audit rounds. |
| **3 · Refine** | [`ce70009`](https://github.com/shinyay/getting-started-with-foundry-toolkit/commit/ce70009) | Add active recall to the learn layer, split the oversized multi-agent page, add the cheatsheet and FAQ, and add CI checks 8–9. |

[2026.08.0]: https://github.com/shinyay/getting-started-with-foundry-toolkit/releases/tag/v2026.08.0
