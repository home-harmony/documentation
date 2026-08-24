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

### 1.2 — Configure AWS CLI with credentials
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
```

### 1.3 — Install AWS SAM CLI
1. Download from: https://github.com/aws/aws-sam-cli/releases/latest
2. Install and verify:
```powershell
sam --version
# Expected: SAM CLI, version 1.x.x
```

---

## Step 2: Create the Project Structure

In your `backend/` or `infrastructure/` workspace, create the layout for IaC files:

```
backend/
├── infrastructure/
│   └── sam/
│       ├── template.yaml        ← SAM template
│       └── samconfig.toml       ← SAM deployment configuration
```

---

## Step 3: SAM Template Reference (`template.yaml`)

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31

Description: "FamilyLedger — Core serverless infrastructure (Cognito, API Gateway, S3, CloudFront)"

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
    Description: Deployment environment name

  CognitoCallbackUrl:
    Type: String
    Default: http://localhost:4200/auth/callback
    Description: URL that Cognito redirects to after login.

  CognitoLogoutUrl:
    Type: String
    Default: http://localhost:4200/auth/logout
    Description: URL that Cognito redirects to after logout.

Globals:
  Function:
    Runtime: provided.al2023
    Architectures: [arm64]
    Timeout: 30
    MemorySize: 256
    Environment:
      Variables:
        ENVIRONMENT: !Ref Environment

Resources:
  # ─── Cognito User Pool ───────────────────────────────────────────────────────
  FamilyLedgerUserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: !Sub "familyledger-users-${Environment}"
      UsernameAttributes: [email]
      AutoVerifiedAttributes: [email]
      Policies:
        PasswordPolicy:
          MinimumLength: 12
          RequireUppercase: true
          RequireLowercase: true
          RequireNumbers: true
          RequireSymbols: false
      VerificationMessageTemplate:
        DefaultEmailOption: CONFIRM_WITH_CODE
        EmailSubject: "Your FamilyLedger verification code"
        EmailMessage: "Your verification code is {####}"
      UserPoolAddOns:
        AdvancedSecurityMode: AUDIT
      Schema:
        - Name: family_id
          AttributeDataType: String
          Mutable: true
        - Name: family_role
          AttributeDataType: String
          Mutable: true

  # ─── Cognito App Client ─────────────────────────────────────────────────────
  FamilyLedgerWebAppClient:
    Type: AWS::Cognito::UserPoolClient
    Properties:
      ClientName: !Sub "familyledger-web-${Environment}"
      UserPoolId: !Ref FamilyLedgerUserPool
      GenerateSecret: false
      AllowedOAuthFlows: [code]
      AllowedOAuthFlowsUserPoolClient: true
      AllowedOAuthScopes: [openid, email, profile]
      CallbackURLs:
        - !Ref CognitoCallbackUrl
      LogoutURLs:
        - !Ref CognitoLogoutUrl
      AccessTokenValidity: 1
      IdTokenValidity: 1
      RefreshTokenValidity: 30
      TokenValidityUnits:
        AccessToken: hours
        IdToken: hours
        RefreshToken: days
      ExplicitAuthFlows:
        - ALLOW_USER_SRP_AUTH
        - ALLOW_REFRESH_TOKEN_AUTH

  FamilyLedgerUserPoolDomain:
    Type: AWS::Cognito::UserPoolDomain
    Properties:
      Domain: !Sub "familyledger-${Environment}"
      UserPoolId: !Ref FamilyLedgerUserPool

  # ─── API Gateway HTTP API ───────────────────────────────────────────────────
  FamilyLedgerApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      StageName: !Ref Environment
      CorsConfiguration:
        AllowOrigins:
          - "http://localhost:4200"
          - !Sub "https://${FamilyLedgerCdn}.cloudfront.net"
        AllowHeaders: [Content-Type, Authorization, X-Amz-Date, X-Api-Key]
        AllowMethods: [GET, POST, PUT, PATCH, DELETE, OPTIONS]
        MaxAge: 300
      Auth:
        DefaultAuthorizer: CognitoJwtAuthorizer
        Authorizers:
          CognitoJwtAuthorizer:
            IdentitySource: "$request.header.Authorization"
            JwtConfiguration:
              Issuer: !Sub "https://cognito-idp.${AWS::Region}.amazonaws.com/${FamilyLedgerUserPool}"
              Audience:
                - !Ref FamilyLedgerWebAppClient

  # ─── S3 Bucket for Angular SPA ──────────────────────────────────────────────
  FamilyLedgerSpaBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "familyledger-spa-${Environment}-${AWS::AccountId}"
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      VersioningConfiguration:
        Status: Enabled

  # ─── CloudFront OAC ─────────────────────────────────────────────────────────
  FamilyLedgerCdnOac:
    Type: AWS::CloudFront::OriginAccessControl
    Properties:
      OriginAccessControlConfig:
        Name: !Sub "familyledger-oac-${Environment}"
        Description: "OAC for FamilyLedger SPA bucket"
        OriginAccessControlOriginType: s3
        SigningBehavior: always
        SigningProtocol: sigv4

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
              Service: cloudfront.amazonaws.com
            Action: s3:GetObject
            Resource: !Sub "${FamilyLedgerSpaBucket.Arn}/*"
            Condition:
              StringEquals:
                AWS:SourceArn: !Sub "arn:aws:cloudfront::${AWS::AccountId}:distribution/${FamilyLedgerCdn}"

  # ─── CloudFront Distribution ────────────────────────────────────────────────
  FamilyLedgerCdn:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Comment: !Sub "FamilyLedger Angular SPA CDN — ${Environment}"
        Enabled: true
        HttpVersion: http2and3
        PriceClass: PriceClass_100
        DefaultRootObject: index.html
        Origins:
          - Id: S3SpaOrigin
            DomainName: !GetAtt FamilyLedgerSpaBucket.RegionalDomainName
            OriginAccessControlId: !GetAtt FamilyLedgerCdnOac.Id
            S3OriginConfig:
              OriginAccessIdentity: ""
        DefaultCacheBehavior:
          TargetOriginId: S3SpaOrigin
          ViewerProtocolPolicy: redirect-to-https
          CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6
          AllowedMethods: [GET, HEAD, OPTIONS]
          CachedMethods: [GET, HEAD]
        CustomErrorResponses:
          - ErrorCode: 403
            ResponseCode: 200
            ResponsePagePath: /index.html
            ErrorCachingMinTTL: 0
          - ErrorCode: 404
            ResponseCode: 200
            ResponsePagePath: /index.html
            ErrorCachingMinTTL: 0

Outputs:
  CognitoUserPoolId:
    Value: !Ref FamilyLedgerUserPool
    Export:
      Name: !Sub "${AWS::StackName}-CognitoUserPoolId"
  CognitoWebAppClientId:
    Value: !Ref FamilyLedgerWebAppClient
    Export:
      Name: !Sub "${AWS::StackName}-CognitoWebClientId"
  ApiGatewayUrl:
    Value: !Sub "https://${FamilyLedgerApi}.execute-api.${AWS::Region}.amazonaws.com/${Environment}"
    Export:
      Name: !Sub "${AWS::StackName}-ApiUrl"
  CloudFrontUrl:
    Value: !Sub "https://${FamilyLedgerCdn}.cloudfront.net"
    Export:
      Name: !Sub "${AWS::StackName}-CloudFrontUrl"
```

---

## Step 4: Deploying with SAM

```powershell
sam deploy --guided
```

Verify deployment outputs and connect Cognito and API Gateway URLs into your application configurations.

