# New Project Setup

Bootstrap a minimal direct-style Scala project using sbt and Ox. The skeleton
below gives a compiling, runnable entry point with nothing else — add
features (HTTP, JSON, persistence, config) by following the chapters listed
at the end. For a project that already includes a Tapir HTTP server, use
[adopt-tapir](https://adopt-tapir.softwaremill.com) with **Scala 3** / **sbt**
/ **Ox stack** / **Netty**.

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
val oxVersion = "1.0.4"

lazy val root = (project in file(".")).settings(
  name         := "myapp",
  version      := "0.1.0-SNAPSHOT",
  organization := "com.example",
  scalaVersion := "3.8.3",
  scalacOptions ++= Seq(
    "-Wunused:all",
    "-Wvalue-discard",
    "-Wnonunit-statement"
  ),
  libraryDependencies ++= Seq(
    "com.softwaremill.ox" %% "core"            % oxVersion,
    "ch.qos.logback"       % "logback-classic" % "1.5.32"
  )
)
```

> **Required:** `scalacOptions` must include `-Wunused:all`,
> `-Wvalue-discard`, and `-Wnonunit-statement`. These catch silently-ignored
> values and unused bindings — common bug sources in direct-style code where
> effects aren't tracked by types. Fix warnings at source; only use `@nowarn`
> for generated code or unfixable third-party issues.

## project/plugins.sbt

```scala
addSbtPlugin("org.scalameta" % "sbt-scalafmt" % "2.4.6")
```

## project/build.properties

```
sbt.version=1.12.5
```

Pins the sbt launcher version so every contributor and CI run uses the same
tool.

## .scalafmt.conf

```
version = 3.11.0
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

- Resource lifecycle — [Resource Management](100-resource-management.md).
- `application.conf` + `Config.read` — [Type-Safe Configuration](120-type-safe-configuration.md).
- Wiring multiple services — [Compile-Time Dependency Injection](130-compile-time-dependency-injection.md).
- HTTP endpoints with synchronous Tapir — [HTTP Server Configuration](310-http-server-configuration.md).
- In-process endpoint testing — [Testing HTTP Endpoints](500-testing-http-endpoints.md).
- PostgreSQL + Flyway — [SQL Persistence](400-sql-persistence.md).
- Observability via OpenTelemetry — [OpenTelemetry Observability](510-opentelemetry-observability.md).
