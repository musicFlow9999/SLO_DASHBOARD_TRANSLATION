## QUERY

timeseries { total = sum(dt.service.request.count), fail = sum(dt.service.request.failure_count) },
  by: { dt.entity.service },
  filter: { in(dt.entity.service, array($Services)) }
| fieldsAdd error_rate = if(total[] == 0, 0.0, fail[] / total[])
| fieldsAdd burn_rate = error_rate / ((100 - toDouble($TargetPct)) / 100.0)
| fieldsAdd service = entityName(dt.entity.service)
| fieldsKeep dt.entity.service, service, burn_rate, error_rate, timeframe, interval
| sort burn_rate desc

## ERROR

TOo many positional parameters have been defined. Optional parameters need to be named. Potential parameters are: else