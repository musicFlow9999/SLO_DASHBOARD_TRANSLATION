# Dynatrace OAuth Clients Documentation

> **Note**: This documentation contains only public information and example patterns for educational purposes. No actual credentials or secrets are included.

## Overview

OAuth clients provide client credentials according to the OAuth standard in Dynatrace. They are managed by Dynatrace administrators and are used to:
- Set up integrations between Dynatrace and external systems
- Automate account management
- Authenticate access to Dynatrace platform services
- Enable external applications to interact with Dynatrace APIs

## Key Concepts

### OAuth2 Standard
OAuth clients in Dynatrace follow the OAuth 2.0 standard for authentication and authorization, providing secure access to Dynatrace resources without sharing user credentials directly.

### Service Users
OAuth clients authenticate as service users who are granted specific access to APIs. It's recommended to:
- Designate any user on your account as a service user
- Use service users exclusively for API access (not for other purposes)
- Ensure service users belong to groups with appropriate permissions

### Client Credentials
- **Client ID**: A unique identifier for your OAuth client
- **Client Secret**: A confidential key that must be stored securely
- **Important**: You can only access your client secret once upon creation - it cannot be revealed afterward

## Creating an OAuth Client

### Prerequisites
- Dynatrace administrator access
- Account Management access
- Defined scope requirements for your use case

### Step-by-Step Process

1. **Navigate to OAuth Clients**
   - Go to Account Management
   - If you have multiple accounts, select the account you want to manage
   - Navigate to: **Identity & access management > OAuth clients**

2. **Create New Client**
   - Select **Create client**
   - Provide the email of the user who owns the client
   - Add a description for the new client
   - Select the required permissions (scopes)
   - Click **Create client**

3. **Save Credentials**
   - Copy the generated Client ID and Client Secret
   - Store them securely in a password manager
   - **Critical**: The client secret is only visible once and cannot be retrieved later

### Best Practices
- Create separate Client IDs for each application or integration
- Don't share clients between different applications
- Use meaningful descriptions to identify client purposes
- Implement proper secret management practices

## Requesting Bearer Tokens

After creating an OAuth2 client, you need to request a bearer token from the Dynatrace SSO system.

### Token Request Endpoint
```
POST https://sso.dynatrace.com/sso/oauth2/token
```

### Request Configuration
- **Content-Type**: `application/x-www-form-urlencoded`
- **Method**: POST

### Required Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `grant_type` | `client_credentials` | OAuth2 grant type |
| `client_id` | Your OAuth client ID | The client identifier |
| `client_secret` | Your OAuth client secret | The client secret |
| `scope` | Space-separated list of scopes | Required permissions |
| `resource` | URN format: `urn:dtaccount:YOUR_ACCOUNT_UUID` | Target resource |

**Important**: URL-encode all parameter values in the request body.

### Example Token Request

```bash
curl --request POST \
  --url 'https://sso.dynatrace.com/sso/oauth2/token' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=client_credentials' \
  --data-urlencode 'client_id=YOUR_CLIENT_ID' \
  --data-urlencode 'client_secret=YOUR_CLIENT_SECRET_HERE' \
  --data-urlencode 'scope=storage:events:write account-idm-read' \
  --data-urlencode 'resource=urn:dtaccount:YOUR_ACCOUNT_UUID'
```

### Token Response
The response contains the bearer token in JSON format:
```json
{
  "access_token": "your_bearer_token_value_here",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "storage:events:write account-idm-read"
}
```

## Using Bearer Tokens

### Authentication Header
Attach the token to the Authorization HTTP header with the Bearer realm:

```bash
--header 'Authorization: Bearer YOUR_ACCESS_TOKEN_HERE'
```

### Example API Call
```bash
curl --request GET \
  --url 'https://api.dynatrace.com/iam/v1/accounts/YOUR_ACCOUNT_ID/users' \
  --header 'Authorization: Bearer YOUR_ACCESS_TOKEN_HERE'
```

## Scopes and Permissions

### Common Scopes

#### Account Management Scopes
- `account-idm-read` - Read identity management data
- `account-idm-write` - Write identity management data
- `account-env-read` - Read environment data
- `account-env-write` - Write environment data
- `account-uac-read` - Read user access control
- `account-uac-write` - Write user access control

#### Storage Scopes
- `storage:events:read` - Read events
- `storage:events:write` - Write events
- `storage:buckets:read` - Read bucket data

#### IAM Scopes
- `iam:policies:read` - Read IAM policies
- `iam:policies:write` - Write IAM policies
- `iam:bindings:read` - Read IAM bindings
- `iam:bindings:write` - Write IAM bindings
- `iam:effective-permissions:read` - Read effective permissions

#### Automation Scopes
- `automation:workflows:read` - Read workflow configurations
- `automation:workflows:write` - Write workflow configurations
- `automation:workflows:admin` - Admin access to workflows
- `automation:calendars:read` - Read calendar data
- `automation:calendars:write` - Write calendar data
- `automation:rules:read` - Read automation rules
- `automation:rules:write` - Write automation rules

