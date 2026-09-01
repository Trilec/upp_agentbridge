# Agent Bridge

Agent Bridge is a lightweight U++ integration layer that lets AI/agent hosts inspect and control a running application without embedding a model runtime inside that application.

The application remains authoritative for its own state, commands, undo/redo, validation and persistence. Agent Bridge standardises the small semantic surface needed by external consumers: application instances, context, capabilities, queries, commands, jobs, events, revisions and resources. A generic MCP adapter projects that surface to MCP-capable chat/agent hosts.

## Current design direction

```text
Chat / agent host
       |
       | MCP
       v
AgentBridgeMcp
       |
       | Agent Bridge semantics
       | over preferred U++ D-Bus transport
       v
Running application
  + AgentBridge package
       |
       v
application services / commands / models
```

The preferred communications foundation is the small native U++ `DBus` package. It already provides typed binary messages, method calls/replies, signals, routing and asynchronous `SocketWaitEvent` integration without `libdbus`/glib. Agent Bridge should reuse that work rather than build a parallel binary framing and value-codec stack unless evidence shows a real gap.

For local desktop use, normal D-Bus is the natural path. For Windows, direct peer use and remote machines, the intended direction is a lightweight TCP adapter around the same D-Bus message layer. The DBus author has indicated such an adapter is a reasonable `DBusTools` addition; if it is not supplied, the source is available and the adapter is small enough for us to implement while preserving the same Agent Bridge semantics.

Remote Internet use must still add explicit authentication and encryption; raw TCP exposure is not the security model.

## Core principles

- **Application authority stays in the application.** Agent Bridge never owns a duplicate document, screenplay, graph, task or asset model.
- **Declare capabilities once.** One application-side declaration drives runtime registration, discovery, MCP schemas/descriptions, documentation and tests.
- **Existing command systems remain valid.** Applications bind Agent Bridge commands to their own command/service layer; Agent Bridge does not require a replacement command framework.
- **Transport is below semantics.** D-Bus is the preferred transport foundation, not the definition of Agent Bridge.
- **Context is first-class.** The bridge describes what the user is working on without dumping the entire application state.
- **Async work is explicit.** Long-running work becomes a job with an ID; progress/completion is observable through state and events.
- **Binary resources stay efficient.** Large images/audio/files are resource streams/handles rather than base64 payloads inside ordinary calls.
- **Local first, remote-capable.** Local D-Bus/direct IPC comes first; TCP/TLS extends the same model to remote machines.
- **Human-readable architecture.** Prefer a few clear files/classes over deep package trees, factories and abstraction layers.

## Planned repository shape

```text
AgentBridge/
    AgentBridge.h
    AgentBridge.cpp
    AgentBridge.upp

AgentBridgeMcp/
    AgentBridgeMcp.h
    AgentBridgeMcp.cpp
    main.cpp
    AgentBridgeMcp.upp

tests/
    AgentBridgeTest/

docs/
    ARCHITECTURE_PLAN.md
    ACTIVE_WORK.md
```

This is a target, not a reason to create empty scaffolding. Split source only when an implemented responsibility genuinely needs it.

See [docs/ARCHITECTURE_PLAN.md](docs/ARCHITECTURE_PLAN.md) for the current architecture and staged implementation plan.
