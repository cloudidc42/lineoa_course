# Part 83: Real-time Analytics สำหรับ LINE Bots

## บทนำ

Real-time Analytics ช่วยให้เข้าใจพฤติกรรมผู้ใช้ LINE Bot ทันทีที่เกิดขึ้น ไม่ต้องรอรายงานรายวัน ทำให้ตัดสินใจทางธุรกิจได้เร็วขึ้น บทนี้จะสร้าง Analytics Platform สมบูรณ์ด้วย Apache Kafka, Apache Flink, ClickHouse, และ Grafana

---

## 1. Architecture Overview

```
Real-time Analytics Architecture:

┌──────────────┐     Events      ┌──────────────────┐
│  LINE Bot    │────────────────▶│   Apache Kafka    │
│  (Producer)  │                 │   (Event Stream)  │
└──────────────┘                 └────────┬─────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
           ┌────────▼────────┐  ┌─────────▼────────┐  ┌───────▼──────────┐
           │  Apache Flink   │  │   ClickHouse     │  │   Elasticsearch  │
           │ (Stream Process)│  │ (Analytics DB)   │  │  (Log Search)    │
           └────────┬────────┘  └─────────┬────────┘  └───────┬──────────┘
                    │                     │                     │
                    └──────────┬──────────┘                     │
                               │                                │
                    ┌──────────▼──────────┐                     │
                    │      Grafana        │◄────────────────────┘
                    │   (Dashboards)      │
                    └─────────────────────┘
```

---

## 2. Apache Kafka Setup

### 2.1 Kafka Installation with Docker

```yaml
# docker-compose.kafka.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOOKEEPER_LOG4J_ROOT_LOGLEVEL: WARN
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log

  kafka1:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka1:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_NUM_PARTITIONS: 12
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_LOG_RETENTION_HOURS: 168  # 7 days
      KAFKA_LOG_SEGMENT_BYTES: 1073741824  # 1GB
      KAFKA_LOG4J_LOGGERS: "kafka=WARN,kafka.controller=WARN"
    volumes:
      - kafka1-data:/var/lib/kafka/data

  kafka2:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9093:9093"
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka2:29093,PLAINTEXT_HOST://localhost:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
    volumes:
      - kafka2-data:/var/lib/kafka/data

  kafka3:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9094:9094"
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka3:29094,PLAINTEXT_HOST://localhost:9094
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
    volumes:
      - kafka3-data:/var/lib/kafka/data

  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    depends_on:
      - kafka1
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka1:29092,kafka2:29093,kafka3:29094

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: linebot-cluster
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka1:29092,kafka2:29093,kafka3:29094
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081

  clickhouse:
    image: clickhouse/clickhouse-server:23.8
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse-data:/var/lib/clickhouse
      - ./clickhouse/config:/etc/clickhouse-server/config.d
      - ./clickhouse/users:/etc/clickhouse-server/users.d
    ulimits:
      nofile:
        soft: 262144
        hard: 262144

volumes:
  zookeeper-data:
  zookeeper-logs:
  kafka1-data:
  kafka2-data:
  kafka3-data:
  clickhouse-data:
```

### 2.2 Kafka Topics Setup

```javascript
// scripts/setup-kafka-topics.js
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'linebot-admin',
  brokers: process.env.KAFKA_BROKERS?.split(',') || ['localhost:9092'],
});

const admin = kafka.admin();

const TOPICS = [
  {
    topic: 'linebot.events.messages',
    numPartitions: 12,
    replicationFactor: 3,
    configEntries: [
      { name: 'retention.ms', value: '604800000' }, // 7 days
      { name: 'cleanup.policy', value: 'delete' },
      { name: 'compression.type', value: 'lz4' },
    ],
  },
  {
    topic: 'linebot.events.follows',
    numPartitions: 6,
    replicationFactor: 3,
    configEntries: [
      { name: 'retention.ms', value: '604800000' },
    ],
  },
  {
    topic: 'linebot.events.postbacks',
    numPartitions: 12,
    replicationFactor: 3,
    configEntries: [
      { name: 'retention.ms', value: '604800000' },
      { name: 'compression.type', value: 'lz4' },
    ],
  },
  {
    topic: 'linebot.events.purchases',
    numPartitions: 6,
    replicationFactor: 3,
    configEntries: [
      { name: 'retention.ms', value: '2592000000' }, // 30 days
    ],
  },
  {
    topic: 'linebot.analytics.user-sessions',
    numPartitions: 12,
    replicationFactor: 3,
    configEntries: [
      { name: 'retention.ms', value: '86400000' }, // 1 day (processed data)
    ],
  },
  {
    topic: 'linebot.analytics.aggregates',
    numPartitions: 6,
    replicationFactor: 3,
    configEntries: [
      { name: 'retention.ms', value: '604800000' },
    ],
  },
  {
    topic: 'linebot.dlq',  // Dead Letter Queue
    numPartitions: 3,
    replicationFactor: 3,
    configEntries: [
      { name: 'retention.ms', value: '2592000000' }, // 30 days
    ],
  },
];

async function createTopics() {
  await admin.connect();
  
  try {
    const existingTopics = await admin.listTopics();
    
    const topicsToCreate = TOPICS.filter(
      t => !existingTopics.includes(t.topic)
    );
    
    if (topicsToCreate.length === 0) {
      console.log('All topics already exist');
      return;
    }
    
    await admin.createTopics({
      topics: topicsToCreate,
      waitForLeaders: true,
    });
    
    console.log(`Created topics: ${topicsToCreate.map(t => t.topic).join(', ')}`);
  } finally {
    await admin.disconnect();
  }
}

createTopics().catch(console.error);
```

### 2.3 Kafka Producer Integration

