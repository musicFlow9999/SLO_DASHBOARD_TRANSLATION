# Plan for `mod_lookup_join` Per-Service Lookup

This document outlines how to enrich availability SLO queries with per-service metadata using Dynatrace Grail lookup tables.

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

## 6. Best Practices and Troubleshooting

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