# Oracle Advanced Performance Analysis Guide

> This guide focuses on advanced Oracle performance diagnostics used to identify real bottlenecks in production environments.

---

## 1. Objective

This document provides advanced techniques to analyze Oracle database performance beyond basic memory checks.

It focuses on:
- Top SQL analysis
- Wait events
- Session activity
- Locking issues
- I/O performance
- Workarea efficiency

---

## 2. Top SQL Analysis

### Purpose
Identify SQL statements consuming the most resources.

### Query
```sql
SELECT *
FROM (
    SELECT
        sql_id,
        executions,
        elapsed_time/1000000 AS elapsed_sec,
        buffer_gets,
        disk_reads,
        rows_processed,
        SUBSTR(sql_text,1,120) sql_text
    FROM v$sql
    ORDER BY elapsed_time DESC
)
WHERE ROWNUM <= 10;
```

### Interpretation
- High elapsed time → slow SQL
- High buffer gets → CPU-intensive SQL
- High disk reads → I/O-intensive SQL

## 3. Wait Events Analysis
### Purpose
Identify where database time is spent.

### Query
```sql
SELECT
    event,
    total_waits,
    time_waited/100 AS time_sec
FROM v$system_event
ORDER BY time_waited DESC;
```
### Common Events

- db file sequential read → index access
- db file scattered read → full table scan
- log file sync →	commit latency
- enq: TX - row lock contention →	locking

## 4. Active Sessions
### Purpose
Identify sessions currently consuming resources.

### Query
```sql
SELECT
    sid,
    serial#,
    username,
    status,
    sql_id,
    event,
    program
FROM v$session
WHERE status = 'ACTIVE';
```

## 5. Blocking Sessions
### Purpose
Detect locking issues.

### Query
```sql
SELECT
    blocking_session,
    sid,
    serial#,
    wait_class,
    seconds_in_wait
FROM v$session
WHERE blocking_session IS NOT NULL;
```

## 6. I/O Analysis
### Purpose
Evaluate storage performance.

### Query
```sql
SELECT
    file_id,
    phyrds,
    phywrts,
    readtim,
    writetim
FROM v$filestat;
```

## 7. Workarea (Sort/Hash Efficiency)
### Purpose
Check efficiency of PGA usage.

### Query
```sql
SELECT
    operation_type,
    SUM(optimal_executions) optimal_execs,
    SUM(onepass_executions) onepass_execs,
    SUM(multipasses_executions) multipass_execs
FROM v$sql_workarea_histogram
GROUP BY operation_type;
```

### Interpretation
- optimal → good
- onepass → acceptable
- multipasses → performance issue

## 8. Redo / Commit Activity
### Purpose
Evaluate transaction and redo behavior.

### Query
```sql
SELECT name, value
FROM v$sysstat
WHERE name IN (
    'user commits',
    'redo size'
);
```

## 9. Conclusion

### These advanced checks help identify:

- Heavy SQL workload
- System bottlenecks
- Locking issues
- I/O limitations
- Inefficient memory usage

### This guide should be used together with the Instance-Level Assessment Guide for complete database performance analysis.
