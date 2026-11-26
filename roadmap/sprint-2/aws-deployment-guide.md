# Sprint 2 AWS Deployment & Live Verification Guide — FamilyLedger (`dev`)

> **Scope**: Copy-pasteable Windows PowerShell (plus Bash equivalents for Step 8) runbook for deploying **Sprint 2** (`family_service` + Option B2 3-stack SAM topology + `frontend-web/` Angular SPA) onto an existing Sprint 1 `dev` AWS environment (`familyledger-core-dev`, `familyledger-data-dev`, and Amazon Aurora DSQL), bootstrapping the least-privilege `familyledger_app` database role, verifying all 9 REST endpoints, and running the Angular SPA against live AWS (`localhost:4200` or S3/CloudFront).
>
> For fresh account bootstrap, day-2 operations across all sprints/environments, or complete teardown, see the overall product runbook: [`aws-deployment-and-teardown.md`](../../architecture/aws-deployment-and-teardown.md).

---

## Overview: Mandatory Upgrade Order

Upgrading from Sprint 1's 2-stack layout to Sprint 2's **Option B2 (3-Stack 1-Way DAG)** (`services` $\rightarrow$ `core` & `data`) requires updating `core` before creating `services` so `FamilyLedgerApi` moves cleanly without an API Gateway collision:

```
1. Pre-flight & session variables
2. Build migrate_runner (ARM64) -> deploy familyledger-data-dev -> invoke migrations (34 -> 36)
3. Bootstrap Aurora DSQL non-admin role (familyledger_app) mapped to FamilyLedgerServiceRole
4. Update familyledger-core-dev FIRST (drops legacy FamilyLedgerApi; preserves Cognito/S3/CloudFront)
5. Build family_service (ARM64) -> deploy familyledger-services-dev (API Gateway + family_service)
6. Sync API_GATEWAY_URL to backend/.env.dev & backend/.aws-resources.md
7. Run live Cognito IdToken + X-Family-Id CLI smoke tests across all 9 endpoints + inspect CloudWatch logs
8. Build/configure frontend-web Angular SPA -> run localhost:4200 OR sync to S3 + CloudFront + Browser E2E
```

---

## 0. Pre-Flight Checks & Session Variables

### 0.1 Verify Local Toolchain & AWS Identity

Run from Windows PowerShell:

```powershell
aws --version          # AWS CLI v2
sam --version          # AWS SAM CLI
cargo lambda --version # cargo-lambda >= 1.9.2
zig version            # zig >= 0.17.1
aws sts get-caller-identity
```

