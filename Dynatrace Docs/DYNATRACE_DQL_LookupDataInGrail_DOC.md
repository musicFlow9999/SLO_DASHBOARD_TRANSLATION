# Lookup Data in Grail - Complete Documentation

## Overview

**Status: Preview**

Storing lookup data in Grail enables you to enrich your observability data with your custom data. You can upload your lookup data and join it with your existing data in Grail, such as:
- Logs
- Events
- Spans
- Metrics

Dynatrace stores lookup data as tabular files in the Resource Store, which is part of Grail. You can upload and manage your lookup data through the Resource Store API. Once stored in Grail, you can use your lookup files to enrich your data within DQL queries.

## File Path Conventions

Static data, such as lookup tables, can be permanently stored in Grail as files. Files are referenced via a file path such as `/lookups/http_status_codes`.

### File Path Requirements

File paths must follow these conventions:

1. **Must contain only**:
   - Alphanumeric characters (`a-zA-Z0-9`)
   - Dash (`-`)
   - Underscore (`_`)
   - Period (`.`)
   - Forward slash (`/`)

2. **Structure requirements**:
   - Must start with `/`
   - Must end with `a-zA-Z0-9`
   - Must contain at least two `/` characters
   - Between any two consecutive `/` characters, there must be at least one alphanumeric character

3. **Lookup data specific**:
   - The file path must start with `/lookups` when storing lookup data in Grail

### Organization Best Practices

The naming conventions allow you to organize your files like a regular file system. Using prefixes that mimic folders, such as:
- `/lookups/my_team/allow_list`
- `/lookups/slo_config`
- `/lookups/error_codes/http`

This makes it convenient to find and manage your lookup files stored in Grail.

## Permissions

### Required Permissions

#### To Read Lookup Data
```
storage:files:read
```

#### To Upload/Delete Lookup Data via REST API
```
storage:files:write
storage:files:delete
```

### Permission Configuration

All permissions can be restricted to specific paths or prefixes, giving users access to only a limited set of files.

**Important Notes:**
- When creating an OAuth token or platform token for API calls, ensure these permissions are configured for the token
- The user linked to that OAuth token or platform token must have these permissions assigned

### Preview Phase Access

Customers with Dynatrace Platform Subscription (DPS) can join the preview for lookup data in Grail. During the preview phase:
- The `storage:files` permissions are not included in the default Grail policies
- You can opt into the preview program by manually adding permissions to access lookup files to your custom policies

### Example Permission Policies

#### Full Access to All Lookup Data
```
ALLOW storage:files:read WHERE storage:file-path startsWith "/lookups/";
ALLOW storage:files:write WHERE storage:file-path startsWith "/lookups/";
ALLOW storage:files:delete WHERE storage:file-path startsWith "/lookups/";
```

#### Read-Only Access to All Lookup Data
```
ALLOW storage:files:read WHERE storage:file-path startsWith "/lookups/";
```

## Limits

The following limits apply for storing lookup data in Grail:

| Limit Type | Value |
|------------|-------|
| Maximum number of files per environment | 100 (during preview) |
| Maximum file size | 100 MB |
| Maximum number of columns per file | 128 |

**Note:** Upon completion of the preview and lifting the maximum number of files that can be stored per environment, the usage of lookup data in Grail can generate Events powered by Grail consumption.

## Resource Store API

### Available Operations

You can manage your lookup files in Grail via the Resource Store API. Dynatrace offers API calls to:

1. **Test parsing** your lookup data without storing it
2. **Upload** your lookup data to Grail
3. **Delete** your lookup data from Grail
4. **Update** a file's content (requires reuploading the whole file)

### API Endpoints

| Operation | Endpoint | Description |
|-----------|----------|-------------|
| Test Parsing | `POST /platform/storage/resource-store/v1/files/tabular/lookup:test-pattern` | Test parsing your data without storing the result in Grail |
| Upload | `POST /platform/storage/resource-store/v1/files/tabular/lookup:upload` | Upload and store lookup data as a new tabular file or replace existing |
| Delete | `POST /platform/storage/resource-store/v1/files:delete` | Delete a file from the Resource Store |

### Parse Lookup Data

The Resource Store API uses the Dynatrace Pattern Language (DPL) to parse uploaded data and convert it into a tabular storage format. This supports various text-based formats:
- CSV
- JSONL
- XML

#### Example: CSV Format

**Data:**
```csv
Code,Category,Message
100,informational,Continue
101,informational,Switching Protocols
```

**DPL Pattern:**
```
INT:code ',' LD:category ',' LD:message
```

#### Example: JSONL Format

**Data:**
```json
{"code": 100, "category": "informational", "message": "Continue"}
{"code": 101, "category": "informational", "message": "Switching Protocols"}
```

**DPL Pattern:**
```
JSON:json
```

### Test Pattern API Call

Example curl command for testing a DPL pattern:

```bash
curl -X 'POST' \
  'https://<environment>.apps.dynatrace.com/platform/storage/resource-store/v1/files/tabular/lookup:test-pattern' \
  -H 'accept: */*' \
  -H 'Content-Type: multipart/form-data' \
  -H 'Authorization: Bearer <platformtoken>' \
  -F 'request={"parsePattern":"JSON:json","lookupField":"code"}' \
  -F 'content=@http_status_codes.jsonl'
```

