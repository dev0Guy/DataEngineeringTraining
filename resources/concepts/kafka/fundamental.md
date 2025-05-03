# Kafka Fundamentals
At its core, Kafka is built on a few fundamental concepts:

- **Topics**: Named channels through which data flows. Producers write messages to topics; consumers read from them.
- **Partitions**: Each topic is split into one or more partitions, enabling parallelism and scalability.
- **Offsets**: A unique ID for each message within a partition. Consumers use offsets to keep track of their position in the log.
- **Brokers**: Kafka servers that store data and serve client requests.
- **Producers and Consumers**: Producers write data to Kafka; consumers read and process that data.
- **Consumer Groups**: Allow for horizontal scaling of consumption, where each consumer processes a unique set of partitions.

Kafka is widely used for **messaging**, **log aggregation**, **real-time stream processing**, **metrics collection**, and **data integration** with big data tools like **Apache Flink**, **Spark**, and **Hadoop**.

By understanding these core building blocks, developers and data engineers can design scalable and fault-tolerant systems for a wide range of use cases.


## Topics

A **Kafka topic** is a named stream of data, conceptually similar to a table in a database like MongoDB — but much more lightweight and flexible.

- Topics are identified by **unique names**.
- A topic can handle **any message format** (e.g., JSON, Avro, Protobuf), because Kafka stores data as **byte arrays**.
- A **datastream** refers to the continuous flow of messages within a topic.
- Topics are **append-only**: once data is written, it cannot be updated or deleted.
- **Kafka does not support querying data** like a traditional database.

> ✅ Topics act as the communication channel between producers and consumers.



## Partitions

To enable scalability and parallelism, **Kafka topics are divided into partitions**.

- Each partition is an **ordered, immutable sequence** of messages.
- **Message order is guaranteed only within a single partition**, not across the whole topic.
- Each message in a partition is assigned a unique and **sequential ID called an _offset_**.
- **Partitions can be distributed across brokers**, allowing Kafka to scale horizontally.
- Once a message is written to a partition, it **cannot be changed** — it’s immutable.

> ✅ More partitions allow higher throughput but increase complexity for ordering and replication.



## Offsets

An **offset** is a unique, monotonically increasing ID assigned to every message within a partition.

- Offsets are **local to the partition** — offset `42` in Partition 0 is unrelated to offset `42` in Partition 1.
- Offsets are used by consumers to track their **read progress**.
- Kafka stores offsets externally (e.g., in Kafka itself or in Zookeeper), so consumers can resume from where they left off.
- Consumers are responsible for **committing offsets** to avoid data loss or duplication.



## Producers

Kafka **producers** are clients that write data to topics.

- Producers send messages to a specific **topic and partition**.
- They are aware of the **Kafka cluster topology** and can determine which broker holds which partition.
- In case of **broker failure**, producers can automatically retry and recover.
- Each message can optionally include a **key**:
    - If the key is `null`, Kafka uses **round-robin** strategy to distribute messages across partitions.
    - If the key is **not null**, Kafka applies a **hash function** (typically Murmur2) to the key.
        - This ensures that all messages with the same key go to the **same partition**, preserving **message order**.

### Kafka Message Structure

Each Kafka message consists of:

- `key`: Binary (optional)
- `value`: Binary (required)
- `compressionType`: e.g., gzip, snappy, lz4 (optional)
- `headers`: Key-value metadata (optional)
- `timestamp`: Set by the producer or Kafka cluster
- `partition`: Partition number
- `offset`: Message offset in the partition

> ✅ The ability to route by key enables **strong message ordering guarantees per key**.



## Consumers

Kafka **consumers** are clients that **read data from topics using a pull model**.

- Consumers **pull messages** from Kafka topics rather than having messages pushed to them.
- They automatically **discover the correct broker and partition** to read from.
- In case of **broker or consumer failure**, the Kafka client handles **automatic recovery**.
- Messages are read **in order (by offset)** within each partition — from the lowest to the highest offset.

### Deserialization

Since Kafka stores everything as bytes, consumers must **deserialize** the key and value:

- Each consumer must define a **deserializer** for the key and one for the value.
- Deserializers convert the raw bytes into usable application objects (e.g., JSON, Avro, Protobuf).
- The serialization format must be **known ahead of time** and **consistent across producers and consumers**.
- Changing the message format (e.g., from JSON to Avro) **requires creating a new topic** — format changes mid-topic are not supported.



