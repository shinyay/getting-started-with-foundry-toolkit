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

- **`CONTRIBUTING.md`, issue forms and a PR template.** `AGENTS.md` was the only contributor
  contract, and GitHub does not surface that file to humans anywhere in its UI. The new
  [`CONTRIBUTING.md`](CONTRIBUTING.md) is deliberately a short entry point that defers to
  `AGENTS.md` rather than restating it — a rule written down twice drifts in one of the two
  copies, which is the same principle as rule 7.
- **A *stale command* issue form.** This repo's characteristic failure is not a typo: it is a
  documented command that silently stops working when a `1.0.0-beta.*` extension ships a
  breaking change. No CI check can detect that, because none of the nine call Azure or run
  `azd` — a reader hitting it is the only detector that exists. The form collects exactly
  what re-verification needs: the page, the block, whether that block was labelled
  **✅ Verified** or **illustrative**, `azd version`, the full
  `azd extension list --installed` table, and the actual output verbatim. It asks the
  reporter to redact subscription and tenant IDs and absolute paths, and to run
  `azd ai agent doctor --local-only` *without* `--unredacted`, which is the flag that would
  otherwise print principal IDs and UPNs into a public issue.
- **`docs-issue.yml` as a catch-all.** Blank issues are now disabled, so disabling them
  without a general form would have made some problems unreportable.
- **`evidence/` — 49 verbatim `--help` captures (212 KB).** The baseline rule 1 is checked
  against now lives in the repo instead of in a throwaway working directory, so a
  contributor can diff a changed command against what was actually observed. Includes four
  captures that are *negative* evidence — proof that `azd ai agent deploy`, `env`, `logs`
  and `provision` do **not** exist, which is the mistake a reader arriving from `azd` core
  is most likely to make.

### Security

