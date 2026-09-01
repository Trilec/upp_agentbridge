# ACTIVE WORK

Remote GitHub `main` is authoritative. Refresh exact HEAD before implementation or publication. Never assume this document's recorded SHA is still current.

## CURRENT STATE — 2026-09-01

STATUS: **ARCHITECTURE REVISED TOWARD U++ D-BUS; NO PRODUCTION IMPLEMENTATION YET.**

BASE:

- `upp_agentbridge/main`: `be3f90e0e8aa21dddc35ec38372e55bc209bd177` before this documentation revision.
- `Trilec/DBus/main`: `818c6f3eed9c1976d3618d5ea4d7f48363e8fd26` reviewed transport baseline.

TASK:

- `AB-001R2` — clarify Agent Bridge transport direction before source implementation.

DECISIONS:

- Agent Bridge owns application/agent semantics, not a custom transport protocol;
- preferred communication foundation is the small native U++ `DBus` package;
- do not implement the previously planned custom binary frame/value codec first;
- D-Bus method/reply/signal primitives map cleanly to Agent Bridge query/command/result/event transport;
- keep capability/context/query/command/job/event/revision/resource as the transport-independent semantic vocabulary;
- normal local D-Bus/session routing should be used where appropriate instead of duplicating local IPC/discovery;
- the planned lightweight D-Bus TCP adapter is the preferred direct/Windows/remote extension;
- if the TCP adapter is not supplied upstream, we can implement it from the available source/fork without changing Agent Bridge semantics;
- raw TCP is not remote security: remote use must add authentication and TLS/encryption;
- large images/audio/files remain Agent Bridge resources and should not be forced into giant ordinary D-Bus control messages;
- keep one reusable `AgentBridge` package and one generic `AgentBridgeMcp` package/executable; add no transport package hierarchy unless implementation proves it necessary;
- no permanent Agent Bridge broker daemon by default;
- existing application command/history/durable state remains authoritative;
- generic MCP progressive discovery remains the preferred model-facing surface.

DOCUMENTATION UPDATED:

- `README.md` — D-Bus is now the preferred communications foundation and TCP adapter direction is explicit.
- `docs/ARCHITECTURE_PLAN.md` — old TCP-first/custom-codec sections replaced with the current D-Bus-first architecture and implementation sequence.
- `docs/ACTIVE_WORK.md` — this checkpoint.

NEXT ACTION — `AB-002`:

1. Refresh `upp_agentbridge/main` and `Trilec/DBus/main` before work.
2. Build only a tiny D-Bus bridge proof: application registration, instance info/manifest, context, one query, one command and one application-originated event.
3. Use the existing U++ DBus package directly; do not build a parallel binary codec/transport stack.
4. If the DBus TCP adapter is already available, add one minimal direct connection proof. If not, document the adapter contract and continue the local proof without blocking on remote work.
5. Do not add MCP, full jobs/resources, remote TLS, Builder tooling or broad application integrations inside AB-002.
6. Keep source/file count minimal and review the complete diff before publication.
