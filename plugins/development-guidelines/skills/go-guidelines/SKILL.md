---
name: go-guidelines
description: Go coding rules for writing, refactoring or reviewing Go code - naming, layout (doc.go, var blocks, blank lines, struct literals, field comments), interfaces, packages, pure vs I/O, error wrapping, slog, secrets, untrusted input, SQL, mutexes, goroutines with Run or Start/Close, debug assertions with a check package and build tag, metrics, runtime/trace, doc comments, tests, port-0 listeners. Use before writing or reviewing any .go file, together with core-guidelines.
---

# Go guidelines

Go applications of `core-guidelines`, plus Go-only rules. Every rule has a Do and Don't example in [references/examples.md](references/examples.md); read the section that matches the code in hand. Examples use a `check` package, a `debug` build tag and Prometheus; substitute the project's equivalents.

## Naming ([examples](references/examples.md#naming))

- The package name is part of the identifier: `cache.Store`, not `cache.CacheStore` or `CacheEntryStorageManager`.
- Let the parameter type speak so the variable stays short: `Put(e Entry)`.
- A method named for a critical section replaces an inline lock, defer and closure block.
- An exported type that moves keeps its old name through a type alias.

## Functions ([examples](references/examples.md#functions))

- Guard clauses keep the happy path at one level.
- Mode flags and option structs become two types behind one small interface, chosen at construction.
- A `switch` on a variant (version, era, backend) inside logic becomes a descriptor per variant with function fields.

## Layout ([examples](references/examples.md#layout))

