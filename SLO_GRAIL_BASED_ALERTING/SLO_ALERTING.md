# SLO Burn-rate Alerting

The following DQL query calculates the burn rate over time for selected services. It compares the error rate to the remaining
error budget derived from the target availability percentage.

```dql
timeseries { total = sum(dt.service.request.count), fail = sum(dt.service.request.failure_count) },
  by: { dt.entity.service },
  filter: { in(dt.entity.service, array($Services)) }
| fieldsAdd error_rate = if(total[] == 0, 0.0, fail[] / total[])
| fieldsAdd burn_rate = error_rate / ((100 - toDouble($TargetPct)) / 100.0)
| fieldsAdd service = entityName(dt.entity.service)
| fieldsKeep dt.entity.service, service, burn_rate, error_rate, timeframe, interval
| sort burn_rate desc
```

To turn this into an alert, evaluate the burn rate against a threshold and emit an event when it is exceeded:

```dql
<base query above>
| summarize burn_rate = avg(burn_rate)
| alert("SLO burn rate exceeded", when: burn_rate > toDouble($Threshold))
```

Replace `$Services`, `$TargetPct`, and `$Threshold` with the desired values or dashboard variables when running the query.
