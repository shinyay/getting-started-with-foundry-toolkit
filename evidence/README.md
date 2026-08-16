# Evidence

This repository claims that every command, flag and output block it shows was taken from a
real run. This folder is where that claim is kept falsifiable.

[`AGENTS.md`](../AGENTS.md) rule 1 says *never invent CLI output*, and *if you change a
command, re-run it or mark the block as illustrative*. Without a captured baseline in the
tree, "re-run it" has nothing to compare against and the rule is unenforceable. That is
what this folder fixes.

---

## What is here

`help/` — **53 files, 228 KB** of verbatim `--help` output. This is the authority behind
every flag name, default, subcommand list and shorthand asserted anywhere in `docs/`.

| Group | Files | Covers |
|---|---:|---|
| `azd ai agent` tree | 43 | `_root.txt` plus every subcommand and sub-subcommand |
| `azd ai toolbox` tree | 2 | `toolbox.txt`, `toolbox-create.txt` |
| `azd extension` tree | 4 | `ext.txt`, `ext-install.txt`, `ext-update.txt`, `ext-upgrade.txt` — see below |
| Negative evidence | 4 | commands that **do not exist** — see below |

Files prefixed `n-` are second-level captures (`n-files-upload.txt` is
`azd ai agent files upload --help`). Files prefixed `ext-` are the `azd extension` tree,
which lives in **core `azd`** rather than in an extension.

---

## Negative evidence — do not delete these as "failed captures"

Four files contain nothing but an error. That error *is* the finding:

```text
ERROR: unknown command "deploy" for "agent"
```

`deploy.txt`, `env.txt`, `logs.txt` and `provision.txt` prove that **`azd ai agent deploy`,
`env`, `logs` and `provision` do not exist**. This matters because `azd deploy` and
`azd provision` *do* exist at the root, so the natural assumption is wrong. A reader who
guesses gets a hard failure; these four captures are why the docs never suggest it.

---

## Why the `azd extension` tree is captured at all

Every other file here comes from an extension. These three come from **core `azd`**, and
they were added because a rename proved the gap was dangerous.

`azd 1.31.0` renamed `azd extension upgrade` to **`azd extension update`** and kept the old
name as an alias. Ten places in this repository tell a reader to run the old name, and
[`docs/tutorial/01-setup.md`](../docs/tutorial/01-setup.md) quotes its output verbatim — yet
nothing in `help/` backed any of it, because the 49 original captures cover only
`azd ai agent` and `azd ai toolbox`.

The three captures pin down what a reader actually gets:

| File | Command | What it proves |
|---|---|---|
| `ext.txt` | `azd extension --help` | the subcommand list names `update` and **does not list `upgrade` at all** |
| `ext-install.txt` | `azd extension install --help` | `--version` can pin a version and `--force` allows downgrades — the flags behind the claim that the verified toolchain is *partly* reproducible |
| `ext-update.txt` | `azd extension update --help` | the flag is `--no-dependency-updates` |
| `ext-upgrade.txt` | `azd extension upgrade --help` | byte-identical to `ext-update.txt` |

So `upgrade` still works but is **undiscoverable** — it appears in no help output. That is
the worst shape a deprecation can take for a documentation repository: the old command keeps
passing, so nothing fails, while every reader who explores the CLI is told it does not exist.

---

## A trap these captures expose

Two files are byte-identical to their parents:

| File | Identical to |
|---|---|
| `n-sample-show.txt` | `sample.txt` |
| `n-sample-list.txt` | `sample-list.txt` |

The first pair is not sloppy capture — it is a real behaviour. `azd ai agent sample` has
exactly one subcommand, `list`. Asking for help on a subcommand that does not exist under
it prints the **parent** help and exits **0**:

```console
$ azd ai agent sample show --help ; echo $?
Browse the curated catalog of agent samples and azd templates.
...
0

$ azd ai agent deploy --help ; echo $?
ERROR: unknown command "deploy" for "agent"
1
```

So an unknown *top-level* agent subcommand fails loudly, while an unknown *nested* one
under `sample` succeeds silently. This is the same class of hazard as
`azd ai agent invoke` returning 0 on an empty response: **exit codes from this toolkit are
not a reliable success signal.** Never gate CI on them.

> Verified 2026-08-12 by direct probe, not inferred from the capture files.

---

## Provenance

The 49 extension captures and the 3 core-`azd` captures were taken at different times,
because the second group was added in response to a rename. Both rows are exact.

| Group | Captured | `azd` | Extensions |
|---|---|---|---|
| the 49 `azd ai …` captures | **2026-08-08** | **1.30.0** | `azure.ai.agents` 1.0.0-beta.9 · `azure.ai.connections` 1.0.0-beta.4 · `azure.ai.inspector` 1.0.0-beta.3 · `azure.ai.projects` 1.0.0-beta.5 · `azure.ai.toolboxes` 1.0.0-beta.5 |
| the 4 `azd extension` captures | **2026-08-15** | **1.31.1** | not applicable — `azd extension` is core `azd` |

| Re-checked | |
|---|---|
| **2026-08-12** | `azd extension list --installed` reported all five *Up to date*, so the captures still described current behaviour |
| **2026-08-15** | the extensions have since moved forward, and the 49 captures were **re-taken and diffed** rather than assumed. Three files differ; the exact diff is recorded in the [canonical table](../docs/reference/README.md#newer-than-the-verified-toolchain). The committed captures deliberately still describe the toolchain the labs were verified on |

The canonical version table lives in [`docs/reference/README.md`](../docs/reference/README.md)
and CI check 7 enforces that the rest of the repo agrees with it. If the two ever disagree,
that table wins and these captures are stale.

---

## What is deliberately not here

Live Azure run logs — provision output, deploy timings, agent cards, generated
infrastructure — are **not** committed. They contain subscription IDs, resource names and
endpoints, and the repo's visibility is still undecided. The verified excerpts that the
docs actually rely on are quoted inline in `docs/reference/`, which is where a reader needs
them anyway.

`help/` was chosen because it is the highest-value, lowest-risk slice: it backs the largest
number of claims, and it contains no tenant data, no local paths and no credentials.

---

## How to re-verify

To confirm a flag or subcommand the docs assert, read the capture first:

```bash
grep -n "protocol" evidence/help/invoke.txt
```

To check whether the toolkit has moved under you:

```bash
azd version
azd extension list --installed        # compare INSTALLED against LATEST
```

If any version differs from the table above, re-capture before trusting a page:

```bash
azd ai agent invoke --help > evidence/help/invoke.txt
```

Then update [`docs/reference/README.md`](../docs/reference/README.md), let CI check 7
propagate the version change, and record it in [`CHANGELOG.md`](../CHANGELOG.md).

---

## The one rule

**A capture is worth nothing if it is edited.** These files are verbatim. If output must be
shortened or redacted for a docs page, do it in the docs page and say so there — never in
`help/`.
