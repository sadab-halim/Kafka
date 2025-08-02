# Kafka

## Core Concepts and Architecture (Kafka Essentials)
### What is Apache Kafka?
Apache Kafka is a distributed streaming platform designed for building real-time data pipelines and streaming applications. It functions as a highly scalable, fault-tolerant, and durable distributed commit log. Unlike traditional message queues that delete messages after consumption, Kafka retains messages for a configurable period, allowing multiple consumers to read the same data stream independently and at their own pace.

### Use Cases in Java-based Development:
- **Log Aggregation**: Centralizing logs from various applications and servers into a single, scalable platform for real-time monitoring and analysis. (e.g., Microservices emitting logs to Kafka, consumed by ELK stack or Splunk).
- **Event Sourcing**: Storing a sequence of state-changing events as the primary source of truth for an application's state, enabling historical replay, auditing, and easier debugging. (e.g., User actions in an e-commerce platform like "Item Added to Cart," "Order Placed").
- **Audit Systems**: Maintaining an immutable, ordered, and durable record of all significant actions within a system for compliance, security, and debugging purposes.
- **Real-time Analytics**: Ingesting and processing data streams immediately for dashboards, anomaly detection, fraud detection, and personalized recommendations. (e.g., Clickstream analysis, financial transaction monitoring).
- **Microservice Communication**: Enabling asynchronous, decoupled communication between microservices, where services publish events and other services subscribe to those events without direct dependencies. (e.g., "Order Service" publishes "Order Placed" event, "Shipping Service" consumes it).

### Components
- **Topics**: A named feed of messages. Similar to a table in a database or a folder in a filesystem. Producers write data to topics, and consumers read from them. Topics are logically divided into partitions.
- **Partitions**: Topics are split into partitions for scalability and parallelism. Each partition is an ordered, immutable sequence of messages (records). A message in a partition is identified by its offset. Messages within a partition are strictly ordered.
- **Producers**: Client applications that publish (write) messages to Kafka topics. Producers choose which partition to write to (round-robin by default, or based on a key).
- **Consumers**: Client applications that subscribe to topics and read (consume) messages from one or more partitions. Consumers belong to consumer groups.
- **Brokers**: A Kafka server. A Kafka cluster consists of one or more brokers. Brokers store topic partitions, handle message production and consumption requests, and replicate data.
- **Clusters**: A group of Kafka brokers working together. A Kafka cluster provides high availability and scalability.
- **ZooKeeper**: (Historically) A distributed coordination service used by Kafka for managing broker metadata, leader election for partitions, and keeping track of access control lists (ACLs). 
- **KRaft (Kafka Raft Metadata)**: (Modern Kafka) Starting with Kafka 2.8 and fully production-ready in 3.0+, KRaft is a new consensus protocol that replaces ZooKeeper for metadata management. It embeds the metadata quorum directly into Kafka brokers, simplifying the architecture and improving scalability. This is a significant shift, making Kafka self-contained.

### Distributed Log Architecture, Partitioning Strategy, Replication, Leader-Follower Model
- **Distributed Log Architecture**: At its heart, Kafka is a distributed commit log. Each partition is an append-only sequence of records. Producers append records to the end of a partition. Consumers read records from a specific offset within a partition. The "log" is distributed across brokers and replicated for fault tolerance.
- **Partitioning Strategy**: When a producer sends a message to a topic, Kafka determines which partition it goes into.
  - **Round-robin**: If no key is provided, messages are distributed evenly across partitions.
  - **Key-based**: If a key is provided, all messages with the same key are guaranteed to go to the same partition. This is crucial for maintaining order for related events.
  - **Custom Partitioner**: Developers can implement custom partitioning logic.
- **Replication**: Each partition can have multiple replicas spread across different brokers. This ensures data durability and high availability. If a broker fails, a replica can take over.
  - **Replication Factor**: The number of copies of each partition. A replication factor of 3 means there are 3 copies of each partition (one leader, two followers).
- **Leader-Follower Model**: For each partition, one broker is designated as the **leader**, and the others are **followers**.
  - Producers always write to the leader.
  - Consumers typically read from the leader, but can be configured to read from followers for specific use cases (though this is less common and adds complexity).
  - Followers asynchronously replicate data from the leader. If the leader fails, one of the in-sync followers is elected as the new leader

