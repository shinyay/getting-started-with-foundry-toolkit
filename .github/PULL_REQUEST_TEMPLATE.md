<!-- Rules: https://github.com/shinyay/getting-started-with-foundry-toolkit/blob/main/AGENTS.md
     Human entry point: https://github.com/shinyay/getting-started-with-foundry-toolkit/blob/main/CONTRIBUTING.md -->

## What changed, and why

<!-- One paragraph. Link the issue if there is one. -->

## Evidence

<!-- Delete the rows that don't apply. -->

| | |
|---|---|
| Does this add or change an output block? | yes / no |
| If yes, was it **re-run** or marked **illustrative**? | |
| Toolchain it was run against | `azd …`, `azure.ai.agents …` |
| Flag / subcommand claims checked against `evidence/help/` | yes / n/a |

> A block left labelled **✅ Verified** whose command changed is the one defect this repo
> treats as serious. If you could not re-run it, downgrade it — that is a valid fix.

## Checklist

- [ ] The [nine checks](https://github.com/shinyay/getting-started-with-foundry-toolkit/blob/main/AGENTS.md#validation-before-committing) pass locally (none need Azure, network or credentials)
- [ ] No CLI output was written from memory
- [ ] Content is in the right layer — `learn/` explains, `tutorial/` is hands-on, `reference/` is lookup
- [ ] Version numbers, timings and explanations are stated once, in `docs/reference/`, and linked from elsewhere
- [ ] `CHANGELOG.md` has an entry under `## [Unreleased]`
