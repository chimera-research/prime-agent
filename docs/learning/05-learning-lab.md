# 5. Hands-on learning lab

[Guide home](README.md) · Previous: [Tradeoffs](04-state-and-tradeoffs.md) · Next: [Evidence](06-evidence.md)

These exercises use the **real Rust binary and real Python kernel**, with a
scripted provider instead of paid inference. They teach the execution machinery,
not how well an actual language model solves tasks. Start in a disposable checkout
with no sensitive files or credentials. A temporary `HOME` avoids ordinary user
settings; it is not a security sandbox or an environment-secret scrubber.

## Build the pinned source

Use a Linux development environment with Git, a C compiler/linker, Rust via
`rustup`, `uv`, and `jq`. The checked runtime requires Python 3.11 or newer.
The build was exercised on Linux x86-64 with Rust **1.98.1**, the toolchain pinned
by [upstream CI][ci]. Other platforms were not verified here. Cargo and Python
bootstrap may download dependencies; budget disk space and several minutes for
the first Rust build.

If you do not already have a checkout, clone upstream into a new directory:

```bash
git clone https://github.com/PrimeIntellect-ai/prime-agent.git prime-agent-study
cd prime-agent-study
git switch --detach c6d4da9421e6087e3fc1a763e947cf37153cd76f
```

