# DAB on ACA with Entra Auth, MCP, and User-Delegated SQL: Use Case and Test Cases

This document explains the customer scenario this sample is meant to reproduce, how the request flows through the system, and how to test each important part of the deployment.

## Customer Use Case

The customer is building a chat agent that needs to call Data API builder (DAB) through MCP. The agent must not use a shared SQL password or a broad service account to read customer data. Instead, the authenticated user's identity should flow through the system so DAB can call Azure SQL on behalf of that user.

In this sample:

1. A user signs in with Microsoft Entra ID.
2. The chat agent or MCP client sends the user's bearer token to DAB on Azure Container Apps.
3. DAB validates the token using the configured Entra issuer and audience.
4. For SQL access, DAB performs an on-behalf-of token exchange for Azure SQL.
5. Azure SQL authorizes the delegated user and returns data.
6. DAB returns the result through REST, GraphQL, or MCP.

This is useful when a customer says, "REST and GraphQL authentication work, but the MCP side fails." In that situation, the same auth concepts apply, but MCP has extra protocol requirements: the right `Accept` header, JSON-RPC payload shape, session handling, and tool call sequence.

## Deployed Demo Environment

Current demo resources:

```text
Resource group: rg-dab-aca-mcp-auth-demo
Location: westus3
Container App: ca-dabmcp-kwcm0e
Container App URL: https://ca-dabmcp-kwcm0e.wonderfulocean-d6833aa0.westus3.azurecontainerapps.io
ACR: acrdabmcpkwcm0e
Key Vault: kv-dabmcp-kwcm0e
SQL server: sql-dabmcp-kwcm0e
SQL database: dabdemo
User-assigned managed identity: id-dabmcp-kwcm0e
DAB app registration client ID: 5d211457-2361-4626-924c-487f5877e79e
Token scope: api://5d211457-2361-4626-924c-487f5877e79e/access_as_user
```

Current verified image:

```text
acrdabmcpkwcm0e.azurecr.io/dab-aca-mcp-demo:d9bbee65db4f
```

Current successful GitHub Actions run:

```text
https://github.com/vedantpatel1997/data-api-builder/actions/runs/24911225048
```

## Architecture Flow

The deployed Container App uses a user-assigned managed identity. That identity is used for:

- pulling the DAB image from ACR,
- reading the OBO client secret from Key Vault,
- allowing DAB to identify itself when it performs token operations.

The DAB API app registration is used for user sign-in and token audience validation. A caller must request this scope:

```text
api://5d211457-2361-4626-924c-487f5877e79e/access_as_user
```

The DAB config validates:

```json
"audience": "5d211457-2361-4626-924c-487f5877e79e",
"issuer": "https://login.microsoftonline.com/be945e7a-2e17-4b44-926f-512e85873eec/v2.0"
```

For SQL, DAB uses:

```json
"user-delegated-auth": {
  "enabled": true,
  "provider": "EntraId",
  "database-audience": "https://database.windows.net"
}
```

## Test Variables

PowerShell:

```powershell
$tenant = "be945e7a-2e17-4b44-926f-512e85873eec"
$scope = "api://5d211457-2361-4626-924c-487f5877e79e/access_as_user"
$base = "https://ca-dabmcp-kwcm0e.wonderfulocean-d6833aa0.westus3.azurecontainerapps.io"
$token = az account get-access-token --tenant $tenant --scope $scope --query accessToken -o tsv
$headers = @{ Authorization = "Bearer $token" }
```

Bash:

```bash
tenant="be945e7a-2e17-4b44-926f-512e85873eec"
scope="api://5d211457-2361-4626-924c-487f5877e79e/access_as_user"
base="https://ca-dabmcp-kwcm0e.wonderfulocean-d6833aa0.westus3.azurecontainerapps.io"
token="$(az account get-access-token --tenant "$tenant" --scope "$scope" --query accessToken -o tsv)"
```

## Test Case 1: ACA Revision Is Healthy

Purpose: Confirm ACA accepted the image, created a ready revision, and sends all traffic to that revision.

