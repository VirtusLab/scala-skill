# Kafka Streaming

## Dependencies

- `"com.softwaremill.ox" %% "kafka"` — Kafka integration for Ox Flows

---

## Consuming from Kafka

Kafka integration is built on Ox Flows — lazy, asynchronous data pipelines that
execute when a terminal operation (`runDrain`, `runForeach`) is called.

`KafkaFlow.subscribe` creates a `Flow[ReceivedMessage[K, V]]` from a Kafka
topic:

```scala
import ox.kafka.{ConsumerSettings, KafkaFlow, ReceivedMessage}
import ox.kafka.ConsumerSettings.AutoOffsetReset

val settings = ConsumerSettings.default("my_group")
  .bootstrapServers("localhost:9092")
  .autoOffsetReset(AutoOffsetReset.Earliest)

KafkaFlow
  .subscribe(settings, "my_topic")
  .map(msg => process(msg.value))
  .runDrain()
```

## Publishing to Kafka

Messages can be published as a terminal operation (drain) or as an intermediate
step that returns metadata:

```scala
// As a drain — fire and forget
Flow.fromIterable(messages)
  .map(msg => ProducerRecord[String, String]("my_topic", msg))
  .pipe(KafkaDrain.runPublish(producerSettings))

// As an intermediate step — returns RecordMetadata per message
Flow.fromIterable(messages)
  .map(msg => ProducerRecord[String, String]("my_topic", msg))
  .mapPublish(producerSettings)
  .runForeach(metadata => println(s"Sent to partition ${metadata.partition()}"))
```

## Consuming, processing, and committing offsets

A common pattern: consume messages, process them (possibly in parallel), and
commit offsets:

```scala
supervised:
  val consumer = consumerSettings.toThreadSafeConsumerWrapper
  KafkaFlow
    .subscribe(consumer, "source_topic")
    .mapPar(10) { msg =>
      process(msg.value)
      CommitPacket(msg)
    }
    .pipe(KafkaDrain.runCommit(consumer))
```

When committing offsets, the consumer must be shared between subscribe and commit
stages via `toThreadSafeConsumerWrapper`. Offsets are committed in batches (by
default, every second), computing the maximum offset per topic-partition.

## Transactional produce-and-commit

For exactly-once semantics when reading from one topic and writing to another:

```scala
supervised:
  val consumer = consumerSettings.toThreadSafeConsumerWrapper
  KafkaFlow
    .subscribe(consumer, "source_topic")
    .map { msg =>
      val output = ProducerRecord[String, String]("target_topic", transform(msg.value))
      SendPacket(output, msg)
    }
    .pipe(KafkaDrain.runPublishAndCommit(producerSettings, consumer))
```

`SendPacket` combines records to publish with messages to commit. The drain uses
Kafka transactions to atomically publish the output records and commit the input
offsets.

## Integration with structured concurrency

Kafka consumers run within Ox scopes. When the scope ends (e.g., application
shutdown), the consumer is closed gracefully — pending offsets are committed and
the consumer group rebalances.

```scala
object Main extends OxApp.Simple:
  override def run(using Ox): Unit =
    KafkaFlow
      .subscribe(settings, "my_topic")
      .mapPar(5)(process)
      .runDrain()
```

`runDrain()` blocks indefinitely (the Kafka consumer keeps polling), so there's
no need for `never`. On SIGTERM, `OxApp` interrupts the scope, closing the
consumer gracefully.
