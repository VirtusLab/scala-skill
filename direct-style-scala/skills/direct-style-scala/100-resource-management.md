# Resource Management

## Dependencies

- `"com.softwaremill.ox" %% "core"` — `useInScope`, `useCloseableInScope`,
  `resourceScope`

---

## useInScope and useCloseableInScope

`useInScope(acquire)(release)` registers a resource and its release function
within the current scope. `useCloseableInScope` is a shorthand for
`AutoCloseable` resources. Resources are released in reverse acquisition order
when the scope ends.

## Application resource setup

The `Dependencies.create` method acquires all resources within the application's
root scope. It only registers resources — no forks — so it declares
`using ResourceScope` rather than `using Ox` (see below); the root `Ox` scope
satisfies it:

```scala
def create(using ResourceScope): Dependencies =
  val config = Config.read.tap(Config.log)
  val otel = initializeOtel()
  val sttpBackend = useInScope(
    Slf4jLoggingBackend(
      OpenTelemetryMetricsBackend(
        OpenTelemetryTracingBackend(HttpClientSyncBackend(), otel),
        otel
      )
    )
  )(_.close())
  val db: DB = useCloseableInScope(DB.createTestMigrate(config.db))

  create(config, otel, sttpBackend, db, DefaultClock)
```

On scope termination (e.g., SIGTERM via `OxApp` — see [Background
Processes](110-background-processes.md)), resources are released in reverse
acquisition order: first the database pool, then the sttp backend.

## Resources without concurrency: resourceScope

When a method only needs scoped cleanup — no forks — don't open a `supervised`
scope just to get `useInScope`: that claims a concurrency capability the code
doesn't use. `resourceScope` (Ox ≥ 1.0.6) starts a dedicated, resource-only
scope, Ox's analogue of `scala.util.Using.Manager`:

```scala
import ox.{resourceScope, useCloseableInScope}
import java.io.{FileReader, FileWriter}

def process(): Unit = resourceScope:
  val in = useCloseableInScope(FileReader("in.txt"))
  val out = useCloseableInScope(FileWriter("out.txt"))
  // both closed when the scope completes, out first
  out.write(in.read())
```

A resource scope can only be started where no concurrency scope is lexically
visible — verified at compile time, because a fork started inside a visible
resource scope could outlive it, using resources after they have been
released. Within a concurrency scope, register
resources directly instead; to use a resource scope e.g. in a fork's body,
extract it to a method that doesn't take `using Ox` — good practice in itself.

Methods that only register resources should declare exactly that capability:
`using ResourceScope` instead of `using Ox`. Every concurrency scope is also a
`ResourceScope`, so such a method can be called from both — while its
signature no longer claims the ability to fork.
