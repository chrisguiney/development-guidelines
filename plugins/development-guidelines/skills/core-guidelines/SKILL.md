---
name: core-guidelines
description: Language-neutral development rules for writing, refactoring or reviewing code in any language - scope of a change, refactor planning, naming, function and file size, pure computation vs I/O, errors, logging, secrets, untrusted input, locks, background work, assertions, metrics, comments, specifications, tests. Use before writing or reviewing code; then load the language skill (for example go-guidelines) for its specifics and examples.
---

# Core development guidelines

New code meets every rule below, even where the surrounding file does not. Language skills (`go-guidelines`, ...) apply these rules to one language and carry Do and Don't examples.

- Write for a reader new to the project and its domain: a component's job is obvious from its name and where it lives.
- Rules marked **(enforce)** are enforced by a test where the repository has one, such as an architecture test. Check the rest in review.
- Code that does not meet a limit yet goes in an allowance file. That file only shrinks; no entry is added.

## Scope of a change

A change either refactors or changes behaviour, never both. Mixed together, a reviewer cannot tell which lines change what the code does, and a regression cannot be bisected to either half.

1. **Refactor first, then change behaviour.** Land a pure refactor (tests unchanged and passing), then the behaviour change on top as its own commit or pull request.
2. **If you are not explicitly refactoring, leave the surrounding code alone.** No renames, reformatting, extracted helpers or "while I'm here" cleanups next to a fix.
3. **Choose one:** refactor first, or make a surgical change now and come back for the refactor later in its own change.
4. A refactor changes no test's expectations. If an expected result has to change, it is a behaviour change.
5. **Plan a large refactor before writing it; don't wing it.** Anything that spans several modules, moves ownership of state or locks, or takes more than a few pull requests gets a written plan, reviewed before the first line of code. The plan names:
    - the target design, with a diagram of the components and what calls what;
    - the ordered steps, each landing on its own with tests green;
    - how behaviour is shown unchanged at each step: tests that move, tests that stay, any model or parity check;
    - what is measured for performance, before and after;
    - where it can stop halfway and still leave the code better than it found it.

## Naming

- Short, human-readable words; aim for 20 characters or fewer. The module and the parameter types carry the rest.
- No Java-style compounds: no `Manager`, `Helper`, `Impl`, `Util`, and no name that repeats its module.
- Name a function for the one thing it does.

## Size

| Unit | Limit | How |
| --- | --- | --- |
| Function length | 80 lines | test (enforce) |
| Nesting depth | 3 levels; aim for 1 | test (enforce) |
| Type members | about 20 fields plus methods; past that, split | review |
| File | 3,000 lines is big; split well before it | review |
| Package or module | about 25 files; past that, split | review |

Reduce depth with guard clauses and early return; turn a long loop body into a method on a small per-iteration type; choose a mode once at construction instead of branching on a flag at every step.

## Structure

- Low coupling, high cohesion: each component has one obvious responsibility and reaches others only through narrow interfaces declared by the consumer.
- **Pure computation is apart from I/O.** A function either computes or does I/O, not both. Pure functions take values and return values (no database, network, clock, file or global state) and are tested with tables of inputs and outputs. The I/O around them stays thin: read, compute, write.
- Where an upstream or reference implementation has the same boundary, take it, and name the upstream file so the two can be read side by side.

## Errors, logging and secrets

1. Every function that does I/O reports failure to its caller (an error return, a result type); it never crashes, swallows the error, or logs and carries on.
2. Wrap each error with the operation and the non-sensitive identifiers it ran on, keeping the original matchable.
3. Sensitive data never goes into an error or a log, at any level: keys, seeds, mnemonics, passwords, tokens, signed payloads, credentials in connection strings, personal data. Log an identifier or a fingerprint instead. Types holding secrets print a redacted value.
4. Log structured key-value records with short, stable `snake_case` keys.
5. Give values in units a human reads at a glance: durations, `12 MiB`, plain integer counts, percentages. Never make a human read E notation or raw nanoseconds.

