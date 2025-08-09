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

Name: DashboardPlan-AvailabilityV1-TF
Description: Same Availability SLI framework, but every query now **uses the inbuilt timeframe picker** (global dashboard timeframe) and **tile-level timeframe overrides** for burn-rate lenses. No hardcoded `from:/to:` anywhere.
Category: taskwork

# Changes at a glance

* **Remove `from:/to:` from all queries** so the **dashboard timeframe** drives evaluation windows. ([Dynatrace Community][1])
* **Burn-rate tiles** use the **tile’s own timeframe override** (set to *Last 1 hour* and *Last 24 hours*) instead of coding windows in DQL. ([Dynatrace Documentation][2])
* Variables simplified: drop `EvalDays`, keep `Services`, `TargetPct`, `WarningPct`. ([Dynatrace Documentation][3])

---

# Variables (updated)

| Var                   | Type                    | Default | Purpose                              | Used As                                   |
| --------------------- | ----------------------- | ------- | ------------------------------------ | ----------------------------------------- |
| `Services`            | DQL, multi-select (ids) | —       | Which services to include            | `in(dt.entity.service, array($Services))` |
| `TargetPct`           | Free text/List (double) | `98`    | SLO target % (fallback if no lookup) | `toDouble($TargetPct)`                    |
| `WarningPct`          | Free text/List (double) | `99`    | Warning threshold %                  | `toDouble($WarningPct)`                   |
| `MgmtZone` (optional) | Tag/List                | —       | Environment scoping                  | Optional filter block                     |

> Notes: Use `$Var` in DQL, `array($Var)` for multi-selects, and `toDouble($Var)` or `:noquote` for numeric use. ([Dynatrace Documentation][3], [Dynatrace Community][4])

---

# DQL modules (timeframe-aware)

## 1) mod\_availability\_timeseries (per service)

*Respects the **dashboard timeframe** automatically.*

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

(Uses Grail’s converged service request metric; failures via the boolean `failed` dimension.) ([Dynatrace Documentation][5])

## 2) mod\_status\_table (period aggregate + EB% + state)

*Aggregates over the **dashboard timeframe**; mirrors native SLO status.*

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

(Only dashboard timeframe drives the period—no explicit `from:/to:`.) ([Dynatrace Community][1])

## 3) mod\_group\_rollup (rollup across selected services)

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

(Compute on totals first; don’t average percentages.) ([Dynatrace Documentation][6])

## 4) mod\_burn\_rate (single-window; tile timeframe decides the window)

*Use this once; make two tiles with different **tile timeframe overrides**: one set to **Last 1 hour**, one to **Last 24 hours**.*

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

**Tile setup:** In the tile’s config, set **Timeframe → Custom → Last 1 hour** (and for the second tile, **Last 24 hours**). Then color by MWMBR thresholds (e.g., page at BR ≥ 14.4 for 1h, BR ≥ 6 for 24h). ([Dynatrace Documentation][2], [Google SRE][7])

## 5) mod\_lookup\_join (per-service targets/warnings/names)

```dql
... // (run one of the modules above)
| lookup [ load "/lookups/slo_config" ],
    sourceField: dt.entity.service,
    lookupField: service_id
```

(Use Grail lookup files; ensure permissions for `/lookups/*` in Grail.) ([Dynatrace Documentation][8])

---

# Layout & tiles (wireframe, with timeframe behavior)

**Global control**

* The **dashboard timeframe picker** (top-right) defines the evaluation period for: *Status KPIs, Status Table, Trend*. No query has `from:/to:`. ([Dynatrace Documentation][9])

**Row 1 – KPIs (Use global timeframe)**

* **Status % (All)** — `mod_group_rollup`
* **Error Budget Remaining % (All)** — computed from rollup + `$TargetPct`
* **State (All)** — OK/Warn/Fail vs `$TargetPct/$WarningPct`

**Row 2 – Burn-Rate lenses (Tile timeframe overrides)**

* **BR (1h)** — `mod_burn_rate` tile with **tile timeframe: Last 1 hour**; color rule page at **≥14.4**
* **BR (24h)** — same query; **tile timeframe: Last 24 hours**; page at **≥6**
  (Do not add `from:-1h`/`-24h` in DQL—let the tile timeframe handle it.) ([Dynatrace Documentation][2], [Google SRE][7])

