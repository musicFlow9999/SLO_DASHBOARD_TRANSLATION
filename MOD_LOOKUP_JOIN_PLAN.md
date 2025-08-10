# Plan for `mod_lookup_join` Per-Service Lookup

### Step 2: Upload Data
```bash
curl -X 'POST' \
  'https://<environment>.apps.dynatrace.com/platform/storage/resource-store/v1/files/tabular/lookup:upload' \
  -H 'Authorization: Bearer <platformtoken>' \
  -H 'Content-Type: multipart/form-data' \
  -F 'request={\n      "parsePattern":"INT:code LD:service_id "," DOUBLE:custom_slo_target "," DOUBLE:warning_threshold "," LD:team "," LD:criticality",\n      "lookupField":"service_id",\n      "filePath":"/lookups/slo_config",\n      "displayName":"SLO Configuration",\n      "description":"Per-service SLO targets and metadata"\n    }' \
  -F 'content=@slo_config.csv'
```utlines how to enrich availability SLO queries with per-service metadata using Dynatrace Grail lookup tables.

## 1. Prepare Lookup Data

Build a table that defines service-level overrides. Example structure:

### CSV Format
```csv
service_id,custom_slo_target,warning_threshold,team,criticality
SERVICE-ABC123,99.9,99.95,Platform,High
SERVICE-DEF456,95.0,97.0,Backend,Medium
SERVICE-GHI789,99.0,99.5,Frontend,High
SERVICE-JKL012,98.0,99.0,Data,Low
```

### JSON Lines Format
```json
{"service_id":"SERVICE-ABC123","custom_slo_target":99.9,"warning_threshold":99.95,"team":"Platform","criticality":"High"}
{"service_id":"SERVICE-DEF456","custom_slo_target":95.0,"warning_threshold":97.0,"team":"Backend","criticality":"Medium"}
```

Store the file under the `/lookups` prefix; file paths must begin with `/`, contain only alphanumeric characters plus `-`, `_`, `.`, and `/`, and follow structured naming like `/lookups/slo_config`.

## 2. Configure Access

- **Dashboard consumption**: Requires `storage:files:read` permission
- **Maintenance operations**: Additionally requires `storage:files:write` and `storage:files:delete`

Example permission policy:
```
ALLOW storage:files:read WHERE storage:file-path startsWith "/lookups/";
ALLOW storage:files:write WHERE storage:file-path startsWith "/lookups/";
ALLOW storage:files:delete WHERE storage:file-path startsWith "/lookups/";
```

## 3. Upload and Maintain Lookup Files

### Step 1: Test Parsing
```bash
curl -X 'POST' \
  'https://<environment>.apps.dynatrace.com/platform/storage/resource-store/v1/files/tabular/lookup:test-pattern' \
  -H 'Authorization: Bearer <platformtoken>' \
  -H 'Content-Type: multipart/form-data' \
  -F 'request={"parsePattern":"INT:code LD:service_id \",\" DOUBLE:custom_slo_target \",\" DOUBLE:warning_threshold \",\" LD:team \",\" LD:criticality","lookupField":"service_id"}' \
  -F 'content=@slo_config.csv'
```

### Step 2: Upload Data
```bash
curl -X 'POST' \
  'https://<environment>.apps.dynatrace.com/platform/storage/resource-store/v1/files/tabular/lookup:upload' \
  -H 'Authorization: Bearer <platformtoken>' \
  -H 'Content-Type: multipart/form-data' \
  -F 'request={
      "parsePattern":"INT:code LD:service_id \",\" DOUBLE:custom_slo_target \",\" DOUBLE:warning_threshold \",\" LD:team \",\" LD:criticality",
      "lookupField":"service_id",
      "filePath":"/lookups/slo_config",
      "displayName":"SLO Configuration",
      "description":"Per-service SLO targets and metadata"
    }' \
  -F 'content=@slo_config.csv'
```

### Step 3: Update or Delete (as needed)
For updates, use the same upload command with `"overwrite": true` in the request.
For deletion:
```bash
curl -X 'POST' \
  'https://<environment>.apps.dynatrace.com/platform/storage/resource-store/v1/files:delete' \
  -H 'Authorization: Bearer <platformtoken>' \
  -d '{"filePath": "/lookups/slo_config"}'
```

## 4. Validate Uploaded Data

List stored files:
```dql
fetch dt.system.files 
| filter filePath startsWith "/lookups/"
```

Inspect contents:
```dql
load "/lookups/slo_config"
```

## 5. Join Lookup Data in SLO Queries

### Basic Integration
```dql
... // output from a prior module
| lookup [ load "/lookups/slo_config" ],
    sourceField: dt.entity.service,
    lookupField: service_id
| fieldsAdd target_pct = if(isNotNull(custom_slo_target),
                            custom_slo_target,
                            else: toDouble($TargetPct))
| fieldsAdd warning_pct = if(isNotNull(warning_threshold),
                             warning_threshold,
                             else: toDouble($WarningPct))
```

