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

- **A tenth CI check: markdown tables must be rectangular.** Every row must have the header's
  cell count, counting only unescaped `|` — which is what GFM itself counts, so a pipe inside
  a code span in a table cell is a real rendering bug unless written `\|`. Written after an
  audit found that an agent editing this repo had widened both `Resources created` rows in
  [`docs/reference/README.md`](docs/reference/README.md) while adding a column to a different
  table, and that nine checks had passed straight over it. GitHub silently drops cells past
  the header width, so the pages *looked*
  right. It also found a `kind: hosted | prompt` in
  [`alt-prompt-agents.md`](docs/tutorial/alt-prompt-agents.md) that GFM was splitting a cell
  on. Reintroducing the original defect makes the check fail, which is how it was verified.
- **`CONTRIBUTING.md`, issue forms and a PR template.** `AGENTS.md` was the only contributor
  contract, and GitHub does not surface that file to humans anywhere in its UI. The new
  [`CONTRIBUTING.md`](CONTRIBUTING.md) is deliberately a short entry point that defers to
  `AGENTS.md` rather than restating it — a rule written down twice drifts in one of the two
  copies, which is the same principle as rule 7.
- **A *stale command* issue form.** This repo's characteristic failure is not a typo: it is a
  documented command that silently stops working when a `1.0.0-beta.*` extension ships a
  breaking change. No CI check can detect that, because none of the ten call Azure or run
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

