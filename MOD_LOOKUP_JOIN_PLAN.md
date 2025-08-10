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
   - Navigate to: **Identity & access management > OAuth clients**
   - Select **Create client**
   - Provide service user email and meaningful description
   - Select required permissions (scopes below)

2. **Required Scopes**: 
   - `storage:files:read` - Read lookup data in DQL queries
   - `storage:files:write` - Create or modify lookup files  
   - `storage:files:delete` - Remove lookup files

3. **Save Credentials Securely**
   - **Critical**: Client secret is only shown once during creation
   - Copy and store Client ID and Client Secret in a secure password manager
   - Use separate clients for different applications/integrations

4. **Obtain Bearer Token for API Calls**:
   ```bash
   # Request OAuth token from Dynatrace SSO
   curl --request POST \
     --url 'https://sso.dynatrace.com/sso/oauth2/token' \
     --header 'Content-Type: application/x-www-form-urlencoded' \
     --data-urlencode 'grant_type=client_credentials' \
     --data-urlencode 'client_id=YOUR_CLIENT_ID' \
     --data-urlencode 'client_secret=YOUR_CLIENT_SECRET_HERE' \
     --data-urlencode 'scope=storage:files:read storage:files:write storage:files:delete' \
     --data-urlencode 'resource=urn:dtaccount:YOUR_ACCOUNT_UUID'
   ```
   
   Response contains the bearer token:
   ```json
   {
     "access_token": "your_bearer_token_value_here",
     "token_type": "Bearer", 
     "expires_in": 3600,
     "scope": "storage:files:read storage:files:write storage:files:delete"
   }
   ```

5. **Service User Setup**:
   - Designate a specific user as service user for API access only
   - Ensure service user belongs to groups with appropriate permissions
   - Follow principle of least privilege for scope assignments

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

## 11. Python Automation with OAuth Integration

### Complete Lookup Manager with OAuth Support

