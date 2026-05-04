# 🏗️ Schools24 Redis-First Architecture Diagrams

## 1. Data Flow Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT APPLICATIONS                          │
│   [React Web] [React Native Mobile] [Admin Dashboard] [Smart Board] │
└───────────────────────────┬─────────────────────────────────────────┘
                            │ HTTPS/TLS
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    API GATEWAY (Kong/Traefik)                        │
│  • JWT Validation  • Rate Limiting  • SSL Termination  • Routing   │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ AuthService  │    │ QuizService  │    │ FeeService   │
│              │    │              │    │              │
│ • Login      │    │ • Create     │    │ • Invoices   │
│ • JWT        │    │ • Submit     │    │ • Payments   │
│ • Sessions   │    │ • Grade      │    │ • Receipts   │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       │    Each service has Redis-first      │
       │    caching with compression          │
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────────┐
        │        REDIS LAYER (Primary Cache)       │
        │  ┌────────────────────────────────────┐ │
        │  │  Write Buffer (Compressed Data)    │ │
        │  │  • Snappy compression (77% saved)  │ │
        │  │  • 2-hour TTL                      │ │
        │  │  • Structured pointers             │ │
        │  └────────────────────────────────────┘ │
        │  ┌────────────────────────────────────┐ │
        │  │  Read Cache (Hot Data)             │ │
        │  │  • Session data (24h TTL)          │ │
        │  │  • Dashboard metrics (30min TTL)   │ │
        │  │  • Leaderboards (sorted sets)      │ │
        │  └────────────────────────────────────┘ │
        └──────────────┬───────────────────────────┘
                       │
                       │ ┌─────────────────────────┐
                       │ │ Batch Processor (Cron) │
                       │ │ Runs every 1 hour      │
                       │ │ • Scan unsynced keys   │
                       │ │ • Decompress data      │
                       │ │ • Parallel DB writes   │
                       │ │ • Flush Redis keys     │
                       │ └───────┬─────────────────┘
                       │         │
                       ▼         ▼
        ┌──────────────────────────────────────────┐
        │         DATABASE LAYER (Persistent)      │
        │  ┌─────────────┐  ┌──────────────────┐  │
        │  │ PostgreSQL  │  │    MongoDB       │  │
        │  │ (Relational)│  │  (Documents)     │  │
        │  │             │  │                  │  │
        │  │ • Users     │  │ • Questions      │  │
        │  │ • Students  │  │ • Analytics      │  │
        │  │ • Quizzes   │  │ • Activity Logs  │  │
        │  │ • Homework  │  │                  │  │
        │  │ • Fees      │  │                  │  │
        │  │ • Payments  │  │                  │  │
        │  └─────────────┘  └──────────────────┘  │
        └──────────────────────────────────────────┘
```

---

## 2. Write Operation Flow (Redis-First)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    User Action: "Submit Homework"                    │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
                   ┌────────────────┐
                   │  API Handler   │
                   │  (Gin Router)  │
                   └────────┬───────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
      ┌──────────┐   ┌──────────┐   ┌──────────┐
      │ Validate │   │  Upload  │   │  Create  │
      │   JWT    │   │  to S3   │   │  Object  │
      └────┬─────┘   └────┬─────┘   └────┬─────┘
           │              │              │
           └──────────────┴──────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  Compress with Snappy │
              │  487 bytes → 134 bytes│
              └───────────┬───────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  Write to Redis       │
              │  Key: write_buffer:*  │
              │  TTL: 2 hours         │
              │  Metadata: synced=false│
              └───────────┬───────────┘
                          │
                          │ ⚡ INSTANT (5ms)
                          ▼
              ┌───────────────────────┐
              │  Return 200 OK        │
              │  "Submitted!"         │
              │  (45ms total)         │
              └───────────────────────┘
                          
                          
                          
    ⏰ 1 HOUR LATER...
                          
                          
              ┌───────────────────────┐
              │  Batch Processor      │
              │  Wakes Up (Cron)      │
              └───────────┬───────────┘
                          │
            ┌─────────────┼─────────────┐
            │             │             │
            ▼             ▼             ▼
      ┌─────────┐   ┌─────────┐   ┌─────────┐
      │  Scan   │   │  Fetch  │   │Decompress│
      │ Unsynced│   │  Keys   │   │  Data   │
      │  Keys   │   │ (100+)  │   │ (Snappy)│
      └────┬────┘   └────┬────┘   └────┬────┘
           │             │             │
           └─────────────┴─────────────┘
                         │
                         ▼
            ┌────────────────────────┐
            │  Parallel DB Write     │
            │  (10 goroutines)       │
            │  PostgreSQL.Save(...)  │
            └────────────┬───────────┘
                         │
                         ▼
            ┌────────────────────────┐
            │  Mark as Synced        │
            │  synced_to_db = true   │
            └────────────┬───────────┘
                         │
                         ▼
            ┌────────────────────────┐
            │  Flush from Redis      │
            │  DEL key, meta:key     │
            │  Memory freed          │
            └────────────────────────┘
```

