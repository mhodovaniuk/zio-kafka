## Contents

- [Quickstart](#quickstart)
- [Consuming Kafka topics using ZIO Streams](#consuming-kafka-topics-using-zio-streams)
- [Example: consuming, producing and committing offset](#example-consuming-producing-and-committing-offset)
- [Partition assignment and offset retrieval](#partition-assignment-and-offset-retrieval)
- [Custom data type serdes](#custom-data-type-serdes)
- [Handling deserialization failures](#handling-deserialization-failures)

## Quickstart

Add the following dependencies to your `build.sbt` file:
```
libraryDependencies += "dev.zio" %% "zio-kafka" % "<version>"
```

Somewhere in your application, configure the `zio.kafka.ConsumerSettings`
data type:
```scala
import zio._
import zio.kafka.consumer._

val settings: ConsumerSettings = 
  ConsumerSettings(List("localhost:9092"))
    .withGroupId("group")
    .withClientId("client")
    .withCloseTimeout(30.seconds)
```

For a lot of use cases where you just want to do something with all messages on a Kafka topic, ZIO Kafka provides the convenience method `Consumer.consumeWith`. This method lets you execute a ZIO effect for each message. Topic partitions will be processed in parallel and offsets are committed after running the effect automatically.

```scala
import zio._
import zio.kafka.consumer._
import zio.kafka.serde._

val subscription = Subscription.topics("topic")

Consumer.consumeWith(settings, subscription, Serde.string, Serde.string) { case (key, value) =>
  Console.printLine(s"Received message ${key}: ${value}")
  // Perform an effect with the received message
}
```

If you require more control over the consumption process, read on!

## Consuming Kafka topics using ZIO Streams

First, create a consumer using the ConsumerSettings instance:
```scala
import zio.Clock, zio.ZLayer, zio.ZManaged
import zio.kafka.consumer.{ Consumer, ConsumerSettings }

val consumerSettings: ConsumerSettings = ConsumerSettings(List("localhost:9092")).withGroupId("group")
val consumerManaged: ZIO[Scope, Throwable, Consumer] =
  Consumer.make(consumerSettings)
val consumer: ZLayer[Clock, Throwable, Consumer] =
  ZLayer.scoped(consumerManaged)
```

The consumer returned from `Consumer.make` is wrapped in a `ZLayer`
to allow for easy composition with other ZIO environment components.
You may provide that layer to effects that require a consumer. Here's
an example:
```scala
import zio._
import zio.kafka.consumer._
import zio.kafka.serde._

val data: RIO[Clock, 
              Chunk[CommittableRecord[String, String]]] = 
  (Consumer.subscribe(Subscription.topics("topic")) *>
  Consumer.plainStream(Serde.string, Serde.string).take(50).runCollect)
    .provideSomeLayer(consumer)
```

You may stream data from Kafka using the `subscribeAnd` and `plainStream`
methods:

```scala
import zio.Clock, zio.Console.printLine
import zio.kafka.consumer._

Consumer.subscribeAnd(Subscription.topics("topic150"))
  .plainStream(Serde.string, Serde.string)
  .tap(cr => printLine(s"key: ${cr.record.key}, value: ${cr.record.value}"))
  .map(_.offset)
  .aggregateAsync(Consumer.offsetBatches)
  .mapZIO(_.commit)
  .runDrain
```

If you need to distinguish between the different partitions assigned
to the consumer, you may use the `Consumer#partitionedStream` method,
which creates a nested stream of partitions:
```scala
import zio.Clock, zio.Console.printLine
import zio.kafka.consumer._

Consumer.subscribeAnd(Subscription.topics("topic150"))
  .partitionedStream(Serde.string, Serde.string)
  .tap(tpAndStr => printLine(s"topic: ${tpAndStr._1.topic}, partition: ${tpAndStr._1.partition}"))
  .flatMap(_._2)
  .tap(cr => printLine(s"key: ${cr.record.key}, value: ${cr.record.value}"))
  .map(_.offset)
  .aggregateAsync(Consumer.offsetBatches)
  .mapZIO(_.commit)
  .runDrain
```

## Example: consuming, producing and committing offset

This example shows how to consume messages from topic `topic_a` and produce transformed messages to `topic_b`, after which consumer offsets are committed. Processing is done in chunks using `ZStreamChunk` for more efficiency.

```scala
import zio.ZLayer
import zio.kafka.consumer._
import zio.kafka.producer._
import zio.kafka.serde._
import org.apache.kafka.clients.producer.ProducerRecord

val consumerSettings: ConsumerSettings = ConsumerSettings(List("localhost:9092")).withGroupId("group")
val producerSettings: ProducerSettings = ProducerSettings(List("localhost:9092"))

val consumerAndProducer = 
  ZLayer.scoped(Consumer.make(consumerSettings)) ++
    ZLayer.scoped(Producer.make(producerSettings, Serde.int, Serde.string))

val consumeProduceStream = Consumer
  .subscribeAnd(Subscription.topics("my-input-topic"))
  .plainStream(Serde.int, Serde.long)
  .map { record =>
    val key: Int    = record.record.key()
    val value: Long = record.record.value()
    val newValue: String = value.toString

    val producerRecord: ProducerRecord[Int, String] = new ProducerRecord("my-output-topic", key, newValue)
    (producerRecord, record.offset)
  }
  .mapChunksZIO { chunk =>
    val records     = chunk.map(_._1)
    val offsetBatch = OffsetBatch(chunk.map(_._2).toSeq)

    Producer.produceChunk[Any, Int, String](records) *> offsetBatch.commit.as(Chunk(()))
  }
  .runDrain
  .provideSomeLayer(consumerAndProducer)
```

## Partition assignment and offset retrieval

`zio-kafka` offers several ways to control which Kafka topics and partitions are assigned to your application.

| Use case | Method |
| --- | --- |
| One or more topics, automatic partition assignment | `Consumer.subscribe(Subscription.topics("my_topic", "other_topic"))` |
| Topics matching a pattern | `Consumer.subscribe(Subscription.pattern("topic.*"))` |
| Manual partition assignment | `Consumer.subscribe(Subscription.manual("my_topic" -> 1, "my_topic" -> 2))` |

By default `zio-kafka` will start streaming a partition from the last committed offset for the consumer group, or the latest message on the topic if no offset has yet been committed. You can also choose to store offsets outside of Kafka. This can be useful in cases where consistency between data stores and consumer offset is required.

| Use case | Method |
| --- | --- |
| Offsets in Kafka, start at latest message if no offset committed | `OffsetRetrieval.Auto()` |
| Offsets in Kafka, start at earliest message if no offset committed | `OffsetRetrieval.Auto(AutoOffsetStrategy.Earliest)` |
| Manual/external offset storage | `Manual(getOffsets: Set[TopicPartition] => Task[Map[TopicPartition, Long]])` |

For manual offset retrieval, the `getOffsets` function will be called for each topic-partition that is assigned to the consumer, either via Kafka's rebalancing or via a manual assignment.

## Custom data type serdes

Serializers and deserializers (serdes) for custom data types can be constructed from scratch or by converting existing serdes. For example, to create a serde for an `Instant`:

```scala
import java.time.Instant
import zio.kafka.serde._

val instantSerde: Serde[Any, Instant] = Serde.long.inmap(java.time.Instant.ofEpochMilli)(_.toEpochMilli)
```

## Handling deserialization failures

The default behavior for a consumer stream when encountering a deserialization failure is to fail the stream. In many cases you may want to handle this situation differently, e.g. by skipping the message that failed to deserialize or by executing an alternative effect. For this purpose, any `Deserializer[T]` for some type `T` can be easily converted into a `Deserializer[Try[T]]` where deserialization failures are converted to a `Failure` using the `asTry` method.

Below is an example of skipping messages that fail to deserialize. The offset is passed downstream to be committed.

```scala
import zio._, stream._
import zio.kafka.consumer._
import zio.kafka.serde._
import scala.util.{Try, Success, Failure}

val consumer = ZLayer.scoped(Consumer.make(consumerSettings))

val stream = Consumer
  .subscribeAnd(Subscription.topics("topic150"))
  .plainStream(Serde.string, Serde.string.asTry)

stream 
  .mapZIO { record => 
    val tryValue: Try[String] = record.record.value()
    val offset: Offset = record.offset
  
    tryValue match {
      case Success(value) =>
        // Action for successful deserialization
        someEffect(value).as(offset)
      case Failure(exception) =>
        // Possibly log the exception or take alternative action
        ZIO.succeed(offset)
    }
  }
  .aggregateAsync(Consumer.offsetBatches)
  .mapZIO(_.commit)
  .runDrain
  .provideSomeLayer(consumer)
```