# Domain Modeling

Worked examples for the functional-programming rules in `SKILL.md`: immutable
state, lawful APIs, opaque types for domain values, and enums instead of
`Boolean`s. Plain Scala 3.

## Dependencies

None.

---

## Immutable state with pure transitions

Model state as an immutable case class and each transition as a pure
`state => state` function. When a loop needs to accumulate state, confine the
`var` to the method running the loop:

```scala
case class ProcessingState(
    processed: Map[String, Long] = Map.empty,
    pending: Set[String] = Set.empty
)

def handleItem(state: ProcessingState, item: Item): ProcessingState =
  state.copy(processed = state.processed.updated(item.key, item.offset))

def run(items: Iterator[Item]): ProcessingState =
  var state = ProcessingState()
  for item <- items do
    state = handleItem(state, item)
  state
```

`handleItem` can be tested without running the loop, and no other thread can
see the `var`.

## Lawful APIs

Pass `Clock`, `Random`, and UUID generation in explicitly, as a parameter or a
constructor argument. Calling them inside a method hides non-determinism and
makes the result impossible to assert in a test:

```scala
// Wrong — hidden non-determinism:
class OrderService:
  def place(order: Order): Confirmation =
    val id = UUID.randomUUID()
    val now = Instant.now()
    Confirmation(id, now)

// Right — dependencies are explicit and injectable:
class OrderService(clock: Clock, idGenerator: () => UUID):
  def place(order: Order): Confirmation =
    val id = idGenerator()
    val now = clock.instant()
    Confirmation(id, now)
```

## Opaque types for domain values

Wrap identifiers (`OrderId`, `ProductCode`), quantities (`Quantity`,
`Amount`), and configuration values (`Port`, `TopicName`) in opaque types. The
compiler then rejects passing a `ProductCode` where an `OrderId` is expected,
at no runtime cost.

When a value has invariants, enforce them in a smart constructor that returns
`Either`, not with `require` — an invalid value is a recoverable failure (see
[Error Handling](200-error-handling.md)):

```scala
opaque type Quantity = Int
object Quantity:
  def apply(value: Int): Either[Fail, Quantity] =
    Either.cond(value > 0, value, Fail.IncorrectInput(s"quantity must be positive, got $value"))
  extension (q: Quantity) def value: Int = q
```

Codecs for opaque types: JSON bodies in [JSON Request and Response
Bodies](350-json-bodies.md), path/query/header inputs in [Endpoint
Inputs](360-endpoint-inputs.md), database columns in [SQL
Persistence](400-sql-persistence.md).

## Enums instead of `Boolean`s

Replace `Boolean` parameters and return values with two-case enums, so a call
site says what it means:

```scala
// Wrong — caller must remember what `true` means:
def recordFlush(success: Boolean, durationMs: Double): Unit

// Right — intent is unambiguous:
enum FlushOutcome:
  case Success, Failure
def recordFlush(outcome: FlushOutcome, duration: Duration): Unit
```