```powershell
az containerapp show `
  --name ca-dabmcp-kwcm0e `
  --resource-group rg-dab-aca-mcp-auth-demo `
  --query "{fqdn:properties.configuration.ingress.fqdn,image:properties.template.containers[0].image,latest:properties.latestRevisionName,ready:properties.latestReadyRevisionName,status:properties.runningStatus}" `
  -o json

az containerapp revision list `
  --name ca-dabmcp-kwcm0e `
  --resource-group rg-dab-aca-mcp-auth-demo `
  --query "[].{name:name,active:properties.active,trafficWeight:properties.trafficWeight,healthState:properties.healthState,provisioningState:properties.provisioningState}" `
  -o table
```

Expected result:

```text
status = Running
latest = ready
active revision = True
trafficWeight = 100
healthState = Healthy
provisioningState = Provisioned
```

## Test Case 2: Container App Identity and Key Vault Secret Are Wired

Purpose: Confirm ACA uses the user-assigned managed identity and reads the OBO secret from Key Vault.

```powershell
az containerapp identity show `
  --name ca-dabmcp-kwcm0e `
  --resource-group rg-dab-aca-mcp-auth-demo `
  -o json

az containerapp secret list `
  --name ca-dabmcp-kwcm0e `
  --resource-group rg-dab-aca-mcp-auth-demo `
  --query "[].{name:name,keyVaultUrl:keyVaultUrl,identity:identity}" `
  -o table

az containerapp show `
  --name ca-dabmcp-kwcm0e `
  --resource-group rg-dab-aca-mcp-auth-demo `
  --query "properties.template.containers[0].env[].{name:name,value:value,secretRef:secretRef}" `
  -o table
```

Expected result:

```text
identity type = UserAssigned
AZURE_CLIENT_ID = cce6065f-d937-4d37-89b8-a58a9803c78c
DAB_OBO_CLIENT_ID = 5d211457-2361-4626-924c-487f5877e79e
DAB_OBO_CLIENT_SECRET uses secretRef obo-secret
obo-secret uses Key Vault URL from kv-dabmcp-kwcm0e
```

## Test Case 3: Unauthenticated REST Request Is Rejected

Purpose: Confirm anonymous callers cannot read data.

```powershell
try {
  Invoke-WebRequest -Uri "$base/api/dbo_Todos?`$first=1" -Method Get -UseBasicParsing
} catch {
  [int]$_.Exception.Response.StatusCode
}
```

Expected result:

```text
403
```

If this unexpectedly returns data, authentication or permissions are too permissive.

## Test Case 4: Authenticated REST Request Returns Data

Purpose: Confirm Entra token validation and SQL OBO are working through the REST endpoint.

```powershell
Invoke-RestMethod `
  -Uri "$base/api/dbo_Todos?`$first=5" `
  -Headers $headers `
  -Method Get |
  ConvertTo-Json -Depth 10
```

Expected result:

```json
{
  "value": [
    {
      "Id": 1,
      "Title": "Confirm DAB REST authentication",
      "IsComplete": false
    }
  ]
}
```

The exact row count should be `3` in the seeded demo database.

## Test Case 5: Authenticated GraphQL Request Returns Data

Purpose: Confirm the same delegated auth path works through GraphQL.

```powershell
$graphqlBody = @{
  query = "{ dbo_Todos(first: 5) { items { Id Title IsComplete } } }"
} | ConvertTo-Json -Compress

Invoke-RestMethod `
  -Uri "$base/graphql" `
  -Headers $headers `
  -Method Post `
  -ContentType "application/json" `
  -Body $graphqlBody |
  ConvertTo-Json -Depth 10
```

Expected result:

```json
{
  "data": {
    "dbo_Todos": {
      "items": [
        {
          "Id": 1,
          "Title": "Confirm DAB REST authentication",
          "IsComplete": false
        }
      ]
    }
  }
}
```

The exact row count should be `3` in the seeded demo database.

## Test Case 6: MCP Initialize Works

Purpose: Confirm the MCP endpoint accepts a valid authenticated JSON-RPC initialize request.

MCP over HTTP requires the correct `Accept` header:

```text
Accept: application/json, text/event-stream
```

PowerShell:

```powershell
$mcpHeaders = @{
  Authorization = "Bearer $token"
  Accept = "application/json, text/event-stream"
}

$initBody = @{
  jsonrpc = "2.0"
  id = 1
  method = "initialize"
  params = @{
    protocolVersion = "2024-11-05"
    capabilities = @{}
    clientInfo = @{
      name = "customer-agent-test"
      version = "1.0.0"
    }
  }
} | ConvertTo-Json -Depth 10 -Compress

