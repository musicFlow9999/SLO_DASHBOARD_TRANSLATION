# Plan for `mod_lookup_join` Per-Service Lookup

This document outlines how to enrich availability SLO queries with per-service metadata using Dynatrace Grail lookup tables.

## 1. Prepare Lookup Data

Build a table that defines service-level overrides. JSON Lines format is recommended for its simplicity and type preservation.

### Recommended: JSON Lines Format (`slo_config.jsonl`)
```json
{"service_id":"SERVICE-ABC123","custom_slo_target":99.9,"warning_threshold":99.95,"team":"Platform","criticality":"High"}
{"service_id":"SERVICE-DEF456","custom_slo_target":95.0,"warning_threshold":97.0,"team":"Backend","criticality":"Medium"}
{"service_id":"SERVICE-GHI789","custom_slo_target":99.0,"warning_threshold":99.5,"team":"Frontend","criticality":"High"}
{"service_id":"SERVICE-JKL012","custom_slo_target":98.0,"warning_threshold":99.0,"team":"Data","criticality":"Low"}
{"service_id":"SERVICE-MNO345","custom_slo_target":99.5,"warning_threshold":99.7,"team":"Platform","criticality":"Critical"}
{"service_id":"SERVICE-PQR678","custom_slo_target":97.0,"warning_threshold":98.0,"team":"Backend","criticality":"Low"}
```

**Important**: Each JSON object must be on a single line (no pretty printing).

### Alternative: CSV Format (`slo_config.csv`)
```csv
service_id,custom_slo_target,warning_threshold,team,criticality
SERVICE-ABC123,99.9,99.95,Platform,High
SERVICE-DEF456,95.0,97.0,Backend,Medium
SERVICE-GHI789,99.0,99.5,Frontend,High
SERVICE-JKL012,98.0,99.0,Data,Low
```

### File Path Requirements
- Must begin with `/lookups/` prefix
- Can only contain: alphanumeric characters (`a-zA-Z0-9`), dash (`-`), underscore (`_`), period (`.`), forward slash (`/`)
- Must start with `/` and end with alphanumeric character
- Example: `/lookups/slo_config` or `/lookups/team/slo_config_v2`

## 2. Configure Access Permissions

### Required Permissions

| Operation | Required Permission | Scope |
|-----------|-------------------|-------|
| Dashboard consumption | `storage:files:read` | Read lookup data in DQL queries |
| Upload/Update files | `storage:files:write` | Create or modify lookup files |
| Delete files | `storage:files:delete` | Remove lookup files |

### Example Permission Policy
```
ALLOW storage:files:read WHERE storage:file-path startsWith "/lookups/";
ALLOW storage:files:write WHERE storage:file-path startsWith "/lookups/";
ALLOW storage:files:delete WHERE storage:file-path startsWith "/lookups/";
```

### Setting Up OAuth Client for API Access

1. **Create OAuth Client** (Account Management > Identity & access management > OAuth clients)
2. **Required Scopes**: 
   - `storage:files:read`
   - `storage:files:write` 
   - `storage:files:delete`
3. **Save credentials securely** - client secret is only shown once

## 3. Upload Lookup Files - Complete Workflow

### Step 1: Test Parsing (Validate Without Storing)

#### For JSON Lines:
```bash
curl -X 'POST' \
  'https://your-environment.apps.dynatrace.com/platform/storage/resource-store/v1/files/tabular/lookup:test-pattern' \
  -H 'Authorization: Bearer YOUR_PLATFORM_TOKEN' \
  -H 'Content-Type: multipart/form-data' \
  -F 'request={
      "parsePattern":"JSON:json",
      "lookupField":"service_id"
    }' \
  -F 'content=@slo_config.jsonl'
```