```python
import requests
import json
import time
from typing import Dict, List, Any, Optional
import logging
from urllib.parse import urlencode

class DynatraceOAuthLookupManager:
    """
    Comprehensive manager for Dynatrace Grail lookup tables with OAuth authentication.
    Handles token management, automatic refresh, and complete CRUD operations.
    """
    
    def __init__(self, environment_url: str, client_id: str, client_secret: str, account_uuid: str):
        """
        Initialize the OAuth-enabled lookup manager.
        
        Args:
            environment_url: Base URL for your Dynatrace environment
            client_id: OAuth client ID
            client_secret: OAuth client secret  
            account_uuid: Dynatrace account UUID for resource URN
        """
        self.environment_url = environment_url.rstrip('/')
        self.client_id = client_id
        self.client_secret = client_secret
        self.account_uuid = account_uuid
        self.base_url = f"{self.environment_url}/platform/storage/resource-store/v1/files/tabular"
        self.sso_url = "https://sso.dynatrace.com/sso/oauth2/token"
        self.access_token = None
        self.token_expires_at = 0
        self.logger = logging.getLogger(__name__)

    def _get_access_token(self) -> str:
        """
        Get or refresh OAuth access token.
        
        Returns:
            Valid access token
        """
        # Check if current token is still valid (with 60 second buffer)
        if self.access_token and time.time() < (self.token_expires_at - 60):
            return self.access_token
            
        # Request new token
        token_data = {
            'grant_type': 'client_credentials',
            'client_id': self.client_id,
            'client_secret': self.client_secret,
            'scope': 'storage:files:read storage:files:write storage:files:delete',
            'resource': f'urn:dtaccount:{self.account_uuid}'
        }
        
        headers = {'Content-Type': 'application/x-www-form-urlencoded'}
        
        try:
            response = requests.post(
                self.sso_url, 
                headers=headers,
                data=urlencode(token_data)
            )
            response.raise_for_status()
            
            token_response = response.json()
            self.access_token = token_response['access_token']
            expires_in = token_response.get('expires_in', 3600)
            self.token_expires_at = time.time() + expires_in
            
            self.logger.info("OAuth token refreshed successfully")
            return self.access_token
            
        except requests.RequestException as e:
            self.logger.error(f"Failed to obtain OAuth token: {e}")
            raise

    def _get_headers(self) -> Dict[str, str]:
        """Get headers with fresh OAuth token."""
        token = self._get_access_token()
        return {'Authorization': f'Bearer {token}'}

    def test_lookup_pattern(self, file_path: str, parse_pattern: str, 
                          lookup_field: str, content: str) -> Dict[str, Any]:
        """
        Test the parsing pattern before uploading data.
        
        Args:
            file_path: Path for the lookup file (e.g., '/lookups/slo_config')
            parse_pattern: Pattern defining data structure
            lookup_field: Field used for lookups
            content: JSON Lines or CSV content to test
            
        Returns:
            API response with parsing results
        """
        url = f"{self.base_url}/lookup:test-pattern"
        
        files = {
            'request': (None, json.dumps({
                'parsePattern': parse_pattern,
                'lookupField': lookup_field
            }), 'application/json'),
            'content': ('test_data', content, 'text/plain')
        }
        
        try:
            response = requests.post(url, headers=self._get_headers(), files=files)
            response.raise_for_status()
            self.logger.info(f"Pattern test successful for {file_path}")
            return response.json()
        except requests.RequestException as e:
            self.logger.error(f"Pattern test failed: {e}")
            raise

    def upload_lookup_data(self, file_path: str, parse_pattern: str, 
                          lookup_field: str, content: str, 
                          display_name: str = None, description: str = None) -> Dict[str, Any]:
        """
        Upload lookup data to Dynatrace Grail.
        
        Args:
            file_path: Path for the lookup file (e.g., '/lookups/slo_config')
            parse_pattern: Pattern defining data structure
            lookup_field: Field used for lookups
            content: JSON Lines or CSV content to upload
            display_name: Optional display name for the lookup table
            description: Optional description for the lookup table
            
        Returns:
            API response from upload operation
        """
        url = f"{self.base_url}/lookup:upload"
        
        request_data = {
            'parsePattern': parse_pattern,
            'lookupField': lookup_field,
            'filePath': file_path
        }
        
        if display_name:
            request_data['displayName'] = display_name
        if description:
            request_data['description'] = description
        
        files = {
            'request': (None, json.dumps(request_data), 'application/json'),
            'content': ('lookup_data', content, 'text/plain')
        }
        
        try:
            response = requests.post(url, headers=self._get_headers(), files=files)
            response.raise_for_status()
            self.logger.info(f"Successfully uploaded lookup data to {file_path}")
            return response.json()
        except requests.RequestException as e:
            self.logger.error(f"Failed to upload lookup data: {e}")
            raise

    def delete_lookup_data(self, file_path: str) -> bool:
        """
        Delete lookup data from Dynatrace Grail.
        
        Args:
            file_path: Path of the lookup file to delete
            
        Returns:
            True if deletion was successful
        """
        url = f"{self.base_url}/lookup:delete"
        headers = {**self._get_headers(), 'Content-Type': 'application/json'}
        payload = {'filePath': file_path}
        
        try:
            response = requests.post(url, headers=headers, json=payload)
            response.raise_for_status()
            self.logger.info(f"Successfully deleted lookup data at {file_path}")
            return True
        except requests.RequestException as e:
            self.logger.error(f"Failed to delete lookup data: {e}")
            return False

    def list_lookup_files(self, prefix: str = "/lookups") -> List[Dict[str, Any]]:
        """
        List available lookup files.
        
        Args:
            prefix: Path prefix to filter files
            
        Returns:
            List of file information dictionaries
        """
        url = f"{self.environment_url}/platform/storage/resource-store/v1/files"
        headers = self._get_headers()
        params = {'prefix': prefix}
        
        try:
            response = requests.get(url, headers=headers, params=params)
            response.raise_for_status()
            return response.json().get('files', [])
        except requests.RequestException as e:
            self.logger.error(f"Failed to list lookup files: {e}")
            return []

# Utility Functions for SLO Configuration Management

def create_slo_config_jsonl(services_config: List[Dict[str, Any]]) -> str:
    """
    Create JSON Lines content for SLO configuration lookup table.
    
    Args:
        services_config: List of service configuration dictionaries
        
    Returns:
        JSON Lines string ready for upload
    """
    if not services_config:
        return ""
    
    lines = []
    for config in services_config:
        # Ensure required fields exist
        if 'service_id' not in config:
            raise ValueError("service_id is required for each configuration")
        lines.append(json.dumps(config))
    
    return '\n'.join(lines)

def validate_slo_config(config: Dict[str, Any]) -> Dict[str, Any]:
    """
    Validate and clean SLO configuration.
    
    Args:
        config: Service configuration dictionary
        
    Returns:
        Validated configuration dictionary
    """
    required_fields = ['service_id']
    for field in required_fields:
        if field not in config:
            raise ValueError(f"Required field '{field}' missing from configuration")
    
    # Set defaults for optional fields
    validated = {
        'service_id': config['service_id'],
        'custom_slo_target': config.get('custom_slo_target', 99.0),
        'warning_threshold': config.get('warning_threshold', 99.5),
        'team': config.get('team', 'Unknown'),
        'criticality': config.get('criticality', 'Medium')
    }
    
    # Validate data types and ranges
    if not (0 <= validated['custom_slo_target'] <= 100):
        raise ValueError("custom_slo_target must be between 0 and 100")
    if not (0 <= validated['warning_threshold'] <= 100):
        raise ValueError("warning_threshold must be between 0 and 100")
    
    return validated

# Example Usage
def main():
    """
    Example usage of the DynatraceOAuthLookupManager.
    """
    # Configuration (use environment variables in production)
    environment_url = "https://your-environment.apps.dynatrace.com"
    client_id = "your-oauth-client-id"
    client_secret = "your-oauth-client-secret"
    account_uuid = "your-account-uuid"
    
    # Initialize manager
    manager = DynatraceOAuthLookupManager(
        environment_url, client_id, client_secret, account_uuid
    )
    
    # Sample SLO configuration data
    slo_configs = [
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
    
    try:
        # Validate configurations
        validated_configs = [validate_slo_config(config) for config in slo_configs]
        
        # Create JSON Lines content
        jsonl_content = create_slo_config_jsonl(validated_configs)
        
        # Upload parameters
        file_path = "/lookups/slo_config"
        parse_pattern = "JSON:json"
        lookup_field = "service_id"
        
        # Test the pattern first
        print("Testing parse pattern...")
        test_result = manager.test_lookup_pattern(
            file_path, parse_pattern, lookup_field, jsonl_content
        )
        print(f"Test successful: {test_result.get('message', 'Pattern validated')}")
        
        # Upload the data
        print("Uploading lookup data...")
        upload_result = manager.upload_lookup_data(
            file_path, parse_pattern, lookup_field, jsonl_content,
            display_name="SLO Configuration",
            description="Per-service SLO targets and team metadata"
        )
        print(f"Upload successful: {upload_result.get('message', 'Data uploaded')}")
        
        # List available files
        print("Available lookup files:")
        files = manager.list_lookup_files()
        for file_info in files:
            print(f"- {file_info.get('path', 'Unknown path')}")
            
    except Exception as e:
        print(f"Operation failed: {e}")

if __name__ == "__main__":
    main()
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