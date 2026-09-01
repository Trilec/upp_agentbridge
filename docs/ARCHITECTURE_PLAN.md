# Agent Bridge Architecture and Implementation Plan

## 0. Current architecture checkpoint

This document is the current architecture direction for `Trilec/upp_agentbridge`.

Remote GitHub remains authoritative. The current transport direction was reviewed against:

- `upp_agentbridge/main` — `be3f90e0e8aa21dddc35ec38372e55bc209bd177` before this revision;
- `Trilec/DBus/main` — `818c6f3eed9c1976d3618d5ea4d7f48363e8fd26`;
- the existing TaskTrack, UiDesigner, Dramatica, UiSymbolPicker and AgentFlow integration shapes already surveyed in the initial architecture pass.

The important revision from the first plan is this:

> **Agent Bridge should not build a custom binary IPC stack first. The native U++ DBus package is now the preferred communications foundation. Agent Bridge owns the application/agent semantics above it.**

The planned TCP adapter for the DBus package is the preferred route for direct/Windows/remote communication. If that adapter is not delivered upstream, we can implement the small adapter ourselves from the available source without changing the Agent Bridge semantic API.

## 1. Objective

Agent Bridge makes a normal application inspectable and controllable by external AI/agent hosts while keeping the application independent of model provider, chatbot UI and MCP internals.

The application should not need to know whether the external consumer is ChatGPT, Codex, Claude, OpenCode, AgentFlow, a test harness, a remote production service or a human-authored automation client.

The reusable contract must support:

- discovery of one or more running application instances;
- application-specific queries and commands;
- current user/application context;
- revision-aware mutation;
- asynchronous jobs and completion/progress;
- application-to-consumer events;
- large/binary resources such as images, audio and generated files;
- multiple concurrent consumers;
- fast local communication;
- a clean path to authenticated remote communication;
- one generic MCP adapter rather than one MCP implementation per application.

The implementation must stay small enough that a human developer can understand the complete bridge without navigating a framework-sized source tree.

## 2. Non-negotiable design rules

### 2.1 Application authority stays in the application

Agent Bridge is not a second application model.

The application remains authoritative for:

- documents and domain state;
- validation;
- command acceptance;
- undo/redo;
- persistence;
- permissions/policy owned by the application;
- durable task/job state where the application already owns it.

Agent Bridge holds only integration state such as connection identity, capability metadata, bounded event state and request/job routing information.

### 2.2 Declare capabilities once

An application-facing capability is described once and projected outward.

Do not maintain independent drifting copies of:

- the C++ binding;
- an Agent Bridge command table;
- an MCP tool table;
- a separate schema file;
- separate machine documentation.

Optional guides/skills may explain workflow, but they are not protocol truth.

### 2.3 Do not force one command framework

Agent Bridge standardises remote command semantics, not application implementation inheritance.

A mature application such as UiDesigner binds to its existing command service. New applications may later use a tiny shared history helper if repeated implementations prove it worthwhile, but Agent Bridge does not require that architecture.

Undo/redo are normal application capabilities. Agent Bridge does not own history.

### 2.4 Keep the production shape small

Do not start with separate `Core`, `Client`, `Comms`, `Host`, `Registry`, `Protocol`, `Transport`, `McpAdapter` and similar packages.

Initial production shape remains:

1. `AgentBridge` — reusable application-facing semantic layer and transport binding;
2. `AgentBridgeMcp` — generic MCP projection.

The external U++ `DBus` package supplies the preferred IPC machinery. If a tiny Agent Bridge-specific adapter is needed, keep it in the same package until a genuine reusable boundary appears.

### 2.5 Transport is below semantics

The semantic contract is:

```text
Application instance
Context
Capability
Query
Command
Job
Event
Revision
Resource
```

D-Bus is the preferred means of carrying that contract. It must not leak so deeply into application registrations that changing or extending transport later rewrites application code.