> [!IMPORTANT]
> **Required Deployer IAM Permissions**: Your IAM user must have:
> 1. `FamilyLedgerDeployerRoleManagementPolicy` (see [`aws-deployment-and-teardown.md` §2.2](../../architecture/aws-deployment-and-teardown.md#22-deployer-iam-permissions)) to manage `familyledger-*` roles.
> 2. `dsql:DbConnectAdmin` on your Aurora DSQL cluster to execute the one-time `familyledger_app` SQL role bootstrap in Step 2.
> 3. Standard CloudFormation, SAM/S3 staging, API Gateway v2, Lambda, EventBridge, CloudWatch Logs, and dev-test Cognito admin actions (`cognito-idp:AdminCreateUser`, `AdminSetUserPassword`, `UpdateUserPoolClient`, `AdminInitiateAuth`, `AdminDeleteUser`) if running the CLI smoke test.

### 0.2 Initialize PowerShell Session Variables

Start inside your `backend/` directory (e.g. `...\home-harmony\backend`), then initialize all session variables:

```powershell
# Run from backend/
$BACKEND_DIR = (Get-Location).Path
$SAM_DIR = Join-Path $BACKEND_DIR "infrastructure\sam"

$REGION = "us-east-1"
$ENV = "dev"
$ACCOUNT_ID = (aws sts get-caller-identity --query Account --output text)

# Inspect active DSQL cluster ID, then assign it below
aws dsql list-clusters --region $REGION --query "clusters[].{identifier:identifier,status:status}" --output table
$DSQL_ID = "<YOUR_DSQL_CLUSTER_ID>"
$DSQL_ENDPOINT = "$DSQL_ID.dsql.$REGION.on.aws"
$SERVICE_ROLE_ARN = "arn:aws:iam::${ACCOUNT_ID}:role/familyledger-service-role-$ENV"

"Backend=$BACKEND_DIR  Account=$ACCOUNT_ID  DSQL=$DSQL_ENDPOINT  ServiceRole=$SERVICE_ROLE_ARN"
```

### 0.3 Optional Local Code & Template Verification Gate

```powershell
Set-Location $BACKEND_DIR
$env:SQLX_OFFLINE = "true"
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace

Set-Location $SAM_DIR
sam validate -t template.yaml --lint
sam validate -t data-stream-template.yaml --lint
sam validate -t services-template.yaml --lint
```

---

## 1. Step 1: Build `migrate_runner`, Deploy `data` Stack & Apply Migrations (`34 -> 36`)

Sprint 2 adds 2 asynchronous index migrations (`idx_members_user` and `idx_invite_tokens_family`, bringing the embedded count to **36** migrations / **18** indexes).

### 1.1 Build ARM64 `migrate_runner` Binary & Deploy `familyledger-data-dev`

```powershell
Set-Location $BACKEND_DIR
cargo lambda build --release --arm64 --package migrate_runner

Set-Location $SAM_DIR
sam validate -t data-stream-template.yaml --lint
sam deploy `
  --template-file data-stream-template.yaml `
  --config-file samconfig-data.toml `
  --capabilities CAPABILITY_NAMED_IAM `
  --parameter-overrides "Environment=$ENV DsqlClusterIdentifier=$DSQL_ID"
```

Confirm the changeset with `y` when prompted (or pass `--no-confirm-changeset` if you prefer non-interactive deploys).

### 1.2 Invoke `familyledger-migrate-dev` & Verify Output

```powershell
Set-Location $BACKEND_DIR
aws lambda invoke `
  --function-name "familyledger-migrate-$ENV" `
  --region $REGION `
  --log-type Tail `
  --cli-binary-format raw-in-base64-out `
  --payload '{}' `
  target\migrate-response.json

Get-Content target\migrate-response.json
Remove-Item target\migrate-response.json -Force
```

Expected JSON output when upgrading from Sprint 1 (`34` previously applied migrations):

```json
{"message":"Schema migrations completed successfully","total_migrations":36,"applied_now":2,"already_applied":34,"active_async_jobs":0}
```

*(On a brand-new Aurora DSQL cluster, expect `applied_now: 36, already_applied: 0, active_async_jobs: 0`.)*

If troubleshooting a migration failure, decode the `LogResult` tail directly in PowerShell:

```powershell
$RAW_LOG = aws lambda invoke `
  --function-name "familyledger-migrate-$ENV" `
  --region $REGION `
  --log-type Tail `
  --cli-binary-format raw-in-base64-out `
  --payload '{}' `
  --query 'LogResult' `
  --output text `
  target\migrate-response.json
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($RAW_LOG))
Remove-Item target\migrate-response.json -Force
```

---

## 2. Step 2: Bootstrap Least-Privilege Aurora DSQL Role (`familyledger_app`)

`FamilyLedgerServiceRole` grants `dsql:DbConnect` without `dsql:DbConnectAdmin`, and `family_service` sets `DSQL_USER=familyledger_app` (omitting `DELETE` enforces soft-deletes at the database privilege boundary).

### 2.1 Generate Admin Auth Token & Connect via `psql`

```powershell
$env:PGPASSWORD = (aws dsql generate-db-connect-admin-auth-token `
  --region $REGION `
  --hostname $DSQL_ENDPOINT `
  --expires-in 3600)
$env:PGSSLMODE = "require"

# Print the exact ARN you will paste in Step 2.2
$SERVICE_ROLE_ARN
```

Choose **Option A** (native PostgreSQL `psql` installed) or **Option B** (Docker fallback if `psql` is not installed on Windows):

```powershell
# Option A: Native psql
psql --host $DSQL_ENDPOINT --username admin --dbname postgres

# Option B: Docker PostgreSQL client fallback
docker run --rm -it `
  -e PGPASSWORD=$env:PGPASSWORD `
  -e PGSSLMODE=require `
  postgres:16-alpine `
  psql --host $DSQL_ENDPOINT --username admin --dbname postgres
```

### 2.2 Execute the 5 Statements Individually (Autocommit Mode)

> [!WARNING]
> **1 Statement per Transaction**: Aurora DSQL rejects multi-statement DDL/authorization transactions. Paste and run **each statement one at a time** at the `postgres=>` prompt (do **not** wrap in `BEGIN ... COMMIT`), substituting your exact `$SERVICE_ROLE_ARN` printed above.

```sql
CREATE ROLE familyledger_app WITH LOGIN;
```
```sql
AWS IAM GRANT familyledger_app TO 'arn:aws:iam::<YOUR_ACCOUNT_ID>:role/familyledger-service-role-dev';
```
```sql
GRANT USAGE ON SCHEMA public TO familyledger_app;
```
```sql
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO familyledger_app;
```
```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE ON TABLES TO familyledger_app;
```

### 2.3 Verify Role Mapping & Close Session

Inside `psql`, confirm the IAM mapping is active:

```sql
SELECT * FROM sys.iam_pg_role_mappings;
```

Verify a row exists with `pg_role_name = familyledger_app` and `arn = arn:aws:iam::<YOUR_ACCOUNT_ID>:role/familyledger-service-role-dev`. Exit with `\q`, then clear the ephemeral token from PowerShell:

```powershell
Remove-Item Env:\PGPASSWORD
```

---

## 3. Step 3: Upgrade `familyledger-core-dev` First (Drop Legacy `FamilyLedgerApi`)

> [!IMPORTANT]
> **Deploy `familyledger-core-dev` BEFORE `familyledger-services-dev`**. [`template.yaml`](../../../backend/infrastructure/sam/template.yaml) removes the old `FamilyLedgerApi` and `ApiGatewayUrl` output so [`services-template.yaml`](../../../backend/infrastructure/sam/services-template.yaml) owns the HTTP API without running two API Gateways side by side or churning the invoke URL. (Export names do not collide: `familyledger-core-dev-ApiUrl` vs `familyledger-services-dev-ApiUrl`.)

```powershell
Set-Location $SAM_DIR
sam validate -t template.yaml --lint
sam deploy `
  --template-file template.yaml `
  --config-file samconfig.toml
```

Before typing `y` to confirm the CloudFormation changeset:
- Confirm it only deletes legacy `FamilyLedgerApi*` resources.
- Confirm `FamilyLedgerUserPool`, `FamilyLedgerWebAppClient`, `FamilyLedgerSpaBucket`, and `FamilyLedgerCdn` show no destructive replacement (`Replacement: False` / untouched).

Verify the 3 remaining `core` outputs:

```powershell
aws cloudformation describe-stacks `
  --stack-name "familyledger-core-$ENV" `
  --region $REGION `
  --query "Stacks[0].Outputs" `
  --output table
```

---

## 4. Step 4: Build `family_service` & Deploy `familyledger-services-dev`

### 4.1 Build the ARM64 Lambda Bootstrap Binary

```powershell
Set-Location $BACKEND_DIR
cargo lambda build --release --arm64 --package family_service
```

Confirm `backend\target\lambda\family_service\bootstrap` is produced cleanly.

### 4.2 Validate & Deploy `familyledger-services-dev`

```powershell
Set-Location $SAM_DIR
sam validate -t services-template.yaml --lint
sam deploy `
  --template-file services-template.yaml `
  --config-file samconfig-services.toml `
  --parameter-overrides "Environment=$ENV DsqlClusterIdentifier=$DSQL_ID CoreStackName=familyledger-core-$ENV DataStackName=familyledger-data-$ENV"
```

Review the changeset (`FamilyLedgerApi`, `FamilyLedgerFamilyServiceFunction`, Lambda API Gateway permissions) and enter `y` to deploy. Inspect stack outputs:

```powershell
aws cloudformation describe-stacks `
  --stack-name "familyledger-services-$ENV" `
  --region $REGION `
  --query "Stacks[0].Outputs" `
  --output table
```

---

## 5. Step 5: Sync Stack Outputs to `backend/.env.dev`

Moving `FamilyLedgerApi` into `familyledger-services-dev` creates a new API Gateway invoke URL (`ApiGatewayUrl`). Refresh PowerShell variables from all 3 stacks and write/update `backend/.env.dev` (gitignored):

```powershell
$USER_POOL_ID = aws cloudformation describe-stacks --stack-name "familyledger-core-$ENV" --region $REGION --query "Stacks[0].Outputs[?OutputKey=='CognitoUserPoolId'].OutputValue" --output text
$CLIENT_ID = aws cloudformation describe-stacks --stack-name "familyledger-core-$ENV" --region $REGION --query "Stacks[0].Outputs[?OutputKey=='CognitoWebAppClientId'].OutputValue" --output text
$CLOUDFRONT_URL = aws cloudformation describe-stacks --stack-name "familyledger-core-$ENV" --region $REGION --query "Stacks[0].Outputs[?OutputKey=='CloudFrontUrl'].OutputValue" --output text
$API_URL = aws cloudformation describe-stacks --stack-name "familyledger-services-$ENV" --region $REGION --query "Stacks[0].Outputs[?OutputKey=='ApiGatewayUrl'].OutputValue" --output text
$EVENT_BUS_NAME = aws cloudformation describe-stacks --stack-name "familyledger-data-$ENV" --region $REGION --query "Stacks[0].Outputs[?OutputKey=='EventBusName'].OutputValue" --output text

$ENV_FILE = Join-Path $BACKEND_DIR ".env.$ENV"
@"
AWS_REGION=$REGION
ENVIRONMENT=$ENV

COGNITO_USER_POOL_ID=$USER_POOL_ID
COGNITO_CLIENT_ID=$CLIENT_ID
COGNITO_ISSUER=https://cognito-idp.$REGION.amazonaws.com/$USER_POOL_ID

API_GATEWAY_URL=$API_URL
CLOUDFRONT_URL=$CLOUDFRONT_URL

DSQL_CLUSTER_ID=$DSQL_ID
DSQL_ENDPOINT=$DSQL_ENDPOINT
DATABASE_URL=postgres://admin@${DSQL_ENDPOINT}:5432/postgres?sslmode=require

EVENT_BUS_NAME=$EVENT_BUS_NAME
"@ | Set-Content -Path $ENV_FILE -Encoding utf8

Get-Content $ENV_FILE
```

*(If you track live outputs in `backend/.aws-resources.md`, update its `ApiGatewayUrl` row to match `$API_URL`.)*

---

## 6. Step 6: Live Smoke Test (Cognito JWT + `X-Family-Id` Isolation + All 9 Endpoints)

> [!IMPORTANT]
> **Send Cognito `IdToken`, NOT `AccessToken`**: [`services-template.yaml`](../../../backend/infrastructure/sam/services-template.yaml) sets `JwtConfiguration.Audience` to `CognitoWebAppClientId`. Cognito includes the `aud` claim and `email` in the **ID token** (`IdToken`), whereas Cognito access tokens emit `client_id` without `aud`. Sending an `AccessToken` fails at API Gateway with `401 Unauthorized`.

### 6.1 Create Two Test Users & Obtain `IdToken`s (Alice = Owner, Bob = Member)

Because [`template.yaml`](../../../backend/infrastructure/sam/template.yaml) intentionally enables only `ALLOW_USER_SRP_AUTH` and `ALLOW_REFRESH_TOKEN_AUTH` for production security, choose **Option A** (no client changes; Hosted UI OAuth2 code flow) or **Option B** (fast CLI-only dev flow that immediately reverts `ExplicitAuthFlows`).

<details>
<summary><strong>Option A (Zero Cognito Client Changes): Hosted UI Authorization Code Exchange</strong></summary>

1. Create two users (`alice@example.com` and `bob@example.com`) with permanent passwords:
   ```powershell
   aws cognito-idp admin-create-user --user-pool-id $USER_POOL_ID --username alice@example.com --user-attributes Name=email,Value=alice@example.com Name=email_verified,Value=true --message-action SUPPRESS --region $REGION
   aws cognito-idp admin-set-user-password --user-pool-id $USER_POOL_ID --username alice@example.com --password "<StrongPass123!>" --permanent --region $REGION
   ```
2. Open `https://familyledger-$ENV-$ACCOUNT_ID.auth.$REGION.amazoncognito.com/login?client_id=$CLIENT_ID&response_type=code&scope=openid+email+profile&redirect_uri=http://localhost:4200/auth/callback` in your browser, log in as Alice, and copy the `code` query parameter from the redirected URL (`http://localhost:4200/auth/callback?code=...`).
3. Exchange the code for `id_token`:
   ```powershell
   $ALICE_AUTH = Invoke-RestMethod -Method Post `
     -Uri "https://familyledger-$ENV-$ACCOUNT_ID.auth.$REGION.amazoncognito.com/oauth2/token" `
     -ContentType "application/x-www-form-urlencoded" `
     -Body "grant_type=authorization_code&client_id=$CLIENT_ID&code=<CODE>&redirect_uri=http://localhost:4200/auth/callback"
   $ALICE_TOKEN = $ALICE_AUTH.id_token
   ```
4. Repeat in a private/incognito window for `bob@example.com` to capture `$BOB_TOKEN`.

</details>

**Option B (Fast CLI-Only Dev Test with Immediate Revert)**:

```powershell
# 1. Create two confirmed test users (adjust emails/passwords as needed)
$TEST_PASS = "HomeHarmony#Dev2026"
aws cognito-idp admin-create-user --user-pool-id $USER_POOL_ID --username smoke-alice@example.com --user-attributes Name=email,Value=smoke-alice@example.com Name=email_verified,Value=true --message-action SUPPRESS --region $REGION
aws cognito-idp admin-set-user-password --user-pool-id $USER_POOL_ID --username smoke-alice@example.com --password $TEST_PASS --permanent --region $REGION

aws cognito-idp admin-create-user --user-pool-id $USER_POOL_ID --username smoke-bob@example.com --user-attributes Name=email,Value=smoke-bob@example.com Name=email_verified,Value=true --message-action SUPPRESS --region $REGION
aws cognito-idp admin-set-user-password --user-pool-id $USER_POOL_ID --username smoke-bob@example.com --password $TEST_PASS --permanent --region $REGION

# 2. Temporarily allow ADMIN_USER_PASSWORD_AUTH on the dev client, fetch IdTokens, then revert immediately
aws cognito-idp update-user-pool-client `
  --user-pool-id $USER_POOL_ID `
  --client-id $CLIENT_ID `
  --explicit-auth-flows ALLOW_ADMIN_USER_PASSWORD_AUTH ALLOW_USER_SRP_AUTH ALLOW_REFRESH_TOKEN_AUTH `
  --allowed-o-auth-flows code `
  --allowed-o-auth-flows-user-pool-client `
  --allowed-o-auth-scopes openid email profile `
  --callback-urls "http://localhost:4200/auth/callback" `
  --logout-urls "http://localhost:4200/auth/logout" `
  --region $REGION

$ALICE_TOKEN = aws cognito-idp admin-initiate-auth `
  --user-pool-id $USER_POOL_ID `
  --client-id $CLIENT_ID `
  --auth-flow ADMIN_USER_PASSWORD_AUTH `
  --auth-parameters USERNAME=smoke-alice@example.com,PASSWORD=$TEST_PASS `
  --region $REGION `
  --query "AuthenticationResult.IdToken" `
  --output text

$BOB_TOKEN = aws cognito-idp admin-initiate-auth `
  --user-pool-id $USER_POOL_ID `
  --client-id $CLIENT_ID `
  --auth-flow ADMIN_USER_PASSWORD_AUTH `
  --auth-parameters USERNAME=smoke-bob@example.com,PASSWORD=$TEST_PASS `
  --region $REGION `
  --query "AuthenticationResult.IdToken" `
  --output text

# 3. Revert client auth flows back to production SRP + Refresh settings
aws cognito-idp update-user-pool-client `
  --user-pool-id $USER_POOL_ID `
  --client-id $CLIENT_ID `
  --explicit-auth-flows ALLOW_USER_SRP_AUTH ALLOW_REFRESH_TOKEN_AUTH `
  --allowed-o-auth-flows code `
  --allowed-o-auth-flows-user-pool-client `
  --allowed-o-auth-scopes openid email profile `
  --callback-urls "http://localhost:4200/auth/callback" `
  --logout-urls "http://localhost:4200/auth/logout" `
  --region $REGION
```

### 6.2 Exercise All 9 Endpoints, Tenant Isolation (`AU-01`–`AU-08`) & EventBridge

Run this PowerShell verification helper + sequence in the same session:

```powershell
function Invoke-FlApi {
    param(
        [string]$Method,
        [string]$Path,
        [string]$Token = "",
        [string]$FamilyId = "",
        [object]$Body = $null
    )
    $headers = @{}
    if ($Token)    { $headers["Authorization"] = "Bearer $Token" }
    if ($FamilyId) { $headers["X-Family-Id"]   = $FamilyId }

    $params = @{
        Method      = $Method
        Uri         = "$API_URL$Path"
        Headers     = $headers
        ContentType = "application/json"
    }
    if ($null -ne $Body) {
        $params["Body"] = ($Body | ConvertTo-Json -Depth 5 -Compress)
    }

    try {
        $resp = Invoke-WebRequest @params
        $json = if ($resp.Content) { $resp.Content | ConvertFrom-Json } else { $null }
        return [PSCustomObject]@{ Status = [int]$resp.StatusCode; Body = $json }
    } catch {
        $status = [int]$_.Exception.Response.StatusCode
        $reader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream())
        $raw = $reader.ReadToEnd()
        $json = if ($raw) { try { $raw | ConvertFrom-Json } catch { $raw } } else { $null }
        return [PSCustomObject]@{ Status = $status; Body = $json }
    }
}

