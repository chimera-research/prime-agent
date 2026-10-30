# 3. Six source walkthroughs

[Guide home](README.md) · Previous: [Architecture](02-architecture.md) · Next: [Tradeoffs](04-state-and-tradeoffs.md)

Read these in order. Each starts with a user-visible behavior, then follows the
smallest useful implementation path. Function names are from the pinned upstream
revision. The abbreviated pseudocode explains control flow; it is not copied Rust
and omits error handling unless the error is the point.

## 1. Assemble a session before asking a model

**Behavior:** a session combines the selected model with the right tools,
instructions, resources, persistence, and host services.

Open [`create_session` in engine.rs][engine], then
[`wire_session_runtime` in runtime_wiring.rs][wiring]. The configuration is the
embedding contract: a daemon worker and a print invocation can provide different
child hosts, cron stores, notice callbacks, and persistence ownership without
duplicating the whole session engine.

The important assembly steps are:

1. Open settings and obtain/create a session manager. Arrange consent-gated trace
   delivery when appropriate.
2. Wire goal state and the `rlm.*` host-request registry. Cron handlers are
   registered only when the embedding supplies cron wiring. A hostless library
   embedding gets explicit no-children behavior, not imaginary working subagents.
3. Load resources, MCP gating, model facts, and skill visibility. A controller-backed
   skill is hidden when its required host handler is unavailable.
4. Create the kernel provisioner and bridge the kernel tool into the loop. The
   default product embeddings supply no other model tools, so this yields only
   `python_repl`.
5. Build the layered system prompt, initialize the agent, and install the session
   subscriptions/hooks that handle state and persistence.

The tool installation is a useful breakpoint: in the inspected source it clones
`config.tools`, checks for `PYTHON_TOOL_NAME`, constructs the IPython-named internal
tool definition, and pushes its bridge. The implementation keeps internal
`ipython` names even though the provider-facing name is `python_repl`; do not infer
an alternate product tool from the filename.

The [prompt builder][prompt] emits static `core`, `usage`, `opinionated`, and
optional model-specific layers first, then dynamic session material. The
[harness digest][digest] is delivered separately as a context message.
`prime-agent prompt --json` exposes the layer breakdown without starting a provider
session; it cannot show a real session's complete evolving history.

**Question to answer in the code:** why does `SessionEngine` strongly own the
provisioner while tool/probe paths use weak references? The source documents a
return path from kernel host handlers to session state; strong ownership in both
directions could keep a dropped session and its kernel alive.

## 2. Drive one request through the model/tool loop

**Behavior:** a prompt can generate text, execute tools, accept steering, and
continue until the loop has no more work.

Read [`run_agent_loop`][entry], [`run_loop`][loop], and
[`stream_assistant_response_inner`][response]. A reduced call tree is:

```text
run_agent_loop
  append prompts to a working context
  emit AgentStart, TurnStart, and prompt message events
  run_loop
    stream_assistant_response
      transform_context (optional)
      convert_to_llm
      call StreamFn and consume provider events
    execute_tool_calls (if requested)
    append tool results and emit TurnEnd
    evaluate stop hooks and poll input/continuation hooks
  return messages produced during this run
```

This is a provider-independent loop. The session supplies a
[`switchable_stream_fn`][adapter]; the adapter reads the selected provider target
at request time and translates loop context to provider context.
[`pa-ai::stream`][ai-stream] resolves the provider by the model's `api` identifier
and dispatches to its implementation. Provider identity, model identity, and API
transport family are related but not interchangeable names.

Input ordering is consequential. Steering is polled at turn boundaries while the
model/tool conversation is still progressing. Follow-ups are checked when that
work settles; only after no follow-up is available does the continuation hook
get an opportunity. This is why a long-running goal need not monopolize the input
queue. It does not imply arbitrary Python code is preempted immediately by typing.

The loop has both sequential and parallel tool dispatch. In
[`execute_tool_calls`][tools], **one sequential tool in a batch makes the whole
batch sequential**. The REPL tool declares that requirement because cells mutate
one namespace. In the parallel path, completion notifications can arrive in
completion order, but result messages are assembled in assistant source order.
General parallel-tool support is not evidence that two Python cells run
concurrently in a session.