### 2.6 Push is responsiveness; state is correctness

Events accelerate awareness but are not the sole authority.

A reconnecting consumer can always query current context/state/revision again. Applications that require durable history, such as TaskTrack, continue to own that durability themselves.

## 3. Runtime topology

### 3.1 Preferred local topology

```text
+----------------------+       MCP        +----------------------+
| Chat / agent host    | <-------------> | AgentBridgeMcp       |
+----------------------+                  +----------+-----------+
                                                   |
                                         Agent Bridge semantics
                                                   |
                                               U++ DBus
                                                   |
             +-------------------+-----------------+------------------+
             |                   |                                    |
             v                   v                                    v
      Designer instance A   TaskTrack instance B               SymbolPicker C
       + AgentBridge         + AgentBridge                      + AgentBridge
```

On systems with an ordinary session bus, application instances can expose their Agent Bridge service through D-Bus service/object/interface identities and use D-Bus routing/discovery rather than inventing a parallel local registry where that is unnecessary.

### 3.2 Direct/TCP topology

The desired extension is:

```text
AgentBridgeMcp / AgentFlow / other consumer
                  |
             TCP adapter
                  |
        D-Bus message/control layer
                  |
          remote/direct application
```

This covers environments where a normal desktop bus is unavailable or inappropriate, including Windows/direct peer communication and remote machines.

The DBus author has indicated that a lightweight TCP bridge is a reasonable `DBusTools` component. We should use it if it remains small and clean. If it does not arrive, we have the source and can implement the adapter ourselves rather than reverting automatically to a second independent protocol stack.

### 3.3 No permanent Agent Bridge broker by default

Do not add a separate Agent Bridge broker daemon merely because the first architecture drawing had one available as an option.

D-Bus already provides brokered local routing where appropriate. Direct TCP can connect endpoint-to-endpoint where appropriate.

A separate Agent Bridge broker should only appear later if cross-machine discovery, central policy or large fan-out proves it necessary.

## 4. Why D-Bus is a good fit

The current U++ `DBus` package is deliberately small and native. Its core package consists of a small set of message/parser/connection source files and depends only on U++ Core.

It already provides the transport-level concepts Agent Bridge needs:

- typed binary values and nested arrays/maps/structs;
- method call/reply correlation;
- structured errors;
- signals/events;
- well-known service names and unique connection identities;
- asynchronous socket integration through `SocketWaitEvent`;
- explicit server dispatch hooks;
- local D-Bus authentication;
- direct access to the underlying U++ socket when adapters need it.

These map naturally to Agent Bridge:

```text
Agent Bridge       D-Bus transport concept
-------------------------------------------
Query              method call
Command            method call
Result             method reply
Error              error reply
Event              signal
Context            method/property result
Instance/service   service + object/interface identity
```

Reusing this avoids building and maintaining our own frame header, request serialisation, nested value codec, routing and event dispatch simply to recreate an existing compact IPC protocol.

## 5. What D-Bus does not replace

D-Bus does not define our application semantics.

For example, a D-Bus method may carry this capability:

```text
id: writer.scene.dialogue.replace
kind: command
undoable: true
revision_scope: scene
input: ...
output: ...
```

The meaning of `kind`, `undoable`, revision semantics, context, jobs, resources and application capabilities remains Agent Bridge's contract.

This is why application code should bind to Agent Bridge, not directly expose arbitrary D-Bus methods as the product API.

## 6. Public application API

The public API should fit primarily in `AgentBridge.h` and read like normal application code.

Conceptual example:

```cpp
AgentBridge bridge;

bridge.Application("com.trilec.uidesigner", "UiDesigner", "1.0");

bridge.Context([&] {
    return MakeCurrentDesignerContext();
});

bridge.Query("designer.selection.get", "Return the current Designer selection")
      .Output(SelectionSchema())
      .Bind([&](const ValueMap&) { return GetSelection(); });

bridge.Command("designer.node.move", "Move a node")
      .Input(MoveNodeSchema())
      .Mutates()
      .Undoable()
      .Bind([&](const ValueMap& args) { return MoveNode(args); });

bridge.Job("designer.export", "Export the current design")
      .Input(ExportSchema())
      .Bind([&](const ValueMap& args) { return StartExport(args); });

bridge.Event("designer.selection.changed", "Selection changed");

bridge.Start();
```

Exact builder syntax is an implementation detail. The important properties are:

- short and readable;
- no subclass hierarchy required;
- lambda/function binding works;
- schemas and descriptions live beside the binding;
- registration may be split across normal application source files;
- no D-Bus-specific boilerplate is required for every application capability.

## 7. Core semantic primitives

### 7.1 Application instance

A running process exposes:

- stable application type ID;
- display name;
- application version;
- unique runtime `instance_id`;
- optional project/document summary;
- Agent Bridge semantic protocol version;
- capability manifest hash.

Application type and runtime instance identity must never be conflated.

### 7.2 Context

Context answers: **what is the user doing now?**

It is intentionally different from the full application state.

Typical optional fields:

- active project/workspace;
- active document/resource;
- active semantic scope;
- selection;
- mode/tool;
- relevant revision IDs;
- small application-specific typed data.

Prefer IDs/references over enormous inline state.

### 7.3 Query

Read-only application operation.

Examples include current selection, scene information, control schema, character context, candidate storyform count and symbol search.

### 7.4 Command

Authoritative mutation request.

A command may include `expected_revision`. The application remains the final acceptance authority.

Keep behavioural metadata limited to useful facts such as `mutates`, `undoable` and `destructive`/confirmation-sensitive.

### 7.5 Job

Work that may outlive the request.

A job has an ID, state, optional progress, optional result and optional resource IDs. The initial call returns promptly; completion is observable through current state and events.

Agent Bridge does not automatically persist jobs. Existing application durability maps through rather than being duplicated.

### 7.6 Event

Application-to-consumer notification.

Events may carry sequence, event ID, revision, job ID and payload. Examples include context changes, document changes, selection changes, job progress/completion, TaskTrack awaiting-agent transitions and application shutdown.

A small bounded replay window is sufficient unless a specific application already owns durable event history.

### 7.7 Resource

Addressable data that should not be forced through ordinary command payloads.

Resource descriptors contain ID, content type, size when known and optional name/hash/metadata.

Examples include screenshots, preview images, WAV output, generated headers/IML, export packages and structured scene-context documents.

## 8. Capability manifest

The capability manifest is the machine-readable heart of Agent Bridge.

Every registered query/command/job/event/resource description is represented in one canonical manifest.

Minimal descriptor:

```text
id
kind
name/title
description
input schema
output schema
small behavioural flags
optional category/tags
```

Do not put C++ implementation addresses, C++ type names, D-Bus-specific routing fields or MCP-specific fields into the canonical semantic manifest.

The manifest has a deterministic hash so consumers can cache unchanged definitions.

## 9. D-Bus projection

Agent Bridge needs only a small stable D-Bus surface. Do not create one hand-coded D-Bus interface implementation per application command.

A likely shape is a stable Agent Bridge service/interface exposing operations conceptually equivalent to:

```text
GetInfo
GetManifest
GetContext
SearchCapabilities
DescribeCapability
Call
GetJob
CancelJob
GetEvents
ReadResource
```

Application capabilities remain identified by Agent Bridge capability ID and are dispatched through the registered bindings.

Important application events can be emitted as Agent Bridge D-Bus signals carrying compact typed metadata.

The exact interface/object naming is an AB-002 implementation decision and should stay boring and deterministic.

## 10. Bulk resource transfer

D-Bus is the control plane; large media must remain efficient.

Do not put a 400 MB image sequence or large audio asset into an ordinary capability response simply because D-Bus can represent a byte array.

Preferred logical model:

```text
Call/query result
    -> resource descriptor / resource ID

ReadResource
    -> bounded chunks or transport-specific stream
```

For local use we may later add shared memory/mmap if measurement proves worthwhile. For TCP/remote use the adapter can stream chunks over the secured connection.

The logical `Resource` API must not change when the physical fast path changes.

## 11. Async behaviour

Agent Bridge distinguishes request lifetime from work lifetime.

```text
Call Export
    -> accepted, job J81

later
    -> job.progress
    -> job.completed
    -> resource R12 available
```

D-Bus method calls/replies handle the immediate control operation. D-Bus signals are a natural local transport for events. Job state remains queryable so correctness never depends on receiving a signal.

Agent Bridge can know an application job completed, but an arbitrary chatbot host may still control whether that completion creates a new model turn. MCP Tasks/subscriptions or host-specific wake mechanisms are adapter concerns.

TaskTrack's existing compatibility wait flow remains valid until an end-to-end host path proves a cleaner immediate-resume mechanism.

## 12. Discovery and multiple instances

On a normal local D-Bus session bus, use D-Bus identities and service discovery where possible instead of inventing a second local registry.

Agent Bridge still needs an application-level `instance_id` because:

- several instances of the same application may run;
- D-Bus connection/service identity is transport identity, not application semantic identity;
- TCP/direct connections also need the same stable vocabulary.

If direct/TCP mode requires a tiny locator/registry, make that adapter-specific. Do not make a per-user registry part of Agent Bridge semantics simply because the original TCP-only plan needed one.

Remote discovery should initially be explicit endpoint configuration. Internet-wide discovery is a separate product concern.

## 13. Threading and UI safety

Incoming transport callbacks must not mutate U++ GUI/application state unsafely.

The current DBus package already supports asynchronous `SocketWaitEvent` integration and protects against reentrant synchronous calls during dispatch.

Agent Bridge application handlers should still execute through the application's appropriate dispatcher/main thread when they mutate GUI/domain state.

Do not build a general actor runtime or thread pool into Agent Bridge.

## 14. Revisions and concurrency

Command calls can carry:

```text
expected_revision
consumer_id
```

Revision meaning remains application-defined. Applications without meaningful revisions can omit it.

Stale mutations must be rejected clearly where revisions are used.

Multiple consumers may connect simultaneously. Agent Bridge does not attempt collaborative document merging; application command/revision authority decides whether a mutation is safe.

## 15. Security and remote use

### 15.1 Local

Use the security properties of the local D-Bus/session environment where appropriate and do not expose a network listener merely for convenience.

Direct local TCP mode must bind narrowly and authenticate the peer appropriately.

### 15.2 Remote

A TCP bridge is a transport adapter, not an Internet security protocol.

Remote mode must not be enabled without:

- encryption, normally TLS;
- explicit authentication;
- endpoint/application identity verification;
- application-appropriate capability/access policy;
- size/rate/time limits.

Do not ship a plaintext `remote=true` shortcut.

The D-Bus message/control layer can remain the same inside the secured remote transport.

## 16. MCP adapter

MCP is a projection, not the Agent Bridge core protocol.

`AgentBridgeMcp` translates between MCP and Agent Bridge semantics. Applications do not expose MCP-specific JSON-RPC internally.

The generic adapter should use a small progressive-discovery surface approximately like:

```text
agentbridge_list_apps
agentbridge_get_context
agentbridge_search_capabilities
agentbridge_describe_capability
agentbridge_call
agentbridge_get_job
agentbridge_get_events
agentbridge_read_resource
```

Selected important capabilities may later be projected as first-class MCP tools if real host testing shows value. That is an optimisation, not the basis of interoperability.

Skills/guides remain optional workflow advice. The capability manifest remains authoritative.

## 17. Application proofs

### 17.1 TaskTrack — first strong transport/async proof

TaskTrack is a particularly useful early integration because it already has:

- a GUI/MCP-independent semantic core;
- durable task and assistance state;
- explicit human/agent phases;
- a separate MCP executable;
- asynchronous waiting/continuation behaviour;
- a host-facing skill.

It can prove that D-Bus/Agent Bridge handles application-originated events and long-lived human interaction without making D-Bus or Agent Bridge the durable task authority.

Map rather than rewrite:

- TaskTrack task ID -> application job/resource identity where useful;
- assistance request -> event/current-state query;
- terminal completion -> job/event result;
- existing skill -> application usage guide.

### 17.2 UiDesigner — live command/revision proof

UiDesigner already contains:

- `UiDesignerCommandService` with atomic mutations and undo/redo history;
- `UiDesignerSession` with document/selection state;
- revision-aware edit intent;
- a substantial machine-facing automation surface.

Agent Bridge should bind the live running session to that existing authority rather than create another command system.

Acceptance should include live selection/context, a query, a mutation, revision change, undo and stale-revision rejection.

### 17.3 Dramatica — query-heavy proof

`ThroughlineCore` already separates domain logic from UI. This proves Agent Bridge can be useful with mostly queries/resources/context and without requiring an undo/redo command architecture.

### 17.4 UiSymbolPicker — scale/resource proof

SymbolPicker's large model-driven catalogue and deterministic asset exports are a good proof for search, progressive discovery, binary resources and generated artifacts.

Never transmit the entire catalogue merely because an agent connected.

### 17.5 AgentFlow — consumer/provider, not a merge

AgentFlow may later consume Agent Bridge as an external application/tool capability source, and its Workbench may also expose itself through Agent Bridge.

Neither role replaces AgentFlow's runtime, workflow authority, evidence/context or side-effect policy.

### 17.6 Future image review and production tracking

These strengthen the need for:

- remote/direct connections;
- large image/audio/file resources;
- comments/annotations;
- context and selection;
- revisions;
- jobs/events;
- secure cross-machine operation.

They do not require media-specific concepts in Agent Bridge core.

## 18. Implementation stages

### AB-001 — architecture contract

Current state: architecture documented and transport direction revised toward U++ D-Bus.

Acceptance:

- application state/commands remain authoritative;
- no large package hierarchy;
- no mandatory Agent Bridge broker daemon;
- capability/context/query/command/job/event/resource vocabulary accepted;
- D-Bus accepted as preferred communication foundation;
- Agent Bridge semantics remain transport-independent;
- TCP adapter is the preferred direct/remote extension, upstream or ours.

### AB-002 — D-Bus bridge proof

Do **not** begin by implementing the old custom frame/value codec.

Build only the smallest proof needed to validate the preferred transport direction:

- tiny AgentBridge application registration API;
- one application instance;
- stable Agent Bridge D-Bus service/interface shape;
- `GetInfo`/manifest transfer;
- basic context;
- one query;
- one command;
- one application-originated event/signal;
- clean shutdown/reconnect;
- deterministic tests for descriptor/dispatch behaviour.

Use the current U++ DBus package directly for the local proof.

If the TCP adapter is already available, add one minimal direct connection proof. If it is not available, do not block AB-002 on remote networking; document the small adapter contract first.

### AB-003 — jobs/events/resources

Add:

- asynchronous job lifecycle;
- bounded event replay/current state;
- resource descriptors and bounded reads;
- cancellation where supplied by the application;
- multi-consumer test.

Keep durable application state outside Agent Bridge.

### AB-004 — generic MCP adapter

Implement `AgentBridgeMcp` using the small stable bootstrap surface.

Requirements:

- discover/identify running instances;
- explicit instance IDs;
- capability search/describe/call;
- context;
- job polling / MCP Tasks mapping where supported;
- resources;
- clear errors;
- manifest caching.

Do not copy application-specific commands into MCP source.

### AB-005 — TaskTrack integration

Use TaskTrack to validate the transport and async model end to end:

