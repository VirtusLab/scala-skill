# OpenTelemetry Observability

## Dependencies

- `"io.opentelemetry" % "opentelemetry-sdk-extension-autoconfigure"` —
  auto-configures the OpenTelemetry SDK from environment variables
- `"io.opentelemetry" % "opentelemetry-exporter-otlp"` — exports traces,
  metrics, and logs via OTLP protocol
- `"io.opentelemetry" % "opentelemetry-exporter-sender-jdk"` — uses JDK's
  `HttpClient` for OTLP transport (replaces the default OkHttp sender)
- `"com.softwaremill.sttp.tapir" %% "tapir-opentelemetry-tracing"` — Tapir
  server interceptor that creates spans for incoming HTTP requests
- `"com.softwaremill.sttp.tapir" %% "tapir-opentelemetry-metrics"` — Tapir
  server interceptor that records HTTP request metrics
- `"com.softwaremill.sttp.client4" %% "opentelemetry-backend"` — wraps an sttp
  HTTP client to propagate trace context and record outgoing request metrics
- `"com.softwaremill.ox" %% "otel-context"` — propagates OpenTelemetry context
  across Ox's virtual thread scopes
- `"io.opentelemetry.instrumentation" % "opentelemetry-runtime-telemetry-java8"`
  — JVM runtime metrics (CPU, memory, GC, threads, classes)
- `"io.opentelemetry.instrumentation" % "opentelemetry-logback-appender-1.0"` —
  sends Logback log records to OpenTelemetry
- `"com.softwaremill.ox" %% "mdc-logback"` — MDC values that propagate across
  virtual threads within Ox scopes

---

## SDK initialization

The OpenTelemetry SDK is configured using the auto-configuration module, which
reads settings from environment variables (`OTEL_SERVICE_NAME`,
`OTEL_EXPORTER_OTLP_ENDPOINT`, etc.):

```scala
import io.opentelemetry.sdk.autoconfigure.AutoConfiguredOpenTelemetrySdk
import io.opentelemetry.instrumentation.runtimemetrics.java8.*
import io.opentelemetry.instrumentation.logback.appender.v1_0.OpenTelemetryAppender

private def initializeOtel(): OpenTelemetry =
  AutoConfiguredOpenTelemetrySdk
    .initialize()
    .getOpenTelemetrySdk()
    .tap { otel =>
      Classes.registerObservers(otel)
      Cpu.registerObservers(otel)
      MemoryPools.registerObservers(otel)
      Threads.registerObservers(otel)
      GarbageCollector.registerObservers(otel, false).discard
    }
    .tap(OpenTelemetryAppender.install)
```

This does three things: (1) creates a fully configured `OpenTelemetry` instance
from `OTEL_*` environment variables (noop exporters when unset), (2) registers
JVM runtime metric observers (CPU, memory, GC, threads, classes), and (3) hooks
Logback into OpenTelemetry so log records are exported as log signals.

## Server-side tracing

Tapir's server interceptors form a pipeline that processes every request. The
tracing interceptor creates an OpenTelemetry span for each incoming HTTP
request:

```scala
import sttp.shared.Identity

val serverOptions: NettySyncServerOptions = NettySyncServerOptions.customiseInterceptors
  .prependInterceptor(OpenTelemetryTracing(otel))
  .prependInterceptor(SetTraceIdInMDCInterceptor)
  .defaultHandlers(...)
  .corsInterceptor(CORSInterceptor.default[Identity])
  .metricsInterceptor(OpenTelemetryMetrics.default[Identity](otel).metricsInterceptor())
  .options
```

`OpenTelemetryTracing(otel)` is **prepended** — it runs first in the interceptor
chain. This is important: it extracts the trace context from incoming request
headers (W3C `traceparent` format) and creates a span before any other
interceptor runs. This way, all subsequent processing (including business logic)
happens within the span's context.

The span is automatically populated with:
- HTTP method (`http.request.method`)
- URL path (`url.path`)
- Response status code (`http.response.status_code`)
- Request/response sizes

### Interceptor ordering

The final interceptor chain is assembled by `CustomiseInterceptors` in a fixed
order:

1. **Prepended interceptors** — in the order `prependInterceptor` is called
2. **Metrics** — `OpenTelemetryMetrics`
3. **Exception/logging handlers** — (from `defaultHandlers`)
4. **CORS** — handles preflight requests
5. **Reject handler** — 404 for unmatched requests (from `defaultHandlers`)
6. **Decode failure handler** — error formatting (from `defaultHandlers`)

The `prependInterceptor` calls maintain their call order — `OpenTelemetryTracing`
runs before `SetTraceIdInMDCInterceptor` because it's prepended first.
`.defaultHandlers(...)` customises the behaviour of the exception, reject, and
decode failure handlers, but doesn't change where they sit in the chain — their
positions are fixed by the framework.

## Server-side metrics

```scala
.metricsInterceptor(OpenTelemetryMetrics.default[Identity](otel).metricsInterceptor())
```

`OpenTelemetryMetrics.default` registers standard HTTP server metrics:
- `http.server.request.duration` — histogram of request durations
- `http.server.request.total` — counter of total requests
- `http.server.active_requests` — up-down counter of in-flight requests

