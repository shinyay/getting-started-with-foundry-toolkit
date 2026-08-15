# Support

This is a personal, best-effort guide, not a supported product. There is no SLA and
Discussions are not enabled — everything routes through issues, and this page exists only so
the [new-issue chooser](https://github.com/shinyay/getting-started-with-foundry-toolkit/issues/new/choose)
reads as a decision rather than as four competing invitations.

**Start here, in this order.**

| Your situation | Where to go |
|---|---|
| A command from the guide failed | [Troubleshooting](docs/reference/troubleshooting.md) first — it is real captured failures, not guesses, and it covers the ones that break most first runs. |
| It is still failing, and the guide says it should work | [Report a documented command that no longer works](https://github.com/shinyay/getting-started-with-foundry-toolkit/issues/new?template=stale-command.yml). This is the **most useful report this repo can receive**; [`CONTRIBUTING.md`](CONTRIBUTING.md) explains why. |
| The output you got differs from a ✅ Verified block | Same form. Paste what you actually saw — never what it should have said. |
| A page is wrong, unclear, or contradicts another page | [Open a docs issue](https://github.com/shinyay/getting-started-with-foundry-toolkit/issues/new?template=docs-issue.yml). |
| `azd ai agent` itself is broken | Upstream: [microsoft/foundry-toolkit](https://github.com/microsoft/foundry-toolkit/issues). Opening one here too is welcome, so the guide can warn readers meanwhile. |
| The preview changed under you | [`WHATS_NEW.md`](https://github.com/microsoft/foundry-toolkit/blob/main/WHATS_NEW.md) upstream is the only continuously-maintained source, and usually explains it. |
| You found something that might be a leaked credential | **Not an issue.** Use the private channel in [`SECURITY.md`](SECURITY.md). |

## What this repo cannot help with

Azure billing, quota and subscription problems; debugging your own agent code; and anything
about Foundry that is not documented on one of these pages. Those belong to
[Azure support](https://azure.microsoft.com/en-us/support/) or upstream.

## Before you file

Have your `azd version` and `azd extension list --installed` output ready. Every page here is
pinned to `1.0.0-beta.*` previews, so those two commands usually explain the difference
between your run and the captured one on their own — the canonical versions this guide was
verified against are in [`docs/reference/README.md`](docs/reference/README.md).
