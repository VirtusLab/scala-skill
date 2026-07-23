# Subprocesses and External Streams

Driving a long-lived subprocess (or any blocking external connection — a raw
socket, an SSE stream) whose output you consume line by line, under structured
concurrency. Getting the ownership and teardown shape right *before* writing
the code avoids a deadlock that only shows up at shutdown.

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `supervised`, `fork`, `forkDiscard`,
  `abandonOnInterruptReads`

---

## Own the work with a scope; return a result, not a live handle

Decide the owning scope first. A reader that consumes a process's output is
concurrent work, so it belongs to a `supervised` scope whose lifetime matches
the work. The reader fork's **return value is the outcome** — read it with
`join()`; don't publish it through a shared `AtomicReference` that another
thread writes and the caller polls. There is one producer and one value.

```scala
def runAndCollect(command: Seq[String]): Summary =
  supervised:
    val process = ProcessBuilder(command*).start()
    try
      val reader = fork:
        scala.io.Source
          .fromInputStream(process.getInputStream)
          .getLines()
          .foldLeft(Summary.empty)(_.add(_)) // the fork returns the Summary
      reader.join()                          // its return value IS the result
    finally process.destroyForcibly().discard // see the next section
```

> **Warning:** Never return an object that owns running forks or threads, to
> be driven by the caller afterwards — its lifetime then matches no scope, so
> cancellation, error propagation, and cleanup become manual flags and
> `try`/`finally` ladders, exactly the bookkeeping structured concurrency
> removes. Keep the work inside the scope and return a plain value; if the
> caller must interleave with the work (send input, react to events), pass a
> *consumer* into the scope (`run(...)(use: Handle => T)`) rather than handing
> a live handle out of it.

## Teardown: a pipe read is not interruptible — destroy before the join

A fork blocked reading a classic `java.io` stream — a subprocess's stdout
pipe, a `FileInputStream`, stdin — does **not** observe `Thread.interrupt`,
permanently and by design (see [which operations are
interruptible](https://ox.softwaremill.com/latest/structured-concurrency/interruptions.html)
in the Ox docs). Ox ends a scope by interrupting its forks and then joining
them, so interruption alone will not stop such a fork, and the join blocks
forever. For a subprocess, what unblocks the read is destroying the process:
that closes the pipe's write-end, and the read returns EOF. Streams that don't
support an asynchronous close — stdin, `FileInputStream` — can't be unblocked
this way at all; wrap those with `abandonOnInterruptReads` (below).

Blocking *socket* reads are different: on virtual threads — which Ox forks run
on — interruption works since Java 21, destructively closing the socket. A
socket-backed reader is thus ended by scope cancellation on its own; the
deadlock here is specific to pipe and other classic stream reads.

The destroy must be sequenced **before** the scope joins the fork — which is
why it sits in the scope body's own `finally` above. Ox releases scope-managed
resources (`useInScope`, `releaseAfterScope`) only *after* all forks complete,
so a kill registered there runs when the join has already deadlocked. On the
normal path the process has already exited and the destroy is a no-op; on
cancellation or error it is what lets the scope complete.

> **Required:** Close or destroy a blocking external resource in the scope
> BODY's `finally`, before the scope joins the reader fork. Do NOT rely on
> `releaseAfterScope` for it — that finalizer runs *after* the join and will
> deadlock on a fork stuck in a non-interruptible read.

## Kill the whole process tree, not just the direct child

If the process you spawned is a launcher or wrapper that forks the real worker
(`some-launcher run the-tool …`), destroying only your direct child orphans the
worker. Destroy descendants first, then the root:

```scala
val handle = process.toHandle
handle.descendants().forEach(_.destroyForcibly().discard)
handle.destroyForcibly().discard
```

> **Warning:** A forked grandchild inherits the pipe file descriptors, and a
> POSIX pipe reports EOF only once *every* write-end is closed. An orphan that
> survives the kill keeps the reader blocked forever — terminate the
> descendants, not just the process you directly spawned.

## Reads you can't unblock by closing: `abandonOnInterruptReads`

Destroy-and-EOF works because a subprocess supports asynchronous close. Some
resources don't: stdin can't be meaningfully closed, and closing a
`FileInputStream` does not unblock a pending read. And sometimes a single read
should be cancellable (a `timeout` around one read) without tearing the
resource down. For these, use `abandonOnInterruptReads` (Ox ≥ 1.0.6):
it wraps an `InputStream` so the actual reads run on a *detached* virtual
thread — unmanaged, never joined — while the calling fork awaits each chunk
interruptibly. On interruption the wait is abandoned: the fork proceeds with an
`InterruptedException`, and the in-flight chunk is not lost — the next read
returns it. With `closeOnAbandon = true`, an interrupted read instead closes
the underlying stream, and the wrapper becomes permanently closed.

```scala
import ox.*
import scala.concurrent.duration.*

// one process-wide wrapper: multiple wrappers over System.in would compete for input
lazy val stdin = abandonOnInterruptReads(System.in)

supervised:
  val firstByte: Option[Int] = timeoutOption(1.second)(stdin.read())
```

The trade-off: an abandoned read leaves the detached thread blocked until the
underlying read completes — for stdin, possibly for the application's lifetime.
That is a cheap virtual thread in exchange for interruptibility. For a
subprocess, keep the body-`finally` destroy as the primary teardown — it
terminates the child *and* EOFs the pipe, leaving nothing behind; wrapping the
process's stdout (`process.getInputStream`) additionally protects the reader
from a mis-sequenced teardown, but does not replace the destroy.

For one-off uninterruptible calls (a JDBC `execute`, a DNS lookup) there is
`abandonOnInterrupt(op)(onAbandon)`, where the cleanup starts on abandonment —
e.g. `abandonOnInterrupt(statement.execute())(connection.close())`. For
resources supporting asynchronous close, the cleanup also unblocks the
abandoned operation, so nothing is leaked. `abandonOnInterruptWrites` covers
the write side — e.g. a write to the child's stdin blocked on a full pipe.

## Don't let an unread pipe stall the process

A subprocess blocks once an OS pipe buffer (~64 KB) fills. If you read stdout but
ignore stderr, a chatty process wedges mid-run. Either let the child inherit the
parent's stderr, or drain stderr in its own fork for the resource's lifetime —
a `forkDiscard`, torn down by the same body-`finally` destroy:

```scala
supervised:
  val process = ProcessBuilder(command*).start()
  try
    forkDiscard:
      try scala.io.Source.fromInputStream(process.getErrorStream).getLines().foreach(log.debug)
      catch case NonFatal(e) => log.debug("stderr drain ended", e)
    val reader = fork(consume(process.getInputStream))
    reader.join()
  finally process.destroyForcibly().discard
```

The drain catches `NonFatal` — logged, so a real failure stays diagnosable —
because a stray read error must not tear down the scope. It needs no stop
signal: the body-`finally` destroy EOFs its read, and the scope then joins it.