#### For CSV:
```bash
curl -X 'POST' \
  'https://your-environment.apps.dynatrace.com/platform/storage/resource-store/v1/files/tabular/lookup:test-pattern' \
  -H 'Authorization: Bearer YOUR_PLATFORM_TOKEN' \
  -H 'Content-Type: multipart/form-data' \
  -F 'request={
      "parsePattern":"LD:service_id \",\" DOUBLE:custom_slo_target \",\" DOUBLE:warning_threshold \",\" LD:team \",\" LD:criticality",
      "lookupField":"service_id",
      "skippedRecords":1
    }' \
  -F 'content=@slo_config.csv'
```

#### Expected Test Response:
```json
{
  "matchedRecords": 6,
  "preview": [
    {
      "service_id": "SERVICE-ABC123",
      "custom_slo_target": 99.9,
      "warning_threshold": 99.95,
      "team": "Platform",
      "criticality": "High"
    },
    {
      "service_id": "SERVICE-DEF456",
      "custom_slo_target": 95.0,
      "warning_threshold": 97.0,
      "team": "Backend",
      "criticality": "Medium"
    }
  ]
}
```

### Step 2: Upload Data (Store in Grail)

#### For JSON Lines (Recommended):
```bash
curl -X 'POST' \
  'https://your-environment.apps.dynatrace.com/platform/storage/resource-store/v1/files/tabular/lookup:upload' \
  -H 'Authorization: Bearer YOUR_PLATFORM_TOKEN' \
  -H 'Content-Type: multipart/form-data' \
  -F 'request={
      "parsePattern":"JSON:json",
      "lookupField":"service_id",
      "filePath":"/lookups/slo_config",
      "displayName":"SLO Configuration",
      "description":"Per-service SLO targets and team metadata"
    }' \
  -F 'content=@slo_config.jsonl'
```

#### For CSV:
```bash
curl -X 'POST' \
  'https://your-environment.apps.dynatrace.com/platform/storage/resource-store/v1/files/tabular/lookup:upload' \
  -H 'Authorization: Bearer YOUR_PLATFORM_TOKEN' \
  -H 'Content-Type: multipart/form-data' \
  -F 'request={
      "parsePattern":"LD:service_id \",\" DOUBLE:custom_slo_target \",\" DOUBLE:warning_threshold \",\" LD:team \",\" LD:criticality",
      "lookupField":"service_id",
      "filePath":"/lookups/slo_config",
      "displayName":"SLO Configuration",
      "description":"Per-service SLO targets and team metadata",
      "skippedRecords":1
    }' \
  -F 'content=@slo_config.csv'
```

### Step 3: Update Existing File
Add `"overwrite": true` to the request:
```bash
curl -X 'POST' \
  'https://your-environment.apps.dynatrace.com/platform/storage/resource-store/v1/files/tabular/lookup:upload' \
  -H 'Authorization: Bearer YOUR_PLATFORM_TOKEN' \
  -H 'Content-Type: multipart/form-data' \
  -F 'request={
      "parsePattern":"JSON:json",
      "lookupField":"service_id",
      "filePath":"/lookups/slo_config",
      "displayName":"SLO Configuration",
      "description":"Updated per-service SLO targets - v2",
      "overwrite": true
    }' \
  -F 'content=@slo_config_v2.jsonl'
```

### Step 4: Delete File (if needed)
```bash
curl -X 'POST' \
  'https://your-environment.apps.dynatrace.com/platform/storage/resource-store/v1/files:delete' \
  -H 'Authorization: Bearer YOUR_PLATFORM_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"filePath": "/lookups/slo_config"}'
```

## 4. Validate Uploaded Data

### List all lookup files:
```dql
fetch dt.system.files 
| filter filePath startsWith "/lookups/"
| fields filePath, displayName, description, size, recordCount
```

### Inspect uploaded contents:
```dql
load "/lookups/slo_config"
```

Expected output:
| service_id | custom_slo_target | warning_threshold | team | criticality |
|------------|-------------------|-------------------|------|-------------|
| SERVICE-ABC123 | 99.9 | 99.95 | Platform | High |
| SERVICE-DEF456 | 95.0 | 97.0 | Backend | Medium |
| SERVICE-GHI789 | 99.0 | 99.5 | Frontend | High |

