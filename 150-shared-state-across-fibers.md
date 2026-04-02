# Shared State Across Fibers

## Dependencies

- `"com.softwaremill.ox" %% "core"` — structured concurrency scopes, `Channel`
  for inter-fiber communication

---

## The problem: concurrent state mutation

When multiple fibers (e.g. a request handler and a periodic background task)
need to share mutable state, naive approaches create data races. Virtual threads
don't change the JVM memory model — `var` writes are not guaranteed visible
across threads without synchronization.

## Single-owner pattern with channels

The safest approach is to confine state mutation to a single fiber and
communicate via Ox channels. Other fibers send signals; the owner fiber acts
on them:

```scala
import ox.*
import ox.channels.{Channel, ChannelClosedException}

def run()(using Ox): Unit =
  var state = initialState()

  val saveSignal = Channel.buffered[Unit](1)

  // Timer sends periodic save signals
  fork:
    forever:
      sleep(saveInterval)
      try saveSignal.send(())
      catch case _: ChannelClosedException => ()
  .discard

  // Single owner: main loop updates state AND saves
  inputSource.foreach: item =>
    state = process(state, item)

    // Non-blocking check for save signal
    saveSignal.tryReceive() match
      case _: ChannelClosedException => ()
      case _ => state = save(state)
```

> **Important:** State is owned by one fiber only — no concurrent reads or
> writes. The timer does not touch state directly; it sends a signal that the
> owner fiber acts on at a safe point.

## AtomicReference pattern

When the single-owner pattern doesn't fit (e.g. the input source itself blocks
the fiber and cannot interleave signal checks), use `AtomicReference` with
**atomic read-modify-write** operations:

```scala
import java.util.concurrent.atomic.AtomicReference

def run()(using Ox): Unit =
  val stateRef = AtomicReference(initialState())

  // Background fiber: atomic read-modify-write
  fork:
    forever:
      sleep(saveInterval)
      stateRef.updateAndGet(save).discard
  .discard

  // Main fiber: atomic read-modify-write
  inputSource.foreach: item =>
    stateRef.updateAndGet(state => process(state, item)).discard
```

> **Warning:** Never use `stateRef.get()` followed by `stateRef.set(newState)`.
> Another fiber can modify the state between the get and set, and the set
> silently overwrites those changes. Always use `updateAndGet` or
> `getAndUpdate` for atomic read-modify-write.

This requires that `process` and `save` are **pure functions** from old state to
new state — they cannot read the `AtomicReference` themselves, or they'll see
stale data:

```scala
// Pure state transitions — no external state reads
def process(state: State, item: Item): State =
  state.copy(counts = state.counts.updated(item.key, state.counts.getOrElse(item.key, 0L) + 1))

def save(state: State): State =
  storage.write(state.counts)
  state.copy(lastSavedAt = Instant.now())
```

## Extracting return values from `updateAndGet`

When a fiber needs both the updated state and a derived value (e.g. whether the
item was accepted or rejected), return both in the state and extract afterward:

```scala
case class StateWithResult(state: State, lastResult: Result)

stateRef.updateAndGet: current =>
  val (newState, result) = process(current.state, item)
  StateWithResult(newState, result)
```

This avoids the anti-pattern of using a mutable `var` to smuggle values out of
the `updateAndGet` lambda.

## When to use which

| Pattern | Use when |
|---------|----------|
| **Single-owner + channels** | One fiber owns all state; others only send signals. Simplest, no races possible. |
| **AtomicReference + updateAndGet** | Multiple fibers must independently update state; state transitions are pure functions. |
| **Ox supervised + actor** | Complex protocols where fibers need request/response interaction, not just fire-and-forget signals. |

> **Important:** Avoid sharing mutable state across fibers whenever possible.
> Restructure the design so one fiber owns the state and others communicate
> through channels. Use `AtomicReference` only when the single-owner pattern
> is impractical.