- Package documentation lives in `doc.go`, which holds the package doc comment and the `package` clause and nothing else. No other file in the package has a package comment ([examples](references/examples.md#package-documentation)).
- Two or more successive `var` declarations become one `var ( ... )` block ([examples](references/examples.md#var-blocks)).
- Blank lines separate the steps of a function, so a reader sees where one task ends and the next begins ([examples](references/examples.md#blank-lines)):
    - before and after a `for` loop;
    - before a `var` declaration, unless it is the first line of the function;
    - between successive related `if` statements. A call stays directly above the `if err != nil` that checks it; the blank line goes between one call-and-check and the next.
- A struct literal with fields spans multiple lines: one field per line, trailing comma ([examples](references/examples.md#struct-literals)).
- Compute every field first, then build the struct in one literal. Don't create a struct and then fill it in with `x.y = ...`; assigning a field later is for state that changes during the value's life, not for finishing construction ([examples](references/examples.md#building-structs)).
- Every struct field has a doc comment on the line directly above it, saying what the field holds and how it is used. The exception is a mutex, which keeps its `// protects a, b, c` comment at the end of its line ([examples](references/examples.md#field-comments)).

## Packages and interfaces ([examples](references/examples.md#interfaces))

- Components live under `internal/`. A component never imports the package that assembles it, and never imports another component's concrete type (enforce).
- Define an interface in the package that uses it, with only the methods that package calls (1 to 5).
- **Accept interfaces, return concrete types.** No `IStore`, `StoreInterface` or `StoreImpl`; no interface mirroring a struct; no interface declared next to its only implementation "for mocking".
- An interface in an exported signature is exported and documented. Name single-method interfaces for the method (`Reader`, `Committer`).
- Add an interface when a second implementation or a test fake needs it, not in advance.
- Pure functions apart from I/O ([examples](references/examples.md#pure-computation)).

## Errors and logging ([examples](references/examples.md#errors))

- A function that does I/O returns `error`. No `panic`, no `_ =` on an I/O error, no log-and-continue.
- Wrap with `fmt.Errorf("load key %s: %w", id, err)`; `%w` keeps `errors.Is` and `errors.As` working. A bare `return err` loses which call failed.
- Types holding secrets implement `String`, `GoString`, `Format`, `MarshalJSON` and `LogValue` to redact.
- Log with `log/slog` key-value attributes; take a `*slog.Logger` in the constructor with the component attached (`logger.With("component", "writer")`) ([examples](references/examples.md#logging)).
- Durations as `time.Duration` rounded (`1.42s`), bytes formatted (`12 MiB`), ratios as percentages. Convert a `float64` before logging so `slog` never prints `1.2582912e+07`.

## Untrusted input ([examples](references/examples.md#untrusted-input))

- Parse into a typed value at the boundary (`parseLimit(string) (Limit, error)`).
- Bound reads with `http.MaxBytesReader` and `io.LimitReader`.
- SQL with `?` or `$1` placeholders through `QueryContext`; never `fmt.Sprintf` or `+`. Sort columns from a fixed map.
- `exec.Command(name, args...)`, never `sh -c`. User-named paths through `os.Root`. HTML through `html/template`.

## Concurrency ([examples](references/examples.md#locks))

- `sync.Mutex` is an unexported field declared directly above the fields it protects, with `// protects a, b, c`. Exported methods take their own locks; no `...Locked` methods or "caller must hold" docs in the public API.
- A component that runs goroutines has one of two shapes ([examples](references/examples.md#goroutines)):
    1. `Run(ctx context.Context) error`: blocks until the context is cancelled or a fatal error, and returns only after every goroutine it started has exited. The caller owns the goroutine, typically through `errgroup`.
    2. `Start(ctx context.Context)` and `Close(ctx context.Context) error`: `Start` derives one cancellable context for every goroutine; `Close` cancels it and waits on one wait group, bounded by the `Close` context's deadline.
- Never start a goroutine from a callback another component runs; enqueue work on the owner.
- Every blocking operation selects on `ctx.Done()`. Timeouts use `context.WithTimeout`. Tickers and timers are stopped by their creator; no `time.Tick` in long-lived code.

## Assertions ([examples](references/examples.md#assertions))

A small `check` package, gated by a `debug` build tag:

- `check.Pre(ok bool, what string)` and `check.Post(ok bool, what string)` panic with the component and method name when `ok` is false in debug builds. The argument is evaluated in every build, so pass only a comparison or two.
- `const check.Slow` is true only with the tag. Anything that iterates, allocates, calls a method or does I/O goes inside `if check.Slow { ... }`; without the tag the compiler deletes the block ([examples](references/examples.md#slow-checks)).
- A callback form (`check.PreFn(func() bool, what)`) is also free in release builds once inlined, and keeps working if checks are also enabled from `testing.Testing()`; prefer it for a single expensive condition, and `if check.Slow` for several asserts sharing one computation.
- An unexported helper that relies on its caller's lock checks it with `TryLock` inside `check.Slow`.
- Slow checks never change state: release builds skip them.
- Tests, the race job and test environments build with the tag; release builds do not.

## Observability ([examples](references/examples.md#traces))

- `runtime/trace`: a task per logical operation crossing goroutines, a region per stage, a region around each non-leaf lock acquisition, `trace.Log` for the job kind at shared resources.
- Task and region names are constants in one package. End regions with `defer` or `trace.WithRegion` so an early return cannot leave one open.

## Comments ([examples](references/examples.md#comments))

- Doc comments on every exported identifier.
- Mermaid diagrams go in the package doc comment in `doc.go`, or in the doc comment of the type they describe.
- Specification links name the section and a pinned version.

## Testing ([examples](references/examples.md#testing))

- Construct through `New...`, inject fakes (clock, sink, store), drive `Run` in a goroutine, cancel, and require its error.
- `defer goleak.VerifyNone(t)` (or the project's leak check) in each component test.
- No package-level `var nowFunc = time.Now` seams; no `time.Sleep` to wait for an event.
- Listen on `127.0.0.1:0`, pass the `net.Listener` to the code under test, dial `ln.Addr().String()`. `httptest.NewServer` for HTTP ([examples](references/examples.md#networking-tests)).
