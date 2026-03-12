# Compile-Time OpenAPI Generation

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-openapi-docs"` — OpenAPI model
  generation from endpoint descriptions
- `"com.softwaremill.sttp.apispec" %% "openapi-circe-yaml"` — serialise OpenAPI
  model to YAML

---

## Why generate at build time

Tapir can serve OpenAPI documentation at runtime (via `SwaggerInterpreter`), but
generating the YAML at build time lets frontend builds consume it to generate
typed HTTP clients.

## Collecting endpoints for docs

Each API module exposes its endpoints for docs via the `EndpointsForDocs` trait:

```scala
trait EndpointsForDocs:
  def endpointsForDocs: List[AnyEndpoint]
```

Each module tags its endpoints for grouping in the docs:

```scala
object UserApi extends EndpointsForDocs:
  override val endpointsForDocs = wireList[AnyEndpoint].map(_.tag("user"))

object PasswordResetApi extends EndpointsForDocs:
  override val endpointsForDocs = wireList[AnyEndpoint].map(_.tag("password-reset"))
```

`wireList` collects all matching values — see [Compile-Time Dependency
Injection](04-compile-time-dependency-injection.md).

All endpoint lists are merged in a single place:

```scala
object Dependencies:
  val endpointsForDocs: List[AnyEndpoint] =
    List(UserApi, PasswordResetApi, VersionApi).flatMap(_.endpointsForDocs)
```

## The generator

A standalone `@main` function converts the endpoint descriptions to YAML and
writes the file:

```scala
import sttp.apispec.openapi.circe.yaml.*
import sttp.tapir.docs.openapi.OpenAPIDocsInterpreter
import java.nio.file.*

object OpenAPIDescription:
  val Title = "My API"
  val Version = "1.0"

@main def writeOpenAPIDescription(path: String): Unit =
  val yaml = OpenAPIDocsInterpreter()
    .toOpenAPI(Dependencies.endpointsForDocs, OpenAPIDescription.Title, OpenAPIDescription.Version)
    .toYaml
  Files.writeString(Paths.get(path), yaml)
```

This runs without starting the HTTP server — no runtime configuration needed.

## Wiring as an sbt task

In `build.sbt`, define a task key and wire the generator:

```scala
val generateOpenAPIDescription = taskKey[Unit]("Generates OpenAPI YAML from endpoint descriptions")

generateOpenAPIDescription := Def.taskDyn {
  val targetPath = ((Compile / target).value / "openapi.yaml").toString
  Def.task {
    (Compile / runMain).toTask(
      s" com.softwaremill.myapp.writeOpenAPIDescription $targetPath"
    ).value
  }
}.value
```

The generated `openapi.yaml` is written to the `target` directory, where the
frontend build can pick it up to generate typed API stubs.
