## QUERY

Summarize scalar values instead of arrays:

```dql
timeseries { total = sum(dt.service.request.count) },
  by:{ dt.entity.service },
  filter:{ in(dt.entity.service, array($services)) }
| join [
    timeseries { failed = sum(dt.service.request.count, default: 0.0) },
      by:{ dt.entity.service },
      filter:{ failed == true and in(dt.entity.service, array($Services)) }
  ],
  kind:leftOuter,
  on:{ dt.entity.service, timeframe, interval },
  prefix:"err."
| fieldsAdd err_failed = arraySum(err.failed) //if multiple entries exist
| summarize {
    total_period  = sum(total),
    failed_period = sum(err_failed)
  }, by:{ dt.entity.service }
| fieldsAdd service = entityName(dt.entity.service)
| fieldsAdd availability_pct = (total_period - failed_period) / total_period * 100
| fieldsAdd target_pct  = toDouble($TargetPct)
| fieldsAdd warning_pct = toDouble($WarningPct)
| fieldsAdd eb_used_pct      = ((100 - availability_pct) / (100 - target_pct)) * 100
| fieldsAdd eb_remaining_pct = if(eb_used_pct < 0, 100, else: if(eb_used_pct > 100, 0, else: 100 - eb_used_pct))
| fieldsAdd state = if(availability_pct < target_pct, "Fail", else: if(availability_pct < warning_pct, "Warn", else: "OK"))
| fields service, target_pct, warning_pct, availability_pct, eb_remaining_pct, state
| sort availability_pct asc
```
## ERROR
```
This parameter of the `sum()` function should be a number or a duration, but was an array
```

