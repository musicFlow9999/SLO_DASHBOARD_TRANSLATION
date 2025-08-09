# Availability Dashboard Framework

This document provides the baseline DQL query and variable definitions for an Availability SLO dashboard.

## Variable Queries

### `Services`
```dql
fetch dt.entity.service
| fields service = entityName(dt.entity.service), id = id(dt.entity.service)
| sort service
```
Multi-select variable that supplies the service IDs for the dashboard.

### `Target`
Numeric variable representing the target availability percentage (e.g. 99.9).

### `Warning`
Numeric variable representing the warning threshold (e.g. 99.95).

### `SLOPeriod`
Numeric variable in hours describing the evaluation period (e.g. `720` for 30 days).

## Availability and SRE Metrics Query

```dql
timeseries {
  total    = sum(dt.service.request.count),
  failures = sum(dt.service.request.failure_count)
}, by: { dt.entity.service },
   filter: { in(dt.entity.service, array($Services)) }
| fieldsAdd
    total_sum    = arraySum(total),
    failures_sum = arraySum(failures),
    availability_pct = if(total_sum > 0,
      100.0 * (total_sum - failures_sum) / total_sum,
      else: 0.0),
    eval_hours = toDouble($SLOPeriod)
| fieldsAdd
    error_budget           = 100.0 - $Target,
    error_budget_remaining = error_budget - (100.0 - availability_pct),
    error_budget_used      = error_budget - error_budget_remaining,
    burn_rate_1h  = ((100.0 - availability_pct) / error_budget) * (eval_hours / 1.0),
    burn_rate_24h = ((100.0 - availability_pct) / error_budget) * (eval_hours / 24.0)
| fieldsAdd
    service_name = entityName(dt.entity.service),
    status_pct   = availability_pct,
    state = if(availability_pct >= $Target, "ok",
            if(availability_pct >= $Warning, "warning", "violation"))
| fieldsKeep service_name, status_pct, error_budget_remaining, error_budget_used,
             burn_rate_1h, burn_rate_24h, state, total_sum, failures_sum
```

This query replicates Dynatrace SLO metrics, calculates error budget usage, remaining budget, and burn rates for 1h and 24h windows.
