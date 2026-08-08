# 🔐 Identity & RBAC

> **Captured live on 2026-08-08** against `azd 1.30.0` / `azure.ai.agents 1.0.0-beta.9`,
> from a real `azd provision` + `azd deploy` of
> [`samples/csharp/01-hello-world`](../../samples/csharp/01-hello-world/) in `eastus2`.
> Every ID below is from that run and has since been purged.

The single most surprising thing about Foundry hosted agents: **there are three different
identities in play**, and confusing them is the root of most "403 Forbidden" bug reports.

---

## The three identities

| # | Identity | Who has it | What it is for |
|---|---|---|---|
| 1 | **You** (user principal) | your `az login` / `azd auth login` session | Creating resources, deploying, invoking from your laptop |
| 2 | **The account identity** | the `Microsoft.CognitiveServices` account | The Foundry account reaching its own dependencies |
| 3 | **The agent instance identity** | **each deployed agent, individually** | What *your running container* authenticates as |

Identity 3 is the one people miss. Your agent code calling `DefaultAzureCredential()` locally
is **you**; the same line running in Foundry is **identity 3**. Code that worked on your
laptop and 403s in the cloud is nearly always this.

---

## Identity 2 — the account

`azd provision` creates the account with a **system-assigned** managed identity:

```bash
az cognitiveservices account show -n <account> -g <rg> --query identity
```

```json
{
  "principalId": "f6b68f7f-bc8c-4f8c-927e-e646e1771b56",
  "tenantId": "16b3c013-d300-468d-ac64-7eda0820b6d3",
  "type": "SystemAssigned",
  "userAssignedIdentities": null
}
```

`SystemAssigned` means its lifecycle is bound to the account — delete the account and the
principal disappears. That is convenient (nothing to clean up) and a constraint: you cannot
pre-create the principal and grant it roles *before* the resource exists, which is exactly
what a locked-down "grant access first, deploy second" pipeline wants to do. If you need
that, you want a user-assigned identity, which is outside the default `azd` flow.

---

## Identity 3 — the per-agent instance identity

This is the important one, and `azd ai agent show` is the only place it surfaces:

```bash
azd ai agent show hello-world
```

```text
Agent GUID                       4e7e0e00-af5c-462b-a40b-09657fc5e964
Instance Identity Principal ID   af65ac56-3c8b-4fc9-ba35-9cb1a37e52da
Instance Identity Client ID      af65ac56-3c8b-4fc9-ba35-9cb1a37e52da
Blueprint Principal ID           68c06f0f-7eee-48d0-8d6b-ddeb81f5c1bb
Blueprint Client ID              6371fa08-38ff-483f-aadf-98edb2ecb0af
Blueprint Reference Type         ManagedAgentIdentityBlueprint
Blueprint Reference ID           hello-world-4e7e0
```

Four facts worth reading twice:

1. **Every agent gets its own principal.** Not one per project, one per *agent*. Deploy two
   agents and you have two principals to grant roles to. This is good security design — a
   compromised agent cannot reach what its neighbour can — and an operational cost nobody
   warns you about.
2. **`Instance Identity Principal ID == Client ID`.** For the account identity these differ;
   here they are the same value. Don't "fix" it when you see it.
3. **`Blueprint Principal ID != Blueprint Client ID`.** The blueprint is a *template* the
   instance identity is minted from — it is not the thing your container authenticates as.
   Granting a role to the blueprint principal does **not** grant it to your agent.
4. **`Blueprint Reference ID` is `<agent-name>-<first 5 of GUID>`** (`hello-world-4e7e0`).
   Useful when you are staring at a portal list and trying to work out which agent is which.

### Proof it is really the identity your code uses

The agent's own container logs show the token cache partitioned by that exact principal:

```bash
azd ai agent monitor hello-world --tail 15
```