## 5. DQL Integration - Complete Examples

### Basic Lookup Integration
```dql
// Any query output...
| lookup [ load "/lookups/slo_config" ],
    sourceField: dt.entity.service,
    lookupField: service_id
| fieldsAdd 
    target_pct = coalesce(custom_slo_target, toDouble($TargetPct)),
    warning_pct = coalesce(warning_threshold, toDouble($WarningPct)),
    team = coalesce(team, "Unknown"),
    criticality = coalesce(criticality, "Medium")
```

### Integration with `mod_status_table`
```dql
timeseries { total = sum(dt.service.request.count) },
  by: { dt.entity.service },
  filter: { in(dt.entity.service, array($services)) }
| join [
    timeseries { failed = sum(dt.service.request.count, default: 0.0) },
      by: { dt.entity.service },
      filter: { failed == true and in(dt.entity.service, array($services)) },
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
| fieldsAdd 
    total = arraySum(total),
    err_failed = arraySum(err.failed)
| summarize { 
    total_w = sum(total), 
    failed_w = sum(err_failed) 
  }, by: { dt.entity.service }
| lookup [ load "/lookups/slo_config" ],
    sourceField: dt.entity.service,
    lookupField: service_id
| fieldsAdd 
    service = entityName(dt.entity.service),
    target_pct = coalesce(custom_slo_target, toDouble($TargetPct)),
    team = coalesce(team, "Unknown"),
    criticality = coalesce(criticality, "Medium")
| fieldsAdd error_rate = if(total_w == 0, 0.0, else: failed_w / total_w)
| fieldsAdd burn_rate = error_rate / ((100 - target_pct) / 100.0)
| fieldsAdd burn_alert = if(criticality == "Critical" AND burn_rate > 10, "🚨 CRITICAL",
                           else: if(burn_rate > 14.4, "⚠️ HIGH",
                           else: if(burn_rate > 6, "⚡ MODERATE", "✅ NORMAL")))
| fields service, team, criticality, target_pct, burn_rate, burn_alert
| sort burn_rate desc
```

## 6. Advanced Use Cases

### Team Performance Dashboard
```dql
timeseries { total = sum(dt.service.request.count) },
  by: { dt.entity.service }
| join [
    timeseries { failed = sum(dt.service.request.count, default: 0.0) },
      by: { dt.entity.service },
      filter: { failed == true }
  ],
  kind: leftOuter,
  on: { dt.entity.service, timeframe, interval },
  prefix: "err."
| fieldsAdd 
    total_sum = arraySum(total),
    failed_sum = arraySum(coalesce(err.failed, 0.0))
| summarize {
    total_requests = sum(total_sum),
    failed_requests = sum(failed_sum)
  }, by: { dt.entity.service }
| fieldsAdd availability_pct = if(total_requests > 0,
                                  (total_requests - failed_requests) / total_requests * 100,
                                  else: 100.0)
| lookup [ load "/lookups/slo_config" ],
    sourceField: dt.entity.service,
    lookupField: service_id
| fieldsAdd 
    team = coalesce(team, "Unknown"),
    criticality = coalesce(criticality, "Medium"),
    target_pct = coalesce(custom_slo_target, 99.0),
    slo_met = availability_pct >= target_pct
| summarize {
    services_count = count(),
    avg_availability = avg(availability_pct),
    services_meeting_slo = countIf(slo_met == true),
    critical_services = countIf(criticality == "Critical" OR criticality == "High")
  }, by: { team }
| fieldsAdd slo_compliance_rate = if(services_count > 0,
                                     (toDouble(services_meeting_slo) / toDouble(services_count)) * 100.0,
                                     else: 0.0)
| fields team, services_count, avg_availability, slo_compliance_rate, critical_services
| sort slo_compliance_rate desc
```

