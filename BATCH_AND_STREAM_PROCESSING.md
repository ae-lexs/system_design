# 15 - Batch vs Stream Processing

## Overview

Data processing paradigms fall into two categories: batch processing (processing bounded datasets periodically) and stream processing (processing unbounded data continuously). Modern systems often combine both in Lambda or Kappa architectures. This document covers the patterns, technologies, and trade-offs for each approach.

---

## Core Mental Model

```mermaid
flowchart TD
    subgraph Batch["Batch Processing"]
        B1["Process bounded dataset"]
        B2["High latency (minutes to hours)"]
        B3["High throughput"]
        B4["Examples: ETL, reporting"]
    end
    
    subgraph Stream["Stream Processing"]
        S1["Process unbounded stream"]
        S2["Low latency (ms to seconds)"]
        S3["Continuous processing"]
        S4["Examples: Real-time analytics, alerting"]
    end
    
    subgraph Hybrid["Hybrid Architectures"]
        H1["Lambda: Batch + Stream"]
        H2["Kappa: Stream only"]
    end
```

---

## Batch Processing

### Characteristics

```mermaid
flowchart LR
    subgraph BatchFlow["Batch Processing Flow"]
        Input["Input Data<br/>(Bounded)"] --> Process["Batch Job<br/>(MapReduce, Spark)"]
        Process --> Output["Output Data<br/>(Complete result)"]
    end
    
    Schedule["Scheduled Trigger<br/>(hourly, daily)"] --> Process
```

| Aspect | Description |
|--------|-------------|
| **Data** | Bounded, known size |
| **Latency** | Minutes to hours |
| **Throughput** | Very high |
| **Processing** | Complete dataset at once |
| **Fault tolerance** | Rerun entire job |
| **State** | Typically stateless |

### MapReduce Pattern

```mermaid
flowchart TB
    subgraph MapReduce["MapReduce Example: Word Count"]
        Input["Input: Log files"]
        
        subgraph Map["Map Phase"]
            M1["Mapper 1: 'hello' → (hello, 1)"]
            M2["Mapper 2: 'world' → (world, 1)"]
            M3["Mapper 3: 'hello' → (hello, 1)"]
        end
        
        subgraph Shuffle["Shuffle & Sort"]
            S["Group by key"]
        end
        
        subgraph Reduce["Reduce Phase"]
            R1["Reducer: (hello, [1,1]) → (hello, 2)"]
            R2["Reducer: (world, [1]) → (world, 1)"]
        end
        
        Output["Output: Word counts"]
    end
    
    Input --> Map
    Map --> Shuffle
    Shuffle --> Reduce
    Reduce --> Output
```

### Spark Example

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, avg

spark = SparkSession.builder.appName("DailyAnalytics").getOrCreate()

# Read day's data
orders = spark.read.parquet("s3://data/orders/date=2024-01-15/")

# Aggregate
daily_stats = orders.groupBy("product_id").agg(
    sum("quantity").alias("total_sold"),
    avg("price").alias("avg_price"),
    sum("revenue").alias("total_revenue")
)

# Write results
daily_stats.write.mode("overwrite").parquet("s3://analytics/daily/date=2024-01-15/")
```

### Batch Use Cases

| Use Case | Description |
|----------|-------------|
| **ETL** | Extract, Transform, Load for data warehousing |
| **ML Training** | Train models on historical data |
| **Daily Reports** | Generate business reports |
| **Data Backfill** | Reprocess historical data |
| **Data Validation** | Verify data quality |

---

## Stream Processing

### Characteristics

```mermaid
flowchart LR
    subgraph StreamFlow["Stream Processing Flow"]
        Input["Input Stream<br/>(Unbounded)"] --> Process["Stream Processor<br/>(Kafka Streams, Flink)"]
        Process --> Output["Output Stream<br/>(Continuous)"]
    end
    
    Note["Events processed<br/>as they arrive"]
```

| Aspect | Description |
|--------|-------------|
| **Data** | Unbounded, continuous |
| **Latency** | Milliseconds to seconds |
| **Throughput** | Moderate (per-event overhead) |
| **Processing** | Event by event |
| **Fault tolerance** | Checkpointing, exactly-once |
| **State** | Often stateful (windows, aggregations) |

### Stream Processing Concepts

```mermaid
flowchart TB
    subgraph Concepts["Key Concepts"]
        Event["Events<br/>(Immutable records)"]
        Time["Time Semantics<br/>(Event time vs Process time)"]
        Window["Windows<br/>(Tumbling, Sliding, Session)"]
        State["State Management<br/>(Aggregations, Joins)"]
        Watermark["Watermarks<br/>(Track event time progress)"]
    end