### Kafka Guarantees: Durability, High Availability, Scalability, Performance
- **Durability**: Messages are persisted to disk on brokers and replicated across multiple brokers. This ensures that even if a broker fails, messages are not lost.
- **High Availability**: Through replication and the leader-follower model, Kafka can tolerate broker failures without downtime for topic partitions. If a leader fails, a new leader is automatically elected.
- **Scalability**: Achieved by partitioning topics and distributing partitions across multiple brokers. You can add more brokers and partitions to scale throughput and storage horizontally.
- **Performance**: Kafka's design leverages sequential disk I/O, batching, and zero-copy principles, allowing it to achieve very high throughput and low latency, even with large volumes of data.

### Kafka vs RabbitMQ vs Pulsar - Design Decisions and When to Use Which

---

## Kafka Setup, Configuration & Tooling
### Kafka Installation (Standalone - Local Development)
For quick local setup, you typically download the Kafka binary distribution and run it. This often involves starting ZooKeeper (if using an older version or not using KRaft mode with standalone Kafka) and then a single Kafka broker.

#### Example (assuming Kafka binaries are downloaded and extracted)
```bash
# Start ZooKeeper (if not using KRaft)
bin/zookeeper-server-start.sh config/zookeeper.properties

# Start Kafka Broker (in standalone mode)
bin/kafka-server-start.sh config/server.properties
```

### Kafka Installation (Cluster - Production)
In production, Kafka is deployed as a cluster of multiple brokers, often across different machines or Kubernetes pods, for high availability and fault tolerance. This involves:
1. **Distributed ZooKeeper Ensemble (if applicable)**: Setting up an odd number of ZooKeeper servers (3 or 5 for fault tolerance).
2. **Multiple Kafka Brokers**: Each broker configured to be part of the same cluster, referencing the ZooKeeper ensemble or a peer broker in KRaft mode.
3. **Network Configuration**: Ensuring proper network connectivity and firewall rules between brokers and clients.
4. **Disk Management**: Dedicated, high-performance disks for Kafka logs

### Configuration of Brokers (server.properties)
The `server.properties` file is the main configuration file for a Kafka broker. Key configurations include:
- `broker.id`: Unique identifier for each broker in the cluster.
- `log.dirs`: Comma-separated list of directories where Kafka logs (topic partitions) are stored. Crucial for performance and durability.
- `zookeeper.connect` (or `metadata.quorum.uri` for KRaft): Connection string to the ZooKeeper ensemble (e.g., `localhost:2181`) or the KRaft quorum.
- `listeners`: The network interfaces and ports the broker listens on (e.g., `PLAINTEXT://localhost:9092`, `SSL://myhost:9093`).
- `advertised.listeners`: The address that clients (producers/consumers) should use to connect to the broker. Essential in environments with NAT or Docker.
- `num.partitions`: Default number of partitions for new topics if not specified by the producer.
- `log.retention.hours`: How long messages are retained before being deleted (e.g., 168 hours = 7 days).
- `auto.create.topics.enable`: Whether Kafka should automatically create topics when a producer attempts to write to a non-existent topic (default: true, but often disabled in production).
- `offsets.topic.replication.factor`: Replication factor for the internal __consumer_offsets topic.
- `min.insync.replicas`: Minimum number of in-sync replicas that must acknowledge a write for it to be considered successful by the producer

### Topic Creation
Topics can be created via the CLI, programmatic APIs, or UI tools.

#### CLI Example
```bash
# Create a topic named 'my-topic' with 3 partitions and a replication factor of 2
bin/kafka-topics.sh --create --topic my-topic --bootstrap-server localhost:9092 --partitions 3 --replication-factor 2
```

### CLI Tools
Kafka provides a set of command-line tools for administration and basic interactions:
#### `kafka-topics.sh`: Manage topics (create, list, describe, delete)
```bash
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
bin/kafka-topics.sh --describe --topic my-topic --bootstrap-server localhost:9092
```

#### `kafka-console-producer.sh`: Produce messages from the console.
```bash
bin/kafka-console-producer.sh --topic my-topic --bootstrap-server localhost:9092
> Hello Kafka!
> This is a test.
```