# 1. Unauthenticated request -> expect 401
$r1 = Invoke-FlApi GET "/profile/families"
"1. Unauthenticated GET /profile/families -> $($r1.Status)"

# 2. Create family as Alice -> expect 201 Created
$r2 = Invoke-FlApi POST "/families" -Token $ALICE_TOKEN -Body @{
    name = "Smoke Test Family"
    home_currency = "USD"
    owner_display_name = "Alice"
}
$FAMILY_ID = $r2.Body.id
"2. POST /families -> $($r2.Status) (FamilyId=$FAMILY_ID)"

# 3. List Alice's families -> expect 200 OK
$r3 = Invoke-FlApi GET "/profile/families" -Token $ALICE_TOKEN
"3. GET /profile/families -> $($r3.Status) (Count=$($r3.Body.items.Count))"

# 4. AU-03: Missing X-Family-Id -> expect 400 MISSING_FAMILY_ID
$r4 = Invoke-FlApi GET "/families/$FAMILY_ID" -Token $ALICE_TOKEN
"4. Missing X-Family-Id -> $($r4.Status) ($($r4.Body.code))"

# 5. AU-05: Route {id} differs from verified X-Family-Id -> expect 403 FAMILY_ID_MISMATCH
$r5 = Invoke-FlApi GET "/families/$([guid]::NewGuid().Guid)" -Token $ALICE_TOKEN -FamilyId $FAMILY_ID
"5. Route id != X-Family-Id -> $($r5.Status) ($($r5.Body.code))"

