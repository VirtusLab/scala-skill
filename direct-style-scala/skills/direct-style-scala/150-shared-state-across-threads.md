# Concurrency and Inter-Thread Communication

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `Flow`, `par`, `supervised`, `fork`,
  `forkDiscard`, `forkUserDiscard`, channels, actors, `computeIntensive`,
  `cede`

---

## Use Ox primitives

Use Ox for threading, mailboxes, fan-out, and async results. Prefer a local
`supervised` scope around one request, message, or job. Use a parent or
application scope only when the spawned work must live as long as that scope.
Do not use raw `Thread.ofVirtual`, `LinkedBlockingQueue`, `synchronized`/`Lock`,
or `AtomicReference` as a lifecycle flag in application code. Use
`java.util.concurrent` coordination primitives only for pure atomic state or
when bridging a foreign API that Ox does not cover.

| Need | Use |
|------|-----|
| Fixed-arity parallel fan-out inside one operation | `par(...)` |
| One-shot async result needing explicit fork control | local `supervised` scope with `fork` and `.join()` before returning |
| App-owned worker that must drain on shutdown | `forkUserDiscard` plus channel shutdown |
| Daemon work that may be cancelled with the scope | `forkDiscard` |
| Mailbox or producer-consumer queue | `ox.channels.Channel[T]` |
| Collection fan-out | `Flow.mapPar` or `Flow.mapParUnordered` |
| Serialized access to mutable state | `Actor` |
| Long CPU-bound computation | `computeIntensive` (see below) |

## Flows

Flows are the preferred way to express concurrent data processing in Ox. A
`Flow[T]` describes a lazy, composable pipeline that handles concurrency,
backpressure, and error propagation behind a declarative API:

```scala
import ox.flow.Flow

Flow.fromValues(1, 2, 3, 5, 6)
  .filter(_ % 2 == 0)
  .map(_ * 10)
  .runForeach(println)
```

Flows are lazy — no elements are emitted until a `.run*` method is called.

### Concurrent processing with mapPar

`mapPar` runs a mapping function across multiple virtual threads, handling fork
management and error propagation automatically:

```scala
Flow.fromIterable(urls)
  .mapPar(8)(url => httpClient.get(url))
  .runForeach(response => process(response))
```

Use `mapParUnordered` when result ordering is irrelevant.

Without Flows, this would require manually creating a `supervised` scope,
forking workers, coordinating via channels, and handling errors — all of which
`mapPar` encapsulates.

### Merging multiple event sources

`merge` combines two flows into one, processing elements from both
concurrently. This replaces manual multi-channel select patterns:

```scala
val userEvents: Flow[Event] = Flow.fromSource(userChannel)
val systemEvents: Flow[Event] = Flow.fromSource(systemChannel)

userEvents
  .merge(systemEvents)
  .runForeach(event => handle(event))
```

### Stateful processing

`mapStateful` threads state through a flow without any shared mutable state:

```scala
Flow.fromIterable(items)
  .mapStateful(initialState()): (state, item) =>
    val newState = process(state, item)
    (newState, newState.lastResult)
  .runForeach(result => emit(result))
```

### Periodic operations with tick

`Flow.tick` creates a periodic signal that can be merged with a data flow:

```scala
val data: Flow[Event] = eventSource()
val flushSignal: Flow[Event] = Flow.tick(flushInterval, FlushTick)

data
  .merge(flushSignal)
  .mapStateful(initialState()):
    case (state, FlushTick)    => val s = flush(state); (s, None)
    case (state, DataEvent(e)) => val s = process(state, e); (s, Some(s.lastResult))
  .collect { case Some(result) => result }
  .runForeach(emit)
```

> **Important:** Use Flows as the default for concurrent data processing. They
> handle fork lifecycle, error propagation, and backpressure automatically.
> Drop to channels or manual forks only when Flow's declarative API doesn't
> fit your use case.

## Channels

Channels are the lower-level building block underneath Flows. Use them when you
need imperative control over sending and receiving — for example, custom
protocols or integration with callback-based APIs:

```scala
import ox.*
import ox.channels.*

supervised:
  val ch = Channel.bufferedDefault[String]

  forkDiscard:
    ch.send("hello")
    ch.send("world")
    ch.done()

  repeatWhile:
    ch.receiveOrClosed() match
      case msg: String =>
        println(msg)
        true
      case ChannelClosed.Done =>
        false
      case ChannelClosed.Error(t) =>
        throw t
```

`send` blocks when the buffer is full; `receive` blocks when the buffer is
empty. Since Ox runs on virtual threads, blocking is cheap.

Channels and Flows interoperate: `Flow.fromSource(channel)` wraps a channel in
a Flow, and `flow.runToChannel()` materialises a Flow into a channel.

### Channel-backed workers

Use this pattern for app-owned workers whose lifetime is intentionally tied to
the parent scope. For request- or job-level concurrency, create a local
`supervised` scope instead.

Give the worker a channel mailbox and start it from a factory. The factory
takes `(using Ox)`, spawns the fork, and returns a plain value; the constructor
does not carry the scope capability.

```scala
package com.example.myapp.workers

import java.io.Closeable
import ox.*
import ox.channels.*
import scala.util.control.NonFatal

final class Worker private[workers] (mailbox: Channel[Task]) extends Closeable:
  def submit(task: Task): Unit =
    mailbox.send(task)

  override def close(): Unit =
    mailbox.doneOrClosed().discard

object Worker:
  def start(...)(using Ox): Worker =
    val mailbox = Channel.bufferedDefault[Task]
    forkUserDiscard:
      repeatWhile:
        mailbox.receiveOrClosed() match
          case task: Task =>
            try
              process(task)
              true
            catch
              case NonFatal(t) =>
                logger.error("Worker task failed", t)
                true
          case ChannelClosed.Done =>
            false
          case ChannelClosed.Error(t) =>
            throw t
    new Worker(mailbox)
```

`close()` signals shutdown through the channel. `done()` lets queued tasks drain
in FIFO order; `doneOrClosed()` makes the user-facing close operation
idempotent without an `AtomicReference` or volatile flag.

> **Important:** A `forkUserDiscard` worker keeps the parent scope open until it
> exits. Wire `close()` into the owner's resource/application lifecycle; don't
> start this pattern from a scope that has no matching shutdown path.

> **Warning:** Never catch `case _: Throwable` in a worker. Let
> `InterruptedException` escape so the fork exits on interruption. Catch
> `NonFatal` around individual tasks only when one bad task should not kill the
> worker; fatal throwables must propagate.

### Signaling between threads

Use a channel instead of a shared flag when one thread needs to signal another:

```scala
supervised:
  val saveTrigger = Channel.rendezvous[Unit]

  forkDiscard:
    forever:
      sleep(saveInterval)
      saveTrigger.send(())

  repeatWhile:
    saveTrigger.receiveOrClosed() match
      case () =>
        saveSnapshot()
        true
      case ChannelClosed.Done =>
        false
      case ChannelClosed.Error(t) =>
        throw t
```

## Actors

When multiple threads need serialized access to a mutable object, use Ox's
built-in `Actor`. It guarantees that method invocations happen one at a time,
even when called from multiple threads:

```scala
import ox.*
import ox.channels.Actor

class StateHolder:
  private var counter: Int = 0

  def increment(delta: Int): Int =
    counter += delta
    counter

  def current: Int = counter

supervised:
  val ref = Actor.create(new StateHolder)

  val sent1 = fork(ref.tell(_.increment(5)))
  val sent2 = fork(ref.tell(_.increment(3)))
  sent1.join()
  sent2.join()

  val total = ref.ask(_.current)
```

`ask` blocks until the invocation completes and returns the result. `tell`
schedules the invocation without waiting — use it for fire-and-forget
operations. The example forks concurrent callers, then joins those sends before
asking for the final state.

## AtomicReference only for pure shared state

For simple cases where channels, actors, or Flows are overkill (e.g. a shared
counter read by many threads), `AtomicReference` with atomic read-modify-write
operations works:

```scala
import java.util.concurrent.atomic.AtomicReference

val stateRef = AtomicReference(initialState())

stateRef.updateAndGet(state => process(state, item)).discard
```

> **Warning:** Never use `stateRef.get()` followed by `stateRef.set(newState)`.
> Another thread can modify the state between the get and set, silently
> overwriting those changes. Always use `updateAndGet` or `getAndUpdate`.