#### `kafka-console-consumer.sh`: Consume messages to the console.
```bash
bin/kafka-console-consumer.sh --topic my-topic --bootstrap-server localhost:9092 --from-beginning
````

#### `kafka-consumer-groups.sh`: Manage consumer groups (list, describe, reset offsets).
```bash
bin/kafka-consumer-groups.sh --list --bootstrap-server localhost:9092
bin/kafka-consumer-groups.sh --describe --group my-consumer-group --bootstrap-server localhost:9092
```

### Kafka UI Tools
- Graphical user interfaces greatly simplify monitoring and managing Kafka clusters:
- **Kafka Manager (deprecated/community)**: Yahoo's open-source tool for managing Kafka clusters. Offers features like cluster overview, topic management, consumer group monitoring.
- **Confluent Control Center**: A comprehensive web-based tool from Confluent (commercial) for managing, monitoring, and debugging Kafka clusters, including Kafka Connect and Kafka Streams applications.
- **AKHQ (A-Know-How-Quickly)**: A popular open-source web UI for Kafka, providing rich features for topic, consumer group, and schema registry management.
- **Kafdrop**: A lightweight open-source web UI for viewing Kafka topics and Browse messages. Useful for simple debugging and monitoring.

### Kafka with Docker, Kubernetes, and Strimzi Operator
- **Docker**: Kafka brokers can be easily containerized using Docker images (e.g., `confluentinc/cp-kafka`). This simplifies local development setups and consistent deployments.
- **Kubernetes**: For production-grade deployments, Kafka is often deployed on Kubernetes. This provides orchestration, scaling, and self-healing capabilities.
- **Strimzi Operator**: A Kubernetes Operator that simplifies the deployment, management, and operation of Kafka clusters on Kubernetes. Strimzi automates tasks like provisioning, scaling, upgrades, and managing configurations, making Kafka a first-class citizen in Kubernetes. It uses Custom Resources to define Kafka components.

---

## Kafka Programming with Java (Core API)
The `kafka-clients` library provides the core Java APIs for interacting with Kafka

### Java-based Producer and Consumer Clients
#### Producer Example
```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

public class MyKafkaProducer {
    public static void main(String[] args) {
        Properties properties = new Properties();
        properties.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        properties.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        properties.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        KafkaProducer<String, String> producer = new KafkaProducer<>(properties);

        for (int i = 0; i < 10; i++) {
            String topic = "my-topic";
            String key = "id_" + (i % 3); // Example key for partitioning
            String value = "Hello Kafka message " + i;

            ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);

            producer.send(record, new Callback() {
                @Override
                public void onCompletion(RecordMetadata metadata, Exception exception) {
                    if (exception == null) {
                        System.out.println("Received new metadata. \n" +
                                "Topic: " + metadata.topic() + "\n" +
                                "Partition: " + metadata.partition() + "\n" +
                                "Offset: " + metadata.offset() + "\n" +
                                "Timestamp: " + metadata.timestamp());
                    } else {
                        System.err.println("Error while producing: " + exception.getMessage());
                    }
                }
            });
        }

