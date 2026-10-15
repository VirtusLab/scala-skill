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
  each top-level declaration's visibility (narrowest the call sites allow).
  Read [Code Organization and Visibility](160-code-organization.md) when
  adding packages/modules or widening visibility.
* each function MUST handle exactly one concern — either a single logical
  operation, or a short orchestration of named steps.
* use Ox `.pipe`/`.tap` (`import ox.*`) to drop single-use `val`s that only
  feed the next line; keep a named `val` when the name documents intent.
* tests MUST be targeted — each test covers exactly one scenario. No
  overlapping or redundant tests.
* every public function, val, and given MUST have an explicit return type — this
  prevents accidental signature drift during refactors.

# Performance

* NEVER materialize unbounded data in memory — stream with `Flow` or page.
* virtual threads are never preempted: long CPU-bound work needs
  `computeIntensive` or `cede()` — see [CPU-intensive
  work](150-shared-state-across-threads.md#cpu-intensive-work).

# Direct-style Scala

* in Tapir, wire logic with `.handle` / `.handleSecurity` / `.handleSuccess` —
  NEVER `.serverLogic*` (it needs an effect wrapper).
* ALWAYS use Ox for threading, channels, and async coordination — not raw
  threads, queues, locks, or lifecycle flags. See [Use Ox
  primitives](150-shared-state-across-threads.md#use-ox-primitives).
* decide the owning scope before writing concurrent code: prefer a local
  `supervised` per request/job; return the fork's result, never an object
  owning running forks. Read [Scope
  propagation](150-shared-state-across-threads.md#scope-propagation) before
  taking `(using Ox)` / `(using ResourceScope)` or forking from a factory.
* a fork blocked reading a subprocess pipe, stdin, a file, or other classic
  `java.io` stream is NOT ended by scope cancellation — shutdown deadlocks.
  BEFORE driving a subprocess or reading a blocking external stream (socket,
  SSE), read [Subprocesses and External
  Streams](170-subprocesses-and-external-streams.md).

# Functional programming

Worked examples: [Domain Modeling](180-domain-modeling.md).

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
  `state => state` functions; confine the `var` to the smallest scope.
* APIs MUST be lawful: pass `Clock`, `Random`, and UUID generation explicitly
  (parameter or constructor) — never call them inside methods.
* wrap `String`, `Int`, `Long`, and `Boolean` domain values in opaque types or
  enums — NEVER use raw primitives for domain concepts (identifiers,
  quantities, configuration values). Enforce invariants with smart constructors
  returning `Either` (not `require`).
* replace `Boolean` parameters and return values with two-case enums.
* recoverable failures: return `Either[E, T]` — never throw, `require`, or bare
  `try`/`catch`. Exceptions only for unrecoverable errors that end the current
  request/message.
* define sealed-trait or enum error hierarchies — NEVER use stringly-typed
  errors.

# Use-Case Guide

BEFORE writing code that uses Tapir, Ox, or sttp, or touches a topic listed
below, read the relevant chapter(s) in full. Follow every API and `> Required` /
`> Important` callout exactly — do NOT substitute another approach. Read the
chapter file from this skill's directory, or with a raw HTTP GET from the base
URL below — never via a tool that summarises the page (summaries drop callouts
and exact API calls).
https://raw.githubusercontent.com/virtuslab/scala-skill/refs/heads/master/direct-style-scala/skills/direct-style-scala/

## Application Structure

- [New Project Setup](140-new-project-setup.md) — sbt + Ox skeleton, `build.sbt`, `scalacOptions`, `OxApp.Simple`, adopt-tapir
- [Code Organization and Visibility](160-code-organization.md) — top-level visibility, file naming, package shadowing, sbt/Scalafix boundaries
- [Domain Modeling](180-domain-modeling.md) — immutable state, lawful APIs (`Clock`, UUID), opaque types, smart constructors, enums vs `Boolean`
- [Resource Management](100-resource-management.md) — `useInScope`, `useCloseableInScope`, `resourceScope`, `ResourceScope`, release order
- [Background Processes](110-background-processes.md) — `OxApp`, `forkDiscard`/`forkUserDiscard`, `forever`/`sleep` loops, orderly shutdown
- [Type-Safe Configuration](120-type-safe-configuration.md) — PureConfig `derives ConfigReader`, env overrides, `Sensitive`, load-time validation
- [Compile-Time Dependency Injection](130-compile-time-dependency-injection.md) — MacWire `autowire`, `autowireMembersOf`, `wireList`
- [Concurrency and Inter-Thread Communication](150-shared-state-across-threads.md) — Flow `mapPar`/`merge`/`mapStateful`, `par`, channels, `Actor`, `AtomicReference`, `computeIntensive`/`cede`, `(using Ox)` scope propagation
- [Subprocesses and External Streams](170-subprocesses-and-external-streams.md) — subprocess/socket/SSE reader forks, destroy before join, process-tree teardown, `abandonOnInterruptReads`, pipe back-pressure

## Error Handling

- [Error Handling](200-error-handling.md) — `Fail` ADT, `either`/`.ok()`, `transactEither`, `.catching`, nesting rules
- [Error Output Customisation](210-error-output-customisation.md) — JSON error responses, `Fail` → status code mapping, `failOutput`, `defaultHandlers`
- [Decode Failure Handling](220-decode-failure-handling.md) — `DefaultDecodeFailureHandler`, `onDecodeFailureNextEndpoint`, custom messages, `hideEndpointsWithAuth`

## HTTP & Endpoints

- [Authentication](300-authentication.md) — `secureEndpoint[T]`, `AuthTokenOps[T]`, `Auth[T]`, `handleSecurity`
- [HTTP Server Configuration](310-http-server-configuration.md) — `Identity` effect, security headers, CORS, SPA static files, request cancellation, `NettySyncServer`
- [Version API](320-version-api.md) — `sbt-buildinfo`, `BuildInfo` git hash endpoint
- [Compile-Time OpenAPI Generation](330-compile-time-openapi-generation.md) — build-time OpenAPI YAML, `EndpointsForDocs`, `@main` generator, sbt task
- [SOAP with scalaxb](340-soap-with-scalaxb.md) — XSD codegen, SOAP envelopes, Tapir XML codecs, `SOAPAction` routing, SOAP faults
- [JSON Request and Response Bodies](350-json-bodies.md) — jsoniter codecs, list-body codecs, enums as strings via `withDiscriminatorFieldName(None)`, opaque types in DTOs
- [Endpoint Inputs](360-endpoint-inputs.md) — path/query/header `PlainCodec`s, opaque-type ids, `Codec.derivedEnumeration`

## Data & Integration

- [SQL Persistence](400-sql-persistence.md) — Magnum, `@Table`, `DbCodec`, `Repo`/`TableInfo`, `sql` interpolation, Flyway, HikariCP
- [Sending Emails](410-sending-emails.md) — `EmailScheduler`, SMTP/Mailgun/dummy senders, templates, batch sending
- [Kafka Streaming](420-kafka-streaming.md) — `KafkaFlow.subscribe`, `mapPar`, `KafkaDrain`, offset commits, transactional produce-and-commit, graceful shutdown

## Testing & Observability

- [Testing HTTP Endpoints](500-testing-http-endpoints.md) — `TapirSyncStubInterpreter`, `SttpClientInterpreter`, public and secured endpoints in-process
- [OpenTelemetry Observability](510-opentelemetry-observability.md) — SDK auto-config, Tapir/sttp instrumentation, custom metrics, `PropagatingVirtualThreadFactory`, MDC correlation
