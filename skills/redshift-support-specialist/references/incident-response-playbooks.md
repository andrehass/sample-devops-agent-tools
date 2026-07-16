# Incident Response Playbooks — Amazon Redshift

Quick-reference playbooks for responding to CloudWatch alarm triggers.

---

## P0 — Immediate Response

### HealthStatus < 1

**Severity:** Critical — cluster unhealthy
**First Response:**
1. Check cluster events: `aws redshift describe-events --source-type cluster --source-identifier <cluster-id> --duration 60`
2. Check cluster status: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query "Clusters[0].ClusterStatus"`
3. Look for recent changes (resizes, maintenance, parameter changes)
4. Check resource exhaustion (disk full, connection limit)

**Common Causes:**
- Hardware failure (node replacement in progress)
- Disk full preventing writes
- Failed resize operation
- Network connectivity issues

---

### PercentageDiskSpaceUsed > 75%

**Severity:** Critical — disk full blocks writes and queries
**First Response:**
1. Run `VACUUM DELETE` on tables with high deletion bloat
2. Check for ghost rows: `SELECT * FROM STL_DISK_FULL_DIAG`
3. Archive old data to S3 via UNLOAD
4. Check for unsorted data: `SELECT "table", unsorted FROM SVV_TABLE_INFO WHERE unsorted > 20 ORDER BY unsorted DESC`

**Immediate Mitigations:**
- `VACUUM DELETE ONLY <table>` on largest tables with deletion bloat
- Drop unused temp tables
- Consider elastic resize to add nodes

---

## P1 — Urgent (respond within 1 hour)

### CPUUtilization > 80% sustained

**First Response:**
1. Identify CPU-heavy queries:
   ```sql
   SELECT query_id, user_id, execution_time/1000000.0 as exec_sec,
          SUBSTRING(query_text, 1, 200) as query
   FROM sys_query_history
   WHERE start_time >= DATEADD(hour, -1, GETDATE())
     AND status = 'success'
   ORDER BY execution_time DESC LIMIT 10;
   ```
2. Check for runaway queries that should be killed
3. Review WLM queue — queries may be competing for resources
4. Check if auto-vacuum or auto-analyze is running

**Common Causes:**
- Runaway query with nested loops
- Multiple concurrent large queries
- Auto-vacuum on large tables during peak hours
- Missing sort keys causing full table scans

---

### DatabaseConnections > 400

**First Response:**
1. Check active sessions:
   ```sql
   SELECT user_name, db_name, COUNT(*) as connection_count
   FROM stv_sessions
   GROUP BY user_name, db_name
   ORDER BY connection_count DESC;
   ```
2. Look for leaked connections (idle sessions with no recent activity)
3. Check application connection pooling configuration

**Mitigations:**
- Kill idle sessions: `SELECT pg_terminate_backend(<pid>)`
- Implement connection pooling (pgBouncer, application-level)
- Set `idle_in_transaction_session_timeout` parameter

---

### WLMQueueWaitTime > 30s

**First Response:**
1. Check current queue state:
   ```sql
   SELECT service_class, num_executing_queries,
          num_queued_queries, num_slots
   FROM stv_wlm_service_class_state
   WHERE service_class > 4
   ORDER BY num_queued_queries DESC;
   ```
2. Identify blocking queries in the queue
3. Review WLM priorities — are low-priority queries starving high-priority ones?

**Mitigations:**
- Enable Concurrency Scaling for burst capacity
- Adjust WLM queue concurrency levels
- Enable Short Query Acceleration (SQA) to route small queries to fast lane
- Kill long-running queries blocking the queue

---

### WLMQueueLength > 10

Same response as WLMQueueWaitTime. Queries are backing up.

---

## P2 — Soon (respond within 4 hours)

### ReadLatency / WriteLatency > 10ms

**First Response:**
1. Check disk space (latency increases as disk fills)
2. Check for I/O-heavy queries (full table scans)
3. Verify sort keys are effective (check zone-map pruning)

---

### QueryDuration (p90) > 60s

**First Response:**
1. Identify the slow queries from SYS_QUERY_HISTORY
2. Check EXPLAIN plans for inefficiencies
3. Review table design (distribution, sort keys, compression)
4. Check for stale statistics (`ANALYZE` needed)

---

### CommitQueueLength > 5

**First Response:**
1. Identify frequent committers:
   ```sql
   SELECT node, COUNT(*) as commit_count,
          AVG(DATEDIFF(microsecond, startqueue, endtime))/1000000.0 as avg_commit_sec
   FROM stl_commit_stats
   WHERE startqueue >= DATEADD(hour, -1, GETDATE())
   GROUP BY node ORDER BY commit_count DESC;
   ```
2. Look for single-row INSERTs (should be batched with COPY)
3. Reduce commit frequency — batch operations

---

## P3 — Advisory

### ConcurrencyScalingActiveClusters > 0

**First Response:**
- Review which queries are triggering burst scaling
- Evaluate cost impact
- Consider resize if burst is sustained
- Optimize heavy queries to reduce concurrent load

### MaintenanceMode > 0

**First Response:**
- Note that maintenance is in progress
- Verify maintenance window is set to off-peak hours
- No action needed unless maintenance is unexpected