**Error boundary:** a failed Rust tool call becomes an error tool result and the
model can continue; an assistant error/aborted stop ends the run's ordinary path.
A Python exception is subtler: the kernel returns structured error details, but
the current [tool bridge][tool-bridge] discards `ToolExecutionResult.is_error`, so
the outer event can incorrectly say `isError=false`. The lab's negative control
demonstrates this. Session retry/compaction policies wrap the loop rather than
making it understand every provider-specific failure. Read the executed
[`agent_loop_scripted` tests][loop-tests], especially tool-error continuation,
unknown-tool rejection, steering, sequential execution, and mid-stream abort.

## 3. Execute Python without losing control messages

**Behavior:** two cells can share `x`, yet their output and host requests remain
correlated to the correct call.

Start with [`create_ipython_tool_definition`][ipython], then follow
[`KernelProvisioner`][provisioner], [`manager/execution.rs`][execution], and the
[protocol types][kernel-protocol]. The tool ensures a kernel exists, forwards
code, maps execution output/attachments/errors into tool results, and supports
streamed progress. The manager owns subprocess I/O and active-execution state.

On the Python side, [`_compile_cell`, `_run_codes`, and `_handle_execute`][repl]
compile/execute against the persistent namespace and support top-level await.
The runtime sends structured `stdout`, `stderr`, result/error, and `done` events.
Host requests have their own IDs and futures. For example, an `rlm.spawn` call
can await a Rust reply without treating ordinary printed text as protocol input.

The interrupt path is more subtle than “cancel the future.” Rust tracks whether
the tool's wait has settled separately from whether the interpreter has actually
become idle. A force-aborted result can be returned while active execution still
needs cleanup. Reuse waits and repeated bounded interrupts prevent the next cell
from silently overlapping the old one; an uncleared kernel surfaces a busy error.
See `force_abort`, `wait_for_active_execution_to_clear_for_reuse`, and
`resolve_execution` in [execution.rs][execution].