# 6. GET & PUT /families/{id} as Owner -> expect 200 OK
$r6a = Invoke-FlApi GET "/families/$FAMILY_ID" -Token $ALICE_TOKEN -FamilyId $FAMILY_ID
$r6b = Invoke-FlApi PUT "/families/$FAMILY_ID" -Token $ALICE_TOKEN -FamilyId $FAMILY_ID -Body @{
    name = "Updated Smoke Family"
    home_currency = "EUR"
}
"6. GET/PUT /families/{id} -> $($r6a.Status) / $($r6b.Status) (Currency=$($r6b.Body.home_currency))"

# 7. Create invite as Owner -> expect 201 Created
$r7 = Invoke-FlApi POST "/families/$FAMILY_ID/invites" -Token $ALICE_TOKEN -FamilyId $FAMILY_ID -Body @{
    role = "member"
    relationship = "Spouse"
    ttl_hours = 24
}
$INVITE_TOKEN = $r7.Body.token
"7. POST /families/{id}/invites -> $($r7.Status) (Token=$INVITE_TOKEN)"

# 8. AU-04: Non-member Bob blocked before joining -> expect 403 USER_NOT_IN_FAMILY
$r8 = Invoke-FlApi GET "/families/$FAMILY_ID" -Token $BOB_TOKEN -FamilyId $FAMILY_ID
"8. Non-member Bob GET /families/{id} -> $($r8.Status) ($($r8.Body.code))"

