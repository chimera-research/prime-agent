# Prime Agent, from the running system to the source

This is a source-grounded learning guide prepared for Chimera Research. It explains
the **current Rust application and the Python runtime it actually executes**, not
the retired TypeScript application. It is independent commentary, not an upstream
design specification or an endorsement by the upstream authors.

**Source baseline:** `PrimeIntellect-ai/prime-agent`, branch `main`, commit
[`c6d4da9421e6087e3fc1a763e947cf37153cd76f`](https://github.com/PrimeIntellect-ai/prime-agent/commit/c6d4da9421e6087e3fc1a763e947cf37153cd76f),
inspected on **2026-10-30**. The workspace version is `0.10.2`. Source links in this
guide pin that commit: a later upstream update does not silently change the
evidence. “Current” throughout means this snapshot, not an indefinitely current
claim about `main`.

## The system in one paragraph

Prime Agent is a coding/research agent that lets a language model write Python in a
persistent execution environment. Python can inspect files, run shell commands,
call integrations, and ask the Rust host to create other agents. Rust owns model
streaming, the turn loop, session policy, persistence, and process supervision.
The terminal UI attaches to that machinery rather than owning its lifetime.
Reusable instructions and memories live in an editable **continual harness**;
changing that harness is not training the underlying model.

## Choose a route

| Your question | Read |
| --- | --- |
| What can I do with it, and what do its terms mean? | [1. Functionality and conceptual model](01-system-model.md) |
| Which processes and crates own which responsibilities? | [2. Architecture and execution modes](02-architecture.md) |
| How does an actual prompt become code, subagents, and a response? | [3. Six source walkthroughs](03-source-walkthroughs.md) |
| Why these implementation choices, and what are the limits? | [4. State, tradeoffs, and trust](04-state-and-tradeoffs.md) |
| How can I learn by running it without buying model calls? | [5. Hands-on learning lab](05-learning-lab.md) |
| What was verified, what contradicted the docs, and what remains uncertain? | [6. Evidence and delivery record](06-evidence.md) |

For a first pass, read chapters 1 and 2, then run the first two lab exercises.
For implementation work, follow chapter 3 with your editor open, and use chapter 4
to avoid confusing conversation context, Python memory, and durable state.

## How to read the claims

- **Implemented:** the linked source contains the mechanism and its wiring.
  This does not mean every operating system or provider was exercised here.
- **Observed:** an executed check in [the evidence record](06-evidence.md) produced
  the stated result.
- **Inference:** an interpretation of a design choice, explicitly identified as
  such; it is not a benchmark or an upstream promise.
- **Limitation:** a boundary in the implementation or in this investigation.

The most important correction to a casual reading is that the shipped default
provider tool list is **`python_repl` only**. `bash` and `edit` are callable inside
Python, not separate default provider tools. Another important distinction is
that daemon-backed children get separate worker processes, whereas print-mode
children are hosted in the CLI process. The chapters show both implementations.

## Attribution and safe use

Prime Agent is upstream work from
[PrimeIntellect-ai/prime-agent](https://github.com/PrimeIntellect-ai/prime-agent).
The original [README and research citation](../../README.md#citation) and
[MIT license](../../LICENSE) remain intact. The license includes the upstream
copyright and the attribution to the earlier TypeScript product and Mario
Zechner. These learning documents are supplied under that same repository
license; they do not replace or rebrand the upstream implementation.

**Workers and Python kernels are not security sandboxes.** Run experiments in a
disposable checkout with restricted credentials. Treat Python skills, project
instructions, MCP servers, shell gates, and restored serialized state as trusted
code/data. Read the [trust boundaries](04-state-and-tradeoffs.md#trust-and-data-leaving-the-machine)
before using private repositories or live integrations.

The guide's publication home is the
[Chimera Research fork, `docs/rust-learning-guide` branch](https://github.com/chimera-research/prime-agent/tree/docs/rust-learning-guide/docs/learning).
The [delivery record](06-evidence.md#repository-delivery) identifies the verified
fork, unchanged source baseline, and documentation-only publication scope.
