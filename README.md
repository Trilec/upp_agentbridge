# Agent Bridge

Agent Bridge is a lightweight U++ integration layer that lets AI/agent hosts inspect and control a running application without embedding a model runtime inside that application.

The project is deliberately small. The application remains authoritative for its own state, commands, undo/redo, validation and persistence. Agent Bridge only standardises discovery, context, capability descriptions, requests, asynchronous jobs/events, revisions and binary resources. A generic MCP adapter projects that surface to MCP-capable chat/agent hosts.

## Design direction

```text
Chat / agent host
       |
       | MCP
       v
AgentBridgeMcp
       |
       | Agent Bridge binary protocol
       v
Running application
  + AgentBridge package
       |
       v
application services / commands / models
```

V1 intentionally avoids a permanent broker daemon and a large package hierarchy. A running application exposes one lightweight Agent Bridge endpoint and advertises that instance locally. Any number of MCP adapters can discover and connect to it. Remote endpoints can use the same wire protocol later over an authenticated TLS connection.

## Core principles

- **Application authority stays in the application.** Agent Bridge never owns a duplicate document, screenplay, graph, task or asset model.
- **Declare capabilities once.** One application-side capability declaration should drive runtime registration, discovery, MCP descriptions/schemas, documentation and tests.
- **Existing command systems remain valid.** Applications bind Agent Bridge commands to their own command/service layer. Agent Bridge does not require a replacement command framework.
- **Context is first-class.** The bridge can describe what the user is currently doing without dumping the entire application state.
- **Async work is explicit.** Work that outlives a request becomes a job with an ID; changes/completions are events, not permanently blocked RPC calls.
- **Binary-safe and fast.** The wire protocol is compact binary framing with typed structured values and raw byte/resource support.
- **Local first, remote-capable.** V1 uses loopback TCP because it is simple, fast and cross-platform. The protocol is transport-safe for later TLS/remote use.
- **Human-readable architecture.** Prefer a few clear classes/files over deep folder trees, factories and abstraction layers.

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

This is a target, not a requirement to create empty scaffolding early. New packages/files should be added only when an implemented responsibility needs them.

See [docs/ARCHITECTURE_PLAN.md](docs/ARCHITECTURE_PLAN.md) for the current architecture and staged implementation plan.