# 9. Bob accepts invite (identity-only auth) -> expect 201 Created
$r9 = Invoke-FlApi POST "/families/$FAMILY_ID/members" -Token $BOB_TOKEN -Body @{
    token = $INVITE_TOKEN
    display_name = "Bob"
}
$BOB_MEMBER_ID = $r9.Body.id
"9. POST /families/{id}/members -> $($r9.Status) (BobMemberId=$BOB_MEMBER_ID)"

# 10. Bob lists members -> expect 200 OK (2 active members)
$r10 = Invoke-FlApi GET "/families/$FAMILY_ID/members" -Token $BOB_TOKEN -FamilyId $FAMILY_ID
"10. GET /families/{id}/members -> $($r10.Status) (Count=$($r10.Body.items.Count))"

# 11. AU-08: Bob (Member) blocked from Owner-only PUT /families/{id} -> expect 403 INSUFFICIENT_ROLE
$r11 = Invoke-FlApi PUT "/families/$FAMILY_ID" -Token $BOB_TOKEN -FamilyId $FAMILY_ID -Body @{
    name = "Hijacked Name"
    home_currency = "USD"
}
"11. Member Bob PUT /families/{id} -> $($r11.Status) ($($r11.Body.code))"

# 12. Alice changes Bob's role to 'other' with custom permissions -> expect 200 OK
$r12 = Invoke-FlApi PATCH "/families/$FAMILY_ID/members/$BOB_MEMBER_ID/role" -Token $ALICE_TOKEN -FamilyId $FAMILY_ID -Body @{
    role = "other"
    permissions = @("view_loans", "view_budget")
}
"12. PATCH member role -> $($r12.Status) (Role=$($r12.Body.role))"

# 13. Alice soft-deletes Bob -> expect 204 No Content
$r13 = Invoke-FlApi DELETE "/families/$FAMILY_ID/members/$BOB_MEMBER_ID" -Token $ALICE_TOKEN -FamilyId $FAMILY_ID
"13. DELETE member -> $($r13.Status)"

# 14. Removed Bob immediately loses access -> expect 403 USER_NOT_IN_FAMILY
$r14 = Invoke-FlApi GET "/families/$FAMILY_ID" -Token $BOB_TOKEN -FamilyId $FAMILY_ID
"14. Soft-deleted Bob GET /families/{id} -> $($r14.Status) ($($r14.Body.code))"

