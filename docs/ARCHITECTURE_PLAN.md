# Agent Bridge Architecture and Implementation Plan

## 0. Status and inspected baselines

This document is the initial architecture checkpoint for `Trilec/upp_agentbridge`.

Remote GitHub is authoritative. The following repository states were inspected before writing this plan:

- `upp_agentbridge/main` — `f8d45cd87641b14a1b5078d7a8328228007be2bd`
- `upp_tasktrack/main` — `6906d9bce4826bb0e7e0c0787f557077e1a27bda`
- `upp_uidesigner/main` — `2e0b9025327749b7d570c76cb09888b7f4db49d7`
- `upp_dramatica/main` — `d8eb2de7487bffb3c895f9b64e49fb9f54c1edd2`
- `upp_uisymbolpicker/main` — `974081449ef9dc0628d52e313baf2782f5b69e6f`
- `upp_agentflow/main` — `4d1dcfb0e426c80fbd29c73f52e838aa890930a6`

The Agent Bridge repository contained only its licence at the inspected base, so there is no earlier implementation contract to preserve.

## 1. Objective

Agent Bridge should make a normal application controllable and inspectable by external AI/agent hosts while keeping the application independent of any model provider or chatbot UI.

The application should not need to know whether the external consumer is ChatGPT, Codex, Claude, OpenCode, a test harness, a future protocol, or a human-authored automation tool.

The reusable contract must support:

- discovery of one or more running application instances;
- application-specific commands and queries;
- current user/application context;
- revision-aware mutation;
- asynchronous jobs and completion/progress;
- application-to-agent events;
- large/binary resources such as images, audio and generated files;
- multiple concurrent agent consumers;
- efficient local communication;
- a path to authenticated remote communication;
- a generic MCP adapter rather than one MCP server implementation per application.

The implementation must remain small enough that a human developer can understand and add it to an application without adopting a large framework.

## 2. Design rules

### 2.1 Keep authority where it already belongs

Agent Bridge is not a second application model.

The application remains authoritative for:

- documents and domain state;
- validation;
- command acceptance;
- undo/redo;
- persistence;
- permissions/policy that belong to the application;
- durable job/task state where the application already owns it.

Agent Bridge holds only integration state: connection identity, capability metadata, bounded notification state and request/job routing information.

### 2.2 Declare once

An application-facing capability should be described once and then projected outward.

Do not maintain independent copies of:

- the C++ binding;
- an Agent Bridge command table;
- an MCP tool table;
- a separate schema file;
- separate documentation that can silently drift.

The application registration should be sufficient to produce the machine-facing capability description. Optional longer guides/skills may add behavioural advice, but they are not the protocol truth.

### 2.3 Do not force one internal command framework

Agent Bridge standardises remote command semantics, not application implementation inheritance.

A mature application such as UiDesigner should bind to its existing command service. A new application may later choose a reusable Agent Bridge command/history helper if repeated use proves one useful, but that helper is not required for the bridge.

Undo/redo are normal application capabilities. Agent Bridge does not own history.

### 2.4 Keep V1 to two production packages

Do not begin with `Core`, `Client`, `Comms`, `Host`, `Registry`, `Protocol`, `McpAdapter`, `Transport` and other package layers.

Initial production shape:

1. `AgentBridge` — reusable application-side library plus shared wire definitions/codec.
2. `AgentBridgeMcp` — generic MCP executable using the same shared library.

Split responsibilities later only when a real dependency or reuse boundary requires it.

### 2.5 Push is responsiveness; state is correctness

Events may be missed when a consumer disconnects. That must not corrupt application truth.

A reconnecting consumer can always query current context/state/revision again. Events accelerate awareness; they are not the sole durable authority.

Applications that require durable task history, such as TaskTrack, continue to own that durability themselves.

## 3. Minimal runtime topology

### 3.1 Local topology

```text
+----------------------+       MCP        +----------------------+
| Chat / agent host    | <-------------> | AgentBridgeMcp       |
+----------------------+                  +----------+-----------+
                                                   |
                                        Agent Bridge binary wire
                                                   |
             +-------------------+-----------------+------------------+
             |                   |                                    |
             v                   v                                    v
      Designer instance A   Writer instance B                  SymbolPicker C
       + AgentBridge         + AgentBridge                      + AgentBridge
```

There is no permanent broker daemon in V1.

Each application instance:

1. creates an Agent Bridge endpoint;
2. listens on loopback on an OS-assigned TCP port;
3. writes a tiny per-user discovery record;
4. removes/invalidates the record on clean shutdown;
5. verifies consumers during handshake.

`AgentBridgeMcp` scans discovery records, verifies live endpoints and exposes the available instances to the MCP host.

This allows:

- several running applications;
- several instances of the same application;
- several independently launched MCP adapters;
- more than one agent connected to the same application;
- no application dependency on which chatbot happens to be running.

### 3.2 Why no broker daemon initially

A broker would solve discovery/routing, but it adds another process, lifecycle, installer/startup concern, security boundary and failure mode before it is proven necessary.

The lightweight registry + direct endpoint model provides the same basic multi-instance discovery without another subsystem.

A broker may be introduced later only if remote routing, cross-machine discovery, centralized policy or very high fan-out genuinely requires one.

## 4. Public application API

The public API should fit primarily in `AgentBridge.h` and read like application code rather than framework setup.

Conceptual example only:

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

- short;
- obvious;
- no subclass hierarchy required;
- lambda/function binding works;
- schemas and descriptions live beside the binding;
- registration can be split across normal application source files without a central giant switch.

## 5. Core semantic primitives

Keep the primitive vocabulary small.

### 5.1 Application instance

A running process/instance exposes:

- stable application type ID, e.g. `com.trilec.uidesigner`;
- display name;
- application version;
- unique runtime `instance_id`;
- optional project/document summary;
- Agent Bridge protocol version;
- capability manifest hash.

Application type and runtime instance identity must never be conflated.

### 5.2 Context

Context answers: **what is the user doing now?**

It is intentionally different from the full application state.

Common fields should be few and optional:

- active project/workspace;
- active document/resource;
- active semantic scope;
- selection;
- mode/tool;
- relevant revision IDs.

Applications can add their own typed map data.

Examples:

Designer:

```text
project      TimelineUI
active_doc   MainWindow
selection    [node-12, node-18]
mode         design
revision     418
```

Writing application:

```text
project      FeatureFilm
active_doc   screenplay
scope        scene SC-042
selection    dialogue-18
revision     981
```

Context should prefer IDs/references over enormous inline data. The agent can query or read related resources as needed.

### 5.3 Query

Read-only application operation.

Examples:

- current selection;
- scene information;
- control schema;
- character context;
- candidate storyform count;
- symbol search.

### 5.4 Command

Authoritative mutation request.

Standard envelope may include `expected_revision`; the application decides whether that revision is required and remains the final acceptance authority.

Useful descriptor flags are intentionally limited:

- `mutates`;
- `undoable`;
- `destructive`/confirmation-sensitive when materially useful.

Do not encode a large policy language into V1 capability metadata.

### 5.5 Job

Work that may outlive the request.

A job has:

- `job_id`;
- state: queued/running/completed/failed/cancelled;
- optional progress;
- optional result value;
- optional resource IDs;
- optional application-owned durable identity.

The initial request returns promptly with the job ID.

Agent Bridge does not automatically persist jobs. If an application already has durable work identity, the binding maps it. Otherwise V1 job state can be process-lifetime state.

### 5.6 Event

Application-to-consumer notification.

Each event contains:

- monotonically increasing per-instance event sequence;
- event ID;
- optional current revision;
- payload.

Examples:

- context changed;
- document changed;
- selection changed;
- job progress/completed/failed;
- task awaiting agent;
- application closing.

A small bounded ring buffer allows a connected/reconnecting consumer to request recent events by sequence without turning Agent Bridge into a database.

### 5.7 Resource

Addressable data that should not be forced through command payloads.

A resource descriptor contains:

- resource ID;
- MIME/content type;
- size when known;
- optional name/hash/metadata;
- read capability with offset/length.

Resources cover both small semantic documents and large binary outputs.

Examples:

- screenshot;
- preview image;
- WAV dialogue performance;
- generated header/IML;
- export package;
- structured scene context document.

Large data is streamed/chunked; it is not base64-expanded inside ordinary metadata messages.

## 6. Capability manifest

The capability manifest is the machine-readable heart of the system.

Every registered query/command/job/event/resource description is represented in one canonical manifest.

Minimal capability descriptor:

```text
id
kind
name/title
description
input schema
output schema
small set of behavioural flags
optional category/tags
```

