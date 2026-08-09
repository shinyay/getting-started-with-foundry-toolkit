# 08 · Versioning

> In Foundry hosted agents, the name stays stable while deployed versions move forward: `my-agent:1`, `my-agent:2`, and so on.

The reference output from a live deploy showed an agent ID shaped like `hello-world:1`, with `Name` as `hello-world` and `Version` as `1`. That shape is the foundation for rollback, comparison and safe iteration.

```mermaid
flowchart LR
    Name["Agent name\nhello-world"] --> V1["hello-world:1"]
    Name --> V2["hello-world:2"]
    Name --> V3["hello-world:3"]
    V1 --> Endpoint["stable agent endpoint / version selection"]
    V2 --> Endpoint
    V3 --> Endpoint
```

## What creates a new agent version

Reusing an agent name creates a new version of that existing agent. The help capture for initialisation says that reusing a name creates a new version instead of a separate agent. The help capture for optimizer deploy says that deploying a winning optimization candidate creates a new agent version.

```mermaid
flowchart TB
    Source["Source or optimised configuration"] --> Deploy["Deploy into existing agent name"]
    Deploy --> New["New version"]
    New --> ID["name:n"]
```

The useful mental model is append-only deployment. You do not edit `my-agent:1` into a different thing. You create `my-agent:2`.

## Immutability is the safety property

Versioning matters because a version should be treated as a fixed deployment artefact. That gives teams a stable object to test, compare, discuss and roll back to.

```mermaid
flowchart LR
    V1["Version 1\nknown behaviour"] --> Compare["Compare"]
    V2["Version 2\nnew behaviour"] --> Compare
    Compare --> Keep{"Decision"}
    Keep -- "V2 wins" --> Promote["Use V2"]
    Keep -- "V1 wins" --> Rollback["Return traffic/session to V1"]
```

The reference layer documents commands that can inspect or download specific versions. This learn page only needs the concept: versions let you name the exact thing you are talking about.

## Rollback is choosing an older known version

Rollback is less mysterious when versions are immutable. You are not rebuilding the past. You are pointing clients or sessions back to a version that already exists and has known behaviour.

```mermaid
sequenceDiagram
    participant Client
    participant Router as Agent endpoint/session selection
    participant V2 as my-agent:2
    participant V1 as my-agent:1
    Client->>Router: use current version
    Router->>V2: request
    Note over V2: regression found
    Client->>Router: use older version
    Router->>V1: request
```

The CLI reference notes that sessions can be created against a specific agent version and that invoke has a version option. The point is not the syntax; the point is that version pinning exists as a first-class concept.

## Version pinning for experiments

Version pinning lets two versions be exercised side by side without pretending one is globally better yet.

```mermaid
flowchart TB
    Test["Same test question"] --> S1["Session pinned to v1"]
    Test --> S2["Session pinned to v2"]
    S1 --> R1["Response and traces v1"]
    S2 --> R2["Response and traces v2"]
    R1 --> Compare["Human or eval comparison"]
    R2 --> Compare
```

This is especially useful for prompt or tool changes, where behaviour can shift without compile errors. A version number gives the evaluation result something concrete to attach to.

## Toolboxes are versioned too

The toolbox endpoint shape makes versioning visible:

```text
/toolboxes/<name>/versions/<n>/mcp
```

The verified toolbox creation output produced an endpoint containing `/toolboxes/agent-tools/versions/1/mcp`. The toolbox help states that a toolbox is a versioned named collection; each version is immutable; mutations create a new version; publishing promotes a version but does not mutate the version itself.

```mermaid
flowchart LR
    TB["toolbox name\nagent-tools"] --> T1["version 1\nMCP endpoint path includes /versions/1/"]
    TB --> T2["version 2\nnew tools or connections"]
    T1 --> Agent["Agent configured with endpoint"]
    T2 --> Publish["Published default"]
```

That endpoint is version-pinned. Publishing a new toolbox version cannot silently change an agent that is configured to call the old version-specific endpoint.

## Agent versions and toolbox versions combine

An agent version can itself depend on a toolbox endpoint that includes a toolbox version. That gives you two axes of change.

```mermaid
flowchart TB
    A1["agent:1"] --> TB1["toolbox:1 endpoint"]
    A2["agent:2"] --> TB1
    A3["agent:3"] --> TB2["toolbox:2 endpoint"]
```

This matters because tool behaviour is agent behaviour. If a tool set changes, an answer can change even when the agent code did not. Version-pinned toolbox endpoints make that dependency visible.

## The practical rule

When discussing behaviour, include both versions when they matter:

| Behaviour source | Version name to preserve |
|---|---|
| Hosted agent code/config | Agent version, e.g. `my-agent:3` |
| Toolbox tools | Toolbox version in the MCP endpoint |
| Model deployment | Model deployment details in the environment/reference layer |
| Eval result | The agent version and tool version used during the run |

Versioning is not bookkeeping. It is how preview-era systems stay debuggable.

## → Next

[Learn how to live with preview](09-living-with-preview.md)

---

<sub>[← Identity model](07-identity-model.md) · [📘 Learn index](README.md) · [Living with preview →](09-living-with-preview.md)</sub>