        producer.flush(); // Ensure all buffered records are sent
        producer.close();
    }
}
```

#### Consumer Example
```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class MyKafkaConsumer {
    public static void main(String[] args) {
        Properties properties = new Properties();
        properties.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        properties.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        properties.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        properties.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "my-java-consumer-group");
        properties.setProperty(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest"); // "earliest", "latest", "none"

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(properties);
        consumer.subscribe(Collections.singletonList("my-topic"));

        // Poll for new data
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100)); // Poll every 100ms

            for (ConsumerRecord<String, String> record : records) {
                System.out.println("Key: " + record.key() +
                        ", Value: " + record.value() +
                        ", Partition: " + record.partition() +
                        ", Offset: " + record.offset());
            }
            // For automatic commit (default): offsets are committed periodically
            // For manual commit, see section below.
        }
        // consumer.close(); // In a real application, you'd handle shutdown gracefully
    }
}
```

#### Producer Configuration
- `acks`: Controls the durability level of messages sent by the producer.
- `0`: Producer doesn't wait for any acknowledgment. Fastest but least durable (data loss possible).
- `1`: Producer waits for the leader to acknowledge the write. Good balance of latency and durability.
- `all (-1)`: Producer waits for the leader and all in-sync replicas to acknowledge. Strongest durability guarantee, slowest. Used for "exactly-once" semantics with transactions.
- `retries`: Number of times the producer will retry sending a message if it fails.
- `linger.ms`: The time the producer will wait before sending a batch of messages. Reduces network requests by batching messages.
- `batch.size`: The maximum size in bytes of a single batch of messages.
- `idempotence (enable.idempotence=true)`: Ensures that retried messages are not duplicated on the broker side, even if multiple sends succeed. Essential for "exactly-once" delivery with a single producer.

#### Consumer Configuration
- `group.id`: A unique string that identifies the consumer group this consumer belongs to. All consumers with the same `group.id` will share the partitions of a topic.
- `enable.auto.commit`: If `true` (default), offsets are committed automatically in the background at intervals defined by `auto.commit.interval.ms`.
- `max.poll.records`: The maximum number of records returned in a single call to `poll()`.
- `auto.offset.reset`: What to do when there is no initial offset in Kafka or if the current offset does not exist any more on the server.
  - `earliest`: Automatically reset the offset to the earliest available offset.
  - `latest`: Automatically reset the offset to the latest offset (i.e., start consuming new messages).
  - `none`: Throw an exception if no previous offset is found

### Manual vs Automatic Offset Commits
- **Automatic Commit (**`enable.auto.commit=true`**)**: Kafka periodically commits the highest offset processed by the consumer. Simplest to use but can lead to "at-least-once" (duplicates on rebalance/crash) or "at-most-once" (data loss on crash before commit) depending on `auto.commit.interval.ms`.
- **Manual Commit (**`enable.auto.commit=false`**)**: Gives the developer explicit control over when offsets are committed. This is crucial for achieving "at-least-once" or "exactly-once" processing guarantees.
  - `consumer.commitSync()`: Synchronous commit. Blocks until the commit is successful. Recommended for critical applications.
  - `consumer.commitAsync()`: Asynchronous commit. Non-blocking, faster, but requires error handling for failed commits

### Manual Commit Example
```java
// ... consumer setup ...
properties.setProperty(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
// ...

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        // Process record
    }
    consumer.commitSync(); // Commit offsets after processing the batch
}
```

### Partitioning Logic, Custom Partitions, Interceptors
- **Partitioning Logic**: 
  - If a message key is provided, Kafka's default partitioner uses a hash of the key to determine the partition. This ensures all messages with the same key go to the same partition, preserving order for events related to that key.
  - If no key is provided, messages are distributed in a round-robin fashion across available partitions.
- **Custom Partitions**: 
  - You can implement the `org.apache.kafka.clients.producer.Partitioner` interface to define your own logic for mapping messages to partitions. This is useful for specific routing requirements or load balancing strategies
  ```java
  // Example: send all messages to partition 0
  public class MyCustomPartitioner implements Partitioner {
  @Override
  public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
  return 0; // Or implement custom logic based on key, value, etc.
  }
  // ... other methods ...
  }
  // Set in producer config: properties.setProperty(ProducerConfig.PARTITIONER_CLASS_CONFIG, MyCustomPartitioner.class.getName());
  ```
- **Interceptors**: Provide a way to intercept and modify messages before they are sent by the producer or after they are received by the consumer
  - `ProducerInterceptor`: Can modify records, add headers, or gather metrics before `send()`.
  - `ConsumerInterceptor`: Can modify `ConsumerRecords`, filter them, or gather metrics after `poll()`.
  - Useful for logging, metrics, tracing, or data transformation

---

## Kafka and Spring Boot (Spring Kafka Integration)

### Spring Kafka Configuration with `application.yml` and `KafkaTemplate`
#### `application.yml`
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      acks: all
      properties:
        enable.idempotence: true # For exactly-once producer
    consumer:
      group-id: spring-kafka-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false # Typically false for manual control
      properties:
        spring.json.trusted.packages: "*" # Required for JSON deserialization of complex types
```