Do not put implementation addresses, C++ type names or MCP-specific fields into the canonical manifest.

The manifest has a deterministic hash. During handshake:

```text
application -> manifest_hash
adapter     -> known / request_manifest
```

The adapter caches unchanged manifests. Large application capability lists therefore do not need to be resent on every connection.

## 7. Command integration and undo/redo

### 7.1 Existing command systems

The preferred integration is a thin binding into the application's existing authority.

UiDesigner already has `UiDesignerCommandService` for atomic mutations/history and `UiDesignerSession` for document/selection state. Agent Bridge should bind to those services, not duplicate them.

Conceptually:

```text
AgentBridge command
      |
      v
UiDesigner automation/binding
      |
      v
UiDesignerCommandService
      |
      v
UiDesignerDocument
```

The same rule applies to future applications with their own command architecture.

### 7.2 New applications

Do not implement a universal inherited `AgentBridgeCommand` hierarchy in the first slice.

After integrations across several applications, reassess whether a tiny optional command/history helper would remove meaningful repeated code. Only then extract it.

### 7.3 Undo/redo

Undo/redo are registered capabilities if the application supports them.

The bridge may advertise `undoable=true` for a command, but the application remains responsible for history grouping and semantics.

## 8. Wire protocol

### 8.1 One transport first

Use TCP as the V1 ordered byte stream.

Local default:

- bind only to loopback;
- choose an OS-assigned port;
- advertise through the local registry;
- never open LAN/public interfaces by default.

Why TCP rather than separate named-pipe/Unix-domain-socket/TCP implementations:

- available cross-platform;
- extremely fast on loopback for the expected control traffic;
- same framing works remotely;
- easy to test;
- no platform-specific discovery/IPC implementation in the first slice.

If profiling later proves loopback TCP inadequate, a local transport can be added behind the same frame codec without changing semantics.

### 8.2 Binary framing

Use a fixed, versioned binary frame header followed by payload bytes.

Provisional frame fields:

```text
magic
protocol_major
protocol_minor
message_type
flags
request_or_stream_id
payload_length
```

Exact widths/endian rules are an AB-002 implementation decision and must be documented/tested before use.

Frames must have explicit maximum sizes and reject malformed/oversized lengths before allocation.

### 8.3 Structured binary value codec

Do not depend on raw U++ `Store()`/memory layouts. The wire must remain versioned and implementable outside U++ later.

Also avoid importing a large serialization framework before it is needed.

V1 should implement a tiny binary value vocabulary sufficient for application APIs:

- null;
- bool;
- signed integer;
- floating point;
- UTF-8 string;
- byte string;
- array;
- string-keyed map.

This deliberately mirrors the useful portable subset of `Value` without serializing U++ implementation details.

The exact tag/length encoding should favor clarity over micro-optimisation. It must have deterministic test vectors.

If a standard format such as CBOR becomes clearly cheaper to maintain after a small dependency spike, it may replace the tiny codec before V1 freezes. Do not add such a dependency solely for theoretical elegance.

### 8.4 Bulk binary data

Ordinary command/query payloads stay bounded.

Large resources use chunked resource-read frames or a dedicated stream ID. The logical resource interface must remain the same even if a future local implementation adds memory mapping/shared memory.

### 8.5 Request correlation

Every request has an ID independent of job identity.

A request response can be immediate even when it creates a long-running job:

```text
request 412 -> accepted, job J81
```

Later events/results refer to `J81`; they do not keep request 412 open indefinitely.

## 9. Discovery and handshake

### 9.1 Local registry

Each application writes a small record under a per-user Agent Bridge directory.

Provisional fields:

```text
application_id
instance_id
pid
port
protocol_version
manifest_hash
display/project hint
local authentication nonce/token reference
```

The record is only a locator. The adapter must connect and handshake to prove the endpoint is alive; stale files are ignored/cleaned safely.

The application should use atomic replace/write so partially written registry files are never treated as live endpoints.

### 9.2 Handshake

Handshake establishes:

- protocol compatibility;
- instance identity;
- consumer identity;
- local authorization proof;
- manifest hash/capability version;
- optional last event sequence for catch-up.

The protocol must not treat an MCP transport session as the application session authority.

## 10. MCP adapter

### 10.1 MCP is a projection, not the core protocol

Agent Bridge semantics must not depend on MCP-specific JSON-RPC structures.

`AgentBridgeMcp` translates between MCP and Agent Bridge.

