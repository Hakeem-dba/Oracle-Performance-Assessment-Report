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
