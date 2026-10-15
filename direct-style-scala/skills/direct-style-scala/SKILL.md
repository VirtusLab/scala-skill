---
name: direct-style-scala
description: Scala coding style, tooling, and functional programming guidance, with dedicated sections on direct-style Scala, Ox structured concurrency, and synchronous Tapir. Auto-load for any task involving Scala code, especially when using direct-style or "plain" Scala.
---

# Scala tooling

* Drive the build with sbt commands. Projects use sbt 2, which differs from sbt 1:
  * `sbt` connects to a background server by default; don't add `--client`. The
    server keeps the environment of its first start; `sbt shutdown` to reset it.
  * multiple commands must be one quoted argument: `sbt "clean ; compile ; test"`
  * `test` only runs failed, new or affected tests; `testFull` runs all
  * `sbt run` starts the app in a separate JVM; killing sbt doesn't stop it, kill
    that JVM. `unknown event: sbt/clientJob` means the sbt runner is too old.
  * `build.sbt` is Scala 3: no postfix calls, slash syntax only (`Test / compile`).
    Bare settings apply to all subprojects (no `ThisBuild /`). Tasks are cached:
    wrap side-effecting custom tasks in `Def.uncached`.
  * outputs are in `target/out/jvm/scala-<version>/<project>/`
* resolve dependency versions from
  `https://repo1.maven.org/maven2/<group-with-slashes>/<artifact>/maven-metadata.xml`,
  including the suffix (`com/softwaremill/ox/core_3`; sbt plugins:
  `_sbt2_3`). Take the latest stable `<version>`; `<release>` may be a prerelease
  (`RC`, `M<n>`, `SNAP`, `alpha`, `beta`, `NIGHTLY`). NEVER use
  `search.maven.org/solrsearch` (stale by months). Find unknown coordinates via
  `https://index.scala-lang.org/api/autocomplete?q=<name>`.
* before committing, ALWAYS run `sbt scalafmtAll`
* the project MUST compile with zero warnings. Ensure `build.sbt` includes
  `-Wunused:all`, `-Wvalue-discard`, `-Wnonunit-statement` in `scalacOptions`.
  Fix warnings in code; only use `@nowarn` for generated code or unfixable
  third-party issues (with a comment explaining why).

# Coding style

* ALWAYS use braceless syntax — do not use `{}`
* responsibilities in code MUST be segregated between appropriately named
  entities
* before creating or moving a `.scala` file, decide its package, filename, and
  top-level visibility. Read [Code Organization and
  Visibility](160-code-organization.md) when adding packages/modules or
  widening visibility.
* every top-level class, trait, enum, and object MUST have intentional
  visibility at declaration time: default-public (no modifier),
  `private[<subpkg>]`, or `private[<rootpkg>]`. Choose by scanning actual call
  sites.
* each function MUST handle exactly one concern — either a single logical
  operation, or a short orchestration of named steps.
* use Ox `.pipe`/`.tap` (`import ox.*`) to drop single-use `val`s that only
  feed the next line; keep a named `val` when the name documents intent.
* tests MUST be targeted — each test covers exactly one scenario. No
  overlapping or redundant tests.
* every public function, val, and given MUST have an explicit return type — this
  prevents accidental signature drift during refactors:

```scala
// Wrong — inferred return type can silently change:
def findUser(id: Id[User])(using DbTx) =
  userModel.findById(id).toRight(Fail.NotFound("user"))

// Right — return type is explicit and stable:
def findUser(id: Id[User])(using DbTx): Either[Fail, User] =
  userModel.findById(id).toRight(Fail.NotFound("user"))
```

# Performance

* NEVER materialize unbounded data in memory — stream with `Flow` or page.
* virtual threads are never preempted: run long CPU-bound work via
  `computeIntensive`; in CPU loops you control, call `cede()` about once per ms.

# Direct-style Scala

* in Tapir, wire logic with `.handle` / `.handleSecurity` / `.handleSuccess` —
  NEVER `.serverLogic*` (it needs an effect wrapper).
* ALWAYS use Ox for threading, channels, and async coordination. Avoid raw
  `Thread.ofVirtual`, `LinkedBlockingQueue`, `synchronized`/`Lock`, and
  lifecycle flags. Use `java.util.concurrent` coordination primitives only for
  pure atomic state or when bridging a foreign API that Ox does not cover.
* prefer local `supervised` scopes per request/job; take `(using Ox)` only when
  a fork or resource must live as long as the caller's scope.
* keep constructors plain. If a factory only registers resources (no forks),
  take `(using ResourceScope)`; for cleanup without concurrency use
  `resourceScope`, not `supervised`.
* decide the owning scope before writing concurrent code; a worker's result is
  its fork's return value. NEVER return an object that owns running forks to be
  driven later — pass a consumer into the scope instead.
