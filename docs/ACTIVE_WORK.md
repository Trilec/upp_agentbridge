# ACTIVE WORK

Remote GitHub `main` is authoritative. Refresh exact HEAD before implementation or publication.

## CURRENT STATE — 2026-09-02

STATUS: **ARCHITECTURE SET; SMALL D-BUS SEMANTIC PROOF IS NEXT. NO PRODUCTION BRIDGE IMPLEMENTATION YET.**

Agent Bridge is reusable infrastructure under the wider AgentFlow integration direction, but it is not AgentFlow's internal node communication or model-provider layer.

Fixed boundary:

```text
AgentFlow internal nodes -> AgentFlowCore / Runtime
AgentFlow external app   -> capability broker -> Agent Bridge
External chatbot         -> MCP -> AgentBridgeMcp -> Agent Bridge
Model provider calls     -> AgentFlowProviders, not Agent Bridge
```

AgentFlow remains authoritative for attempts/generations, retries, budgets, grants, cancellation, stale completion and evidence when it consumes a Bridge capability.

## TRANSPORT DIRECTION

- Agent Bridge owns `Application / Context / Capability / Query / Command / Job / Event / Revision / Resource` semantics.
- Native U++ D-Bus is the preferred first transport/control foundation.
- Do not build the retired custom binary codec/framing stack first.
- D-Bus remains below semantics; application code must not become D-Bus-specific.
- ordinary local D-Bus routing is useful where available;
- direct/Windows/macOS/remote use keeps the same semantics through a suitable peer/TCP transport;
- remote transport requires authentication/encryption/identity/limits;
- large media/files remain resources, not giant ordinary control messages.

Current local reference fork: `Trilec/DBus`. Refresh it against upstream before implementation because upstream may advance rapidly.

## NEXT — AB-002

Build only a tiny proof:

1. application registration/info;
2. manifest;
3. Context;
4. one Query;
5. one mutating Command;
6. one application-originated Event.

Use the U++ DBus package directly for the first local proof. If a clean direct/TCP adapter is available, it may receive one minimal proof, but remote work must not block local semantic validation.

Do not add full MCP, Jobs/Resources, TLS, broad application integrations or AgentFlow coupling inside AB-002.

Potential follow-up proof applications: TaskTrack and Dramatica. AgentFlow integration waits for AgentFlow's capability/side-effect broker boundary.

## DEVELOPMENT RULE

Keep the Bridge small and transport-independent. Existing application command/history/state remains authoritative. Review complete touched files and publish coherent checkpoints.
