# Production-Ready Apache Kafka in Node.js (TypeScript)

When building distributed systems, microservices need to exchange information reliably. Synchronous HTTP/REST calls couple services together: if Service B is down, Service A fails too. 

**Apache Kafka** is a distributed, partitioned, replicated commit log service. It acts as an event streaming platform designed to handle massive volumes of real-time data feeds, allowing microservices to communicate asynchronously, reliably, and at extreme scale.

---

## 1. What is Apache Kafka? (Layman's Terms)

### The Daily Newspaper Analogy

Imagine a traditional newspaper printing and delivery company.

*   **Producers (The Journalists):** Journalists write stories (events/messages). They don't know who reads the paper; they just submit their articles to the publisher.
*   **Topics (The Newspaper Sections):** The publisher organizes stories into sections like *Sports*, *Finance*, and *Weather*.
*   **Partitions (The Printing Presses):** To print millions of newspapers quickly, the publisher runs multiple printing presses in parallel. Each press prints a subset of the pages.
*   **Consumers (The Readers):** Readers buy the newspaper and read the sections they care about. Some readers only read the *Sports* section, while others read *Finance*.
*   **Consumer Groups (A Household):** If a household of 3 people subscribes to the paper, they might split the sections: Alice reads *Sports*, Bob reads *Finance*, and Charlie reads *Weather*. No two people in the household read the exact same physical copy of a section at the same time—they distribute the work.
*   **Offsets (The Bookmark):** When you stop reading a section to answer the phone, you place a bookmark (Offset) on the last line you read so you can resume reading exactly where you left off.

---

## 2. Core Technical Concepts

To use Kafka effectively, you must understand its key architectural blocks:

```
PRODUCER ---> [ Kafka Topic ] ----------------------------------------+
                     |                                                |
            +--------+--------+                                       v
            v                 v                              CONSUMER GROUP
      Partition 0       Partition 1                             (Worker Fleet)
     [Msg 0][Offset 0] [Msg 0][Offset 0]                             /       \
     [Msg 1][Offset 1] [Msg 1][Offset 1]                            v         v
     [Msg 2][Offset 2]                                         Worker 1    Worker 2
            |                 |                                   |          |
            v                 v                                   v          v
     Assigned to ===> Worker 1                               Partition 0  Partition 1
     Assigned to ====> Worker 2
```

*   **Event (Message):** A record consisting of a **Key**, **Value**, **Timestamp**, and optional **Headers**.
*   **Topic:** A logical channel or category to which messages are published (e.g., `user-signup-events`). Topics are append-only logs.
*   **Partition:** A single physical log file inside a topic. Topics are split into multiple partitions distributed across brokers for scalability and parallel processing. **Order is only guaranteed within a single partition.**
*   **Offset:** A sequential, incremental integer assigned to each message inside a partition. It uniquely identifies the message's position.
*   **Broker:** A single Kafka server that stores data and serves client requests.
*   **Cluster:** A group of brokers working together for high availability and replication.
*   **Consumer Group:** A group of consumers that cooperate to consume messages from a topic. Kafka guarantees that each partition inside a topic is assigned to only **one** consumer inside a group at any given time.

---

## 3. Message Broker Comparison

Choosing the correct messaging system depends on your architecture requirements:

| Feature | Apache Kafka | RabbitMQ | Redis Pub/Sub |
| :--- | :--- | :--- | :--- |
| **Model** | Pull-based (log consumer logs progress). | Push-based (broker tracks delivery). | Push-based (fire-and-forget). |
| **Persistence** | **Yes (Durable).** Log files are kept on disk. | Yes (Transient/Configurable queues). | **No.** Messages are lost if no one is listening. |
| **Throughput** | **Ultra-High (Millions of messages/sec).** | High (Tens of thousands/sec). | Medium. |
| **Message Replay** | **Yes.** Consumers can reset offsets and re-read. | No (Messages are deleted after ACK). | No. |
| **Ordering** | Guaranteed within a partition. | Guaranteed within a queue. | No guarantees. |
| **Use Case** | Event sourcing, metrics tracking, audit logs. | Complex routing, task queues, RPC. | Live chat, real-time dashboards, caches. |

