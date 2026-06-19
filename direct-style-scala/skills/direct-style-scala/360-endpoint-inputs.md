# Endpoint Inputs

An endpoint's inputs are declared with `.in(...)` — path segments, query
parameters, headers, and bodies. This chapter covers decoding non-body inputs into
custom types, and how several inputs are passed to the handler.

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

## Multiple inputs

Each `.in(...)` adds to the endpoint's input type; several of them make it a tuple.
The handler receives that tuple untupled — one parameter per input, in declaration
order — because Scala 3 untuples the function literal, so there's nothing to
destructure (`.handle` wires the logic; see [Error Handling](200-error-handling.md)):

```scala
val updateUser = baseEndpoint.put
  .in("todos" / path[TodoId]("id"))
  .in(jsonBody[UpdateTodo_IN])
  .out(jsonBody[Todo_OUT])

updateUser.handle: (id, body) =>
  todoService.update(id, body) // id: TodoId, body: UpdateTodo_IN
```
