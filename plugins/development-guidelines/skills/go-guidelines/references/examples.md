# Go examples

Do and Don't pairs for each rule in `SKILL.md`. Identifiers are illustrative.

## Scope of a change

Do: two commits, each reviewable on its own.

```text
refactor: extract fee calculation from Quote      (no behaviour change, tests unchanged)
fix: round fee up to the next whole unit          (one line plus a failing-first test)
```

Don't: one commit that fixes rounding, renames three types, moves two files and reorders imports.

```text
fix: fee rounding and cleanup                     (412 lines changed; the fix is 1 of them)
```

Do: a surgical fix when you are not refactoring.

```diff
-	return base + size*perByte
+	return base + (size*perByte+999)/1000*1000
```

Don't: the same fix with the function rewritten around it.

```diff
-func calc(b, s, p uint64) uint64 {
-	return b + s*p
-}
+func fee(size int, p Params) uint64 {
+	var total uint64 = p.Base
+	// ... 20 new lines ...
+}
```

Do: a reviewed plan, then small steps that each land green.

```text
plan: split Store into cache, writer, reader     (reviewed; diagram; 6 steps; benchmarks named)
refactor: move Entry and errors to store/types    (step 1 of 6)
refactor: extract writer behind Committer         (step 2 of 6)
...
```

Don't: start pulling threads and see where it goes.

```text
refactor: start splitting Store                   (3,800 lines, 41 files, tests disabled "for now")
wip: more splitting
fix: tests after split
```

## Naming

Do:

```go
package cache

type Store struct{ ... }

func (s *Store) Put(e Entry) error
```

Don't:

```go
package cache

type CacheEntryStorageManager struct{ ... }

func (m *CacheEntryStorageManager) PutCacheEntryIntoStore(newCacheEntryToStore CacheEntry) error
```

## Functions

Do: guard clauses keep the happy path at one level.

```go
func (s *Store) Load(key string) (Entry, error) {
	if key == "" {
		return Entry{}, ErrNoKey
	}

	raw, err := s.db.Get(key)
	if err != nil {
		return Entry{}, fmt.Errorf("get %q: %w", key, err)
	}

	return decode(raw)
}
```

Don't: each check pushes the work one level deeper.

```go
func (s *Store) Load(key string) (Entry, error) {
	if key != "" {
		raw, err := s.db.Get(key)
		if err == nil {
			return decode(raw)
		} else {
			return Entry{}, err
		}
	}
	return Entry{}, ErrNoKey
}
```

Do: choose the mode once, at construction.

```go
type verifier interface{ Verify(b Block) error }

func newVerifier(trusted bool) verifier {
	if trusted {
		return skipVerifier{}
	}
	return fullVerifier{}
}
```

Don't: branch on a flag at every step.

```go
func (p *Processor) step(b Block) error {
	if !p.opts.Trusted {
		// verify
	}
	// ...
	if !p.opts.Trusted && p.opts.Strict {
		// verify again
	}
}
```

## Layout

### Package documentation

Do: `doc.go` holds the package comment and nothing else.

```go
// doc.go

// Package cache keeps recently read entries in memory so repeated
// lookups skip the database.
package cache
```

```go
// store.go

package cache

type Store struct{ ... }
```

Don't: the package comment sits in whichever file was written first, or is split across several.

```go
// store.go

// Package cache provides a cache.
package cache

type Store struct{ ... }
```

```go
// evict.go

// Package cache evicts least recently used entries.
package cache
```

### Var blocks

Do:

```go
var (
	total   uint64
	skipped int
	seen    = make(map[string]bool)
)
```

Don't:

```go
var total uint64
var skipped int
var seen = make(map[string]bool)
```

### Blank lines

Do: each step stands on its own.

```go
func (s *Store) Compact(ctx context.Context) (int, error) {
	keys, err := s.db.Keys(ctx)
	if err != nil {
		return 0, fmt.Errorf("compact: list keys: %w", err)
	}

	cutoff, err := s.cutoff(ctx)
	if err != nil {
		return 0, fmt.Errorf("compact: cutoff: %w", err)
	}

	var stale []string

	for _, k := range keys {
		if k.Written.Before(cutoff) {
			stale = append(stale, k.ID)
		}
	}

	if len(stale) == 0 {
		return 0, nil
	}

	removed, err := s.db.Delete(ctx, stale)
	if err != nil {
		return 0, fmt.Errorf("compact: delete %d keys: %w", len(stale), err)
	}

	return removed, nil
}
```

Don't: one wall of code, and a check pulled away from the call it checks.

