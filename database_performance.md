
[Back](./README.md)

# 📊 Database Performance Techniques Cheat Sheet

| Technique             | Category    | What It Does                         | When to Use                        | What It Improves         | Trade-offs                      |
| --------------------- | ----------- | ------------------------------------ | ---------------------------------- | ------------------------ | ------------------------------- |
| **Indexing**          | Data access | Adds data structures for fast lookup | Frequent filtering, joins, sorting | Query speed              | Slower writes, storage overhead |
| **Composite Index**   | Data access | Index on multiple columns            | Multi-column filters               | Query efficiency         | Needs correct column order      |
| **Covering Index**    | Data access | Index contains all needed columns    | Avoid table lookup                 | Faster reads             | Larger index size               |
| **Full-text Index**   | Search      | Enables text search                  | Searching text fields              | Search speed             | Limited use cases               |
| **Columnstore Index** | Analytics   | Stores data column-wise              | Aggregations, OLAP                 | Scan & aggregation speed | Slower writes                   |

---

| Technique         | Category  | What It Does                       | When to Use                  | What It Improves         | Trade-offs       |
| ----------------- | --------- | ---------------------------------- | ---------------------------- | ------------------------ | ---------------- |
| **Partitioning**  | Data size | Splits table into chunks           | Large tables, date filtering | Reduces scanned data     | Added complexity |
| **Sharding**      | Scaling   | Splits DB across servers           | Massive data or writes       | Read + write scalability | Complex joins    |
| **Read Replicas** | Scaling   | Copies DB for reads                | High read traffic            | Read throughput          | Replication lag  |
| **Clustering**    | Scaling   | Multiple DB nodes working together | High availability + scale    | Fault tolerance, scaling | Complex setup    |

---

| Technique                       | Category      | What It Does             | When to Use        | What It Improves   | Trade-offs         |
| ------------------------------- | ------------- | ------------------------ | ------------------ | ------------------ | ------------------ |
| **Caching (Redis / Memcached)** | Avoid DB hits | Stores results in memory | Repeated queries   | Massive speed gain | Cache invalidation |
| **Materialized Views**          | Precompute    | Stores query results     | Heavy aggregations | Instant reads      | Needs refresh      |
| **Denormalization**             | Schema design | Reduces joins            | Read-heavy systems | Faster queries     | Data duplication   |

---

| Technique                          | Category        | What It Does             | When to Use      | What It Improves    | Trade-offs         |
| ---------------------------------- | --------------- | ------------------------ | ---------------- | ------------------- | ------------------ |
| **Connection Pooling (PgBouncer)** | Connection mgmt | Reuses DB connections    | High concurrency | Throughput, latency | Pool tuning needed |
| **Query Routing (ProxySQL)**       | Traffic mgmt    | Routes reads/writes      | Using replicas   | Load balancing      | Extra layer        |
| **Parallel Execution**             | Execution       | Runs queries in parallel | Large scans      | Faster processing   | CPU usage          |

---

| Technique            | Category       | What It Does        | When to Use     | What It Improves  | Trade-offs              |
| -------------------- | -------------- | ------------------- | --------------- | ----------------- | ----------------------- |
| **Compression**      | Storage        | Reduces data size   | Large datasets  | I/O speed         | CPU overhead            |
| **Archiving**        | Data lifecycle | Moves old data out  | Historical data | Smaller active DB | Access complexity       |
| **Batch Processing** | Workload       | Processes in chunks | Heavy writes    | Stability         | Latency (not real-time) |

---

| Technique                         | Category    | What It Does           | When to Use          | What It Improves    | Trade-offs      |
| --------------------------------- | ----------- | ---------------------- | -------------------- | ------------------- | --------------- |
| **Search Engine (Elasticsearch)** | Specialized | External search system | Full-text, filtering | Search performance  | Sync complexity |
| **In-Memory DB (Redis)**          | Storage     | Keeps data in RAM      | Ultra-fast access    | Latency             | Cost (RAM)      |
| **Hardware Scaling**              | Infra       | Better CPU/RAM/SSD     | Resource bottlenecks | Overall performance | Cost            |

---

# 🧭 Quick Decision Table

| Problem            | Best Techniques         |
| ------------------ | ----------------------- |
| Slow queries       | Indexing, columnstore   |
| Too many reads     | Replicas, caching       |
| Large tables       | Partitioning, archiving |
| Write bottlenecks  | Sharding, batching      |
| Heavy aggregations | Materialized views      |
| Full-text search   | Elasticsearch           |
| High concurrency   | Connection pooling      |
| Disk I/O slow      | Compression, SSD        |

---

# ✅ Key Takeaway

* **Make queries efficient → Indexing**
* **Reduce data → Partitioning**
* **Handle load → Replicas**
* **Scale system → Sharding**
* **Avoid work → Caching**

---


Here’s a **step-by-step decision flowchart** you can follow whenever you face a database performance issue. Think of it as a guided path from simplest fixes → advanced scaling.

---

# 🧭 Database Performance Decision Flowchart

```
START
  ↓
Is the query slow?
  ↓
YES → Check execution plan (EXPLAIN ANALYZE)
  ↓
Is it scanning too many rows?
  ↓
YES → Add / fix indexes
  ↓
Still slow?
  ↓
YES → Reduce data (filters, LIMIT, pagination)
  ↓
Still slow?
  ↓
YES → Optimize joins / remove unnecessary work
  ↓
Still slow?
  ↓
YES → Use columnstore / materialized view (for aggregation)
  ↓
Still slow?
  ↓
YES → Cache results (Redis)
  ↓
Still slow?
  ↓
YES → Partition large tables
  ↓
Still slow?
  ↓
YES → Consider sharding
  ↓
END
```

---

# 🔁 Parallel Path: System Load Problem

```
START
  ↓
Is database CPU / load high?
  ↓
YES → Are queries already optimized?
  ↓
NO → Fix queries first
  ↓
YES
  ↓
Is workload read-heavy?
  ↓
YES → Add read replicas
  ↓
Still overloaded?
  ↓
YES → Add caching (Redis)
  ↓
Still overloaded?
  ↓
YES → Scale hardware (CPU/RAM/SSD)
  ↓
Still overloaded?
  ↓
YES → Shard database
  ↓
END
```

---

# 📊 Data Size Problem

```
Is table very large?
  ↓
YES
  ↓
Do queries filter by a column (e.g., date)?
  ↓
YES → Use partitioning
  ↓
NO → Consider sharding
```

---

# 🔎 Search-Specific Problem

```
Is query doing text search or complex filtering?
  ↓
YES → Use Elasticsearch
```

---

# ⚡ Simplified Mental Flow (Interview Version)

1. **Check execution plan**
2. **Fix indexes**
3. **Reduce scanned data**
4. **Optimize joins**
5. **Precompute (materialized views / columnstore)**
6. **Cache results**
7. **Scale reads (replicas)**
8. **Split data (partitioning)**
9. **Scale writes (sharding)**

---

# 🧠 Key Insight (Most Important Rule)

👉 **Never jump to sharding or replicas first**
Always go in this order:

**Query → Index → Data → Cache → Scale**

---

# ✅ Real-World Example Walkthrough

**Problem:** Dashboard query is slow

Flow:

1. EXPLAIN → full table scan
2. Add index ✅
3. Still slow → aggregation heavy
4. Add materialized view ✅
5. Traffic grows → DB CPU high
6. Add read replicas ✅
7. Still heavy → add Redis caching ✅

---


[Back](./README.md)