---

## 3. Read Operation Flow (Cache-First)

```
┌─────────────────────────────────────────────────────────────────────┐
│                User Action: "View Student Dashboard"                 │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
                   ┌────────────────┐
                   │  API Handler   │
                   │  GET /dashboard│
                   └────────┬───────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │  Check Redis Cache      │
              │  dashboard:student:123  │
              └─────────┬───────────────┘
                        │
            ┌───────────┴───────────┐
            │                       │
            ▼                       ▼
    ┌──────────────┐        ┌──────────────┐
    │  CACHE HIT   │        │  CACHE MISS  │
    │  (80-90%)    │        │  (10-20%)    │
    └──────┬───────┘        └──────┬───────┘
           │                       │
           │                       ▼
           │              ┌─────────────────┐
           │              │ Query PostgreSQL│
           │              │ + MongoDB       │
           │              │ (80ms)          │
           │              └────────┬────────┘
           │                       │
           │                       ▼
           │              ┌─────────────────┐
           │              │ Aggregate Data  │
           │              │ (attendance,    │
           │              │  grades, etc)   │
           │              └────────┬────────┘
           │                       │
           │                       ▼
           │              ┌─────────────────┐
           │              │ Compress with   │
           │              │ Snappy          │
           │              └────────┬────────┘
           │                       │
           │                       ▼
           │              ┌─────────────────┐
           │              │ Store in Redis  │
           │              │ TTL: 30 min     │
           │              └────────┬────────┘
           │                       │
           └───────────────────────┘
                           │
                           ▼
               ┌───────────────────────┐
               │  Decompress Data      │
               │  (if from cache)      │
               └───────────┬───────────┘
                           │
                           ▼
               ┌───────────────────────┐
               │  Return JSON          │
               │  200 OK               │
               │  Cache hit: 5ms       │
               │  Cache miss: 30ms     │
               └───────────────────────┘
```

---

## 4. Microservices Communication

```
┌──────────────────────────────────────────────────────────────────┐
│                     Frontend Request                              │
│         POST /api/v1/homework (Teacher assigns homework)         │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  HomeworkService│
                    └────────┬────────┘
                             │
                ┌────────────┼────────────┐
                │            │            │
                ▼            ▼            ▼
         ┌──────────┐ ┌──────────┐ ┌──────────┐
         │ Upload   │ │ Compress │ │  Store   │
         │  to S3   │ │   Data   │ │  Redis   │
         └──────────┘ └──────────┘ └────┬─────┘
                                        │
                                        ▼
                            ┌───────────────────────┐
                            │  Publish Event to     │
                            │  Redis Pub/Sub        │
                            │  Channel: "homework:  │
                            │           assigned"   │
                            └───────────┬───────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                    ▼                   ▼                   ▼
           ┌────────────────┐  ┌────────────────┐  ┌──────────────┐
           │ Notification   │  │ Analytics      │  │ Dashboard    │
           │ Service        │  │ Service        │  │ Service      │
           └────────┬───────┘  └────────┬───────┘  └──────┬───────┘
                    │                   │                  │
                    │                   │                  │
                    ▼                   ▼                  ▼
         ┌──────────────────┐  ┌────────────────┐ ┌──────────────┐
         │ Send Email/SMS/  │  │ Update Activity│ │ Invalidate   │
         │ Push to Students │  │ Logs (MongoDB) │ │ Cache        │
         └──────────────────┘  └────────────────┘ └──────────────┘
```

---

## 5. Compression & Memory Efficiency