### Criticality-Based Alerting
```dql
timeseries { total = sum(dt.service.request.count) },
  by: { dt.entity.service },
  from: now() - 1h
| join [
    timeseries { failed = sum(dt.service.request.count, default: 0.0) },
      by: { dt.entity.service },
      filter: { failed == true },
      from: now() - 1h
  ],
  kind: leftOuter,
  on: { dt.entity.service, timeframe, interval },
  prefix: "err."
| fieldsAdd 
    total_1h = arraySum(total),
    failed_1h = arraySum(coalesce(err.failed, 0.0))
| summarize {
    total = sum(total_1h),
    failed = sum(failed_1h)
  }, by: { dt.entity.service }
| fieldsAdd current_availability = if(total > 0,
                                      (total - failed) / total * 100.0,
                                      else: 100.0)
| lookup [ load "/lookups/slo_config" ],
    sourceField: dt.entity.service,
    lookupField: service_id
| fieldsAdd 
    service = entityName(dt.entity.service),
    target_pct = coalesce(custom_slo_target, 99.0),
    team = coalesce(team, "Unknown"),
    criticality = coalesce(criticality, "Medium")
| fieldsAdd 
    deviation = target_pct - current_availability,
    alert_level = if(current_availability < target_pct,
                     if(criticality == "Critical", "PAGE",
                        else: if(criticality == "High", "ALERT",
                        else: if(criticality == "Medium", "WARN", "INFO"))),
                     else: "OK")
| filter alert_level != "OK"
| fields service, team, criticality, current_availability, target_pct, deviation, alert_level
| sort alert_level asc, deviation desc
```

## 7. Working with Complex JSON Structures

### Nested JSON Example (`slo_config_nested.jsonl`):
```json
{"service_id":"SERVICE-ABC123","slo":{"target":99.9,"warning":99.95},"metadata":{"team":"Platform","criticality":"High","owner":"john.doe@company.com","cost_center":"CC-100"}}
{"service_id":"SERVICE-DEF456","slo":{"target":95.0,"warning":97.0},"metadata":{"team":"Backend","criticality":"Medium","owner":"jane.smith@company.com","cost_center":"CC-200"}}
```

### Upload Nested JSON:
```bash
curl -X 'POST' \
  'https://your-environment.apps.dynatrace.com/platform/storage/resource-store/v1/files/tabular/lookup:upload' \
  -H 'Authorization: Bearer YOUR_PLATFORM_TOKEN' \
  -H 'Content-Type: multipart/form-data' \
  -F 'request={
      "parsePattern":"JSON:json",
      "lookupField":"service_id",
      "filePath":"/lookups/slo_config_nested",
      "displayName":"Nested SLO Configuration",
      "description":"Complex JSON structure with nested SLO and metadata"
    }' \
  -F 'content=@slo_config_nested.jsonl'
```

### Using Nested Fields in DQL:
```dql
// Access nested fields after lookup
| lookup [ load "/lookups/slo_config_nested" ],
    sourceField: dt.entity.service,
    lookupField: service_id
| fieldsAdd 
    target_pct = coalesce(json.slo.target, toDouble($TargetPct)),
    warning_pct = coalesce(json.slo.warning, toDouble($WarningPct)),
    team = coalesce(json.metadata.team, "Unknown"),
    owner = coalesce(json.metadata.owner, "unassigned"),
    cost_center = coalesce(json.metadata.cost_center, "N/A")
```

## 8. Best Practices

### File Format Selection

| Criteria | JSON Lines | CSV |
|----------|------------|-----|
| **Parse Pattern Complexity** | Simple: `JSON:json` | Complex: field-by-field with delimiters |
| **Type Preservation** | ✅ Automatic | ❌ Requires type specification |
| **Null Handling** | ✅ Native support | ❌ Empty strings |
| **Nested Data** | ✅ Supported | ❌ Flat only |
| **Human Readability** | ✅ Self-documenting | ⚠️ Requires header reference |
| **Excel Compatibility** | ❌ Requires conversion | ✅ Direct open/edit |

**Recommendation**: Use JSON Lines unless you need direct Excel editing capability.

