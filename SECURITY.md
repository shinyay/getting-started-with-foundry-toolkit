# Security policy

This repository is **documentation and runnable samples** for the Microsoft Foundry Toolkit.
It ships no package, exposes no API and runs no service, so it has almost no attack surface of
its own. What it does have is **verbatim output from real Azure runs**, and that is where a
genuine report is most likely to come from.

## Reporting

**[Open a private security advisory](https://github.com/shinyay/getting-started-with-foundry-toolkit/security/advisories/new)**
— private vulnerability reporting is enabled on this repository.

Please do **not** open a public issue for anything you believe is a live credential, key or
token. A public issue is the one place a suspected exposure should never go first, which is
why neither [issue form](https://github.com/shinyay/getting-started-with-foundry-toolkit/issues/new/choose)
offers that route.

This is a personal, best-effort project with no on-call rotation, so no response time is
promised — stated here rather than left to be assumed.

## Identifiers published on purpose

Several ✅ Verified blocks contain **real Azure identifiers**. They are left in deliberately:
[rule 1](AGENTS.md#rules-when-editing-this-repo) forbids editing captured output, and
redacting a value is editing it. The decision is recorded in
[issue #2](https://github.com/shinyay/getting-started-with-foundry-toolkit/issues/2).

| Identifier | Where | Why publishing it is safe |
|---|---|---|
| A subscription ID, in full | [`docs/tutorial/07-container-mode.md`](docs/tutorial/07-container-mode.md) | Names a subscription; grants nothing without an identity that has a role on it. Every resource it created has been destroyed, and each teardown was confirmed with `az group exists` **and** `az cognitiveservices account list-deleted`. |
| A tenant ID, in full | [`docs/reference/identity-and-rbac.md`](docs/reference/identity-and-rbac.md) | **Discoverable by design.** Any tenant's ID can be read from its public `.well-known/openid-configuration` endpoint, given only a domain name. |
| A managed identity's principal ID | [`docs/reference/identity-and-rbac.md`](docs/reference/identity-and-rbac.md) | Belonged to a Cognitive Services account that no longer exists. |
| Session, conversation and trace IDs | throughout the labs | Scoped to one request against resources that are gone. |

Both the subscription and tenant IDs also appear truncated in
[`docs/reference/environment-variables.md`](docs/reference/environment-variables.md).

**None of the above is a credential.** If you find something that *is* — a key, an access
token, a connection string, a client secret or a SAS URL — that is a real finding, and the
private channel above is the right place for it. Verified blocks are pasted from live runs,
so this is a plausible failure mode rather than a theoretical one.

## Out of scope here

| Finding | Where it goes |
|---|---|
| A bug or vulnerability in `azd ai agent` itself | Upstream: [microsoft/foundry-toolkit](https://github.com/microsoft/foundry-toolkit/issues). This repo documents the toolkit; it is not the toolkit. |
| A documented command that silently stopped working | Not a vulnerability, and the most useful thing you can report — use the [stale-command form](https://github.com/shinyay/getting-started-with-foundry-toolkit/issues/new?template=stale-command.yml). |
| Samples not being production-hardened | By design. Each teaches one idea; none is a template for production. Open a [docs issue](https://github.com/shinyay/getting-started-with-foundry-toolkit/issues/new?template=docs-issue.yml) if a page implies otherwise. |

Everything else about contributing: [`CONTRIBUTING.md`](CONTRIBUTING.md).