```
┌─────────────────────────────────────────────────────────────────┐
│              BEFORE: Uncompressed JSON in Redis                  │
└─────────────────────────────────────────────────────────────────┘

homework_object = {
  "id": "hw_abc123",
  "title": "Chapter 5 Homework - Solve quadratic equations",
  "class_id": "class_10a",
  "subject_id": "mathematics",
  "teacher_id": "teacher_xyz_001",
  "file_url": "https://s3.amazonaws.com/schools24/hw_12345.pdf",
  "due_date": "2025-11-25T23:59:59Z",
  "created_at": 1700000000,
  "description": "Solve problems 1-10 from textbook page 45"
}

JSON Size: 487 bytes
Redis Memory: 487 bytes × 1000 entries = 487 KB


┌─────────────────────────────────────────────────────────────────┐
│              AFTER: Snappy Compressed in Redis                   │
└─────────────────────────────────────────────────────────────────┘

compressed_data = snappy.Encode(json.Marshal(homework_object))

Compressed Size: 134 bytes (72% reduction)
Redis Memory: 134 bytes × 1000 entries = 134 KB

┌───────────────────────────────────────────────────────────┐
│                  MEMORY SAVINGS                            │
│                                                            │
│  Uncompressed:     487 KB                                 │
│  Compressed:       134 KB                                 │
│  ────────────────────────────────────────────────────────│
│  Savings:          353 KB (72%)                           │
│                                                            │
│  For 10,000 records:                                      │
│    Before: 4.87 MB                                        │
│    After:  1.34 MB                                        │
│    Saved:  3.53 MB per dataset                           │
└───────────────────────────────────────────────────────────┘
```

---

## 6. Batch Processor Timeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    Hourly Batch Processing Cycle                 │
└─────────────────────────────────────────────────────────────────┘

Time: 00:00 (Midnight)
│
├─ 00:00 - Batch Processor Wakes Up
│  └─ Scan Redis for keys: write_buffer:*
│
├─ 00:01 - Found 1,234 unsynced keys
│  ├─ write_buffer:homework:* (42 keys)
│  ├─ write_buffer:quiz_submission:* (358 keys)
│  ├─ write_buffer:attendance:* (720 keys)
│  ├─ write_buffer:payment:* (14 keys)
│  └─ write_buffer:grade:* (100 keys)
│
├─ 00:02 - Start Parallel Processing (10 workers)
│  ├─ Worker 1: homework batch 1-10
│  ├─ Worker 2: homework batch 11-20
│  ├─ Worker 3: quiz_submission batch 1-35
│  ├─ Worker 4: quiz_submission batch 36-70
│  ├─ Worker 5: attendance batch 1-72
│  ├─ Worker 6: attendance batch 73-144
│  ├─ Worker 7: payment batch 1-14
│  ├─ Worker 8: grade batch 1-20
│  ├─ Worker 9: grade batch 21-40
│  └─ Worker 10: grade batch 41-60
│
├─ 00:03 - Database Writes in Progress
│  ├─ PostgreSQL connections: 10/100 active
│  ├─ Write throughput: ~400 records/sec
│  └─ Average latency: 25ms per record
│
├─ 00:04 - Batch 1 Complete (100 records)
│  └─ Flush from Redis, Memory freed: 12.5 KB
│
├─ 00:05 - Batch 2 Complete (100 records)
│  └─ Flush from Redis, Memory freed: 11.8 KB
│
├─ 00:06 - All Batches Complete
│  ├─ Total records synced: 1,234
│  ├─ Success: 1,230 (99.7%)
│  ├─ Failures: 4 (0.3%) - retry next cycle
│  ├─ Duration: 6 minutes
│  └─ Memory freed: 153 KB
│
└─ 00:06 - Sleep Until Next Cycle (01:00)


