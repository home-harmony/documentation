# Task 1.1 Detailed Guide — IAM & CloudFormation/SAM Template Setup

> **Goal**: Define all AWS infrastructure for the FamilyLedger project as code (Infrastructure-as-Code / IaC) using a single AWS SAM template file (`template.yaml`). By the end you will have a deployable template that provisions: Cognito User Pool, API Gateway with JWT auth, S3 bucket for the Angular SPA, and a CloudFront distribution serving the SPA.

---

## What You Will Build (Mental Model)

```
                       Internet
                          │
              ┌───────────▼───────────┐
              │    CloudFront CDN     │  ← Serves the Angular web app fast
              └───────────┬───────────┘    from edge locations worldwide
                          │ (fetches files)
              ┌───────────▼───────────┐
              │    S3 Bucket          │  ← Stores the Angular build output
              │  (private, locked)    │     index.html, main.js, styles.css
              └───────────────────────┘

              ┌───────────────────────┐
              │  Cognito User Pool    │  ← Stores user accounts + issues JWTs
              └───────────┬───────────┘
                          │ (validates tokens)
              ┌───────────▼───────────┐
              │  API Gateway HTTP API │  ← HTTPS front-door to all Rust Lambdas
              └───────────────────────┘
```

---

## Prerequisites: AWS Concepts Explained

### What is AWS SAM?
**AWS Serverless Application Model (SAM)** is a tool that lets you describe your entire AWS infrastructure in one YAML file (`template.yaml`). Instead of clicking through the AWS Console for every resource, you write the configuration once and deploy it with a single command. SAM is built on top of CloudFormation and adds shorthand for Lambda, API Gateway, and other serverless resources.

### What is IAM?
**Identity and Access Management (IAM)** controls *who* (users, services, Lambdas) can do *what* (read S3, invoke Lambda) on *which* AWS resources. Think of it as a permissions system. In our template we define IAM Roles that allow Lambdas to talk to Aurora DSQL, CloudWatch, etc.

### What is AWS Cognito?
**Amazon Cognito** is a managed user authentication service. It:
- Stores user accounts (email + hashed password)
- Handles registration/login flows
- Issues **JWT tokens** (JSON Web Tokens) on successful login — a signed proof that "this user is authenticated"
- API Gateway can validate these JWTs automatically, rejecting requests without a valid token

### What is API Gateway (HTTP API)?
**Amazon API Gateway HTTP API** is the HTTPS front-door to your Rust Lambda functions. Every request to `https://api.familyledger.com/families` goes through API Gateway, which:
1. Validates the JWT token against Cognito (rejects unauthenticated requests)
2. Forwards valid requests to the correct Lambda function

### What is S3?
**Amazon S3 (Simple Storage Service)** is a file storage service. We use it to store the compiled Angular SPA files (`index.html`, JS bundles, CSS). The bucket will be **private** — users cannot access it directly; only CloudFront can.