This protects the applications from MCP version churn and enables other consumers later.

### 10.2 MCP 2026-07-28 direction

The current MCP 2026-07-28 specification moves the core protocol toward stateless request/response, explicit handles, cacheable list results and the Tasks extension for long-running work. This aligns well with Agent Bridge's explicit instance/job IDs rather than hidden transport sessions.

Reference material:

- https://blog.modelcontextprotocol.io/posts/2026-07-28/
- https://tasks.extensions.modelcontextprotocol.io/

Agent Bridge should not copy MCP Tasks internally. It has its own generic job abstraction; the adapter maps jobs to MCP Tasks where the host supports that extension.

### 10.3 Stable bootstrap MCP surface

Do not require hundreds of application tools to be injected into model context merely because the application has a large API.

The generic adapter should always expose a small stable bootstrap surface, approximately:

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

Names may be refined, but the principle is progressive discovery.

A model can ask what is relevant to "dialogue", "selection", "export", etc., receive the matching descriptors/schemas, then call by capability ID.

This also works on MCP hosts that are imperfect at dynamic tool-list refresh.

### 10.4 Optional first-class MCP projection

The adapter may later project selected/high-value application capabilities directly as MCP tools using `tools/list` and list-change/cache semantics.

Do not make this necessary for the first bridge proof.

A future capability flag such as `direct_tool` may be considered only after real host testing demonstrates value. Avoid adding projection policy fields before that evidence exists.

### 10.5 Skills/guides

A skill is useful for workflow knowledge that schemas cannot express, but it is host-specific and therefore optional.

TaskTrack is good evidence: its skill explains when to invoke TaskTrack and how to remain in the interaction lifecycle. That knowledge should remain possible.

Agent Bridge should support an application-level guide resource/text that can be exposed by MCP. A Builder may later generate a starter skill/guide, but the capability manifest remains authoritative.

## 11. Async conversational limitation

Agent Bridge can reliably know that an application job/event completed. It cannot force an arbitrary chatbot product to start a new model turn unless that host supports such a wake/resume mechanism.

Therefore:

- application completion is stored/queryable;
- events notify connected consumers;
- MCP Tasks/subscriptions are used when supported;
- the next agent turn can always discover unseen/current completion state;
- no application correctness depends on spontaneous model wake-up.

TaskTrack's current long-poll compatibility flow remains a valid fallback for hosts where maintaining the active tool round is still the only way to guarantee immediate continuation.

## 12. Threading and UI safety

Network callbacks must not mutate U++ GUI/application state directly from an I/O thread.

V1 default:

- one small I/O/connection thread per endpoint or a simple bounded connection loop;
- registered application handlers are marshalled to the application's designated dispatcher/main thread;
- result returns to the connection after handler completion;
- asynchronous jobs explicitly detach from the request and publish completion later.

Do not build a general actor runtime or thread pool into Agent Bridge.

## 13. Revisions and concurrency

A command request envelope can carry:

```text
expected_revision
consumer_id
```

Revision meaning remains application-defined. Applications without useful revisions can omit it.

When supplied, stale mutation must be rejected clearly rather than silently applied.

Multiple consumers may connect simultaneously. Agent Bridge does not attempt collaborative document merging. The application's normal command/revision authority decides whether a mutation is safe.

## 14. Security and remote use

### 14.1 Local default

- loopback bind only;
- per-user discovery location;
- random instance authentication material;
- verify endpoint/instance identity during handshake;
- do not trust a discovery file alone.

### 14.2 Remote future

The same frame protocol should be able to run through TLS/TCP.

Remote mode must not be enabled until there is:

- encryption (TLS);
- explicit authentication;
- endpoint/application identity verification;
- capability/access policy appropriate to the host application;
- sane rate/size/time limits.

Do not ship a plaintext "remote=true" switch as an interim shortcut.

Remote discovery is explicit endpoint configuration initially. Internet-wide service discovery is a separate concern and should not be embedded in Agent Bridge V1.

## 15. Existing application survey

### 15.1 UiDesigner — strongest first full application proof

Current architecture already contains:

- `UiDesignerCommandService` with atomic mutations and undo/redo history;
- `UiDesignerSession` with document/selection state and revision-aware edit intent;
- `UiDesignerAutomationService` with a substantial machine-facing query/command surface;
- an MCP process whose tool list is currently hand-authored.