```

### Windowing Strategies

```mermaid
flowchart TB
    subgraph Tumbling["Tumbling Window (5 min)"]
        T1["00:00-05:00"]
        T2["05:00-10:00"]
        T3["10:00-15:00"]
    end
    
    subgraph Sliding["Sliding Window (5 min, slide 1 min)"]
        S1["00:00-05:00"]
        S2["01:00-06:00"]
        S3["02:00-07:00"]
    end
    
    subgraph Session["Session Window (gap: 5 min)"]
        SS1["User activity cluster 1"]
        Gap["Gap > 5 min"]
        SS2["User activity cluster 2"]
    end
```

| Window Type | Description | Use Case |
|-------------|-------------|----------|
| **Tumbling** | Fixed, non-overlapping | Hourly aggregates |
| **Sliding** | Fixed, overlapping | Moving averages |
| **Session** | Dynamic, activity-based | User sessions |

### Event Time vs Processing Time

```mermaid
sequenceDiagram
    participant Source
    participant Kafka
    participant Processor
    
    Note over Source: Event created at T=10:00:00
    Source->>Kafka: Send event (delayed)
    Note over Kafka: Received at T=10:00:05
    Kafka->>Processor: Deliver event
    Note over Processor: Processed at T=10:00:07
    
    Note over Source,Processor: Event Time: 10:00:00<br/>Processing Time: 10:00:07
```

**Event Time**: When the event actually occurred (embedded in event)
**Processing Time**: When the system processes the event

**Why it matters**: Late-arriving events, out-of-order events

### Watermarks

```mermaid
flowchart LR
    subgraph Watermark["Watermark Concept"]
        Events["Events arriving"]
        WM["Watermark: 10:05<br/>'All events before 10:05<br/>have arrived'"]
        Late["Late event: 10:03<br/>(after watermark)"]
    end
    
    Events --> WM
    Late -->|"Handle specially"| WM
```

### Flink Example

```java
// Real-time click analytics
DataStream<Click> clicks = env
    .addSource(new FlinkKafkaConsumer<>("clicks", schema, props));

DataStream<ClickStats> stats = clicks
    // Use event time
    .assignTimestampsAndWatermarks(
        WatermarkStrategy.<Click>forBoundedOutOfOrderness(Duration.ofSeconds(5))
            .withTimestampAssigner((click, ts) -> click.getTimestamp())
    )
    // Group by page
    .keyBy(Click::getPageId)
    // 5-minute tumbling window
    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
    // Aggregate
    .aggregate(new ClickCountAggregate());

stats.addSink(new FlinkKafkaProducer<>("click-stats", schema, props));
```

### Kafka Streams Example

```java
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Order> orders = builder.stream("orders");

// Real-time revenue per product
KTable<Windowed<String>, Double> revenuePerProduct = orders
    .groupBy((key, order) -> order.getProductId())
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .aggregate(
        () -> 0.0,
        (key, order, total) -> total + order.getRevenue(),
        Materialized.with(Serdes.String(), Serdes.Double())
    );

revenuePerProduct.toStream()
    .to("revenue-by-product");
```

### Stream Use Cases

| Use Case | Description |
|----------|-------------|
| **Real-time analytics** | Live dashboards, monitoring |
| **Fraud detection** | Immediate anomaly detection |
| **Alerting** | Threshold-based notifications |
| **Event-driven apps** | Microservices communication |
| **IoT processing** | Sensor data analysis |

---

## Exactly-Once Semantics

### Delivery Guarantees

```mermaid
flowchart TB
    subgraph Guarantees["Delivery Guarantees"]
        AtMost["At-Most-Once<br/>May lose events"]
        AtLeast["At-Least-Once<br/>May duplicate events"]
        Exactly["Exactly-Once<br/>Each event processed once"]
    end
```

| Guarantee | Description | Trade-off |
|-----------|-------------|-----------|
| **At-most-once** | Fire and forget | Fast, may lose data |
| **At-least-once** | Retry on failure | Safe, may duplicate |
| **Exactly-once** | Transactional | Slow, complex |

### Achieving Exactly-Once

```mermaid
sequenceDiagram
    participant Source as Kafka Source
    participant Processor
    participant Sink as Kafka Sink
    
    Source->>Processor: Read event (offset 100)
    Processor->>Processor: Process event
    
    Note over Processor,Sink: Transactional write
    Processor->>Sink: Write result + commit offset 100
    
    Note over Processor: If crash before commit:<br/>Replay from offset 100
