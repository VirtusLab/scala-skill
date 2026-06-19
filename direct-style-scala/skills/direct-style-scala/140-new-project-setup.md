# New Project Setup

For an **HTTP project**, generate the skeleton with
[adopt-tapir](https://adopt-tapir.softwaremill.com) (**Scala 3** / **sbt** / **Ox
stack** / **Netty**) rather than assembling the server by hand. It produces a
compiling, runnable project — entry point, server wiring, an example endpoint,
JSON, and a test — already on the direct-style stack; the chapters then explain
each part and add what the skeleton omits (error handling, auth, config,
persistence).

For a **non-HTTP** app (CLI, worker, batch job), bootstrap the minimal sbt + Ox
skeleton below — a compiling, runnable entry point with nothing else. Either way,
add features (HTTP, JSON, persistence, config) by following the chapters listed at
the end.

## Dependencies

- `"com.softwaremill.ox" %% "core"` — direct-style concurrency (`OxApp`,
  `supervised`, `fork`, `useInScope`)
- `"ch.qos.logback" % "logback-classic"` — logging backend
- sbt plugin: `sbt-scalafmt` — formatting

---

## Directory layout

```
myapp/
  build.sbt
  project/
    build.properties
    plugins.sbt
  src/
    main/
      scala/com/example/
        Main.scala
      resources/
        logback.xml
  .gitignore
  .scalafmt.conf
```

The package path under `scala/` mirrors the `organization` in `build.sbt`.

## build.sbt

```scala
val oxVersion = "<latest>"

lazy val root = (project in file(".")).settings(
  name         := "myapp",
  version      := "0.1.0-SNAPSHOT",
  organization := "com.example",
  scalaVersion := "<latest>",
  scalacOptions ++= Seq(
    "-Wunused:all",
    "-Wvalue-discard",
    "-Wnonunit-statement"
  ),
  libraryDependencies ++= Seq(
    "com.softwaremill.ox" %% "core"            % oxVersion,
    "ch.qos.logback"       % "logback-classic" % "<latest>"
  )
)
```

> **Required:** `scalacOptions` must include `-Wunused:all`,
> `-Wvalue-discard`, and `-Wnonunit-statement`. These catch silently-ignored
> values and unused bindings — common bug sources in direct-style code where
> effects aren't tracked by types. Fix warnings at source; only use `@nowarn`
> for generated code or unfixable third-party issues.

> **Note:** `<latest>` (here and below) is a placeholder — resolve the current
> release of each library and tool rather than copying a pinned number, which
> goes stale.

## project/plugins.sbt

```scala
addSbtPlugin("org.scalameta" % "sbt-scalafmt" % "<latest>")
```

## project/build.properties

```
sbt.version=<latest>
```

Pins the sbt launcher version so every contributor and CI run uses the same
tool.

## .scalafmt.conf

```
version = <latest>
maxColumn = 140
runner.dialect = scala3
```

`runner.dialect = scala3` is mandatory for Scala 3 / braceless syntax to
format correctly.

## .gitignore

```
.idea/
.metals/
.vscode/
.bloop
target
metals.sbt
project/project
```

## Main.scala

```scala
package com.example

import ox.*

object Main extends OxApp.Simple:
  def run(using Ox): Unit =
    println("myapp started")
```

`OxApp.Simple` opens the root `supervised` scope. When `run` returns the
scope ends, cleaning up any resources registered with `useInScope` and
joining all forks. For a long-running application, end `run` with `never`
instead — see [Background Processes](110-background-processes.md).

## src/main/resources/logback.xml

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    <root level="DEBUG">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

## Extending the skeleton

Add features by following the relevant chapter:

- Scala visibility, package, file, and module rules — [Code Organization and
  Visibility](160-code-organization.md).
- Resource lifecycle — [Resource Management](100-resource-management.md).
- `application.conf` + `Config.read` — [Type-Safe Configuration](120-type-safe-configuration.md).
- Wiring multiple services — [Compile-Time Dependency Injection](130-compile-time-dependency-injection.md).
- HTTP endpoints with synchronous Tapir — [HTTP Server Configuration](310-http-server-configuration.md).
- In-process endpoint testing — [Testing HTTP Endpoints](500-testing-http-endpoints.md).
- PostgreSQL + Flyway — [SQL Persistence](400-sql-persistence.md).
- Observability via OpenTelemetry — [OpenTelemetry Observability](510-opentelemetry-observability.md).