```text
16:46:59  stdout  MSAL 4.84.2.0 MSAL.NetCore .NET 10.0.9 Linux
                  [FindAccessTokenAsync] Discovered 0 access tokens in cache using
                  partition key: af65ac56-3c8b-4fc9-ba35-9cb1a37e52da_managed_identity_AppTokenCache
16:46:59  stdout  [LogMetricsFromAuthResult] Cache Refresh Reason: NoCachedAccessToken
16:46:59  stdout  [LogMetricsFromAuthResult] DurationTotalInMs: 309
16:46:59  stdout  TokenEndpoint: ****
```

`…_managed_identity_AppTokenCache` keyed on `af65ac56…` is `DefaultAzureCredential` resolving
to the **managed identity** branch, using the instance identity. Not a service principal, not
a secret, not your user account. This is the thing to grant roles to.

> [!TIP]
> `Cache Refresh Reason: NoCachedAccessToken` on the first call, ~309 ms — that is the cold-start
> token fetch. It is cached afterwards, which is part of why a first invoke is slower than the
> rest. See [cost](./cost.md) and the timing in [`troubleshooting`](./troubleshooting.md).

---

## What `azd provision` does **not** do

This surprised me enough to double-check it. Query role assignments scoped to the resource
group, excluding inherited ones:

```bash
az role assignment list --scope /subscriptions/<sub>/resourceGroups/rg-<env> -o table
```

```text
(empty)
```

**Zero.** The default `azd ai agent` flow creates **no role assignments at the resource group
scope at all**. Everything works because the deploying user already has broad inherited
rights — in the captured run, `Owner` at subscription scope plus a tenant-level
`Foundry User`.

The implication is uncomfortable and worth stating plainly:

> [!WARNING]
> **The golden path succeeds partly because you are over-privileged.** If it works for you as
> a subscription Owner, that is *not* evidence it will work for a least-privilege service
> principal in CI. The permissions were inherited, not granted. Test your pipeline identity
> separately — see [CI/CD](../guide-cicd/README.md).

To see what you are actually relying on, re-run with inheritance shown:

```bash
az role assignment list --scope <rg-scope> --include-inherited -o table
```

---

## Roles that matter

| Role | Granted to | Needed for |
|---|---|---|
| **Azure AI User** / **Foundry User** | you, or your CI principal | Create projects, deploy agents, invoke |
| **Cognitive Services User** | you | Call model endpoints |
| **AcrPull** | the agent instance identity | Pull the image in **container mode** ([deploy modes](./deploy-modes.md)) |
| **Contributor** on the RG | your CI principal | `azd provision` creating resources |

`AcrPull` only enters the picture in container mode. In the code-mode run captured here no ACR
existed at all (`AZURE_CONTAINER_REGISTRY_ENDPOINT=""`), so there was nothing to pull and
nothing to grant.

---

## Diagnosing identity problems

```bash
azd ai agent doctor --unredacted
```

`doctor` **redacts principal IDs, scope ARNs and UPNs by default**; `--unredacted` shows them.
Use it when you need to confirm *which* principal is being denied — and don't paste the result
into a bug report unredacted. Use `--local-only` to skip network checks entirely.

| Symptom | Almost always |
|---|---|
| Works locally, 403 in Foundry | Code runs as **identity 3**, not you. Grant the instance identity the role. |
| Granted a role, still 403 | Granted it to the **blueprint** principal instead of the **instance** principal. |
| CI fails where laptop succeeds | You have inherited rights the CI principal does not. |
| `ImagePullBackOff`-style failure | Instance identity missing **AcrPull** in container mode. |

---

## See also

- 👉 [`deploy-modes.md`](./deploy-modes.md) — when ACR and `AcrPull` become relevant
- 👉 [`cost.md`](./cost.md) — what the provisioned resources bill
- 👉 [`observability.md`](./observability.md) — reading the logs quoted above
- 👉 [`troubleshooting.md`](./troubleshooting.md) — the full symptom index