$init = Invoke-WebRequest `
  -Uri "$base/mcp" `
  -Headers $mcpHeaders `
  -Method Post `
  -ContentType "application/json" `
  -Body $initBody

$init.StatusCode
$init.Headers["Content-Type"]
$init.Content
```

Expected result:

```text
StatusCode = 200
Content-Type = text/event-stream
Response includes serverInfo and tools capability
```

Save the session ID if returned:

```powershell
$mcpHeaders["mcp-session-id"] = $init.Headers["mcp-session-id"]
```

## Test Case 7: MCP Tools List Includes DAB Tools

Purpose: Confirm DAB exposes MCP tools for configured entities.

```powershell
$toolsBody = @{
  jsonrpc = "2.0"
  id = 2
  method = "tools/list"
  params = @{}
} | ConvertTo-Json -Depth 10 -Compress

$tools = Invoke-WebRequest `
  -Uri "$base/mcp" `
  -Headers $mcpHeaders `
  -Method Post `
  -ContentType "application/json" `
  -Body $toolsBody

$tools.Content
```

Expected result includes:

```text
describe_entities
read_records
create_record
update_record
delete_record
aggregate_records
execute_entity
```

## Test Case 8: MCP read_records Returns SQL Data

Purpose: Confirm the customer-critical path: MCP tool call, DAB auth, SQL OBO, and data response.

```powershell
$callBody = @{
  jsonrpc = "2.0"
  id = 3
  method = "tools/call"
  params = @{
    name = "read_records"
    arguments = @{
      entity = "dbo_Todos"
      first = 3
    }
  }
} | ConvertTo-Json -Depth 10 -Compress

$mcp = Invoke-WebRequest `
  -Uri "$base/mcp" `
  -Headers $mcpHeaders `
  -Method Post `
  -ContentType "application/json" `
  -Body $callBody

$mcp.StatusCode
$mcp.Headers["Content-Type"]
$mcp.Content
```

Expected result:

```text
StatusCode = 200
Content-Type = text/event-stream
Response includes: Successfully read records for entity 'dbo_Todos'
Response includes the seeded Todo rows
```

## Test Case 9: MCP Missing Accept Header Fails

Purpose: Reproduce a common MCP client issue.

```powershell
$badMcpHeaders = @{
  Authorization = "Bearer $token"
}

try {
  Invoke-WebRequest `
    -Uri "$base/mcp" `
    -Headers $badMcpHeaders `
    -Method Post `
    -ContentType "application/json" `
    -Body $initBody
} catch {
  [int]$_.Exception.Response.StatusCode
}
```

Expected result:

```text
406
```

Meaning: the request reached MCP, but the client did not advertise that it accepts the MCP HTTP response media types.

## Test Case 10: Wrong Audience Token Fails

Purpose: Confirm DAB only accepts tokens for the configured API audience.

```powershell
$wrongToken = az account get-access-token `
  --tenant $tenant `
  --scope "https://database.windows.net/.default" `
  --query accessToken `
  -o tsv

try {
  Invoke-WebRequest `
    -Uri "$base/api/dbo_Todos?`$first=1" `
    -Headers @{ Authorization = "Bearer $wrongToken" } `
    -Method Get `
    -UseBasicParsing
} catch {
  [int]$_.Exception.Response.StatusCode
}
```

Expected result:

```text
401 or 403
```

Meaning: a SQL token is not a valid token for the DAB API audience.

## Test Case 11: GitHub Actions Deploys to ACA

Purpose: Confirm CI/CD builds and deploys the image.

```powershell
$runs = Invoke-RestMethod `
  -Uri "https://api.github.com/repos/vedantpatel1997/data-api-builder/actions/workflows/deploy-dab-aca.yml/runs?branch=main&per_page=3" `
  -Headers @{ "User-Agent" = "dab-aca-test" }

$runs.workflow_runs |
  Select-Object id,head_sha,status,conclusion,html_url,created_at,updated_at |
  ConvertTo-Json -Depth 4
