# gala-acp — Agent Communication Protocol

A standalone, transport-neutral [GALA](https://github.com/martianoff) library for
building **reliable** agent orchestrators. It is the shared protocol kernel
extracted from [`gala_team`](https://github.com/martianoff/gala_team)'s
`app/protocol` and designed to fit
[`gala-assimilator`](https://github.com/martianoff/gala-assimilator)'s engine
ACP — one library, reused by both.

Everything here is **pure data + pure functions**: no subprocess, no
filesystem, no UI, no clock. Side effects (streaming over a socket, spawning a
subprocess, ticking a live UI) live at the edges of the consuming application;
the protocol itself never performs IO.

## What's inside

One package, `acp`, four concerns:

| File | What it gives you |
|------|-------------------|
| `envelope.gala` | `Envelope` — the on-the-wire message with **content-hash identity**. Re-deriving an envelope from the same inputs yields the same `Id`; idempotency, dedup, and the stale-redelivery defence all fall out of that one fact. |
| `ledger.gala` | `Ledger` + `DedupById` — the durable dedup gate keyed on that identity. Admit each `Id` exactly once; re-emitted chunks / replays / redeliveries collapse. |
| `delivery.gala` | The delivery / ack / retry / nudge **liveness FSM** (`DeliveryState`, `DeliveryAction`, `Tick`, `OnChunk`, `OnTerminal`, `AggregateNudges`). A pure function of (state, event, clock, policy). |
| `run.gala` | The generic **agent-run contract**: `AgentRunner[A, R, C]` port, `RunTranscript` (stream-as-data), `RunOutcome` (result / clarification / aborted terminus), `ProgressEvent`. |

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
