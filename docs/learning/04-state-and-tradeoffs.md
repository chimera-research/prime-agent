# 4. State, tradeoffs, and trust

[Guide home](README.md) · Previous: [Walkthroughs](03-source-walkthroughs.md) · Next: [Learning lab](05-learning-lab.md)

## Durability is several mechanisms, not one guarantee

| State | Mechanism | Important limitation |
| --- | --- | --- |
| Conversation and branch state | JSONL entries, IDs, parent links, append paths and atomic rewrites | It is not literally an immutable append-only file under every operation |
| Pending daemon work | Recovery journals, descriptors, leases and admission bookkeeping | Recovery is not proof an external side effect happened exactly once |
| Python variables | Best-effort per-variable snapshots and restore | Unserializable/oversized objects can be skipped; not a process image |
| Harness | Local/global structured entries and refinement history | Incorrect lessons can be durable too |
| Scheduled jobs | Durable store plus runtime scheduler/dispatch lanes | A stopped machine cannot execute the timer |
| Factory definitions | Harness specs or machine-library files | Definition persistence is not live run persistence |
| Factory runs | Python executor's in-memory run registry | Kernel restart loses the run coordinator; daemon-owned children can remain |

Read [session persistence][persist], [daemon writes][daemon-write],
[recovery journals][journal], [snapshot shapes][snapshot], and
[`FactoryExecutor._runs`][factory]. The distinction between stored history and
active model context explains how a compacted or branched session can preserve
old records without sending all of them to the provider.

The snapshot defaults are **256 MiB overall and 16 MiB per variable**. Results
explicitly report saved, skipped, failed, and optionally pruned names. Snapshot
files use `dill`; do not restore untrusted files. A file handle, active network
connection, subprocess, or arbitrary Python object should not be assumed to
resume with its original behavior merely because some namespace values restore.

**Inference:** JSONL plus filesystem artifacts makes incidents inspectable with
ordinary tools and supports compatibility with existing sessions. The cost is
substantial coordination code: leases, tail repair, indexes/windows, checkpoints,
and careful distinction between full-history operations and windowed reads.

## Prompt caching is a construction rule, not a cache-hit promise

[`system_prompt_breakdown`][prompt] emits stable static text first and dynamic
session context last. It exposes `cached_prefix_len` as a **byte length**, while
individual diagnostic segment counts are characters. Per-model additions and
custom static prompts change that prefix; different sessions do not necessarily
share one prefix universally.

The harness digest is a separate user-context message rather than a mutation of
the core static layer. **Inference:** this reduces unnecessary prefix churn and
makes durable lessons separable from product policy. Actual provider cache hits,
billing, retention, and speedups depend on provider behavior and were not measured.

## Compaction is constrained by both retention and provider capacity

The default requested threshold is 95%, but
[`compaction_threshold`][compaction] does not simply multiply the window by 0.95.
In pseudocode, using saturating subtraction for the capacity calculation:

```text
requested = floor(context_window * threshold_percent / 100)
retention_floor = keep_recent_tokens + 2 * reserve_tokens + 1
headroom = max(reserve_tokens, 4096)
capacity = saturating_sub(context_window, max_output_tokens, headroom)
threshold = min(max(requested, retention_floor), capacity)
```

The retention floor prevents repeated compaction of essentially the same retained
tail. The capacity cap takes priority, even over that floor, because providers
can reject input plus requested output above their context ceiling. Defaults are
20,000 retained tokens and 16,384 reserve tokens.

For a 100,000-token window, 4,096 requested output tokens, and default settings:
the percentage is 95,000, the retention floor is 52,769, and capacity is 79,520;
the effective threshold is **79,520**, not 95,000. This calculation is derived
from the source formula, not a live provider measurement.

The provider [streaming facade][stream] separately clamps requested output to the
room left by an input estimate. It leaves an input-already-full request to the
overflow recovery path. Text/image heuristics are not a tokenizer, and summary
quality is not guaranteed. Safety margins reduce avoidable errors; they do not
prove arbitrary requests fit every provider.

## Responsiveness depends on bounded work on the visible path

The [catalog chain][catalog] resolves from disk cache, bundled snapshot, or
compiled fallback; refresh is a separate concern. Registry/catalog availability
does not imply the user has valid credentials or that an inference endpoint works.

The [stream coalescer][coalescer] parks a latest assistant snapshot and merges
additive deltas, with a 50 ms flush interval. Non-update events flush pending
updates first. This reduces broadcast/serialization pressure without making text
disappear just because an intermediate snapshot was superseded. **Tradeoff:**
roughly bounded batching delay versus per-token IPC/render overhead, not a claim
that every paint completes in 50 ms.