```go
func (s *Store) Compact(ctx context.Context) (int, error) {
	keys, err := s.db.Keys(ctx)

	if err != nil {
		return 0, fmt.Errorf("compact: list keys: %w", err)
	}
	cutoff, err := s.cutoff(ctx)
	if err != nil {
		return 0, fmt.Errorf("compact: cutoff: %w", err)
	}
	var removed int
	var stale []string
	for _, k := range keys {
		if k.Written.Before(cutoff) {
			stale = append(stale, k.ID)
		}
	}
	if len(stale) == 0 {
		return 0, nil
	}
	removed, err = s.db.Delete(ctx, stale)
	if err != nil {
		return 0, fmt.Errorf("compact: delete %d keys: %w", len(stale), err)
	}
	return removed, nil
}
```

### Struct literals

Do:

```go
e := Entry{
	Key:     key,
	Value:   raw,
	Written: now,
}
```

Don't:

```go
e := Entry{Key: key, Value: raw, Written: now}
```

### Building structs

Do: compute every field, then build the value once.

```go
func NewWorker(clk Clock, sink Sink, every time.Duration, logger *slog.Logger) *Worker {
	if every <= 0 {
		every = time.Second
	}

	return &Worker{
		clk:    clk,
		sink:   sink,
		every:  every,
		logger: logger.With("component", "worker"),
	}
}
```

Don't: build a half-empty value and patch it, so a reader has to scan the whole function to learn what it holds.

```go
func NewWorker(clk Clock, sink Sink, every time.Duration, logger *slog.Logger) *Worker {
	w := &Worker{}
	w.clk = clk
	w.sink = sink
	w.every = every
	if w.every <= 0 {
		w.every = time.Second
	}
	w.logger = logger.With("component", "worker")
	return w
}
```

### Field comments

Do: each field says what it holds and how it is used.

```go
type Worker struct {
	// clk supplies the time for each tick; tests pass a fake clock.
	clk Clock

	// sink receives each flushed batch; Run returns its error.
	sink Sink

	// every is the time between flushes. NewWorker defaults it to one second.
	every time.Duration

	// logger is tagged with component=worker.
	logger *slog.Logger
}
```

Don't: a reader has to find every use of a field to learn what it is.

```go
type Worker struct {
	clk    Clock
	sink   Sink
	every  time.Duration // interval
	logger *slog.Logger
}
```

## Interfaces

Do: the consumer declares what it calls, exported because it appears in `New`.

```go
package quote

// TxSource is the part of a transaction store that Service needs.
type TxSource interface {
	Tx(ctx context.Context, id string) (Tx, error)
}

func New(src TxSource) *Service {
	return &Service{
		src: src,
	}
}
```

```go
package store

type Store struct{ ... }

func New(db *sql.DB) *Store
func (s *Store) Tx(ctx context.Context, id string) (Tx, error)
func (s *Store) Put(ctx context.Context, tx Tx) error
```

Don't: the producer declares a mirror of itself and returns it.

```go
package store

type IStore interface {
	Tx(ctx context.Context, id string) (Tx, error)
	Put(ctx context.Context, tx Tx) error
	Delete(ctx context.Context, id string) error
	List(ctx context.Context) ([]Tx, error)
}

type StoreImpl struct{ ... }

func NewStore(db *sql.DB) IStore { return &StoreImpl{db: db} }
```

## Pure computation

Do:

```go
// fee is pure: tested with a table of inputs.
func fee(size int, p Params) uint64 {
	return p.Base + uint64(size)*p.PerByte
}

func (s *Service) Quote(ctx context.Context, id string) (uint64, error) {
	tx, err := s.db.Tx(ctx, id)
	if err != nil {
		return 0, fmt.Errorf("quote %s: %w", id, err)
	}

	p, err := s.db.Params(ctx)
	if err != nil {
		return 0, fmt.Errorf("quote %s: params: %w", id, err)
	}

	return fee(tx.Size, p), nil
}
```

Don't: the calculation can only be tested with a database.

```go
func (s *Service) Quote(ctx context.Context, id string) (uint64, error) {
	tx, err := s.db.Tx(ctx, id)
	if err != nil {
		return 0, err
	}
	base, _ := s.db.Param(ctx, "base")
	fee := base
	for i := 0; i < tx.Size; i++ {
		per, _ := s.db.Param(ctx, "per_byte")
		fee += per
	}
	return fee, nil
}
```

## Errors

Do:

```go
func (s *Store) Load(ctx context.Context, id string) (Key, error) {
	raw, err := s.db.Get(ctx, id)
	if err != nil {
		return Key{}, fmt.Errorf("load key %s: %w", id, err)
	}

	k, err := parseKey(raw)
	if err != nil {
		return Key{}, fmt.Errorf("parse key %s: %w", id, err)
	}

	return k, nil
}

type Secret []byte

func (Secret) String() string        { return "[redacted]" }
func (Secret) LogValue() slog.Value { return slog.StringValue("[redacted]") }
```

Don't: I/O without an error return, an unwrapped error, and a secret in both the error and the log.