- **Lab 04's C# track walked a second time, 2026-08-14 — §§ 1–7, closing the repo's last
  never-run instruction.** The first C# walk had skipped § 5, so the page still carried one
  fragment nobody had compiled. Applied verbatim, it built **0 Warning(s), 0 Error(s)**, and
  the page's last *illustrative* label is gone. Eight further findings, all now in
  [Lab 04](docs/tutorial/04-add-tools.md):
  - **The no-reload IMPORTANT now has proof, and it is worse than a stale answer.** The same
    question, on the same server, before and after a restart: **$867** (= $289 × 3, the old
    tool's listing price) and **$630** (= $210 × 3, the new `GetRoomRate`). No error, no
    warning, no indication which tool ran — just confident arithmetic on the wrong data.
  - **C# has one local tool signal, `OutputCount`, and both of its limits matter.** A tool
    round-trip logs `OutputCount=3` against `1` for a plain reply. But it **counts without
    naming**: the stale run above also scored 3, so trusting it would have graded that failure
    a pass. And it **does not exist remotely** — hosted responses stream and end with
    `SSE stream completed`, carrying no count.
  - **The hosted C# agent logs nothing about tools, and the reason is not the one first
    written down.** `monitor` streams container stdout/stderr; Python's tool lines arrive
    because the `agent_framework` *logger* emits them. C# has no equivalent logger call — that
    is all. It *does* instrument tool execution (`ExecuteToolScope`), but as spans, which would
    never reach `monitor` regardless. An earlier draft of this entry blamed the exporter defect
    below; that was wrong and the two are unrelated.
  - **`Agent365-only mode active` is a consequence of the golden path, not a misconfiguration.**
    The transitive `Microsoft.OpenTelemetry` package auto-enables Agent365 export and enables
    Azure Monitor only when a connection string exists; `azd provision` sets no
    `APPLICATIONINSIGHTS_CONNECTION_STRING`, which is also why startup logs
    `AppInsightsConfigured=False`. **Whether wiring App Insights would surface the tool spans
    was not tested** and the page says so rather than guessing.
  - **A reproducible upstream defect in the hosted C# agent.** Every request answered
    `HTTP 200` and was then followed by `Agent365Exporter: Unhandled export exception.
    System.ArgumentException: An item with the same key has already been added.
    Key: gen_ai.conversation.id` — 3 of 3 requests. It throws in `ExportFormatter.MapAttributes`
    while *building* the span, before anything is sent, so it is independent of tenant and
    permissions. Now a troubleshooting row that tells the reader to do nothing.
  - **`--new-conversation` alone drops replayed history.** The same question cost 6989 ms with
    the previous exchange in the conversation and 2903 ms without it, while the session ID was
    unchanged. The page's tip to pass `--new-session --new-conversation` overstated what is
    required, and the 9101 ms tool-call figure should be read against 2903 ms, not 6989 ms.
  - **The `curl` 404 is a property of the protocol, not of Python.** The C# server logs the
    same `404` for `/invocations/docs/openapi.json` at startup and then prints the same
    suggestion; a manual `curl` confirmed it. Recorded where the explanation already lives, in
    [Lab 02](docs/tutorial/02-first-agent.md), rather than duplicated.
  - **`monitor` output is not safe to quote as-is.** It wraps lines **mid-token** at the
    caller's terminal width — one capture split `traceparent:` into `traceparen` / `t: ,` — and
    prints its own `status` lines out of chronological order. Added to the capture rules in
    [`docs/reference/README.md`](docs/reference/README.md).

  Timings: provision **1m24s**, deploy **2m13s**, local invoke **9.115s**, remote **6.152s**,
  teardown **4m1s** — which is a new maximum and widens the recorded `azd down` spread to
  **1m45s–4m1s**. Torn down and confirmed with both checks.

- **Lab 04's C# track walked live for the first time, 2026-08-14.** The page has presented two
  tracks side by side since it was written, but every verified block on it came from Python;
  § 2's mapping table described C# from *reading* the sample, not from running it. A full
  lifecycle — init, provision, run, invoke, deploy, show, remote invoke, teardown — closes
  that for §§ 1–4, 6 and 7. § 5's C# fragment was still unrun at that point and was labelled
  illustrative rather than sitting silently beside verified Python output; the second walk
  above closed it. The lifecycle itself proved
  language-neutral (provision **1m39s**, deploy **2m24s**,
  local invoke **9.020s**, remote **9.587s**, down **2m36s**, all inside the Python spread) and
  the `doctor` output was **byte-identical to the Python block, all 30 lines**. Four
  predictions were written down before the first live command; three held, one did not, and
  one of the three was right for the wrong reason. What was not language-neutral is now in the
  lab:
  - **.NET's `DefaultAzureCredential` treats an IMDS timeout as fatal**, not as "credential
    unavailable", so it never falls through to `az login` the way Python's does. On a laptop
    or under WSL the local run burns 100 s and dies with
    `ManagedIdentityCredential authentication failed`. `AZURE_TOKEN_CREDENTIALS=dev` fixes it;
    deployed agents were never affected. This is the single biggest difference between the two
    tracks and the page said nothing about it.
  - **The sample's `?? "gpt-4o"` fallback is unreachable.** `azure.yaml` expands an unset
    `AZURE_AI_MODEL_DEPLOYMENT_NAME` to an empty string, and `??` tests for null — so C# throws
    `System.ArgumentException: Argument is whitespace (Parameter 'model')` where Python throws
    `ValueError: Model is required`. One root cause, two messages, both now in the
    troubleshooting table. The lab's claim that this track *"does not need"* the variable set
    was wrong and has been replaced.
  - **`gen_ai.tool.definitions` is Python-only.** The C# server logs no equivalent, so the two
    troubleshooting rows that tell you to read it back are now marked as having no C#
    counterpart. The tool itself was called and answered — the arguments to
    `AIFunctionFactory.Create` do reach the model; you simply cannot see what was sent.
  - **The project-naming rule was tested on its other side.** Python's 51-character
    environment name had shown truncation at 32; C#'s 15-character one came through untouched.
    The rule truncates — it does not pad, hash or rename. The two scaffold folder names also
    come from **different sources**: Python's from the upstream sample directory, C#'s from the
    `name:` field in the sample's `azure.yaml`.
- **The pty rule is now per-command rather than blanket.** Counting cursor-movement escapes
  across one human-driven walk and two agent-driven ones shows the degradation is a property
  of the command, not a matter of degree: `doctor`, `show` and `invoke` draw no live UI and
  their captures are **identical** either way, while `init` (1463 → 0), `provision` (488 → 0),
  `deploy` (768 → 0) and `down` (665 → 0) lose theirs completely. Recorded in
  [`docs/reference/README.md`](docs/reference/README.md); it is why the C# walk promoted its
  `doctor` and `invoke` output and nothing else.
- **Lab 04 walked a third time, 2026-08-14, driven end-to-end by an agent rather than a
  human.** Every timing landed within noise, `gen_ai.tool.definitions` reproduced
  **byte-for-byte** against the previous walk's capture, and the `finish_reasons` distribution
  came out identical for the third time running (1 × `tool_calls`, 6 × `stop`, one
  `Function name: get_weather`). Torn down and confirmed with both checks. Two limits are
  worth recording: **`azd` refused to prompt** — *"Continuing because `--no-prompt` was
  specified"*, with no such flag, under a real pty and under `script -qec` alike — so § 1's
  interactive questions could **not** be re-verified from this run and still rest on the
  human-driven capture; and only `invoke` output matched a human's capture escape-for-escape,
  so nothing spinner-driven from this run was promoted.
- **Lab 04 walked a second time from an empty directory, 2026-08-13.** Same day, same
  toolchain (`azd 1.30.0`, all five extensions unchanged), following the corrected page rather
  than this repository's samples, with thirteen predictions registered *before* the first live
  command. Nine reproduced exactly — including the `gen_ai.tool.definitions` escaping, the
  `finish_reasons` distribution, the OTel span-dump structure, the two clocks nine hours apart
  in `monitor`, the `7 passed, 1 failed, 5 skipped` doctor state, the 32-character project
  name and `ERROR: no project exists` from the wrong directory. Four did not, and each became
  a fix above. Torn down and confirmed with **both** `az group exists` → `false` and
  `az cognitiveservices account list-deleted` → empty — a distinction this run is the reason
  for.
- **Lab 04 walked end-to-end on live Azure, 2026-08-13.** `init` (interactive) → `doctor` →
  `provision` → local `run` → two local `invoke`s → `deploy` → remote `invoke` → `show` →
  `monitor` → `down --force --purge`, torn down and confirmed with `az group exists` → `false`.
  Timings are in the [Verified runs table](docs/reference/README.md#verified-runs). This was the
  first lab walked with the *tool-calling* sample rather than the basic one, which is what
  exposed the tool-call evidence gap below. The Agent Inspector timeline was **not** verified.
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

- **An audit of Labs 01–04 against the retained captures, 2026-08-14.** The four lab pages
  were re-parsed block by block and every reproducible claim was re-run. Findings:

  - **The checker enforcing all of this was itself broken.** It matched `<details>` but not
    `<details open>`, so it silently skipped **16 of the 30** verified blocks in Labs 01–04 —
    every block in Labs 02 and 03, which use the `open` form throughout. The clean bill of
    health those pages had been given was an artefact of a tool printing nothing. Lab 04 was
    unaffected, which is why its three walks caught what they caught.
  - **[Lab 01](docs/tutorial/01-setup.md) was quoting `azd extension upgrade --all` with
    output removed and no declaration.** The real command prints an `Upgrading <extension>`
    line before each `(-) Skipped:` line; the capture contains no cursor-movement escape, so
    those lines are permanent, not a redrawn progress indicator. Re-captured. They repeat
    non-deterministically — three consecutive runs on an unchanged machine emitted 8, 10 and
    8 of them — which the block now states.
  - **The sample catalog had drifted, and the count had five homes.** A live recount found
    **24 Python + 13 C#** where the repo claimed 21 + 13: three `bring-your-own` samples were
    added upstream between 2026-08-12 and 2026-08-14 **with `azure.ai.agents` unchanged at
    `1.0.0-beta.9`**, which is the clearest demonstration so far that the catalog is fetched
    from GitHub at call time. [`sample-catalog.md`](docs/reference/sample-catalog.md) is now
    the single home; [Lab 02](docs/tutorial/02-first-agent.md) links to it instead of
    restating a number, and its ordinals are framed as things to verify rather than trust.
    What did not move: six `featured`, exactly one `recommended`, this lab's sample 5th.
  - **[Lab 02](docs/tutorial/02-first-agent.md) left the reader inside a foreground process.**
    It starts `azd ai agent run`, mentions Ctrl+C only as a bullet in a notes list, and hands
    off to [Lab 03](docs/tutorial/03-deploy.md), which opens straight into `azd deploy`. Its
    `→ Next` now states what to stop, what to leave alone and what Lab 03 starts from — the
    same defect fixed in Lab 04 § 6.
  - **Two claims in Lab 02 were checked against a real run for the first time.** Ctrl+C does
    print `Stopping agent...` then `Agent stopped.`, verified by sending `SIGINT` to a running
    server; a process that dies on its own prints only the second line, which the page now
    distinguishes. Running `run` before `provision` raises
    `KeyError: 'FOUNDRY_PROJECT_ENDPOINT'` — a new troubleshooting row, and a different
    failure from Lab 04's `ValueError`, because the two samples read the variable differently.
  - **Both `Resources created` rows in
    [`docs/reference/README.md`](docs/reference/README.md) had been widened by mistake** while
    a column was added to a different table — the canonical two-column table had grown two
    extra cells, the container-mode table one. Fixed, and check 10 now prevents it.
  - **Six blocks in Labs 02 and 03 have no retained capture**, and the ledger now
    [says so explicitly](docs/reference/README.md#blocks-whose-capture-file-was-not-kept)
    rather than leaving a ✅ that nothing backs. They are not relabelled illustrative — each
    came from a real run, and claiming otherwise would be its own false statement. Lab 02's
    `init` block is corroborated by the Lab 04 pty capture; its `run` block matches the saved
    capture except for timestamps, PID and hostname, so a second unsaved session was quoted.
    Closing the gap needs a billed run of Labs 02 and 03 with the captures kept.

  Labs 01 and 04 now reconcile completely: **24 of 30 blocks EXACT, and the other six are the
  documented gap above.**

- **A third Lab 04 walk, driven end-to-end by an agent, found six more defects — and one of
  them was mine.**
  - **The re-run column added in the previous entry went into the wrong table.** It was
    appended to *Container mode* in `docs/reference/README.md`, giving a three-column header
    four columns of data and captioning Lab 04 timings as container-mode measurements.
    Reverted, and put in *Verified runs* where it belongs.
  - **§ 5 *Add your own tool* had never been run.** It was the only instruction in the lab
    with no evidence behind it. Applying its `get_stock_price` snippet verbatim works: the
    model calls it with `{"symbol":"MSFT"}`, and `gen_ai.tool.definitions` then carries both
    tools. Both are now verified blocks, and the section says what it did not before — that
    **`azd ai agent run` has no reload flag**, so the server must be restarted or you keep
    testing the old code.
  - **The troubleshooting table listed an error this lab cannot produce.**
    `RuntimeError: Model deployment name is not configured.` belongs to *Lab 02's* sample,
    which raises it explicitly. Lab 04's sample raises
    `ValueError: Model is required. Set via 'model' parameter or 'FOUNDRY_MODEL' environment
    variable.` — measured by running it with the variable unset. The mechanism is now
    documented too: `azure.yaml` maps the variable through as
    `value: ${AZURE_AI_MODEL_DEPLOYMENT_NAME}`, so the container gets an **empty string**, not
    nothing, which is why it is never a `KeyError`.
  - **`azd ai agent doctor` does not catch that missing value.** It reports `(✓) manual env
    vars set` and the same `7 passed, 1 failed, 5 skipped` either way. § 3 sent readers to
    `doctor` without saying so.
  - **Counting environment values is not a reliable way to tell a finished `init` from an
    interrupted one.** The previous entry claimed 11 versus 5. An `init` that completes
    *without prompting* leaves **6**, with a different key set again. The discriminator is the
    presence of `AZURE_AI_MODEL_DEPLOYMENT_NAME`, and the page now says that instead.
  - **`azd ai agent show` has no `tools` key** — asserted before, verified now, with the
    actual `definition` key list as the evidence.
  - **The two-clocks note assumed your offset puts `monitor` ahead of the container.** Across
    midnight UTC it is behind, and the two stamps disagree on the date: `07:20:23` beside
    `2026-08-13 22:20:23`. `monitor`'s stamp carries no date to warn you.
  - **Reading the page as a tutorial rather than as a set of claims found three more
    problems.** The ✅ Checkpoint asked for a local invoke, but sits *after* § 7 has destroyed
    the project and stopped the agent — it was not runnable where it stood, and it now says
    where to check it and what actually counts as passing. The Exercise had the same ordering
    problem and did not mention the restart. And § 6 never told anyone to stop the local
    server.
  - **The page never admitted the C# track is unverified.** Every ✅ block on it is Python;
    the C# commands and code are read from the sample and have never been run. § 1 now says
    so, rather than leaving a C# reader to assume the evidence covers them.
- **Walking Lab 04 a second time, from an empty directory, found five more defects — three of
  them only reachable on the path that works.** The first walk hit
  `SubscriptionNotFound` during `init` and recovered by setting environment variables by hand.
  That detour skipped part of `init` entirely, and the page was written from the detour.
  - **`init` asks *five* questions, not four.** The fifth — *Model deployment `gpt-5.4-mini`
    is defined in the azure.yaml … How would you like to proceed?* — appeared nowhere in the
    page, because the first walk aborted before reaching it. The captured evidence contained
    **zero** occurrences of it.
  - **§ 3 told the reader to run a command they do not need.** That fifth question sets
    `AZURE_AI_MODEL_DEPLOYMENT_NAME`, so `azd env set AZURE_AI_MODEL_DEPLOYMENT_NAME
    gpt-5.4-mini` is redundant here. The page had imported Lab 02's gotcha, which is real only
    because Lab 02 passes `--no-prompt` and skips the question. A completed `init` leaves
    **11** environment values; an interrupted one leaves 5.
  - **`init` prints a `Next:` block that says `cd` into the new folder.** The previous entry
    described the missing `cd` as the page losing the reader; azd was in fact telling them,
    and only the page failed to. `provision`, by contrast, prints no `Next:` block at all.
  - **`azd deploy` of unchanged code failed once with `404 / "Project not found"` — on a
    project that existed.** `az resource list`, `azd ai agent show` and a successful remote
    `invoke` all confirmed it at the same moment. Re-running unchanged succeeded in 14 s.
    Intermittent, cause not established, now a troubleshooting row.
  - **`azd down --force --purge` failed at the purge step with `409 RequestConflict`, and
    `az group exists` still returned `false`.** The resource group was gone; the Cognitive
    Services account was left soft-deleted, holding its name and its quota. The page said
    *"`false` is the only confirmation that counts"* — that is now falsified, and § 7 checks
    `az cognitiveservices account list-deleted` as well. Recovery took a manual
    `az cognitiveservices account purge` about three minutes later.
  - **The `All tenants` warning blamed the wrong thing.** A second run picked `3. All tenants`
    and reached the correct subscription without incident. The hazard is two subscriptions
    whose names differ only in the middle (`…-71560-2024-…` versus `…-118084-2025-…`).
- **Re-checking Lab 04's own verified blocks against the stored captures found four blocks
  the previous entry had silently altered.** The claim *"this was captured from a real run"*
  had never been checked mechanically, only asserted. Stripping terminal control sequences
  and nothing else, then requiring every remaining line of a `✅ Verified` block to appear
  verbatim in a capture file, showed:
  - **Two log blocks had their timestamps removed** — § 4's tool call in the `run` log and
    § 6's the same call in `monitor`. The timestamps are restored, and they turn out to
    carry the argument: `,656` → `,658` → `,660` → `,670` is the model asking, the Python
    running and the answer returning, which is what the surrounding prose claims.
  - **§ 4 also presented five non-adjacent log lines as if they were consecutive.** Two
    OpenTelemetry span dumps — `chat gpt-5.4-mini` (47 lines) and `execute_tool get_weather`
    (39 lines) — sit between them. The omission is now stated in the `<summary>`, which is
    what [`AGENTS.md`](AGENTS.md) rule 1 requires of a shortened block.
  - **§ 4's `gen_ai.tool.definitions` block had been un-escaped for readability.** The
    attribute really arrives as `"[{\"type\": \"function\", …}]"` — JSON inside a JSON
    string, because an OpenTelemetry attribute holds a string. The verified block is now the
    raw form; the readable form is kept beside it and labelled as derived from it, not as a
    separate capture.
  - **§ 6's `monitor` lines carry two clocks nine hours apart** — `22:00:19` is `monitor`
    stamping in local time, `13:00:19` is the container logging in UTC. Removing the inner
    timestamp had hidden this; a reader correlating by time rather than by `trace-id` would
    have been misled.
  - **§ 7's `az group exists` → `false` had never been captured to a file.** It has been now,
    against the same resource group, which is still absent. It is also split into its own
    `<details>`: it was sharing a `<summary>` that described the `azd down` run instead.
- **Walking Lab 04 on live Azure found the least-verified page in the repo, and closed a
  question Lab 02 had recorded as unexplained.** [`04-add-tools.md`](docs/tutorial/04-add-tools.md)
  carried exactly **one** output block for the whole lab, and it was labelled *illustrative*.
  Every section now carries a verified capture. The defects, in the order a reader hits them:
  - **§ 1 lost the reader before Azure was ever touched.** It said `mkdir 02-tools && cd
    02-tools`, then `init`, then — in § 3 — `azd provision`, with no `cd` into the folder
    `init` nests. Verified by running it: the Python command creates
    `agent-framework-agent-with-local-tools-responses/`, and `azd provision` from the outer
    folder fails with `ERROR: no project exists; to create a new project, run 'azd init'`.
    [Lab 02 § 2](docs/tutorial/02-first-agent.md) teaches this explicitly, so the page
    contradicted its own prerequisite. [Lab 05](docs/tutorial/05-mcp-toolbox.md) § 1 had the
    identical defect and is fixed with it.
  - **§ 1's command has no `--no-prompt`, so `init` asks four questions the page never showed.**
    The first — *Select a Foundry project to host your agent…* — appears nowhere in the
    tutorial, because Lab 02 uses `--no-prompt` and prints a different tail entirely.
  - **`3. All tenants` can offer a subscription Azure then refuses.** Picking one produced
    `RESPONSE 404 / SubscriptionNotFound`, and azd's own `Suggestion:` (set `AZURE_LOCATION`)
    does not recover it — the bad subscription is already in the environment. Now a warning in
    § 1 and a row in the troubleshooting table, with the mechanism marked *not established*.
  - **The lab's central claim was never evidenced: you cannot see a tool call.** A tool-backed
    answer and an invented one are the same shape in `invoke` output — no marker, no timing
    breakdown, nothing. The evidence is in the `azd ai agent run` terminal locally and in
    `azd ai agent monitor` after deploying, and both are now shown. `Function duration:
    0.001342s` local / `0.000173s` remote makes the point that the tool is never the slow part.
  - **§ 2's mapping table is now data rather than a claim.** The `gen_ai.tool.definitions`
    span attribute is the literal JSON the model receives, showing the Python docstring
    arriving as `description` and `Field(description=…)` as the parameter description.
  - **`invoke` is stateful, and the lab's own Exercise was affected.** Two separately issued
    invokes printed the same `Session:` and `Conversation:`, and the server log replayed the
    first exchange as input to the second. `evidence/help/invoke.txt` documents this; no lab
    mentioned it. The Exercise now uses `--new-session --new-conversation`, and its answer is
    backed by a measurement: one `finish_reasons: ["tool_calls"]` against six `["stop"]`, with
    `Function name: get_weather` appearing exactly once across three invokes.
  - **§ 6's "nothing is registered with Foundry" now has its evidence.** `azd ai agent show`
    returns no tools field at all. Redeploying unchanged code took **12 s** against 2 m 22 s
    for the first deploy, and left `Version` at `1`.
  - **§ 7 had no teardown verification.** `az group exists` → `false` is added, matching
    [Lab 03 § 6](docs/tutorial/03-deploy.md).
- **A seventh `doctor` state existed and was in no table.** `7 passed, 1 failed, 5 skipped` is
  what a reader gets running plain `azd ai agent doctor` after scaffolding — the documented
  `6 passed, 1 failed, 6 skipped` is that same lifecycle point run with `--local-only`, which
  [Lab 01 § 7](docs/tutorial/01-setup.md) did not say. Both rows are now qualified.
- **Skips come in three kinds, not two.** Lab 03 established *not applicable* versus
  *cascading from a failure*. A third appeared: `Developer has required role on Foundry
  project -- skipped (AZURE_AI_PROJECT_ID is not set in the current azd environment.)` — a
  missing prerequisite, which does **not** clear when you fix the topmost `(x)`.
  [Lab 01 § 7](docs/tutorial/01-setup.md) said only that skips cascade.
- **The resource-group hash suffix is explained.** [Lab 02 § 5](docs/tutorial/02-first-agent.md)
  recorded it as observed-but-unestablished. `azd ai agent init` writes a random 8-character
  `AZD_RESOURCE_TOKEN_SALT` and derives `AZURE_RESOURCE_GROUP` from it; `azd env new` writes
  none, which is why `rg-lab03-verify` has no suffix. Proven twice over: cancelling `init` at
  its first prompt left an environment holding exactly three values — salt, environment name,
  and the group name built from both — and two consecutive inits produced `21507901` then
  `aa8a1e14` with the group name following each time. **The group name is decided before Azure
  is contacted at all.**
- **The naming rule confirmed a third time, by prediction.** A 51-character environment name
  was registered before provisioning with the predicted project
  `agent-framework-agent-with-local` — its first 32 characters, a cut that lands mid-word so no
  other candidate string produces it. Exact match.
- **A claim shipped earlier in this same release was too strong, and is corrected.** Lab 03 and
  the entry below said azd's per-step deploy lines "only exist in a redirected capture". They do
  not: a `script -qec '…' /dev/null` capture — a real pty, `test -t 1` true — produced the
  per-step form, and still did after the pty was given an explicit size with `stty rows 40 cols
  200`. The user-side capture of the same command contains **514** escape sequences; the
  agent-side one contains **9**. A redirect always degrades the output, but a pty does not
  always preserve it, and the mechanism is not established. [`AGENTS.md`](AGENTS.md) rule 1 now
  warns that a pty is necessary but not sufficient and gives the escape-sequence count as the
  check.
- **Two retractions from this walk, recorded because the reasoning was wrong, not the fix.**
  A `SubscriptionNotFound` failure was attributed to a stale `AZURE_TENANT_ID`; testing showed
  azd had written exactly the tenant `az account show` reports, so the hypothesis is withdrawn
  and the mechanism left open. Separately, the generic `To fix:` / `Review the fix: notes
  above` block was reported as undocumented; it is already in
  [Lab 02 § 4](docs/tutorial/02-first-agent.md). Neither reached the docs.
- **Walking Lab 03 on live Azure found eleven defects, and disproved a rule this repo had been
  repeating.** The page said the Foundry project name was random, and [Lab 02](docs/tutorial/02-first-agent.md)
  — corrected earlier in this same release — said it came from the *service* name. Both were
  wrong. A prediction test settled it: provisioning into an environment called `lab03-verify`
  produced the project `lab03-verify` and the group `rg-lab03-verify`, where the service was
  still `agent-framework-agent-basic-responses`. **The project name is the azd environment name
  cut to 32 characters, and the resource group is `rg-` plus that name; only the Cognitive
  Services account is random.** The two earlier runs could not distinguish the hypotheses
  because the service name and the environment name share their first 32 characters. It also
  explains `rdpy`, which this repo had recorded as an unexplained random string for months: the
  environment was named `rdpy`. Lab 03's opening NOTE now maps all four substitutions to their
  sources, and Lab 02 § 5 carries the rule.
- **`azd deploy` renders a live table; the page showed a redirect.** A terminal draws
  `Service / Status / Duration` and rewrites the rows in place; the per-step lines the page
  printed (`hello-world: Deploying (Polling agent status (1/30)) [57s]`) only exist in a
  redirected capture. This also closes a question left open during the Lab 02 walk, where a
  table format was observed but could not be attributed — it belongs to `azd deploy`. The
  block now also carries the four things a terminal shows and the old capture had dropped: the
  `aka.ms/azd-agents-invoke` line, an unsolicited `eval generate` suggestion, the `Next:` block,
  and the portal URL printed after `SUCCESS`.
- **`azd down`'s verified block listed lines that no terminal keeps.** `Listing Cognitive
  Services accounts in …`, `Deleting model deployment …` and `Purging soft-deleted …` are
  progress lines rewritten in place; a pty capture of the whole command is four lines long.
  Both the § 6 block and the Checkpoint carried the redirect form. Both replaced, and the
  Checkpoint now also verifies teardown with `az group exists`, because `SUCCESS` is the
  command's opinion and `false` is Azure's. The single `1m46s` timing is replaced by the
  measured range across four runs (1m46s – 2m40s).
- **The tutorial contradicted itself about skipped checks, and the wrong side was in bold.**
  Lab 03 § 4 asserted *"A skip means 'not applicable', never 'broken'"*, while
  [Lab 01 § 7](docs/tutorial/01-setup.md) said, correctly, that skips cascade from failures.
  Measured: with the project endpoint broken, two checks skip with the reason
  `(Foundry endpoint did not respond (see check `remote.foundry-endpoint`).)` — naming the
  upstream check. The parenthesis is how the two kinds of skip are told apart, and the page had
  been **truncating it away** from both skip lines while paraphrasing its contents in prose.
- **A sixth `doctor` state.** `9 passed, 1 failed, 3 skipped` — endpoint set, project
  unreachable — is reachable by anyone who tears down and comes back. Added to Lab 01 § 7,
  whose table listed five.
- **`SUCCESS: Your application was provisioned` does not mean the project is usable.** Observed
  2026-08-12: provision succeeded twice, ARM reported `Succeeded` for the account and the
  project, the model deployment existed, and `properties.endpoints["AI Foundry API"]` matched
  `FOUNDRY_PROJECT_ENDPOINT` exactly — yet the data plane answered `404 / "Project not found"`
  for over half an hour, so `azd deploy` failed. `doctor` diagnoses it precisely; its suggested
  fix (`azd provision`) does **not** work, because nothing is missing. Recovery was
  `azd down --force --purge` into a new environment. Documented in Lab 03's troubleshooting
  with the cause explicitly marked unknown.
- **`azd ai agent monitor` was introduced as "Live logs" but does not stream.** Bare, it fetches
  `--tail` (default 50) and exits `0` — measured at 53 lines, returning in seconds under a
  `timeout 25`. `--help` says exactly this, and the CLI's own `Next:` block recommends
  `--follow`. Corrected, with a troubleshooting row.
- **Two claims that had no evidence behind them now have it.** § 1 asserted that
  `azd ai agent eval` names the environment variable it resolved the agent from; it does, in a
  `Detected eval target:` header that the page never showed, and that header is now a verified
  block. § 3 said remote invocations are traced; the `Trace ID` it prints is the same value the
  container log carries as `x-request-id` and `trace-id`, so it can be grepped and not only
  pasted into the portal.
- **The hosted agent is where the identity model becomes visible.** The container log shows
  `DefaultAzureCredential acquired a token from ManagedIdentityCredential`, against the same
  code that falls back to `AzureCliCredential` on a laptop in Lab 02 § 6. Added, along with the
  benign `Content type 'usage' is not supported yet` warning and the fact that the hosted agent
  calls the same model endpoint you called locally.
- **Verified with no changes needed:** the five injected `AGENT_<SERVICE>_*` variables and their
  exact names; `azd ai agent show`'s seventeen table fields, their order, and every field the
  § 2 NOTE claims the JSON adds (`cpu`, `memory`, `dependency_resolution: remote_build`,
  `runtime: python_3_13`, `protocol_versions: [{responses, 2.0.0}]`); `azd ai agent list` not
  existing; the required-argument error from a bare `eval generate`; `11 passed, 0 failed,
  2 skipped` and its exit code `0`; and the "13-point diagnostic" count.

- **Walking Lab 02 on a second machine, against live Azure, found sixteen more defects.** Every
  step from `sample list` to teardown was run by hand and its real output compared with the
  page. Two classes dominated.
  - **Numbers and names copied from the wrong run.** § 5 timed `azd provision` at `1m20s`,
    which is the `01-hello-world` figure from 2026-08-08; both this lab's captured run and the
    2026-08-12 re-run took **1m24s**, exactly as
    [`docs/reference/README.md`](docs/reference/README.md#verified-runs) already recorded. The
    resource names told the same story: the page showed `cog-56mzb54ouruu6/rdpy`, a random
    four-character project name that this flow never produces. The catalog sample names its
    project after the service, truncated to 32 characters —
    `cog-…/agent-framework-agent-basic-resp` — so a reader checking their own output against
    the page would find it structurally different. Both corrected, and the
    `FOUNDRY_PROJECT_ENDPOINT` example with them.
  - **Verified blocks that were quietly edited.** Two `doctor` lines had been hand-wrapped to
    fit the page (the real ones are 147 and 133 characters, unwrapped); the `RuntimeError` in
    the model-name gotcha had been split across two lines; twelve identical
    `Foundry deployment in progress` lines had been collapsed into `…`; and the `run` startup
    log had fields deleted from the *middle* of lines with no ellipsis. All restored verbatim.
    [`AGENTS.md`](AGENTS.md) rule 1 now forbids re-wrapping a verified block, and requires pty
    capture — a redirect drops azd's `Next:` blocks entirely while adding spinner lines no
    reader sees, which is where five of these defects came from.
- **The `run` section blamed the wrong component for the scary traceback.** The NOTE described
  a `169.254.169.254` timeout as `DefaultAzureCredential` probing for a managed identity, but
  quoted the URL `/metadata/instance/compute`. There are in fact **two** such spans and the
  page conflated them: the startup one is the OpenTelemetry distro's Azure-VM resource detector
  (`python-requests`, 0.2 s), and the credential one appears later on first invoke against
  `/metadata/identity/oauth2/token` (`azsdk-python-identity`, 1 s), immediately followed by
  `DefaultAzureCredential acquired a token from AzureCliCredential`. Both are now documented,
  with the evidence that distinguishes them.
- **`azd ai agent run` tells you to `curl` a URL that cannot work for this sample.** Its
  terminal hint points at `/invocations/docs/openapi.json`, which is registered only by the
  Invocations protocol package; this sample speaks Responses and serves `POST /responses`. The
  CLI probes that URL itself at startup and logs its own `404` one line above the hint. Now
  carries a warning naming the cause, so a reader who follows the CLI's advice knows the 404 is
  not theirs.
- **Nothing said that running "locally" still bills you.** § 4 says everything before
  `provision` is free, which is true, and then the local loop calls
  `POST …/openai/v1/responses` for every turn. The emitted span records the token counts.
  Stated explicitly.
- **`doctor` does not check the two values § 3 sets.** Measured: running it *before*
  `azd env set` gives the identical `6 passed, 1 failed, 6 skipped`, with `manual env vars set`
  green while `AZURE_SUBSCRIPTION_ID` and `AZURE_LOCATION` are unset. A section titled *check
  before you spend money* now says what it cannot check.
- **The fifth `doctor` state was undocumented.** After `provision` and before `deploy` it
  reports `10 passed, 1 failed, 2 skipped`, the single failure being `Hosted agents are active`
  with `fix: Run azd deploy`. Captured and added to Lab 02 § 5. Lab 01 § 7 claimed there were
  "three states you will actually see" and listed three; the table now lists all five with
  their counts.
- **Terminal-only output that the page never showed.** `init`, `run` and `invoke` each end with
  a `Next:` block naming the next command — including the `cd` that Lab 02 spends a callout
  insisting on, and `azd ai agent monitor --follow`, which no lab had mentioned. `sample list`
  ends with a line pointing at `--output json`. `init` prints a four-line
  `Set the missing values before running azd provision` remediation, including the optional
  `AZURE_AI_DEPLOYMENTS_LOCATION`. `run` prints
  `Activate with: source .venv/bin/activate.<shell>`. All were absent because they do not
  survive a redirect. All restored.
- **The subscription ID leaks in three new places**, now flagged where each appears: the
  `Subscription: <name> (<guid>)` line and portal URL that `provision` prints, and
  `microsoft.foundry.project.id` — a full ARM resource ID — on **every** OpenTelemetry span the
  running agent writes to stdout, alongside the prompt, the reply and token counts. In a
  captured run, 373 of 732 stdout lines were span JSON; the page had shown a tidy four-line
  startup log.
- **`-o table` drops `id` *and* `type`, and the fix is one character.** Lab 01 § 4 documented
  the `id` case as an unavoidable quirk and worked around it with a second command. Asking for
  the same value under three key names showed that only the lowercase `id` and `type` columns
  vanish — `{sub:name, Id:id}` renders both. Lab 01 § 4 now states the rule, and Lab 02 § 5's
  `az resource list` query uses `Type:type` so its output block matches what the reader gets.
- **Smaller corrections from the same run.** `sample list` is ordered six `featured` entries
  first then the other fifteen, each group alphabetical, so the sample this lab uses is 5th —
  the page said only "not the first one listed"; `featured` and `recommended` are invisible in
  the text form, which is the practical reason to use `--output json`; four more sample titles
  besides `01-basic` ship in both protocols; `init` creates *and selects* an environment named
  `<project>-dev`, so `azd env new` is never needed; `.env` is created mode `0600` and matched
  by `.gitignore` line 1; and `azure.yaml`'s `runtime: python_3_13` is the hosted runtime, not
  the `CPython 3.14.3` that `run` fetches locally — verified on a machine whose system Python
  is 3.12.3.
- **Lab 03 understated how far its capture diverges from Lab 02.** Its opening NOTE said "only
  the agent name differs", but its Playground URL and endpoint also embed a Cognitive Services
  account and a Foundry project name that a Lab 02 reader will not have. The NOTE now names all
  three substitutions.

- **Walking Lab 01 on a second machine found eight more defects.** A reader ran every step and
  pasted real output; each divergence from the page is fixed below.
  - **`az account show --query "{sub:name, id:id, user:user.name}" -o table` never printed the
    ID it asked for.** Azure CLI's table renderer drops an `id` column silently — the same
    query with `-o json` returns it. The very next instruction was
    `az account set --subscription <subscription-id>`, and Lab 02 § 3 asks for the same value,
    so the tutorial demanded an ID it gave the reader no way to obtain. Now shows
    `az account show --query id -o tsv` as its own step, with the trap documented.
  - **The Checkpoint told the reader to run bare `az account show`,** which prints tenant ID,
    subscription ID, tenant display name and tenant domain. A checkpoint is precisely the
    output people paste into issues when asking for help — and this repo's own issue forms ask
    for command output. Narrowed to `--query "{sub:name, user:user.name}"`, with a CAUTION.
    Lab 01 § 4 was already using a narrowed query, so the page disagreed with itself.
  - **The `azd extension upgrade --all` block matched neither a terminal nor a log file.** It
    omitted the two-line command banner that always appears, and printed one
    `Upgrading <extension>` line per extension — but a terminal shows none of them (`azd`
    rewrites that line in place) and a redirect shows two of each. The block had been
    hand-trimmed from a redirected capture into a form that exists nowhere. Replaced with what
    a terminal shows, plus a note explaining why CI logs look different.
  - **`azd extension list --installed` was missing its `SOURCE` column and separator rule.**
  - **`azd version` output omitted the trailing `(stable)`** — the release channel, which is
    the first thing worth knowing when someone reports odd behaviour.
  - **The Checkpoint's `doctor` capture was one line short,** missing
    `Then re-run \`azd ai agent doctor\` to verify.`
  - **§ 8's new warning quoted the CLI's bad advice without correcting it.** The error says
    *"to create a new project, run `azd init`"*; that is the wrong command, and the correction
    lived only in the Checkpoint TIP — *after* § 8. A reader going in order met the misleading
    advice first.
  - **Nothing said which shell the command blocks assume.** They assume `bash`/`zsh`, and
    `fish` breaks on `echo $?` (used in Lab 09) and on `VAR=value azd …` (used in
    `reference/`). Both substitutions, plus the `code` → `code-insiders` swap for VS Code
    Insiders, are now stated once in § 1.
- **Confirmed unchanged, on a second machine, 2026-08-12:** all five extension versions, the
  `ms-windows-ai-studio.windows-ai-studio` marketplace ID (HTTP 200) and the 404 on the ID the
  official docs link to, `azd env set` refusing outside a project with exit `1`, and the
  Checkpoint's `1 passed, 1 failed, 11 skipped`.
- **Labs 01 and 02 contained four more commands or blocks that cannot be run where they are
  printed.** The `11/0/2` checkpoint fixed below was not an isolated slip but one instance of
  a pattern, so both pages were audited line by line against a real machine:
  - **Lab 01 §7 showed `azd ai agent doctor`, but the output under it was captured with
    `--local-only`** — five lines read `skipped (remote check excluded by --local-only)`.
    A reader running the printed command gets a different result.
  - **Lab 01 §7's `6 passed, 1 failed, 6 skipped` block needed a project Lab 01 never
    creates.** Lab 01 contains no `azd ai agent init` — every mention of `init` on the page is
    a forward reference. The block, and the "no environment selected" variant beside it, have
    moved to a new **Lab 02 § 4**, placed between `env` and `provision` so it earns its keep:
    it is the last free checkpoint before the first billable command. Lab 01 §7 keeps the
    heading (it is linked from the glossary) and now teaches only how to *read* `doctor`.
  - **Lab 01 §8 told the reader to run `azd env set`, which fails there.** Verified
    2026-08-12: outside a project it exits `1` with `ERROR: no project exists`. `azd env set`
    needs an environment, and the first one is created by `azd ai agent init` in Lab 02. The
    section now carries the region/quota decision and defers the commands to Lab 02 § 3.
  - **Lab 02 § 2 explained the mandatory `cd` in prose but never gave it as a command.** The
    IMPORTANT callout said to `cd` into the nested folder; the next step went straight to
    `azd env set`, which fails from the parent. `cd` is now a step, with `ls azure.yaml` to
    confirm it worked.
- **Lab 02 § 1's JSON block was an undisclosed excerpt.** It showed 4 of the 9 fields the CLI
  actually returns and elided the `initCommand` URL to `…/01-basic/azure.yaml`, so the two
  fields that matter most were invisible: `recommended`, which is how you identify the
  starting sample, and the full ready-to-paste `initCommand`. Replaced with one complete
  entry, verbatim. The `text` form is now shown too — it is the default, so it is what the
  reader sees first, and the page previously described only the `json` shape.
- **Lab 02 § 1 now warns that two samples are called `01-basic`.** `agent-framework/responses/`
  and `agent-framework/invocations/` each ship one, with titles differing by a single word.
  This lab depends on the *responses* variant; matching on `01-basic` alone is a coin flip,
  and the invocations variant speaks a different wire protocol.
- **Lab 02's flow diagram was numbered `① init … ⑩ down`, which matched neither the lab's own
  section numbers nor its scope** — half those steps belong to Lab 03. It is now split into
  two labelled subgraphs with the numbering removed.
- **Lab 01's opening line promised the wrong outcome.** *"Go from a clean machine to
  `azd ai agent doctor` reporting all green"* is not what Lab 01 does, and directly
  contradicted the callout added below explaining that it cannot.
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