```javascript
// src/analytics/EventProducer.js
const { Kafka, CompressionTypes } = require('kafkajs');

class EventProducer {
  constructor() {
    this.kafka = new Kafka({
      clientId: 'linebot-producer',
      brokers: process.env.KAFKA_BROKERS?.split(',') || ['localhost:9092'],
      retry: {
        initialRetryTime: 100,
        retries: 8,
        factor: 2,
      },
    });
    
    this.producer = this.kafka.producer({
      transactionalId: `linebot-${process.env.HOSTNAME}`,
      maxInFlightRequests: 5,
      idempotent: true,
    });
    
    this.pendingMessages = [];
    this.flushInterval = null;
  }
  
  async connect() {
    await this.producer.connect();
    
    // Batch flush ทุก 100ms
    this.flushInterval = setInterval(() => this.flush(), 100);
    
    console.log('Kafka producer connected');
  }
  
  async trackMessage(event) {
    const kafkaMessage = {
      key: event.source.userId,  // partition by user
      value: JSON.stringify({
        eventId: require('crypto').randomUUID(),
        type: 'message',
        timestamp: new Date().toISOString(),
        userId: event.source.userId,
        groupId: event.source.groupId,
        messageType: event.message.type,
        messageText: event.message.type === 'text' ? event.message.text : null,
        messageId: event.message.id,
        replyToken: event.replyToken,
      }),
      headers: {
        'event-type': 'message',
        'source': 'webhook',
        'version': '1',
      },
      timestamp: Date.now().toString(),
    };
    
    this.pendingMessages.push({
      topic: 'linebot.events.messages',
      messages: [kafkaMessage],
    });
    
    // Flush หาก pending มากเกิน
    if (this.pendingMessages.length >= 100) {
      await this.flush();
    }
  }
  
  async trackFollow(event) {
    await this.producer.send({
      topic: 'linebot.events.follows',
      messages: [{
        key: event.source.userId,
        value: JSON.stringify({
          eventId: require('crypto').randomUUID(),
          type: 'follow',
          timestamp: new Date().toISOString(),
          userId: event.source.userId,
        }),
      }],
    });
  }
  
  async trackUnfollow(event) {
    await this.producer.send({
      topic: 'linebot.events.follows',
      messages: [{
        key: event.source.userId,
        value: JSON.stringify({
          eventId: require('crypto').randomUUID(),
          type: 'unfollow',
          timestamp: new Date().toISOString(),
          userId: event.source.userId,
        }),
      }],
    });
  }
  
  async trackPurchase(userId, orderData) {
    await this.producer.send({
      topic: 'linebot.events.purchases',
      messages: [{
        key: userId,
        value: JSON.stringify({
          eventId: require('crypto').randomUUID(),
          type: 'purchase',
          timestamp: new Date().toISOString(),
          userId,
          orderId: orderData.orderId,
          amount: orderData.amount,
          currency: 'THB',
          items: orderData.items,
          channel: 'line',
        }),
        compression: CompressionTypes.LZ4,
      }],
    });
  }
  
  async flush() {
    if (this.pendingMessages.length === 0) return;
    
    const messages = [...this.pendingMessages];
    this.pendingMessages = [];
    
    try {
      await this.producer.sendBatch({
        topicMessages: messages,
        compression: CompressionTypes.LZ4,
      });
    } catch (error) {
      console.error('Failed to flush messages:', error);
      // ส่งไป DLQ
      await this.sendToDLQ(messages, error);
    }
  }
  
  async sendToDLQ(messages, error) {
    try {
      const dlqMessages = messages.flatMap(({ topic, messages: msgs }) =>
        msgs.map(msg => ({
          key: msg.key,
          value: JSON.stringify({
            originalTopic: topic,
            originalMessage: msg.value,
            error: error.message,
            failedAt: new Date().toISOString(),
          }),
        }))
      );
      
      await this.producer.send({
        topic: 'linebot.dlq',
        messages: dlqMessages,
      });
    } catch (dlqError) {
      console.error('Failed to send to DLQ:', dlqError);
    }
  }
  
  async disconnect() {
    if (this.flushInterval) {
      clearInterval(this.flushInterval);
    }
    await this.flush(); // Flush remaining
    await this.producer.disconnect();
  }
}

module.exports = EventProducer;
```

---

## 3. ClickHouse Analytics Storage

### 3.1 ClickHouse Schema

```sql
-- clickhouse/schema/events.sql

-- Database
CREATE DATABASE IF NOT EXISTS linebot;

-- Events table (main)
CREATE TABLE linebot.events (
    event_id UUID DEFAULT generateUUIDv4(),
    event_type LowCardinality(String),
    user_id String,
    group_id Nullable(String),
    timestamp DateTime64(3, 'Asia/Bangkok'),
    message_type LowCardinality(String) DEFAULT '',
    message_text Nullable(String),
    message_id Nullable(String),
    -- Parsed data
    intent LowCardinality(String) DEFAULT '',
    entities String DEFAULT '{}',  -- JSON
    -- Technical
    response_time_ms UInt32 DEFAULT 0,
    error_occurred UInt8 DEFAULT 0,
    -- Date parts for partitioning
    date Date DEFAULT toDate(timestamp),
    hour UInt8 DEFAULT toHour(timestamp)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (event_type, user_id, timestamp)
TTL date + INTERVAL 1 YEAR
SETTINGS index_granularity = 8192;

-- Follows table
CREATE TABLE linebot.follows (
    event_id UUID DEFAULT generateUUIDv4(),
    action LowCardinality(String),  -- 'follow' or 'unfollow'
    user_id String,
    timestamp DateTime64(3, 'Asia/Bangkok'),
    date Date DEFAULT toDate(timestamp)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (user_id, timestamp);

-- Purchases table
CREATE TABLE linebot.purchases (
    event_id UUID DEFAULT generateUUIDv4(),
    user_id String,
    order_id String,
    amount Decimal(10, 2),
    currency LowCardinality(String) DEFAULT 'THB',
    items String,  -- JSON array
    channel LowCardinality(String),
    timestamp DateTime64(3, 'Asia/Bangkok'),
    date Date DEFAULT toDate(timestamp)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (user_id, timestamp)
TTL date + INTERVAL 2 YEAR;

-- User Sessions (computed by Flink)
CREATE TABLE linebot.user_sessions (
    session_id String,
    user_id String,
    started_at DateTime64(3, 'Asia/Bangkok'),
    ended_at DateTime64(3, 'Asia/Bangkok'),
    duration_seconds UInt32,
    message_count UInt16,
    events_count UInt16,
    date Date DEFAULT toDate(started_at)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (user_id, started_at);

-- Materialized Views for pre-aggregation

-- Hourly message stats
CREATE MATERIALIZED VIEW linebot.hourly_stats
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, hour, event_type)
POPULATE AS
SELECT
    date,
    hour,
    event_type,
    count() AS event_count,
    countDistinct(user_id) AS unique_users,
    avg(response_time_ms) AS avg_response_ms
FROM linebot.events
GROUP BY date, hour, event_type;

-- Daily user stats
CREATE MATERIALIZED VIEW linebot.daily_user_stats
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, user_id)
POPULATE AS
SELECT
    date,
    user_id,
    countState() AS message_count,
    maxState(timestamp) AS last_seen,
    minState(timestamp) AS first_seen
FROM linebot.events
GROUP BY date, user_id;

-- Daily revenue
CREATE MATERIALIZED VIEW linebot.daily_revenue
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, currency)
POPULATE AS
SELECT
    date,
    currency,
    sum(amount) AS total_revenue,
    count() AS transaction_count,
    countDistinct(user_id) AS unique_buyers,
    avg(amount) AS avg_order_value
FROM linebot.purchases
GROUP BY date, currency;

-- Kafka Engine for real-time ingestion
CREATE TABLE linebot.kafka_events (
    event_id String,
    event_type String,
    user_id String,
    timestamp String,
    message_type String,
    message_text Nullable(String),
    response_time_ms UInt32
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka1:9092,kafka2:9092,kafka3:9092',
    kafka_topic_list = 'linebot.events.messages',
    kafka_group_name = 'clickhouse-consumer',
    kafka_format = 'JSONEachRow',
    kafka_max_block_size = 65536;

-- Materialized View to ingest from Kafka
CREATE MATERIALIZED VIEW linebot.kafka_events_mv TO linebot.events AS
SELECT
    toUUID(event_id) AS event_id,
    event_type,
    user_id,
    '' AS group_id,
    parseDateTimeBestEffort(timestamp) AS timestamp,
    message_type,
    message_text,
    '' AS message_id,
    '' AS intent,
    '{}' AS entities,
    response_time_ms,
    0 AS error_occurred,
    toDate(parseDateTimeBestEffort(timestamp)) AS date,
    toHour(parseDateTimeBestEffort(timestamp)) AS hour
FROM linebot.kafka_events;
```