## Untrusted input

Everything from outside the process is hostile until validated: requests, peers, files, environment, config, and storage another process can write.

1. Validate at the boundary, once, into a typed value; inner code accepts the type, not raw strings or bytes.
2. Allowlist what is valid; never blocklist what looks bad.
3. Bound every read: body size, item counts, nesting depth. Check a length field before allocating from it.
4. SQL always uses query parameters, never string building. Identifiers that cannot be parameters come from a fixed allowlist.
5. No shell: run commands with an argument vector. Confine user-named paths to a root. Render HTML with an escaping template engine.

## Concurrency

- **Locking is encapsulated.** A lock is never part of a public interface: callers never lock, and nothing exported asks them to.
- Each lock is a private field of the type whose fields it protects, declared next to them with `// protects a, b, c` (enforce).
- Write down the lock order, outermost first. Leaf locks never wrap a call out of their type.
- Never hold a lock across an event publish, a blocking send, a network call or a wait for a shared resource.
- Every background task has one owner, one cancellation signal and one join point. The owner that starts it stops it; every blocking operation observes cancellation; timeouts come from the cancellation context, never a bare wait.
- On a serialized resource (a single database writer, one connection, one event lane) keep work short, never block it on another lock, and measure changes to that path before they merge.

## Assertions

State each function's contract as pre- and post-condition checks that run in debug builds and tests and cost nothing in release builds. Expensive checks (iteration, allocation, I/O) must be removed by the compiler in release builds, and checks never have side effects.

## Observability

- Never rename or remove a published metric; dashboards and alerts depend on the names (enforce: compare the registered set before and after).
- Each component registers the metrics for the state it owns. Add metrics that answer a real operational question: wait and hold time on shared resources, step durations, staleness, background jobs started, finished and discarded.
- Trace each logical operation and its stages, with names defined once as constants.

## Comments and documentation

- Comment why: an invariant, a non-obvious algorithm, a gotcha. Delete comments that restate the code, label sections or narrate history.
- Public API gets proper doc comments.
- **Link to the specification** the code implements (RFC, CIP, protocol or format spec, formal model, upstream source file), to the section, at a pinned version where the spec changes over time.
- Don't cite an issue or pull request as a stand-in for the reason. A citation is fine in addition, for history on a problem that keeps coming back.
- Draw component structure, lock order and data flow as Mermaid in the source file the diagram describes, so it changes with the code.

## Testing

1. Test each component as a unit, built through its constructor, with dependencies injected (clock, store, network). Keep whole-system integration tests to a small safety net over startup, restart, recovery and lifecycle transitions.
2. No global seam variables for tests; make the value a constructor parameter.
3. Every component test checks for leaked background tasks after shutdown.
4. Remove an integration test only in the change that adds unit tests for the same behaviour, and list the pairs.
5. **Networking tests bind port 0** on the loopback address and read the port back from the open listener. Never hardcode a port, and never probe for a free one: between the probe closing and the test binding, another process can take it.

A unit test package stays under 60 seconds and 2 GiB under the race detector.

## Review checklist

- [ ] Behaviour unchanged for users, unless the change says otherwise
- [ ] Refactor or behaviour change, not both; no cleanups around a fix; large refactors have a reviewed plan
- [ ] No function over 80 lines or nested deeper than 3 levels; no new allowance entry
- [ ] Names short and plain; each type has one responsibility
- [ ] Pure computation apart from I/O; interfaces narrow and declared by the consumer
- [ ] I/O reports errors, wrapped with context; no secrets in errors or logs; structured logs with readable units
- [ ] Input validated at the boundary, reads bounded, SQL parameterized, no shell
- [ ] No lock in a public API; locks documented and ordered; none held across blocking work
- [ ] Every background task owned, cancellable and joined
- [ ] Contracts stated as debug-only checks; no metric renamed or removed
- [ ] Unit tests through constructors, leak-checked, sockets on port 0
- [ ] Serialized paths measured if touched
