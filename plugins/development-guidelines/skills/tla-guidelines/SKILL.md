---
name: tla-guidelines
description: TLA+ and TLC rules for modelling real code (Go services in particular) - model-first defect workflow (Bug and Fixed configs before any code or test), modelling goroutines, channels (capacity, blocking, close, select with default), mutexes, wait groups, timers and context cancellation as well as state transitions, deadlock and liveness checking without masks, bounded queues, refinement and interface specs for composition, TLC performance (workers, symmetry soundness, split safety and liveness configs, atomicity, construct-don't-filter, views), running TLC safely (heap caps, one run at a time), verification discipline (every Bug config fails on its named property, verdicts unchanged after optimisation) and keeping models traceable to code (pinned commits, drift tables). Use before writing, extending, optimising or reviewing any .tla spec or .cfg file, together with core-guidelines.
---

# TLA+ guidelines

TLA+ applications of `core-guidelines`, plus rules that exist only for specifications. Every rule has a Do and Don't example in [references/examples.md](references/examples.md); read the section that matches the work in hand. Examples model Go; the same shapes apply to any language with threads and channels.

A model earns its keep by finding the interleaving a human would not. Everything below serves that: model what can actually interleave, check it without hiding results, keep it fast enough to run, and keep it honest about which code it describes.

## Model-first workflow ([examples](references/examples.md#model-first))

- A suspected defect is reproduced in the model before any test or fix is written: a `<Name>Bug` config that models the code as it is and fails, and a `<Name>Fixed` config that models the proposed fix and passes.
- Then a test in the real code that fails for the same reason, then the fix. The model tells you what the test must reproduce.
- Stress loops and reruns confirm what the model found; they never replace it. A flake that does not reproduce in isolation is a cue to model, not to rerun.
- Every review finding on the fix goes back into the model (a new Bug config the finding fails), so the model never trails the code.
- Model the test harness too when the defect is in a test: what the test waits for, and what that wait actually guarantees. "The gate is free" can mean started *or* rolled back.

## Traceability ([examples](references/examples.md#traceability))

- A spec names the code commit it models and, for every action, the file and line it stands for.
- Where the model deliberately differs from the code or from a reference implementation, record it in a drift table: what the code does, what the model does, why, and which config shows it.
- Models live in their own repository. Product code, test names, test file names and commit messages never mention the model, its configs or TLA+.
- Re-check a model against the code it claims to model whenever that code moves; a model of an older commit hides the goroutines added since.

## Concurrency, not just state ([examples](references/examples.md#goroutines))

- Each goroutine (`go` statement, worker pool member, timer callback) is its own process, interleaving with the others. One action that does what several goroutines do hides every bug between them.
- A channel is a sequence with its real capacity and a `closed` flag ([examples](references/examples.md#channels)):
    - send is enabled only when the channel is not full; an unbuffered channel is a rendezvous;
    - receive is enabled only when it is non-empty or closed; a receive on a closed, empty channel yields the zero value;
    - send on a closed channel and close of a closed channel are panics: model them as reachable error states and check an invariant that they never happen.
- `select` is a nondeterministic choice among its enabled cases. A `default` branch is enabled exactly when no other case is; model it, because that is where signals are dropped ([examples](references/examples.md#select-default)).
- A channel closed and replaced to wake waiters ("broadcast") is a generation counter; a waiter that read the old generation and the closer are separate steps, so lost wakeups are reachable ([examples](references/examples.md#broadcast)).
- A mutex is an explicit holder. Split the critical section into steps only where something inside it can block (a channel send, a wait, a call that takes another lock); keep it one step otherwise ([examples](references/examples.md#locks)).
- A wait group is a counter with `Add`, `Done` and `Wait` enabled at zero. Joins matter: model goroutine exit, and check that nothing outlives the owner that should have joined it.
- Timers and timeouts fire nondeterministically once armed; don't model wall-clock time unless an ordering of deadlines is the bug.
- Context cancellation is a flag every blocked step can observe; a step that cannot observe it is a step that can block forever.
- Bound every queue as the code bounds it. An unbounded model queue hides the deadlock where a lock is held across a send to a full queue ([examples](references/examples.md#bounded-queues)).
- Use one shared module for the language's primitives (channel, mutex, wait group, once, timer) so every spec names them the same way and specs can be composed.

## Deadlock and liveness ([examples](references/examples.md#deadlock))

- Keep TLC's deadlock check on. A `Next` with an unconditional stuttering disjunct, or `CHECK_DEADLOCK FALSE`, hides every blocked caller.
- When a config must turn the deadlock check off (a modelled terminal state, or deadlocks known and reported elsewhere), the cfg says why, and a liveness property ("every call eventually returns") covers what the deadlock check would have.
- Liveness needs fairness: weak fairness on each process that must make progress, strong fairness only where an action is enabled intermittently. A liveness property without fairness fails for a trivial stuttering reason.
- Safety (invariants, action properties) and liveness (temporal properties) live in separate configs ([examples](references/examples.md#split-configs)).

## Composition ([examples](references/examples.md#refinement))

- Compose through interfaces, not by pasting specs together. Write a small abstract spec of what a component promises (its states, messages, agency, done and error), and check with TLC that the detailed spec refines it.
- Compose the abstract specs; keep detail in the component's own spec, checked once.
- Components that will be composed use the same names for the same objects (the send queue, the done channel, the registration), from the shared module.

## Performance ([examples](references/examples.md#performance))

State count dominates run time: shrinking the state space usually buys 10x, speeding up each state 5-10%.

- Run safety configs with `-workers auto`; the command line defaults to one worker. Liveness checking is single-threaded regardless.
- Keep a small-constants config for iteration; run the large one only once the small one passes. Most bugs appear at the smallest sizes that allow them.
- Use symmetry sets for interchangeable model values (peers, connections, workers): about n! fewer states. Only on safety configs, and only when sound ([examples](references/examples.md#symmetry)):
    - the elements are model values, not numbers or strings;
    - no `CHOOSE` over the set or anything derived from it;
    - nothing orders or compares the elements, and no element is named in `Init` or `Next`;
    - the config checks no liveness property (TLC's liveness checking is unsound under symmetry).
- Choose atomicity deliberately: fine-grained steps only where goroutines interleave on shared state; coarse elsewhere. Too coarse hides bugs; too fine explodes the state space.
- Remove processes that only load initial state; enumerate the initial states instead.
- Use a deterministic choice (`CHOOSE`) only where the code's choice is genuinely arbitrary; nondeterminism the code does not have only adds states.
- Use bags, not sequences, when order does not matter.
- Construct the set you need; don't generate a large set and filter it ([examples](references/examples.md#construct-dont-filter)).
- A `VIEW` or state constraint that drops states can hide bugs. Use one only when the dropped part is provably irrelevant, and say why in the cfg.
- Profile before tuning: TLC's profiler shows which actions and expressions cost the most.
- After any optimisation, every verdict is unchanged ([examples](references/examples.md#verdicts)).

## Running TLC safely ([examples](references/examples.md#running-tlc))

- Cap the heap (`-Xmx`, or `JAVA_TOOL_OPTIONS=-Xmx6g`). The JVM defaults to a quarter of RAM; two uncapped TLC runs plus a test suite will exhaust a workstation.
- Run one TLC process at a time on a shared machine; serialise with a lock file when several agents or scripts run checks.
- Never run `make -j` over TLC targets.
- Check one spec directory at a time while working; the full suite is for merges.
- A fresh worktree needs the TLA+ tools (jar, JDK) in place before any run; a missing toolchain makes every config report as failed, which reads like a model regression.

## Verification discipline ([examples](references/examples.md#verdicts))

- A Bug config passes review only if it fails on the property it names. "It failed" is not enough; check which invariant or property TLC reported.
- Give every invariant teeth: a mutation config (the fix removed, a guard dropped) that the invariant catches.
- A config's expected outcome is in its name (`*Bug.cfg` fails, everything else passes), and the runner reports "as expected" or "wrong", not raw pass/fail.
- Record states and time per config before and after any change to a spec; an unexplained drop in distinct states is a lost behaviour until shown otherwise.
- Classify every function of the modelled code: modelled (an invariant names it), checked against reference vectors, a documented caller contract, or a gap. Report the gaps; don't let "has a model" stand for "is covered".
