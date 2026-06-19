# Endpoint Input Codecs

Path, query, and header values are decoded from strings by a Tapir `Codec` — a
`PlainCodec[T]`, i.e. `Codec[String, T, CodecFormat.TextPlain]` — which is a
separate mechanism from the JSON body codecs in [JSON Request and Response
Bodies](350-json-bodies.md). Built-in types resolve a codec automatically; a
custom type needs a given one in scope, which `path[T]` / `query[T]` then pick up.

## Dependencies

- `"com.softwaremill.sttp.tapir" %% "tapir-core"` — `Codec`, `path` / `query`
  inputs, `derivedEnumeration`

---

## Opaque-type identifiers

Map a built-in codec to and from the opaque type. `.map` is for a total
conversion — every underlying value is valid, as for an id:

```scala
import sttp.tapir.{Codec, CodecFormat}

opaque type TodoId = Long
object TodoId:
  def apply(value: Long): TodoId = value
  extension (id: TodoId) def value: Long = id

  given Codec[String, TodoId, CodecFormat.TextPlain] = Codec.long.map(TodoId(_))(_.value)

// endpoint.in("todos" / path[TodoId]("id"))
```

If the conversion can fail, use `.mapDecode` (returning a `DecodeResult`) instead,
so a bad value produces a 400, not a 500.

## Enums

Use `derivedEnumeration` — the `Codec` analogue of the `Schema.derivedEnumeration`
used for JSON-body enums — so the value decodes from the case name and anything
else is rejected:

```scala
import sttp.tapir.{Codec, CodecFormat}

enum Sort:
  case Asc, Desc
object Sort:
  given Codec[String, Sort, CodecFormat.TextPlain] =
    Codec.derivedEnumeration[String, Sort].defaultStringBased

// endpoint.in(query[Sort]("sort"))
```