### Integration with `mod_status_table`
```dql
timeseries { total = sum(dt.service.request.count) },
  by: { dt.entity.service },
  filter: { in(dt.entity.service, array($services)) }
| join [
    timeseries { failed = sum(dt.service.request.count, default: 0.0) },
      by: { dt.entity.service },
      filter: { failed == true and in(dt.entity.service, array($services)) }
  ],
  kind: leftOuter,
  on: { dt.entity.service, timeframe, interval },
  prefix: "err."
| fieldsAdd total = arraySum(total),
           err_failed = arraySum(err.failed)
| summarize {
    total_period = sum(total),
    failed_period = sum(err_failed)
  }, by: { dt.entity.service }
| fieldsAdd availability_pct = (total_period - failed_period) / total_period * 100
| lookup [ load "/lookups/slo_config" ],
    sourceField: dt.entity.service,
    lookupField: service_id
| fieldsAdd 
    service = entityName(dt.entity.service),
    target_pct = coalesce(custom_slo_target, toDouble($TargetPct)),
    warning_pct = coalesce(warning_threshold, toDouble($WarningPct))
| fieldsAdd 
    eb_used_pct = ((100 - availability_pct) / (100 - target_pct)) * 100,
    eb_remaining_pct = if(eb_used_pct < 0, 100, 
                         else: if(eb_used_pct > 100, 0, 
                         else: 100 - eb_used_pct))
| fieldsAdd state = if(availability_pct < target_pct, "Fail", 
                      else: if(availability_pct < warning_pct, "Warn", 
                      else: "OK"))
| fields service, team, criticality, target_pct, warning_pct, 
         availability_pct, eb_remaining_pct, state
| sort availability_pct asc
```

### Integration with `mod_burn_rate`
```dql
timeseries { total = sum(dt.service.request.count) },
  by: { dt.entity.service },
  filter: { in(dt.entity.service, array($services)) }
| join [
    timeseries { failed = sum(dt.service.request.count, default: 0.0) },
      by: { dt.entity.service },
      filter: { failed == true and in(dt.entity.service, array($services)) }
  ],
  kind: leftOuter,
  on: { dt.entity.service, timeframe, interval },
  prefix: "err."
| fieldsAdd total = arraySum(total),
           err_failed = arraySum(err.failed)
| summarize { total_w = sum(total), failed_w = sum(err_failed) }, 
  by: { dt.entity.service }
| lookup [ load "/lookups/slo_config" ],
    sourceField: dt.entity.service,
    lookupField: service_id
| fieldsAdd 
    service = entityName(dt.entity.service),
    target_pct = coalesce(custom_slo_target, toDouble($TargetPct))
| fieldsAdd error_rate = if(total_w == 0, 0.0, else: failed_w / total_w)
| fieldsAdd burn_rate = error_rate / ((100 - target_pct) / 100.0)
| fields service, team, criticality, target_pct, burn_rate
```

## 6. Grail Data Visualization Examples

### Service Health Dashboard with Lookup Integration

```dql
// Query with enriched service metadata for comprehensive dashboard
fetch events
| filter event.type == "AVAILABILITY_EVENT"
| fields service_id = entity.service.id,
         availability = isNotNull(event.end) ? 0 : 1,
         timestamp = timestamp
| summarize 
    total_checks = count(),
    successful_checks = sum(availability),
    by: {service_id, bin(timestamp, 1h)}
| fields service_id,
         availability_pct = (toDouble(successful_checks) / toDouble(total_checks)) * 100.0,
         timestamp,
         period = "1h"
| lookup [
    fetch dt.system.files
    | filter path == "/lookups/slo_config"
    | fields content
  ], sourceField:service_id, lookupField:service_id
| fields service_id,
         availability_pct,
         timestamp,
         period,
         slo_target = coalesce(lookup.custom_slo_target, 99.0),
         warning_threshold = coalesce(lookup.warning_threshold, 99.5),
         team = coalesce(lookup.team, "Unknown"),
         criticality = coalesce(lookup.criticality, "Medium"),
         slo_status = if(availability_pct >= lookup.custom_slo_target, "Met", 
                         if(availability_pct >= lookup.warning_threshold, "Warning", "Failed")),
         health_score = availability_pct / lookup.custom_slo_target * 100.0
| sort timestamp desc
```

### Team Performance Overview