- **History rewritten to drop an absolute capture path.** [`5b856bb`](https://github.com/shinyay/getting-started-with-foundry-toolkit/commit/5b856bb) shortened
  `/home/<user>/.copilot/session-state/<uuid>/files/rdcheck/infra` to `<work-dir>` in
  [infrastructure](docs/reference/infrastructure.md), but only at the tip — the original
  string survived in the parent commit, so publishing the repository would have republished
  exactly what that commit set out to remove. Rewritten with `git filter-repo` before going
  public, which is the last moment it is cheap: after the first fork or clone it is
  permanent. The tree is byte-identical (`48cde55…` before and after) — **only history
  changed**. Nothing else in any of the 12 commits needed removing: a scan of all 239 blobs
  found no credentials, tokens, private keys or personal email addresses.
- **Commit SHAs from `12fdc05` onward changed.** The links in *How this repo got here* below
  were repaired. Any clone taken before this must be re-cloned; `git pull` will not
  reconcile.
- **The rewrite did not delete the old objects from GitHub, and going public exposed the
  SHAs that address them.** A force-push abandons commits; it does not purge them. The
  pre-rewrite commit is still served by the API to anyone who knows its SHA, and
  `GET /repos/{owner}/{repo}/events` — readable without authentication on a public repo —
  lists the `before`/`head` SHA of every push, so the abandoned commits are *enumerable*,
  not merely guessable. Both facts were verified against the live API rather than assumed.
  What remains exposed is one absolute path: a Linux username identical to the owner's
  public GitHub handle, plus a session UUID meaningless outside the machine that made it —
  strictly less identifying than the tenant and subscription IDs this repo publishes on
  purpose. **A purge request to GitHub Support was drafted and deliberately not filed.**
  GitHub
  [states](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)
  Support "won't remove non-sensitive data, and will only assist in the removal of sensitive
  data in cases where we determine that the risk can't be mitigated by rotating affected
  credentials" — a filesystem path has no credential to rotate and does not clear that bar,
  so the request would have consumed a support engineer's time to arrive at the answer
  already known. The resolution is time instead: GitHub's events endpoint
  [retains only 30 days](https://docs.github.com/en/rest/activity/events), so the abandoned
  SHAs stop being enumerable on or about **2026-09-11**, tracked in #8. Recorded here rather
  than quietly fixed, because the reasonable assumption — *rewriting history removes the
  data* — is wrong, and anyone repeating this procedure will make the same mistake. If there
  is a next time, `git-filter-repo`'s `--sensitive-data-removal` flag (≥ 2.47) is the
  documented way to do it; this rewrite used plain `--replace-text`.

### Changed

- **The repository is public.** A getting-started guide nobody can open has no readers: the
  `v2026.08.0` release notes and the README badges were owner-only. Publishing also turns on
  `validations: required` in the two issue forms, which GitHub honours on public
  repositories only — until today the forms looked strict but enforced nothing. Added 14
  topics and set the homepage to the [tutorial index](docs/tutorial/README.md); Issue #2
  proposed `docs/README.md`, which does not exist.

### Verified

- **Labs 01–03 re-run end-to-end on live Azure, 2026-08-12.** Provision → run → local invoke →
  deploy → show → remote invoke → doctor → `down --purge`, then torn down; teardown confirmed
  independently with `az group exists` and `az cognitiveservices account list-deleted` rather
  than trusting `azd`'s exit code. Timings are in the
  [Verified runs table](docs/reference/README.md#verified-runs). Unlike the 2026-08-08/09 runs
  this one followed the *tutorial as written* — the catalog sample — instead of this
  repository's own samples, which is what exposed the drifts below. **Labs 04–10 were not
  re-run.**
- **Reproduced field-for-field:** the `doctor` 13-point diagnostic (11 passed / 0 failed /
  2 skipped) and its `--local-only` variant, all 17 fields of `azd ai agent show`, the eight-line
  remote `invoke` header, the two-resource provision, the `--purge` teardown at 1m46s, the
  `curl localhost:8088/` 404, the harmless `169.254.169.254` traceback, and the
  `eval generate` required-flag error — the last matching the documented prose verbatim.
- **Toolchain re-checked 2026-08-12.** `azd` 1.30.0 and all five extensions still report
  *Up to date*, so `v2026.08.0` remains accurate — no re-cut needed.
- **A second exit-code false positive.** `azd ai agent sample show --help` exits **0** and
  silently prints the parent help, while `azd ai agent deploy --help` exits **1**. An
  unknown *nested* subcommand therefore succeeds where an unknown top-level one fails.
  Recorded in the [FAQ](docs/reference/faq.md), which previously described the `invoke`
  case as if it were isolated.

### Fixed

- **Lab 01's checkpoint demanded a state Lab 01 cannot reach.** It required
  `11 passed, 0 failed, 2 skipped` — all green, including `(✓) FOUNDRY_PROJECT_ENDPOINT set`
  and three passing *Remote* checks — from a lab whose own header says
  *"$0 · Creates 0 Azure resources"*, and whose §7 explicitly teaches the opposite fifty lines
  earlier: *"A red `(x)` before you provision is correct. `FOUNDRY_PROJECT_ENDPOINT` cannot
  exist yet."* The page contradicted itself, and the checkpoint is this repo's pass/fail gate —
  a reader who did everything correctly saw a mismatch and was sent to *If that didn't work*,
  where nothing could help because nothing was wrong. All-green is the end state of **Lab 03**,
  two labs later. The block has been moved there, to the `doctor` step where it is first
  achievable; Lab 01's checkpoint now verifies what Lab 01 actually produces — the toolchain,
  both sign-ins, and `doctor` being reachable at all. Found by walking the tutorial by hand,
  not by CI: check 4 verifies that a checkpoint *exists*, not that it is *reachable*.
- **A fourth `doctor` state is now documented.** Running it outside any project — where every
  reader stands the moment they finish Lab 01 — reports `1 passed, 1 failed, 11 skipped`, and
  none of the three previously documented states matched it. Captured verbatim and made Lab 01's
  checkpoint, with the cascade spelled out: eleven skips from one real problem. Also noted that
  the CLI contradicts itself in that output — the error body says `azd init`, the `fix:` line
  says `azd ai agent init`, and only the latter produces a project with an agent service.
- **The repo's last uncaptured tutorial block is now verbatim — and the inference in it was
  wrong.** [Lab 02](docs/tutorial/02-first-agent.md) honestly flagged its `invoke --local`
  output as *"❌ not captured verbatim"*, having derived the shape from the remote invoke on
  the assumption that both print an identical header. They do not. A real capture differs in
  five ways: the first field is `Target:` (host and port) not `Agent:`; `Session:` appears once
  as a plain UUID rather than twice; `Conversation:` is unprefixed, not `conv_…`; the reply is
  prefixed `[local]`, not the agent name — the local server logs `agent_name=(not set)` even
  though `azure.yaml` sets `name:` — and **there is no `Trace ID:` line at all**, which is the
  one field a reader would most likely go looking for. Replaced with the capture and promoted
  to ✅ verified. This is the case for rule 1: the block was labelled as an inference, the
  inference was reasonable, and it was still wrong in every detail that mattered.
- **Lab 02's directory tree was missing a level, and it is the level that breaks the lab.**
  `azd ai agent init` creates `my-agent/agent-framework-agent-basic-responses/` and puts
  `azure.yaml`, `.azure/`, `src/` — and its own `git init` — inside *that*, not in `my-agent/`.
  The verified `init` output had elided the destination path to
  `…/agent-framework-agent-basic-responses`, so the diagram and the capture never visibly
  contradicted each other. A reader who follows the page literally then runs `azd env set` one
  directory too high and is told no project exists.
- **Labs 03 and 06 showed an agent name the reader cannot produce.** Their captures came from
  this repo's `samples/python/01-hello-world` (`name: hello-world`), but Lab 02 initialises the
  *catalog* sample and `init` never renames it — so every "verified" block on those two pages
  named something a reader following the tutorial would never see. The captures are real, so
  they have **not** been edited; both pages now state their provenance and the single
  substitution to make. Labs 07 and 08 were already consistent.
- **`azd deploy` injects five env vars, not two.** Lab 03 listed only `_NAME` and `_VERSION`;
  it also writes `_ENDPOINT`, `_PROJECT_ENDPOINT` and `_RESPONSES_ENDPOINT`. Not cosmetic —
  `azd ai agent eval` resolves its target from these and names the variable it read.
- **Lab 03 claimed `show` cannot emit JSON.** The note told readers to check `--help` for
  "the output flags your installed version supports", while the same page's troubleshooting
  table already instructed `azd ai agent show --output json`, and
  [`evidence/help/show.txt`](evidence/help/show.txt) has documented `-o, --output` with a
  worked JSON example all along. Verified working. The note was written without checking the
  evidence file that exists precisely to prevent this — rule 1 cuts both ways: do not invent
  output, and do not under-claim against a capture you already hold.
- **Lab 03 pointed at the wrong half of Lab 01 for "full green output".** The anchor resolved
  to §7, which deliberately shows a *failing* `--local-only` run; the genuine
  11 passed / 0 failed / 2 skipped block is in Lab 01's checkpoint. CI check 9 could not catch
  this — the anchor was real, just wrong.
- **"Provision writes ~15 variables"** — it was 24 in this run. Replaced the estimate with the
  measured count.
- **Two new failure modes documented in [troubleshooting](docs/reference/troubleshooting.md).**
  §7d gains the `doctor` case: the *"Hosted agents are active"* probe can fail its token
  acquisition and is then counted as **skipped**, not failed, so `doctor` still exits `0` and
  reports `10 passed, 0 failed, 3 skipped` for a perfectly healthy agent — hit on two of three
  consecutive runs. §10 gains the local-vs-hosted Python gap: `uv` fetched CPython 3.14.3
  locally while the deployed agent reported `python_3_13`, so "works locally" does not prove
  "works hosted".
- **Removed the agent's local capture directory from two "verified" logs.** The Bicep and
  Terraform ejection excerpts in [infrastructure](docs/reference/infrastructure.md) carried
  an absolute session-state path. It was genuine `azd` output, not a fabrication, but it is
  meaningless to a reader — shortened to `<work-dir>` and labelled as shortened, so the
  block stays honest about no longer being character-for-character verbatim.
- **CI moved off Node 20 actions.** `actions/checkout` and `actions/setup-python` were
  being force-run on Node 24 with a deprecation warning; when Node 20 is removed the nine
  validation checks would have stopped running, which is this repo's only guarantee that
  its documentation still holds. Both bumped to `@v7`.
- **CI check 2 was blind to every dot-directory, including its own workflow file.** Python's
  `glob` wildcards do not descend into paths beginning with a dot, so `**/*.y*ml` matched 9
  of the repo's 11 YAML files — silently exempting `.github/workflows/validate.yml` and the
  sample asset `samples/python/04-eval/src/hello-world/.agent_configs/baseline/metadata.yaml`.
  The gap mattered immediately: the issue forms added above live under `.github/` and would
  never have been parsed, and a malformed issue form does not fail loudly — it just stops
  rendering on GitHub. Switched to `os.walk`; the check now reports how many files it
  scanned, so the next such gap is visible rather than inferred.

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
| **1 · Review** | [`c612262`](https://github.com/shinyay/getting-started-with-foundry-toolkit/commit/c612262), [`12fdc05`](https://github.com/shinyay/getting-started-with-foundry-toolkit/commit/12fdc05) | Rubber-duck review for correctness, depth and clarity; remediate 69 findings; re-audit the audit and validate the Python path live. |
| **2 · Restructure** | [`3ee8aad`](https://github.com/shinyay/getting-started-with-foundry-toolkit/commit/3ee8aad) | Split into the three modes, add four live-verified labs, add CI checks 5–7, and replace every fabricated block found by two audit rounds. |
| **3 · Refine** | [`c107a4f`](https://github.com/shinyay/getting-started-with-foundry-toolkit/commit/c107a4f) | Add active recall to the learn layer, split the oversized multi-agent page, add the cheatsheet and FAQ, and add CI checks 8–9. |

[2026.08.0]: https://github.com/shinyay/getting-started-with-foundry-toolkit/releases/tag/v2026.08.0