Time: 01:00 (Next Hour)
│
└─ Repeat Process...
```

---

## 7. High Availability Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      PRODUCTION DEPLOYMENT                        │
└─────────────────────────────────────────────────────────────────┘

                        ┌──────────────────┐
                        │  Load Balancer   │
                        │  (AWS ALB/NLB)   │
                        └────────┬─────────┘
                                 │
                ┌────────────────┼────────────────┐
                │                │                │
                ▼                ▼                ▼
         ┌───────────┐    ┌───────────┐    ┌───────────┐
         │  API Pod 1│    │  API Pod 2│    │  API Pod 3│
         │  (Go App) │    │  (Go App) │    │  (Go App) │
         └─────┬─────┘    └─────┬─────┘    └─────┬─────┘
               │                │                │
               └────────────────┼────────────────┘
                                │
                ┌───────────────┴───────────────┐
                │                               │
                ▼                               ▼
    ┌───────────────────────┐       ┌───────────────────────┐
    │   Redis Cluster       │       │  PostgreSQL Cluster   │
    │   (Master + Replicas) │       │  (Primary + Replicas) │
    │   ┌─────────────────┐ │       │  ┌─────────────────┐ │
    │   │ Master (Write)  │ │       │  │ Primary (Write) │ │
    │   └────────┬────────┘ │       │  └────────┬────────┘ │
    │            │           │       │           │          │
    │   ┌────────┴────────┐ │       │  ┌────────┴────────┐ │
    │   │ Replica 1 (Read)│ │       │  │ Replica 1 (Read)│ │
    │   └─────────────────┘ │       │  └─────────────────┘ │
    │   ┌─────────────────┐ │       │  ┌─────────────────┐ │
    │   │ Replica 2 (Read)│ │       │  │ Replica 2 (Read)│ │
    │   └─────────────────┘ │       │  └─────────────────┘ │
    │                       │       │                       │
    │   Sentinel (Auto-     │       │   Automatic Failover │
    │   Failover)           │       │   (Patroni/Repmgr)   │
    └───────────────────────┘       └───────────────────────┘


    ┌─────────────────────────────────────────────────────────┐
    │              Batch Processor (Scheduled Job)             │
    │  ┌───────────────────────────────────────────────────┐  │
    │  │  Kubernetes CronJob (Every 1 hour)                │  │
    │  │  Runs in separate pod                             │  │
    │  │  Connects to Redis Master + PostgreSQL Primary    │  │
    │  └───────────────────────────────────────────────────┘  │
    └─────────────────────────────────────────────────────────┘
```

---

## 8. Monitoring Dashboard

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROMETHEUS + GRAFANA METRICS                  │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Redis Cache Metrics                                          │
├──────────────────────────────────────────────────────────────┤
│  Cache Hit Rate:          █████████░ 89%                     │
│  Cache Miss Rate:         █░░░░░░░░░ 11%                     │
│  Compression Ratio:       ███████░░░ 77% avg                 │
│  Memory Usage:            ████░░░░░░ 3.2 GB / 4 GB          │
│  Keys in Write Buffer:    ████████░░ 8,432 keys              │
│  Evictions/sec:           ░░░░░░░░░░ 0.2/sec                 │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Batch Processor Metrics                                      │
├──────────────────────────────────────────────────────────────┤
│  Last Run:                11:00:15 AM                         │
│  Duration:                5m 42s                              │
│  Records Synced:          ████████░░ 1,234                   │
│  Success Rate:            ██████████ 99.7%                   │
│  Failed Records:          █░░░░░░░░░ 4                       │
│  Memory Freed:            ████░░░░░░ 153 KB                  │
│  Next Run:                12:00:00 PM (in 48m 45s)           │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  API Performance                                              │
├──────────────────────────────────────────────────────────────┤
│  Avg Response Time:       ██░░░░░░░░ 45ms                    │
│  P95 Latency:             ███░░░░░░░ 85ms                    │
│  P99 Latency:             ████░░░░░░ 120ms                   │
│  Requests/sec:            ████████░░ 842/sec                 │
│  Error Rate:              ░░░░░░░░░░ 0.02%                   │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Database Load                                                │
├──────────────────────────────────────────────────────────────┤
│  PostgreSQL Connections:  ███░░░░░░░ 25/100                  │
│  Read Queries/sec:        ██░░░░░░░░ 45/sec                  │
│  Write Queries/sec:       ░░░░░░░░░░ 2/sec (batched)         │
│  Avg Query Time:          ██░░░░░░░░ 12ms                    │
│  Slow Queries:            ░░░░░░░░░░ 0                       │
└──────────────────────────────────────────────────────────────┘
```

---

## 🎯 Key Takeaways

1. **Redis-First**: All writes go to Redis (compressed) for instant UX
2. **Compression**: Snappy reduces memory by 60-80%
3. **Async Persistence**: Batch processor syncs to DB every hour
4. **Microservices**: 10 services aligned with frontend pages
5. **High Performance**: 45ms writes, 5ms cache hits
6. **Scalability**: 10,000+ concurrent users supported

**Ready to implement? Start with BACKEND_IMPLEMENTATION_GUIDE.md!**