* a fork blocked reading a subprocess pipe, stdin, a file, or any other
  classic `java.io` stream is NOT ended by scope cancellation — without the
  right teardown shape, shutdown deadlocks. BEFORE writing code that drives a
  subprocess or reads a blocking external stream (socket, SSE), read
  [Subprocesses and External
  Streams](170-subprocesses-and-external-streams.md).

# Functional programming

* use pure functions, immutable data, higher-order functions, ADTs. NEVER use
  shared mutable state.
* `var` declarations MUST be inside methods (e.g. processing loops), **never**
  as class fields. Class-level `var`s break reasoning and testability. The sole
  exception is mutable state encapsulated by an Ox `Actor`, which serialises every
  invocation onto a single thread — the actor is what makes the field safe to hold
  (see [Concurrency and Inter-Thread Communication](150-shared-state-across-threads.md)).
  Use only immutable collections (`Map`, `Set`, `List`) — never `mutable.Map`,
  `mutable.Set`, `mutable.Buffer` (mutable collections are fine in test fakes).
* model state as an immutable case class; transitions are pure
  `state => state` functions; confine the `var` to the smallest scope:

```scala
case class ProcessingState(
    processed: Map[String, Long] = Map.empty,
    pending: Set[String] = Set.empty
)

def handleItem(state: ProcessingState, item: Item): ProcessingState =
  state.copy(processed = state.processed.updated(item.key, item.offset))

def run(items: Iterator[Item]): ProcessingState =
  var state = ProcessingState()
  for item <- items do
    state = handleItem(state, item)
  state
```

* APIs MUST be lawful: pass `Clock`, `Random`, and UUID generation explicitly
  (parameter or constructor) — never call them inside methods:

```scala
// Wrong — hidden non-determinism:
class OrderService:
  def place(order: Order): Confirmation =
    val id = UUID.randomUUID()
    val now = Instant.now()
    Confirmation(id, now)

// Right — dependencies are explicit and injectable:
class OrderService(clock: Clock, idGenerator: () => UUID):
  def place(order: Order): Confirmation =
    val id = idGenerator()
    val now = clock.instant()
    Confirmation(id, now)
```

* wrap `String`, `Int`, `Long`, and `Boolean` domain values in opaque types or
  enums — NEVER use raw primitives for domain concepts. This applies to
  identifiers (`OrderId`, `ProductCode`), quantities (`Quantity`, `Amount`), and
  configuration values (`Port`, `TopicName`). When a generated library (e.g.
  scalaxb) produces raw `String` fields, introduce opaque types at the boundary
  where generated types are converted to domain types; enforce invariants with
  smart constructors returning `Either` (not `require`).
* replace `Boolean` parameters and return values with two-case enums:

```scala
// Wrong — caller must remember what `true` means:
def recordFlush(success: Boolean, durationMs: Double): Unit

// Right — intent is unambiguous:
enum FlushOutcome:
  case Success, Failure
def recordFlush(outcome: FlushOutcome, duration: Duration): Unit
```

* recoverable failures: return `Either[E, T]` — never throw, `require`, or bare
  `try`/`catch`. Exceptions only for unrecoverable errors that end the current
  request/message.
* define sealed-trait or enum error hierarchies — NEVER use stringly-typed
  errors.

# Use-Case Guide

BEFORE writing any code that uses Tapir, Ox, sttp, or direct-style Scala, you MUST 
fetch the chapter(s) relevant to your current task from this guide and follow the 
patterns shown there. This is not optional — code that ignores guide patterns will 
be rejected in review.

Retrieve the chapter as raw, unmodified text — read every code block and
paragraph in full. Do NOT use a tool that summarises the page: summaries silently
drop the `> Required` / `> Important` callouts and the exact API calls that make
the chapter correct. Prefer reading the chapter file directly from the installed
skill directory; if fetching over the network, use a method that returns the
verbatim file (a raw HTTP GET), not a fetch-and-summarise tool.

Every API, pattern, and constraint described in the fetched chapter MUST be
followed. If the chapter says to use a specific API (e.g. `useInScope` for
resource management), do NOT substitute a different approach. If the chapter
marks something as required, it is required.

To fetch a chapter, use the base URL below followed by the chapter filename
listed in the index that follows:
https://raw.githubusercontent.com/virtuslab/scala-skill/refs/heads/master/direct-style-scala/skills/direct-style-scala/

## Application Structure

- [New Project Setup](140-new-project-setup.md) — minimal direct-style
  Scala project skeleton with sbt and Ox: directory layout, `build.sbt`,
  required `scalacOptions`, `OxApp.Simple` entry point. adopt-tapir as a
  starting point for HTTP projects.

- [Code Organization and Visibility](160-code-organization.md) — top-level
  visibility, file naming exceptions, Scala 3 package shadowing, and
  sbt/Scalafix boundary enforcement.

- [Resource Management](100-resource-management.md) — `useInScope`,
  `useCloseableInScope`, reverse-order release, scope-based cleanup;
  `resourceScope` for cleanup without concurrency, `using ResourceScope` as
  the narrower capability.

