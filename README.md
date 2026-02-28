# 📈 RealStream — Real-Time Data Pipeline with Kafka & Spark

[![Scala](https://img.shields.io/badge/Scala-2.13-DC322F?style=for-the-badge&logo=scala&logoColor=white)](https://www.scala-lang.org/)
[![Apache Kafka](https://img.shields.io/badge/Apache_Kafka-3.x-231F20?style=for-the-badge&logo=apache-kafka&logoColor=white)](https://kafka.apache.org/)
[![Apache Spark](https://img.shields.io/badge/Apache_Spark-3.x-E25A1C?style=for-the-badge&logo=apachespark&logoColor=white)](https://spark.apache.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-316192?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Docker](https://img.shields.io/badge/Docker-2CA5E0?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)

> An end-to-end real-time data pipeline that ingests high-volume event streams via Apache Kafka, processes and transforms them using Apache Spark Structured Streaming, and persists results to PostgreSQL — fully containerized with Docker.

---

## 📋 Table of Contents
- [Architecture](#architecture)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Getting Started](#getting-started)
- [Pipeline Stages](#pipeline-stages)
- [Data Flow](#data-flow)

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    REALSTREAM PIPELINE                          │
│                                                                 │
│  Data Sources         Ingestion           Processing           │
│  ┌──────────┐        ┌─────────┐         ┌──────────────┐     │
│  │ Events   │──────▶ │  Kafka  │────────▶│ Spark        │     │
│  │ Logs     │        │ Topics  │         │ Structured   │     │
│  │ Metrics  │        │         │         │ Streaming    │     │
│  └──────────┘        └─────────┘         └──────┬───────┘     │
│                                                  │             │
│                        Storage                   ▼             │
│                      ┌───────────────────────────────┐        │
│                      │         PostgreSQL             │        │
│                      │    (Processed Data Sink)       │        │
│                      └───────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

---

## ✨ Features

- 🚀 **Real-Time Ingestion** — Kafka producers push events at high throughput
- ⚡ **Stream Processing** — Spark Structured Streaming with windowed aggregations
- 🔄 **ETL Transformations** — Schema validation, data cleansing, enrichment
- 🗄️ **Persistent Storage** — Processed records written to PostgreSQL
- 📊 **Aggregations** — Time-window metrics, event counts, rolling averages
- 🐳 **Dockerized** — Single `docker-compose up` to spin up the full stack
- 📝 **Schema Evolution** — Handles evolving event schemas gracefully
- 🔁 **Fault Tolerant** — Kafka consumer group offsets for at-least-once delivery

---

## 🛠️ Tech Stack

| Component | Technology |
|---|---|
| Streaming Platform | Apache Kafka 3.x |
| Stream Processor | Apache Spark 3.x Structured Streaming |
| Language | Scala 2.13 |
| Database (Sink) | PostgreSQL 15 |
| Containerization | Docker + Docker Compose |
| Build Tool | SBT (Scala Build Tool) |

---

## 🚀 Getting Started

### Prerequisites
- Docker & Docker Compose
- Java 11+
- SBT (Scala Build Tool)

### Run with Docker
```bash
git clone https://github.com/Amanrai1480/RealStream-DataPipeline.git
cd RealStream-DataPipeline

# Start Kafka, Zookeeper, and PostgreSQL
docker-compose up -d

# Build the Spark application
sbt assembly

# Submit the Spark streaming job
spark-submit \
  --class com.realstream.StreamProcessor \
  --master local[*] \
  target/scala-2.13/realstream-assembly-1.0.jar
```

### Create Kafka Topics
```bash
# Create input topic
kafka-topics.sh --create --topic events-input \
  --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1

# Create output/error topics
kafka-topics.sh --create --topic events-processed \
  --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
```

---

## 🔄 Pipeline Stages

### Stage 1: Data Ingestion (Kafka Producer)
```scala
// KafkaProducer pushes JSON events to input topic
val producer = new KafkaProducer[String, String](producerProps)
val record = new ProducerRecord("events-input", key, jsonEvent)
producer.send(record)
```

### Stage 2: Stream Processing (Spark)
```scala
// Read stream from Kafka
val kafkaDF = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "events-input")
  .load()

// Parse JSON and apply transformations
val eventsDF = kafkaDF
  .selectExpr("CAST(value AS STRING) as json_str")
  .select(from_json(col("json_str"), eventSchema).as("event"))
  .select("event.*")
  .filter(col("event_type").isNotNull)

// Windowed aggregation — count events per type per 5-minute window
val aggregated = eventsDF
  .withWatermark("event_time", "10 minutes")
  .groupBy(window(col("event_time"), "5 minutes"), col("event_type"))
  .agg(count("*").as("event_count"), avg("value").as("avg_value"))
```

### Stage 3: Sink to PostgreSQL
```scala
// Write processed stream to PostgreSQL
aggregated.writeStream
  .foreachBatch { (batchDF, batchId) =>
    batchDF.write
      .format("jdbc")
      .option("url", "jdbc:postgresql://localhost:5432/realstream_db")
      .option("dbtable", "event_aggregations")
      .mode("append")
      .save()
  }
  .outputMode("update")
  .start()
  .awaitTermination()
```

---

## 📊 Data Flow

```
Raw Events (JSON)
     │
     ▼
Kafka Topic: events-input
     │
     ▼
Spark Structured Streaming
  ├── Schema Validation
  ├── Null / Bad Record Filtering
  ├── Field Transformation (type casting, enrichment)
  ├── Windowed Aggregation (5-min tumbling windows)
  └── Watermark handling (10-min late data)
     │
     ▼
PostgreSQL Tables:
  ├── raw_events          (all valid incoming events)
  ├── event_aggregations  (windowed aggregation results)
  └── pipeline_errors     (schema/parse failures)
```

---

## 🗄️ Database Schema

```sql
-- Raw Events
CREATE TABLE raw_events (
    id              BIGSERIAL PRIMARY KEY,
    event_id        VARCHAR(50) UNIQUE,
    event_type      VARCHAR(100),
    user_id         BIGINT,
    value           NUMERIC,
    payload         JSONB,
    event_time      TIMESTAMP,
    ingested_at     TIMESTAMP DEFAULT NOW()
);

-- Windowed Aggregations
CREATE TABLE event_aggregations (
    id              BIGSERIAL PRIMARY KEY,
    window_start    TIMESTAMP,
    window_end      TIMESTAMP,
    event_type      VARCHAR(100),
    event_count     BIGINT,
    avg_value       NUMERIC,
    processed_at    TIMESTAMP DEFAULT NOW()
);
```

---

## 🐳 Docker Compose Services

```yaml
services:
  zookeeper:   # Kafka coordination
  kafka:       # Message broker (port 9092)
  postgres:    # Data sink (port 5432)
  kafka-ui:    # Kafka monitoring UI (port 8080)
```

---

## 📄 License
[MIT](LICENSE)
