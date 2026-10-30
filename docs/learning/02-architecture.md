# 2. Architecture and execution modes

[Guide home](README.md) · Previous: [Conceptual model](01-system-model.md) · Next: [Walkthroughs](03-source-walkthroughs.md)

## The process boundary is as important as the crate boundary

For an interactive daemon-backed session, the CLI starts/locates the supervisor
and the terminal client attaches. The supervisor routes commands and owns worker
lifecycle. Each active session has a worker process hosting a session engine.
That engine calls a model provider and provisions a separate Python process for
REPL work. RLM children get their own supervised workers.

| Runtime participant | Owns | Does not own |
| --- | --- | --- |
| CLI/TUI client | Terminal input, rendering, client-side preferences and commands | The lifetime of a resident agent after detachment |
| Supervisor | Worker discovery, launch, routing, adoption/recovery, descriptors | The model's Python namespace |
| Session worker | Session engine, turn admission/queueing, durable events, session services | All other sessions in one shared engine |
| Python kernel | Cell execution, namespace, runtime modules, integration clients | Provider requests or supervisor policy |
| Provider service | Model inference and provider-specific streaming | Local file access, unless code sends it content |

Follow [interactive composition][interactive], [Supervisor][supervisor],
[worker creation][worker-create], [kernel manager][kernel-manager], and
[supervisor-backed children][children]. “Separate process” improves fault and
lifecycle separation; it does not establish a security boundary.

Print/text and JSON output are a second composition. They host the session engine
**inside the CLI process**, and their recursive children are inline session nodes
in that process. The print driver waits for admitted descendants and pending
messages to settle before disposal. There is no service left behind to wake a
print session later. See [print composition][print] and [inline children][inline].
Do not use the daemon process description as a universal description of all modes.

## Ten crates, with explicit ownership

The workspace manifests, not just prose READMEs, establish the dependency graph.
The table below lists **direct workspace dependencies**, excluding third-party
libraries and dev-only test dependencies.

| Crate | Responsibility | Direct workspace dependencies |
| --- | --- | --- |
| `pa-types` | Shared wire/domain vocabulary, platform contracts | None |
| `pa-telemetry` | Typed events, sanitization, nonblocking delivery | None |
| `pa-agent` | Provider-independent loop, tools, streams, cancellation | None |
| `pa-ai` | Provider implementations, streaming facade and registry | `pa-types` |
| `pa-models` | Live catalog, disk/bundled/compiled fallback chain | `pa-ai`, `pa-types` |
| `pa-core` | Session semantics, settings, resources, kernel, goals, refinement | `pa-types`, `pa-ai`, `pa-models`, `pa-agent`, `pa-telemetry` |
| `pa-daemon` | Supervisor, workers, transport and session hosting | `pa-types`, `pa-core`, `pa-agent`, `pa-ai`, `pa-telemetry`, `pa-models` |
| `pa-tui` | Terminal client and rendering from events | `pa-types` |
| `pa-cli` | Binary entry point and composition of execution modes | `pa-types`, `pa-core`, `pa-ai`, `pa-agent`, `pa-daemon`, `pa-tui`, `pa-telemetry` |
| `pa-perf` | Development/CI performance harness | `pa-types`, `pa-core`, `pa-daemon` |

Start at [the workspace manifest][workspace] and each member's `Cargo.toml`.
Two especially useful constraints are that `pa-tui` does **not** link `pa-core`,
and `pa-agent` does **not** link `pa-ai`. UI code cannot casually reach into live
engine state; the loop cannot casually depend on one provider's transport.

There is a deliberate adaptation cost: `pa-agent` uses its own loop-facing types,
and [provider_adapter.rs][adapter] crosses into `pa-ai`/`pa-types` with JSON-shape
conversion. This is not one shared Rust struct propagated everywhere. The adapter
also carries cancellation, headers, and live provider-target selection. **Inference:**
independent loop reuse and compatibility are prioritized over zero-copy conversion
and compile-time identity across those representations.

## The Python sidecar is part of the current product

`prime-agent-runtime/` is not abandoned TypeScript-era scaffolding. The Rust
bootstrap locates the trusted local runtime sources, resolves/provisions a Python
environment with `uv`, and starts its REPL protocol. Source checkout discovery
and executable-adjacent package layouts are supported. `PI_PACKAGE_DIR` is a
compatibility environment variable that still has meaning in the Rust product.