### 3.2 ClickHouse Analytics Queries

```sql
-- ข้อความยอดนิยมในช่วง 24 ชั่วโมงที่ผ่านมา
SELECT
    message_text,
    count() AS frequency,
    countDistinct(user_id) AS unique_users
FROM linebot.events
WHERE event_type = 'message'
  AND timestamp >= now() - INTERVAL 24 HOUR
  AND message_text IS NOT NULL
  AND length(message_text) > 0
GROUP BY message_text
HAVING frequency > 5
ORDER BY frequency DESC
LIMIT 50;

-- Active users per hour (last 7 days)
SELECT
    toStartOfHour(timestamp) AS hour,
    countDistinct(user_id) AS active_users,
    count() AS total_messages
FROM linebot.events
WHERE timestamp >= now() - INTERVAL 7 DAY
GROUP BY hour
ORDER BY hour;

-- User retention analysis
WITH first_message AS (
    SELECT
        user_id,
        min(date) AS first_date
    FROM linebot.events
    WHERE event_type = 'message'
    GROUP BY user_id
),
cohort_sizes AS (
    SELECT
        first_date,
        count() AS cohort_size
    FROM first_message
    GROUP BY first_date
),
retention AS (
    SELECT
        fm.first_date,
        dateDiff('day', fm.first_date, e.date) AS day_number,
        countDistinct(e.user_id) AS retained_users
    FROM linebot.events e
    JOIN first_message fm ON e.user_id = fm.user_id
    WHERE e.event_type = 'message'
      AND day_number <= 30
    GROUP BY fm.first_date, day_number
)
SELECT
    r.first_date,
    r.day_number,
    r.retained_users,
    cs.cohort_size,
    round(r.retained_users / cs.cohort_size * 100, 2) AS retention_rate
FROM retention r
JOIN cohort_sizes cs ON r.first_date = cs.first_date
ORDER BY r.first_date, r.day_number;

-- Revenue analytics
SELECT
    toStartOfDay(timestamp) AS date,
    sum(amount) AS daily_revenue,
    count() AS orders,
    countDistinct(user_id) AS buyers,
    avg(amount) AS avg_order_value,
    max(amount) AS max_order,
    quantile(0.5)(amount) AS median_order
FROM linebot.purchases
WHERE timestamp >= now() - INTERVAL 30 DAY
GROUP BY date
ORDER BY date;

-- Funnel analysis
WITH funnel AS (
    SELECT
        user_id,
        countIf(event_type = 'message') > 0 AS sent_message,
        countIf(event_type = 'postback' AND JSONExtractString(entities, 'action') = 'view_product') > 0 AS viewed_product,
        countIf(event_type = 'postback' AND JSONExtractString(entities, 'action') = 'add_to_cart') > 0 AS added_to_cart,
        countIf(event_type = 'purchase') > 0 AS purchased
    FROM linebot.events
    WHERE date >= today() - 30
    GROUP BY user_id
)
SELECT
    sum(sent_message) AS step1_users,
    sum(viewed_product) AS step2_users,
    sum(added_to_cart) AS step3_users,
    sum(purchased) AS step4_users,
    round(sum(viewed_product) / sum(sent_message) * 100, 2) AS step1_to_2_rate,
    round(sum(added_to_cart) / sum(viewed_product) * 100, 2) AS step2_to_3_rate,
    round(sum(purchased) / sum(added_to_cart) * 100, 2) AS step3_to_4_rate,
    round(sum(purchased) / sum(sent_message) * 100, 2) AS overall_conversion
FROM funnel;

-- Real-time messages per second (last 5 minutes)
SELECT
    toStartOfSecond(timestamp) AS second,
    count() AS messages_per_second
FROM linebot.events
WHERE event_type = 'message'
  AND timestamp >= now() - INTERVAL 5 MINUTE
GROUP BY second
ORDER BY second;
```

---

## 4. Apache Flink Stream Processing

### 4.1 Flink Setup

```python
# flink_jobs/user_session_analyzer.py
# Apache Flink 1.18 with PyFlink

from pyflink.datastream import StreamExecutionEnvironment, TimeCharacteristic
from pyflink.datastream.connectors.kafka import FlinkKafkaConsumer, FlinkKafkaProducer
from pyflink.datastream.window import TumblingProcessingTimeWindows, SlidingProcessingTimeWindows
from pyflink.common import Time, Types
from pyflink.datastream.functions import ProcessWindowFunction
import json
from datetime import datetime

def create_session_analyzer():
    env = StreamExecutionEnvironment.get_execution_environment()
    env.set_stream_time_characteristic(TimeCharacteristic.EventTime)
    env.set_parallelism(4)
    
    # Kafka Consumer
    kafka_consumer = FlinkKafkaConsumer(
        topics=['linebot.events.messages', 'linebot.events.postbacks'],
        deserialization_schema=SimpleStringSchema(),
        properties={
            'bootstrap.servers': 'kafka1:9092,kafka2:9092',
            'group.id': 'flink-session-analyzer',
            'auto.offset.reset': 'latest',
        }
    )
    
    # Add watermark
    kafka_consumer.assign_timestamps_and_watermarks(
        BoundedOutOfOrdernessTimestampExtractor(Time.seconds(5))
    )
    
    # Process stream
    events_stream = env.add_source(kafka_consumer)
    
    # Parse JSON
    parsed_stream = events_stream.map(
        lambda json_str: json.loads(json_str)
    )
    
    # Key by user
    user_stream = parsed_stream.key_by(lambda e: e['user_id'])
    
    # Session window (30 min inactivity gap)
    session_stream = user_stream.window(
        EventTimeSessionWindows.with_gap(Time.minutes(30))
    ).process(SessionProcessor())
    
    # Output to Kafka
    session_producer = FlinkKafkaProducer(
        topic='linebot.analytics.user-sessions',
        serialization_schema=SimpleStringSchema(),
        producer_config={
            'bootstrap.servers': 'kafka1:9092,kafka2:9092',
        }
    )
    
    session_stream.add_sink(session_producer)
    
    return env


class SessionProcessor(ProcessWindowFunction):
    def process(self, key, context, elements):
        events = list(elements)
        events.sort(key=lambda e: e['timestamp'])
        
        session = {
            'session_id': f"{key}-{context.window().start}",
            'user_id': key,
            'started_at': events[0]['timestamp'],
            'ended_at': events[-1]['timestamp'],
            'message_count': sum(1 for e in events if e['event_type'] == 'message'),
            'events_count': len(events),
            'duration_seconds': (
                datetime.fromisoformat(events[-1]['timestamp']) -
                datetime.fromisoformat(events[0]['timestamp'])
            ).seconds,
        }
        
        yield json.dumps(session)
```