#### Document Scopes
- `document:documents:read` - Read documents
- `document:documents:write` - Write documents
- `document:documents:delete` - Delete documents
- `document:trash.documents:delete` - Delete documents from trash

### Scope Assignment
- You can assign multiple scopes to a single token
- Alternatively, generate several tokens with different access levels
- Follow your organization's security policies for best practices

## Token Format

Dynatrace uses a unique token format with three components separated by dots:

**Token Structure:**
```
[PREFIX].[PUBLIC_IDENTIFIER].[SECRET_COMPONENT]
```

**Example Format:**
- Prefix: `dt0s01`
- Public portion: `ABC123DEF456GHI789JKL012` (24 characters)
- Secret portion: `[CONFIDENTIAL_PART_NOT_SHOWN]`

### Components
1. **Prefix** (`dt0s01`): Identifies the token type
2. **Public portion** (24 characters): Public identifier
3. **Secret portion**: The confidential part of the token

### Token Prefixes
- `dt0s01` - OAuth2 Clients for Account Management and Dynatrace Apps
- Other prefixes exist for different token types and use cases

## Integration Examples

### Terraform Integration

For Dynatrace Configuration as Code via Terraform:

1. Create an OAuth client with appropriate permissions
2. Set environment variables:
   ```bash
   export DYNATRACE_CLIENT_ID=your_client_id
   export DYNATRACE_CLIENT_SECRET=your_client_secret_here
   ```
3. The Terraform Provider will use these credentials to request OAuth access tokens

### Jenkins Pipeline Integration

Example for sending business events from Jenkins:

```groovy
def ssoResponse = sh(script: """
  curl --location --request POST '${ssoEndpoint}' \\
    --header 'Content-Type: application/x-www-form-urlencoded' \\
    --data-urlencode 'grant_type=client_credentials' \\
    --data-urlencode 'client_id=${oauthClientId}' \\
    --data-urlencode 'client_secret=${oauthClientSecret}' \\
    --data-urlencode 'scope=storage:events:write'
""", returnStdout: true).trim()

def ssoResponseJSON = readJSON(text: ssoResponse)
def accessToken = ssoResponseJSON.access_token

// Use the token for API calls
sh(script: """
  curl --location --request POST '${monitoringTenant}/platform/classic/environment-api/v2/bizevents/ingest' \\
    --header 'Content-Type: application/json' \\
    --header 'Authorization: Bearer ${accessToken}' \\
    --data-raw '${payload}'
""")
```

## Troubleshooting

### Common Issues

#### HTTP 400 Bad Request
- Ensure `grant_type` value is `client_credentials` without quotes
- Verify all parameters are properly URL-encoded
- Check that scopes match those configured for the OAuth client
- Confirm the resource URN format is correct

#### Authentication Failed
- Verify client credentials are correct
- Ensure the OAuth client has the required scopes
- Check that the service user has appropriate group permissions
- Confirm the account UUID in the resource parameter

#### Token Not Working
- Ensure the token hasn't expired
- Verify the token is included in the Authorization header correctly
- Check that the API endpoint accepts OAuth tokens
- Confirm the token has the necessary scopes for the operation

### Validation Steps

1. **Verify OAuth Client Creation**
   - Check client exists in Account Management
   - Confirm assigned scopes match requirements

2. **Test Token Request**
   - Use curl or Postman to test the token endpoint
   - Verify response contains access_token

3. **Validate API Access**
   - Test with a simple GET request first
   - Check response headers for authentication errors

## Security Best Practices

### Credential Management
- Never commit client secrets to version control
- Use environment variables or secret management tools
- Rotate credentials regularly
- Monitor OAuth client usage

### Access Control
- Follow the principle of least privilege
- Grant only necessary scopes
- Use separate clients for different applications
- Regularly audit OAuth client permissions

### Token Handling
- Implement proper token refresh mechanisms
- Don't log or expose tokens in application logs
- Use HTTPS for all API communications
- Implement token expiration handling

## Additional Considerations

### Multiple Accounts
- OAuth clients are created per account
- Ensure service user permissions span all required environments
- Consider account-specific resource URNs

### Environment Access
- Verify service user groups grant same scopes as OAuth client
- Check permissions across all target environments
- Test access before production deployment

### Rate Limiting
- Be aware of API rate limits
- Implement exponential backoff for retries
- Cache tokens appropriately to minimize requests

## Resources and References

- [Dynatrace OAuth Clients Documentation](https://docs.dynatrace.com/docs/manage/identity-access-management/access-tokens-and-oauth-clients/oauth-clients)
- [Account Management API](https://docs.dynatrace.com/docs/manage/account-management/identity-access-management/oauth)
- [Dynatrace API Authentication](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-api/basics/dynatrace-api-authentication)
- [IAM Policies Management](https://docs.dynatrace.com/docs/manage/identity-access-management/manage-iam-policies)
- [Terraform Provider OAuth Setup](https://docs.dynatrace.com/docs/deliver/configuration-as-code/terraform/guides/create-oauth-client)