# Kafka Order Processing System

A practical implementation of Apache Kafka for real-time order processing using Python and Confluent Kafka. This project demonstrates producer-consumer patterns with Kafka running in KRaft mode (without Zookeeper).

## Table of Contents
- [Overview](#overview)
- [What is Apache Kafka?](#what-is-apache-kafka)
- [Key Concepts](#key-concepts)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation & Setup](#installation--setup)
- [Running the Application](#running-the-application)
- [Project Structure](#project-structure)
- [How It Works](#how-it-works)
- [Configuration Details](#configuration-details)
- [Common Use Cases](#common-use-cases)
- [Troubleshooting](#troubleshooting)

## Overview

This project implements a simple order processing system using Apache Kafka to demonstrate:
- **Producer**: Sends order messages to a Kafka topic
- **Consumer**: Reads and processes order messages from the topic
- **Kafka Broker**: Runs in KRaft mode using Docker for simplified setup

## What is Apache Kafka?

Apache Kafka is a distributed event streaming platform designed for:
- **High-throughput**: Handles millions of messages per second
- **Scalability**: Horizontally scalable across multiple servers
- **Fault-tolerance**: Data replication ensures no data loss
- **Real-time processing**: Process data streams as they arrive
- **Durability**: Messages are persisted to disk

### Why Use Kafka?

- **Decoupling**: Separates data producers from consumers
- **Buffering**: Handles traffic spikes by storing messages
- **Asynchronous Communication**: Non-blocking message delivery
- **Multiple Consumers**: Same data can be consumed by multiple applications
- **Data Integration**: Connect different systems and services

## Key Concepts

### Topics
- Logical channels where messages are published
- Similar to tables in a database or folders in a filesystem
- In this project: `orders` topic stores all order messages

### Producers
- Applications that publish/send messages to Kafka topics
- In this project: `producer.py` sends order data to the `orders` topic
- Supports callbacks for delivery confirmation

### Consumers
- Applications that subscribe to topics and process messages
- In this project: `consumer.py` reads orders from the `orders` topic
- Organized into consumer groups for load balancing

### Partitions
- Topics are split into partitions for parallelism
- Each partition is an ordered, immutable sequence of messages
- Enables horizontal scaling and parallel processing
- Messages within a partition maintain order

### Offsets
- Unique identifier for each message within a partition
- Consumers track their position using offsets
- Allows consumers to resume from where they left off

### Consumer Groups
- Multiple consumers working together to process a topic
- Each partition is consumed by only one consumer in the group
- Provides load balancing and fault tolerance
- In this project: `order-tracker` group

### KRaft Mode (Kafka Raft)
- Modern Kafka deployment without Zookeeper dependency
- Simplified architecture and operations
- Built-in consensus mechanism for metadata management
- Faster controller failover

## Architecture

### High-Level Overview

```
+---------------+         +------------------+         +---------------+
|               | produce |                  | consume |               |
|   Producer    +-------->|  Kafka Broker    +-------->|   Consumer    |
| (producer.py) |         |   (Docker)       |         | (consumer.py) |
|               |         |                  |         |               |
+---------------+         |  Topic: orders   |         +---------------+
                          |  Port: 9092      |
                          +------------------+
```

### Complete Workflow Sequence Diagram

![Kafka Sequence Diagram](https://raw.githubusercontent.com/AnupCloud/KAFKA_PYTHON/main/kafka_sequence_diagram.png)

*The diagram above illustrates the complete message flow between Producer, Kafka Broker, and Consumer components, showing the step-by-step interaction including message production, storage, acknowledgment, and consumption.*

### Detailed Sequence Diagram (Text Version)

```
Producer              Kafka Broker           Consumer
(producer.py)         (localhost:9092)       (consumer.py)
    |                      |                      |
    |--[1] Create Order--->|                      |
    |   (JSON + UUID)      |                      |
    |                      |                      |
    |--[2] produce()------>|                      |
    |   topic="orders"     |                      |
    |                      |                      |
    |                      |--[3] Store Message-->|
    |                      |   (partition+offset) |
    |                      |                      |
    |<-[4] callback--------|                      |
    |   delivery_report()  |                      |
    |   (partition,offset) |                      |
    |                      |                      |
    |--[5] flush()-------->|                      |
    |   (ensure delivery)  |                      |
    |                      |                      |
    |                      |<-----[6] poll()------|
    |                      |      (timeout: 1.0s) |
    |                      |                      |
    |                      |------[7] Message---->|
    |                      |                      |
    |                      |                      |--[8] Decode JSON
    |                      |                      |    & Process Order
    |                      |                      |
    |                      |<-----[9] poll()------|
    |                      |      (continuous)    |
```

### Data Flow Explained

#### Producer Flow (producer.py)

1. **Create Order Message**: Generates order with UUID, user, item, quantity ([producer.py:19-24](producer.py#L19-L24))
2. **Serialize to JSON**: Converts dictionary to JSON and encodes to UTF-8 bytes ([producer.py:26](producer.py#L26))
3. **Send to Topic**: Calls `producer.produce()` to send message to "orders" topic ([producer.py:28-32](producer.py#L28-L32))
4. **Delivery Callback**: `delivery_report()` confirms successful delivery with partition and offset ([producer.py:12-17](producer.py#L12-L17))
5. **Flush**: Ensures all buffered messages are sent before exit ([producer.py:34](producer.py#L34))

#### Kafka Broker Flow

1. **Receive Message**: Accepts message from producer on port 9092
2. **Store in Partition**: Writes message to partition with unique offset
3. **Persist to Disk**: Saves to `/tmp/kraft-combined-logs` (durable storage)
4. **Acknowledge**: Sends delivery confirmation back to producer
5. **Serve to Consumers**: Makes message available for consumer polling

#### Consumer Flow (consumer.py)

1. **Subscribe**: Joins consumer group "order-tracker" and subscribes to "orders" topic ([consumer.py:13](consumer.py#L13))
2. **Poll Continuously**: Calls `consumer.poll(1.0)` in infinite loop ([consumer.py:19](consumer.py#L19))
3. **Receive Message**: Gets message from Kafka when available
4. **Error Check**: Validates message has no errors ([consumer.py:22-24](consumer.py#L22-L24))
5. **Deserialize**: Decodes UTF-8 bytes and parses JSON ([consumer.py:26-27](consumer.py#L26-L27))
6. **Process**: Displays order details ([consumer.py:28](consumer.py#L28))
7. **Auto-Commit Offset**: Automatically tracks consumer position
8. **Repeat**: Continues polling for new messages

## Prerequisites

- **Docker & Docker Compose**: For running Kafka
- **Python 3.8+**: For producer and consumer scripts
- **pip or uv**: Python package manager

## Installation & Setup

### 1. Clone and Navigate to Project

```bash
cd kafka_project
```

### 2. Install Python Dependencies

Using pip:
```bash
pip install -r requirements.txt
```

Using uv (recommended for faster installation):
```bash
uv pip install -r requirements.txt
```

### 3. Start Kafka Broker

```bash
docker-compose up -d
```

This starts:
- Kafka broker on `localhost:9092`
- KRaft controller on port `9093`
- Persistent volume for data storage

### 4. Verify Kafka is Running

```bash
docker ps
```

You should see the `kafka` container running.

Check logs:
```bash
docker-compose logs kafka
```

## Running the Application

### Step 1: Start the Consumer (First Terminal)

```bash
python consumer.py
```

Output:
```
🟢 Consumer is running and subscribed to orders topic
```

The consumer will continuously poll for messages.

### Step 2: Run the Producer (Second Terminal)

```bash
python producer.py
```

Output:
```
✅ Delivered {"order_id": "...", "user": "david", "item": "chicken burger", "quantity": 5}
✅ Delivered to orders : partition 0 : at offset 0
```

### Step 3: Observe the Consumer

The consumer terminal will show:
```
📦 Received order: 5 x chicken burger from david
```

## Project Structure

```
kafka_project/
├── docker-compose.yaml    # Kafka broker configuration
├── producer.py            # Message producer script
├── consumer.py            # Message consumer script
├── requirements.txt       # Python dependencies
├── pyproject.toml         # Project metadata (uv)
├── uv.lock                # Dependency lock file
└── README.md              # This file
```

## How It Works

### Producer Flow (`producer.py`)

1. **Configuration**: Connects to Kafka at `localhost:9092`
2. **Create Message**: Generates order with UUID, user, item, and quantity
3. **Serialize**: Converts order dictionary to JSON and encodes to bytes
4. **Send**: Publishes message to `orders` topic
5. **Callback**: `delivery_report()` confirms successful delivery
6. **Flush**: Ensures all messages are sent before exit

Key code:
```python
producer.produce(
    topic="orders",
    value=value,
    callback=delivery_report
)
producer.flush()
```

### Consumer Flow (`consumer.py`)

1. **Configuration**:
   - Connects to Kafka at `localhost:9092`
   - Joins consumer group `order-tracker`
   - Sets `auto.offset.reset` to `earliest` (reads from beginning)

2. **Subscribe**: Listens to `orders` topic
3. **Poll**: Continuously checks for new messages (1 second timeout)
4. **Process**: Decodes message and parses JSON
5. **Handle**: Prints order details
6. **Error Handling**: Graceful shutdown on KeyboardInterrupt

Key code:
```python
consumer.subscribe(["orders"])
while True:
    msg = consumer.poll(1.0)
    # Process message
```

### Docker Compose Configuration

- **Image**: `confluentinc/cp-kafka:7.8.3`
- **Mode**: KRaft (no Zookeeper required)
- **Ports**: 9092 (client), 9093 (controller)
- **Replication Factor**: 1 (single broker setup)
- **Volumes**: Persistent storage for Kafka data

## Configuration Details

### Producer Configuration

```python
producer_config = {
    "bootstrap.servers": "localhost:9092"
}
```

Additional options:
- `acks`: Message durability (0, 1, all)
- `compression.type`: Message compression (gzip, snappy, lz4)
- `batch.size`: Batching for throughput
- `linger.ms`: Wait time before sending batch

### Consumer Configuration

```python
consumer_config = {
    "bootstrap.servers": "localhost:9092",
    "group.id": "order-tracker",
    "auto.offset.reset": "earliest"
}
```

Configuration explained:
- `bootstrap.servers`: Kafka broker address
- `group.id`: Consumer group identifier
- `auto.offset.reset`: What to do when no offset exists
  - `earliest`: Start from beginning
  - `latest`: Start from newest messages

Additional options:
- `enable.auto.commit`: Automatic offset commit
- `auto.commit.interval.ms`: Commit frequency
- `max.poll.records`: Max messages per poll

## Common Use Cases

### 1. Event-Driven Architecture
- Microservices communication
- Real-time notifications
- Audit logs

### 2. Stream Processing
- Real-time analytics
- Fraud detection
- Recommendation engines

### 3. Log Aggregation
- Centralized logging
- Application monitoring
- System metrics

### 4. Message Queue
- Task distribution
- Background job processing
- Order processing (like this project)

### 5. Data Integration
- ETL pipelines
- Data lake ingestion
- System synchronization

## Troubleshooting

### Kafka Won't Start

Check if port 9092 is already in use:
```bash
lsof -i :9092
```

Stop existing containers:
```bash
docker-compose down
docker-compose up -d
```

### Consumer Not Receiving Messages

1. Verify Kafka is running:
   ```bash
   docker ps
   ```

2. Check topic exists:
   ```bash
   docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list
   ```

3. Verify messages in topic:
   ```bash
   docker exec -it kafka kafka-console-consumer \
     --bootstrap-server localhost:9092 \
     --topic orders \
     --from-beginning
   ```

### Connection Refused Error

Ensure Kafka has fully started (takes 10-15 seconds):
```bash
docker-compose logs -f kafka
```

Wait for: `[KafkaServer id=1] started`

### Producer Delivery Failed

Check Kafka broker logs:
```bash
docker-compose logs kafka | grep ERROR
```

Verify network connectivity:
```bash
telnet localhost 9092
```

### Reset Consumer Offset

To reprocess all messages from the beginning:
```bash
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group order-tracker \
  --reset-offsets --to-earliest \
  --topic orders --execute
```

## Advanced Features to Explore

1. **Multiple Partitions**: Scale horizontally
2. **Message Keys**: Ensure ordering for specific keys
3. **Avro/Protobuf**: Schema-based serialization
4. **Kafka Streams**: Real-time stream processing
5. **Monitoring**: Kafka Manager, Confluent Control Center
6. **Security**: SSL/TLS, SASL authentication
7. **Replication**: Multi-broker setup for fault tolerance

## Cleanup

Stop and remove Kafka:
```bash
docker-compose down
```

Remove volumes (deletes all data):
```bash
docker-compose down -v
```

## Resources

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Confluent Kafka Python](https://docs.confluent.io/kafka-clients/python/current/overview.html)
- [Kafka in KRaft Mode](https://kafka.apache.org/documentation/#kraft)
- [Best Practices](https://kafka.apache.org/documentation/#bestpractices)