# Error Output Customisation

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-netty-server-sync"` — HTTP server
  with synchronous backend
- `"com.github.plokhotnyuk.jsoniter-scala" %% "jsoniter-scala-macros"` — JSON
  codec derivation

---

## Defining a JSON error output

All errors are returned as a JSON body with a single `error` field:

```scala
case class Error_OUT(error: String) derives ConfiguredJsonValueCodec, Schema
```

## Mapping Fail to HTTP responses

The `Fail` ADT (see [Error Handling](200-error-handling.md)) needs a bidirectional
mapping to HTTP status codes and error messages: `failToResponseData` (forward) is
used by the server, `responseDataToFail` (reverse) by the client interpreter in
tests, and `failOutput` combines them into the endpoint error output.

```scala
import sttp.model.StatusCode
import sttp.tapir.*
import sttp.tapir.json.jsoniter.jsonBody

private val failToResponseData: Fail => (StatusCode, String) =
  case Fail.NotFound(what)      => (StatusCode.NotFound, what)
  case Fail.Conflict(msg)       => (StatusCode.Conflict, msg)
  case Fail.IncorrectInput(msg) => (StatusCode.BadRequest, msg)
  case Fail.Forbidden           => (StatusCode.Forbidden, "Forbidden")
  case Fail.Unauthorized(msg)   => (StatusCode.Unauthorized, msg)
  case _                        => (StatusCode.InternalServerError, "Internal server error")

private val responseDataToFail: (StatusCode, String) => Fail =
  case (StatusCode.NotFound, what)    => Fail.NotFound(what)
  case (StatusCode.Conflict, msg)     => Fail.Conflict(msg)
  case (StatusCode.Forbidden, _)      => Fail.Forbidden
  case (StatusCode.Unauthorized, msg) => Fail.Unauthorized(msg)
  case (_, msg)                       => Fail.IncorrectInput(msg)

val jsonErrorOutOutput: EndpointOutput[Error_OUT] = jsonBody[Error_OUT]

private val failOutput: EndpointOutput[Fail] =
  statusCode
    .and(jsonErrorOutOutput.map(_.error)(Error_OUT.apply))
    .map(responseDataToFail.tupled)(failToResponseData)
```

> **Important:** these are `val`s, evaluated top-to-bottom as the enclosing object
> initialises. The mapping functions must be declared before `failOutput`, and
> `failOutput` before `baseEndpoint` (below); otherwise the dependent val reads an
> uninitialised `null` and the server fails at startup with a `NullPointerException`.

## Applying to all endpoints

The `failOutput` is used in `baseEndpoint`, from which all endpoints in the
application inherit:

```scala
val baseEndpoint: PublicEndpoint[Unit, Fail, Unit, Any] =
  endpoint.errorOut(failOutput)
```

## Customising default error handlers

Tapir has built-in error handling for situations that happen outside endpoint
logic — decode failures, unmatched routes, and unhandled exceptions. By default,
these produce plain-text responses. To make them return JSON too:

```scala
import sttp.tapir.server.model.ValuedEndpointOutput
import sttp.tapir.server.netty.sync.NettySyncServerOptions

val serverOptions: NettySyncServerOptions = NettySyncServerOptions.customiseInterceptors
  .defaultHandlers(
    msg => ValuedEndpointOutput(jsonErrorOutOutput, Error_OUT(msg)),
    notFoundWhenRejected = true
  )
  .options
```

`defaultHandlers` takes a function that wraps any error message string in the
same `Error_OUT` JSON format used by endpoint error outputs.
`notFoundWhenRejected = true` returns a 404 (as JSON) when no endpoint matches
the request, instead of propagating the rejection to the underlying server.