---

## 4. Local Environment Setup (Docker Compose)

Modern Kafka versions support **KRaft** mode (Kafka Raft Metadata Mode), which replaces the legacy Zookeeper coordinator, simplifying local setups.

Create a `docker-compose.yml` file:

```yaml
version: '3.8'

services:
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: local-kafka
    ports:
      - "9092:9092"
    environment:
      # Kafka broker identification
      KAFKA_NODE_ID: 1
      # KRaft mode configuration (Process roles)
      KAFKA_PROCESS_ROLES: 'broker,controller'
      # Controller listener configuration
      KAFKA_CONTROLLER_LISTENERS: 'CONTROLLER://:9093'
      # Listeners definition for internal and external traffic
      KAFKA_LISTENERS: 'PLAINTEXT://0.0.0.0:29092,CONTROLLER://0.0.0.0:9093,EXTERNAL://0.0.0.0:9092'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka:29092,EXTERNAL://localhost:9092'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,EXTERNAL:PLAINTEXT'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@localhost:9093'
      KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'
      # Required for KRaft cluster setup
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
      # Offsets retention settings
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_LOG_DIRS: '/tmp/kraft-combined-logs'
```

Start the broker via terminal:
```bash
docker compose up -d
```

---

## 5. TypeScript Client Implementation (`kafkajs`)

This step-by-step implementation demonstrates how to build a type-safe Producer and Consumer in Node.js.

### Step 1: Install Dependencies
```bash
npm install kafkajs
npm install -D typescript @types/node ts-node
```

### Step 2: Initialize Kafka Client (`kafkaClient.ts`)
Create a file to manage the base connection configuration:

```typescript
import { Kafka, logLogLevel } from 'kafkajs';

export const kafka = new Kafka({
  clientId: 'my-nodejs-app',
  brokers: ['localhost:9092'], // Matches the EXTERNAL listener port in docker-compose
  connectionTimeout: 3000,
  logLevel: logLogLevel.ERROR, // Keep logs clean, only show errors
});
```

### Step 3: Implement the Producer (`producer.ts`)
The Producer writes event records into a Kafka topic.

```typescript
import { kafka } from './kafkaClient';

const producer = kafka.producer({
  allowAutoTopicCreation: true, // Auto-creates topic if it doesn't exist
});

async function runProducer() {
  try {
    // 1. Connect to the Kafka Broker
    await producer.connect();
    console.log('Producer connected successfully!');

    // 2. Define message details
    const topic = 'user-signup-events';
    const messagePayload = {
      userId: 'user_10293',
      email: 'rahul@example.com',
      timestamp: new Date().toISOString(),
    };

    // 3. Send Message
    // Setting a 'key' is critical: Messages with the same key always go to the same partition
    await producer.send({
      topic,
      messages: [
        {
          key: messagePayload.userId,
          value: JSON.stringify(messagePayload),
        },
      ],
    });

    console.log(`Message sent to topic [${topic}]:`, messagePayload);
  } catch (error) {
    console.error('Error in Kafka Producer:', error);
  } finally {
    // 4. Gracefully disconnect from broker
    await producer.disconnect();
  }
}

runProducer();
```

### Step 4: Implement the Consumer (`consumer.ts`)
The Consumer reads event records from a topic as part of a Consumer Group.