```dql
// Aggregate availability by team using lookup data
fetch events
| filter event.type == "AVAILABILITY_EVENT"
| fields service_id = entity.service.id,
         availability = isNotNull(event.end) ? 0 : 1
| summarize 
    total_checks = count(),
    successful_checks = sum(availability),
    by: {service_id}
| fields service_id,
         availability_pct = (toDouble(successful_checks) / toDouble(total_checks)) * 100.0
| lookup [
    fetch dt.system.files
    | filter path == "/lookups/slo_config"
    | fields content
  ], sourceField:service_id, lookupField:service_id
| fields team = coalesce(lookup.team, "Unknown"),
         criticality = coalesce(lookup.criticality, "Medium"),
         service_id,
         availability_pct,
         slo_target = coalesce(lookup.custom_slo_target, 99.0),
         slo_met = availability_pct >= lookup.custom_slo_target
| summarize 
    services_count = countDistinct(service_id),
    avg_availability = avg(availability_pct),
    services_meeting_slo = countDistinct(if(slo_met, service_id)),
    high_criticality_services = countDistinct(if(criticality == "High", service_id)),
    by: {team}
| fields team,
         services_count,
         avg_availability,
         slo_compliance_rate = (toDouble(services_meeting_slo) / toDouble(services_count)) * 100.0,
         high_criticality_services
| sort slo_compliance_rate desc
```

### Criticality-Based Alerting Query

```dql
// Alert conditions based on service criticality from lookup data
fetch events
| filter event.type == "AVAILABILITY_EVENT"
| filter timestamp >= now() - 1h
| fields service_id = entity.service.id,
         availability = isNotNull(event.end) ? 0 : 1,
         timestamp
| summarize 
    total_checks = count(),
    successful_checks = sum(availability),
    by: {service_id}
| fields service_id,
         current_availability = (toDouble(successful_checks) / toDouble(total_checks)) * 100.0
| lookup [
    fetch dt.system.files
    | filter path == "/lookups/slo_config"
    | fields content
  ], sourceField:service_id, lookupField:service_id
| fields service_id,
         current_availability,
         slo_target = coalesce(lookup.custom_slo_target, 99.0),
         warning_threshold = coalesce(lookup.warning_threshold, 99.5),
         team = coalesce(lookup.team, "Unknown"),
         criticality = coalesce(lookup.criticality, "Medium")
| fields service_id,
         team,
         criticality,
         current_availability,
         slo_target,
         warning_threshold,
         alert_level = if(current_availability < slo_target,
                         if(criticality == "High", "CRITICAL",
                            if(criticality == "Medium", "WARNING", "INFO")),
                         "OK"),
         deviation = slo_target - current_availability
| filter alert_level != "OK"
| sort criticality desc, deviation desc
```

### Service Dependency Impact Analysis

```dql
// Analyze impact of service issues considering criticality and team ownership
fetch events
| filter event.type == "AVAILABILITY_EVENT"
| filter timestamp >= now() - 24h
| fields service_id = entity.service.id,
         availability = isNotNull(event.end) ? 0 : 1,
         timestamp,
         duration = if(isNotNull(event.end), event.end - event.start, now() - event.start)
| summarize 
    total_checks = count(),
    successful_checks = sum(availability),
    total_downtime_minutes = sum(if(availability == 0, duration/1000000000/60, 0)),
    incident_count = count(if(availability == 0, 1)),
    by: {service_id}
| fields service_id,
         availability_pct = (toDouble(successful_checks) / toDouble(total_checks)) * 100.0,
         total_downtime_minutes,
         incident_count,
         mttr_minutes = if(incident_count > 0, total_downtime_minutes / incident_count, 0)
| lookup [
    fetch dt.system.files
    | filter path == "/lookups/slo_config"
    | fields content
  ], sourceField:service_id, lookupField:service_id
| fields service_id,
         team = coalesce(lookup.team, "Unknown"),
         criticality = coalesce(lookup.criticality, "Medium"),
         availability_pct,
         total_downtime_minutes,
         incident_count,
         mttr_minutes,
         slo_target = coalesce(lookup.custom_slo_target, 99.0),
         impact_score = if(criticality == "High", total_downtime_minutes * 3,
                          if(criticality == "Medium", total_downtime_minutes * 2, 
                             total_downtime_minutes))
| sort impact_score desc
```

### Real-Time SLO Burn Rate Monitor