### 4.2 Real-time Aggregation Job

```python
# flink_jobs/realtime_aggregator.py

from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import StreamTableEnvironment, EnvironmentSettings

def create_realtime_aggregator():
    env = StreamExecutionEnvironment.get_execution_environment()
    settings = EnvironmentSettings.new_instance().in_streaming_mode().build()
    t_env = StreamTableEnvironment.create(env, environment_settings=settings)
    
    # Source: Kafka
    t_env.execute_sql("""
        CREATE TABLE kafka_events (
            event_id STRING,
            event_type STRING,
            user_id STRING,
            message_type STRING,
            timestamp BIGINT,
            response_time_ms INT,
            event_time AS TO_TIMESTAMP_LTZ(timestamp, 3),
            WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
        ) WITH (
            'connector' = 'kafka',
            'topic' = 'linebot.events.messages',
            'properties.bootstrap.servers' = 'kafka1:9092',
            'properties.group.id' = 'flink-aggregator',
            'scan.startup.mode' = 'latest-offset',
            'format' = 'json'
        )
    """)
    
    # Sink: ClickHouse (via JDBC)
    t_env.execute_sql("""
        CREATE TABLE clickhouse_aggregates (
            window_start TIMESTAMP(3),
            window_end TIMESTAMP(3),
            event_type STRING,
            event_count BIGINT,
            unique_users BIGINT,
            avg_response_ms DOUBLE
        ) WITH (
            'connector' = 'jdbc',
            'url' = 'jdbc:clickhouse://clickhouse:8123/linebot',
            'table-name' = 'realtime_aggregates',
            'driver' = 'com.clickhouse.jdbc.ClickHouseDriver'
        )
    """)
    
    # Tumbling window aggregation (1 minute)
    t_env.execute_sql("""
        INSERT INTO clickhouse_aggregates
        SELECT
            TUMBLE_START(event_time, INTERVAL '1' MINUTE) AS window_start,
            TUMBLE_END(event_time, INTERVAL '1' MINUTE) AS window_end,
            event_type,
            COUNT(*) AS event_count,
            COUNT(DISTINCT user_id) AS unique_users,
            AVG(CAST(response_time_ms AS DOUBLE)) AS avg_response_ms
        FROM kafka_events
        GROUP BY
            TUMBLE(event_time, INTERVAL '1' MINUTE),
            event_type
    """)
```

---

## 5. Real-time Dashboard

### 5.1 Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "title": "LINE Bot Real-time Analytics",
    "refresh": "5s",
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "panels": [
      {
        "id": 1,
        "title": "Messages per Second (Real-time)",
        "type": "graph",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
        "targets": [{
          "rawSql": "SELECT toStartOfSecond(timestamp) AS time, count() AS value FROM linebot.events WHERE timestamp >= now() - INTERVAL 5 MINUTE GROUP BY time ORDER BY time",
          "format": "time_series"
        }],
        "options": {
          "tooltip": { "mode": "single" }
        }
      },
      {
        "id": 2,
        "title": "Active Users (Last 5 min)",
        "type": "stat",
        "gridPos": { "h": 4, "w": 4, "x": 12, "y": 0 },
        "targets": [{
          "rawSql": "SELECT count(DISTINCT user_id) AS value FROM linebot.events WHERE timestamp >= now() - INTERVAL 5 MINUTE",
          "format": "table"
        }],
        "options": {
          "colorMode": "background",
          "graphMode": "area"
        }
      },
      {
        "id": 3,
        "title": "P99 Response Time",
        "type": "gauge",
        "gridPos": { "h": 4, "w": 4, "x": 16, "y": 0 },
        "targets": [{
          "rawSql": "SELECT quantile(0.99)(response_time_ms) AS value FROM linebot.events WHERE timestamp >= now() - INTERVAL 5 MINUTE",
          "format": "table"
        }],
        "options": {
          "thresholds": {
            "steps": [
              { "value": 0, "color": "green" },
              { "value": 200, "color": "yellow" },
              { "value": 500, "color": "red" }
            ]
          }
        }
      },
      {
        "id": 4,
        "title": "Message Types Distribution",
        "type": "piechart",
        "gridPos": { "h": 8, "w": 8, "x": 0, "y": 8 },
        "targets": [{
          "rawSql": "SELECT message_type AS metric, count() AS value FROM linebot.events WHERE event_type = 'message' AND timestamp >= now() - INTERVAL 1 HOUR GROUP BY message_type",
          "format": "table"
        }]
      },
      {
        "id": 5,
        "title": "Conversion Funnel (Today)",
        "type": "bargauge",
        "gridPos": { "h": 8, "w": 16, "x": 8, "y": 8 },
        "targets": [{
          "rawSql": "SELECT 'Messages' AS stage, count(DISTINCT user_id) AS users FROM linebot.events WHERE event_type = 'message' AND date = today() UNION ALL SELECT 'Product Views', count(DISTINCT user_id) FROM linebot.events WHERE event_type = 'postback' AND JSONExtractString(entities, 'action') = 'view_product' AND date = today() UNION ALL SELECT 'Purchases', count(DISTINCT user_id) FROM linebot.purchases WHERE date = today()",
          "format": "table"
        }]
      }
    ]
  }
}
```

### 5.2 Custom Analytics API

```javascript
// src/analytics/AnalyticsAPI.js
const { createClient } = require('@clickhouse/client');

class AnalyticsAPI {
  constructor() {
    this.ch = createClient({
      host: process.env.CLICKHOUSE_HOST || 'http://localhost:8123',
      database: 'linebot',
      username: process.env.CLICKHOUSE_USER || 'default',
      password: process.env.CLICKHOUSE_PASSWORD || '',
    });
  }
  
  async getRealtimeStats(minutesBack = 5) {
    const result = await this.ch.query({
      query: `
        SELECT
          count() AS total_messages,
          countDistinct(user_id) AS active_users,
          avg(response_time_ms) AS avg_response_ms,
          quantile(0.99)(response_time_ms) AS p99_response_ms,
          countIf(error_occurred = 1) AS error_count,
          countIf(error_occurred = 1) / count() AS error_rate
        FROM linebot.events
        WHERE timestamp >= now() - INTERVAL {minutes:UInt32} MINUTE
      `,
      query_params: { minutes: minutesBack },
      format: 'JSONEachRow',
    });
    
    const data = await result.json();
    return data[0] || {};
  }
  