Important current limitation: `UiDesignerMcpServer` owns its own headless `UiDesignerSession`, so it is not a bridge into the live GUI session.

Agent Bridge integration should reuse the automation/service layer while moving transport/session ownership to the running application.

Acceptance proof:

1. discover two running Designer instances;
2. fetch current context/selection from the chosen live instance;
3. query a control/property schema;
4. execute a mutation through existing command authority;
5. observe revision change/event;
6. undo through existing history;
7. reject an intentionally stale expected revision.

### 15.2 TaskTrack — async/lifecycle proof

TaskTrack already has:

- a GUI/MCP-independent semantic core;
- durable task and assistance state;
- a separate MCP executable;
- explicit human/agent interaction phases;
- a host-facing skill;
- current compatibility long polling for hosts that cannot otherwise resume the model.

It is a useful second integration because it stresses the hardest bridge area: application events, human input, asynchronous completion and agent assistance.

Do not rewrite TaskTrack's durable evidence authority into Agent Bridge. Instead map:

- TaskTrack task ID -> application job/resource identity where useful;
- assistance request -> event/current-state query;
- terminal completion -> job/event result;
- existing skill -> application usage guide.

Keep the existing compatibility path until AgentBridgeMcp + target hosts prove a better Tasks/subscription path end to end.

### 15.3 Dramatica — query/service proof

`ThroughlineCore` already separates domain logic from UI. `StoryFilterService` exposes clear read/query operations over candidate state and hierarchy.

This is a good proof that Agent Bridge does not require an undo/redo command system to be useful. Initial integration can be mostly queries/resources/context, adding mutation commands only where the application naturally owns them.

### 15.4 UiSymbolPicker — scale/resource proof

SymbolPicker manages a model-driven catalogue of more than 15,000 generated entries and deterministic asset export.

It is a good proof for:

- search rather than dumping a giant catalogue;
- resource IDs;
- binary image/export data;
- deterministic generated artifacts;
- progressive capability/data discovery.

Agent Bridge should never transmit the whole symbol catalogue merely because the agent connected.

### 15.5 AgentFlow — later integration, not a merge

AgentFlow already separates workflow authority, runtime, provider adapters and side effects, and explicitly states that MCP is an integration transport rather than a trust boundary.

Agent Bridge can later appear as one external application/tool capability provider available to AgentFlowRuntime. It must not replace AgentFlow's runtime, evidence/context contracts or side-effect authorization broker.

AgentFlow may also eventually expose its Workbench through Agent Bridge as an application. Those are separate roles.

### 15.6 Future writing/image-review/production applications

These are important acceptance targets even before their concrete integrations exist.

Writing example:

1. context says the active semantic scope is scene `SC-042`;
2. agent queries scene context and participating characters;
3. agent proposes/preview-mutates dialogue through normal application commands;
4. agent starts an asynchronous multi-voice performance job;
5. application exposes WAV/audio outputs as resources;
6. human continues talking while job runs;
7. completion is observable without binding application correctness to a blocked request.

Image review and production tracking similarly require large resources, comments/annotations, selection/context, revisions and potentially remote connections. The generic primitives above are intended to cover them without adding media- or production-specific concepts to Agent Bridge itself.

## 16. Implementation stages

### AB-001 — architecture contract

This document and project README.

Acceptance:

- agree on minimal two-package shape;
- agree that application state/commands remain authoritative;
- agree on no broker daemon for V1;
- agree on capability/context/query/command/job/event/resource vocabulary;
- agree on loopback TCP + binary framing direction.

### AB-002 — wire + tiny demo

Implement only the minimum reusable `AgentBridge` package and a console/demo application.

Scope:

- protocol/version constants;
- binary frame parser/writer with strict limits;
- tiny binary value codec;
- application/instance handshake;
- local registry;
- manifest transfer/hash;
- query call;
- command call;
- basic context;
- connection shutdown/reconnect tests.

No MCP yet.

Tests must include malformed lengths, unknown message types, version mismatch, stale registry record and deterministic codec vectors.

### AB-003 — jobs/events/resources

Add:

- asynchronous job lifecycle;
- bounded event sequence/ring;
- resource descriptors and chunk reads;
- cancellation where the application supplies it;
- multi-consumer connection test.

Keep persistence outside Agent Bridge.

### AB-004 — generic MCP adapter

Implement `AgentBridgeMcp` with the small stable bootstrap tool surface.

