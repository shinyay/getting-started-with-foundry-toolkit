# Contributing

Thanks for helping keep this guide honest.

This repository is **documentation plus runnable samples** for the Microsoft Foundry Toolkit.
It ships no package and exposes no API. Its only real asset is that **every output block
labelled *verified* was captured from a real run against real Azure**, so the single most
valuable contribution is evidence that something no longer matches reality.

> [!IMPORTANT]
> **The rules live in [`AGENTS.md`](AGENTS.md), not here.** That file is the contract — seven
> rules, the three-layer contract, and the copy-pasteable runner for the ten CI checks. It
> is written for coding agents but applies identically to humans, and this page deliberately
> does not repeat it, because a rule stated in two places drifts in one of them.
>
> Read it before your first change: [rules](AGENTS.md#rules-when-editing-this-repo) ·
> [validation](AGENTS.md#validation-before-committing).

---

## The most useful thing you can report

This guide is pinned to a **preview** toolchain — the `azd` extensions are all
`1.0.0-beta.*`. Its characteristic failure is therefore not a typo: it is a **documented
command that silently stops working** because an extension shipped a breaking change.
Nothing in CI can detect that, because none of the ten checks call Azure or run `azd`.
Only a reader hitting it can.

→ **[Report a documented command that no longer works](https://github.com/shinyay/getting-started-with-foundry-toolkit/issues/new?template=stale-command.yml)**

That form asks for the page, the block, your `azd version`, your
`azd extension list --installed`, and the actual output — which is exactly what is needed to
re-verify the block rather than argue about it.

## What a good change looks like

| If you are… | Then… |
|---|---|
| correcting a **verified** block | re-run the command and paste what you actually saw. Do not edit it to what it *should* say |
| adding a command, flag or default | check it against [`evidence/help/`](evidence/) first — 52 verbatim `--help` captures are the authority |
| documenting something you could not run | say so in the page. A gap stated out loud is worth more than a plausible guess |
| adding an explanation | put it in `docs/learn/`; put the commands in `docs/tutorial/` and link between them |

### The two labels

Every output block in `docs/` carries one of these, and they are not interchangeable:

| Label | Means |
|---|---|
| **✅ Verified `<date>`** | captured character-for-character from a real run on that date |
| **Illustrative** | a pattern to adapt, or a shape derived from docs — **never executed** |

If you change a command inside a ✅ block and cannot re-run it, downgrade the block to
*illustrative* in the same change. Leaving it marked verified is the one edit this repo
treats as a defect.

## Before you open a pull request

1. Run the ten checks locally — the runner is in
   [`AGENTS.md`](AGENTS.md#validation-before-committing). None of them need Azure, network
   access or credentials, so there is no reason not to.
2. Add an entry under `## [Unreleased]` in [`CHANGELOG.md`](CHANGELOG.md).
3. Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) —
   this repo uses `docs:`, `fix:` and `ci:`.

## Versioning

Releases are **CalVer** (`vYYYY.0M.PATCH`), not SemVer, because the only question a reader
has is *"how stale is this?"*. The policy is in
[`CHANGELOG.md`](CHANGELOG.md#versioning-policy).

## Scope

This repo documents the toolkit; it is not the toolkit. Bugs in `azd ai agent` itself belong
[upstream](https://github.com/microsoft/foundry-toolkit/issues) — though a note here is still
welcome, so the guide can warn readers while the upstream fix is pending.
