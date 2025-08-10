# Plan for `mod_lookup_join` Per-Service Lookup

This document outlines how to enrich availability SLO queries with per-service metadata using Dynatrace Grail lookup tables.

## 1. Prepare Lookup Data
- Build a table that defines service-level overrides (for example: `service_id`, `custom_slo_target`, `warning_threshold`, `team`, `criticality`).
- Store the file under the `/lookups` prefix; file paths must begin with `/`, contain only alphanumeric characters plus `-`, `_`, `.`, and `/`, and follow structured naming like `/lookups/slo_config`【F:DYNATRACE_DQL_LookupDataInGrail_DOC.md†L15-L45】

## 2. Configure Access
- Dashboards need `storage:files:read` to consume the lookup data; maintenance operations additionally require `storage:files:write` and `storage:files:delete`【F:DYNATRACE_DQL_LookupDataInGrail_DOC.md†L48-L89】

## 3. Upload and Maintain Lookup Files
1. **Test parsing** with the Resource Store API (`lookup:test-pattern`).
2. **Upload** data using `lookup:upload`; specify `parsePattern`, `lookupField`, and the target `filePath`.
3. **Delete or overwrite** files as needed (`files:delete`).
API endpoints and sample payloads are provided in the Lookup Data in Grail documentation【F:DYNATRACE_DQL_LookupDataInGrail_DOC.md†L103-L205】

## 4. Validate Uploaded Data
- List stored files with `fetch dt.system.files | filter filePath startsWith "/lookups/"`.
- Inspect contents via `load "/lookups/slo_config"`【F:DYNATRACE_DQL_LookupDataInGrail_DOC.md†L219-L245】

## 5. Join Lookup Data in SLO Queries
Embed the lookup into any of the availability SLO modules:

```dql
... // output from a prior module
| lookup [ load "/lookups/slo_config" ],
    sourceField: dt.entity.service,
    lookupField: service_id
| fieldsAdd target_pct = if(isNotNull(custom_slo_target),
                            custom_slo_target,
                            else: toDouble($default_target))
```
This extends the optional `mod_lookup_join` step referenced in the dashboard DQL file【F:AVAILABILITY_DASHBOARD_DQL.md†L129-L135】【F:DYNATRACE_DQL_LookupDataInGrail_DOC.md†L259-L276】

## 6. Best Practices and Troubleshooting
- Organize lookup files logically, keep them under 100 MB, and deduplicate records via `lookupField`.
- Limit permissions by path, audit access, and document ownership.
- For common issues (pattern errors, permission denied, file not found), use the troubleshooting tips in the reference documentation【F:DYNATRACE_DQL_LookupDataInGrail_DOC.md†L311-L356】

## 7. Next Steps for Dashboard Integration
- Add variables such as `default_target` to handle services without overrides.
- Combine with modules like `mod_status_table` and `mod_burn_rate` to present enriched SLO data.

This plan enables scalable per-service configuration for availability dashboards using Dynatrace Grail lookup tables.
