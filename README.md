#logback-kafka-appender

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.github.danielwegener/logback-kafka-appender/badge.svg)](https://maven-badges.herokuapp.com/maven-central/com.github.danielwegener/logback-kafka-appender)
[![Build Status](https://api.travis-ci.org/danielwegener/logback-kafka-appender.svg)](https://travis-ci.org/logback-kafka-appender/logback-kafka-appender)
[![Coverage Status](https://img.shields.io/coveralls/danielwegener/logback-kafka-appender.svg)](https://coveralls.io/r/danielwegener/logback-kafka-appender)

This appender provides a way for applications to publish their application logs to Apache Kafka.
This is ideal for applications within immutable containers without a writable filesystem.


## Example
This is an example `logback.xml` that uses a common `PatternLayout` to encode a log message as a string.

Add `logback-kafka-appender` and `logback-classic` as libraray dependencies to your project (maven example).

```xml
[pom.xml]
<dependency>
    <groupId>com.github.danielwegener</groupId>
    <artifactId>logback-kafka-appender</artifactId>
    <version>0.0.1</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.1.2</version>
</dependency>
```


```xml
[src/main/resources/logback.xml]
<configuration>
   <appender name="kafkaAppender" class="com.github.danielwegener.logback.kafka.KafkaAppender">
       <encoder class="com.github.danielwegener.logback.kafka.encoding.PatternLayoutKafkaEncoder">
           <layout class="ch.qos.logback.classic.PatternLayout">
               <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
           </layout>
       </encoder>
       <topic>logs</topic>
       <producerConfig>bootstrap.servers=localhost:9092</producerConfig>
    </appender>
    <root level="info">
        <appender-ref ref="kafkaAppender" />
    </root>
</configuration>

```


## Full configuration example

TBD

### Delivery strategies

Direct logging over the network is not a trivial thing because it might be much less reliable than the local file system and has a much bigger impact on the application performance if the transport has hiccups.

You need make a essential decision: Is it more important to deliver all logs to the remote Kafka or is it more important to keep the application running smoothly? Either of this decisions allows you to tune this appender for throughput.

#### Fallback-Appender

If, for whatever reason, the kafka-producer decides that it cannot publish a log message, the message could still be logged to a fallback appender (a `ConsoleAppender` on STDOUT or STDERR would be a reasonable choice for that).

Just add your fallback appender(s) as `appender-ref` to the `KafkaAppender` section in your `logback.xml`. Every message that cannot be delivered to kafka will be written to these appenders eventually.

Note that the `AsynchronousDeliveryStrategy` will reuse the producers IO-Thread to write the message onto the fallback appenders. Thus the fallback appenders should be reasonable fast so it does not slow down.


### Producer tuning

This appender uses the [_new_ kafka producer](https://kafka.apache.org/documentation.html#newproducerconfigs) introduced in kafka-0.8.2.
It uses the producer default configuration.

You may override any known kafka producer config with an `<producerConfig>Name=Value</producerConfig>` block (note that the `boostrap.servers` config is mandatory).
This allows a lot of fine tuning potential (eg. with `batch.size`, `compression.type` and `linger.ms`).

## Serialization

Kafka ships with an `PatternLayoutKafkaEncoder` that works like a common `PatternLayoutEncoder`
(with the distinction that it creates kafka message payloads instead of appending to a synchronous `OutputStream`).

The `PatternLayoutKafkaEncoder` takes a common `ch.qos.logback.core.Layout` as layout-parameter.

This allows you to use any layout that is capable of laying out an `ILoggingEvent` like for example the
[logstash-logback-encoder's `LogstashLayout`](https://github.com/logstash/logstash-logback-encoder#usage).

### Custom Serialization

TBD Just roll your own `KafkaMessageEncoder`. The interface is quite simple:

```java
package com.github.danielwegener.logback.kafka.encoding;
public interface KafkaMessageEncoder<E> {
    byte[] doEncode(E loggingEvent);
}

```
Your encoder should be type-parameterized for any subtype of ILoggingEvent like in
```java
public class MyEncoder extends KafkaMessageEncoderBase<ILoggingEvent> { //...
```

You may also extend The `KafkaMessageEncoderBase` class that already implements the `ContextAware` and `Lifecycle` interfaces.

## Delivery strategies

TBD

### Custom delivery strategies

TBD


## Keying strategies / Partitioning

Kafka's scalability and ordering guarantees heavily rely on the concepts of partitions ([more details here](https://kafka.apache.org/documentation.html#introduction)).
For application logging this means that we need to decide how we want to distribute our log messages over multiple kafka
topic partitions. One implication of this decision is how messages are ordered when they are consumed from a
arbitrary multi-partition consumer since kafka only provides a guaranteed read order only on each single partition.
Another implication is how evenly our log messages are distributed across all available partitions and therefore balanced
between multiple brokers.

The order may or may not be important, depending on the intended consumer-audience (e.g. a logstash indexer will reorder all message by its timestamp anyway).
The kafka producer client uses a messages key as partitioner. Thus `logback-kafka-appender` supports the following partitioning strategies:

| Strategy   | Description  |
|---|---|
| `RoundRobinPartitioningStrategy` (default)   | Evenly distributes all written log messages over all available kafka partitions. This strategy may lead to unexpected read orders on clients.   |
| `HostNamePartitioningStrategy` | This strategy uses the HOSTNAME to partition the log messages to kafka. This is useful because it ensures that all log messages issued by this host will remain in the correct order for any consumer. But this strategy can lead to uneven log distribution for a small number of hosts (compared to the number of partitions). |
| `ContextNamePartitioningStrategy` |  This strategy uses logbacks CONTEXT_NAME to partition the log messages to kafka. This is ensures that all log messages logged by the same logging context will remain in the correct order for any consumer. But this strategy can lead to uneven log distribution for a small number of hosts (compared to the number of partitions).  |
| `ThreadNamePartitioningStrategy` |  This strategy uses the calling threads name as partitioning key. This ensures that all messages logged by the same thread will remain in the correct order for any consumer. But this strategy can lead to uneven log distribution for a small number of thread(-names) (compared to the number of partitions). |
| `LoggerNamePartitioningStrategy` | * This strategy uses the logger name as partitioning key. This ensures that all messages logged by the same logger will remain in the correct order for any consumer. But this strategy can lead to uneven log distribution for a small number of distinct loggers (compared to the number of partitions). |



### Custom keying strategies

If none of the above partitioners satisfies your requirements, you can easily implement your own partitioner by implementing a custom `KeyingStrategy`:

```java
package foo;
com.github.danielwegener.logback.kafka.keying.KeyingStrategy;

public class LevelKeyingStrategy implements KeyingStrategy {
    @Override
    public byte[] createKey(ILoggingEvent e) {
        return ByteBuffer.allocate(4).putInt(e.getLevel()).array();
    }
}
```

As most custom logback component, your custom partitioning strategy may implement the
`ch.qos.logback.core.spi.ContextAware` and `ch.qos.logback.core.spi.LifeCycle` interfaces.

A custom keying strategy may especially become handy when you want to use kafka's log compactation facility.

## FAQ

- Q: I want to log to different/multiple topics!
- A: No problem, create an appender for each topic.


## License

This project is licensed under the [Apache License Version 2.0](LICENSE).