```go
func (s *Store) Load(id string) Key {
	raw, _ := s.db.Get(context.Background(), id)
	k, err := parseKey(raw)
	if err != nil {
		log.Printf("bad key %x: %v", raw, err)
		panic(fmt.Errorf("invalid key material %x", raw))
	}
	return k
}

return err // which call failed, and on what?
```

## Logging

Do:

```go
logger.Info("batch committed",
	"blocks", n,
	"took", time.Since(start).Round(time.Millisecond),
	"size", humanize.IBytes(uint64(size)),
	"cache_hit_pct", fmt.Sprintf("%.1f", hitRatio*100),
)
// level=INFO msg="batch committed" component=writer blocks=50 took=1.42s size="12 MiB" cache_hit_pct=97.3
```

`humanize` is `github.com/dustin/go-humanize`; any byte formatter will do.

Don't: unstructured text, raw nanoseconds and E notation.

```go
log.Printf("committed %d blocks in %v ns, %v bytes, hit %v",
	n, time.Since(start).Nanoseconds(), float64(size), hitRatio)
// committed 50 blocks in 1423117000 ns, 1.2582912e+07 bytes, hit 0.9731
```

## Untrusted input

Do:

```go
type Limit int

func parseLimit(s string) (Limit, error) {
	n, err := strconv.Atoi(s)
	if err != nil || n < 1 || n > 1000 {
		return 0, fmt.Errorf("limit must be 1 to 1000, got %q", s)
	}
	return Limit(n), nil
}

var sortCols = map[string]string{"slot": "slot", "time": "created_at"}

func (s *Store) Txs(ctx context.Context, addr string, sort string, lim Limit) ([]Tx, error) {
	col, ok := sortCols[sort]
	if !ok {
		return nil, fmt.Errorf("unknown sort %q", sort)
	}

	q := "SELECT id, slot FROM txs WHERE addr = ? ORDER BY " + col + " LIMIT ?"
	rows, err := s.db.QueryContext(ctx, q, addr, int(lim))
	// ...
}
```

Don't: user input pasted into SQL and a shell, with no bound on the body.

```go
body, _ := io.ReadAll(r.Body)
addr := r.URL.Query().Get("addr")
q := fmt.Sprintf("SELECT * FROM txs WHERE addr = '%s' ORDER BY %s", addr, r.URL.Query().Get("sort"))
rows, _ := db.Query(q)

exec.Command("sh", "-c", "tar xf "+r.URL.Query().Get("file")).Run()
```

## Locks

Do:

```go
type Store struct {
	mu sync.Mutex // protects entries, hits

	// entries maps a key to its entry; Get and Put read and write it.
	entries map[string]Entry

	// hits counts Get calls that found their key, for the hit-rate metric.
	hits int
}

func (s *Store) Get(key string) (Entry, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()

	e, ok := s.entries[key]
	if ok {
		s.hits++
	}
	return e, ok
}
```

Don't: the lock leaks into the API, and nothing says what it guards.

```go
type Store struct {
	Mu      sync.Mutex
	Entries map[string]Entry
}

// GetLocked returns an entry. Caller must hold s.Mu.
func (s *Store) GetLocked(key string) Entry
```

## Goroutines

Do:

```go
func (w *Worker) Run(ctx context.Context) error {
	g, ctx := errgroup.WithContext(ctx)
	g.Go(func() error { return w.poll(ctx) })
	g.Go(func() error { return w.flush(ctx) })
	return g.Wait()
}

func (w *Worker) poll(ctx context.Context) error {
	t := time.NewTicker(w.every)
	defer t.Stop()

	for {
		select {
		case <-ctx.Done():
			return nil
		case <-t.C:
			w.tick(ctx)
		}
	}
}
```

Don't: no context, no join, a ticker that is never stopped.

```go
func (w *Worker) Start() {
	go func() {
		for range time.Tick(w.every) {
			w.tick(context.Background())
		}
	}()
}
```

## Assertions

The package:

```go
//go:build debug

package check

const tagged = true
```

```go
package check

// Pre panics when a caller broke the method's contract.
func Pre(ok bool, what string)

// Post panics when the method broke its own contract.
func Post(ok bool, what string)

// Slow is constant false without the build tag, so guarded blocks are removed.
const Slow = tagged
```

Do:

```go
func (s *Store) Put(e Entry) {
	check.Pre(e.Key != "", "Store.Put: entry has a key")
	// ...
	check.Post(len(s.entries) > 0, "Store.Put: store not empty")
}
```

Don't: the check is only a comment, so nothing catches a caller that breaks it.

```go
// Put stores e. e.Key must not be empty.
func (s *Store) Put(e Entry) { ... }
```

## Slow checks

Do:

```go
// insert requires s.mu held.
func (s *Store) insert(e Entry) {
	if check.Slow {
		if s.mu.TryLock() {
			s.mu.Unlock()
			check.Pre(false, "Store.insert: s.mu held")
		}
	}

	s.order = append(s.order, e.Key)
	s.entries[e.Key] = e

	if check.Slow {
		check.Post(len(s.order) == len(s.entries), "Store.insert: index matches map")
		check.Post(sort.StringsAreSorted(s.order), "Store.insert: order sorted")
	}
}
```

Don't: the scan runs in every release build, because the argument is evaluated before `Post` is called. Measured on Go 1.27: 239 ns per call for a 1,000-item slice, against 1.6 ns for the guarded or callback form.

```go
func (s *Store) insert(e Entry) {
	// ...
	check.Post(sort.StringsAreSorted(s.order), "order sorted")
}
```

Don't: a slow check that changes state. Release builds skip the block, so they behave differently.

```go
if check.Slow {
	s.cache = rebuild(s.entries) // side effect
	check.Post(s.cache.Valid(), "cache valid")
}
```

## Traces

Do:

```go
func (l *Loop) applyChunk(ctx context.Context, c chunk) error {
	ctx, task := trace.NewTask(ctx, tracing.TaskApplyChunk)
	defer task.End()

	trace.WithRegion(ctx, tracing.RegionValidate, func() { c.validate() })

	r := trace.StartRegion(ctx, tracing.RegionWriterWait)
	w := l.writer.Acquire(ctx)
	r.End()
	defer w.Release()
	trace.Log(ctx, "job", "apply_chunk")
	return w.Commit(ctx, c.delta())
}
```

Don't: names typed inline drift apart, and a region left open on an early return hides the stage that failed.

```go
r := trace.StartRegion(ctx, "apply validate")
if err := c.validate(); err != nil {
	return err // region never ended
}
r.End()
```

## Comments

Do:

```go
// Publish the tip before releasing mu: a rollback that takes mu
// between commit and publish reads a stale tip and skips the undo.
// This race has reappeared three times (#101, #214, #377).
s.tip.Publish(next)
s.mu.Unlock()
```

Don't: the first restates the code; the second hides the reason behind a link.

```go
// unlock the mutex
s.mu.Unlock()

// see #377
s.tip.Publish(next)
```

Do: the diagram sits in the package doc comment, in `doc.go`.

````go
// doc.go

// Package writer owns the single database write connection.
//
//	```mermaid
//	flowchart LR
//	  apply --> writer --> db[(database)]
//	  rollback --> writer
//	```
package writer
````

Do: link the exact section the code follows, at a pinned version.

```go
// decodeHeader parses a frame header as defined in RFC 9113, section 4.1:
// https://www.rfc-editor.org/rfc/rfc9113#section-4.1
// The reserved bit is ignored on receipt, as that section requires.
func decodeHeader(b []byte) (Header, error)
```

Don't: name the spec without a link, or link a moving target.

```go
// decodeHeader parses the header per the spec.
func decodeHeader(b []byte) (Header, error)

// See https://github.com/example/spec/blob/main/README.md
func decodeHeader(b []byte) (Header, error)
```

## Testing

Do: the constructor takes the dependencies; the test injects fakes and checks for leaks.

```go
func TestWorkerFlushesOnTick(t *testing.T) {
	defer goleak.VerifyNone(t)
	clk := fakeclock.New()
	sink := &recordSink{}
	w := NewWorker(clk, sink, time.Second)

	ctx, cancel := context.WithCancel(t.Context())
	done := make(chan error)
	go func() { done <- w.Run(ctx) }()

	clk.Advance(time.Second)
	sink.waitFor(t, 1)
	cancel()
	require.NoError(t, <-done)
}
```

Don't: a struct literal with half its fields, a package-level seam, and a sleep.

```go
var nowFunc = time.Now // seam

func TestWorkerFlushesOnTick(t *testing.T) {
	nowFunc = func() time.Time { return fixed }
	w := &Worker{sink: &recordSink{}, every: time.Millisecond}
	w.Start()
	time.Sleep(50 * time.Millisecond)
	// no Close, no leak check
}
```

## Networking tests

Do:

```go
ln, err := net.Listen("tcp", "127.0.0.1:0")
require.NoError(t, err)
t.Cleanup(func() { ln.Close() })

srv := NewServer(ln) // takes the listener, not a port
go srv.Serve(t.Context())

conn, err := net.Dial("tcp", ln.Addr().String())
```

Don't: a fixed port collides with parallel tests and anything else on the machine.

```go
srv := NewServer(":3001")
```

Don't: probe-then-bind races; the port can be taken between `Close` and `Listen`.

```go
func freePort() int {
	l, _ := net.Listen("tcp", "127.0.0.1:0")
	defer l.Close()
	return l.Addr().(*net.TCPAddr).Port
}

srv := NewServer(fmt.Sprintf(":%d", freePort()))
```