Do not switch an existing working tree over uncommitted work. These commands
reproduce the inspected upstream source; the learning documents themselves live
on the [Chimera Research documentation branch](https://github.com/chimera-research/prime-agent/tree/docs/rust-learning-guide/docs/learning).
From the pinned checkout's root:

```bash
rustup toolchain install 1.98.1 --profile minimal
CARGO_BUILD_JOBS=4 CARGO_PROFILE_DEV_DEBUG=0 \
  cargo +1.98.1 build --locked -p pa-cli --bin prime-agent
export REPO="$PWD"
export BIN="$REPO/target/debug/prime-agent"
"$BIN" --version
```

Expected binary version: `0.10.2`. Four jobs and reduced debug info keep this
debug build manageable on a small development machine; they are not release
optimization settings. Keep `prime-agent-runtime/` alongside the source binary:
the Rust host needs those actual Python sources.

## Isolate the exercises

Use Bash for the remaining blocks, in the same shell. This helper changes only
each child command's environment, not your shell's `HOME` or Rust installation:

```bash
export LAB="$(mktemp -d "${TMPDIR:-/tmp}/prime-agent-study.XXXXXX")"
pa_lab() {
  env -u RLM_DEPTH -u PRIME_AGENT_KERNEL_PYTHON \
    -u PRIME_AGENT_SESSION_DIR -u PRIME_AGENT_CODING_AGENT_SESSION_DIR \
    -u PRIME_AGENT_FAUX_CHILD_SCRIPT \
    HOME="$LAB" PRIME_AGENT_CODING_AGENT_DIR="$LAB/agent" \
    PRIME_AGENT_KERNEL_VENV="$LAB/kernel-venv" PI_PACKAGE_DIR="$REPO" \
    DO_NOT_TRACK=1 \
    "$BIN" --cwd "$LAB" "$@"
}
printf 'Lab directory: %s\n' "$LAB"
```

Fresh lab settings leave content trace sharing off by default. `DO_NOT_TRACK`
disables adoption telemetry; do not treat it as a universal data-loss-prevention
switch. `--offline` below disables startup network operations, **not all network
access**: first kernel setup can still fetch Python dependencies. Run in an
externally restricted environment if you need a network prohibition.

## Exercise 1: inspect instructions before inference

```bash
pa_lab prompt --json > "$LAB/prompt.json"
jq '{cachedPrefixLen, segments}' "$LAB/prompt.json"
```

Identify static `core`, `usage`, and `opinionated` segments, followed by dynamic
segments such as packages, skill inventory, environment, and session role.
The inspected baseline reported `cachedPrefixLen: 24013`; do not hard-code this
as an invariant across models, configuration, or revisions. This command inspects
the assembled prompt, not a provider's actual cache. Compare it with
[`system_prompt_breakdown`][prompt-source].

## Exercise 2: drive the real loop with a fixed response

```bash
PRIME_AGENT_FAUX_SCRIPT='{"responses":["Learning harness OK"]}' \
  pa_lab --offline -p 'hello'
```

Expected stdout: `Learning harness OK`. The special environment variable is an
existing [verification seam][faux-source], not a production provider credential.
It queues responses for a faux model. CLI parsing, session construction, the loop,
and final output selection are real; the “model answer” is deliberately supplied
by the test. An empty response queue is an error, not an invitation to silently
use a paid provider.

## Exercise 3: execute two real Python cells

```bash
PRIME_AGENT_FAUX_SCRIPT='{"responses":[
  {"content":[{"type":"toolCall","id":"cell1","name":"python_repl",
    "arguments":{"code":"x = 19\nprint(x * 2)"}}]},
  {"content":[{"type":"toolCall","id":"cell2","name":"python_repl",
    "arguments":{"code":"print(x + 4)"}}]},
  "Kernel exercise finished"
]}' pa_lab --offline --mode json -p 'exercise the persistent kernel' \
  > "$LAB/kernel-events.jsonl"

jq -c 'select(.type == "tool_execution_end") |
  {toolCallId, isError, status: .result.details.status, stdout: .result.details.stdout}' \
  "$LAB/kernel-events.jsonl"

jq -s -e '
  [.[] | select(.type == "tool_execution_end") |
    {toolCallId, isError, status: .result.details.status, stdout: .result.details.stdout}] == [
      {"toolCallId":"cell1", "isError":false, "status":"ok", "stdout":"38\n"},
      {"toolCallId":"cell2", "isError":false, "status":"ok", "stdout":"23\n"}
    ]' "$LAB/kernel-events.jsonl"
```

The last command must print `true` and exit zero. This checks **results**, not just
process exit: a scripted final response could still appear after a tool error.
`23` establishes that the second call sees the first call's namespace. The IDs
establish which execution produced which result. First run also exercises kernel
bootstrap; subsequent runs can reuse its environment.

For a useful negative control, change the second cell to `print(missing_name + 4)`.
The observed `result.details.status` becomes `error`, with a `NameError` traceback,
and the equality check fails. **At this revision the outer `isError` remains
false:** the [tool bridge][tool-bridge] discards the kernel tool's error flag.
The final scripted assistant sentence can remain unchanged. Inspect the nested
status and actual output, not just process exit or the outer error flag. This
negative control was executed; the runtime defect was documented, not fixed here.

## Exercise 4: read the stored trajectory

```bash
rg --files "$LAB/agent" -g '*.jsonl'
jq -r 'select(.type == "message_end") |
  [.message.role, (.message.toolName // "")] | @tsv' \
  "$LAB/kernel-events.jsonl"
```

The event stream shows user/assistant/tool-result boundaries; the session JSONL
files contain persisted entries, which are a different representation. Open one
and locate its header, entry IDs, parent links, assistant calls, and tool results.
Do not concatenate every historical row and call it the provider's current input:
branch selection, control records, compaction, and conversion all matter.

Avoid publishing these files from real work. They can contain prompts, code,
outputs, and local paths. The supplied learning artifacts contain conclusions and
small synthetic observations, not exported personal sessions.

## Exercise 5: use existing tests as executable reading material

From the repository root, with the ordinary shell environment:

```bash
export CARGO_BUILD_JOBS=4 CARGO_PROFILE_DEV_DEBUG=0
cargo +1.98.1 test --locked -p pa-agent --lib
cargo +1.98.1 test --locked -p pa-agent --test agent_loop_scripted
cargo +1.98.1 test --locked -p pa-cli --test print_runtime_e2e
```

Observed results: **12, 14, and 24 tests passed**, respectively. Read the
[scripted-loop tests][loop-tests] for tool errors, ordering and abort, and the
[print tests][print-tests] for stored sessions, resume, compaction, and exit status.
These counts belong to the pinned source; new revisions can change them.

The Python suite is also valuable, but it was **not fully green** in this orb:

```bash
cd prime-agent-runtime
uv run --locked --python 3.11 python -m unittest discover -s test
```

See the [evidence record](06-evidence.md#python-runtime-results) for the inherited
signal issue, the normalized rerun, and a separate journal-test error. Do not
ignore those failures or describe this as a complete regression certification.

## Move to a real provider deliberately

This part is a suggested next exercise, **not executed in this investigation**.
Use a restricted disposable repository, then start `prime-agent` from an actual
terminal. `/login` opens provider setup; `/model` selects a model. Follow the
selected provider's current supported flow. Do not paste keys into shell command
arguments, transcripts, or repository files.

Start with a narrow read-only task: “Explain this small function and cite its
source; do not modify files.” Inspect the resulting Python cells and compare their
outputs with the final answer. Next, request two named, non-overlapping child
investigations, and inspect admission, messages, collection, and settlement.
Use `/goal`, `/autonomous`, `/refine`, schedules, or external integrations only
after understanding their budget, persistence, and trust boundaries.

Use `prime-agent status` and `prime-agent agents` to inspect background sessions.
`prime-agent shutdown` stops **all** agents/services for that installation, not
just your experiment; do not run it casually against a shared development setup.
The lab above uses print mode and does not need a resident daemon left running.

## Pick the next code-reading exercise

| Want to learn | Follow | Check your explanation against |
| --- | --- | --- |
| Provider adapters | `provider_adapter.rs`, `pa-ai/src/stream.rs`, one provider module | Context conversion, cancellation, and provider error mapping |
| Recursion | `rlm.__init__`, `rlm_host.rs`, `rlm_children/host.rs` | Admission vs completion; name/depth guards; process ownership |
| Continuity | `session/`, `compact_session/`, `kernel/state_snapshot.rs` | What survives process exit, and what is only reconstructed |
| Continual learning | `refinement/`, `harness_digest.rs` | Proposed edits vs applied edits vs measured improvement |
| UI ownership | `pa-tui/src/daemon_client/`, `app.rs`, `view/` | Which event updates visible state without importing `pa-core` |

When finished, retain the lab directory only if you want to inspect the synthetic
state; it is disposable. Keep its environment, caches, and JSONL files out of the
repository. Use the [evidence index](06-evidence.md) when updating this guide to a
new upstream revision.

[ci]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/.github/workflows/ci.yml
[prompt-source]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/prompts/system_prompt.rs
[faux-source]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-ai/src/providers/faux/script.rs
[loop-tests]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-agent/tests/agent_loop_scripted.rs
[print-tests]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-cli/tests/print_runtime_e2e.rs
[tool-bridge]: https://github.com/PrimeIntellect-ai/prime-agent/blob/c6d4da9421e6087e3fc1a763e947cf37153cd76f/crates/pa-core/src/session_engine/tool_bridge.rs#L112-L125
