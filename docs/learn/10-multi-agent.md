# 🧩 Multi-agent patterns — when and why to split

> A multi-agent system is not automatically better. Split only when the boundary gives you a
> clearer prompt, a safer permission model, a reusable specialist or a measurable quality win.

---

## When splitting into multiple agents is worth it

| Split when… | Why it helps | Example |
|---|---|---|
| A step needs a **different instruction set** | Smaller prompts reduce ambiguity and make eval failures easier to localise. | Writer → legal reviewer → formatter. |
| A step needs **different permissions** | Each deployed agent can have its own managed identity and role assignments. | Planner agent has read-only project access; executor agent can write tickets. |
| A capability is **reusable** | A specialist can serve several orchestrators through A2A or a toolbox-like boundary. | Translation, policy review, search, summarisation. |
| You need **independent versioning** | One specialist can change without redeploying the whole system. | Upgrade the legal reviewer from `:3` to `:4` only. |
| You need **parallel work** | Fan-out can reduce wall-clock time if tasks are independent. | Ask three retrieval agents to search separate corpora. |
| You need **auditability** | A trace can show which specialist made which decision. | Compliance approval chains. |

---

## When NOT to go multi-agent

Do **not** split only because the diagram looks more sophisticated. Every extra agent call adds
latency, model tokens, possible cold starts, auth checks and another failure mode.

| Situation | Stay single-agent because… |
|---|---|
| The "specialist" prompt is ≤ 3 sentences different from the orchestrator's. | You gain nothing that a section header in one prompt cannot do. |
| You have no independent eval for the specialist. | You cannot tell whether splitting helped or hurt. |
| Latency is critical and the steps are tightly coupled. | Sequential calls are additive; an internal function is near-zero cost. |
| Only one team owns all the agents. | Independent versioning has no consumer yet. |
| The "boundary" is just string formatting or deterministic logic. | Use a tool or a function — no model call needed. |
| You want A2A but reuse is hypothetical. | A2A is the least settled area of the preview; avoid it until the consumer exists. |

---

## Patterns at a glance

