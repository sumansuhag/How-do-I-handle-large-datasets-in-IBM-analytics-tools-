# High-Performance Analytics Optimization Guide

> Proven strategies for optimizing IBM analytics tools and databases to handle large-scale datasets efficiently

---

## Overview

This guide provides battle-tested optimization techniques for IBM analytics platforms including Cognos Analytics, SPSS Modeler, watsonx.data, and Db2. Learn how to dramatically reduce query execution times, minimize resource usage, and handle large datasets efficiently through database tuning, query optimization, and hardware configuration.


- Reduce query execution times by 50-80%
- Handle datasets 10x larger with same hardware
- Lower infrastructure costs through efficient resource usage
- Achieve over 90% cache hit ratios on critical workloads
- Optimize end-to-end from database to analytics layer

---

## Table of Contents

1. [Database Tuning (Db2)](#database-tuning-db2)
2. [Query Optimization](#query-optimization)
3. [Tool-Specific Best Practices](#tool-specific-best-practices)
4. [Hardware & Monitoring](#hardware--monitoring)
5. [Quick Reference](#quick-reference)

---

## Database Tuning (Db2)

### Data Partitioning Strategies

Divide large tables into manageable chunks for faster queries:

```sql
-- Range-based partitioning (by date)
CREATE TABLE sales_data (
    sale_id INT,
    sale_date DATE,
    amount DECIMAL(10,2)
)
PARTITION BY RANGE (sale_date)
(
    PARTITION q1_2024 STARTING '2024-01-01' ENDING '2024-03-31',
    PARTITION q2_2024 STARTING '2024-04-01' ENDING '2024-06-30',
    PARTITION q3_2024 STARTING '2024-07-01' ENDING '2024-09-30',
    PARTITION q4_2024 STARTING '2024-10-01' ENDING '2024-12-31'
);

-- Hash-based partitioning (for even distribution)
CREATE TABLE customer_data (
    customer_id INT,
    name VARCHAR(100)
)
PARTITION BY HASH (customer_id)
PARTITIONS 16;
```

**Benefits:**
- Query only relevant partitions (partition elimination)
- Parallel processing across partitions
- Easier maintenance (drop old partitions vs. DELETE)

---

### Buffer Pool Optimization

Target: Over 90% hit ratio for optimal performance

```sql
-- Monitor buffer pool performance
SELECT 
    BP_NAME,
    POOL_DATA_L_READS + POOL_INDEX_L_READS AS LOGICAL_READS,
    POOL_DATA_P_READS + POOL_INDEX_P_READS AS PHYSICAL_READS,
    DECIMAL(
        (1 - FLOAT(POOL_DATA_P_READS + POOL_INDEX_P_READS) / 
        FLOAT(POOL_DATA_L_READS + POOL_INDEX_L_READS)) * 100, 
        5, 2
    ) AS HIT_RATIO
FROM SYSIBMADM.BP_HITRATIO
WHERE BP_NAME = 'IBMDEFAULTBP';

-- Increase buffer pool size
ALTER BUFFERPOOL IBMDEFAULTBP SIZE 50000 AUTOMATIC;
```

**Optimization Targets:**

| Metric | Poor | Good | Excellent |
|--------|------|------|-----------|
| Buffer Pool Hit Ratio | Less than 80% | 85-90% | Greater than 90% |
| Package Cache Hit Ratio | Less than 75% | 80-90% | Greater than 90% |
| Sort Overflow | Greater than 5% | 1-5% | Less than 1% |

---

### Index Strategy

```sql
-- Create targeted indexes
CREATE INDEX idx_sales_date ON sales_data(sale_date);
CREATE INDEX idx_customer_region ON customer_data(region, customer_id);

-- Monitor index effectiveness
SELECT 
    TABSCHEMA,
    TABNAME,
    INDNAME,
    NLEAF,
    NLEVELS,
    CLUSTERRATIO
FROM SYSCAT.INDEXES
WHERE TABSCHEMA = 'YOUR_SCHEMA'
ORDER BY CLUSTERRATIO DESC;

-- Reorganize when CLUSTERRATIO drops
REORG TABLE sales_data INDEX idx_sales_date;
RUNSTATS ON TABLE sales_data WITH DISTRIBUTION AND DETAILED INDEXES ALL;
```

---

### Concurrency & Deadlock Analysis

Leverage IFCID 172 for deadlock detection:

```sql
-- Identify lock wait times
SELECT 
    AGENT_ID,
    APPL_NAME,
    LOCK_WAIT_TIME,
    LOCK_OBJECT_NAME
FROM SYSIBMADM.LOCKS_HELD
WHERE LOCK_WAIT_TIME > 10000  -- 10 seconds
ORDER BY LOCK_WAIT_TIME DESC;

-- Monitor lock escalations (target: 0)
SELECT 
    TABSCHEMA,
    TABNAME,
    LOCK_ESCALATIONS
FROM SYSIBMADM.SNAPDB
WHERE LOCK_ESCALATIONS > 0;
```

**Best Practices:**
- Use row-level locking instead of table-level
- Keep transactions short
- Access tables in consistent order
- Monitor IFCID 172 traces for deadlock patterns

---

## Query Optimization

### General SQL Best Practices

```sql
-- AVOID: Full table scans
SELECT * FROM large_table WHERE expensive_function(column) = 'value';

-- USE: Indexed predicates
SELECT column1, column2 FROM large_table WHERE indexed_column = 'value';

-- AVOID: Unnecessary outer joins
SELECT a.*, b.* FROM table_a a LEFT OUTER JOIN table_b b ON a.id = b.id;

-- USE: Inner joins when possible
SELECT a.*, b.* FROM table_a a INNER JOIN table_b b ON a.id = b.id;

-- AVOID: Data type conversions
SELECT * FROM sales WHERE CHAR(sale_date) = '2024-01-01';

-- USE: Native data types
SELECT * FROM sales WHERE sale_date = DATE('2024-01-01');
```

---

### Wait-Time Analytics

```sql
-- Identify slow queries from historical trends
SELECT 
    STMT_TEXT,
    AVG(TOTAL_EXEC_TIME) AS AVG_EXEC_TIME_MS,
    COUNT(*) AS EXECUTIONS,
    SUM(TOTAL_EXEC_TIME) AS TOTAL_TIME_MS
FROM SYSIBMADM.SNAPDYN_SQL
GROUP BY STMT_TEXT
HAVING AVG(TOTAL_EXEC_TIME) > 5000  -- Queries averaging over 5 seconds
ORDER BY TOTAL_TIME_MS DESC
FETCH FIRST 20 ROWS ONLY;
```

---

### Push Processing to Database

```sql
-- AVOID: Loading millions of rows into analytics tool
-- Then filtering/aggregating in-memory

-- USE: Stored procedures for filtering/aggregation
CREATE PROCEDURE get_sales_summary(IN start_date DATE, IN end_date DATE)
LANGUAGE SQL
BEGIN
    SELECT 
        region,
        SUM(amount) AS total_sales,
        COUNT(*) AS num_transactions
    FROM sales_data
    WHERE sale_date BETWEEN start_date AND end_date
    GROUP BY region;
END;

-- Call from analytics tool with pre-aggregated data
CALL get_sales_summary('2024-01-01', '2024-12-31');
```

---

## Tool-Specific Best Practices

### Cognos Analytics

#### Handle Large Datasets Efficiently

```xml
<!-- Apply cascading filters to avoid full scans -->
<filter>
    <filterExpression>
        [Sales].[Region] = 'North America'
        AND [Sales].[Year] = 2024
        AND [Sales].[Amount] > 10000
    </filterExpression>
</filter>
```

**Performance Tuning:**

1. Increase JVM Heap Size for query services:
```bash
# configuration/cogstartup.xml
<jvm>
    <heap_size>8192</heap_size>  <!-- 8GB -->
    <max_heap_size>16384</max_heap_size>  <!-- 16GB -->
</jvm>
```

2. Minimize data type conversions in report expressions
3. Avoid complex joins in Framework Manager - pre-join in database views
4. Use query caching for frequently accessed reports

**Optimization Checklist:**
- Enable query result caching (default 1 hour)
- Use database aggregation instead of report-level
- Limit prompt cascades to 2-3 levels
- Monitor Query Service heap usage in Administration Console

---

### SPSS Modeler

#### In-Database Processing

```python
# Enable SQL pushback for data manipulation
stream = Modeler.open("your_stream.str")

# Configure database node
db_node = stream.findByType("databaseexport")
db_node.setPropertyValue("pushback", True)
db_node.setPropertyValue("optimization", "speed")
```

**Best Practices:**

1. Filter fields early - Remove unnecessary columns at source
```
[Database] → [Select (keep only needed fields)] → [Filter] → [Model]
```

2. Reorder nodes to reduce data volume upstream:
```
AVOID: [Source] → [Derive] → [Filter] → [Select]
USE:   [Source] → [Select] → [Filter] → [Derive]
```

3. Use multi-threaded modeling nodes (-AS suffix):
   - Neural Net-AS instead of Neural Net
   - C&R Tree-AS instead of C&R Tree
   - Logistic-AS instead of Logistic

4. Enable in-memory processing for medium datasets:
```python
modeler_node.setPropertyValue("use_cache", True)
modeler_node.setPropertyValue("max_memory", 8192)  # 8GB
```

**Performance Impact:**

| Optimization | Speed Improvement | Memory Reduction |
|--------------|-------------------|------------------|
| In-database processing | 3-5x | 70-80% |
| Early filtering | 2-4x | 50-60% |
| Multi-threaded nodes | 1.5-3x | Minimal |
| Field selection | 1.2-2x | 30-40% |

---

### watsonx.data & AI Workloads

#### Model Optimization

**1. Quantization - Reduce model size and inference time:**

```python
# Convert FP32 to FP16 (2x smaller, 1.5-2x faster)
import tensorflow as tf

converter = tf.lite.TFLiteConverter.from_saved_model("model_path")
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.target_spec.supported_types = [tf.float16]
tflite_model = converter.convert()

# INT8 quantization (4x smaller, 2-4x faster)
converter.representative_dataset = representative_data_gen
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
```

**2. Dynamic Batching - Maximize GPU utilization:**

```python
# Configure dynamic batching
model_config = {
    "max_batch_size": 32,
    "dynamic_batching": {
        "preferred_batch_size": [8, 16, 32],
        "max_queue_delay_microseconds": 100
    }
}
```

**3. Workload-Specific Engines:**

| Workload Type | Recommended Engine | Latency Reduction |
|---------------|-------------------|-------------------|
| Structured data queries | Presto | 40-60% |
| Time-series analytics | Apache Spark | 30-50% |
| Real-time inference | TensorFlow Serving | 50-70% |
| Batch predictions | Apache Spark MLlib | 20-40% |

**Performance Targets:**

```
Inference latency targets:
  - P50: Less than 50ms
  - P95: Less than 200ms
  - P99: Less than 500ms

Throughput targets:
  - Minimum: 100 inferences/second
  - Target: 500 inferences/second
  - Optimal: 1000+ inferences/second
```

---

## Hardware & Monitoring

### System Requirements

#### Recommended Configurations

**Cognos Dispatcher Server:**
```
CPU:    16+ cores (24 cores optimal)
RAM:    64GB+ (128GB for large deployments)
Disk:   SSD with 500+ MB/s throughput
Network: 10 Gbps
```

**Database Server (Db2):**
```
CPU:    32+ cores (for parallel query execution)
RAM:    128GB+ (256GB for over 1TB databases)
Disk:   NVMe SSD RAID 10 (for buffer pools & temp space)
Network: 10 Gbps bonded
```

**Development/Testing Environment:**
```
CPU:    8-16 cores
RAM:    32GB (ideal for testing)
Disk:   512GB SSD minimum
OS:     Ubuntu 22.04 LTS / RHEL 8+ / IBM i 7.4+
```

---

### JVM Heap Sizing for Cognos

```bash
# Calculate optimal heap size
# Rule: 25-50% of available RAM, max 32GB per JVM

# For 64GB server:
-Xms16g -Xmx16g  # Query Service
-Xms8g -Xmx8g    # Report Service
-Xms4g -Xmx4g    # Content Manager Service

# Monitor heap usage in Administration Console
# Navigate to: System → Dispatchers & Services → Metrics
```

**Heap Tuning Guidelines:**

| Service | Min Heap | Max Heap | Notes |
|---------|----------|----------|-------|
| Query Service | 8GB | 16GB | Increase for large datasets |
| Report Service | 4GB | 8GB | Scale with concurrent users |
| Content Manager | 2GB | 4GB | Usually sufficient |

---

### Monitoring & Profiling

**Db2 Health Checks:**

```bash
# Daily monitoring script
db2 "SELECT * FROM SYSIBMADM.BP_HITRATIO"
db2 "SELECT * FROM SYSIBMADM.SNAPDB"
db2 "SELECT * FROM SYSIBMADM.LOCKS_HELD WHERE LOCK_WAIT_TIME > 5000"

# Weekly performance report
db2pd -d DBNAME -bufferpools
db2pd -d DBNAME -tablespaces
db2pd -d DBNAME -latches
```

**Cognos Administration Console:**

1. Navigate to: Status → System → Dispatchers and Services
2. Monitor metrics:
   - Active sessions
   - Queue length (keep less than 10)
   - JVM heap usage (keep less than 80%)
   - CPU utilization (balance across dispatchers)

**SolarWinds Integration:**

```sql
-- Custom SQL for Db2 monitoring in SolarWinds
SELECT 
    'Db2 Performance' AS metric_group,
    BP_NAME,
    HIT_RATIO,
    CASE 
        WHEN HIT_RATIO >= 90 THEN 'Healthy'
        WHEN HIT_RATIO >= 80 THEN 'Warning'
        ELSE 'Critical'
    END AS status
FROM SYSIBMADM.BP_HITRATIO;
```

---

### IBM i / AIX Specific

**Ensure Current PTFs:**

```bash
# IBM i - Check PTF status
WRKPTFGRP

# AIX - Check TL/SP level
oslevel -s

# Apply latest Group PTFs
GO PTF → Option 8 (Install)
```

**Performance Tools:**

```bash
# IBM i - Performance monitoring
WRKACTJOB
WRKSYSSTS
WRKDSKSTS

# AIX - System performance
topas
vmstat 5
iostat -T 5
```

---

## Quick Reference

### Performance Optimization Checklist

#### Database Layer
- Buffer pool hit ratio over 90%
- Package cache hit ratio over 90%
- Indexes on all foreign keys and filter columns
- Table/index statistics current (RUNSTATS weekly)
- Lock escalations equal to 0
- Partitioning for tables over 10M rows

#### Query Layer
- No SELECT *
- Indexed WHERE clauses
- Avoid data type conversions
- Use INNER JOIN where possible
- Filtering in stored procedures
- Query execution time less than 5 seconds (P95)

#### Application Layer
- Cognos JVM heap sized appropriately
- SPSS in-database processing enabled
- watsonx.data models quantized
- Result caching enabled
- Connection pooling configured

#### System Layer
- CPUs: 16+ cores for production
- RAM: 64GB+ for analytics servers
- Disk: SSD/NVMe for buffer pools
- Network: 10 Gbps minimum
- OS patches current

---

### Common Issues & Solutions

| Issue | Symptom | Solution |
|-------|---------|----------|
| Slow queries | Execution over 30s | Check buffer pool hit ratio, add indexes |
| High memory usage | JVM crashes | Increase heap size, enable result caching |
| Lock contention | Timeouts, deadlocks | Shorten transactions, analyze IFCID 172 |
| Report timeout | Cognos errors | Push aggregation to database, filter early |
| Model slow inference | P95 over 1s | Quantize model, enable dynamic batching |

---

## Performance Metrics

### Target KPIs

```
Query Performance:
  - P50: Less than 2 seconds
  - P95: Less than 10 seconds
  - P99: Less than 30 seconds

Resource Utilization:
  - CPU: 60-80% (sustained)
  - RAM: 70-85% (buffer pools)
  - Disk I/O: Less than 80% saturation

Cache Efficiency:
  - Buffer pool: Over 90%
  - Package cache: Over 90%
  - Query result: Over 70%

Concurrency:
  - Active sessions: 100+ simultaneous
  - Lock waits: Less than 1% of transactions
  - Deadlocks: 0 per day
```

---

## Contributing

Found optimization techniques that work? Share them! This guide benefits from:
- Performance tuning case studies
- Tool-specific configurations
- Hardware benchmarks
- Monitoring scripts

---

## License

Apache License 2.0 - Educational purposes

---

**Built on IBM i, AIX, and Linux Experience**

"Optimized for handling large-scale analytics workloads efficiently."