The response includes:
- Number of records that matched the pattern
- Preview of up to 100 records

### Upload Lookup Data API Call

Required parameters for upload:
- `parsePattern`: DPL pattern to parse the uploaded data
- `lookupField`: Field with the identifier of the record
- `filePath`: Fully qualified file path to store the lookup data

Optional parameters:
- `displayName`: Display name for the file
- `description`: Description of the lookup data
- `overwrite`: Use when updating existing files

Example curl command for uploading:

```bash
curl -X 'POST' \
  'https://<environment>.apps.dynatrace.com/platform/storage/resource-store/v1/files/tabular/lookup:upload' \
  -H 'accept: */*' \
  -H 'Content-Type: multipart/form-data' \
  -H 'Authorization: Bearer <platformtoken>' \
  -F 'request={
      "parsePattern":"JSON:json",
      "lookupField":"code",
      "filePath":"/lookups/http_status_codes",
      "displayName":"My lookup data",
      "description":"Description of my lookup data"
    }' \
  -F 'content=@http_status_codes.jsonl'
```

### Delete Lookup Data API Call

Example curl command for deleting:

```bash
curl -X 'POST' \
  'https://<environment>.apps.dynatrace.com/platform/storage/resource-store/v1/files:delete' \
  -H 'accept: */*' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <platformtoken>' \
  -d '{"filePath": "/lookups/http_status_codes"}'
```

**Warning:** Deleting a file is irreversible.

## Using Lookup Data in DQL

### List All Files

Get a list of all accessible files stored in Grail:

```dql
fetch dt.system.files
```

### Search for Specific Files

Add search or filter commands:

```dql
fetch dt.system.files
| filter filePath startsWith "/lookups/"
```

### Inspect File Contents

Use the `load` command to view the contents of a file:

```dql
load "/lookups/http_status_codes"
```

### Enrich Observability Data

Use the `load` command with `lookup` or `join` to add context to your observability data:

```dql
fetch spans
| lookup [ load "/lookups/http_status_codes" ],
    sourceField: http.response.status_code,
    lookupField: code
```

## Practical Use Cases

### 1. Service-Specific SLO Configuration

```csv
service_id,custom_slo_target,warning_threshold,team,criticality
SERVICE-ABC123,99.9,99.95,Platform,High
SERVICE-DEF456,95.0,97.0,Backend,Medium
```

**Usage in DQL:**
```dql
// Your service metrics query
| lookup [ load "/lookups/slo_config" ],
    sourceField: dt.entity.service,
    lookupField: service_id
| fieldsAdd target_pct = if(isNotNull(custom_slo_target), 
                            custom_slo_target, 
                            else: toDouble($default_target))
```

### 2. Error Code Mapping

```json
{"code": 404, "category": "client_error", "message": "Not Found", "severity": "low"}
{"code": 500, "category": "server_error", "message": "Internal Server Error", "severity": "high"}
```

**Usage in DQL:**
```dql
fetch logs
| lookup [ load "/lookups/error_codes" ],
    sourceField: status_code,
    lookupField: code
| fields timestamp, status_code, category, message, severity
```

### 3. IP Address Enrichment

```csv
ip_address,location,risk_level,last_seen
192.168.1.100,internal,low,2025-01-15
10.0.0.50,dmz,medium,2025-01-14
```

**Usage in DQL:**
```dql
fetch events
| lookup [ load "/lookups/ip_metadata" ],
    sourceField: client_ip,
    lookupField: ip_address
| filter risk_level == "high"
```

## Best Practices

1. **File Organization**
   - Use logical folder structures: `/lookups/team/purpose/filename`
   - Keep related lookup tables in the same "folder"

2. **File Naming**
   - Use descriptive names that indicate the content
   - Include version numbers if needed: `/lookups/config_v2`

3. **Data Management**
   - Keep lookup files under the 100 MB limit
   - Use the `lookupField` parameter to deduplicate records
   - Regularly review and clean up unused lookup files

4. **Performance**
   - Use specific fields in lookup operations rather than entire records
   - Consider the size of lookup tables when designing queries
   - Test patterns before uploading large datasets

5. **Security**
   - Implement least-privilege access using specific path restrictions
   - Regularly audit who has write/delete permissions
   - Document the purpose and ownership of each lookup table

## Troubleshooting

### Common Issues

1. **Pattern Matching Failures**
   - Test your DPL pattern using the test endpoint first
   - Check for header lines that need to be skipped using `skippedRecords`

2. **Permission Denied**
   - Verify your user group has the required `storage:files:*` permissions
   - Check that permissions include the specific file path prefix

3. **File Not Found**
   - Use `fetch dt.system.files` to list available files
   - Verify the exact file path including case sensitivity

4. **Upload Failures**
   - Ensure file size is under 100 MB
   - Check that the file path starts with `/lookups/`
   - Use the `overwrite` parameter when updating existing files

## Summary

Lookup data in Grail provides a powerful way to enrich your observability data with custom context. By following the conventions and best practices outlined in this documentation, you can effectively manage and utilize lookup tables to enhance your monitoring and analysis capabilities in Dynatrace.