```

Expected result:

```text
Latest run status = completed
Latest run conclusion = success
```

Then confirm ACA is running the same commit tag:

```powershell
az containerapp show `
  --name ca-dabmcp-kwcm0e `
  --resource-group rg-dab-aca-mcp-auth-demo `
  --query "properties.template.containers[0].image" `
  -o tsv
```

Expected result:

```text
acrdabmcpkwcm0e.azurecr.io/dab-aca-mcp-demo:<first-12-chars-of-commit-sha>
```

## Test Case 12: CI/CD Azure RBAC Is Correct

Purpose: Confirm the GitHub OIDC identity can read ACR metadata, push images, and update ACA.

```powershell
$spObjectId = "8b3fa851-60c0-4e82-8849-17f9f6442941"
$acrScope = "/subscriptions/6a3bb170-5159-4bff-860b-aa74fb762697/resourceGroups/rg-dab-aca-mcp-auth-demo/providers/Microsoft.ContainerRegistry/registries/acrdabmcpkwcm0e"
$acaScope = "/subscriptions/6a3bb170-5159-4bff-860b-aa74fb762697/resourceGroups/rg-dab-aca-mcp-auth-demo/providers/Microsoft.App/containerApps/ca-dabmcp-kwcm0e"

az role assignment list `
  --scope $acrScope `
  --query "[?principalId=='$spObjectId'].{role:roleDefinitionName,principalId:principalId}" `
  -o table

az role assignment list `
  --scope $acaScope `
  --query "[?principalId=='$spObjectId'].{role:roleDefinitionName,principalId:principalId}" `
  -o table
```

Expected result:

```text
ACR:
AcrPush
Reader

ACA:
Container Apps Contributor
```

## Troubleshooting Guide

### MCP returns 406

Most likely cause: missing `Accept` header.

Fix:

```text
Accept: application/json, text/event-stream
```

### MCP returns 401 or 403

Likely causes:

- no bearer token was sent,
- token audience does not match the DAB API audience,
- token issuer does not match the configured tenant,
- the MCP client is not forwarding the user's access token.

Check:

```powershell
$token = az account get-access-token --tenant $tenant --scope $scope --query accessToken -o tsv
```

### REST and GraphQL work, but MCP fails

Check the MCP-specific items first:

- `Accept` header includes `application/json, text/event-stream`,
- request body is valid JSON-RPC,
- client sends the returned `mcp-session-id` header on later calls when the server provides one,
- client calls `initialize` before `tools/list` or `tools/call`,
- tool name and entity name match what `tools/list` and `describe_entities` return.

### DAB authenticates the user but SQL fails

Likely causes:

- the DAB app registration does not have Azure SQL delegated permission/admin consent,
- the user or mapped principal does not exist in SQL,
- the managed identity SQL user SID was created with the wrong client ID byte order,
- `DAB_OBO_CLIENT_SECRET` is missing or cannot be read from Key Vault.

Check ACA logs:

```powershell
az containerapp logs show `
  --name ca-dabmcp-kwcm0e `
  --resource-group rg-dab-aca-mcp-auth-demo `
  --tail 100
```

Successful OBO logs include:

```text
OboTokenAcquired
OboTokenCacheHit
```

### GitHub Actions login fails

Check the federated credential:

```text
issuer: https://token.actions.githubusercontent.com
subject: repo:vedantpatel1997/data-api-builder:ref:refs/heads/main
audience: api://AzureADTokenExchange
```

If the subject is wrong, Azure login will fail even when the client ID and tenant ID are correct.

### GitHub Actions can login but cannot read ACR

The workflow uses `az acr show`, so the GitHub Actions service principal needs `Reader` on the ACR scope in addition to `AcrPush`.

### GitHub Actions build cannot find dab-config.generated.json

That file is intentionally ignored by git because it is environment-specific. The workflow must generate it from `dab-config.template.json` before `docker build`.

## Current Verification Result

Verified on 2026-04-24:

```text
ACA status: Running
Active revision: ca-dabmcp-kwcm0e--0000003
Traffic: 100 percent
Image: acrdabmcpkwcm0e.azurecr.io/dab-aca-mcp-demo:d9bbee65db4f
No-token REST: 403
Authenticated REST: 3 rows
Authenticated GraphQL: 3 rows
MCP tools/list: includes read_records
MCP read_records: 200 and returned dbo_Todos rows
Latest GitHub Actions deploy run: success
```

