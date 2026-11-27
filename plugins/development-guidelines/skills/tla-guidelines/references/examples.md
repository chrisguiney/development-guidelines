# TLA+ examples

Do and Don't pairs for the rules in [SKILL.md](../SKILL.md). Specs model Go; operator names follow a shared `GoConcurrency` module where one helps.

## Model-first

Do: reproduce the defect as a failing config before touching the code.

```text
specs/indexer-store-ready/
  IndexerStoreReady.tla
  IndexerStoreReadyBug.cfg        \* the code as it is: fails WriteSeesReadyStore
  IndexerStoreReadyFixed.cfg      \* the proposed fix: passes
  IndexerStoreReadyWorkerBug.cfg  \* same defect with the new worker present: still fails
```

```text
$ scripts/run-model.sh specs/indexer-store-ready
  fails  IndexerStoreReadyBug.cfg (as expected)
  pass   IndexerStoreReadyFixed.cfg
specs/indexer-store-ready: 5 as expected, 0 wrong
```

Then the Go test that fails for the reason TLC printed, then the fix.

Don't: rerun the flaky test in a loop for an hour, then write the model afterwards to match the fix you already chose.

## Traceability

Do:

```tla
(***************************************************************************)
(* Models dingo node.go at a0d5b50a4.                                      *)
(*   StartWriter   node.go:1397  indexer writer started under the gate     *)
(*   FailLater     node.go:1412  a later startup step returns an error     *)
(*   Rollback      node.go:1602  cleanupFailedStartup closes the store     *)
(*   ReleaseGate   node.go:1610  gate released after rollback              *)
(***************************************************************************)
```

```markdown
| Id | Code | Model | Why | Config |
| D44 | stall clock sampled every 30 s | clock advanced on applied tip | the sample is the bug | AntiFlapPinSampledStampBug |
```

Don't: a spec with no commit and no line references. When the code moves, nobody can tell which actions still describe it.

## Goroutines

Do: one process per goroutine, so their steps interleave.

```tla
\* go s.readLoop(); go s.sendLoop(); the caller returns
Spawn == /\ pc["caller"] = "spawn"
         /\ pc' = [pc EXCEPT !["caller"] = "done",
                              !["read"]   = "loop",
                              !["send"]   = "loop"]
         /\ UNCHANGED <<queue, closed>>
```

Don't: one action that reads, sends and returns at once.

```tla
Serve == /\ msg' = Read(conn)
         /\ conn' = Send(conn, Reply(msg'))   \* no interleaving between reader and sender
```

## Channels

Do: capacity, blocking and close semantics, with panics as checked states.

```tla
CanSend(ch)   == ~closed[ch] /\ Len(buf[ch]) < Cap[ch]
Send(ch, v)   == /\ IF closed[ch] THEN panicked' = TRUE ELSE panicked' = panicked
                 /\ Len(buf[ch]) < Cap[ch] \/ closed[ch]
                 /\ buf' = IF closed[ch] THEN buf ELSE [buf EXCEPT ![ch] = Append(@, v)]
CanRecv(ch)   == buf[ch] # <<>> \/ closed[ch]
Close(ch)     == /\ IF closed[ch] THEN panicked' = TRUE ELSE panicked' = panicked
                 /\ closed' = [closed EXCEPT ![ch] = TRUE]
NoPanic       == ~panicked
```

Don't:

```tla
Send(ch, v) == buf' = [buf EXCEPT ![ch] = Append(@, v)]   \* never full, never closed
```

That model cannot find a blocked sender or a send-on-closed panic, which are the bugs channels cause.

## Select default

Do: `default` is enabled exactly when no other case is.

```tla
\* select { case done <- struct{}{}: ; default: }   // non-blocking notify
Notify == \/ /\ CanSend("done") /\ Send("done", 1)
          \/ /\ ~CanSend("done")
             /\ dropped' = dropped + 1                 \* the signal is lost
             /\ UNCHANGED <<buf, closed, panicked>>
```

Don't: model the notify as always delivered; the lost wakeup disappears with it.

## Broadcast

Do: close-and-replace as a generation; reading the generation and waiting are separate steps.

```tla
Wake   == /\ gen' = gen + 1                          \* close(ch); ch = make(chan struct{})
ReadCh == /\ pc[w] = "read" /\ seen' = [seen EXCEPT ![w] = gen] /\ pc' = [pc EXCEPT ![w] = "wait"]
Wait   == /\ pc[w] = "wait" /\ gen > seen[w]          \* woken only by a later close
          /\ pc' = [pc EXCEPT ![w] = "run"]
```