#### `KafkaTemplate` for Producing
```java
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class MySpringProducer {

    private final KafkaTemplate<String, String> kafkaTemplate;

    public MySpringProducer(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendMessage(String topic, String key, String message) {
        kafkaTemplate.send(topic, key, message)
                .whenComplete((result, ex) -> {
                    if (ex == null) {
                        System.out.println("Sent message=[" + message +
                                "] with offset=[" + result.getRecordMetadata().offset() + "]");
                    } else {
                        System.err.println("Unable to send message=[" + message + "] due to : " + ex.getMessage());
                    }
                });
    }

    public void sendTransactionalMessage(String topic, String key, String message) {
        // Example for transactional sending (requires transaction manager config)
        kafkaTemplate.executeInTransaction(kafkaTemplate -> {
            kafkaTemplate.send(topic, key, message);
            // Optionally send to another topic or perform other transactional operations
            return true;
        });
    }
}
```

#### `KafkaListener` Annotations and Listener Containers
Spring Kafka provides the `@KafkaListener` annotation to easily create message-driven POJOs (Plain Old Java Objects) that consume messages from Kafka topics.

```java
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

@Component
public class MySpringConsumer {

    @KafkaListener(topics = "my-topic", groupId = "spring-kafka-group")
    public void listen(@Payload String message,
                       @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                       @Header(KafkaHeaders.OFFSET) long offset) {
        System.out.println("Received message: " + message + " from partition " + partition + " with offset " + offset);
        // Spring handles offset commits automatically by default, or manually if enable-auto-commit is false
    }

    // Example with manual ack
    @KafkaListener(topics = "my-topic-manual-ack", groupId = "manual-ack-group", containerFactory = "kafkaListenerContainerFactory")
    public void listenManualAck(@Payload String message, Acknowledgment acknowledgment) {
        System.out.println("Received message for manual ack: " + message);
        // Process message...
        acknowledgment.acknowledge(); // Manually commit the offset
    }
}
```

Behind the scenes, `@KafkaListener` uses `ConcurrentMessageListenerContainer` to manage consumer threads, polling, and offset commits.

### Batch Consumption, Error Handling, Retries, Backoff Policies
- **Batch Consumption**: Process multiple messages in a single listener invocation. Requires configuring a `BatchMessageConverter` and a `ConcurrentKafkaListenerContainerFactory` for batch listeners
  ```java
  @KafkaListener(topics = "my-topic", groupId = "batch-group", containerFactory = "batchKafkaListenerContainerFactory")
  public void listenBatch(@Payload List<String> messages,
                          @Header(KafkaHeaders.RECEIVED_PARTITION_ID) List<Integer> partitions,
                          @Header(KafkaHeaders.OFFSET) List<Long> offsets) {
      for (int i = 0; i < messages.size(); i++) {
          System.out.println("Batch message: " + messages.get(i) + " from partition " + partitions.get(i) + " at offset " + offsets.get(i));
      }
  }
  ```
- **Error Handling**: Spring Kafka provides robust error handling mechanisms.
  - `ErrorHandler`: Interface for custom error handling logic when a listener throws an exception. 
  - `ContainerProperties.setErrorHandler()`: Set an error handler on the listener container
- **Retries and Backoff Policies**: Spring Kafka integrates with Spring Retry
  - You can configure `DefaultErrorHandler` or `SeekToCurrentErrorHandler` with retry policies (e.g., exponential backoff) to re-process failed messages. 
  - If a message repeatedly fails, it can be sent to a Dead Letter Topic (DLT).

### Transactional Messaging with Spring Kafka + `@KafkaListener`
Spring Kafka supports transactional producers and consumers, enabling "exactly-once" processing guarantees across multiple Kafka topics or even across Kafka and an external database.

1. **Producer Side**: Configure `KafkaTransactionManager` and enable transactions for `KafkaTemplate`.
  ```java
  @Configuration
  public class KafkaConfig {
      @Bean
      public KafkaTransactionManager<String, String> kafkaTransactionManager(
              ProducerFactory<String, String> producerFactory) {
          return new KafkaTransactionManager<>(producerFactory);
      }
  
      @Bean
      public KafkaTemplate<String, String> kafkaTemplate(ProducerFactory<String, String> producerFactory) {
          return new KafkaTemplate<>(producerFactory);
      }
  }
  ```
Then use `kafkaTemplate.executeInTransaction(...)` or `@Transactional` with `KafkaTemplate` methods

