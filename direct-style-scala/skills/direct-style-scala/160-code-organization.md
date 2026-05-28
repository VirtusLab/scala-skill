# Code Organization and Visibility

Use this chapter when creating packages/modules, moving Scala files, or widening
visibility. It documents Scala-specific rules and project policy that generic
code organization advice often misses.

## Before creating a file

Before creating a `.scala` file, answer:

1. Which existing package does this belong to?
2. What visibility does each top-level declaration deserve?
3. Does the filename match the primary type or concept?

## Top-level visibility

Every top-level `class`, `trait`, `enum`, and `object` MUST be classified
explicitly:

- default-public: no modifier, only for intentionally public API.
- `private[<subpkg>]`: visible only inside the current concept package.
- `private[<rootpkg>]`: visible across sbt modules that share the root package.

Pick by scanning actual call sites. Start narrow; widen only when a caller
needs access.

```scala
package com.example.myapp.git

private[git] final class GitCommandParser:
  def parse(raw: String): GitCommand = ???

private[myapp] trait GitClient:
  def status(repo: RepoPath): Either[Fail, GitStatus]

final class GitService(client: GitClient):
  def status(repo: RepoPath): Either[Fail, GitStatus] =
    client.status(repo)
```

## Files

The discriminator for "own file vs. shared file" is **where a type is
constructed**, not where it is referenced. A service may share its file with
return types and exceptions it throws — if the service is the sole constructor
of those types. A type constructed in multiple places (several services, codec 
deserialization, etc.) gets its own file named after itself.

- Bundle sealed hierarchies in one file.
- A trait may share a file with its single canonical implementation (e.g.
  `Default*`).
- Tightly coupled pairs may share a file, e.g. an enum and its sole consumer
  callback type.
- Otherwise, use one top-level type per file.

Do not create `GitTypes.scala`, `Models.scala`, or `Helpers.scala`. The "Types"
suffix is itself the smell — when the only thing the contents share is "they're
types", split per the rules above.

## Packages

- Use concept package names (`events`, `git`, `plan`, `auth`, `billing`), not
  mechanism names (`util`, `io`, `core`, `common`).
- At most keep one tiny `<root>.util` leaf for shared primitives; split it once
  it grows past about five files.
- Avoid single-file packages. Inline them into the parent or fold them into a
  sibling package.
- A sub-package name MUST NOT shadow a top-level `def` or `val` in the same
  root. Scala 3 forbids `def git` and `package com.example.myapp.git`
  together. Sink the package deeper instead, e.g. `com.example.myapp.tools.git`.

## Modules and boundaries

Use sbt modules only for separate publish artifacts, optional dependencies, or
enforced import direction. Enforce boundaries by construction first:

1. sbt `dependsOn` edges define allowed module direction.
2. `private[<rootpkg>]` keeps implementation details inside the shared root.
3. Use a custom Scalafix `ForbidImports` rule only when sbt and visibility
   cannot express the constraint.