- create/inspect interaction;
- application-originated events;
- awaiting-human / awaiting-agent transitions;
- terminal completion;
- durable TaskTrack authority remains intact;
- compatibility fallback remains available where the host requires it.

### AB-006 — UiDesigner integration

Expose a live Designer session through Agent Bridge while preserving existing command/session/history authority.

Do not regress standalone CLI/headless automation merely to prove live bridging.

### AB-007 — breadth proof

Integrate Dramatica and/or UiSymbolPicker. If both fit without changing the core semantics, reuse is strongly demonstrated.

### AB-008 — TCP/remote proof

Once the DBus TCP/direct adapter is available upstream or implemented by us:

- prove direct local operation where a session bus is not used;
- prove Windows-oriented/direct endpoint shape as applicable;
- add authenticated TLS transport for remote proof;
- validate the same Agent Bridge semantic calls/events over local D-Bus and remote/direct transport;
- measure resource streaming behaviour.

Remote security is part of acceptance, not a later optional patch.

### AB-009 — Builder/generator experiment

Only after the descriptor/binding API stabilises.

A small Builder may inspect source/API surfaces with AI assistance and generate readable capability registration skeletons, bindings, schemas/descriptions, tests and optional guide drafts.

The Builder is an accelerator, never required runtime infrastructure.

## 19. Source layout target

Keep the source intentionally boring:

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

examples/
    BridgeDemo/

tests/
    AgentBridgeTest/

docs/
    ARCHITECTURE_PLAN.md
    ACTIVE_WORK.md
```

Dependency direction:

```text
application
    -> AgentBridge
        -> DBus

AgentBridgeMcp
    -> AgentBridge
        -> DBus
```

If the TCP adapter lives in `DBusTools`, add that dependency only where needed. If we implement a tiny adapter locally, keep it in one obvious source file until reuse proves it deserves a package.

## 20. What not to build yet

Do not add these without concrete pressure:

- custom binary framing/value codec duplicating D-Bus;
- permanent Agent Bridge broker daemon;
- database;
- distributed event log;
- universal application object model;
- universal command inheritance framework;
- plugin framework;
- dependency injection container;
- runtime reflection system;
- separate package for every conceptual noun;
- shared-memory transport before measurement;
- application-specific MCP servers;
- plaintext remote mode;
- hidden duplicate application state in the adapter.

## 21. V1 success criteria

Agent Bridge V1 is successful when:

- adding it to an existing U++ application is small and readable;
- each capability is described in one place;
- the generic MCP adapter can operate several application types without app-specific MCP code;
- local IPC uses the compact U++ D-Bus foundation cleanly;
- TaskTrack-style async completion works without Agent Bridge owning task durability;
- a live UiDesigner instance can be inspected and mutated through its existing command authority;
- large resources transfer without base64 expansion or giant ordinary control messages;
- several application instances/consumers work safely;
- stale revision writes are rejected where revisions are supported;
- a TCP/direct path can carry the same semantic contract to remote or non-session-bus environments;
- remote mode is authenticated/encrypted;
- bridge overhead is insignificant compared with model/tool latency;
- a new developer can understand the entire bridge without navigating a bulky framework.

## 22. Decisions deliberately left open until implementation evidence

These are bounded implementation choices, not architecture gaps:

- exact D-Bus service/object/interface naming;
- exact mapping of manifest schemas into D-Bus typed values;
- whether the planned TCP adapter arrives in `DBusTools` or is implemented in our fork/Agent Bridge;
- exact direct-endpoint discovery when no session bus is present;
- exact TLS/auth mechanism for remote mode;
- exact resource streaming/chunk framing used by the TCP adapter;
- whether selected app capabilities should later be projected as first-class MCP tools;
- whether repeated integrations justify an optional shared command/history helper.

Resolve each in the smallest milestone that actually needs it and record the reason in `docs/ACTIVE_WORK.md`.