Don't: `Wait == pc[w] = "wait" /\ work # {}` checks the condition directly and never misses a wakeup the code can miss.

## Locks

Do: split the critical section where it can block.

```tla
\* mu.Lock(); queue <- msg; mu.Unlock()
Lock    == holder = None /\ holder' = self
SendIn  == holder = self /\ CanSend("queue") /\ Send("queue", msg)   \* may block while holding mu
Unlock  == holder = self /\ holder' = None
```

Don't: `LockSendUnlock == CanSend("queue") /\ Send("queue", msg)` as one step: the lock is never observed held while the send blocks, so the deadlock with a goroutine that needs `mu` to drain the queue is unreachable.

## Bounded queues

Do: `Cap["sendQueue"] = 80` as in the code, or a small constant that still lets it fill in the model (2 or 3).

Don't: `Cap["sendQueue"] = Infinity`, or a sequence with no capacity check.

## Deadlock

Do: keep the check on and name what the code waits for.

```tla
Next == \E p \in Procs : Step(p)
Spec == Init /\ [][Next]_vars /\ \A p \in Procs : WF_vars(Step(p))
EveryCallReturns == \A c \in Callers : pc[c] = "call" ~> pc[c] = "returned"
```

Don't:

```tla
Next == \/ \E p \in Procs : Step(p)
        \/ UNCHANGED vars                  \* TLC can never report a deadlock
```

or `CHECK_DEADLOCK FALSE` with no reason in the cfg and no liveness property.

## Split configs

Do:

```text
ConnLifecycleSafety.cfg     INVARIANTS NoPanic NoLeak   SYMMETRY Symmetry   (run with -workers auto)
ConnLifecycleLiveness.cfg   PROPERTY EveryCallReturns   smaller constants, no SYMMETRY
```

Don't: one cfg with both, which forces single-threaded checking and rules out symmetry for the invariants.

## Refinement

Do: an abstract interface spec, and a refinement check.

```tla
---- MODULE ChainSyncIface ----   \* agency, messages, done/error only
...
---- MODULE ChainSyncClient ----
Iface == INSTANCE ChainSyncIface WITH state <- AbstractState(pc, queue)
Refines == Iface!Spec
```

```text
ChainSyncClientRefines.cfg:  PROPERTY Refines
```

Compose `ChainSyncIface` with the muxer and handshake interfaces in the connection model.

Don't: paste the detailed chain-sync, block-fetch and muxer specs into one module; the product of their state spaces will not finish.

## Performance

Do: a fast config for iteration beside the real one.

```text
BlockFetchSmall.cfg:  Peers = {p1, p2}   MaxBlocks = 2
BlockFetch.cfg:       Peers = {p1, p2, p3}  MaxBlocks = 4
```

Don't: iterate on the large config and wait twenty minutes per edit.

## Symmetry

Do: model values, no CHOOSE, safety only.

```tla
CONSTANT Peers                       \* model values
Symmetry == Permutations(Peers)
```

```text
CONSTANT Peers = {p1, p2, p3}
SYMMETRY Symmetry
INVARIANT NoDoubleRegistration
```

Don't:

```tla
Leader == CHOOSE p \in Peers : TRUE     \* breaks symmetry: TLC may report wrong results
```

or `SYMMETRY` on a config that also checks `PROPERTY EveryCallReturns`.

## Construct, don't filter

Do:

```tla
Assignments == { [i \in Items |-> f[i]] : f \in [Items -> Workers] }
```

Don't:

```tla
Assignments == { a \in SUBSET (Items \X Workers) : IsFunction(a) }   \* 2^(|I|*|W|) candidates
```

## Verdicts

Do: compare per config before and after any change.

```text
config                          before                  after
BlockFetchQueueBug.cfg          fails NoLostRange       fails NoLostRange      unchanged
BlockFetchQueue.cfg             pass  1,204,331 states  pass  211,880 states    unchanged (symmetry)
```

Don't: accept "all configs as expected" after an optimisation without checking that each Bug config still fails on the same property. A Bug config that starts passing means the optimisation removed the behaviour the bug needs.

## Running TLC

Do:

```sh
flock "$LOCK" env JAVA_TOOL_OPTIONS=-Xmx6g scripts/run-model.sh specs/muxer
```

Don't:

```sh
make -j check &          # every config spawns a JVM at 25% of RAM
make -j check -C ../other-models &
go test -race ./... &
```

This exact combination exhausted 61 GB and the kernel killed the session.
