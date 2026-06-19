---
name: direct-style-scala
description: Scala coding style, tooling, and functional programming guidance, with dedicated sections on direct-style Scala, Ox structured concurrency, and synchronous Tapir. Auto-load for any task involving Scala code, especially when using direct-style or "plain" Scala.
---

You are an expert backend software engineer and architect.

# Scala tooling

* ALWAYS use tools to compile and run tests instead of relying on bash commands
* after adding a dependency to `build.sbt`, ALWAYS run the `import-build` tool
* to lookup a dependency or the latest version, use the `find-dep` tool
* to lookup the API of a class, use the `inspect` tool. To lookup the docs or
  usages, use the `get-docs` and `get-usages` tools
* to compile the project, use `compile-full`, `compile-module` tools
* to search for symbols, use `glob-search` and `typed-glob-search` tools
* if you do need to use `sbt`, use `sbt --client` instead of `sbt` to connect to
  a running sbt server for faster execution
* to verify that the app starts use `sbt run`, WITHOUT `--client`, as it
  prevents interrupting the process
* before committing, ALWAYS format all changed Scala files using the sbt
  `scalafmt` plugin: `sbt --client scalafmtAll`
* the project MUST compile with zero warnings. Ensure `build.sbt` includes
  `-Wunused:all`, `-Wvalue-discard`, `-Wnonunit-statement` in `scalacOptions`.
  Fix warnings in code; only use `@nowarn` for generated code or unfixable
  third-party issues (with a comment explaining why).

# Coding style

* ALWAYS use braceless syntax — do not use `{}`
* before creating or moving a `.scala` file, decide its package, filename, and
  top-level visibility. Read [Code Organization and
  Visibility](160-code-organization.md) when adding packages/modules or
  widening visibility.
* when dealing with resources, properly track who owns which resources, and
  ensure proper ordering on cleanup
* every top-level class, trait, enum, and object MUST have intentional
  visibility at declaration time: default-public (no modifier),
  `private[<subpkg>]`, or `private[<rootpkg>]`. Choose by scanning actual call
  sites.
* comment on any aspects that aren't obvious from the implementation, but are
  important to know when reading the code
* each function MUST handle exactly one concern — extract each step (validate,
  transform, persist, …) into a well-named function so an orchestrator reads as a
  sequence of intentions; naming a single-use step is reason enough to extract.
* use Ox's `.pipe` and `.tap` (`import ox.*`) to drop single-use `val`s that
  only feed the next line. `.pipe(f)` returns `f(value)`; `.tap(f)` runs a side
  effect and returns the value unchanged. Keep a named `val` when the name
  documents intent or the value is reused
* tests MUST be targeted — each covers exactly one scenario, with no overlap.
* every public function, val, and given MUST have an explicit return type — this
  prevents accidental signature drift during refactors.

# Performance

* NEVER materialize unbounded data into memory. Use streaming with `Flow` or
  paging to process large datasets and paginated API results incrementally.

# Direct-style Scala

* in Tapir, use `.handle` / `.handleSecurity` / `.handleSuccess` to wire
  endpoint logic — NEVER use `.serverLogic` / `.serverSecurityLogic`. The
  `.handle` family is the direct-style API. The `.serverLogic` family requires
  a monadic wrapper (`Future`, `IO`) and MUST NOT be used.
* ALWAYS use Ox for threading, channels, and async coordination. Avoid raw
  `Thread.ofVirtual`, `LinkedBlockingQueue`, `synchronized`/`Lock`, and
  lifecycle flags. Use `java.util.concurrent` coordination primitives only for
  pure atomic state or when bridging a foreign API that Ox does not cover.
* create local, focused `supervised` scopes for request-, message-, or
  job-level concurrency. Accept a parent `(using Ox)` only when a fork or
  resource must be tied to that parent scope's lifetime.
* keep constructors plain; use factories that take `(using Ox)` and return
  values that do not carry the capability.

# Functional programming

* use pure functions, immutable data, higher-order functions, ADTs. NEVER use
  shared mutable state.
* `var` declarations MUST be inside methods (e.g. processing loops), **never**
  as class fields. Class-level `var`s break reasoning and testability. Use only
  immutable collections (`Map`, `Set`, `List`) — never `mutable.Map`,
  `mutable.Set`, `mutable.Buffer`.
* model state as an immutable case class. State transitions are pure functions
  that take the current state and return a new one via `.copy()`. Confine the
  `var` that threads state to the smallest possible scope.
* push side effects behind traits so that state transitions are testable without
  real infrastructure. Tests substitute in-memory implementations — mutable
  collections are acceptable in test helpers that simulate external systems.
* APIs MUST be **lawful**: given identical arguments and explicit dependencies,
  they yield the same observable result. Do not hide dependencies like `Clock`,
  `Random`, or `UUID` inside methods — pass them explicitly or capture them in
  the class constructor.
* wrap `String`, `Int`, `Long`, and `Boolean` domain values in opaque types or
  enums — NEVER use raw primitives for domain concepts. This applies to
  identifiers (`OrderId`, `ProductCode`), quantities (`Quantity`, `Amount`), and
  configuration values (`Port`, `TopicName`). When a generated library (e.g.
  scalaxb) produces raw `String` fields, introduce opaque types at the boundary
  where generated types are converted to domain types.
* eliminate boolean blindness — replace `Boolean` parameters and return values
  with two-case enums so intent is explicit and exhaustiveness is checked.
* NEVER throw exceptions for recoverable failures. Instead, return an `Either[E, T]`.
  Use exceptions only for unrecoverable errors, which should terminate the current
  processing unit (request, message handling, etc.)
* if a value can be absent, use `Option[T]` — NEVER use `null` or sentinel
  values. `Option` is for presence/absence only, not for errors.
* model different states of an entity as separate types — NEVER use `Option`
  fields to represent state transitions.
* design domain models so that invalid data CANNOT be constructed. Use enums,
  opaque types, or smart constructors to encode invariants.
* define sealed-trait or enum error hierarchies — NEVER use stringly-typed
  errors.
* NEVER use bare `try`/`catch` for recoverable failures. Reserve `try`/`catch` for
  defect or unrecoverable error boundaries only.

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
  `useCloseableInScope`, reverse-order release, scope-based cleanup.

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
  shutdown, actors for serialized mutable state.

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

- [Endpoint Input Codecs](360-endpoint-input-codecs.md) — `PlainCodec`s for
  path/query/header inputs: mapping a built-in codec onto an opaque-type id, and
  `Codec.derivedEnumeration` for enum-valued inputs.

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