```

**Key techniques**:
1. **Idempotent operations**: Same input always produces same output
2. **Transactional commits**: Output + offset committed together
3. **Checkpointing**: Periodic state snapshots

---

## Lambda Architecture

### Pattern

```mermaid
flowchart TB
    subgraph Lambda["Lambda Architecture"]
        Source["Data Source"]
        
        subgraph Batch["Batch Layer"]
            BStorage["Data Lake<br/>(S3, HDFS)"]
            BProcess["Batch Processing<br/>(Spark)"]
            BView["Batch Views"]
        end
        
        subgraph Speed["Speed Layer"]
            SQueue["Message Queue<br/>(Kafka)"]
            SProcess["Stream Processing<br/>(Flink)"]
            SView["Real-time Views"]
        end
        
        subgraph Serving["Serving Layer"]
            Merge["Merge Views"]
            Query["Query Interface"]
        end
    end
    
    Source --> BStorage
    Source --> SQueue
    BStorage --> BProcess
    BProcess --> BView
    SQueue --> SProcess
    SProcess --> SView
    BView --> Merge
    SView --> Merge
    Merge --> Query
```

### Characteristics

| Aspect | Batch Layer | Speed Layer |
|--------|-------------|-------------|
| **Latency** | High (hours) | Low (seconds) |
| **Accuracy** | High (complete data) | Approximate |
| **Complexity** | Lower | Higher |
| **Data** | All historical | Recent only |

### Pros and Cons

| Pros | Cons |
|------|------|
| Best of both worlds | Code duplication (batch + stream) |
| Accurate historical data | Operational complexity |
| Low-latency real-time | Two systems to maintain |
| Fault tolerant | Debugging across layers |

---

## Kappa Architecture

### Pattern

```mermaid
flowchart TB
    subgraph Kappa["Kappa Architecture"]
        Source["Data Source"]
        Queue["Message Queue<br/>(Kafka)<br/>Retains all data"]
        Process["Stream Processing<br/>(Flink)"]
        View["Materialized Views"]
        Query["Query Interface"]
    end
    
    Source --> Queue
    Queue --> Process
    Process --> View
    View --> Query
    
    Note["For reprocessing:<br/>Replay from Kafka"]
```

### Reprocessing in Kappa

```mermaid
sequenceDiagram
    participant Kafka as Kafka (Full History)
    participant V1 as Processor V1
    participant V2 as Processor V2
    participant View as Materialized View
    
    Note over Kafka,View: Normal operation
    Kafka->>V1: Stream events
    V1->>View: Update view (Table A)
    
    Note over Kafka,V2: Deploy new processor version
    Kafka->>V2: Replay all events from beginning
    V2->>View: Build new view (Table B)
    
    Note over View: Switch queries to Table B
    Note over V1,View: Decommission V1 and Table A
```

### Lambda vs Kappa

| Aspect | Lambda | Kappa |
|--------|--------|-------|
| **Complexity** | Two codebases | One codebase |
| **Reprocessing** | Batch job | Replay stream |
| **Storage** | Data lake + queue | Queue with retention |
| **Accuracy** | Batch is source of truth | Stream is source of truth |
| **Use case** | Complex historical queries | Event-driven systems |

---

## Choosing Batch vs Stream

### Decision Framework

```mermaid
flowchart TD
    Start[Processing Need] --> Q1{Latency<br/>requirement?}
    
    Q1 -->|"Minutes/hours OK"| Q2{Data volume?}
    Q1 -->|"Seconds/ms needed"| Stream[Stream Processing]
    
    Q2 -->|"Very large (TB+)"| Batch[Batch Processing]
    Q2 -->|"Moderate"| Q3{Need real-time<br/>AND historical?}
    
    Q3 -->|"Yes"| Lambda[Lambda Architecture]
    Q3 -->|"Real-time only"| Kappa[Kappa Architecture]