2. **Consumer Side**: When `enable-auto-commit` is `false` and `idempotence` is `true` for the consumer, coupled with manual commits within a transaction, you can achieve "exactly-once" semantics (EOS). The consumer's offset commit is part of the transaction.
  ```java
  // Consumer configured with isolation.level: read_committed
  @KafkaListener(topics = "source-topic", groupId = "tx-consumer-group")
  @Transactional // Requires JTA or Spring's PlatformTransactionManager setup
  public void processAndProduce(@Payload String message) {
      // Process the message
      // ...
      // Produce a new message to an output topic within the same transaction
      kafkaTemplate.send("output-topic", "processed-" + message);
      // If an exception occurs, both consumption and production are rolled back.
  }
  ```

### Dead Letter Topics (DLTs) and Retry Topics
- **Dead Letter Topics (DLTs)**: A dedicated topic where messages that repeatedly fail processing (after retries are exhausted) are sent. This prevents "poison pill" messages from blocking the consumer and allows for later inspection and manual intervention.
  Spring Kafka's `DefaultErrorHandler` can be configured to send failed messages to a DLT.
- **Retry Topics**: Instead of immediate retries within the same `poll()` cycle, messages are sent to a sequence of retry topics (e.g., `original-topic.retry.1s`, `original-topic.retry.5s`, `original-topic.retry.10s`). Each retry topic has a different delay configured using Kafka's time-based message retention or a message timestamp. This allows the application to recover from transient issues (e.g., database connection outage) without blocking the main consumer. After a few retries, if still failing, it goes to the DLT. Spring Kafka's `DefaultErrorHandler` supports DLTs and retry topics out-of-the-box.

---

## Serialization and Schema Management
Kafka messages are essentially byte arrays. To make sense of these bytes, data must be serialized by producers and deserialized by consumers

### Kafka Serializers: `StringSerializer`, `ByteArraySerializer`, Avro, Protobuf, JSON, Custom
- `StringSerializer`/`StringDeserializer`: Simplest, but only for plain strings. Good for quick tests.
- `ByteArraySerializer`/`ByteArrayDeserializer`: Most basic, as Kafka fundamentally works with byte arrays. You manually convert objects to and from byte arrays.
- `JsonSerializer`/`JsonDeserializer`: Popular for human-readable data. Uses libraries like Jackson to convert JSON strings to Java objects. Less efficient and lacks strong schema enforcement compared to binary formats.
  - **Limitations**: Schema evolution can be tricky without a schema registry. Requires `spring.json.trusted.packages` for Spring Kafka when deserializing complex types.
- **Avro**: A data serialization system that provides rich data structures with a compact binary format. It heavily relies on schemas.
  - **Advantages**: Language-agnostic, efficient binary format, excellent schema evolution support (backward and forward compatibility).
- **Protobuf (Protocol Buffers)**: Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data.
  - **Advantages**: Very efficient, strongly typed, schema definition using `.proto` files.
- **Custom Serializers/Deserializers**: You can implement `org.apache.kafka.common.serialization.Serializer` and `org.apache.kafka.common.serialization.Deserializer` interfaces for custom object serialization (e.g., custom binary formats, specific domain objects)

### Schema Registry (Confluent), Schema Evolution, Compatibility Rules
- **Schema Registry**: A standalone service (most commonly from Confluent) that stores a versioned history of schemas for Kafka message keys and values. It allows producers to register schemas and consumers to retrieve them, ensuring compatibility between different versions of data.
- **How it Works:**
  1. Producer sends data with a schema ID.
  2. Serializer (e.g., AvroSerializer) talks to Schema Registry to register/retrieve schema and then serializes the data.
  3. Consumer receives data, extracts schema ID, talks to Schema Registry to retrieve the corresponding schema, and then deserializes the data.
- **Schema Evolution**: The ability to change a schema over time (e.g., add new fields, remove old fields) without breaking existing producers or consumers.
- **Compatibility Rules (in Schema Registry)**:
  - **NONE**: No compatibility checks. 
  - **FORWARD**: New consumers can read old data. (e.g., adding a new field with a default value). 
  - **BACKWARD**: Old consumers can read new data. (e.g., removing an optional field). 
  - **FULL**: Both forward and backward compatible. (e.g., adding a new optional field with a default value). 
  - **TRANSITIVE**: Same as above, but checks compatibility against all prior versions, not just the immediate previous one. 
  - **Configuring**: `properties.put("schema.registry.url", "http://localhost:8081");` for serializers/deserializers.