- [Background Processes](110-background-processes.md) — `OxApp` entry point,
  `forkDiscard`/`forkUserDiscard` for daemon vs. user threads,
  `forever`/`sleep` for periodic loops, orderly shutdown.

- [Type-Safe Configuration](120-type-safe-configuration.md) — PureConfig with
  `derives ConfigReader`, environment variable overrides, `Sensitive` wrapper,
  load-time validation.

- [Compile-Time Dependency Injection](130-compile-time-dependency-injection.md)
  — MacWire `autowire`, `autowireMembersOf` for config extraction, `wireList`
  for collecting endpoints.

- [Concurrency and Inter-Thread Communication](150-shared-state-across-threads.md)
  — Flows for declarative concurrent pipelines (`mapPar`, `merge`,
  `mapStateful`), Ox primitive selection, channels for worker mailboxes and
  shutdown, actors for serialized mutable state, `computeIntensive`/`cede` for
  CPU-bound work on virtual threads.

- [Subprocesses and External Streams](170-subprocesses-and-external-streams.md)
  — driving a subprocess / socket / SSE reader as a fork whose return value is
  the result; why a non-interruptible pipe read needs the resource destroyed
  in the scope body's `finally` (before the join) rather than via
  `releaseAfterScope`; process-tree teardown; `abandonOnInterruptReads` for
  reads that can't be unblocked by closing; pipe back-pressure.

## Error Handling

- [Error Handling](200-error-handling.md) — `Fail` ADT, Ox `either` blocks with
  `.ok()` short-circuiting, `transactEither`, `.catching`, nesting rules.

- [Error Output Customisation](210-error-output-customisation.md) — JSON error
  responses for all error types. Bidirectional `Fail` → HTTP status code
  mapping, `failOutput`, `defaultHandlers` for decode failures and 404s.

- [Decode Failure Handling](220-decode-failure-handling.md) —
  `DefaultDecodeFailureHandler` customisation: respond/message/response pipeline,
  `onDecodeFailureNextEndpoint`, custom failure messages,
  `hideEndpointsWithAuth`.

## HTTP & Endpoints

- [Authentication](300-authentication.md) — `secureEndpoint[T]`,
  `AuthTokenOps[T]` trait, `Auth[T]` authenticator, `handleSecurity` wiring.

- [HTTP Server Configuration](310-http-server-configuration.md) — Security
  headers, CORS, serving static files for SPAs, request cancellation,
  `NettySyncServer` startup.

- [Version API](320-version-api.md) — `sbt-buildinfo` generating `BuildInfo`
  with git commit hash, served from a Tapir endpoint.

- [Compile-Time OpenAPI Generation](330-compile-time-openapi-generation.md) —
  Build-time OpenAPI YAML generation for frontend client codegen (not runtime
  Swagger UI). `EndpointsForDocs`, `@main` generator, sbt task wiring.

- [SOAP with scalaxb](340-soap-with-scalaxb.md) — XSD-to-Scala code generation,
  SOAP envelope wrapping/unwrapping, Tapir XML codecs for scalaxb types,
  `SOAPAction`-based endpoint routing, SOAP fault error handlers.

- [JSON Request and Response Bodies](350-json-bodies.md) — jsoniter codec
  derivation for DTOs, why list bodies need their own codec, encoding
  parameterless enums as plain strings via `withDiscriminatorFieldName(None)`,
  and using opaque-type identifiers directly in DTOs.

- [Endpoint Inputs](360-endpoint-inputs.md) — `PlainCodec`s for
  path/query/header inputs: mapping a built-in codec onto an opaque-type id,
  `Codec.derivedEnumeration` for enum-valued inputs, and how multiple inputs reach
  the handler.

## Data & Integration

- [SQL Persistence](400-sql-persistence.md) — Magnum with PostgreSQL: `@Table`
  case classes, `DbCodec` for opaque types, `Repo`/`TableInfo`, `sql`
  interpolation, Flyway migrations, HikariCP.

- [Sending Emails](410-sending-emails.md) — `EmailScheduler` trait, pluggable
  senders (SMTP, Mailgun, dummy), email templates, background batch processing.

- [Kafka Streaming](420-kafka-streaming.md) — `KafkaFlow.subscribe`, `mapPar`,
  `KafkaDrain` publishing, offset commits, transactional produce-and-commit,
  graceful shutdown.

## Testing & Observability

- [Testing HTTP Endpoints](500-testing-http-endpoints.md) —
  `TapirSyncStubInterpreter` stub backend, `SttpClientInterpreter` for
  type-safe requests, testing public and secured endpoints in-process.

- [OpenTelemetry Observability](510-opentelemetry-observability.md) — SDK
  auto-configuration, Tapir tracing/metrics interceptors, sttp client
  instrumentation, custom metrics, `PropagatingVirtualThreadFactory` for
  context propagation, MDC log correlation.