Requirements:

- discover running instances;
- explicit instance IDs in calls rather than hidden MCP transport session state;
- capability search/describe/call;
- context;
- job polling / MCP Tasks mapping where supported;
- resources;
- clear host-compatible errors;
- capability/manifest caching.

Do not copy application-specific commands into MCP source.

### AB-005 — UiDesigner integration

Replace/augment the current headless-only MCP path with live running-application Agent Bridge access while preserving existing UiDesigner command/session authority.

Do not regress standalone CLI/headless automation merely to prove the bridge.

### AB-006 — TaskTrack async integration

Use TaskTrack to validate:

- application-originated events;
- awaiting-human / awaiting-agent transitions;
- async completion;
- MCP Tasks/subscription behaviour on capable hosts;
- compatibility fallback on hosts that still require the current wait loop;
- skill/guide separation from capability truth.

### AB-007 — breadth proof

Integrate at least one of:

- Dramatica for query-heavy service access;
- UiSymbolPicker for large catalogue/resource/artifact access.

If both fit without changes to core semantics, the abstraction has strong evidence of reuse.

### AB-008 — Builder/generator experiment

Only after the descriptor/binding API stabilises.

A small Agent Bridge Builder may inspect source/API surfaces with AI assistance and generate:

- capability registration skeleton;
- bindings to likely services/commands;
- schemas/descriptions;
- initial tests;
- optional application guide/skill draft.

Generated code must remain ordinary readable C++, not a runtime reflection dependency.

## 17. Builder philosophy

The Builder should accelerate integration, not become required infrastructure.

A developer must always be able to integrate Agent Bridge manually with a few clear registrations.

AI/source inspection may identify likely commands and queries, but generated bindings are reviewed C++ and committed normally. No runtime C++ parser, code-generation daemon or metadata compiler is required for Agent Bridge operation.

## 18. What not to build yet

Do not add these without concrete pressure:

- permanent local broker daemon;
- database;
- distributed event log;
- universal application object model;
- universal command inheritance framework;
- plugin framework;
- dependency injection container;
- reflection system;
- separate package for every conceptual noun;
- local shared-memory transport before measurement;
- application-specific MCP servers;
- remote plaintext mode;
- hidden duplicate application state in the adapter.

## 19. Initial source layout target

Keep this intentionally boring:

```text
AgentBridge/
    AgentBridge.h          public API + small public structs
    AgentBridge.cpp        registry, endpoint, codec, transport, routing
    AgentBridge.upp

AgentBridgeMcp/
    AgentBridgeMcp.h       MCP translation surface
    AgentBridgeMcp.cpp
    main.cpp
    AgentBridgeMcp.upp

examples/
    BridgeDemo/            only when AB-002 needs it

tests/
    AgentBridgeTest/       protocol/registry/semantic tests

docs/
    ARCHITECTURE_PLAN.md
    ACTIVE_WORK.md
```

If `AgentBridge.cpp` genuinely becomes too large, split by implementation responsibility then. Do not pre-split based on the architecture diagram.

## 20. V1 success criteria

Agent Bridge V1 is successful when all of the following are true:

- adding Agent Bridge to an existing U++ application requires a small, readable integration surface;
- the application describes each capability in one place;
- the generic MCP adapter can discover and operate several application types without application-specific MCP code;
- a live UiDesigner instance can be inspected and mutated through its existing command authority;
- TaskTrack-style asynchronous completion does not require Agent Bridge itself to keep the original application request blocked;
- large binary resources can be transferred without base64 expansion;
- multiple application instances and multiple agent consumers work safely;
- stale revision writes are demonstrably rejected by an application that supports revisions;
- local transport is fast enough that bridge overhead is insignificant compared with model/tool latency;
- the source remains compact enough that a new developer can understand the full bridge without navigating a framework-sized directory tree.

## 21. Decisions deliberately left open until implementation evidence

These are bounded implementation choices, not architecture gaps:

- exact frame header widths;
- exact binary value type tags/length encoding;
- exact per-user registry path/name;
- exact local token/challenge mechanism;
- whether selected app capabilities should later be projected as first-class MCP tools in addition to generic discovery;
- exact TLS/auth mechanism for remote mode;
- whether repeated application integrations justify an optional shared command/history helper.

Resolve these in the smallest milestone that needs them and record the reason in `docs/ACTIVE_WORK.md`.
