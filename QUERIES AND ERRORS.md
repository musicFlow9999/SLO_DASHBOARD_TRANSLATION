

```
## 2. mod_status_table — aggregate status with error budget and state
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
| fieldsAdd eb_remaining_pct = if(eb_used_pct < 0, 100, else: if(eb_used_pct > 100, 0, else: 100 - eb_used_pct))
| fieldsAdd state = if(availability_pct < target_pct, "Fail", else: if(availability_pct < warning_pct, "Warn", else: "OK"))
| fields service, target_pct, warning_pct, availability_pct, eb_remaining_pct, state
| sort availability_pct asc

This parameter of the operator `-` should be a number, a timestamp, a duration, a timeframe or an ip address, but was an array.

This parameter of the operator `-` should be a number, a timestamp, a duration or an ip address, but was an array.

This parameter of the operator `/` should be a number or a duration, but was an array.
```

These appear to be type validation error messages indicating that certain operators are receiving array values when they expect scalar values like numbers, timestamps, durations, timeframes, or IP addresses.