```typescript
import { kafka } from './kafkaClient';

// Consumers must belong to a Consumer Group (group-id)
const consumer = kafka.consumer({
  groupId: 'notification-service-group',
});

async function runConsumer() {
  try {
    // 1. Connect to the Kafka Broker
    await consumer.connect();
    console.log('Consumer connected successfully!');

    // 2. Subscribe to the topic
    // fromBeginning: true reads older messages if the group offset hasn't been established yet
    await consumer.subscribe({
      topic: 'user-signup-events',
      fromBeginning: true,
    });

    // 3. Start reading messages
    await consumer.run({
      // eachMessage is executed automatically for every record consumed
      eachMessage: async ({ topic, partition, message }) => {
        const key = message.key?.toString();
        const value = message.value?.toString();
        
        console.log({
          info: 'New Message Received',
          topic,
          partition,
          offset: message.offset,
          key,
          value: value ? JSON.parse(value) : null,
        });

        // Add your application logic here (e.g. send confirmation email)
      },
    });
  } catch (error) {
    console.error('Error in Kafka Consumer:', error);
  }
}

// Graceful shutdown handling
const errorTypes = ['unhandledRejection', 'uncaughtException'];
const signalTypes = ['SIGINT', 'SIGTERM', 'SIGQUIT'];

errorTypes.forEach((type) => {
  process.on(type, async (err) => {
    try {
      console.log(`process.on ${type}`);
      console.error(err);
      await consumer.disconnect();
      process.exit(0);
    } catch (_) {
      process.exit(1);
    }
  });
});

signalTypes.forEach((type) => {
  process.once(type, async () => {
    try {
      console.log(`Disconnecting consumer due to ${type}...`);
      await consumer.disconnect();
      process.exit(0);
    } finally {
      process.exit(0);
    }
  });
});

runConsumer();
```

---

## 6. Message Delivery Semantics

Kafka supports three distinct modes of message consumption and processing confirmations:

### A. At-most-once (Data loss possible)
*   **How it works:** Consumer receives messages, **commits the offsets immediately**, and then processes the message.
*   **Failure Scenario:** If the consumer crashes midway while processing, the message is lost because Kafka thinks it was successfully processed.
*   **Use case:** Non-critical data (metrics logging, live video statistics).

### B. At-least-once (Duplicate messages possible - DEFAULT)
*   **How it works:** Consumer receives messages, **processes them successfully**, and then commits the offsets.
*   **Failure Scenario:** If the consumer crashes *after* processing but *before* committing the offset, it will re-receive the message upon reboot.
*   **Requirement:** Your consumer code **must be idempotent** (processing the same message twice does not cause errors or duplicate DB records).

### C. Exactly-once (Transactional)
*   **How it works:** Involves writing to multiple topics using Kafka transactions where offset commits and message writes succeed or fail together as a single atomic transaction unit.

---

## 7. Production Best Practices & Gotchas

### A. Partition Key Strategy
*   Never publish messages without keys if ordering matters.
*   Kafka uses a hashing algorithm on the message key to determine the partition. By using a key (e.g., `userId` or `orderId`), you guarantee that all updates for that specific user or order are processed **in sequential order** by the same partition.

### B. Consumer Rebalancing Storms
*   **Symptom:** Consumers frequently disconnect, stop consuming, and trigger group rebalances.
*   **Cause:** Your `eachMessage` function performs a heavy task (e.g., a slow database query or HTTP API call) that takes longer than the configured `maxPollIntervalMs` (default: 5 minutes). Kafka assumes the consumer has crashed and initiates group rebalancing.
*   **Fix:**
    *   Optimize the processing code.
    *   Or, increase the `maxPollIntervalMs` configuration.
    *   Or, use batch processing (`eachBatch`) instead of `eachMessage` and process items concurrently using async worker pools.

### C. Dead Letter Queues (DLQ)
*   If a message is corrupted or fails processing (e.g., parsing error), do not block the partition by retrying indefinitely.
*   Catch the error, send the failing message payload to a separate topic (e.g., `user-signup-events-dlq`) for manual audit, and commit the offset to continue processing subsequent messages.

---

## 8. Kafka Production Checklist

- [ ] Select appropriate partition count per topic based on target throughput goals (more partitions = higher parallelism).
- [ ] Set `acks: 'all'` in production producers to guarantee zero message loss (wait for all replica brokers to write message to disk).
- [ ] Define logical partition keys on messages to ensure order consistency for related events.
- [ ] Ensure all consumers inside a group contain idempotent business logic to handle at-least-once delivery duplicates.
- [ ] Set up a Dead Letter Queue (DLQ) topic to intercept and isolate corrupted message payloads.
- [ ] Adjust `maxPollIntervalMs` to prevent unnecessary consumer rebalances on long-running tasks.
- [ ] Monitor broker metrics including Under-Replicated Partitions, Active Controller Count, and Consumer Lag.
- [ ] Implement graceful shutdown handling to disconnect consumers and producers safely when stopping application containers.
