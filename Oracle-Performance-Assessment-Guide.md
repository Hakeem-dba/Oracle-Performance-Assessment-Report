# Oracle Performance Assessment – Step-by-Step Guide

## Objective
This document provides a structured approach to assess Oracle database performance and identify bottlenecks.

Each section includes:
- SQL queries  
- Expected results  
- Interpretation  
- Recommended actions  

---

# 1. SGA (Buffer Cache) Health Check

## Purpose
Evaluate how efficiently Oracle reads data from memory vs disk.

## Query
```sql
SELECT
    SUM(CASE WHEN name IN ('db block gets','consistent gets') THEN value ELSE 0 END) AS logical_reads,
    SUM(CASE WHEN name = 'physical reads' THEN value ELSE 0 END) AS physical_reads,
    ROUND(
      100 * (
        SUM(CASE WHEN name IN ('db block gets','consistent gets') THEN value ELSE 0 END)
        - SUM(CASE WHEN name = 'physical reads' THEN value ELSE 0 END)
      ) / NULLIF(SUM(CASE WHEN name IN ('db block gets','consistent gets') THEN value ELSE 0 END),0),
      2
    ) AS buffer_hit_ratio
FROM v$sysstat
WHERE name IN ('db block gets','consistent gets','physical reads');
```
## Expected Result

- 90% → Good
- 80–90% → Acceptable
- < 80% → Needs investigation

## Interpretation
- Low ratio may indicate more disk I/O
- Not sufficient alone for tuning

## Action

- Do NOT increase buffer cache blindly
- Use V$DB_CACHE_ADVICE


# 2. Data Dictionary Cache

## Query
```sql
SELECT SUM(gets),
       SUM(getmisses),
       TRUNC((1-(SUM(getmisses)/SUM(gets)))*100)
FROM v$rowcache;
```
## Expected Result

- 95% → Excellent
- 90–95% → Acceptable
- < 90% → Needs tuning

## Interpretation
Low ratio → shared pool may be undersized

##Action
Increase shared_pool_size only if confirmed

# 3. Library Cache (SQL Reuse)
## Query
```sql
SELECT namespace,
       TRUNC(gethitratio * 100) AS hit_ratio,
       TRUNC(pinhitratio * 100) AS pin_hit_ratio,
       reloads
FROM v$librarycache;
```

## Expected Result
- Hit Ratio > 90%
- Reloads should be low
## Interpretation
- Low SQL AREA hit ratio → poor SQL reuse
- High reloads → shared pool churn
## Action
- Check bind variables
- Check version count
- Investigate child cursors

# 4. Parsing Activity
## Query
```sql
SELECT 
    SUM(parse_calls) total_parse_calls,
    SUM(executions) total_executions,
    ROUND(SUM(parse_calls) / NULLIF(SUM(executions),0) * 100, 2) parse_pct
FROM v$sql;
```

## Expected Result
- < 10% → Good
- 10–30% → Moderate
- 30% → Problem

## Hard Parse Check
```sql
SELECT name, value
FROM v$sysstat
WHERE name IN (
  'parse count (total)',
  'parse count (hard)'
);
```

## Expected Result
Hard parse ratio: 
- < 1% → Excellent
- 1-5% → Problem
- > 5% → critical

## Interpretation
High parsing → CPU overhead

##Action
- Use bind variables
- Improve cursor reuse

# 5. PGA (Memory for Sort/Hash)
##Query
```sql
SELECT name, value
FROM v$pgastat;
```
## Expected Result
- Cache Hit %	> 90%
- Over Allocation	0

## Interpretation
- Low cache hit → spills to TEMP
- High over allocation → PGA too small

##Action
- Increase PGA if needed

# 6. PGA Target Advisory
## Query
```sql
SELECT
    round(pga_target_for_estimate/1024/1024) target_mb,
    estd_pga_cache_hit_percentage,
    estd_overalloc_count
FROM v$pga_target_advice
ORDER BY pga_target_for_estimate;
```

## Expected Result
Choose value where:
- Cache hit ~95–100%
- Overalloc = 0

## Action
```sql
ALTER SYSTEM SET pga_aggregate_target = <VALUE> SCOPE=BOTH;
```

# 7. TEMP Tablespace Usage
## Query
```sql
SELECT
    s.sid,
    s.serial#,
    s.username,
    s.program,
    u.tablespace,
    u.segtype,
    u.blocks * t.block_size / 1024 / 1024 AS mb_used
FROM v$sort_usage u
JOIN v$session s
  ON s.saddr = u.session_addr
JOIN dba_tablespaces t
  ON t.tablespace_name = u.tablespace
ORDER BY mb_used DESC;
```

## Expected Result
- Moderate TEMP usage

## Interpretation
- High usage → sorts, hash joins, PGA shortage

## Action
- Increase PGA
- Tune SQL


# 8. SGA Free Memory
## Query
```sql
SELECT pool, name, bytes
FROM v$sgastat
WHERE name = 'free memory'
ORDER BY bytes DESC;
```

## Expected Result
- Some free memory available
## Interpretation
- No free memory → pressure
## Action
- Do not increase SGA blindly


# Final Assessment Logic
## Case 1

### High parsing + low SQL reuse
- → Fix SQL / bind variables

## Case 2

### Low PGA cache hit + high TEMP
- → Increase PGA

## Case 3

### Low buffer cache ratio
- → Validate with DB cache advice
