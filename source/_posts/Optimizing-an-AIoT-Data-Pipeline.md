---
title: "Optimizing an AIoT Data Pipeline: From 20 Seconds to 50 Milliseconds"
date: 2025-11-15 10:00:00
tags:
  - Backend
  - Node.js
  - PostgreSQL
  - RabbitMQ
  - Performance Optimization
  - System Design
  - Docker
  - MQTT
categories:
  - Backend Engineering
---

### Building Award-Winning Healthcare Technology

During my internship at the Institute for Information Industry (III/資策會), I had the privilege of contributing to the **"iMat for Smart Care"** project — a 5G and AI-enabled Smart Mattress Care System. This wasn't just another internship project; it was a system that would eventually be **deployed in Long-Term Care Centers, benefiting over 5,400 individuals**.

Even more rewarding, the project was later recognized with the prestigious **[R&D 100 Award](https://www.iii.org.tw/zh-TW/news/newsroom/iii-news/439d152b-0597-44d4-858c-145affb8f140)** (one of the world's top 100 technology R&D awards) along with a **Silver Medal in the Special Contribution Award for Corporate Social Responsibility**.

But when I joined the project, the system had a critical problem: **performance**.

---

### The Challenge: A Struggling Real-Time System

The iMat system collected real-time physiological data—respiration rate, body movements, and bed occupancy status—from smart mattress sensors. This data was processed, stored, and fed into AI models to predict user behavior and trigger fall prevention alerts.

The stakes were high. In a Long-Term Care environment, **delays in fall detection could have real consequences**. Yet when I joined, a single data pipeline execution was taking **over 20 seconds**.

My mission was clear: diagnose the bottlenecks and engineer a solution that could operate in real-time.

This article documents my journey of transforming that struggling system into one that processes data in **under 50 milliseconds** — a **400x performance improvement**.

---

### The Technical Stack

| Layer | Technology |
|-------|------------|
| **Backend** | Node.js, Express.js |
| **Messaging** | MQTT (sensor communication), RabbitMQ (async processing) |
| **Database** | PostgreSQL with TimescaleDB (time-series optimization) |
| **Containerization** | Docker, Docker Compose |
| **Process Management** | PM2 |
| **Deployment** | Linux servers, Shell scripts |

---

### System Architecture Overview

```
Smart Mattress Sensors (IoT Devices)
        │
        │ MQTT (dozens of readings/second)
        ▼
┌─────────────────────┐
│   MQTT Broker       │  (Mosquitto with TLS)
│   (Local Server)    │
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│   Node.js Backend   │  ← Performance bottleneck
│   (Data Pipeline)   │
└─────────────────────┘
        │
        ├──────────────────────┐
        ▼                      ▼
┌─────────────────┐    ┌─────────────────┐
│   PostgreSQL    │    │   AI Models     │
│   (TimescaleDB) │    │ (Fall Detection)│
└─────────────────┘    └─────────────────┘
        │
        ▼
┌─────────────────────┐
│   Alert System      │  → Caregiver notifications
└─────────────────────┘
```

---

### Phase 1: Diagnosing the Root Cause

Using PM2's monitoring capabilities along with strategic logging, I traced the data flow and identified that the primary bottleneck was in the **database layer**.

The original architecture stored all sensor data in a single table. As months of physiological data accumulated, this table grew to millions of rows. Every query—whether for real-time monitoring or historical analysis—was scanning the same massive table.

#### The "Aha" Moment

I realized that optimizing SQL queries was like putting a band-aid on a broken leg. The real problem was that we were treating **hot data** (recent, frequently accessed) and **cold data** (historical, rarely accessed) identically. No matter how clever our indexing strategy, querying a 10-million-row table would always be slower than querying a 10,000-row table.

---

### Phase 2: Hot/Cold Data Tiering

The solution was to fundamentally redesign the data architecture:

```
┌─────────────────────────────────────────────────────┐
│              Incoming Sensor Data                    │
└─────────────────────┬───────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────┐
│           sensor_data_hot (TimescaleDB)             │
│         (Last 24 hours only — ~50K rows)            │
│         Blazing fast queries for real-time alerts   │
└─────────────────────┬───────────────────────────────┘
                      │ Automated Cron Job (Hourly)
                      ▼
┌─────────────────────────────────────────────────────┐
│         sensor_data_historical (TimescaleDB)        │
│         (Compressed, partitioned by time)           │
│         Optimized for batch analytics               │
└─────────────────────────────────────────────────────┘
```

#### Implementation

```javascript
const cron = require('node-cron');
const { Pool } = require('pg');

// Run every hour
cron.schedule('0 * * * *', async () => {
    const client = await pool.connect();
    try {
        await client.query('BEGIN');
        
        const result = await client.query(`
            WITH moved_rows AS (
                DELETE FROM sensor_data_hot
                WHERE timestamp < NOW() - INTERVAL '24 hours'
                RETURNING *
            )
            INSERT INTO sensor_data_historical
            SELECT * FROM moved_rows
        `);
        
        console.log(`[Migration] Moved ${result.rowCount} rows to historical`);
        await client.query('COMMIT');
    } catch (error) {
        await client.query('ROLLBACK');
        console.error('[Migration] Failed:', error);
    } finally {
        client.release();
    }
});
```

#### Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Hot table size | 10M+ rows | ~50K rows | 99.5% reduction |
| Pipeline execution | 20+ seconds | 40-50ms | **400x faster** |
| P99 latency | 45 seconds | 150ms | **300x faster** |

---

### Phase 3: Asynchronous Processing with RabbitMQ

With the database optimized, I addressed the coupling between data ingestion and AI model inference. The original synchronous design meant slow AI predictions would block the entire pipeline.

I introduced RabbitMQ to decouple these components:

```
MQTT Broker → Node.js Preprocessor → RabbitMQ → AI Worker → Alerts
```

#### Message Persistence for Reliability

In a healthcare system, losing data is unacceptable:

```javascript
// Durable queue survives broker restart
channel.assertQueue('sensor_analysis', { durable: true });

// Persistent messages survive broker restart
channel.sendToQueue('sensor_analysis', 
    Buffer.from(JSON.stringify(data)),
    { persistent: true }
);
```

#### Consumer Acknowledgments

Workers only acknowledge after successful processing:

```javascript
channel.consume('sensor_analysis', async (msg) => {
    try {
        const data = JSON.parse(msg.content.toString());
        await runFallDetection(data);
        channel.ack(msg);
    } catch (error) {
        channel.nack(msg, false, true);  // Requeue for retry
    }
}, { noAck: false });
```

---

### Phase 4: Containerization with Docker

To streamline deployment across environments, I containerized the system:

```yaml
# docker-compose.yml
version: '3.8'
services:
  mqtt-broker:
    image: eclipse-mosquitto:2
    ports:
      - "8883:8883"
      
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      
  data-pipeline:
    build: ./pipeline
    depends_on:
      - mqtt-broker
      - rabbitmq
```

This enabled one-command deployments and consistent environments across dev/staging/production.

---

### Phase 5: Production Monitoring with PM2

PM2 was invaluable for production operations:

```javascript
// ecosystem.config.js
module.exports = {
    apps: [{
        name: 'data-pipeline',
        script: './index.js',
        instances: 'max',
        exec_mode: 'cluster',
        max_memory_restart: '500M',
        exp_backoff_restart_delay: 100
    }]
};
```

Key capabilities I relied on:
- **Real-time log streaming**: `pm2 logs --lines 500`
- **Process monitoring**: `pm2 monit`
- **Zero-downtime deploys**: `pm2 reload data-pipeline`

---

### Real-World Impact

The optimizations I implemented weren't just technical achievements—they had **real-world impact**:

- **5,400+ individuals** in Long-Term Care Centers benefited from the fall prevention system
- The project contributed to the **R&D 100 Award** recognition
- The system achieved the reliability required for 24/7 healthcare monitoring

---

### Key Takeaways

1. **Measure before optimizing**: Without profiling, I might have optimized the wrong component entirely.

2. **Architecture trumps micro-optimization**: No SQL tuning could match what a data tiering strategy achieved.

3. **Design for data lifecycle**: Hot and cold data have fundamentally different access patterns.

4. **Message queues add resilience**: RabbitMQ made the system fault-tolerant and scalable.

5. **Containerization simplifies operations**: Docker eliminated deployment inconsistencies.

---

### Conclusion

Transforming a 20-second pipeline into a 50-millisecond one required:

- **Deep diagnosis** to find the true bottlenecks
- **Architectural thinking** to redesign the data layer
- **Reliability engineering** to ensure no data loss
- **Operational excellence** for production monitoring

These experiences shaped my approach to backend engineering: always understand the problem before jumping to solutions, and never underestimate the power of good architecture.

---

*Building reliable, high-performance systems is what I love most about backend engineering. Feel free to connect with me on [LinkedIn](https://linkedin.com/in/yu-wei-chang-6714a91a4) to discuss similar challenges.*