| # | Pattern | Shape | Status |
|---|---|---|---|
| 1 | **Sequential chain** | A → B → C | ✅ Verified samples exist (Python, C#) |
| 2 | **Parallel / fan-out** | Orchestrator fans out, then joins | Illustrative — no sample in verified catalog |
| 3 | **Router / hand-off** | One branch selected per request | Illustrative |
| 4 | **Workflow agent** | Many agents behind one hosted endpoint | ✅ Both verified samples use this |
| 5 | **A2A (agent-to-agent)** | Cross-endpoint delegation via agent card | ✅ Deployed live; delegation **not** yet successful — see [Lab 09](../tutorial/09-multi-agent-a2a.md) |

For detailed diagrams, verified sample breakdowns, agent-card JSON and CLI surface, see the
[Multi-agent reference](../reference/multi-agent.md).

---

## Three boundary levels — pick the lowest that fits

| Level | What happens | Best for |
|---|---|---|
| **Internal workflow node** | One process, several Agent Framework agents chained in code. | Tight pipeline, same owner, same permissions. |
| **Tool / MCP / toolbox** | Orchestrator calls a tool endpoint. | Deterministic capabilities or external APIs — no separate reasoning loop needed. |
| **Remote agent (A2A)** | Orchestrator calls another deployed agent's endpoint. | Independent teams, independent permissions, reusable specialists. |

The rule of thumb: if you do not need a separate identity, a separate version, or a separate
team's ownership, an internal workflow node is simpler and faster.

---

## A2A, MCP toolboxes and plain functions — what is what

| Mechanism | Model call in target? | Separate identity? | Discovery protocol? |
|---|---|---|---|
| Function / tool call | No | No | N/A — hardcoded in code. |
| MCP server / Foundry Toolbox | Depends on the tool implementation | Connection-level auth | MCP `tools/list`. |
| A2A remote agent | **Yes** — the target is a full agent | Yes — separate hosted agent | Agent card at a well-known URL. |

> [!IMPORTANT]
> A2A is a trust boundary, not just a function call. Treat the remote agent like an external
> service: authenticate it, authorise the caller, version it, monitor it and decide what user
> context may cross the boundary.

---

## Design checklist before you split

| Question | If the answer is no… |
|---|---|
| Does the specialist need a materially different prompt? | Keep it in one agent. |
| Does it need different permissions or identity? | An internal workflow may be enough. |
| Will another product or team reuse it? | Avoid A2A until reuse is real. |
| Can you evaluate it independently? | Add evals before adding routing complexity. |
| Is the added latency acceptable? | Prefer a local tool or single-agent prompt. |
| Do you know what context crosses the boundary? | Define a minimal hand-off schema first. |

---

## Cost and latency — the honest trade-off

| Cost driver | Why it grows | Mitigation |
|---|---|---|
| Model calls | One per agent per step. | Combine steps only when separation adds no value. |
| Tokens | Each agent carries its own instructions. | Keep specialist prompts short. |
| Cold starts | Separate hosted agents can each need a session. | Avoid over-splitting tiny tasks. |
| Network hops | A2A and remote tools add endpoint latency. | Prefer internal workflows for tightly coupled pipelines. |
| Evaluation | More branches = more test cases. | Evaluate each specialist plus the orchestrator path. |

Sequential latency is roughly additive. Parallel latency is bounded by the slowest branch
plus synthesis — but cost still grows with every branch. The numbers are illustrative; measure
your actual agents with traces and eval runs.

---

## Further reading

| Resource | What it covers |
|---|---|
| [Multi-agent reference](../reference/multi-agent.md) | Verified sample details, agent-card JSON, CLI surface, identity/trust, versioning, eval strategy. |
| [Lab 09 — A2A tutorial](../tutorial/09-multi-agent-a2a.md) | Hands-on deployment of two agents with a `RemoteA2A` connection. |

---

## ✅ Check your understanding

Three questions. If you can answer all three, move on.

<details>
<summary><b>1.</b> Your single agent has grown to fourteen tools and its answers have become unreliable. Is splitting it into three agents guaranteed to help, and what do you pay for the attempt?</summary>

Not guaranteed. Splitting helps when the tools serve genuinely different jobs; it does not help if the real problem is a vague instruction or a weak model. You pay in model calls (one per agent per step), tokens (each agent carries its own instructions), latency (network hops), and evaluation surface (every branch needs its own cases). Try tightening the instruction first — it is free.
</details>

<details>
<summary><b>2.</b> You want agent A to call agent B. When should that be A2A, and when should it be a plain tool call instead?</summary>

Use a tool (MCP or a function) when B is a capability — a lookup, a calculation, a write. Use A2A when B is genuinely another **agent**: it reasons, owns its own instructions and model, and could be operated by a different team. A2A buys you an independent deployment and identity boundary; if you do not need that boundary, a tool call is cheaper and easier to debug.
</details>

<details>
<summary><b>3.</b> What would happen if you deployed a two-agent A2A pair today, following this guide exactly?</summary>

You would get as far as two deployed agents and a `RemoteA2A` connection — and the delegation call itself would not work. This is the one path in this repo that does **not** finish green: `azd provision` drops the manifest's `audience` on a `RemoteA2A` connection, and four workarounds were attempted without success. See [Lab 09](../tutorial/09-multi-agent-a2a.md) for the full transcript and [the reference page](../reference/multi-agent.md) for what *is* verified.
</details>

---

## → Next

You have finished the 読む layer. Everything here was understanding; nothing asked you to
type a command. Now go and do it.

| Go to | If you want to |
|---|---|
| [🧪 Tutorial index](../tutorial/README.md) | Start the hands-on labs from the beginning |
| [Lab 09 — A2A](../tutorial/09-multi-agent-a2a.md) | Jump straight to the multi-agent lab |
| [Multi-agent reference](../reference/multi-agent.md) | Look up sample details, agent-card JSON, identity and eval strategy |

---

<sub>[← Living with preview](09-living-with-preview.md) · [📘 Learn index](README.md) · [🧪 Tutorial index →](../tutorial/README.md)</sub>