> **Warning:** `updateAndGet` may retry the function under contention (CAS
> loop). The function passed to it MUST be side-effect-free — no I/O, no
> logging, no channel operations.

> **Warning:** Never use `AtomicReference` as a lifecycle, cancellation, or
> shutdown flag. Model lifecycle with Ox scope structure or channel completion.

## CPU-intensive work

Ox forks run on virtual threads, which are never preempted: a thread yields
only when it blocks. A long CPU-bound computation therefore monopolizes a
carrier thread — and since there are only as many carriers as CPU cores (by
default), a handful of such computations can starve every other virtual
thread in the process, including latency-sensitive ones like HTTP handlers.
Two remedies
(Ox ≥ 1.0.6), depending on the computation:

- for short bursts in code you control, call `cede()` about once every
  millisecond of computation (a call costs ~1µs): it yields the virtual thread
  back to the scheduler and checks the interrupt flag, making the loop both
  fair and cancellable;
- for long-running computations, or code that can't be instrumented with
  yields (a third-party library), wrap the call in `computeIntensive`: the
  computation runs on a pool of platform threads — which the OS preempts —
  while the calling virtual thread blocks until the result is available.

```scala
import ox.*

def expensive(): BigInt = (1 to 1_000_000).map(BigInt(_)).product

supervised:
  val f1 = fork(computeIntensive(expensive()))
  val f2 = fork(computeIntensive(expensive()))
  f1.join() + f2.join()
```

Because the caller blocks, `computeIntensive` stays structured — the
computation never outlives the enclosing scope — and composes with `fork`,
`par`, `race`, and `mapPar`. On cancellation the pool task is interrupted and
the caller waits for it to complete, so computations should still call
`checkInterrupt()` (or `cede()`) periodically where possible — a
non-cooperating computation delays the scope's shutdown until it finishes, and
a `timeout` around it overshoots accordingly.

> **Warning:** Scope context does not propagate into the computation: `fork`
> inside `computeIntensive` fails, `ForkLocal`s read their defaults, and
> thread-local integrations (MDC, OpenTelemetry context — see [OpenTelemetry
> Observability](510-opentelemetry-observability.md)) do not cross the
> boundary. Compute the value on the pool; do the forking, logging context,
> and tracing on the virtual-thread side.

## Scope propagation

Prefer local, focused scopes. If a method only needs concurrency to compute its
own result, create a `supervised` scope inside the method, start forks there,
and join before returning. Accept `(using Ox)` only when the fork or resource
must attach to the caller's lifetime.

Do not put `(using Ox)` on a constructor or on a class just because
construction starts a worker; use a factory like `Worker.start(...)(using Ox)`
and return a plain value.

```scala
// Avoid — leaks the caller's scope and exposes accidental concurrency:
def loadBoth(userId: UserId, accountId: AccountId)(using Ox): (Fork[User], Fork[Account]) =
  (fork(loadUser(userId)), fork(loadAccount(accountId)))

// Prefer — concurrency is contained and joined before returning:
def loadBoth(userId: UserId, accountId: AccountId): (User, Account) =
  supervised:
    val user = fork(loadUser(userId))
    val account = fork(loadAccount(accountId))
    (user.join(), account.join())
```

> **Important:** `(using Ox)` in a method signature means "I will start
> forks or register resources in your scope." If the method only registers
> resources and starts no forks, declare the narrower `(using ResourceScope)`
> instead (see [Resource Management](100-resource-management.md)); if it
> manages its own concurrency lifecycle, use a local `supervised` block.

## Choosing the right pattern

Prefer the highest-level primitive that fits the shape of the problem:

| Pattern | Use when |
|---------|----------|
| **Flow** | Processing collections or streams with mapping, merging, stateful transforms, backpressure, or bounded parallelism. |
| **par** | Running a small fixed number of independent computations and returning their results together. |
| **Channel** | Modeling an explicit protocol, mailbox, producer-consumer queue, or callback boundary. |
| **Actor** | Multiple concurrent callers must access one mutable object serially. |
| **AtomicReference** | A single shared value needs pure atomic updates; never use it for lifecycle flags. |
| **computeIntensive** | Long or non-instrumentable CPU-bound work that must not starve virtual threads. |