The [runtime-source resolver][runtime-source] checks for the manifest and actual
`rlm` sources. It explicitly refuses to substitute a package-index runtime when
the local sidecar is missing. Its content hash includes Python sources and the
packaged machine library so a changed runtime invalidates the environment cache.
The [bootstrap module][bootstrap] also checks default Python packages and skill
imports. A Rust binary alone is therefore not the whole installation.

Ownership on this side is concrete:

- `repl.py`: request loop, execution, output routing, interrupt/snapshot handling.
- `__init__.py`: typed `rlm.*` host-call wrappers and result parsing.
- `bash.py`: shell execution and persistent/background process behavior.
- `harness.py`: Python-facing harness state operations.
- `factory.py`: declarative workflow validation and execution.
- `mcp.py`: generic MCP connections, discovery, and tool calls.

## Three transports that should not be conflated

| Boundary | Encoding / role | Source |
| --- | --- | --- |
| Public daemon command connection | JSON-lines commands, responses, events over local transport | [Supervisor routing][routing], [protocol][protocol] |
| Private worker/direct-attach connection | Eight-byte big-endian length prefix, JSON header, binary payload | [Shared private framing][framing] |
| Rust host to Python runtime | Protocol v3 JSON-lines requests and events; IDs correlate replies | [Kernel protocol][kernel-protocol], [runtime request handling][repl] |

The private prefix contains two 32-bit lengths: header bytes and payload bytes.
The decoder checks bounds and retains incomplete chunks until a full frame
arrives. It is not simply another newline-delimited socket.

The kernel protocol includes `execute`, `interrupt`, `host_reply`, `snapshot`,
`restore`, `list_names`, `mcp_status`, and `shutdown`. Python sends `ready`, output,
result/display, host-request, error, and done events. The runtime separates
protocol output from user output so a cell's `print` does not become malformed
control JSON. A `host_request` can ask Rust to spawn a child or mutate session
state while a Python cell is running.

## Where to follow an input

This is a reading order, not a literal single call stack: asynchronous tasks and
process messages cross between these stages.

1. **Select a composition:** `pa-cli/src/main.rs`, `lib.rs`, and `mode.rs`.
   Explicit daemon/RPC/ACP/JSON modes take priority; print is also selected for
   non-TTY stdin.
2. **Admit a prompt:** daemon worker input/queue code, or print runtime admission.
   Session commands such as compaction are not merely ordinary model prompts.
3. **Build/drive the session:** `pa-core/src/session_engine/` prepares resources,
   messages, tools, policy hooks, and persistence.
4. **Generate a turn:** `pa-agent/src/agent_loop/` streams an assistant response
   and executes its tool calls.
5. **Cross outward:** the provider adapter sends inference work to `pa-ai`;
   `python_repl` sends execution work to the kernel manager.
6. **Publish and persist:** engine events become session records and client events.
   The TUI renders its event-derived state.

The boundaries explain why adding a provider belongs in `pa-ai`, while changing
compaction policy belongs in `pa-core`, and changing how a tool card looks belongs
in `pa-tui`. A feature that requires the TUI to import the session engine would
cut across an existing architectural constraint.

[interactive]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-cli/src/interactive_mode.rs
[supervisor]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-daemon/src/supervisor.rs
[worker-create]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-daemon/src/worker/create.rs
[kernel-manager]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/kernel/manager/mod.rs
[children]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-daemon/src/rlm_children.rs
[print]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-cli/src/print_runtime.rs
[inline]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-cli/src/print_rlm.rs
[workspace]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/Cargo.toml
[adapter]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/provider_adapter.rs
[runtime-source]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/kernel/bootstrap/venv/runtime_source.rs#L9-L87
[bootstrap]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/kernel/bootstrap/mod.rs
[routing]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-daemon/src/supervisor/routing.rs
[protocol]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-daemon/src/protocol.rs
[framing]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-types/src/daemon/framing.rs#L1-L150
[kernel-protocol]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/kernel/protocol.rs#L1-L162
[repl]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/prime-agent-runtime/src/rlm/repl.py