### Organization Guidelines
- **File Structure**: `/lookups/{team}/{purpose}/{version}`
  - Example: `/lookups/platform/slo_config/v2`
- **Naming Convention**: Use descriptive, versioned names
- **Size Management**: Keep files under 100 MB
- **Deduplication**: Use `lookupField` to ensure unique records
- **Documentation**: Include description in upload request

### Security Best Practices
- **Least Privilege**: Grant minimum required permissions
- **Path Restrictions**: Limit access to specific `/lookups/` subdirectories
- **Audit Trail**: Document who owns and maintains each lookup table
- **Sensitive Data**: Avoid storing secrets or PII in lookup tables

## 9. Troubleshooting Guide

### Common Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **Pattern Match Failure** | Test returns 0 matched records | • Verify JSON is valid (one object per line)<br>• Check CSV delimiter escaping<br>• Use `skippedRecords` for CSV headers |
| **Permission Denied** | 403 error on API calls | • Verify OAuth token has `storage:files:*` permissions<br>• Check path starts with `/lookups/` |
| **File Not Found** | Load returns empty | • Use `fetch dt.system.files` to verify exact path<br>• Check case sensitivity |
| **Upload Failure** | 400 error on upload | • Ensure file < 100 MB<br>• Verify `filePath` format<br>• Use `overwrite: true` for updates |
| **Null After Join** | Missing enrichment data | • Use `coalesce()` for fallback values<br>• Verify `lookupField` matches data |
| **Type Mismatch** | Calculation errors | • JSON preserves types automatically<br>• For CSV, specify types in parse pattern |

### Validation Commands

```bash
# Validate JSON syntax locally
jq . slo_config.jsonl

# Count records
wc -l slo_config.jsonl

# Check file size
ls -lh slo_config.jsonl

# Test first few records
head -3 slo_config.jsonl | jq .
```

## 10. Dashboard Variable Configuration

Configure these dashboard variables for fallback values:

| Variable | Type | Default | Purpose | Example Query |
|----------|------|---------|---------|---------------|
| `Services` | DQL, multi-select | — | Service selection | See below |
| `TargetPct` | Free text/Double | `99.0` | Fallback SLO target | — |
| `WarningPct` | Free text/Double | `99.5` | Fallback warning threshold | — |
| `Team` | DQL, single-select | `All` | Filter by team | See below |

### Services Variable Query:
```dql
fetch dt.entity.service
| fields id = dt.entity.service, name = entityName(dt.entity.service)
| sort name asc
```

### Team Variable Query (from lookup):
```dql
load "/lookups/slo_config"
| fields team
| distinct team
| sort team asc
```

## 11. Complete Dashboard Integration Example

```dql
// Full integration with all features
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
// Apply team filter if not "All"
| filter $Team == "All" OR team == $Team
| fieldsAdd 
    service = entityName(dt.entity.service),
    availability_pct = if(total_period > 0, 
                         (total_period - failed_period) / total_period * 100,
                         else: 100.0),
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

## 12. Migration and Deployment Steps

1. **Prepare Data File**
   - Convert existing configurations to JSON Lines format
   - Validate JSON syntax with `jq` or online validators

2. **Test in Non-Production**
   - Upload to test environment first
   - Verify all lookups resolve correctly
   - Test dashboard queries with sample data

3. **Deploy to Production**
   - Upload lookup file using API with proper authentication
   - Update dashboard variables to include fallback values
   - Test each module integration separately

4. **Establish Maintenance Process**
   - Document update procedures
   - Set up CI/CD for automated updates
   - Create version control for lookup files

5. **Monitor and Iterate**
   - Track lookup usage in dashboards
   - Gather feedback from teams
   - Optimize query performance as needed

## Summary

This plan enables scalable per-service SLO configuration using Dynatrace Grail lookup tables. JSON Lines format is recommended for its simplicity, type safety, and native support for complex data structures. The lookup integration provides flexibility for team-specific thresholds while maintaining dashboard simplicity through fallback values.