```dql
// Monitor SLO burn rate with different thresholds per service
timeseries 
  successful_requests = sum(if(http.response_status_code < 400, 1, 0)),
  total_requests = count(),
  by: {service.name}, interval:5m
| fields 
    service_id = service.name,
    timestamp,
    availability = if(total_requests > 0, 
                     toDouble(successful_requests) / toDouble(total_requests) * 100.0, 
                     100.0)
| lookup [
    fetch dt.system.files
    | filter path == "/lookups/slo_config"
    | fields content
  ], sourceField:service_id, lookupField:service_id
| fields service_id,
         timestamp,
         availability,
         slo_target = coalesce(lookup.custom_slo_target, 99.0),
         team = coalesce(lookup.team, "Unknown"),
         criticality = coalesce(lookup.criticality, "Medium")
| fields service_id,
         timestamp,
         team,
         criticality,
         availability,
         slo_target,
         error_budget_consumed = max(0, slo_target - availability),
         burn_rate = if(slo_target > 0, 
                       error_budget_consumed / (100.0 - slo_target) * 100.0, 
                       0),
         burn_rate_status = if(burn_rate > 50, "FAST_BURN",
                              if(burn_rate > 20, "MODERATE_BURN",
                                 if(burn_rate > 5, "SLOW_BURN", "HEALTHY")))
| filter burn_rate_status != "HEALTHY"
| sort timestamp desc, burn_rate desc
```

## 7. Best Practices and Troubleshooting

### Best Practices
- **File Organization**: Use logical structures like `/lookups/team/purpose/filename`
- **Size Limits**: Keep files under 100 MB
- **Deduplication**: Use `lookupField` parameter to ensure unique records
- **Permissions**: Implement least-privilege access with path restrictions
- **Documentation**: Maintain ownership and purpose documentation for each lookup table
- **Version Control**: Consider including version numbers in filenames if needed

### Common Issues and Solutions

| Issue | Solution |
|-------|----------|
| Pattern matching failures | Test DPL pattern with test endpoint first; check for header lines |
| Permission denied | Verify `storage:files:*` permissions include the specific path |
| File not found | Use `fetch dt.system.files` to verify exact path and case sensitivity |
| Upload failures | Ensure file < 100 MB, path starts with `/lookups/`, use `overwrite` for updates |
| Null values after join | Use `coalesce()` to handle services without lookup entries |

## 7. Dashboard Variable Configuration

Add these dashboard variables to support fallback values:

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `Services` | DQL, multi-select | — | Service selection |
| `TargetPct` | Free text/List | `98` | Fallback SLO target |
| `WarningPct` | Free text/List | `99` | Fallback warning threshold |

## 8. Complete Integration Example

Here's how all pieces work together in a comprehensive dashboard query:

```dql
// mod_status_table with lookup enrichment
timeseries { total = sum(dt.service.request.count) },
  by: { dt.entity.service },
  filter: { in(dt.entity.service, array($Services)) }
| join [
    timeseries { failed = sum(dt.service.request.count, default: 0.0) },
      by: { dt.entity.service },
      filter: { failed == true and in(dt.entity.service, array($Services)) },
      nonempty: true
  ],
  kind: leftOuter,
  on: { dt.entity.service, timeframe, interval },
  prefix: "err."
| fieldsAdd 
    total = arraySum(total),
    err_failed = arraySum(coalesce(err.failed, 0.0))
| summarize {
    total_period = sum(total),
    failed_period = sum(err_failed)
  }, by: { dt.entity.service }
| lookup [ load "/lookups/slo_config" ],
    sourceField: dt.entity.service,
    lookupField: service_id
| fieldsAdd 
    service = entityName(dt.entity.service),
    availability_pct = if(total_period > 0, 
                         (total_period - failed_period) / total_period * 100,
                         else: 100.0),
    // Use lookup values if available, otherwise fall back to dashboard variables
    target_pct = coalesce(custom_slo_target, toDouble($TargetPct)),
    warning_pct = coalesce(warning_threshold, toDouble($WarningPct)),
    team = coalesce(team, "Unknown"),
    criticality = coalesce(criticality, "Medium")
| fieldsAdd 
    eb_used_pct = if(target_pct < 100, 
                     ((100 - availability_pct) / (100 - target_pct)) * 100,
                     else: 0),
    eb_remaining_pct = if(eb_used_pct < 0, 100, 
                         else: if(eb_used_pct > 100, 0, 
                         else: 100 - eb_used_pct))
| fieldsAdd state = if(availability_pct < target_pct, "❌ Fail", 
                      else: if(availability_pct < warning_pct, "⚠️ Warn", 
                      else: "✅ OK"))
| fields service, team, criticality, target_pct, warning_pct, 
         availability_pct, eb_remaining_pct, state
| sort criticality desc, availability_pct asc
```

## 9. Next Steps

1. **Create and upload** the initial lookup file with your service configurations
2. **Test integration** with a single module first (recommend starting with `mod_status_table`)
3. **Update all modules** to support optional lookup enrichment
4. **Add filtering capabilities** based on team or criticality from lookup data
5. **Consider automation** for updating lookup files via CI/CD pipeline
6. **Document** the lookup schema for team members maintaining configurations

This plan enables scalable per-service configuration for availability dashboards using Dynatrace Grail lookup tables, providing flexibility while maintaining dashboard simplicity.