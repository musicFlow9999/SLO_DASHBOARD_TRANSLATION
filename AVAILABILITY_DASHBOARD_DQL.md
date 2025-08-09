# Availability SLO Dashboard – DQL & Variable Queries

This document provides ready-to-use variable definitions and DQL modules for building the availability-focused SLO dashboard described in `README.md` and `AVAILABILITY_DASHBOARD_FRAMEWORK.md`.

## Dashboard Variables

| Variable | Type | Default | Purpose | Variable Query (if applicable) |
| --- | --- | --- | --- | --- |
| `Services` | DQL, multi-select (service IDs) | — | Selects which services to evaluate | See [Services variable query](#services-variable-query) |
| `TargetPct` | Free text / List (double) | `98` | SLO target percentage (fallback) | — |
| `WarningPct` | Free text / List (double) | `99` | Warning threshold percentage | — |
| `MgmtZone` *(optional)* | Tag/List or DQL | — | Scope environment to a management zone | See [MgmtZone variable query](#mgmtzone-variable-query) |

### Services variable query
```dql
fetch dt.entity.service
| fields id = dt.entity.service, name = entityName(dt.entity.service)
| sort name asc
```

### MgmtZone variable query
```dql
fetch dt.entity.management_zone
| fields id = dt.entity.management_zone, name = entityName(dt.entity.management_zone)
| sort name asc
```

*Use `$Var` in DQL, `array($Var)` for multi-selects, and `toDouble($Var)` for numeric usage.*

---

## DQL Modules

Each module respects the **dashboard timeframe** (no `from:/to:` clauses) except where a **tile timeframe override** is explicitly used for burn-rate calculations.

### 1. `mod_availability_timeseries` — service availability trend
```dql
// Availability % = (total - failed) / total * 100
timeseries { total = sum(dt.service.request.count) },
  by:{ dt.entity.service },
  filter:{ in(dt.entity.service, array($Services)) }
| join [
    timeseries { failed = sum(dt.service.request.count, default: 0.0) },
      by:{ dt.entity.service },
      filter:{ failed == true and in(dt.entity.service, array($Services)) }
  ],
  kind:leftOuter,
  on:{ dt.entity.service, timeframe, interval },
  prefix:"err."
| fieldsAdd availability_pct = (total[] - err.failed[]) / total[] * 100
| fieldsAdd service = entityName(dt.entity.service)
| fields service, availability_pct, timeframe, interval
```

### 2. `mod_status_table` — aggregate status with error budget and state
```dql
timeseries { total = sum(dt.service.request.count) },
  by:{ dt.entity.service },
  filter:{ in(dt.entity.service, array($Services)) }
| join [
    timeseries { failed = sum(dt.service.request.count, default: 0.0) },
      by:{ dt.entity.service },
      filter:{ failed == true and in(dt.entity.service, array($Services)) }
  ],
  kind:leftOuter,
  on:{ dt.entity.service, timeframe, interval },
  prefix:"err."
| summarize {
    total_period  = sum(total[]),
    failed_period = sum(err.failed[])
  }, by:{ dt.entity.service }
| fieldsAdd service = entityName(dt.entity.service)
| fieldsAdd availability_pct = (total_period - failed_period) / total_period * 100
| fieldsAdd target_pct  = toDouble($TargetPct)
| fieldsAdd warning_pct = toDouble($WarningPct)
| fieldsAdd eb_used_pct      = ((100 - availability_pct) / (100 - target_pct)) * 100
| fieldsAdd eb_remaining_pct = if(eb_used_pct < 0, 100, if(eb_used_pct > 100, 0, 100 - eb_used_pct))
| fieldsAdd state = if(availability_pct < target_pct, "Fail",
                  if(availability_pct < warning_pct, "Warn", "OK"))
| fields service, target_pct, warning_pct, availability_pct, eb_remaining_pct, state
| sort availability_pct asc
```

### 3. `mod_group_rollup` — rollup across selected services
```dql
timeseries { total = sum(dt.service.request.count) },
  filter:{ in(dt.entity.service, array($Services)) }
| join [
    timeseries { failed = sum(dt.service.request.count, default: 0.0) },
      filter:{ failed == true and in(dt.entity.service, array($Services)) }
  ],
  kind:leftOuter,
  on:{ timeframe, interval },
  prefix:"err."
| summarize { total_period = sum(total[]), failed_period = sum(err.failed[]) }
| fieldsAdd availability_pct = (total_period - failed_period) / total_period * 100
```

### 4. `mod_burn_rate` — burn-rate calculation (timeframe override per tile)
```dql
timeseries { total = sum(dt.service.request.count) },
  by:{ dt.entity.service },
  filter:{ in(dt.entity.service, array($Services)) }
| join [
    timeseries { failed = sum(dt.service.request.count, default: 0.0) },
      by:{ dt.entity.service },
      filter:{ failed == true and in(dt.entity.service, array($Services)) }
  ],
  kind:leftOuter,
  on:{ dt.entity.service, timeframe, interval },
  prefix:"err."
| summarize { total_w = sum(total[]), failed_w = sum(err.failed[]) }, by:{ dt.entity.service }
| fieldsAdd service = entityName(dt.entity.service)
| fieldsAdd target_pct = toDouble($TargetPct)
| fieldsAdd error_rate = if(total_w == 0, 0.0, failed_w / total_w)
| fieldsAdd burn_rate = error_rate / ((100 - target_pct)/100.0)
| fields service, burn_rate
```
*Configure two tiles with this query: set one tile’s timeframe to **Last 1 hour** and another to **Last 24 hours**.*

### 5. `mod_lookup_join` — optional per-service lookup
```dql
... // any of the modules above
| lookup [ load "/lookups/slo_config" ],
    sourceField: dt.entity.service,
    lookupField: service_id
```

---

These queries, combined with the variable definitions above, provide the core building blocks for the availability SLO dashboard that mirrors Dynatrace's native SLO screen while leveraging Grail-backed data and flexible dashboard timeframes.
