# Subprocesses and External Streams

Driving a long-lived subprocess (or any blocking external connection — a raw
socket, an SSE stream) whose output you consume line by line, under structured
concurrency. The reader runs as a fork; its result is the fork's return value;
and teardown must account for a subprocess pipe read not being interruptible.
This is where structured concurrency is easiest to get subtly wrong — getting
the ownership and teardown right *before* writing the code avoids a deadlock that
only shows up at shutdown.

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `supervised`, `fork`, `forkDiscard`,
  `abandonOnInterruptReads`

---

## Own the work with a scope; return a result, not a live handle

Decide the owning scope first. A reader that consumes a process's output is
concurrent work, so it belongs to a `supervised` scope whose lifetime matches the
work: open the scope, spawn the process, fork the reader, drive it, and return
the computed value. The reader fork's **return value is the outcome** — read it
with `join()`. Don't publish the result through a shared `AtomicReference` that
another thread writes and the caller polls; there is one producer and one value.

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

The opposite shape — a constructor that spawns reader threads and returns an
object the caller drives later — is the pattern to avoid. Its lifetime no longer
matches any scope, so cancellation, error propagation, and cleanup all become
manual flags and `try`/`finally` ladders, exactly the bookkeeping structured
concurrency removes.

> **Warning:** Never return an object that owns running forks or threads, to be
> driven by the caller afterwards. Keep the work inside the scope that owns it
> and return a plain value. If the caller must interleave with the work (send
> input, react to events), pass a *consumer* into the scope (`run(...)(use:
> Handle => T)`) rather than handing a live handle out of it.

## Teardown: a pipe read is not interruptible — destroy before the join

A fork blocked reading a classic `java.io` stream — a subprocess's stdout pipe,
a `FileInputStream`, stdin — does **not** observe `Thread.interrupt`. This is
intended, permanent JVM behavior (the request to change it was closed as "Won't
Fix"; see [which operations are
interruptible](https://ox.softwaremill.com/latest/structured-concurrency/interruptions.html)
in the Ox docs). Ox ends a scope by interrupting its forks and then joining
them, so interruption alone will not stop such a fork, and the join blocks
forever. For a subprocess, what unblocks the read is destroying the process:
that closes the pipe's write-end, and the read returns EOF. Streams that don't
support an asynchronous close — stdin, `FileInputStream` — can't be unblocked
this way at all; wrap those with `abandonOnInterruptReads` (below).

Blocking *socket* reads are different: on virtual threads — which Ox forks run
on — they are interruptible since Java 21, though destructively (the interrupt
closes the socket). A socket-backed reader is therefore ended by scope
cancellation on its own; the deadlock below is specific to pipe and other
classic stream reads.

That close must happen **before** the scope joins the fork. Ox releases
scope-managed resources (`useInScope`, `releaseAfterScope`) only *after* all
forks complete — so, as the Ox docs note, they won't unblock a fork stuck in
uninterruptible I/O: the join has already deadlocked by the time the finalizer
would run. Put the destroy in your own body `finally` (as above) — it is
sequenced before the scope joins the forks. On the normal path the process has
already exited and the destroy is a no-op; on cancellation or error it is what
lets the scope complete.

> **Required:** Close or destroy a blocking external resource in the scope
> BODY's `finally`, before the scope joins the reader fork. Do NOT rely on
> `releaseAfterScope` for it — that finalizer runs *after* the join and will
> deadlock on a fork stuck in a non-interruptible read.

## Kill the whole process tree, not just the direct child

If the process you spawned is a launcher or wrapper that forks the real worker
(`some-launcher run the-tool …`), destroying only your direct child orphans the
worker — and the orphan keeps the inherited stdout/stderr pipe write-ends open.
A POSIX pipe reports EOF only once *every* write-end is closed, so the reader
fork never unblocks and the join still hangs. Destroy descendants first, then the
root:

```scala
val handle = process.toHandle
handle.descendants().forEach(_.destroyForcibly().discard)
handle.destroyForcibly().discard
```

> **Warning:** A forked grandchild inherits the pipe file descriptors. If it
> survives the kill, the reader never sees EOF — so terminate the descendants,
> not just the process you directly spawned.

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
subprocess, the body-`finally` destroy remains the primary teardown: it
terminates the child *and* EOFs the pipe, leaving nothing behind. Wrapping the
process's stdout (`process.getInputStream`) as well makes the reader fork
immune to a mis-sequenced teardown (scope cancellation can then always end
it) — but it never replaces destroying the process.

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

The drain fork swallows `NonFatal` so a stray read error can't tear down the
scope, and logs rather than discards silently so a real failure stays
diagnosable. It needs no separate stop signal: the body-`finally` destroy EOFs
its read, and the scope then joins it.