# 15. Concurrent single-use invite token race -> expect exactly one 201 and one 409 (MEMBER_ALREADY_EXISTS for same user, or TOKEN_ALREADY_USED for distinct users; zero 500s)
$r15Inv = Invoke-FlApi POST "/families/$FAMILY_ID/invites" -Token $ALICE_TOKEN -FamilyId $FAMILY_ID -Body @{ role = "child" }
$RACE_TOKEN = $r15Inv.Body.token
$raceJobs = 1..2 | ForEach-Object {
    Start-ThreadJob -ScriptBlock {
        param($ApiUrl, $FamId, $Tok, $Jwt, $Name)
        $body = @{ token = $Tok; display_name = $Name } | ConvertTo-Json -Compress
        try {
            $resp = Invoke-WebRequest -Method Post -Uri "$ApiUrl/families/$FamId/members" `
                -Headers @{ Authorization = "Bearer $Jwt" } -ContentType "application/json" -Body $body
            [PSCustomObject]@{ Status = [int]$resp.StatusCode; Code = "OK" }
        } catch {
            $raw = (New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream())).ReadToEnd()
            $err = try { $raw | ConvertFrom-Json } catch { $null }
            [PSCustomObject]@{ Status = [int]$_.Exception.Response.StatusCode; Code = $err.code }
        }
    } -ArgumentList $API_URL, $FAMILY_ID, $RACE_TOKEN, $BOB_TOKEN, "Bob Race $_"
}
$raceResults = $raceJobs | Wait-Job | Receive-Job
$raceJobs | Remove-Job
"15. Concurrent invite race -> " + (($raceResults | ForEach-Object { "$($_.Status):$($_.Code)" }) -join ", ")
```

### 6.3 Check CloudWatch Logs & Clean Up Temporary Test Users

Confirm zero `ERROR` logs (verifying Aurora DSQL connectivity under `familyledger_app` and EventBridge `PutEvents`):

```powershell
aws logs tail "/aws/lambda/familyledger-family-service-$ENV" --since 10m --region $REGION
```

Optionally delete the temporary smoke-test Cognito users (the soft-deleted test family row only consumes negligible bytes):

```powershell
aws cognito-idp admin-delete-user --user-pool-id $USER_POOL_ID --username smoke-alice@example.com --region $REGION
aws cognito-idp admin-delete-user --user-pool-id $USER_POOL_ID --username smoke-bob@example.com --region $REGION
```

---

## 7. Troubleshooting, Rollback & Sprint 2 Sign-Off

| Symptom | Root Cause | Resolution |
| :--- | :--- | :--- |
| **`401 Unauthorized` from API Gateway** | Sent Cognito `AccessToken` instead of `IdToken`, or stale `USER_POOL_ID` / `CLIENT_ID` | Extract `AuthenticationResult.IdToken` (contains `aud` matching `CognitoWebAppClientId`) and confirm `API_URL` points to `familyledger-services-dev`. |
| **`500 INTERNAL_ERROR` with IAM/permission errors in CloudWatch** | Missing `familyledger_app` DSQL role, missing `AWS IAM GRANT`, or un-granted tables | Re-run Step 2 (`SELECT * FROM sys.iam_pg_role_mappings;` + `GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO familyledger_app;`). |
| **`404 Not Found` from Axum handler** | Deployed older binary without `strip_apigw_stage_prefix` | Rebuild `family_service` (`cargo lambda build --release --arm64 --package family_service`) and re-deploy `familyledger-services-dev`. |
| **Two API Gateways exist / API URL changed unexpectedly** | Deployed `services` before updating `core` (export names are distinct, so this is not an export collision) | Run `sam deploy` on `template.yaml` (`familyledger-core-dev`) so the legacy `FamilyLedgerApi` and `ApiGatewayUrl` export are dropped, then use the `familyledger-services-dev` `ApiGatewayUrl`. |
| **Migration Lambda times out or reports active async jobs** | Re-index job still running on DSQL | Re-invoke `familyledger-migrate-dev` (`payload '{}'`); migration runner is idempotent and waits on `CALL sys.wait_for_job`. |

### Safe Rollback
Deleting `familyledger-services-dev` cleanly removes only Sprint 2 API Gateway + Lambda resources without affecting `Cognito`, `S3`, `CloudFront`, `EventBridge`, or `Aurora DSQL`:

```powershell
sam delete --stack-name "familyledger-services-$ENV" --region $REGION
```

### Final Sprint 2 Git & Plan Sign-Off
Once live verification passes:
1. Check off the final two DoD items in [`sprint_2_plan.md`](./sprint_2_plan.md#5-sprint-2-definition-of-done-dod).
2. Review `git status` in `backend/` (including `backend/.sqlx/`), `frontend-web/` (making sure `src/environments/environment*.ts` placeholders were not overwritten with real AWS IDs), and `documentation/`, then commit your branch changes.

---

## 8. Step 8: Sprint 2 Angular Web SPA (`frontend-web/`) Deployment & Live Verification

Execute this section on your AWS deployment machine once Steps 1–7 above are complete and `familyledger-core-dev` + `familyledger-services-dev` are live.

### 8.1 Prerequisites on the Deployment Machine

- **Node.js `>= 22`** and **`pnpm >= 12`** (`corepack enable` or npm global install)
- **AWS CLI v2** authenticated against the target AWS account

```bash
node --version   # >= v22
pnpm --version   # >= 12
aws sts get-caller-identity
```

### 8.2 Collect Live `dev` Stack Outputs

Run either **PowerShell** (continuing from Steps 0–7) or **Bash/Zsh**:

**PowerShell:**
```powershell
$FRONTEND_DIR = Join-Path (Split-Path $BACKEND_DIR -Parent) "frontend-web"
$USER_POOL_ID   = aws cloudformation describe-stacks --stack-name "familyledger-core-$ENV"     --region $REGION --query "Stacks[0].Outputs[?OutputKey=='CognitoUserPoolId'].OutputValue"     --output text
$CLIENT_ID      = aws cloudformation describe-stacks --stack-name "familyledger-core-$ENV"     --region $REGION --query "Stacks[0].Outputs[?OutputKey=='CognitoWebAppClientId'].OutputValue" --output text
$CLOUDFRONT_URL = aws cloudformation describe-stacks --stack-name "familyledger-core-$ENV"     --region $REGION --query "Stacks[0].Outputs[?OutputKey=='CloudFrontUrl'].OutputValue"         --output text
$API_URL        = aws cloudformation describe-stacks --stack-name "familyledger-services-$ENV" --region $REGION --query "Stacks[0].Outputs[?OutputKey=='ApiGatewayUrl'].OutputValue"         --output text
$COGNITO_DOMAIN = "https://familyledger-$ENV-$ACCOUNT_ID.auth.$REGION.amazoncognito.com"
$SPA_BUCKET     = "familyledger-spa-$ENV-$ACCOUNT_ID"
$DIST_ID        = aws cloudformation list-stack-resources --stack-name "familyledger-core-$ENV" --region $REGION --query "StackResourceSummaries[?LogicalResourceId=='FamilyLedgerCdn'].PhysicalResourceId" --output text
```

**Bash / Zsh:**
```bash
export REGION="us-east-1"
export ENV="dev"
export ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export USER_POOL_ID="$(aws cloudformation describe-stacks --stack-name "familyledger-core-$ENV" --region "$REGION" --query "Stacks[0].Outputs[?OutputKey=='CognitoUserPoolId'].OutputValue" --output text)"
export CLIENT_ID="$(aws cloudformation describe-stacks --stack-name "familyledger-core-$ENV" --region "$REGION" --query "Stacks[0].Outputs[?OutputKey=='CognitoWebAppClientId'].OutputValue" --output text)"
export CLOUDFRONT_URL="$(aws cloudformation describe-stacks --stack-name "familyledger-core-$ENV" --region "$REGION" --query "Stacks[0].Outputs[?OutputKey=='CloudFrontUrl'].OutputValue" --output text)"
export API_URL="$(aws cloudformation describe-stacks --stack-name "familyledger-services-$ENV" --region "$REGION" --query "Stacks[0].Outputs[?OutputKey=='ApiGatewayUrl'].OutputValue" --output text)"
export COGNITO_DOMAIN="https://familyledger-${ENV}-${ACCOUNT_ID}.auth.${REGION}.amazoncognito.com"
export SPA_BUCKET="familyledger-spa-${ENV}-${ACCOUNT_ID}"
export DIST_ID="$(aws cloudformation list-stack-resources --stack-name "familyledger-core-$ENV" --region "$REGION" --query "StackResourceSummaries[?LogicalResourceId=='FamilyLedgerCdn'].PhysicalResourceId" --output text)"
```

> [!WARNING]
> **Never commit live AWS account IDs or endpoints.** Leave `frontend-web/src/environments/environment*.ts` placeholders committed in Git and restore them with `git checkout -- src/environments/` after building/testing.

### 8.3 Option A: Run SPA Locally (`http://localhost:4200`) Against Live AWS `dev`

[`services-template.yaml`](../../../backend/infrastructure/sam/services-template.yaml) already permits `http://localhost:4200` in CORS when `Environment=dev`, and [`template.yaml`](../../../backend/infrastructure/sam/template.yaml) defaults Cognito callbacks to `http://localhost:4200/auth/{callback,logout}`.

1. Populate `frontend-web/src/environments/environment.ts` locally:
   ```ts
   import type { AppConfig } from '../app/core/config/app-config';

   export const environment: AppConfig = {
     production: false,
     apiUrl: '<API_URL without trailing slash, e.g. https://xxxx.execute-api.us-east-1.amazonaws.com/dev>',
     cognitoDomain: '<COGNITO_DOMAIN>',
     cognitoClientId: '<CLIENT_ID>',
     callbackUrl: 'http://localhost:4200/auth/callback',
     logoutUrl: 'http://localhost:4200/auth/logout',
   };
   ```
2. Install dependencies, run local verification gates, and start the dev server:
   ```bash
   cd frontend-web
   pnpm install --frozen-lockfile
   pnpm lint && pnpm test
   pnpm start
   ```
3. Open `http://localhost:4200` and execute the Browser Smoke Checklist in §8.5.

### 8.4 Option B: Production Build & Deploy SPA to AWS S3 + CloudFront (`dev`)

1. **Align Cognito Callback & Logout URLs with CloudFront**:
   [`template.yaml`](../../../backend/infrastructure/sam/template.yaml) accepts a single `CognitoCallbackUrl` / `CognitoLogoutUrl`. If you want CloudFront **only**, redeploy `core` with overrides; if you want **both** `http://localhost:4200` and `$CLOUDFRONT_URL` during `dev` testing, update the User Pool Client directly via AWS CLI (remembering a subsequent `sam deploy` on `core` resets to the single CloudFormation parameter value):
   ```powershell
   # Keep BOTH CloudFront and localhost:4200 active in dev:
   aws cognito-idp update-user-pool-client `
     --user-pool-id $USER_POOL_ID `
     --client-id $CLIENT_ID `
     --region $REGION `
     --allowed-o-auth-flows code `
     --allowed-o-auth-flows-user-pool-client `
     --allowed-o-auth-scopes openid email profile `
     --supported-identity-providers COGNITO `
     --explicit-auth-flows ALLOW_USER_SRP_AUTH ALLOW_REFRESH_TOKEN_AUTH `
     --callback-ur-ls "$CLOUDFRONT_URL/auth/callback" "http://localhost:4200/auth/callback" `
     --logout-ur-ls "$CLOUDFRONT_URL/auth/logout" "http://localhost:4200/auth/logout"
   ```
   *(If Google OIDC is configured via `GoogleClientId`, include `Google` alongside `COGNITO` in `--supported-identity-providers`.)*

2. **Configure `frontend-web/src/environments/environment.prod.ts`**:
   ```ts
   import type { AppConfig } from '../app/core/config/app-config';

   export const environment: AppConfig = {
     production: true,
     apiUrl: '<API_URL without trailing slash>',
     cognitoDomain: '<COGNITO_DOMAIN>',
     cognitoClientId: '<CLIENT_ID>',
     callbackUrl: '<CLOUDFRONT_URL>/auth/callback',
     logoutUrl: '<CLOUDFRONT_URL>/auth/logout',
   };
   ```

3. **Run Quality Gates & Production Build**:
   ```bash
   cd frontend-web
   pnpm install --frozen-lockfile
   pnpm lint && pnpm test && pnpm build
   ```
   Confirm `dist/family-ledger-web/browser` exists and initial bundle budgets remain under `500 kB`.

4. **Upload Assets to S3 (2-Step Cache Strategy) & Invalidate CloudFront**:
   ```powershell
   # 1. Sync hashed JS/CSS/font assets with immutable 1-year browser/CDN caching
   aws s3 sync "$FRONTEND_DIR\dist\family-ledger-web\browser" "s3://$SPA_BUCKET" `
     --region $REGION `
     --delete `
     --exclude "index.html" `
     --cache-control "public,max-age=31536000,immutable"

   # 2. Upload index.html with no-cache so new hashed chunks take effect immediately
   aws s3 cp "$FRONTEND_DIR\dist\family-ledger-web\browser\index.html" "s3://$SPA_BUCKET/index.html" `
     --region $REGION `
     --content-type "text/html; charset=utf-8" `
     --cache-control "no-cache,no-store,must-revalidate"

   # 3. Invalidate CloudFront edge cache
   aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/*"
   ```
   ```bash
   # Equivalent Bash/Zsh commands:
   aws s3 sync dist/family-ledger-web/browser "s3://${SPA_BUCKET}" \
     --region "$REGION" --delete --exclude "index.html" \
     --cache-control "public,max-age=31536000,immutable"
   aws s3 cp dist/family-ledger-web/browser/index.html "s3://${SPA_BUCKET}/index.html" \
     --region "$REGION" --content-type "text/html; charset=utf-8" \
     --cache-control "no-cache,no-store,must-revalidate"
   aws cloudfront create-invalidation --distribution-id "$DIST_ID" --paths "/*"
   ```

5. **Restore Git Placeholders** (so AWS IDs do not stay dirty in your working tree):
   ```bash
   git -C frontend-web checkout -- src/environments/environment.ts src/environments/environment.prod.ts
   ```

### 8.5 Sprint 2 Live Browser E2E & Zero-CSP-Violation Smoke Checklist

Test against `$CLOUDFRONT_URL` (or `http://localhost:4200`) using two browser profiles/incognito sessions (**User A Owner** and **User B Member/Other**):

1. **Auth & Onboarding (User A)**: Open `$CLOUDFRONT_URL` -> redirected to Cognito Hosted UI PKCE login -> sign up / sign in with verified email -> land on `/onboarding` -> create workspace (`POST /families`) -> redirected to `/workspace` Overview.
2. **Overview & Settings (User A)**: Open `/workspace/settings`, update family name and `HomeDisplayCurrency` (`PUT /families/{id}`), confirm top-bar currency badge updates immediately.
3. **Owner Invite with Custom Permissions (User A)**: Click **Invite Member**, select **Other** (verify `Owner` is excluded), tick custom permissions (e.g., `view_budget`, `view_reports`), submit (`POST /families/{id}/invites`), copy `/join?familyId=...&token=...`.
4. **Unauthenticated Deep-Link Join Round-Trip (User B)**: Paste `/join?familyId=...&token=...` into an incognito window -> redirected to Cognito Hosted UI -> sign in -> confirm auto-redirect back to `/join?...` -> enter display name -> submit (`POST /families/{id}/members` omitting `X-Family-Id`) -> confirm auto-activation into joined workspace.
5. **Members Directory & Owner Governance**:
   - **User B (`Other`)** navigates to `/workspace/members`: verify read-only banner is shown and `Invite Member` / `Edit Role` / `Remove` buttons are hidden.
   - **User A (`Owner`)** navigates to `/workspace/members`: verify Owner row shows `Protected (Owner)`; change User B's role via **Edit Role** (`PATCH .../role`) -> verify table updates; remove User B via **Remove** (`DELETE .../members/{mid}`) -> verify User B disappears from roster.
   - **User B** reloads `/workspace` or switches workspace -> confirm `403 USER_NOT_IN_FAMILY` eviction card appears cleanly with zero raw backend diagnostics leaked.
6. **CloudFront CSP & Deep-Link Headers Check**:
   - Reload browser directly on `$CLOUDFRONT_URL/workspace/members` and `$CLOUDFRONT_URL/join` -> confirm HTTP `200` (CloudFront 403/404 -> `/index.html`).
   - Open DevTools Console & Network -> confirm **0 Content-Security-Policy (CSP) violations** under `FamilyLedgerSecurityHeadersPolicy` (`script-src 'self'`, `style-src 'self' 'unsafe-inline'`, `connect-src` allowing API Gateway & Cognito), clean EN/RU toggle, and all 4 OKLCH themes × Light/Dark work smoothly.
