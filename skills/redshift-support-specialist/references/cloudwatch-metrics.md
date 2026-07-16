# CloudWatch Metrics Reference — Amazon Redshift

## Provisioned Clusters

Namespace: `AWS/Redshift`
Dimensions: `ClusterIdentifier` (cluster-level) or `ClusterIdentifier` + `NodeID` (node-level)

### Compute & Resource Metrics

| Metric | Statistic | 🟢 PASS | ⚠️ WARN | ❌ FAIL | Action |
|--------|-----------|---------|---------|---------|--------|
| CPUUtilization | Average, Maximum | < 60% avg | 60-80% avg or > 90% max | > 80% avg sustained | Identify CPU-heavy queries; consider resize or query optimization |
| PercentageDiskSpaceUsed | Average | < 60% | 60-75% | > 75% | Run VACUUM DELETE; archive old data; consider resize |
| DatabaseConnections | Maximum | < 300 | 300-400 | > 400 (approaching 500 limit) | Implement connection pooling; check for leaked connections |
| HealthStatus | Minimum | = 1 (healthy) | — | < 1 (unhealthy) | Investigate immediately — cluster events, recent changes, resource exhaustion |
| MaintenanceMode | Maximum | = 0 | = 1 (maintenance active) | — | Note that maintenance is active; check maintenance window schedule |

### I/O Metrics (Per Node)

| Metric | Statistic | 🟢 PASS | ⚠️ WARN | ❌ FAIL | Action |
|--------|-----------|---------|---------|---------|--------|
| ReadIOPS | Average | Baseline ± 20% | > 2x baseline sustained | > 5x baseline sustained | Check for full table scans; verify sort keys |
| WriteIOPS | Average | Baseline ± 20% | > 2x baseline sustained | > 5x baseline sustained | Check for excessive COPY/INSERT/UPDATE; review commit frequency |
| ReadLatency | Average | < 5ms | 5-10ms | > 10ms sustained | Check disk health; verify no disk-full conditions |
| WriteLatency | Average | < 5ms | 5-10ms | > 10ms sustained | Check commit queue; reduce write concurrency |

### Query Performance Metrics

| Metric | Statistic | 🟢 PASS | ⚠️ WARN | ❌ FAIL | Action |
|--------|-----------|---------|---------|---------|--------|
| QueryDuration | p90 | < 30s | 30-60s | > 60s | Identify slow queries; check EXPLAIN plans |
| WLMQueueWaitTime | Average, Maximum | < 10s avg | 10-30s avg | > 30s avg | Review WLM; enable Concurrency Scaling |
| WLMRunningQueries | Maximum | < concurrency limit | At limit frequently | At limit sustained | Increase concurrency or enable Concurrency Scaling |
| WLMQueueLength | Maximum | < 5 | 5-10 | > 10 sustained | Review WLM priorities and concurrency |
| CommitQueueLength | Maximum | < 3 | 3-5 | > 5 sustained | Reduce write transaction frequency; batch commits |
| ConcurrencyScalingActiveClusters | Sum | 0 | > 0 occasionally | > 0 sustained | Review cost; optimize queries |

### Network Metrics (Per Node)

| Metric | Statistic | 🟢 PASS | ⚠️ WARN | Action |
|--------|-----------|---------|---------|--------|
| NetworkReceiveThroughput | Average | Baseline ± 30% | Sustained saturation | Check for data redistribution; review DISTKEY |
| NetworkTransmitThroughput | Average | Baseline ± 30% | Sustained saturation | Check for large result sets; review UNLOAD operations |

---

## Serverless Workgroups

Namespace: `AWS/Redshift-Serverless`
Dimension: `Workgroup`

| Metric | Statistic | 🟢 PASS | ⚠️ WARN | ❌ FAIL | Action |
|--------|-----------|---------|---------|---------|--------|
| ComputeCapacity (RPU) | Maximum | < 80% of max RPU | At max RPU frequently | At max RPU sustained | Increase max RPU; optimize queries |
| ComputeSeconds | Sum (24h) | Within expected budget | > 1.5x expected | > 2x expected | Review query patterns; check for runaway queries |
| DatabaseConnections | Maximum | < 80% of limit | 80-95% of limit | > 95% of limit | Implement connection pooling |
| QueryDuration | p90 | < 30s | 30-60s | > 60s | Identify slow queries via SYS_QUERY_HISTORY |
| QueriesCompletedPerSecond | Average | Baseline ± 20% | > 30% drop | > 50% drop | Check for resource contention |
| QueriesRunning | Maximum | < concurrency limit | At limit frequently | At limit sustained | Increase RPU; optimize query concurrency |

---

## AWS CLI Examples

**Get metric statistics (last 7 days, daily aggregates):**
```bash
aws cloudwatch get-metric-statistics \
  --namespace "AWS/Redshift" \
  --metric-name "CPUUtilization" \
  --dimensions Name=ClusterIdentifier,Value=my-cluster \
  --start-time $(date -u -v-7d +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 86400 \
  --statistics Average Maximum \
  --profile your-profile \
  --region us-east-1
```

**List all Redshift alarms:**
```bash
aws cloudwatch describe-alarms \
  --alarm-name-prefix "Redshift" \
  --profile your-profile \
  --region us-east-1
```

**Describe alarms by namespace:**
```bash
aws cloudwatch describe-alarms \
  --query "MetricAlarms[?Namespace=='AWS/Redshift']" \
  --profile your-profile \
  --region us-east-1
```