### What is CloudFront?
**Amazon CloudFront** is a CDN (Content Delivery Network). It:
- Caches your Angular files at edge locations worldwide (fast for users in any country)
- Handles HTTPS termination
- Implements the **SPA routing fallback**: if the user visits `/families/123`, CloudFront returns `index.html` instead of a 404 (Angular's router then handles the URL client-side)

### What is Origin Access Control (OAC)?
**OAC** is the secure bridge between CloudFront and S3. It lets CloudFront read from a **private** S3 bucket using IAM-style signatures, without making the bucket public. Users can only get files via CloudFront — they cannot access S3 directly.

---

## Step 1: Install Prerequisites

### 1.1 — Install the AWS CLI

The AWS CLI lets you interact with AWS from PowerShell.

1. Download the MSI installer from: https://aws.amazon.com/cli/
2. Run it and follow the wizard
3. Verify:
```powershell
aws --version
# Expected: aws-cli/2.x.x ...
```

### 1.2 — Configure AWS CLI with your credentials

You need an AWS account. If you don't have one, create one at https://aws.amazon.com.

Then create an IAM user (or use IAM Identity Center):

1. Go to **AWS Console → IAM → Users → Create User**
2. Give it a name like `familyledger-dev`
3. Attach the managed policy **`AdministratorAccess`** (for development only — we'll restrict this later)
4. Go to **Security credentials → Create access key → CLI use case**
5. Copy the **Access Key ID** and **Secret Access Key**

Now configure the CLI:
```powershell
aws configure
# AWS Access Key ID: [paste your key]
# AWS Secret Access Key: [paste your secret]
# Default region name: us-east-1
# Default output format: json
```

Verify it works:
```powershell
aws sts get-caller-identity
# Should return your account ID and user ARN
```

### 1.3 — Install AWS SAM CLI

SAM CLI is the tool that deploys your `template.yaml`.

1. Download from: https://github.com/aws/aws-sam-cli/releases/latest (pick the `.msi` for Windows)
2. Install and verify:
```powershell
sam --version
# Expected: SAM CLI, version 1.x.x
```

---

## Step 2: Create the Project Structure

In your `home-harmony/` folder, create the following layout for the IaC files:

```
home-harmony/
├── infrastructure/
│   └── sam/
│       ├── template.yaml        ← The main SAM/CloudFormation template (you'll write this)
│       └── samconfig.toml       ← SAM deployment configuration
└── ... (Rust crates will go here in later tasks)
```

Run in PowerShell:
```powershell
New-Item -ItemType Directory -Path "c:\Users\demiu\my-rust-projects\home-harmony\infrastructure\sam" -Force
```

---

## Step 3: Write the SAM Template (`template.yaml`)

Create `infrastructure/sam/template.yaml`. We will build it section by section.

> [!IMPORTANT]
> YAML is **whitespace-sensitive**. Always use 2-space indentation. Never mix tabs and spaces.

### 3.1 — Template Header and Global Configuration

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31      # Enables SAM shorthand syntax

Description: "FamilyLedger — Core serverless infrastructure (Cognito, API Gateway, S3, CloudFront)"

# ─── Parameters ───────────────────────────────────────────────────────────────
# Parameters let you pass different values for different environments
# (e.g., dev vs prod) without changing the template file.
Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
    Description: Deployment environment name

  CognitoCallbackUrl:
    Type: String
    Default: http://localhost:4200/auth/callback
    Description: >
      URL that Cognito redirects to after login. 
      In dev this is your local Angular server.
      In prod this will be your CloudFront domain.

  CognitoLogoutUrl:
    Type: String
    Default: http://localhost:4200/auth/logout
    Description: URL that Cognito redirects to after logout.
```

**Explanation:**
- `AWSTemplateFormatVersion` — required boilerplate; always this exact value
- `Transform: AWS::Serverless-2016-10-31` — enables SAM syntax (like `AWS::Serverless::Function`)
- `Parameters` — values you pass at deploy time; `Default` is used if you don't pass one

---

### 3.2 — Global SAM Defaults

```yaml
# ─── Globals ──────────────────────────────────────────────────────────────────
# These settings apply to ALL Lambda functions in the template,
# so you don't have to repeat them on each function.
Globals:
  Function:
    Runtime: provided.al2023     # Our Rust binaries use the "provided" runtime (ARM64 custom runtime)
    Architectures: [arm64]       # ARM64 is cheaper and faster for Rust Lambda
    Timeout: 30                  # Max seconds a Lambda can run before timing out
    MemorySize: 256              # MB of RAM; Rust is very memory-efficient
    Environment:
      Variables:
        ENVIRONMENT: !Ref Environment  # Injects the "Environment" parameter as an env var
```

---

### 3.3 — Cognito User Pool

The **User Pool** is the database of user accounts. Think of it as the "users table" managed by AWS.

```yaml
# ─── Resources ────────────────────────────────────────────────────────────────
# Everything in Resources gets actually created in AWS when you deploy.
Resources:

  # ════════════════════════════════════════════════════════════════════════════
  # SECTION 1: Authentication — Cognito User Pool
  # ════════════════════════════════════════════════════════════════════════════

  FamilyLedgerUserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: !Sub "familyledger-users-${Environment}"   # e.g. "familyledger-users-dev"
      
      # ── How users sign in ──────────────────────────────────────────────────
      UsernameAttributes:
        - email                 # Users log in with their email address (not a username)
      
      AutoVerifiedAttributes:
        - email                 # Cognito automatically sends a verification email
      
      # ── Password policy ────────────────────────────────────────────────────
      Policies:
        PasswordPolicy:
          MinimumLength: 12
          RequireUppercase: true
          RequireLowercase: true
          RequireNumbers: true
          RequireSymbols: false  # Keep it friendly — no need for $ # @ etc.
      
      # ── Email verification message ─────────────────────────────────────────
      VerificationMessageTemplate:
        DefaultEmailOption: CONFIRM_WITH_CODE   # Send a 6-digit code (not a link)
        EmailSubject: "Your FamilyLedger verification code"
        EmailMessage: "Your verification code is {####}"
      
      # ── Token expiry settings ──────────────────────────────────────────────
      # JWT = JSON Web Token. Cognito issues two tokens on login:
      #   - Access Token: short-lived, used to call your API (1 hour)
      #   - Refresh Token: long-lived, used to get a new Access Token silently
      UserPoolAddOns:
        AdvancedSecurityMode: AUDIT  # Audit mode logs anomalies; doesn't block
      
      # ── Custom attributes stored on each user ─────────────────────────────
      # "sub" (subject) is built-in — it's the unique UUID Cognito assigns each user.
      # We can add custom attributes:
      Schema:
        - Name: family_id          # UUID of the family this user belongs to
          AttributeDataType: String
          Mutable: true
          Required: false
        - Name: family_role        # 'owner' | 'member' | 'child' | 'other'
          AttributeDataType: String
          Mutable: true
          Required: false
```

> [!NOTE]
> The `sub` attribute (Cognito's auto-generated UUID for each user) becomes the `user_id` stored in your `family_members` table. When a Lambda validates the JWT, it reads `sub` from the token claims.

---

### 3.4 — Cognito App Client

An **App Client** represents *one application* (Angular SPA, Flutter app) that is allowed to talk to your User Pool. Each client gets its own `client_id`.

```yaml
  FamilyLedgerWebAppClient:
    Type: AWS::Cognito::UserPoolClient
    Properties:
      ClientName: !Sub "familyledger-web-${Environment}"
      UserPoolId: !Ref FamilyLedgerUserPool    # Links this client to the User Pool above
      
      # ── Security settings ─────────────────────────────────────────────────
      GenerateSecret: false      # SPAs (Angular) cannot securely store a secret → no secret
      
      # ── Which OAuth flows are allowed ─────────────────────────────────────
      # PKCE (Proof Key for Code Exchange) is the secure flow for browser apps.
      # It prevents authorization code interception attacks.
      AllowedOAuthFlows:
        - code                   # Authorization Code flow (with PKCE)
      
      AllowedOAuthFlowsUserPoolClient: true
      
      # ── Allowed OAuth scopes ──────────────────────────────────────────────
      # These are the permissions the Angular app requests from Cognito.
      AllowedOAuthScopes:
        - openid                 # Required: enables ID token with user info
        - email                  # Include email in the ID token
        - profile                # Include name/profile info in the ID token
      
      # ── Redirect URLs ─────────────────────────────────────────────────────
      # After login, Cognito redirects to one of these URLs with the auth code.
      # Angular exchanges the code for tokens.
      CallbackURLs:
        - !Ref CognitoCallbackUrl   # http://localhost:4200/auth/callback (dev)
      
      LogoutURLs:
        - !Ref CognitoLogoutUrl     # http://localhost:4200/auth/logout (dev)
      
      # ── Token validity ────────────────────────────────────────────────────
      AccessTokenValidity: 1         # 1 hour — short, for security
      IdTokenValidity: 1             # 1 hour
      RefreshTokenValidity: 30       # 30 days — user stays logged in for a month
      TokenValidityUnits:
        AccessToken: hours
        IdToken: hours
        RefreshToken: days
      
      # ── Which auth flows are enabled ──────────────────────────────────────
      ExplicitAuthFlows:
        - ALLOW_USER_SRP_AUTH         # SRP = Secure Remote Password; used by Amplify SDK
        - ALLOW_REFRESH_TOKEN_AUTH    # Allow the app to refresh tokens silently
      
      # ── Attribute read/write permissions for the client ───────────────────
      ReadAttributes:
        - email
        - email_verified
        - custom:family_id
        - custom:family_role
      
      WriteAttributes:
        - email
        - custom:family_id
        - custom:family_role
```

---

### 3.5 — Cognito User Pool Domain

The **domain** provides a hosted UI for login/registration. Cognito hosts the login page for you — you don't have to build one initially.

```yaml
  FamilyLedgerUserPoolDomain:
    Type: AWS::Cognito::UserPoolDomain
    Properties:
      # The domain becomes: familyledger-dev.auth.us-east-1.amazoncognito.com
      Domain: !Sub "familyledger-${Environment}"
      UserPoolId: !Ref FamilyLedgerUserPool
```

---

### 3.6 — API Gateway HTTP API with JWT Authorizer

**API Gateway HTTP API** is the HTTPS entry point. The JWT Authorizer validates the Cognito token on every request.

```yaml
  # ════════════════════════════════════════════════════════════════════════════
  # SECTION 2: API Gateway HTTP API
  # ════════════════════════════════════════════════════════════════════════════

  FamilyLedgerApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      StageName: !Ref Environment    # Creates URL like: https://xxx.execute-api.us-east-1.amazonaws.com/dev/

      # ── CORS Configuration ────────────────────────────────────────────────
      # CORS (Cross-Origin Resource Sharing) allows your Angular app at localhost:4200
      # or at your CloudFront domain to call the API. Without this, the browser blocks requests.
      CorsConfiguration:
        AllowOrigins:
          - "http://localhost:4200"                # Angular dev server
          - !Sub "https://${FamilyLedgerCdn}.cloudfront.net"  # CloudFront domain (prod)
        AllowHeaders:
          - Content-Type
          - Authorization    # The JWT token is sent in this header
          - X-Amz-Date
          - X-Api-Key
        AllowMethods:
          - GET
          - POST
          - PUT
          - PATCH
          - DELETE
          - OPTIONS           # OPTIONS is used by browsers for preflight checks
        MaxAge: 300           # Cache preflight result for 5 minutes

      # ── JWT Authorizer ────────────────────────────────────────────────────
      # This tells API Gateway: "validate every request's Authorization header
      # against our Cognito User Pool's public keys".
      Auth:
        DefaultAuthorizer: CognitoJwtAuthorizer
        Authorizers:
          CognitoJwtAuthorizer:
            IdentitySource: "$request.header.Authorization"  # Look for token here
            JwtConfiguration:
              # The "issuer" is the Cognito URL that published the signing keys.
              # API Gateway fetches the public keys from here automatically.
              Issuer: !Sub "https://cognito-idp.${AWS::Region}.amazonaws.com/${FamilyLedgerUserPool}"
              # The "audience" is the App Client ID — ensures tokens are meant for THIS app
              Audience:
                - !Ref FamilyLedgerWebAppClient
```

> [!NOTE]
> How JWT validation works: When Angular calls `POST /families`, it includes `Authorization: Bearer eyJ...` in the header. API Gateway verifies the JWT signature against Cognito's public key, checks it's not expired, and checks the audience matches `FamilyLedgerWebAppClient`. Only then does the request reach your Lambda. **Your Lambda never needs to validate tokens itself** — API Gateway already did it.

---

### 3.7 — S3 Bucket for Angular SPA

The S3 bucket stores the compiled Angular app files. It is **private** — CloudFront is the only authorized reader.

```yaml
  # ════════════════════════════════════════════════════════════════════════════
  # SECTION 3: S3 Bucket for Angular SPA Static Files
  # ════════════════════════════════════════════════════════════════════════════

  FamilyLedgerSpaBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "familyledger-spa-${Environment}-${AWS::AccountId}"
      # Note: We include AccountId to make the bucket name globally unique.
      # S3 bucket names must be unique across ALL AWS accounts worldwide.

      # ── Block ALL public access ────────────────────────────────────────────
      # No one can access this bucket directly — only CloudFront via OAC.
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

      # ── Encryption at rest ─────────────────────────────────────────────────
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256   # S3-managed encryption key

      # ── Versioning (optional but recommended) ─────────────────────────────
      # Keeps previous versions of files — useful if you deploy a bad build.
      VersioningConfiguration:
        Status: Enabled
```

---

### 3.8 — CloudFront Origin Access Control (OAC)

**OAC** is the identity that CloudFront uses when fetching files from the private S3 bucket.

```yaml
  # ════════════════════════════════════════════════════════════════════════════
  # SECTION 4: CloudFront — CDN for Angular SPA
  # ════════════════════════════════════════════════════════════════════════════

  FamilyLedgerCdnOac:
    Type: AWS::CloudFront::OriginAccessControl
    Properties:
      OriginAccessControlConfig:
        Name: !Sub "familyledger-oac-${Environment}"
        Description: "OAC for FamilyLedger SPA bucket — CloudFront only access"
        OriginAccessControlOriginType: s3       # This OAC is for an S3 origin
        SigningBehavior: always                  # Always sign requests to S3
        SigningProtocol: sigv4                   # Use AWS Signature v4 (standard)
```

---

### 3.9 — S3 Bucket Policy: Allow CloudFront to Read

We must explicitly grant the CloudFront distribution permission to read from S3.

```yaml
  FamilyLedgerSpaBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref FamilyLedgerSpaBucket
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Sid: "AllowCloudFrontServicePrincipal"
            Effect: Allow
            Principal:
              Service: cloudfront.amazonaws.com   # The CloudFront service itself
            Action: s3:GetObject                  # Only allow reading files, not listing or writing
            Resource: !Sub "${FamilyLedgerSpaBucket.Arn}/*"   # All objects in the bucket
            Condition:
              StringEquals:
                # This restricts access to ONLY our specific CloudFront distribution.
                # Even other CloudFront distributions cannot access this bucket.
                AWS:SourceArn: !Sub "arn:aws:cloudfront::${AWS::AccountId}:distribution/${FamilyLedgerCdn}"
```

> [!IMPORTANT]
> Notice `!Sub "...${FamilyLedgerCdn}"` — this references the CloudFront distribution resource we define next. CloudFormation resolves these references automatically at deploy time.

---

### 3.10 — CloudFront Distribution

The CloudFront distribution ties everything together: it points to the S3 bucket, uses OAC, and implements the SPA routing fallback.

```yaml
  FamilyLedgerCdn:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Comment: !Sub "FamilyLedger Angular SPA CDN — ${Environment}"
        Enabled: true
        HttpVersion: http2and3       # Supports both HTTP/2 and HTTP/3 for speed
        PriceClass: PriceClass_100  # Only use edge locations in US, Canada, Europe (cheapest)
                                     # Change to PriceClass_All if you have global users

        # ── Default root object ────────────────────────────────────────────
        # When someone visits the bare CloudFront URL (e.g., https://abc123.cloudfront.net/)
        # CloudFront serves this file from S3.
        DefaultRootObject: index.html

        # ── Origins: Where CloudFront fetches files from ───────────────────
        Origins:
          - Id: S3SpaOrigin             # A logical ID we give this origin
            DomainName: !GetAtt FamilyLedgerSpaBucket.RegionalDomainName
            # Important: Use RegionalDomainName (not DomainName) to avoid
            # redirect issues when the bucket was just created.

            # Use OAC (not the legacy OAI) for secure access
            OriginAccessControlId: !GetAtt FamilyLedgerCdnOac.Id
            S3OriginConfig:
              OriginAccessIdentity: ""  # Must be empty when using OAC

        # ── Default Cache Behavior ─────────────────────────────────────────
        # This rule applies to all requests (/*) unless overridden.
        DefaultCacheBehavior:
          TargetOriginId: S3SpaOrigin    # Fetch from S3
          ViewerProtocolPolicy: redirect-to-https   # Force HTTPS; redirect HTTP → HTTPS
          CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6
          # ^ This is the AWS-managed "CachingOptimized" cache policy ID (do NOT change)
          # It caches based on Accept-Encoding header and compresses responses (gzip/br).

          # Forward these headers to the origin (S3 ignores them but CloudFront uses them)
          AllowedMethods: [GET, HEAD, OPTIONS]
          CachedMethods: [GET, HEAD]

          # ── Response Headers Policy ──────────────────────────────────────
          # Adds security headers to all responses (best practice for web security)
          ResponseHeadersPolicyId: !Ref FamilyLedgerResponseHeadersPolicy

        # ── Custom Error Responses (SPA Routing Fallback) ─────────────────
        # This is the critical SPA configuration. Here's the problem it solves:
        #
        # A user shares a URL: https://your-cdn.net/families/abc123/transactions
        # CloudFront asks S3 for the file at path /families/abc123/transactions
        # S3 returns 403 (no such file) — because Angular routes are not real files!
        # Without this config, the user sees an error.
        #
        # Fix: When S3 returns 403 or 404, CloudFront instead returns index.html
        # with HTTP 200. Angular's router then reads the URL and renders the right page.
        CustomErrorResponses:
          - ErrorCode: 403          # S3 returns 403 for missing objects (because bucket is private)
            ResponseCode: 200       # Tell the browser: "everything is fine"
            ResponsePagePath: /index.html   # Serve Angular's entry point
            ErrorCachingMinTTL: 0   # Don't cache error responses

          - ErrorCode: 404          # Standard "not found" — also handled for completeness
            ResponseCode: 200
            ResponsePagePath: /index.html
            ErrorCachingMinTTL: 0

  # ── Security Response Headers Policy ────────────────────────────────────────
  # These HTTP headers improve browser security for all users of the app.
  FamilyLedgerResponseHeadersPolicy:
    Type: AWS::CloudFront::ResponseHeadersPolicy
    Properties:
      ResponseHeadersPolicyConfig:
        Name: !Sub "familyledger-security-headers-${Environment}"
        SecurityHeadersConfig:
          ContentTypeOptions:
            Override: true              # X-Content-Type-Options: nosniff
          FrameOptions:
            FrameOption: DENY           # X-Frame-Options: DENY — prevents clickjacking
            Override: true
          ReferrerPolicy:
            ReferrerPolicy: strict-origin-when-cross-origin
            Override: true
          StrictTransportSecurity:
            AccessControlMaxAgeSec: 63072000   # 2 years — HTTPS only, including subdomains
            IncludeSubdomains: true
            Preload: true
            Override: true
          ContentSecurityPolicy:
            ContentSecurityPolicy: "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline' fonts.googleapis.com; font-src fonts.gstatic.com; connect-src 'self' https://*.amazonaws.com; img-src 'self' data:"
            Override: true
```

---

### 3.11 — Outputs Section

Outputs expose important values after deployment (like the CloudFront URL or Cognito Pool ID) so you can use them elsewhere.

```yaml
# ─── Outputs ──────────────────────────────────────────────────────────────────
# These values are displayed after `sam deploy` completes.
# They can also be referenced by other CloudFormation stacks.
Outputs:

  CognitoUserPoolId:
    Description: "Cognito User Pool ID — needed for Angular Amplify configuration"
    Value: !Ref FamilyLedgerUserPool
    Export:
      Name: !Sub "${AWS::StackName}-CognitoUserPoolId"

  CognitoWebAppClientId:
    Description: "Cognito Web App Client ID — used in Angular environment.ts"
    Value: !Ref FamilyLedgerWebAppClient
    Export:
      Name: !Sub "${AWS::StackName}-CognitoWebClientId"

  CognitoDomain:
    Description: "Cognito Hosted UI domain — use for OAuth redirect"
    Value: !Sub "https://familyledger-${Environment}.auth.${AWS::Region}.amazoncognito.com"

  ApiGatewayUrl:
    Description: "API Gateway base URL — use in Angular environment.ts"
    Value: !Sub "https://${FamilyLedgerApi}.execute-api.${AWS::Region}.amazonaws.com/${Environment}"
    Export:
      Name: !Sub "${AWS::StackName}-ApiUrl"

  SpaBucketName:
    Description: "S3 bucket name for Angular SPA deployment"
    Value: !Ref FamilyLedgerSpaBucket
    Export:
      Name: !Sub "${AWS::StackName}-SpaBucketName"

  CloudFrontUrl:
    Description: "CloudFront distribution URL — the public URL of your Angular app"
    Value: !Sub "https://${FamilyLedgerCdn}.cloudfront.net"
    Export:
      Name: !Sub "${AWS::StackName}-CloudFrontUrl"

  CloudFrontDistributionId:
    Description: "CloudFront distribution ID — needed for cache invalidation after deploy"
    Value: !Ref FamilyLedgerCdn
    Export:
      Name: !Sub "${AWS::StackName}-CloudFrontDistributionId"
```

---

## Step 4: Create the SAM Deploy Configuration

Create `infrastructure/sam/samconfig.toml`. This file stores the default deployment parameters so you don't have to type them every time.

```toml
version = 0.1

[default]
[default.global.parameters]
stack_name = "familyledger-dev"
region = "us-east-1"
confirm_changeset = true

[default.deploy.parameters]
capabilities = "CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND"
parameter_overrides = "Environment=dev CognitoCallbackUrl=http://localhost:4200/auth/callback CognitoLogoutUrl=http://localhost:4200/auth/logout"
s3_bucket = ""              # SAM will create a deployment bucket for you on first run
s3_prefix = "familyledger"
resolve_s3 = true           # Automatically create/use the SAM deployment bucket
```

---

## Step 5: Deploy the Template

### 5.1 — First deployment (guided)

On the very first deploy, run the guided mode to let SAM create the S3 bucket for deployment artifacts:

```powershell
cd c:\Users\demiu\my-rust-projects\home-harmony\infrastructure\sam
sam deploy --guided
```

SAM will ask you questions — accept defaults or enter the values from `samconfig.toml`. It will:
1. Upload the template to a new S3 bucket it creates
2. Show you the **changeset** (a list of resources it will create)
3. Ask for confirmation
4. Create all resources

**Expected output:**
```
CloudFormation outputs from deployed stack
-------------------------------------------------
Key         CognitoUserPoolId
Value       us-east-1_XXXXXXXXX

Key         ApiGatewayUrl  
Value       https://abc123.execute-api.us-east-1.amazonaws.com/dev

Key         CloudFrontUrl
Value       https://d1234abcd.cloudfront.net
```

### 5.2 — Subsequent deployments

```powershell
sam deploy
# Uses settings from samconfig.toml automatically
```

---

## Step 6: Verify the Deployment

### 6.1 — Check Cognito User Pool

```powershell
aws cognito-idp list-user-pools --max-results 10
# Should list "familyledger-users-dev"
```

### 6.2 — Check API Gateway

```powershell
aws apigatewayv2 get-apis
# Should list your HTTP API
```

### 6.3 — Check S3 Bucket

```powershell
aws s3 ls | findstr familyledger
# Should show your bucket
```

### 6.4 — Check CloudFront

```powershell
aws cloudfront list-distributions --query "DistributionList.Items[*].[DomainName,Comment]" --output table
# Should show your distribution domain and comment
```

### 6.5 — Test the SPA fallback

After uploading a test `index.html` to S3:
```powershell
# Upload a test file
echo "<h1>FamilyLedger SPA</h1>" | aws s3 cp - s3://familyledger-spa-dev-ACCOUNTID/index.html --content-type "text/html"

# Test the CloudFront URL (wait ~5 minutes for DNS propagation)
curl https://d1234abcd.cloudfront.net/

# Test SPA routing fallback — this URL has no matching file in S3
curl https://d1234abcd.cloudfront.net/families/test
# Should return the same index.html content (200 OK), not a 404
```

---

## Step 7: Tear Down (When Needed)

To delete all resources and avoid charges:
```powershell
aws cloudformation delete-stack --stack-name familyledger-dev
# Wait for completion:
aws cloudformation wait stack-delete-complete --stack-name familyledger-dev
```

> [!CAUTION]
> This deletes everything including the S3 bucket contents. Use only when you're done with the environment.

---

## Summary of What You Built

| Resource | Logical ID in Template | Purpose |
|---|---|---|
| Cognito User Pool | `FamilyLedgerUserPool` | Stores user accounts, sends verification emails |
| Cognito App Client | `FamilyLedgerWebAppClient` | Allows Angular SPA to authenticate users via PKCE OAuth |
| Cognito Domain | `FamilyLedgerUserPoolDomain` | Provides hosted login UI |
| API Gateway HTTP API | `FamilyLedgerApi` | HTTPS entry point for all Rust Lambda functions, with JWT validation |
| S3 Bucket | `FamilyLedgerSpaBucket` | Private storage for Angular SPA files |
| S3 Bucket Policy | `FamilyLedgerSpaBucketPolicy` | Grants CloudFront read access to S3 |
| CloudFront OAC | `FamilyLedgerCdnOac` | Secure identity for CloudFront → S3 access |
| CloudFront Distribution | `FamilyLedgerCdn` | CDN + HTTPS + SPA routing fallback |
| Response Headers Policy | `FamilyLedgerResponseHeadersPolicy` | Security HTTP headers on all responses |

---

## Next Steps

After Task 1.1 is complete:
- **Task 1.2** provisions the Aurora DSQL database and Kinesis stream — also added to the same `template.yaml`
- Angular `environment.ts` will be configured with the Cognito Pool ID, Client ID, and API URL from the Outputs section
- The Flutter app will reuse the same Cognito Pool ID and Client ID (different App Client for mobile with a different callback scheme)