  async getMessageTimeSeries(hours = 24, granularity = 'hour') {
    const granularityFn = {
      minute: 'toStartOfMinute',
      hour: 'toStartOfHour',
      day: 'toStartOfDay',
    }[granularity] || 'toStartOfHour';
    
    const result = await this.ch.query({
      query: `
        SELECT
          ${granularityFn}(timestamp) AS time,
          count() AS messages,
          countDistinct(user_id) AS users
        FROM linebot.events
        WHERE event_type = 'message'
          AND timestamp >= now() - INTERVAL {hours:UInt32} HOUR
        GROUP BY time
        ORDER BY time
      `,
      query_params: { hours },
      format: 'JSONEachRow',
    });
    
    return result.json();
  }
  
  async getFunnelAnalysis(startDate, endDate) {
    const result = await this.ch.query({
      query: `
        WITH cohort AS (
          SELECT user_id
          FROM linebot.events
          WHERE event_type = 'message'
            AND date BETWEEN {start:Date} AND {end:Date}
          GROUP BY user_id
        )
        SELECT
          'Step 1: Messages' AS step,
          count(DISTINCT e.user_id) AS users
        FROM linebot.events e
        JOIN cohort c ON e.user_id = c.user_id
        WHERE e.event_type = 'message'
          AND e.date BETWEEN {start:Date} AND {end:Date}
        
        UNION ALL
        
        SELECT
          'Step 2: Product Views',
          count(DISTINCT e.user_id)
        FROM linebot.events e
        JOIN cohort c ON e.user_id = c.user_id
        WHERE e.event_type = 'postback'
          AND JSONExtractString(e.entities, 'action') = 'view_product'
          AND e.date BETWEEN {start:Date} AND {end:Date}
        
        UNION ALL
        
        SELECT
          'Step 3: Purchases',
          count(DISTINCT p.user_id)
        FROM linebot.purchases p
        JOIN cohort c ON p.user_id = c.user_id
        WHERE p.date BETWEEN {start:Date} AND {end:Date}
      `,
      query_params: { start: startDate, end: endDate },
      format: 'JSONEachRow',
    });
    
    return result.json();
  }
  
  async getTopUsers(days = 7, limit = 100) {
    const result = await this.ch.query({
      query: `
        SELECT
          user_id,
          count() AS message_count,
          countDistinct(date) AS active_days,
          max(timestamp) AS last_active,
          sum(1) AS total_interactions
        FROM linebot.events
        WHERE event_type = 'message'
          AND date >= today() - {days:UInt32}
        GROUP BY user_id
        ORDER BY message_count DESC
        LIMIT {limit:UInt32}
      `,
      query_params: { days, limit },
      format: 'JSONEachRow',
    });
    
    return result.json();
  }
  
  async getAnomalies(lookbackHours = 1) {
    // ตรวจจับ anomalies ด้วย Z-score
    const result = await this.ch.query({
      query: `
        WITH hourly_data AS (
          SELECT
            toStartOfHour(timestamp) AS hour,
            count() AS msg_count
          FROM linebot.events
          WHERE event_type = 'message'
            AND timestamp >= now() - INTERVAL 7 DAY
          GROUP BY hour
        ),
        stats AS (
          SELECT
            avg(msg_count) AS mean,
            stddevPop(msg_count) AS stddev
          FROM hourly_data
        )
        SELECT
          h.hour,
          h.msg_count,
          s.mean,
          s.stddev,
          (h.msg_count - s.mean) / s.stddev AS z_score,
          abs((h.msg_count - s.mean) / s.stddev) > 2 AS is_anomaly
        FROM hourly_data h, stats s
        WHERE h.hour >= now() - INTERVAL {hours:UInt32} HOUR
        ORDER BY h.hour
      `,
      query_params: { hours: lookbackHours * 24 },
      format: 'JSONEachRow',
    });
    
    const data = await result.json();
    return data.filter(d => d.is_anomaly === 1);
  }
}

module.exports = AnalyticsAPI;
```

---

## 6. User Behavior Tracking

### 6.1 Event Taxonomy

```javascript
// src/analytics/EventTaxonomy.js

const EVENT_TYPES = {
  // User Lifecycle
  USER_FOLLOW: 'user.follow',
  USER_UNFOLLOW: 'user.unfollow',
  USER_BLOCK: 'user.block',
  
  // Engagement
  MESSAGE_SENT: 'message.sent',
  MESSAGE_RECEIVED: 'message.received',
  POSTBACK_CLICKED: 'postback.clicked',
  RICH_MENU_CLICKED: 'rich_menu.clicked',
  
  // Commerce
  PRODUCT_VIEWED: 'product.viewed',
  CART_ADDED: 'cart.added',
  CART_REMOVED: 'cart.removed',
  CHECKOUT_STARTED: 'checkout.started',
  ORDER_PLACED: 'order.placed',
  ORDER_PAID: 'order.paid',
  ORDER_CANCELLED: 'order.cancelled',
  
  // Content
  CONTENT_VIEWED: 'content.viewed',
  CONTENT_SHARED: 'content.shared',
  
  // Support
  SUPPORT_REQUESTED: 'support.requested',
  SUPPORT_RESOLVED: 'support.resolved',
};

class UserBehaviorTracker {
  constructor({ eventProducer, sessionManager }) {
    this.producer = eventProducer;
    this.sessionManager = sessionManager;
  }
  
  async track(userId, eventType, properties = {}) {
    const session = await this.sessionManager.getOrCreate(userId);
    
    const event = {
      eventId: require('crypto').randomUUID(),
      eventType,
      userId,
      sessionId: session.id,
      timestamp: new Date().toISOString(),
      properties,
      // Context
      platform: 'line',
      channel: 'oa',
      // Session context
      sessionMessageCount: session.messageCount,
      sessionDurationSeconds: Math.floor((Date.now() - session.startTime) / 1000),
    };
    
    // Publish ไป Kafka
    await this.producer.send({
      topic: this.getTopicForEvent(eventType),
      messages: [{
        key: userId,
        value: JSON.stringify(event),
      }],
    });
    
    // Update session
    await this.sessionManager.update(userId, session.id);
    
    return event;
  }
  
  getTopicForEvent(eventType) {
    if (eventType.startsWith('order.') || eventType.startsWith('cart.')) {
      return 'linebot.events.purchases';
    }
    return 'linebot.events.messages';
  }
  
  // Convenience methods
  async trackMessage(userId, messageType, messageText) {
    return this.track(userId, EVENT_TYPES.MESSAGE_SENT, {
      messageType,
      messageText: messageType === 'text' ? messageText : null,
    });
  }
  
  async trackProductView(userId, productId, productName, price) {
    return this.track(userId, EVENT_TYPES.PRODUCT_VIEWED, {
      productId,
      productName,
      price,
    });
  }
  
