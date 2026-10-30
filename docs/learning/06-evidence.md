# 6. Evidence, uncertainty, and delivery

[Guide home](README.md) · Previous: [Learning lab](05-learning-lab.md)

## Baseline and method

This guide inspects upstream Rust `main` at
[`c6d4da9421e6087e3fc1a763e947cf37153cd76f`](https://github.com/PrimeIntellect-ai/prime-agent/commit/c6d4da9421e6087e3fc1a763e947cf37153cd76f),
workspace version `0.10.2`, on 2026-10-30. The checkout is not shallow. No old
TypeScript checkout was used as the implementation source of truth.

The investigation followed concrete source paths across composition, agent loop,
provider adaptation, kernel execution, daemon workers, session storage, prompts,
compaction, goals, scheduling, refinement, skills, MCP, factory execution, and
telemetry. Manifests establish crate dependencies. Existing tests help describe
contracts; only the explicitly listed commands below were executed. Source links
pin the baseline rather than mutable `main`.

## Corrections that matter when reading older prose

| Claim to avoid | Evidence at this revision | Consequence |
| --- | --- | --- |
| The model gets three default tools: bash, edit, and Python | [Print defaults][print-tools] are empty before [engine tool installation][engine-tools]; [daemon construction][daemon-tools] also supplies no extra tools | Default provider surface is `python_repl`; shell/edit are programmatic capabilities inside it |
| The Rust runtime is unavailable because `mode.rs` still describes merge-time placeholders | [CLI composition][cli] dispatches to linked interactive, print, RPC, ACP, and daemon implementations; the binary was built and executed | Comments about placeholder types are not proof the product still uses those types |
| Trace upload is unported | Some [CLI prose][cli-readme] says that, but [continuous delivery][trace] exists and [engine assembly][engine] wires it | Distinguish sanitized analytics from opt-in session-content delivery; neither should be inferred from a stale README |
| Every child agent is a separate worker process | [Daemon children][children] use supervised workers; [print children][inline] use inline session nodes | Lifetime and fault isolation depend on execution mode |
| Session history is never rewritten | [Persistence][persist] includes append and rewrite paths; [daemon writing][daemon-write] includes atomic rewriting | “Append-oriented JSONL tree” is more accurate than universally immutable log |
| `/refine` trains the model | [Planner][planner] proposes structured harness edits and [executor][refine-executor] applies them | External memory/instruction adaptation, not weight training or measured improvement |
| Factory resume survives interpreter death | [`FactoryExecutor`][factory] keeps its run registry in memory and documents restart limits | Durable definitions and supervised children do not make the coordinator durable |
| `--offline` prohibits all network I/O | [CLI help source][help] says startup operations; the real kernel lab bootstrapped dependencies | Use an external network restriction when that is the actual requirement |

These are observations about this source snapshot, not allegations that every
upstream document is stale. The top-level README already describes several of
these boundaries accurately. Existing attribution, research citation, and license
were preserved rather than replaced.

## Executed checks

Environment: Linux x86-64 orb, Rust/Cargo 1.98.1, Python 3.11, `uv`; Rust commands
used `CARGO_BUILD_JOBS=4 CARGO_PROFILE_DEV_DEBUG=0`. Commands are relative to the
repository root unless otherwise stated.

| Command or scenario | Observed result | What it does not establish |
| --- | --- | --- |
| `cargo build --locked -p pa-cli --bin prime-agent` | Build succeeded; vendored `crossterm` warnings appeared | Release packaging, full clippy gate, all platforms |
| `cargo test --locked -p pa-agent --lib` | 12 passed, 0 failed | Every session policy or provider |
| `cargo test --locked -p pa-agent --test agent_loop_scripted` | 14 passed, 0 failed | Real provider availability or output quality |
| `cargo test --locked -p pa-cli --test print_runtime_e2e` | 24 passed, 0 failed | Full interactive/daemon UI behavior |
| Isolated `prime-agent prompt --json` | Static/dynamic segments; cached prefix length 24,013 bytes | Provider cache-hit rate or cost saving |
| Faux text response through print mode | `Learning harness OK`, successful exit | Reasoning ability; response was supplied by the script |
| Faux tool requests through a real Python kernel | First stdout `38`, then `23`; both nested status `ok` and `isError=false` | Namespace survival across interpreter/process death |
| Negative control: undefined Python variable | Nested status `error` and `NameError`; the output assertion fails, although outer `isError=false` | An outer success flag is not sufficient evidence of cell success |

See [the lab](05-learning-lab.md) for runnable commands and exact output assertions.
These exercises use no paid provider requests. They intentionally run the actual
kernel rather than replacing `python_repl` with a mock.

The negative control exposed a concrete boundary defect: [the kernel tool][kernel-result]
sets `is_error` for Python errors, but [the bridge][tool-bridge] reads and discards
it. The loop sees a successful Rust `Result` and emits a false outer error flag.
Structured traceback/error details still reach the result. This behavior is
documented rather than repaired, because the requested artifacts explain the
inspected implementation; they do not introduce unrelated runtime changes.

### Python runtime results

From `prime-agent-runtime/`:

```bash
uv run --locked --python 3.11 python -m unittest discover -s test
```

**754 tests ran; two failed**, both in `test_repl.RestoreApplyShieldTest`:
`test_handler_restored_and_no_interrupt_bleed` and
`test_sigint_during_staging_leaves_namespace_unchanged`. They expected
`KeyboardInterrupt`; the process inherited `SIGINT=SIG_IGN` in this orb. Inspecting
`signal.getsignal(signal.SIGINT)` returned `1`, confirming that disposition.

To distinguish an inherited signal environment from runtime behavior, a second
full run explicitly restored Python's normal interrupt handler before discovery:

```bash
uv run --locked --python 3.11 python -c '
import signal, unittest
signal.signal(signal.SIGINT, signal.default_int_handler)
unittest.main(module=None, argv=["unittest", "discover", "-s", "test"])
'
```

That run also executed **754 tests**. The two interrupt failures disappeared,
but `test_bash.BashTest.test_journal_partial_writes_complete_the_record` errored
with `JSONDecodeError: Extra data`. The journal test plus the restore-shield test
class then passed in isolation (**8 tests**). This is evidence of an intermittent
or test-order-sensitive result, **not a fully passing Python suite or a confirmed
root cause**. The runs also emitted background watcher exceptions and an unclosed
file warning; those diagnostics were not repaired in this documentation task.

No runtime source or tests were changed to produce a green report. No full Rust
workspace test run, complete merge-gate run, TypeScript parity run, live-provider
test, UI visual check, security audit, or performance benchmark is claimed.

## What remains uncertain

- Real models' planning quality, recursion usefulness, refinement quality, and
  long-task success rates. Implementation presence is not evaluation evidence.
- Behavior across all provider API variants, auth flows, MCP servers, and operating
  systems. The adapter and configuration paths were read, not all exercised.
- End-to-end restart/recovery under kill signals, host failure, and concurrent
  external effects. Filesystem recovery code cannot prove exactly-once effects.
- Latency and memory under many workers or large transcripts. Architectural
  choices and performance tests in the tree are not measurements from this run.
- The root cause of the Python journal test's full-suite error.

## Repository delivery

The publication destination is the
[`docs/rust-learning-guide` branch of `chimera-research/prime-agent`](https://github.com/chimera-research/prime-agent/tree/docs/rust-learning-guide/docs/learning).
GitHub metadata confirmed that this is a public fork whose parent and source are
both `PrimeIntellect-ai/prime-agent`. Initial fork creation had failed with HTTP
403; publication resumed after the user created the organizational fork.

Before publication, the fork had only `main`, at the exact inspected revision
`c6d4da9421e6087e3fc1a763e947cf37153cd76f`, and no open pull requests. A fresh fetch
of upstream `main` returned the same revision. There was no revision drift or
fork-specific work to reconcile, and the source links remain pinned to that
baseline rather than being silently advanced.

The publication adds only this guide and a root README link, on a new branch.
The fork's `main` remains untouched: its existing push workflows include automatic
nightly release machinery, and the integration could not inspect the fork's
Actions configuration. No workflow settings were changed or workflows manually
triggered. No personal fork, force push, upstream push, or pull request was used.

The unchanged `LICENSE` retains Prime Intellect Ltd's copyright and the original
TypeScript attribution. Runtime code, lockfiles, and CI workflows are unchanged.
The checks above were performed on the same source revision; they are not new
test runs merely because the documentation has now been published.

## Updating this guide responsibly

1. Record the new full upstream commit before editing claims or links.
2. Re-read the owning implementation and its composition. A leaf helper's presence
   alone does not establish that the shipped modes use it.
3. Re-run the synthetic lab and relevant existing tests; record failures too.
4. Recheck provider-visible tools, child hosting, persistence, trace consent, and
   factory durability. These are easy places for old prose to become misleading.
5. Update evidence labels and pinned references together. Do not silently turn a
   previous inference into an upstream guarantee or a previous test into a current
   passing result.

[print-tools]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-cli/src/print_runtime.rs#L1490-L1494
[engine-tools]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/engine.rs#L520-L570
[daemon-tools]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-daemon/src/agent_engine/lifecycle.rs#L1157-L1185
[cli]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-cli/src/lib.rs
[cli-readme]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-cli/README.md#L6
[trace]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/agent_traces/continuous.rs
[engine]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/engine.rs
[children]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-daemon/src/rlm_children/host.rs
[inline]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-cli/src/print_rlm.rs
[persist]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session/manager/persist.rs
[daemon-write]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-daemon/src/session_store/write.rs
[planner]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/refinement/planner.rs
[refine-executor]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/refinement/executor.rs
[factory]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/prime-agent-runtime/src/rlm/factory.py#L1655-L1692
[help]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-cli/src/command_registry.rs#L320-L340
[kernel-result]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/tools/ipython.rs#L502-L506
[tool-bridge]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/tool_bridge.rs#L112-L125
