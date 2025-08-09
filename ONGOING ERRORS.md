## QUERY

timeseries { total = sum(dt.service.request.count) },
  by:{ dt.entity.service },
  filter:{ in(dt.entity.service, array($services)) }
| join [
    timeseries { failed = sum(dt.service.request.count, default: 0.0) },
      by:{ dt.entity.service },
      filter:{ failed == true and in(dt.entity.service, array($services)) }
  ],
  kind:leftOuter,
  on:{ dt.entity.service, timeframe, interval },
  prefix:"err."
| fieldsAdd total = arraySum(total),
           err_failed = arraySum(err.failed)
| summarize { total_w = sum(total), failed_w = sum(err_failed) }, by:{ dt.entity.service }
| fieldsAdd service = entityName(dt.entity.service)
| fieldsAdd target_pct = toDouble($target)
| fieldsAdd error_rate = if(total_w == 0, 0.0, failed_w / total_w)
| fieldsAdd burn_rate = error_rate / ((100 - target_pct)/100.0)
| fields service, burn_rate


## ERROR

TOo many positional parameters have been defined. Optional parameters need to be named. Potential parameters are: else