The [lab](05-learning-lab.md#exercise-3-execute-two-real-python-cells) exercises the
real Rust-to-Python path: `x = 19; print(x * 2)` yields `38`, then `print(x + 4)`
yields `23`. Only the provider is scripted. A fixed final assistant sentence is
not the evidence; the successful `tool_execution_end` payloads are.

## 4. Spawn and collect an actual child

**Behavior:** a parent can admit named work, retain its handle, and inspect results
without confusing task admission with task completion.

Follow this cross-language reading path:

1. [`rlm.spawn` / `_spawn` in Python][python-api] validates the prompt and sends
   `host_request("rlm.run", ...)`. Its result parser requires handle fields.
2. [`register_rlm_host_handlers` and `RlmSubagentHost`][rlm-host] define the Rust
   contract. The bridge validates and normalizes input; the host owns lifecycle.
3. [`SupervisorChildSessions::spawn`][child-host] checks recursion depth, reserves
   an explicitly requested name before awaiting, resolves the model with the
   allowlist, validates thinking support, resolves `cwd`, and creates the child.
4. The child registry tracks identity and observed outcome. `collect` returns
   typed state/result envelopes; deletion and messaging are separate operations.

The name reservation is a real concurrency decision: checking a name, awaiting a
worker launch, then registering it would let simultaneous same-name admissions
both pass the first check. The reservation spans the await and is released by a
guard. The [spawn-name tests][name-tests] are a focused place to study that race.

In print mode, start instead at [`InlineSessionNode`][inline]. The implementation
uses an in-process tree, its own admission gate, a messaging controller, and a
quiescence drive. Quiescence means admitted descendants and pending receipts are
settled, not merely that the root produced a final text message.

**Tradeoff:** daemon child workers isolate crashes and permit independent
lifecycle management but add process/IPC overhead. Print children avoid that
worker topology but cannot outlive the CLI process. The source establishes the
mechanism; no comparative overhead benchmark was run for this guide.

## 5. Compact context without pretending history vanished

**Behavior:** the model can continue after its effective context grows too large,
using a summary and retained recent messages rather than replaying every old row.

Read [`compaction.rs`][compaction] for the pure threshold/cut logic,
[`prepare_compaction`][prepare] for guards and previous-summary handling, then
[`compact_session.rs`][compact] for execution. The code distinguishes a branch
that already ends in compaction from one too short to summarize. It carries
forward an earlier summary and a recent-state anchor, and handles a cut that
splits a turn instead of assuming every cut is between complete turns.

The compact executor requests model summaries, accounts their usage, commits a
compaction entry, and rebuilds model context. Abort is checked again before commit.
Kernel-requested compaction/refinement is represented in
[`TurnBoundaryRequests`][boundary] and consumed at an appropriate boundary rather
than recursively rewriting context inside the requesting Python call.

Read the existing tests in [`compact_session/tests`][compact-tests] alongside the
implementation. They cover skip guards, second compaction, cut math, and split
turns. The [print end-to-end tests][print-tests] also exercise overflow-triggered
compact/retry behavior and persistence across resume.

**Do not infer:** that the original JSONL history is deleted, that summarization
is lossless, or that Python heap size is a model-context token count.

## 6. Refine reusable state, then make it visible

**Behavior:** `/refine` can propose small changes to durable harness entries that
affect future context, without editing the base prompt.

Open [`refinement/executor.rs`][refine-executor] and
[`refinement/planner.rs`][planner]. Planning is separated from application because
a model call can take time: the executor re-reads the store before applying its
proposal. The proposal contains edits, rationale, and expected outcome. Validation
and per-edit outcomes matter; an expected outcome is not a measured result, and a
proposal should not be described as an atomic success merely because it parsed.

Then read [`harness_digest.rs`][digest] and [`refinement/ranking.rs`][ranking]. The
digest merges local/global state, ranks entries against the active goal and recent
conversation terms, and delivers a bounded view at cold-context boundaries such
as start/resume/compaction. Goal terms receive stronger weight than recent text.
This is explicit retrieval/ranking logic, not an implicit embedding database.

Rollback proposes reversals of recorded harness edits. It is not rollback of
arbitrary files, external API calls, or model training. Durable state should be
reviewed for relevance and confidentiality before choosing global scope.

[engine]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/engine.rs
[wiring]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/runtime_wiring.rs
[prompt]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/prompts/system_prompt.rs#L117-L200
[digest]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/harness_digest.rs
[entry]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-agent/src/agent_loop/entry.rs
[loop]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-agent/src/agent_loop/run.rs
[response]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-agent/src/agent_loop/response.rs#L76-L155
[adapter]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/provider_adapter.rs#L101-L180
[ai-stream]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-ai/src/stream.rs
[tools]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-agent/src/agent_loop/tools.rs
[loop-tests]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-agent/tests/agent_loop_scripted.rs
[ipython]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/tools/ipython.rs#L511-L570
[provisioner]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/kernel/provisioner.rs
[execution]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/kernel/manager/execution.rs#L19-L175
[kernel-protocol]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/kernel/protocol.rs
[repl]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/prime-agent-runtime/src/rlm/repl.py#L597-L697
[python-api]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/prime-agent-runtime/src/rlm/__init__.py#L179-L224
[rlm-host]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/rlm_host.rs
[child-host]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-daemon/src/rlm_children/host.rs#L80-L175
[name-tests]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-daemon/src/rlm_children/spawn_name_reservation_tests.rs
[inline]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-cli/src/print_rlm.rs
[compaction]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/compaction.rs
[prepare]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/compact_session/prepare.rs
[compact]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/compact_session.rs
[boundary]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/turn_boundary.rs
[compact-tests]: https://github.com/PrimeIntellect-ai/prime-agent/tree/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/compact_session/tests
[print-tests]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-cli/tests/print_runtime_e2e.rs
[refine-executor]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/refinement/executor.rs
[planner]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/refinement/planner.rs
[ranking]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/refinement/ranking.rs
[tool-bridge]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/tool_bridge.rs#L112-L125