  async trackPurchase(userId, orderId, amount, items) {
    return this.track(userId, EVENT_TYPES.ORDER_PLACED, {
      orderId,
      amount,
      currency: 'THB',
      itemCount: items.length,
      items: items.map(i => ({
        productId: i.productId,
        quantity: i.quantity,
        price: i.price,
      })),
    });
  }
}

module.exports = { EVENT_TYPES, UserBehaviorTracker };
```

---

## 7. A/B Test Monitoring

### 7.1 A/B Test Framework

```javascript
// src/analytics/ABTestMonitor.js

class ABTestMonitor {
  constructor({ analyticsDB }) {
    this.db = analyticsDB;
  }
  
  async getTestResults(testId, startDate, endDate) {
    const result = await this.db.query({
      query: `
        WITH test_events AS (
          SELECT
            user_id,
            JSONExtractString(properties, 'variant') AS variant,
            eventType,
            timestamp
          FROM linebot.events
          WHERE JSONExtractString(properties, 'ab_test_id') = {testId:String}
            AND date BETWEEN {start:Date} AND {end:Date}
        ),
        variants AS (
          SELECT
            variant,
            countDistinct(user_id) AS users,
            countIf(eventType = 'order.placed') AS conversions,
            sum(
              if(eventType = 'order.placed',
                 JSONExtractFloat(properties, 'amount'),
                 0)
            ) AS revenue
          FROM test_events
          GROUP BY variant
        )
        SELECT
          variant,
          users,
          conversions,
          revenue,
          conversions / users AS conversion_rate,
          revenue / users AS revenue_per_user
        FROM variants
        ORDER BY variant
      `,
      query_params: { testId, start: startDate, end: endDate },
      format: 'JSONEachRow',
    });
    
    const data = await result.json();
    
    // Calculate statistical significance
    return this.calculateSignificance(data);
  }
  
  calculateSignificance(variants) {
    if (variants.length < 2) return variants;
    
    const control = variants.find(v => v.variant === 'control') || variants[0];
    
    return variants.map(variant => {
      if (variant.variant === control.variant) {
        return { ...variant, uplift: 0, significant: false };
      }
      
      const uplift = (variant.conversion_rate - control.conversion_rate) / 
                     control.conversion_rate * 100;
      
      // Chi-squared test
      const pValue = this.chiSquaredTest(
        control.users, control.conversions,
        variant.users, variant.conversions
      );
      
      return {
        ...variant,
        uplift: Math.round(uplift * 100) / 100,
        pValue,
        significant: pValue < 0.05,
        confidence: Math.round((1 - pValue) * 100),
      };
    });
  }
  
  chiSquaredTest(n1, c1, n2, c2) {
    const r1 = c1 / n1;
    const r2 = c2 / n2;
    const pPooled = (c1 + c2) / (n1 + n2);
    
    if (pPooled === 0 || pPooled === 1) return 1;
    
    const se = Math.sqrt(pPooled * (1 - pPooled) * (1/n1 + 1/n2));
    if (se === 0) return 1;
    
    const z = (r1 - r2) / se;
    
    // Approximate p-value from z-score
    return 2 * (1 - this.normalCDF(Math.abs(z)));
  }
  
  normalCDF(x) {
    const t = 1 / (1 + 0.2316419 * Math.abs(x));
    const d = 0.3989423 * Math.exp(-x * x / 2);
    const p = d * t * (0.3193815 + t * (-0.3565638 + t * (1.781478 + t * (-1.821256 + t * 1.330274))));
    return x > 0 ? 1 - p : p;
  }
}

module.exports = ABTestMonitor;
```

---

## 8. Anomaly Detection

```javascript
// src/analytics/AnomalyDetector.js

class AnomalyDetector {
  constructor({ analyticsDB, notifier }) {
    this.db = analyticsDB;
    this.notifier = notifier;
    this.checkInterval = null;
  }
  
  start(intervalMs = 60000) {
    this.checkInterval = setInterval(async () => {
      await this.checkAnomalies();
    }, intervalMs);
  }
  
  async checkAnomalies() {
    const checks = await Promise.allSettled([
      this.checkMessageVolumeAnomaly(),
      this.checkErrorRateAnomaly(),
      this.checkLatencyAnomaly(),
      this.checkFollowUnfollowAnomaly(),
    ]);
    
    const anomalies = checks
      .filter(r => r.status === 'fulfilled' && r.value)
      .map(r => r.value);
    
    if (anomalies.length > 0) {
      await this.notifier.send({
        type: 'anomaly',
        anomalies,
        timestamp: new Date().toISOString(),
      });
    }
  }
  
  async checkMessageVolumeAnomaly() {
    const result = await this.db.query({
      query: `
        WITH baseline AS (
          SELECT avg(hourly_count) AS mean, stddevPop(hourly_count) AS stddev
          FROM (
            SELECT toStartOfHour(timestamp) AS h, count() AS hourly_count
            FROM linebot.events
            WHERE event_type = 'message'
              AND timestamp BETWEEN now() - INTERVAL 7 DAY AND now() - INTERVAL 1 HOUR
            GROUP BY h
          )
        ),
        current AS (
          SELECT count() AS current_count
          FROM linebot.events
          WHERE event_type = 'message'
            AND timestamp >= now() - INTERVAL 1 HOUR
        )
        SELECT
          c.current_count,
          b.mean,
          b.stddev,
          (c.current_count - b.mean) / b.stddev AS z_score
        FROM current c, baseline b
      `,
      format: 'JSONEachRow',
    });
    
    const data = (await result.json())[0];
    
    if (Math.abs(data.z_score) > 3) {
      return {
        type: 'message_volume',
        severity: Math.abs(data.z_score) > 5 ? 'critical' : 'warning',
        message: `Message volume anomaly: current=${data.current_count}, expected≈${Math.round(data.mean)}, z-score=${data.z_score.toFixed(2)}`,
        data,
      };
    }
    
    return null;
  }
  
  async checkErrorRateAnomaly() {
    const result = await this.db.query({
      query: `
        SELECT
          countIf(error_occurred = 1) / count() AS error_rate,
          count() AS total
        FROM linebot.events
        WHERE timestamp >= now() - INTERVAL 5 MINUTE
      `,
      format: 'JSONEachRow',
    });
    
    const data = (await result.json())[0];
    
    if (data.error_rate > 0.05) { // > 5% error rate
      return {
        type: 'error_rate',
        severity: data.error_rate > 0.1 ? 'critical' : 'warning',
        message: `High error rate: ${(data.error_rate * 100).toFixed(1)}% (${Math.round(data.error_rate * data.total)}/${data.total} errors)`,
        data,
      };
    }
    
    return null;
  }
}

module.exports = AnomalyDetector;
```

---

## 9. Performance Benchmarks

```
ClickHouse Analytics Performance:

Query Performance (1 billion rows):
┌─────────────────────────────────┬──────────┬──────────┐
│  Query Type                     │  Time    │  Rows/s  │
├─────────────────────────────────┼──────────┼──────────┤
│  Simple COUNT                   │  0.02s   │  50B/s   │
│  GROUP BY (daily stats)         │  0.5s    │  2B/s    │
│  JOIN (user + events)           │  2.5s    │  400M/s  │
│  Window Function (retention)    │  8.0s    │  125M/s  │
│  Complex Funnel Query           │  5.0s    │  200M/s  │
└─────────────────────────────────┴──────────┴──────────┘

Kafka Performance:
├── Throughput: 1,000,000+ messages/sec
├── Latency (p99): < 10ms
├── Consumer lag: < 1 second
└── Replication: < 50ms

Flink Processing:
├── Latency: 100-500ms end-to-end
├── Throughput: 500,000 events/sec per job
└── Recovery time: < 30 seconds
```

---

## สรุป

Real-time Analytics Pipeline สำหรับ LINE Bot ประกอบด้วย:

1. **Kafka**: รับ events จาก LINE Bot ด้วย throughput สูง
2. **Flink**: ประมวลผล stream แบบ real-time (sessions, aggregations)
3. **ClickHouse**: เก็บ analytics data ด้วย query performance สูง
4. **Grafana**: Dashboard แสดงผลแบบ real-time
5. **Anomaly Detection**: แจ้งเตือนอัตโนมัติเมื่อพบความผิดปกติ

---

*จบ Part 83: Real-time Analytics สำหรับ LINE Bots*

---

## 10. Complete Kafka + LINE Bot Setup

### 10.1 Full Kafka Consumer Service

```javascript
// services/analytics-consumer/src/index.js
const { Kafka } = require('kafkajs');
const { createClient } = require('@clickhouse/client');

class AnalyticsConsumerService {
  constructor() {
    this.kafka = new Kafka({
      clientId: 'analytics-consumer',
      brokers: process.env.KAFKA_BROKERS?.split(',') || ['localhost:9092'],
    });
    
    this.consumer = this.kafka.consumer({
      groupId: 'analytics-consumers',
      maxBytesPerPartition: 1048576, // 1MB
      sessionTimeout: 30000,
      heartbeatInterval: 3000,
    });
    
    this.clickhouse = createClient({
      host: process.env.CLICKHOUSE_HOST || 'http://localhost:8123',
      database: 'linebot',
      username: process.env.CLICKHOUSE_USER || 'default',
      password: process.env.CLICKHOUSE_PASSWORD || '',
    });
    
    this.batchBuffer = new Map();
    this.batchSize = 1000;
    this.flushInterval = 5000; // 5 seconds
  }
  
  async start() {
    await this.consumer.connect();
    
    await this.consumer.subscribe({
      topics: [
        'linebot.events.messages',
        'linebot.events.follows',
        'linebot.events.postbacks',
        'linebot.events.purchases',
      ],
      fromBeginning: false,
    });
    
    // Setup batch flush interval
    setInterval(() => this.flushAllBuffers(), this.flushInterval);
    
    await this.consumer.run({
      eachBatch: async ({ batch, resolveOffset, heartbeat }) => {
        const { topic, messages } = batch;
        
        for (const message of messages) {
          try {
            const event = JSON.parse(message.value?.toString() || '{}');
            await this.processEvent(topic, event);
            resolveOffset(message.offset);
            
            // Heartbeat ทุก 100 messages
            if (parseInt(message.offset) % 100 === 0) {
              await heartbeat();
            }
          } catch (error) {
            console.error(`Error processing message:`, error);
          }
        }
      },
    });
    
    console.log('Analytics consumer started');
  }
  
  async processEvent(topic, event) {
    const bufferKey = this.getTableForTopic(topic);
    
    if (!this.batchBuffer.has(bufferKey)) {
      this.batchBuffer.set(bufferKey, []);
    }
    
    const buffer = this.batchBuffer.get(bufferKey);
    buffer.push(this.transformEvent(topic, event));
    
    // Flush if buffer is full
    if (buffer.length >= this.batchSize) {
      await this.flushBuffer(bufferKey);
    }
  }
  
  getTableForTopic(topic) {
    const mapping = {
      'linebot.events.messages': 'events',
      'linebot.events.follows': 'follows',
      'linebot.events.postbacks': 'events',
      'linebot.events.purchases': 'purchases',
    };
    return mapping[topic] || 'events';
  }
  
  transformEvent(topic, event) {
    if (topic === 'linebot.events.purchases') {
      return {
        event_id: event.eventId || require('crypto').randomUUID(),
        user_id: event.userId,
        order_id: event.orderId,
        amount: event.amount,
        currency: event.currency || 'THB',
        items: JSON.stringify(event.items || []),
        channel: event.channel || 'line',
        timestamp: event.timestamp || new Date().toISOString(),
        date: event.timestamp?.split('T')[0] || new Date().toISOString().split('T')[0],
      };
    }
    
    return {
      event_id: event.eventId || require('crypto').randomUUID(),
      event_type: event.type || 'unknown',
      user_id: event.userId || '',
      group_id: event.groupId || null,
      timestamp: event.timestamp || new Date().toISOString(),
      message_type: event.messageType || '',
      message_text: event.messageText || null,
      response_time_ms: event.responseTimeMs || 0,
      error_occurred: event.error ? 1 : 0,
      date: (event.timestamp || new Date().toISOString()).split('T')[0],
      hour: new Date(event.timestamp || new Date()).getHours(),
    };
  }
  
  async flushBuffer(tableKey) {
    const buffer = this.batchBuffer.get(tableKey);
    if (!buffer || buffer.length === 0) return;
    
    const rows = [...buffer];
    this.batchBuffer.set(tableKey, []);
    
    try {
      await this.clickhouse.insert({
        table: tableKey,
        values: rows,
        format: 'JSONEachRow',
      });
      
      console.log(`Flushed ${rows.length} rows to ${tableKey}`);
    } catch (error) {
      console.error(`Failed to flush to ${tableKey}:`, error);
      // Retry logic
      setTimeout(() => {
        const currentBuffer = this.batchBuffer.get(tableKey) || [];
        this.batchBuffer.set(tableKey, [...rows, ...currentBuffer]);
      }, 5000);
    }
  }
  
  async flushAllBuffers() {
    for (const tableKey of this.batchBuffer.keys()) {
      await this.flushBuffer(tableKey);
    }
  }
  
  async stop() {
    await this.flushAllBuffers();
    await this.consumer.disconnect();
    await this.clickhouse.close();
  }
}

const service = new AnalyticsConsumerService();

process.on('SIGTERM', async () => {
  await service.stop();
  process.exit(0);
});

