# Agent Bridge

Agent Bridge is a lightweight U++ integration layer for inspecting and controlling running applications/runtimes while leaving each application authoritative for its own state, commands, validation, undo/redo and persistence.

The transport-independent semantic vocabulary is:

```text
Application / Context / Capability / Query / Command / Job / Event / Revision / Resource
```

## Place in the wider AgentFlow direction

Agent Bridge is a sibling infrastructure project under the broader AgentFlow integration programme, but it remains reusable outside AgentFlow.

Three paths must stay distinct:

```text
A. AgentFlow internal node communication
   AgentFlowCore / Runtime only

B. AgentFlow -> external application/runtime
   Runtime -> capability/side-effect broker -> Agent Bridge -> application

C. external chatbot -> application
   chatbot -> MCP -> AgentBridgeMcp -> Agent Bridge -> application
```

Agent Bridge must not become AgentFlow's scheduler, retry engine, budget system, permission authority or internal packet model. When AgentFlow calls a Bridge capability, AgentFlow remains authoritative for attempt/generation identity, grants, retries, budgets, cancellation, stale completion and evidence.

`AgentFlowProviders` is a separate concern: it supplies model intelligence (OpenAI/DeepSeek/Anthropic/local/etc.) for Runtime attempts. A node may use a model through AgentFlowProviders while also using an application capability such as Dramatica through Agent Bridge.

When AgentFlow is the primary intelligence of an application, prefer embedding AgentFlow directly and use Agent Bridge for capabilities living outside that process. MCP remains useful when the intelligence/consumer is an external chatbot or agent host.

## Current transport direction

```text
Agent Bridge semantic API
          |
      transport binding
       /          \
local D-Bus     direct/remote peer transport
```

The preferred first communications foundation is the small native U++ `DBus` package. It already provides typed binary values/messages, method calls/replies, errors, signals, routing and asynchronous `SocketWaitEvent` integration without libdbus/glib.

For normal local Unix/Linux use, ordinary D-Bus routing is attractive. For Windows, macOS/direct peer and remote-machine use, keep the same Agent Bridge semantics and reuse the D-Bus message/control layer where practical through a lightweight peer/TCP adapter.

D-Bus is below Agent Bridge semantics. Application registrations must not expose D-Bus-specific types as the product API.

Remote use still requires explicit authentication, encryption/TLS, endpoint identity and size/rate/time limits. Raw TCP is not a security model.

Large images/audio/files remain Agent Bridge Resources rather than giant ordinary control messages.

## Core principles

- **Application authority stays in the application.** Agent Bridge never owns a duplicate document, screenplay, graph, task or asset model.
- **Declare capabilities once.** One semantic declaration drives runtime registration, discovery, MCP projection, documentation and tests.
- **Transport is below semantics.** D-Bus/TCP can evolve without rewriting application capability definitions.
- **Context is first-class.** Describe what the user/process is working on without dumping entire application state.
- **Async work is explicit.** Long operations become Jobs with stable identity; progress/completion is observable and state remains queryable.
- **Push is responsiveness; state is correctness.** Reconnect can query current context/job/revision again.
- **Resources stay efficient.** Large binary assets use resource handles/streaming.
- **No permanent Agent Bridge broker by default.** Use D-Bus routing locally or direct connections where appropriate; add a central broker only if a real need emerges.
- **Small U++ implementation.** Prefer a few readable files/classes over framework-sized package hierarchies.

## Initial proof

The first implementation proof should remain deliberately small:

1. application registration/info;
2. capability manifest;
3. current Context;
4. one Query;
5. one mutating Command with revision-aware rejection where relevant;
6. one application-originated Event.

Do not pull MCP, full job/resource machinery, remote TLS or broad application integrations into that first proof.

Strong eventual proof applications include TaskTrack, Dramatica/Story development, Designer and AgentFlow itself.

See `docs/ARCHITECTURE_PLAN.md` and `docs/ACTIVE_WORK.md` for detailed current work.
