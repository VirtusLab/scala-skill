# HTTP Server Configuration

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-netty-server-sync"` — HTTP server
  with synchronous backend
- `"com.softwaremill.sttp.tapir" %% "tapir-files"` — serving static files

---

## Security headers

HTTP responses should include headers that prevent common attacks. These are
added to `baseEndpoint`, so every endpoint in the application inherits them:

```scala
val baseEndpoint: PublicEndpoint[Unit, Fail, Unit, Any] =
  endpoint
    .errorOut(failOutput)
    .out(header("X-Frame-Options", "DENY"))
    .out(header("Content-Security-Policy", "frame-ancestors 'none'"))
```

Both headers prevent clickjacking — `X-Frame-Options` for older browsers,
`Content-Security-Policy` for modern ones. Since these are Tapir output headers, they're included in every successful
response. Error responses (decode failures, 404s) are handled separately by the
server options — see [Error Output Customisation](06-error-output-customisation.md).

## CORS

Cross-Origin Resource Sharing headers are needed when the frontend is served
from a different origin than the API (e.g., during development with a local dev
server). Tapir provides a built-in CORS interceptor:

```scala
val serverOptions: NettySyncServerOptions = NettySyncServerOptions.customiseInterceptors
  .corsInterceptor(CORSInterceptor.default[Identity])
  // ... other interceptors
  .options
```

`CORSInterceptor.default[Identity]` allows all origins and common HTTP methods.
For production, restrict the allowed origins:

```scala
import sttp.tapir.server.interceptor.cors.{CORSConfig, CORSInterceptor}

val corsInterceptor = CORSInterceptor.customOrThrow[Identity](
  CORSConfig.default.allowOrigin("https://example.com")
)
```

The CORS interceptor is part of the server options interceptor chain (see
[OpenTelemetry Observability](15-opentelemetry-observability.md) for the full
chain ordering). It handles preflight `OPTIONS` requests automatically.

## Serving static files

For single-page applications, the backend serves the frontend's static assets
(HTML, JS, CSS) and falls back to `index.html` for client-side routes:

```scala
import sttp.tapir.files.{FilesOptions, staticResourcesGetServerEndpoint}

val webappEndpoints = List(
  staticResourcesGetServerEndpoint[Identity](emptyInput: EndpointInput[Unit])(
    classOf[HttpApi].getClassLoader,
    "webapp",
    FilesOptions.default[Identity].defaultFile(List("index.html"))
  )
)
```

`staticResourcesGetServerEndpoint` serves files from the `webapp` directory on
the classpath. The `emptyInput` means it matches any path not already handled by
API endpoints.

`defaultFile(List("index.html"))` is the key for SPA support: when a request
doesn't match any static file (e.g., `/login`, `/settings`), the server returns
`index.html` instead of 404.

These endpoints are added after the API endpoints:

```scala
val allEndpoints = apiEndpoints ++ webappEndpoints
```

Order matters — API endpoints are matched first.

## Request cancellation

When a client disconnects mid-request (e.g., a user navigates away), the server
can either interrupt the in-progress handler or let it finish. By default, Tapir
interrupts the handler. For applications using database connections, this can
cause problems:

```scala
val serverOptions: NettySyncServerOptions = NettySyncServerOptions.customiseInterceptors
  // ... interceptors
  .options
  .copy(interruptServerLogicWhenRequestCancelled = false)
```

Setting `interruptServerLogicWhenRequestCancelled = false` lets cancelled
requests run to completion. This avoids a specific issue with connection pools:
when a JDBC call is interrupted mid-execution, the connection pool (e.g.,
HikariCP) may mark that connection as broken. Re-establishing connections is
more expensive than finishing the already-running request.