## Consumer Groups

Kafka enables **horizontal scalability** of consumers through **consumer groups**:

- A **consumer group** is a group of consumers **working together** to read data from a topic.
- Within a group:
    - Each partition is assigned to **only one consumer**.
    - If there are more consumers than partitions, some consumers will be **idle**.
    - If there are more partitions than consumers, some consumers will **handle multiple partitions**.
- Kafka ensures **load balancing** and **fault tolerance** within the group.

> ✅ Consumer groups allow scaling consumption without duplicating data processing.



### Offset Management

Kafka uses **offsets** to track which messages have been read by a consumer:

- Each **message** within a **partition** has a unique **offset** (a monotonically increasing ID).
- Each **consumer** keeps track of the **last processed offset**.
- Kafka stores this progress in a **special internal topic**:  
  `__consumer_offsets`
- Consumers must **commit offsets** to persist their read position. This allows them to **resume** correctly after a restart or crash.

> Committed offsets ensure consumers don’t reprocess or lose messages unintentionally.

#### Delivery Guarantees Based on Offset Commit Timing

| Guarantee         | Description                                                                 |
|-|--|
| At Most Once      | Offset is committed **before** processing. If the process crashes, data is lost. |
| At Least Once     | Offset is committed **after** processing. On crash, Kafka re-sends the message (may lead to duplicates). |
| Exactly Once      | Uses Kafka **transactions** to process and commit offset atomically. Requires additional configuration and broker support.|

Kafka clients **automatically commit offsets periodically** unless set to manual mode.



### Acknowledgment Settings (`acks`)

When a **producer** sends data to Kafka, the **`acks` setting** determines the level of **acknowledgment** required from the broker before considering the message successfully sent.

| `acks` Value | Description                                                                                  | Data Safety     |
|--|-|--|
| `0`          | Producer does **not wait** for any acknowledgment. No guarantee the message was received.    | ❌ Possible loss |
| `1`          | Producer waits for **ack from the leader** of the partition. Fast, but leader failure may cause loss. | ⚠️ Somewhat safe |
| `all`        | Producer waits for **ack from all in-sync replicas (ISR)**. Ensures highest durability.      | ✅ Safe |

> The higher the acks level, the **more durable** (but slower) the write becomes.

Kafka producers also support **retry**, **idempotency**, and **batching**, which further impact reliability and performance.


## Kafka Broker

A **Kafka broker** is a server that stores and manages topic data. A Kafka **cluster** is composed of **multiple brokers** working together.

- Each broker is identified by a unique **integer ID**.
- **Topics are split into partitions**, and **partitions are distributed across brokers**.
- A broker may hold **one or more partitions** for different topics.

### Connecting to a Kafka Cluster

Kafka clients (producers or consumers) only need to connect to **one broker** (called a **bootstrap broker**) to discover the rest of the cluster:

- Once connected to a bootstrap broker, the client automatically **retrieves metadata** about:
    - All other brokers in the cluster
    - Topic-partition assignments
    - Partition leaders

This enables **full cluster awareness** even if the client only connects to a single broker initially.

### Broker and Partition Distribution

- If you have **N partitions** and **N brokers**, Kafka will ideally **distribute partitions evenly**, with **one partition per broker**.
- Kafka ensures **load balancing** across brokers and also supports **replication** for fault tolerance.



## Replication in Kafka

To ensure **fault tolerance and high availability**, Kafka uses **replication**:

- Each topic can be configured with a **replication factor** (usually **≥ 3**).
- **Replication factor** defines how many brokers will store copies of each partition.
- **Replication factor can never exceed the number of brokers** in the cluster.
- Kafka ensures that **each replica** of a partition is placed on a **different broker**.

### Leader and In-Sync Replicas (ISR)

- Every partition has exactly **one leader replica** — this is the only replica that **producers** and **consumers** interact with by default.
- Other replicas are called **in-sync replicas (ISR)** — they **continuously replicate data** from the leader.
- If the leader broker fails, Kafka **automatically promotes one of the ISR replicas** to become the new leader.

### Replica Fetching and Reading

- Since **Kafka 2.4**, it is possible to **configure consumers** to read from **follower replicas (not just the leader)**, reducing cross-data-center latency in geo-distributed setups.

> Kafka replication makes the system resilient to broker failures and supports high availability in production environments.