## Client-side instrumentation

Outgoing HTTP calls (e.g., to external APIs) are instrumented by wrapping sttp's
backend:

```scala
val sttpBackend = Slf4jLoggingBackend(
  OpenTelemetryMetricsBackend(
    OpenTelemetryTracingBackend(
      HttpClientSyncBackend(),
      otel
    ),
    otel
  )
)
```

The backends compose as wrappers, innermost first:
`OpenTelemetryTracingBackend` creates a child span for each outgoing request and
injects the `traceparent` header, enabling distributed tracing.
`OpenTelemetryMetricsBackend` records client-side HTTP metrics. Child spans are
automatically linked to the server span, producing end-to-end traces.

## Custom business metrics

Application-specific metrics are defined using the OpenTelemetry Meter API:

```scala
class Metrics(otel: OpenTelemetry):
  private val meter = otel.meterBuilder("bootzooka")
    .setInstrumentationVersion("1.0")
    .build()

  lazy val registeredUsersCounter: LongCounter =
    meter
      .counterBuilder("bootzooka_registered_users_counter")
      .setDescription("How many users registered on this instance since it was started")
      .build()

  lazy val emailQueueGauge: DoubleGauge =
    meter
      .gaugeBuilder("bootzooka_email_queue_gauge")
      .setDescription("How many emails are waiting to be sent")
      .build()
```

Usage in endpoint logic is straightforward — just call the metric:

```scala
private val registerUserServerEndpoint = registerUserEndpoint.handle { data =>
  val apiKeyResult = db.transactEither(
    userService.registerNewUser(data.login, data.email, data.password)
  )
  metrics.registeredUsersCounter.add(1)
  apiKeyResult.map(apiKey => Register_OUT(apiKey.id.toString))
}
```

In tests, `OpenTelemetry.noop()` is passed, so metric calls become no-ops.

## Trace context propagation with Ox

Virtual threads created within Ox scopes don't automatically inherit the
OpenTelemetry context from the parent thread. This is a problem: if a Tapir
endpoint handler forks work using `ox.fork`, the forked code won't have access
to the current span.

The fix is `PropagatingVirtualThreadFactory` from the `ox otel-context` module:

```scala
import ox.otel.context.PropagatingVirtualThreadFactory

object Main extends OxApp.Simple:
  override protected def settings: Settings =
    Settings.Default.copy(threadFactory = Some(PropagatingVirtualThreadFactory()))

  override def run(using Ox): Unit =
    val deps = Dependencies.create
    deps.emailService.startProcesses()
    deps.httpApi.start().discard
    never
```

`PropagatingVirtualThreadFactory` creates virtual threads that automatically
copy the OpenTelemetry context from the thread that created them. This means:

- When Tapir's tracing interceptor sets up a span on the request-handling
  thread, and the handler forks work using `supervised` / `fork`, the forked
  threads see the same span context.
- Outgoing HTTP calls made from forked threads will correctly produce child
  spans linked to the request span.
- Log statements from forked threads will carry the correct trace ID in the MDC.

Without this factory, forked threads would have an empty context, producing
orphaned spans and logs without trace correlation.

## Log correlation via MDC

The MDC (Mapped Diagnostic Context) is a per-thread map of key-value pairs that
logging frameworks include in log output. The challenge with virtual threads:
standard MDC (`ThreadLocal`-based) breaks when work moves between threads.

Ox solves this with `InheritableMDC`, which propagates MDC values across virtual
threads within `supervised` scopes. The application initializes it at startup:

```scala
object Main extends OxApp.Simple:
  InheritableMDC.init
```

A custom Tapir interceptor extracts the trace ID from the current span and sets
it in the MDC:

```scala
object SetTraceIdInMDCInterceptor extends RequestInterceptor[Identity]:
  val MDCKey = "traceId"

  override def apply[R, B](
      responder: Responder[Identity, B],
      requestHandler: EndpointInterceptor[Identity] => RequestHandler[Identity, R, B]
  ): RequestHandler[Identity, R, B] =
    RequestHandler.from { case (request, endpoints, monad) =>
      val traceId = Span.current().getSpanContext().getTraceId()
      InheritableMDC.unsupervisedWhere(MDCKey -> traceId) {
        requestHandler(EndpointInterceptor.noop)(request, endpoints)(using monad)
      }
    }
```

This interceptor:

1. Reads the current span (set by `OpenTelemetryTracing` which runs earlier in
   the chain).
2. Extracts the trace ID string.
3. Wraps the remaining request handling inside
   `InheritableMDC.unsupervisedWhere`, which sets `traceId` in the MDC for the
   duration of the block — including any virtual threads forked within it.

The result: every log line produced while handling a request includes the trace
ID. In a Logback pattern like `%d [%X{traceId}] %msg%n`, you get:

```
2025-03-10 12:34:56 [abc123def456] Registering new user: john@example.com
2025-03-10 12:34:56 [abc123def456] Creating a new api key for user 789
```

The same trace ID appears in the span sent to your tracing backend, enabling you
to click a trace in Jaeger/Grafana and see the corresponding log lines.