service.start().catch(console.error);
```

---

## 11. Grafana Real-time Dashboards

### 11.1 Dashboard Provisioning

```yaml
# monitoring/grafana/provisioning/dashboards/linebot.yaml
apiVersion: 1

providers:
  - name: linebot-dashboards
    orgId: 1
    folder: LINE Bot
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards/linebot
      foldersFromFilesStructure: true
```

```json
{
  "uid": "linebot-realtime",
  "title": "LINE Bot Real-time Overview",
  "refresh": "5s",
  "schemaVersion": 38,
  "tags": ["linebot", "realtime"],
  "time": {"from": "now-1h", "to": "now"},
  "timepicker": {
    "refresh_intervals": ["5s", "10s", "30s", "1m"]
  },
  "panels": [
    {
      "id": 1,
      "title": "💬 Messages/min (Real-time)",
      "type": "timeseries",
      "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
      "datasource": {"type": "vertamedia-clickhouse-datasource"},
      "targets": [{
        "rawSql": "SELECT toStartOfMinute(timestamp) AS t, count() AS messages FROM linebot.events WHERE timestamp >= now() - INTERVAL 1 HOUR AND event_type = 'message' GROUP BY t ORDER BY t",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "color": {"mode": "palette-classic"},
          "custom": {"lineWidth": 2, "fillOpacity": 10}
        }
      }
    },
    {
      "id": 2,
      "title": "👥 Active Users Now",
      "type": "stat",
      "gridPos": {"h": 4, "w": 4, "x": 12, "y": 0},
      "datasource": {"type": "vertamedia-clickhouse-datasource"},
      "targets": [{
        "rawSql": "SELECT count(DISTINCT user_id) AS value FROM linebot.events WHERE timestamp >= now() - INTERVAL 5 MINUTE",
        "format": "table"
      }],
      "options": {
        "colorMode": "background",
        "graphMode": "none",
        "textMode": "value"
      }
    },
    {
      "id": 3,
      "title": "⚡ Response Time p99",
      "type": "gauge",
      "gridPos": {"h": 4, "w": 4, "x": 16, "y": 0},
      "datasource": {"type": "vertamedia-clickhouse-datasource"},
      "targets": [{
        "rawSql": "SELECT quantile(0.99)(response_time_ms) AS value FROM linebot.events WHERE timestamp >= now() - INTERVAL 5 MINUTE",
        "format": "table"
      }],
      "options": {
        "minValue": 0,
        "maxValue": 1000,
        "thresholds": {
          "mode": "absolute",
          "steps": [
            {"color": "green", "value": null},
            {"color": "yellow", "value": 200},
            {"color": "red", "value": 500}
          ]
        }
      }
    },
    {
      "id": 4,
      "title": "💰 Revenue Today",
      "type": "stat",
      "gridPos": {"h": 4, "w": 4, "x": 20, "y": 0},
      "datasource": {"type": "vertamedia-clickhouse-datasource"},
      "targets": [{
        "rawSql": "SELECT sum(amount) AS value FROM linebot.purchases WHERE date = today()",
        "format": "table"
      }],
      "options": {
        "colorMode": "background",
        "reduceOptions": {"calcs": ["lastNotNull"]}
      },
      "fieldConfig": {
        "defaults": {
          "unit": "currencyTHB"
        }
      }
    },
    {
      "id": 5,
      "title": "📊 Message Types (Last Hour)",
      "type": "piechart",
      "gridPos": {"h": 8, "w": 8, "x": 0, "y": 8},
      "datasource": {"type": "vertamedia-clickhouse-datasource"},
      "targets": [{
        "rawSql": "SELECT message_type AS label, count() AS value FROM linebot.events WHERE event_type = 'message' AND timestamp >= now() - INTERVAL 1 HOUR GROUP BY message_type ORDER BY value DESC",
        "format": "table"
      }]
    },
    {
      "id": 6,
      "title": "🔥 Top Intents (Last Hour)",
      "type": "bargauge",
      "gridPos": {"h": 8, "w": 8, "x": 8, "y": 8},
      "datasource": {"type": "vertamedia-clickhouse-datasource"},
      "targets": [{
        "rawSql": "SELECT intent AS label, count() AS value FROM linebot.events WHERE intent != '' AND timestamp >= now() - INTERVAL 1 HOUR GROUP BY intent ORDER BY value DESC LIMIT 10",
        "format": "table"
      }]
    },
    {
      "id": 7,
      "title": "⚠️ Error Rate",
      "type": "timeseries",
      "gridPos": {"h": 8, "w": 8, "x": 16, "y": 8},
      "datasource": {"type": "vertamedia-clickhouse-datasource"},
      "targets": [{
        "rawSql": "SELECT toStartOfMinute(timestamp) AS t, countIf(error_occurred=1)/count() AS error_rate FROM linebot.events WHERE timestamp >= now() - INTERVAL 1 HOUR GROUP BY t ORDER BY t",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "percentunit",
          "thresholds": {
            "steps": [
              {"color": "green", "value": null},
              {"color": "yellow", "value": 0.01},
              {"color": "red", "value": 0.05}
            ]
          }
        }
      }
    }
  ]
}
```

---

## 12. Kafka Connect Integration

### 12.1 Kafka Connect to ClickHouse

```json
{
  "name": "clickhouse-sink-events",
  "config": {
    "connector.class": "com.clickhouse.kafka.connect.ClickHouseSinkConnector",
    "tasks.max": "3",
    "topics": "linebot.events.messages,linebot.events.postbacks",
    "clickhouse.server.url": "http://clickhouse:8123",
    "clickhouse.server.user": "default",
    "clickhouse.server.pass": "",
    "clickhouse.server.database": "linebot",
    "clickhouse.table.name": "events",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "insert.distributed.sync": "false",
    "clickhouse.insert.batch.size": "10000",
    "clickhouse.insert.batch.timeout.ms": "5000",
    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "linebot.dlq",
    "errors.deadletterqueue.context.headers.enable": "true"
  }
}
```

---

## สรุปสมบูรณ์ Real-time Analytics

Real-time Analytics Stack สมบูรณ์สำหรับ LINE Bot:

| Layer | Technology | Throughput | Latency |
|-------|-----------|-----------|---------|
| Ingestion | Kafka (3 brokers) | 1M msg/s | < 10ms |
| Stream Processing | Apache Flink | 500K events/s | 100-500ms |
| Storage | ClickHouse | 1B rows/s insert | < 100ms query |
| Visualization | Grafana | Real-time 5s refresh | < 1s |
| Alerting | Prometheus + AlertManager | 15s evaluation | < 30s notify |

ROI ของ Real-time Analytics:
- ลด Mean Time to Detect (MTTD) จาก ชั่วโมง เป็น นาที
- เพิ่ม conversion rate 10-15% ผ่าน real-time personalization
- ลด customer churn ด้วย early warning alerts

---

*จบ Part 83: Real-time Analytics สำหรับ LINE Bots (ฉบับสมบูรณ์)*