The [TUI frame code][tui-frame] renders from event-derived state and cached visible
rows; it does not execute the session engine. The [telemetry client][telemetry]
hands work to a background queue, sanitizes events, and batches delivery. Its
front-door channel is unbounded even though individual sink queues have caps:
“nonblocking” and “globally bounded memory” are different properties.

**Inference:** the architectural direction and these implementation choices
prioritize time to visible feedback, but they are not substitutes for latency
measurements under load. This guide did not run the `pa-perf` benchmarks.

## Continuation, retries, and success are different policies

The [autonomous gate driver][gates] checks configured shell commands and budget
limits. A prior failing gate is not rerun over an unchanged captured Git worktree;
failure information instead informs the next continuation. This can save repeated
work, but a Git worktree snapshot cannot represent every external dependency of a
shell command. Design gates around the property you genuinely need to verify.

The [goal driver][goal] persists objective state and tracks progress/budgets.
Its time-used calculation is age since creation, not exclusively CPU time or
time in active status. Goal continuation and generic autonomous continuation
should not be read as the same state machine.

Provider failure recovery has separate quick retry, failover, and parking logic.
The [failover driver][failover] distinguishes permanent/lifecycle failures and
context overflow from retryable failures and has a whole-episode retry ceiling.
The [cron scheduler][cron] serializes dispatch per session and backs off repeated
failures; its failure streaks are in memory. None of these mechanisms makes an
arbitrary file write, purchase, or external API mutation transactional or
exactly-once. Use idempotent operations and explicit checks where side effects
matter.

## Trust and data leaving the machine

1. **Execution permissions:** model-generated Python, shell commands, Python
   skills, and quality gates run as the invoking user. Process isolation is
   lifecycle management, not a sandbox. Restrict the host environment yourself.
2. **Model requests:** system/project instructions, selected conversation content,
   tool output, and attached images can reach the selected provider. Not printing
   a large object can reduce exposure but is not a general confidentiality policy.
3. **MCP:** the host resolves configuration/auth, while the kernel opens configured
   servers and sends tool arguments. Tool schemas/results are server-controlled.
   The [URL checker][url-checks] explicitly performs structural checks, not complete
   DNS/redirect/rebinding enforcement. Do not interpret it as a network sandbox.
4. **Two telemetry surfaces:** [adoption telemetry][telemetry] is sanitized event
   metadata. [Trace uploading][traces] is separate, consent-gated session-content
   delivery. `DO_NOT_TRACK=1` is useful for the learning runs but must not be
   mistaken for proof that every content-sharing feature is disabled. Review
   `agentTraces` settings and `/traces` separately.
5. **Configuration and artifacts:** normal settings include global
   `~/.prime/agent/settings.json` and project `.prime/agent/settings.json`.
   [Deep merge][merge] recursively merges objects but replaces arrays/scalars;
   null is treated as unset. Do not assume project settings can safely override
   every security-sensitive global service setting: MCP execution wiring reads
   user/global configuration.

Auth files, transcripts, snapshots, and local harness memories can all be
sensitive. Avoid committing them while experimenting. Restoring a session or
installing a skill is not equivalent to reading inert Markdown.

## Factories are useful, but not a general durable workflow service

The current [factory implementation][factory] validates bounded state machines,
supports guarded transitions and joins, and compiles the older DAG form into
machine form. It admits children through `rlm.spawn`, observes them through
`rlm.collect`, and has pause/resume and failure policies. The namespace is opt-in
through `factory.enabled` (default off).

`resume(run_id)` resumes a paused run in the existing executor. It does **not**
rehydrate that executor after kernel death. Definitions may persist while running
coordination does not. States with unsupported wait blocks are rejected during
validation; do not confuse `rlm.factory.watch` (observing factory changes) with a
general `rlm.watch.*` communication system. These boundaries matter before using
factories for work that must survive machine or interpreter failure.

[persist]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session/manager/persist.rs
[daemon-write]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-daemon/src/session_store/write.rs
[journal]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-daemon/src/journal.rs
[snapshot]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/kernel/state_snapshot.rs
[factory]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/prime-agent-runtime/src/rlm/factory.py
[prompt]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/prompts/system_prompt.rs
[compaction]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/compaction.rs#L158-L184
[stream]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-ai/src/stream.rs#L85-L139
[catalog]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-models/src/chain.rs
[coalescer]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-daemon/src/streaming.rs#L1-L160
[tui-frame]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-tui/src/view/frame.rs
[telemetry]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-telemetry/src/client.rs
[gates]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/autonomous/gates.rs
[goal]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/goal_driver.rs
[failover]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/provider_failover.rs
[cron]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/cron/scheduler.rs
[url-checks]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/mcp/url_checks.rs
[traces]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/agent_traces/continuous.rs
[merge]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/settings/merge.rs
