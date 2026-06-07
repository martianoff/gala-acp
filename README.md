# gala-acp — Agent Communication Protocol

A standalone, transport-neutral [GALA](https://github.com/martianoff) library for
building **reliable** agent orchestrators. It is the shared protocol kernel
extracted from [`gala_team`](https://github.com/martianoff/gala_team)'s
`app/protocol` and designed to fit
[`gala-assimilator`](https://github.com/martianoff/gala-assimilator)'s engine
ACP — one library, reused by both.

The **`acp` kernel** is **pure data + pure functions**: no subprocess, no
filesystem, no UI, no clock. The optional **`agent` package** is the impure
edge — a transport-neutral agent port plus a `claude` CLI backend — kept in its
own package so the kernel stays pure and dependency-light.

## What's inside

### `acp` — the pure protocol kernel

| File | What it gives you |
|------|-------------------|
| `envelope.gala` | `Envelope` — the on-the-wire message with **content-hash identity**. Re-deriving an envelope from the same inputs yields the same `Id`; idempotency, dedup, and the stale-redelivery defence all fall out of that one fact. |
| `ledger.gala` | `Ledger` + `DedupById` — the durable dedup gate keyed on that identity. Admit each `Id` exactly once; re-emitted chunks / replays / redeliveries collapse. |
| `delivery.gala` | The delivery / ack / retry / nudge **liveness FSM** (`DeliveryState`, `DeliveryAction`, `Tick`, `OnChunk`, `OnTerminal`, `AggregateNudges`). A pure function of (state, event, clock, policy). |
| `run.gala` | The generic **agent-run contract**: `AgentRunner[A, R, C]` port, `RunTranscript` (stream-as-data), `RunOutcome` (result / clarification / aborted terminus), `ProgressEvent`. |

### `agent` — the transport port + Claude backend (`github.com/martianoff/gala-acp/agent`)

The impure edge: where "an agent" stops being pure data and becomes a live
conversation. Import this only if you need to actually *run* an agent.

| File | What it gives you |
|------|-------------------|
| `agent.gala` | The transport-neutral port: `AgentTransport` (open a session), `AgentSession` (`Send`/`EndTurn`/`NextEvent`/`Close`), and the `AgentEvent` / `AgentInput` / `AgentSpec` value types. Swap the backend without touching the orchestrator. |
| `claude_transport.gala` | `ClaudeProcessTransport` — the default backend: a local `claude` CLI subprocess over stream-json (prompt on stdin, `--input-format`/`--output-format stream-json`), with the IO ↔ `AgentEvent` mapping and the transport-auth-error gate. |
| `claude_stream_json.gala` | The claude-code stream-json classifiers/extractors (envelope shape, tool_result blocks, stale-resume detection). |
| `session_id.gala` | `NewSessionId` — RFC 4122 v4 UUIDs for `--session-id` / `--resume`, so parallel member subprocesses don't race on claude's "most recent in cwd" semantics. |

`AgentSpec` carries only what a transport needs (`Model`, `Workspace`,
`SessionId`, `IsNewSession`, `DangerouslySkipPermissions`) — no consumer-
specific member type. The `TraceClaudeLine` diagnostic hook (`trace.gala`) is a
documented no-op: tracing to a debug sink is a consumer concern, so the package
ships no file IO.

### Vocabulary is the consumer's

The kernel carries **no domain message taxonomy**. `Envelope.Kind` is an opaque
lowercase string tag, so one app can speak `dispatch / finished / consult`
while another speaks `task_assignment / result / verdict` — both ride the same
envelope, the same dedup `Ledger`, the same delivery FSM. A consumer that wants
an exhaustive sealed kind keeps its own sealed type and projects it to the tag
at the boundary (the tag is all the kernel needs for identity and routing).

The run contract (`run.gala`) is generic over the consumer's payload types
(`A` = assignment, `R` = result, `C` = clarification) so each application plugs
in its own shapes while sharing the port, the transcript, and the terminus
invariant.

## Using it

### Bazel (bzlmod)

```starlark
# MODULE.bazel
bazel_dep(name = "gala_acp", version = "0.1.0")

# Until gala-acp is published to a registry, point at a local checkout:
local_path_override(module_name = "gala_acp", path = "../gala-acp")
```

A cross-module GALA library must sit on the transpiler's search path, so wire
it through **`gala_deps`**, not `deps`:

```starlark
gala_library(
    name = "...",
    srcs = [...],
    gala_deps = ["@gala_acp//:gala_acp"],
    deps = [...],
)
```

If you use the gazelle extension, add a resolve directive so it keeps the dep in
`gala_deps`:

```
# gazelle:resolve gala github.com/martianoff/gala-acp @gala_acp//:gala_acp
```

### Pure GALA (`gala build` / `gala test`)

```
# gala.mod
require github.com/martianoff/gala-acp v0.1.0
```

Then `import "github.com/martianoff/gala-acp"` and reference `acp.Envelope`,
`acp.Tick`, `acp.AgentRunner`, etc.

## Build & test

```sh
bazel build //...
bazel test  //...
```

## A taste

```gala
import . "github.com/martianoff/gala-acp"

// Reliable messaging: identity is a content hash, dedup is a one-liner.
val m       = NewEnvelope("_lead", "Felix", "dispatch", "build the parser", 0)
val (l2, _) = EmptyLedger().Admit(m)   // admits once; a re-decoded duplicate is rejected

// Liveness: one total FSM decides retry / nudge / abandon.
val (next, action) = Tick(Sent(AtNanos = 0, Attempt = 1), m, nowNanos, DefaultDeliveryPolicy())

// The run contract: a Submit is a pure TaskAssignment -> RunTranscript value.
val runner = StaticRunner[string, string, string](
    RunTranscript[string, string](
        Steps = ArrayOf[RunStep](ProgressStep("task-1", "reading", "reading source")),
        Final = OutcomeResult[string, string](Value = "// translation"),
    ),
    Cancelled(),
)
```
