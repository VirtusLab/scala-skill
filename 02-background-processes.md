# Background Processes

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `supervised`, `fork`, `forkUser`,
  `forever`, `sleep`, `never`

---

## Application entry point

`OxApp` provides the application's root scope — a `supervised` block that
manages all forks and resources:

```scala
object Main extends OxApp.Simple:
  InheritableMDC.init

  override def run(using Ox): Unit =
    val deps = Dependencies.create
    deps.emailService.startProcesses()
    deps.httpApi.start().discard
    never
```

`never` blocks the main fork indefinitely. Without it, the `run` method would
return, the scope would end, and all forked processes (HTTP server, email
sender) would be interrupted.

`OxApp` handles SIGINT/SIGTERM by interrupting the root scope, which triggers
orderly shutdown: all forks are interrupted, resources are released in reverse
order.

## Daemon forks with forever

Background processes use `fork` + `forever` + `sleep` to create periodic loops:

```scala
private def foreverPeriodically(errorMsg: String)(t: => Unit)(using Ox): Fork[Nothing] =
  fork {
    forever {
      sleep(config.emailSendInterval)
      try t
      catch case NonFatal(e) => logger.error(errorMsg, e)
    }
  }
```

The interval (`config.emailSendInterval`) comes from the service's own
configuration — each background process can define its own schedule.

`fork` creates a daemon fork — it doesn't prevent the scope from ending (only
user forks do). `forever` repeats the block indefinitely.

The `try`/`catch` around the body is important: without it, a single exception
would crash the fork and — because it's supervised — terminate the entire
application. By catching `NonFatal` exceptions, the fork logs the error and
continues to the next iteration.

## Starting multiple background processes

Multiple processes are started by forking multiple times within the same scope:

```scala
def startProcesses()(using Ox): Unit =
  foreverPeriodically("Exception when sending emails") {
    sendBatch()
  }.discard

  foreverPeriodically("Exception when counting emails") {
    val count = db.transact(emailModel.count())
    metrics.emailQueueGauge.set(count.toDouble)
  }.discard
```

Background processes use `fork` — a supervised daemon fork. If the process fails
unexpectedly (after the `try`/`catch`), the application terminates rather than
silently continuing without it. Unlike `forkUser`, a daemon fork doesn't prevent
the scope from ending.

## The HTTP server as a fork

The Netty HTTP server is also started within the scope:

```scala
def start()(using Ox): NettySyncServerBinding =
  NettySyncServer(serverOptions, NettyConfig.default.host(config.host).port(config.port))
    .addEndpoints(allEndpoints)
    .start()
```

`NettySyncServer.start()` takes `(using Ox)` — it internally forks a user thread
to accept connections. When the scope ends, the server is stopped gracefully.
