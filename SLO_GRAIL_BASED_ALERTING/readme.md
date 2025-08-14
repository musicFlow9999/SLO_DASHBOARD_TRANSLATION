The following is the starting SLO that is configured in Dynatrace:

timeseries { total=sum(dt.service.request.count), failures=sum(dt.service.request.failure_count) }
, by: { dt.entity.service }
, filter: { in (dt.entity.service, { "SERVICE-5704D468C6099AE4" }) }
| fieldsAdd sli=((total[]-failures[])/total[])*100
| fieldsAdd entityName(dt.entity.service)
| fieldsRemove total, failures



The below information is being presented in a screen related to the SLO dql query above


Configuration Details
    • Target: 98% availability over the selected evaluation period.
    • Evaluation Period: Last 14 days.
    • Show Warning: Enabled — warning threshold set to 99% availability.
    • Preview Mode: Displays the availability over time in a graph or table format (currently showing the table option).
Current Status (as of the snapshot)
    • Status: 100% — the service has fully met the availability target for the evaluation period.
    • Error Budget: 2% — meaning the service can afford up to 2% downtime/errors before breaching the SLO.
    • Target: 98% — the defined success rate goal for availability.
Evaluation Period: Last 14 days — all calculations and status checks are based on the past two weeks’ data.



Goal:
I need to create a dql query that gives me burn rate over time. I will then set up a threshold that will alert if the burn rate is above or below by inputted threshold value.
Please review the Dynatrace DOCs related to this and make sure it aligns with correct syntax