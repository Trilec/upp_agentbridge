# ACTIVE WORK

Remote GitHub `main` is authoritative. Refresh exact HEAD before implementation or publication. Never assume this document's recorded SHA is still current.

## CURRENT STATE — 2026-08-26

STATUS: **INITIAL ARCHITECTURE PLAN CREATED; NO PRODUCTION IMPLEMENTATION YET.**

BASE:

- `upp_agentbridge/main`: `f8d45cd87641b14a1b5078d7a8328228007be2bd`

TASK:

- `AB-001` — establish the lightweight reusable Agent Bridge architecture and staged implementation plan.

INSPECTED REFERENCE BASELINES:

- `upp_tasktrack/main`: `6906d9bce4826bb0e7e0c0787f557077e1a27bda`
- `upp_uidesigner/main`: `2e0b9025327749b7d570c76cb09888b7f4db49d7`
- `upp_dramatica/main`: `d8eb2de7487bffb3c895f9b64e49fb9f54c1edd2`
- `upp_uisymbolpicker/main`: `974081449ef9dc0628d52e313baf2782f5b69e6f`
- `upp_agentflow/main`: `4d1dcfb0e426c80fbd29c73f52e838aa890930a6`

DECISIONS:

- keep V1 to one reusable `AgentBridge` application package and one generic `AgentBridgeMcp` executable/package;
- no permanent broker daemon in V1;
- application owns state, commands, undo/redo, validation and persistence;
- capability declaration is single-source and projected outward;
- context/query/command/job/event/resource are the small reusable semantic vocabulary;
- existing application command systems are bound, not replaced;
- local discovery uses per-user instance records plus verified direct endpoints;
- V1 transport direction is loopback TCP with versioned binary framing;
- bulk binary resources are chunked/raw rather than base64 encoded;
- remote mode reuses the same protocol but must require TLS/auth before it is enabled;
- generic MCP progressive discovery is preferred before exposing very large application tool lists;
- MCP Tasks/subscriptions are adapter features, not Agent Bridge core semantics;
- skills/guides may explain application workflows but are not capability truth.

TOUCHED:

- `README.md`
- `docs/ARCHITECTURE_PLAN.md`
- `docs/ACTIVE_WORK.md`

VALIDATION:

- architecture derived against the current Designer command/session/automation path;
- TaskTrack lifecycle/MCP/skill reviewed as async and human-agent interaction evidence;
- Dramatica service boundary reviewed as a query-heavy integration case;
- UiSymbolPicker reviewed as large-catalogue/binary-artifact case;
- AgentFlow provider/core direction reviewed to avoid merging bridge responsibilities into its runtime;
- MCP 2026-07-28 direction checked for stateless core, explicit handles, cacheable lists, Tasks extension and subscription changes.

NEXT ACTION:

1. Refresh `upp_agentbridge/main` and confirm this documentation checkpoint is present.
2. Review/accept the AB-001 design decisions before source implementation.
3. Start `AB-002` as a bounded wire/demo slice only: protocol constants, strict binary frames, tiny value codec, local registry/handshake, manifest transfer, context, one query and one command.
4. Do **not** add MCP, jobs/resources, remote TLS, Builder tooling or application integrations inside AB-002.
5. Keep the first production package/file surface compact; split files only when implementation size/ownership demonstrates the need.
