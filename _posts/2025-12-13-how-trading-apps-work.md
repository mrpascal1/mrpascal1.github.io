---
layout: post
author: Shahid Raza
title: How Trading Apps Handle 1 Million+ Concurrent WebSocket Connections
tags: Engineering Backend Concurrency Trading
---

*(Inspired by Zerodha, Upstox, Groww, Robinhood, TD Ameritrade)*
<!--more--> 

# **Introduction**

Modern trading platforms like **Zerodha**, **Upstox**, **Groww**, **Robinhood**, etc., have one major technical challenge:

> **How do you stream real-time prices, order book depth, and trade ticks to millions of users simultaneously — with sub-100ms latency and zero downtime?**

This is one of the toughest engineering problems in FinTech because market data arrives fast, unpredictably, and at massive scale.

This blog breaks down **how real trading apps handle 1M+ concurrent WebSocket connections** while staying real-time, consistent, and cost-efficient.

---

# Understanding the Scale

### A typical day in Indian markets:

* **3–5 million simultaneous retail traders online**
* **Millions of WebSocket connections** to broker apps
* **Price updates 5–50 times per second**
* **NSE & BSE depth updates every 10–50ms**
* **10,000+ symbols streaming concurrently**

This is real **high-throughput + low-latency** engineering.

---

# **High-Level Architecture Overview**

A trading app’s real-time architecture looks like this:

![Basic-Trading-Architecture](https://raw.githubusercontent.com/mrpascal1/mrpascal1.github.io/master/imgs/122025/basic_arch_trade.png)

Let’s break it down.

---

# 1. **Market Data Ingestion Layer**

Trading apps receive raw market feeds from exchanges:

* **TBT (Tick-by-Tick)** feed
* **Depth feed**
* **Trade feed**
* **Index feed**
* **Option chain feed**

These come in **binary formats** via leased lines.

To process them, firms use:

* **C++** or **Rust** for ultra-low-latency ingestion
* **RDMA** or **zero-copy sockets**
* **Folly, Boost ASIO, or custom TCP frameworks**

Latency budget here is around **5ms**.

---

# 2. **Tick Normalizer + Broadcaster**

Since exchanges send raw binary messages, brokers must:

* Convert them to JSON/Protobuf
* Merge depth + trade + LTP updates
* Remove redundant ticks
* Apply throttling (e.g., send 10 updates/sec per symbol)
* Distribute data to Kafka/Pulsar/Redis

Tools used:

* **Kafka** (high throughput)
* **Redis Streams** (low latency)
* **NATS** (ultra-fast messaging)
* **Aerospike** (if storing short-term tick data)

Throughput: **millions of ticks per second**

---

# 3. **WebSocket Delivery Architecture**

This is the core part.

### Why WebSockets?

* Bidirectional communication
* Low latency
* Persistent connections
* Lightweight frames
* Perfect for real-time apps

More scalable than REST, less heavy than MQTT.

---

# 4. **WebSocket Cluster (Horizontal Scaling)**

To handle **1M+ concurrent connections**, brokers deploy **large clusters**:

* **100–500 WebSocket servers**
* Each handling **10,000–40,000 connections**
* Load balancers route clients to least-loaded nodes

### Typical technology stack:

| Layer            | Tools                                                            |
| ---------------- | ---------------------------------------------------------------- |
| Load Balancer    | **NGINX**, **Envoy**, **HAProxy**, **AWS ALB/NLB**               |
| WebSocket Server | **Go**, **Node.js**, **Elixir/Phoenix**, **Java Netty**, **C++** |
| Pub/Sub          | **Redis**, **Kafka**, **NATS**, **Pulsar**                       |
| Routing          | **Consistent Hashing**, **Service Mesh**                         |

Companies like Zerodha use **Go** and **Redis**, while Upstox and Robinhood heavily use **Kafka**.

---

# 5. **Connection Distribution Strategy**

### To avoid overloading a single node:

#### **Technique 1: Consistent Hashing**

Users subscribe to symbols (e.g., NIFTY, RELIANCE).
Same symbol streams are always pushed from the same few nodes.

#### **Technique 2: Sharded Streams**

Symbols are partitioned into shards:

```
Shard 1 → Node A
Shard 2 → Node B
Shard 3 → Node C
```

#### **Technique 3: Sticky Sessions**

Load balancer keeps the same client on the same WebSocket server.

---

# 6. **Backpressure & Throttling**

If a user has a slow network, you cannot push 50 ticks/second.

Trading systems use:

### Drop old messages (send only the latest)

Because only latest price matters.

### Throttle updates

Send only 5–10 messages/second.

### Avoid queue buildup per-client

If queue grows, disconnect the client (industry standard).

---

# 7. **In-Memory Caching for Ticks**

To achieve **sub-10ms fan-out latency**, brokers use in-memory stores:

* **Redis**
* **Hazelcast**
* **Memcached**
* **Aerospike**

They maintain:

* Latest price
* OHLC
* Volume
* Order book depth
* Index metadata

All updated in **real-time** via message streams.

---

# 8. **High Availability & Fault Tolerance**

Trading platforms use:

### Multi-zone WebSocket clusters

One AZ failing should not drop 500,000 users.

### Hot standby nodes

Auto-join cluster when load increases.

### Message replay using Kafka

If WebSocket node crashes, it reconnects to Kafka and resumes streaming instantly.

### Health-based routing

Faulty node = removed by LB.

---

# 9. **Handling Market Open Explosion (The 9:15 AM Spike)**

At 9:15 AM, traffic increases **20x within 5 seconds**.

Techniques:

* Pre-warm WebSocket nodes
* Autoscale via CPU + connection count
* Preload symbols
* Throttle depth updates
* Use edge caching/CDN for static Web data

---

# **10. Client-Side Architecture (Mobile/Web)**

### On mobile:

* WebSocket reconnect strategy
* Local price cache
* Delta compression
* Adaptive throttling
* Fallback to REST if socket drops

### On web:

* Shared worker for websockets
* Backpressure logic
* Efficient DOM diffing

---

# 11. Cost Optimization at Scale

Running 200 WebSocket servers costs a lot.

Brokers reduce cost using:

* Bare-metal servers (instead of cloud)
* Go or Elixir (less CPU/memory per connection)
* Zero-copy network IO
* Dedicated NICs
* Low GC pressure runtimes

Zerodha and Upstox famously use **co-located servers near exchanges** for speed + cost.

---

# **Conclusion**

Handling **1 million+ concurrent WebSocket connections** is one of the hardest FinTech engineering problems.

It requires:

* Ultra-fast ingestion
* Real-time stream processing
* Horizontally scalable WebSocket clusters
* Smart sharding
* Aggressive throttling
* Failover-ready design
* Low-latency, in-memory pipelines
* Network optimizations at every layer

This is why only a handful of trading apps in India (Zerodha, Upstox, Groww) have mastered this at scale.

## Key Citations
- [Scaling Node.js to 1 Million Concurrent WebSocket Clients](https://medium.com/@connect.hashblock/scaling-node-js-to-1-million-concurrent-websocket-clients-with-horizontal-sharding-51c20091088e)
- [How to Scale WebSockets for High-Concurrency Systems](https://ably.com/topic/the-challenge-of-scaling-websockets)
- [Stock Market App Development: Real-Time WebSocket Dashboards](https://openwebsolutions.in/blog/stock-market-app-development-real-time-websocket-dashboards-2025/)
- [Designing Scalable Trading Apps with Real-Time Market Data APIs](https://finage.co.uk/blog/designing-scalable-trading-apps-with-realtime-market-data-apis--684b0c01ab6efd9ba320f588)
- [Challenges of Scaling WebSockets](https://dev.to/ably/challenges-of-scaling-websockets-3493)
- [Scaling WebSockets to Millions](https://dyte.io/blog/scaling-websockets-to-millions/)
- [WebSockets at Scale - Production Architecture](https://websocket.org/guides/websockets-at-scale/)
- [Handling 1M WebSocket Connections in Spring Boot](https://blog.stackademic.com/handling-1-million-websocket-connections-in-spring-boot-without-losing-my-mind-f69ee013cbb9)
- [Best Way to Handle One Million Concurrent Socket Connections](https://stackoverflow.com/questions/49398241/best-way-to-handle-one-million-concurrent-socket-connections)
- [How to Handle 1M WebSocket Connections (Nginx/HAProxy)](https://serverfault.com/questions/725224/how-to-handle-1m-websocket-connections-nginx-haproxy-amazon-google)