```

### Comparison

| Aspect | Batch | Stream |
|--------|-------|--------|
| **Latency** | Minutes to hours | Milliseconds to seconds |
| **Throughput** | Very high | Moderate |
| **Data completeness** | Always complete | May have late events |
| **Fault tolerance** | Rerun job | Checkpointing |
| **Complexity** | Lower | Higher (state, time) |
| **Cost efficiency** | Better for large volumes | Better for continuous |

---

## Technology Comparison

### Batch Technologies

| Technology | Description | Best For |
|------------|-------------|----------|
| **Apache Spark** | General-purpose, fast | ML, ETL, SQL |
| **Hadoop MapReduce** | Original big data | Legacy, very large scale |
| **AWS Glue** | Serverless ETL | AWS-native workloads |
| **dbt** | SQL-based transforms | Data warehouse transforms |

### Stream Technologies

| Technology | Description | Best For |
|------------|-------------|----------|
| **Apache Kafka Streams** | Library, Kafka-native | Simple streaming |
| **Apache Flink** | Stateful, exactly-once | Complex event processing |
| **Apache Spark Streaming** | Micro-batch | Spark ecosystem |
| **AWS Kinesis** | Managed streaming | AWS-native |

---

## Interview Scenarios

### Scenario 1: Real-Time Fraud Detection

**Requirements**: Detect fraudulent transactions within seconds

```mermaid
flowchart LR
    subgraph Solution
        S1["Kafka for event ingestion"]
        S2["Flink for pattern detection"]
        S3["Feature store for ML"]
        S4["Alert system"]
    end
```

**Design**:
- Stream processing (Flink) for real-time
- Windowed aggregations (last 1 hour of user activity)
- ML model scoring per transaction
- Sub-second latency requirement

### Scenario 2: Daily Business Reports

**Requirements**: Generate reports from billions of events

```mermaid
flowchart LR
    subgraph Solution
        S1["Data lake storage"]
        S2["Spark batch job"]
        S3["Schedule daily via Airflow"]
        S4["Output to data warehouse"]
    end
```

**Design**:
- Batch processing (Spark) - latency not critical
- Run overnight when resources cheap
- Full historical data available
- Accurate aggregations

### Scenario 3: Real-Time Dashboard + Historical Analytics

**Requirements**: Live metrics AND historical analysis

```mermaid
flowchart TB
    subgraph Solution["Kappa Architecture"]
        Kafka["Kafka (7-day retention)"]
        Flink["Flink Processor"]
        Live["Live Metrics<br/>(Redis)"]
        Historical["Historical<br/>(ClickHouse)"]
    end
    
    Kafka --> Flink
    Flink --> Live
    Flink --> Historical
```

**Design**:
- Single stream processor
- Output to multiple sinks
- Replay from Kafka for reprocessing
- ClickHouse for historical queries

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────────┐
│             BATCH VS STREAM PROCESSING CHEAT SHEET              │
├─────────────────────────────────────────────────────────────────┤
│ BATCH PROCESSING:                                               │
│   • Bounded data, high latency (hours)                          │
│   • Very high throughput, cost efficient                        │
│   • Technologies: Spark, Hadoop, AWS Glue                       │
│   • Use for: ETL, reports, ML training, backfill                │
├─────────────────────────────────────────────────────────────────┤
│ STREAM PROCESSING:                                              │
│   • Unbounded data, low latency (seconds)                       │
│   • Event-by-event, stateful processing                         │
│   • Technologies: Flink, Kafka Streams, Kinesis                 │
│   • Use for: Real-time analytics, alerting, fraud detection     │
├─────────────────────────────────────────────────────────────────┤
│ KEY CONCEPTS:                                                   │
│   • Event Time vs Processing Time                               │
│   • Watermarks (track late events)                              │
│   • Windows (tumbling, sliding, session)                        │
│   • Exactly-once semantics                                      │
├─────────────────────────────────────────────────────────────────┤
│ ARCHITECTURES:                                                  │
│   • Lambda: Batch + Stream (accuracy + speed)                   │
│   • Kappa: Stream only (simpler, replay for reprocess)          │
├─────────────────────────────────────────────────────────────────┤
│ DECISION:                                                       │
│   • Latency critical? → Stream                                  │
│   • Massive historical data? → Batch                            │
│   • Need both? → Lambda or Kappa                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Practice Questions

1. Design a system to detect credit card fraud in real-time. What processing paradigm would you use?
2. How would you handle late-arriving events in a stream processing system?
3. Compare Lambda and Kappa architectures. When would you choose each?
4. Design a real-time leaderboard for an online game with millions of players.
5. How would you reprocess historical data when your stream processing logic changes?
