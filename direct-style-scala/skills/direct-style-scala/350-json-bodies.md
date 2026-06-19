# JSON Request and Response Bodies

Endpoint bodies are encoded with [jsoniter-scala], whose codecs are derived at
compile time. `JsonCodecMaker` builds a codec for an entire type in one pass —
nested case classes, collections, enums, and simple opaque types included — so
most DTOs need only a `derives` clause. The cases that need more — list bodies,
enums, and complex types — are below.

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-jsoniter-scala"` — `jsonBody`
  integration backed by jsoniter codecs
- `"com.github.plokhotnyuk.jsoniter-scala" %% "jsoniter-scala-macros"` —
  compile-time codec derivation

---

## Deriving DTO codecs

A request/response DTO derives a jsoniter codec (`ConfiguredJsonValueCodec`, a
`JsonValueCodec`) and a Tapir `Schema`; `jsonBody[T]` requires both in scope:

```scala
import com.github.plokhotnyuk.jsoniter_scala.macros.ConfiguredJsonValueCodec
import sttp.tapir.Schema
import sttp.tapir.json.jsoniter.jsonBody

case class CreateTodo_IN(title: String) derives ConfiguredJsonValueCodec, Schema

val createBody = jsonBody[CreateTodo_IN]
```

## List bodies need their own codec

A `List[T]` *field* is handled as part of its enclosing codec, but when the body
itself is a list, `jsonBody[List[Todo_OUT]]` needs a `JsonValueCodec[List[Todo_OUT]]`
of its own — deriving `Todo_OUT`'s codec does not produce one for the list type:

```scala
import com.github.plokhotnyuk.jsoniter_scala.core.JsonValueCodec
import com.github.plokhotnyuk.jsoniter_scala.macros.JsonCodecMaker

given JsonValueCodec[List[Todo_OUT]] = JsonCodecMaker.make
```

If the element's codec needs configuration — e.g. it contains an enum (see
below) — pass the same config to the list codec.

## Encoding enums

A parameterless Scala 3 `enum` is described by two independent things that must
agree: its Tapir `Schema` (validation and OpenAPI docs) and its jsoniter codec
(the actual JSON).

On the **schema** side, an enum needs an explicit `Schema` built with
`derivedEnumeration` — the general `derives Schema` / `Schema.derived` can't
enumerate the cases, so it does not produce a proper string enumeration.
`.defaultStringBased` encodes each case as its name.

On the **codec** side, jsoniter encodes a parameterless enum as a tagged object,
`{"type":"Active"}`, by default. For a plain string (`"Active"`), build the codec
with `withDiscriminatorFieldName(None)`. jsoniter derives the enum inline as part
of the enclosing codec, so its representation follows that codec's config — set the
config on every DTO and list codec that contains the enum, dropping
`derives ConfiguredJsonValueCodec` (which gives no place to pass config) in favour
of an explicit `given`:

```scala
import com.github.plokhotnyuk.jsoniter_scala.core.JsonValueCodec
import com.github.plokhotnyuk.jsoniter_scala.macros.{CodecMakerConfig, JsonCodecMaker}
import sttp.tapir.Schema

enum TodoStatus:
  case Active, Done
object TodoStatus:
  given Schema[TodoStatus] = Schema.derivedEnumeration[TodoStatus].defaultStringBased

case class Todo_OUT(id: Long, title: String, status: TodoStatus) derives Schema
object Todo_OUT:
  given JsonValueCodec[Todo_OUT] =
    JsonCodecMaker.make(CodecMakerConfig.withDiscriminatorFieldName(None))
  given JsonValueCodec[List[Todo_OUT]] =
    JsonCodecMaker.make(CodecMakerConfig.withDiscriminatorFieldName(None))
```

> **Important:** the schema and the codec must describe the same wire format —
> `defaultStringBased` (string in the docs) paired with
> `withDiscriminatorFieldName(None)` (string on the wire). And since each generated
> codec is configured independently, setting the config on `Todo_OUT` but leaving
> `List[Todo_OUT]` on the default would emit `"Active"` for a single todo and
> `{"type":"Active"}` inside a list — configure both the same way.

## Opaque-type identifiers

Domain models wrap identifiers in opaque types. jsoniter derives codecs for
*simple* opaque types automatically — `opaque type TodoId = Long` serializes as a
bare `Long` — so a DTO uses the opaque type directly, with no hand-written codec.
It does need a `Schema`, mapped from the underlying type:

```scala
import sttp.tapir.Schema

opaque type TodoId = Long
object TodoId:
  def apply(value: Long): TodoId = value
  extension (id: TodoId) def value: Long = id
  given Schema[TodoId] = Schema.schemaForLong.as[TodoId]

case class Todo_OUT(id: TodoId, title: String, status: TodoStatus) derives Schema
```

`JsonCodecMaker.make` cannot derive *union* types or *complex* opaque types; for
those, put a `given JsonValueCodec[T]` in scope before the `make` call and it will
be used.

[jsoniter-scala]: https://github.com/plokhotnyuk/jsoniter-scala