**Row 3 – SLO Status Table (Use global timeframe)**

* `mod_status_table` + optional `mod_lookup_join` (shows `name | target | warning | status % | EB remaining % | state`)

**Row 4 – Trend (Use global timeframe)**

* `mod_availability_timeseries` (service legend)

---

# Guard-rails & gotchas

* If you put `from:/to:` in a tile, you **override** the picker. Omit them unless you intend to fix a window. ([Dynatrace Community][1])
* For failures, use the **`failed` boolean dimension** on `dt.service.request.count`; Dynatrace converged service metrics rely on this pattern in Grail. ([Dynatrace Documentation][10])
* Lookups require **Grail file permissions** (read/write to `/lookups/`). ([Dynatrace Documentation][11])

---

# Implementation steps (quick)

1. **Add tiles** → *Add > DQL* → paste the module DQL → *Run* → switch to *Visual* to pick table/line/KPI. ([Dynatrace Documentation][12])
2. **Set timeframe**

   * For “global” tiles (Status/Trend/Table): **Use dashboard timeframe**.
   * For burn-rate tiles: **set tile timeframe override** (1h and 24h). ([Dynatrace Documentation][2])
3. **Add variables** (`Services`, `TargetPct`, `WarningPct`) and reference them as shown. ([Dynatrace Documentation][3])
4. **(Optional) Join lookup** `/lookups/slo_config` for per-service targets/names. ([Dynatrace Documentation][8])

---

Want me to output a **ready-to-import dashboard JSON** that already sets tile timeframe overrides for the two burn-rate tiles and uses the global picker everywhere else?

[1]: https://community.dynatrace.com/t5/Open-Q-A/New-dashboards-and-time-frame-selector/m-p/214100?utm_source=chatgpt.com "New dashboards and time frame selector? - Dynatrace Community"
[2]: https://docs.dynatrace.com/docs/analyze-explore-automate/dashboards-classic/dashboards/dashboard-timeframe?utm_source=chatgpt.com "Dynatrace dashboard timeframe and management zone settings"
[3]: https://docs.dynatrace.com/docs/analyze-explore-automate/dashboards-and-notebooks/dashboards-new/components/dashboard-component-variable?utm_source=chatgpt.com "Add a variable to a dashboard - Dynatrace Documentation"
[4]: https://community.dynatrace.com/t5/DQL/Conditional-DQL-statements-based-on-value-of-input-variable/m-p/256566?utm_source=chatgpt.com "Solved: Conditional DQL statements based on value of input variable"
[5]: https://docs.dynatrace.com/docs/analyze-explore-automate/metrics/upgrade/service-metric-migration?utm_source=chatgpt.com "Service metrics migration guide - Dynatrace Documentation"
[6]: https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/commands/metric-commands?utm_source=chatgpt.com "DQL Metric commands — Dynatrace Docs"
[7]: https://sre.google/workbook/alerting-on-slos/?utm_source=chatgpt.com "Prometheus Alerting: Turn SLOs into Alerts - Google SRE"
[8]: https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/commands/data-source-commands?utm_source=chatgpt.com "DQL Data source commands - Dynatrace Documentation"
[9]: https://docs.dynatrace.com/docs/analyze-explore-automate/dashboards-and-notebooks/dashboards-new?utm_source=chatgpt.com "Dashboards — Dynatrace Docs"
[10]: https://docs.dynatrace.com/docs/analyze-explore-automate/metrics/upgrade/calculated-service-metrics-upgrade?utm_source=chatgpt.com "Calculated service metrics upgrade guide - Dynatrace Documentation"
[11]: https://docs.dynatrace.com/docs/discover-dynatrace/platform/grail/data-model/assign-permissions-in-grail?utm_source=chatgpt.com "Permissions in Grail - Dynatrace Documentation"
[12]: https://docs.dynatrace.com/docs/analyze-explore-automate/dashboards-and-notebooks/dashboards-new/components/dashboard-component-data?utm_source=chatgpt.com "Add data to a dashboard — Dynatrace Docs"
