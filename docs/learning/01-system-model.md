# 1. Functionality and conceptual model

[Guide home](README.md) · Next: [Architecture](02-architecture.md)

## Think of a session as a small, programmable work environment

A session combines a conversation, a selected model, a working directory, a
Python namespace, and host services. The model does not execute directly inside
Rust: it requests `python_repl` with a `code` string. The session's Python process
executes the cell and returns output. Its namespace survives later cells, so a
large document can stay in a variable while only selected excerpts or computed
results return to the model.

This is the concrete meaning of the repository's **recursive language model
(RLM)** framing: code can retain and manipulate context outside the model's token
window and call child agents as operations. It is not evidence of a particular
accuracy or cost advantage. The real implementation is ordinary provider calls,
a Rust agent loop, a persistent Python interpreter, and host-request messages.
See [session assembly][engine], [the tool definition][tool], and [the runtime][repl].

For example, in a repository investigation the model could:

1. Load filenames or a long log into Python rather than print the entire dataset.
2. Compute a shortlist and print only relevant excerpts.
3. Spawn two named children to investigate independent questions.
4. Collect their results or receive explicit messages, inspect their evidence,
   and continue using the same namespace.
5. Save reusable lessons into the harness, while keeping temporary data local.

This is an **illustrative workflow**, not a promise that any selected model will
choose those steps or use them correctly. Children can access the same working
directory; parallel spawning does not automatically isolate their file edits.

## The vocabulary separates four kinds of state

| Term | What it is | What it is not |
| --- | --- | --- |
| Model context | System prompt plus converted conversation messages sent to a provider | All Python variables or every historical session row |
| Python namespace | Live objects retained by the kernel between cells | A lossless, restart-proof database |
| Session history | Durable JSONL entries with IDs and parent links; includes messages and control records | Just the flat text visible on screen |
| Continual harness | Editable prompt notes, memories, skill references, subagent specifications, and factory specifications | Model weights or the immutable base prompt |

Compaction reduces the first category using summaries and retained history. Kernel
snapshots help recover the second. Session storage maintains the third. Refinement
edits the fourth. They interact, but they solve different problems. See
[session trees][tree], [snapshot result shapes][snapshots], and
[refinement planning][planner].

A **turn** here is one assistant response and its tool execution phase. A single
user request can produce several turns. A **run** of the agent loop can also
include steering, follow-up messages, and policy-generated continuations before it
ends. A **worker** is a process hosting a daemon-backed session; it is not a
synonym for a model call or a Python kernel. These distinctions matter when
reading usage counters, abort handling, and retry code.

## What the inspected implementation can do

| Capability | Concrete mechanism | Boundary to remember |
| --- | --- | --- |
| Code, files, shell commands | Model calls `python_repl`; Python imports skills and exposes `bash` | Code runs with the operating-system user's permissions |
| Interactive sessions | Rust TUI sends commands and consumes daemon events | Closing a client is different from stopping a session |
| Scriptable execution | Print/text, JSON event output, RPC, ACP, and daemon modes | These modes do not all have the same hosting/lifetime model |
| Recursive delegation | `rlm.spawn`, `rlm.collect`, `rlm.list_subagents`, deletion and messaging | Admission returns a handle, not the child's completed answer |
| Persistent objectives | Goal state, progress/budget bookkeeping, continuation messages | A goal is not independent evidence that the task succeeded |
| Bounded autonomous work | Turn/token/time limits and optional shell quality gates | A passing gate proves only the property that command checks |
| Scheduled work | Durable jobs, a scheduler, and per-session dispatch lanes | Daemonless print runs have no independent scheduler after exit |
| Long-context work | Context estimates, threshold/overflow/manual compaction | Summaries are lossy and provider token counts are not exact estimates |
| Reusable capabilities | Markdown skills and detected Python skill packages | A skill description is different from installing executable code |
| Integrations | Rust resolves MCP configuration/auth; Python connects and calls servers | External services and their schemas remain separate trust boundaries |
| Harness refinement | Model-proposed create/update/delete edits and recorded history/rollback | No weight training; no guarantee edits improve later behavior |
| Factory workflows | Opt-in Python state-machine execution over child agents | Factory run state is kernel-local, even though specs can be durable |

Implementation entry points: [mode selection][modes], [RLM host contract][rlm],
[goal driver][goals], [autonomous driver][autonomous], [scheduler][scheduler],
[compaction][compaction], [skill discovery][skills], [MCP runtime][mcp], and
[factory executor][factory]. This table establishes source presence, not an
exhaustive end-to-end certification of each feature.

## Delegation is asynchronous, and collection is explicit

`await rlm.spawn(prompt, name="reader")` awaits **admission**. The returned handle
contains child identity, name, session directory, and model. Python's `await` does
not turn it into an answer-producing call. The wrapper sends the compatibility
wire request `rlm.run`; the Rust bridge delegates to the embedding's child host.

The current [Python API][python-api] implements `rlm.collect`: with `timeout_ms=0`
it returns current snapshots; with a positive timeout it waits up to that limit.
A timeout returns the states reached so far, not an exception or a guarantee of
completion. Results distinguish `queued`, `running`, `done`, `error`, and
`cancelled`, plus a `settled` flag and optional answer/error fields.

Explicit agent-to-agent messages serve a different purpose: they communicate or
steer work. A child having replied and a child having settled are different facts.
`rlm.create_session` is different again: it creates a resident depth-zero session
and is supported only from a daemon-backed depth-zero caller.

## “Self-improving” has a narrow, inspectable meaning

The [refiner][planner] asks a model for a JSON proposal describing edits to
supplemental harness entries. The [executor][refine-executor] plans, re-reads the
store, applies edits, and records outcomes. Local session scope is the default;
global refinement must be requested explicitly. The base system prompt is not
one of the editable entry kinds.

Those entries later influence context through a separately delivered
`[harness-digest]` message. This is **external state adaptation**. It is not
fine-tuning, reinforcement-learning training, or proof of continual performance
improvement. A recorded rationale and expected outcome are model claims until an
independent check validates them. Skill references also do not magically create
and test the Python packages they name.

## Check your understanding

- If a child spawn returns successfully, has the child's task finished? **No.**
- If a session compacts, are all Python variables necessarily gone? **No; these
  are separate mechanisms, although explicit snapshot/pruning behavior matters.**
- If the terminal disconnects, must the agent stop? **Not in a daemon-backed
  session.** A print process has a different lifetime.
- If a quality gate passes, is the whole research conclusion true? **Only if the
  gate actually establishes that conclusion.**
- Does a `memory` harness entry change model weights? **No.**

[engine]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/engine.rs#L450-L595
[tool]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/tools/ipython.rs#L344-L357
[repl]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/prime-agent-runtime/src/rlm/repl.py#L597-L697
[tree]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session/tree.rs
[snapshots]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/kernel/state_snapshot.rs
[planner]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/refinement/planner.rs
[modes]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-cli/src/mode.rs#L9-L60
[rlm]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/rlm_host.rs
[goals]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/goal_driver.rs
[autonomous]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/autonomous/driver.rs
[scheduler]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/cron/scheduler.rs
[compaction]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/compaction.rs
[skills]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/skills/discovery.rs#L35-L145
[mcp]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/prime-agent-runtime/src/rlm/mcp.py
[factory]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/prime-agent-runtime/src/rlm/factory.py#L1655-L1905
[python-api]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/prime-agent-runtime/src/rlm/__init__.py#L179-L473
[refine-executor]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/refinement/executor.rs
