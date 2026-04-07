# CloudFormation Stack — Internal Team Readiness Guide

> **Document Type:** Internal Reference
> **Stack Name:** `nginx-app` (configurable via `AppName` parameter)
> **Purpose:** Fully automated ECS Fargate deployment of an NGINX container using S3, CodeBuild, and ECR — zero manual steps post-deployment.
> **IAM Capability Required:** `CAPABILITY_IAM` only *(no CAPABILITY_NAMED_IAM — all role names are auto-generated)*

---

## Table of Contents

1. [Stack Overview](#1-stack-overview)
2. [Architecture Diagram (Text)](#2-architecture-diagram-text)
3. [Resource Creation Flow](#3-resource-creation-flow)
4. [Component Reference](#4-component-reference)
   - [Networking](#41-networking-resources)
   - [Storage](#42-storage-resources)
   - [IAM](#43-iam-roles)
   - [Container Registry](#44-container-registry)
   - [Automation — Lambda Functions](#45-automation--lambda-functions)
   - [Automation — Custom Resources](#46-automation--custom-resources)
   - [Build Pipeline](#47-build-pipeline)
   - [Load Balancer](#48-load-balancer)
   - [ECS Compute](#49-ecs-compute)
   - [Observability](#410-observability)
5. [Inter-Service Dependency Map](#5-inter-service-dependency-map)
6. [Parameters](#6-parameters)
7. [Outputs](#7-outputs)
8. [IAM Permission Summary](#8-iam-permission-summary)
9. [Failure Points & Troubleshooting](#9-failure-points--troubleshooting)
10. [Known Design Decisions & Fixes](#10-known-design-decisions--fixes)
11. [Re-Deployment Checklist](#11-re-deployment-checklist)

---

## 1. Stack Overview

This stack provisions a **complete, production-ready containerized web application** on AWS using CloudFormation. When the stack reaches `CREATE_COMPLETE`, the NGINX application is already live and accessible via an ALB DNS URL.

### What the stack builds automatically

| Stage | What Happens |
|---|---|
| **Pre-flight** | IAM service-linked roles for ECS and ELB are verified/created; stack waits 30 seconds for IAM propagation before continuing |
| **Bootstrap** | S3 bucket created; Lambda uploads `Dockerfile` + `index.html` as `app.zip` |
| **Build** | CodeBuild pulls `app.zip` from S3, builds a Docker image, pushes it to ECR |
| **Wait** | A Lambda polls CodeBuild every 15 seconds; CloudFormation waits for it to confirm success |
| **Serve** | ECS Fargate pulls the ECR image; ALB routes HTTP traffic to the container |
| **Force Deploy** | A final Lambda triggers a `ForceNewDeployment` on the ECS service to guarantee the latest image is running |

> ⚠️ **ECS Service will not start until the CodeBuild build is confirmed SUCCEEDED.** This is enforced via a `DependsOn: WaitForBuild` on the ECS Service resource.

> ⚠️ **ECS Cluster and ALB will not be created until service-linked IAM roles are confirmed present.** This is enforced via a `DependsOn: EnsureServiceLinkedRoles` on both the `ECSCluster` and `ALB` resources.

---

## 2. Architecture Diagram (Text)

```
Internet
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│                        VPC (10.0.0.0/16)                │
│                                                         │
│   Internet Gateway ──► Public Route Table               │
│                                                         │
│   ┌─────────────┐       ┌─────────────┐                 │
│   │  Subnet AZ1 │       │  Subnet AZ2 │                 │
│   │ 10.0.1.0/24 │       │ 10.0.2.0/24 │                 │
│   └──────┬──────┘       └──────┬──────┘                 │
│          └────────┬────────────┘                        │
│                   ▼                                     │
│         ┌──────────────────┐                            │
│         │  ALB (Port 80)   │◄── ALB Security Group      │
│         └────────┬─────────┘     (0.0.0.0/0:80)         │
│                  │                                      │
│                  ▼                                      │
│         ┌──────────────────┐                            │
│         │  Target Group    │                            │
│         │ (Health: 200-399)│                            │
│         └────────┬─────────┘                            │
│                  │                                      │
│                  ▼                                      │
│         ┌──────────────────┐                            │
│         │  ECS Fargate Task│◄── ECS Security Group      │
│         │  NGINX 1.29      │    (ALB SG → Port 80 only) │
│         │  (ECR Public)    │                            │
│         └──────────────────┘                            │
│                  ▲                                      │
│                  │ pulls image                          │
│         ┌────────┴─────────┐                            │
│         │   ECR Repository  │                           │
│         └──────────────────┘                            │
│                  ▲                                      │
│                  │ docker push                          │
│         ┌────────┴─────────┐                            │
│         │   CodeBuild      │◄── S3 bucket (app.zip)     │
│         └──────────────────┘                            │
└─────────────────────────────────────────────────────────┘

CloudFormation Orchestration Layer
────────────────────────────────────────────────────────────
EnsureServiceLinkedRolesLambda ──► IAM (create ECS + ELB SLRs)
UploadFilesLambda              ──► S3 (app.zip)
WaitForBuildLambda             ──► CodeBuild (start + poll)
ForceDeployLambda              ──► ECS (ForceNewDeployment)
CloudFormation     ──► Waits on Custom Resources before ECS
```

---

## 3. Resource Creation Flow

CloudFormation creates resources in dependency order. Below is the exact sequential flow with logical IDs.

```
PHASE 1 — Foundation (no dependencies)
─────────────────────────────────────────────────────────────
SourceBucket         → S3 bucket for source artifacts (versioning enabled)
VPC                  → Network boundary
InternetGateway      → Internet access gateway
ECRRepository        → Private Docker image registry (scan-on-push enabled)
ECSLogGroup          → CloudWatch log group for ECS tasks
SetupLambdaRole      → IAM role for EnsureServiceLinkedRoles Lambda
UploadLambdaRole     → IAM role for upload Lambda
CodeBuildRole        → IAM role for CodeBuild
ECSTaskExecutionRole → IAM role for ECS task
BuildLambdaRole      → IAM role for wait Lambda
ForceDeployLambdaRole → IAM role for force deploy Lambda

         │
         ▼

PHASE 2 — Network Wiring
─────────────────────────────────────────────────────────────
IGWAttachment              → Attaches IGW to VPC
PublicSubnet1 / Subnet2    → Two AZ subnets inside VPC
ALBSecurityGroup           → SG for ALB
ECSSecurityGroup           → SG for ECS (source: ALB SG)
PublicRouteTable           → Route table for subnets
PublicRoute          (DependsOn: IGWAttachment)
Subnet1RouteTableAssociation
Subnet2RouteTableAssociation

         │
         ▼

PHASE 3 — Lambda Functions
─────────────────────────────────────────────────────────────
EnsureServiceLinkedRolesLambda → Uses SetupLambdaRole
UploadFilesLambda    → Uses UploadLambdaRole, DependsOn: SourceBucket
WaitForBuildLambda   → Uses BuildLambdaRole
ForceDeployLambda    → Uses ForceDeployLambdaRole

         │
         ▼

PHASE 4 — Pre-flight (Custom Resource ensures IAM SLRs exist)
─────────────────────────────────────────────────────────────
EnsureServiceLinkedRoles  → Custom Resource → calls EnsureServiceLinkedRolesLambda
                            → Checks for ECS and ELB service-linked roles
                            → Creates them if missing
                            → Waits 30 seconds for IAM propagation
                            → ECSCluster and ALB DependsOn this resource

         │
         ▼

PHASE 5 — Bootstrap (Custom Resource triggers upload Lambda)
─────────────────────────────────────────────────────────────
UploadFilesToS3      → Custom Resource → calls UploadFilesLambda
                       → Lambda creates app.zip and puts it in S3

         │
         ▼

PHASE 6 — Build Project
─────────────────────────────────────────────────────────────
CodeBuildProject     → DependsOn: UploadFilesToS3
                       → Source: S3 bucket / app.zip
                       → Pushes image to ECR with :latest and :YYYYMMDDHHMMSS tags

         │
         ▼

PHASE 7 — Build Execution (Custom Resource waits for build)
─────────────────────────────────────────────────────────────
WaitForBuild         → Custom Resource → calls WaitForBuildLambda
                       → Lambda starts CodeBuild build
                       → Polls every 15s until SUCCEEDED / FAILED
                       → CloudFormation blocked here until build finishes

         │
         ▼

PHASE 8 — Load Balancer
─────────────────────────────────────────────────────────────
ALB                  → DependsOn: IGWAttachment + EnsureServiceLinkedRoles
ALBTargetGroup       → Depends on VPC
ALBListener          → Connects ALB → TargetGroup

         │
         ▼

PHASE 9 — ECS (only after build confirmed + ALB ready)
─────────────────────────────────────────────────────────────
ECSCluster           → DependsOn: EnsureServiceLinkedRoles
                       → Fargate cluster with FARGATE capacity provider
ECSTaskDefinition    → Task with ECR image reference
ECSService           → DependsOn: ALBListener + WaitForBuild
                       → Pulls image from ECR
                       → Registers in ALBTargetGroup

         │
         ▼

PHASE 10 — Post-Deploy Force Redeploy
─────────────────────────────────────────────────────────────
ForceECSDeployment   → Custom Resource → calls ForceDeployLambda
                       → DependsOn: ECSService + WaitForBuild
                       → Issues ForceNewDeployment to ECS service
                       → Ensures the latest ECR image is pulled every time
                         DeployVersion is incremented

         │
         ▼

STACK STATUS: CREATE_COMPLETE
Application is live at ALB DNS URL
```

---

## 4. Component Reference

### 4.1 Networking Resources

#### `VPC`
- **Type:** `AWS::EC2::VPC`
- **CIDR:** `10.0.0.0/16`
- **Purpose:** Isolated network boundary for all resources. DNS support and hostname resolution enabled — required for ECS tasks to resolve ECR endpoints.

---

#### `InternetGateway` + `IGWAttachment`
- **Type:** `AWS::EC2::InternetGateway` / `AWS::EC2::VPCGatewayAttachment`
- **Purpose:** Provides internet connectivity to the VPC. The `IGWAttachment` links the gateway to the VPC. The ALB and ECS tasks depend on this for inbound/outbound internet access.

---

#### `PublicSubnet1` / `PublicSubnet2`
- **Type:** `AWS::EC2::Subnet`
- **CIDRs:** `10.0.1.0/24` (AZ-0), `10.0.2.0/24` (AZ-1)
- **Purpose:** Two subnets across different Availability Zones. Required by the ALB (which mandates at least 2 AZs). ECS tasks run in these subnets with public IPs assigned automatically (`MapPublicIpOnLaunch: true`).

---

#### `PublicRouteTable` + `PublicRoute`
- **Type:** `AWS::EC2::RouteTable` / `AWS::EC2::Route`
- **Purpose:** Routes all outbound traffic (`0.0.0.0/0`) through the Internet Gateway. Both subnets are associated with this route table via `Subnet1RouteTableAssociation` and `Subnet2RouteTableAssociation`.

---

#### `ALBSecurityGroup`
- **Type:** `AWS::EC2::SecurityGroup`
- **Inbound:** TCP port 80 from `0.0.0.0/0` (entire internet)
- **Purpose:** Controls what traffic reaches the ALB. Only HTTP (port 80) is open from the internet.

---

#### `ECSSecurityGroup`
- **Type:** `AWS::EC2::SecurityGroup`
- **Inbound:** TCP port 80 **only from `ALBSecurityGroup`** (not from internet directly)
- **Outbound:** All traffic allowed (needed for ECR pulls and CloudWatch logs)
- **Purpose:** Locks down ECS tasks so only the ALB can send traffic to them. This is a security best practice — containers are never directly reachable from the internet.

---

### 4.2 Storage Resources

#### `SourceBucket`
- **Type:** `AWS::S3::Bucket`
- **Name:** `{AppName}-source-{AccountId}-{Region}` *(globally unique)*
- **Versioning:** Enabled — every upload of `app.zip` is retained as a distinct version
- **Purpose:** Stores `app.zip` which contains the `Dockerfile` and `index.html`. CodeBuild pulls from this bucket to build the Docker image. The bucket is created first — all other automation depends on it.

---

### 4.3 IAM Roles

> All roles use **auto-generated names** (no `RoleName` property). This means only `CAPABILITY_IAM` is needed — not `CAPABILITY_NAMED_IAM`. This is intentional for compatibility with automated deployment platforms like CloudLabs.

#### `SetupLambdaRole`
- **Used By:** `EnsureServiceLinkedRolesLambda`
- **Permissions:**
  - `iam:CreateServiceLinkedRole` on `*` (required to create ECS and ELB service-linked roles)
  - `iam:GetRole` on `*` (to check whether the roles already exist)
  - `logs:*` on CloudWatch for Lambda logging

#### `UploadLambdaRole`
- **Used By:** `UploadFilesLambda`
- **Permissions:**
  - `s3:PutObject`, `s3:DeleteObject` on the source bucket
  - `logs:*` on CloudWatch for Lambda logging

#### `CodeBuildRole`
- **Used By:** `CodeBuildProject`
- **Permissions:**
  - `ecr:GetAuthorizationToken` (resource: `*` — AWS requirement; token API does not support resource-level scoping)
  - ECR push actions (`BatchCheckLayerAvailability`, `PutImage`, `InitiateLayerUpload`, `UploadLayerPart`, `CompleteLayerUpload`) scoped to the ECR repository ARN
  - `s3:GetObject`, `s3:GetObjectVersion` on the source bucket
  - `logs:*` for build logs

#### `ECSTaskExecutionRole`
- **Used By:** `ECSTaskDefinition`
- **Permissions:** AWS Managed Policy `AmazonECSTaskExecutionRolePolicy`
  - Allows ECS to pull images from ECR and send logs to CloudWatch on behalf of the task

#### `BuildLambdaRole`
- **Used By:** `WaitForBuildLambda`
- **Permissions:**
  - `codebuild:StartBuild`, `codebuild:BatchGetBuilds` scoped to the `CodeBuildProject` ARN
  - `logs:*` for Lambda logging

#### `ForceDeployLambdaRole`
- **Used By:** `ForceDeployLambda`
- **Permissions:**
  - `ecs:UpdateService` scoped to the specific ECS service ARN (`{AppName}-cluster/{AppName}-service`)
  - `ecs:DescribeServices` scoped to both the ECS service ARN and cluster ARN
  - `logs:*` for Lambda logging

---

### 4.4 Container Registry

#### `ECRRepository`
- **Type:** `AWS::ECR::Repository`
- **Name:** `{AppName}-repo`
- **Scan on Push:** Enabled (automatic vulnerability scanning on every new image)
- **Purpose:** Private Docker image registry. CodeBuild pushes the built image here; ECS pulls from here when launching tasks. The image is tagged as both `:latest` and a timestamp tag (e.g., `:20240315120000`).

---

### 4.5 Automation — Lambda Functions

#### `EnsureServiceLinkedRolesLambda`
- **Function Name:** `{AppName}-ensure-slr`
- **Runtime:** Python 3.9
- **Timeout:** 60 seconds
- **Trigger:** CloudFormation Custom Resource (`EnsureServiceLinkedRoles`)
- **What it does:**
  - On `Create`/`Update`: Iterates over the required service-linked roles for `ecs.amazonaws.com` (`AWSServiceRoleForECS`) and `elasticloadbalancing.amazonaws.com` (`AWSServiceRoleForElasticLoadBalancing`). For each, attempts `iam:GetRole` — if the role does not exist, calls `iam:CreateServiceLinkedRole`. If any role was created, waits 30 seconds for IAM propagation before returning.
  - On `Delete`: Immediately returns `SUCCESS` (service-linked roles are not deleted on stack teardown).
- **Why it exists:** Brand-new AWS accounts do not have the ECS or ELB service-linked roles pre-created. Without them, creating an ECS cluster or an ALB fails. This Lambda ensures the roles are present before either resource is provisioned, making the stack deployable in any account regardless of its history.

---

#### `UploadFilesLambda`
- **Function Name:** `{AppName}-upload-files`
- **Runtime:** Python 3.12
- **Timeout:** 60 seconds
- **Trigger:** CloudFormation Custom Resource (`UploadFilesToS3`)
- **What it does:**
  - On `Create`/`Update`: Generates a ZIP file in memory containing a hardcoded `Dockerfile` and `index.html`, then uploads it to the S3 bucket as `app.zip`.
  - On `Delete`: Removes `app.zip` from S3 (cleanup).
- **Note on base image:** The `Dockerfile` uses `public.ecr.aws/nginx/nginx:1.29` (Amazon ECR Public Gallery) rather than Docker Hub. This avoids anonymous pull rate limits (100 pulls per 6 hours per IP) that frequently affect shared CodeBuild NAT IPs.
- **Why it exists:** CloudFormation cannot natively write files to S3. This Lambda bridges that gap — making the stack truly self-contained with zero pre-requisites.

---

#### `WaitForBuildLambda`
- **Function Name:** `{AppName}-wait-for-build`
- **Runtime:** Python 3.12
- **Timeout:** 900 seconds (15 minutes — maximum Lambda allows)
- **Trigger:** CloudFormation Custom Resource (`WaitForBuild`)
- **What it does:**
  - On `Create`/`Update`: Calls `codebuild:StartBuild` to kick off a build, then polls `codebuild:BatchGetBuilds` every 15 seconds.
  - Returns `SUCCESS` to CloudFormation when build status is `SUCCEEDED`.
  - Returns `FAILED` if build status is `FAILED`, `FAULT`, `TIMED_OUT`, or `STOPPED`.
  - Has a self-timeout guard: if Lambda has less than 60 seconds remaining, it sends `FAILED` to CloudFormation before Lambda itself times out.
  - On `Delete`: Immediately returns `SUCCESS` (no cleanup needed).
- **Why it exists:** CloudFormation has no native mechanism to wait for a CodeBuild job to complete. Without this, ECS would start before the Docker image exists in ECR and the service would fail to launch.

---

#### `ForceDeployLambda`
- **Function Name:** `{AppName}-force-deploy`
- **Runtime:** Python 3.12
- **Timeout:** 60 seconds
- **Trigger:** CloudFormation Custom Resource (`ForceECSDeployment`)
- **What it does:**
  - On `Create`/`Update`: Calls `ecs:UpdateService` with `forceNewDeployment: true` on the ECS service. This causes ECS to pull the latest image from ECR and replace the running task, even if the task definition has not changed.
  - On `Delete`: Immediately returns `SUCCESS` (no cleanup needed).
- **Why it exists:** When `DeployVersion` is incremented and the stack is updated, the new Docker image is built and pushed to ECR but the `ECSTaskDefinition` resource does not change (it always references `:latest`). Without a forced redeploy, ECS would continue running the old container. This Lambda guarantees every update cycle ends with the newest image running.

---

### 4.6 Automation — Custom Resources

#### `EnsureServiceLinkedRoles`
- **Type:** `Custom::EnsureServiceLinkedRoles`
- **Backed by:** `EnsureServiceLinkedRolesLambda`
- **Role in stack:** First gate in the deployment sequence. Both `ECSCluster` and `ALB` have `DependsOn: EnsureServiceLinkedRoles`, so neither can be created until IAM service-linked roles are confirmed present. This is a silent pre-flight check that runs once per stack creation or update.

---

#### `UploadFilesToS3`
- **Type:** `Custom::UploadFilesToS3`
- **Backed by:** `UploadFilesLambda`
- **Properties passed:** `BucketName`, `ZipKey` (`app.zip`), `Version` (bound to `DeployVersion` parameter)
- **Role in stack:** Acts as the trigger for file upload. CloudFormation treats this like any other resource — it waits for the Lambda to respond before proceeding to create `CodeBuildProject`. The `Version` property ensures CloudFormation re-invokes the Lambda (and therefore re-uploads `app.zip`) whenever `DeployVersion` changes.

---

#### `WaitForBuild`
- **Type:** `Custom::WaitForBuild`
- **Backed by:** `WaitForBuildLambda`
- **Properties passed:** `ProjectName` (CodeBuild project name), `Version` (bound to `DeployVersion` parameter)
- **DependsOn:** `CodeBuildProject`, `UploadFilesToS3`
- **Role in stack:** This is the primary synchronization point of the entire stack. CloudFormation is blocked here until the CodeBuild build finishes. `ECSService` has `DependsOn: WaitForBuild`, guaranteeing the Docker image is in ECR before ECS tries to pull it.

---

#### `ForceECSDeployment`
- **Type:** `Custom::ForceECSDeployment`
- **Backed by:** `ForceDeployLambda`
- **Properties passed:** `ClusterName`, `ServiceName`, `Version` (bound to `DeployVersion` parameter)
- **DependsOn:** `ECSService`, `WaitForBuild`
- **Role in stack:** Final step of every deployment. Runs after the ECS service is up and the build is confirmed complete. Triggers a forced redeployment so ECS always pulls the latest ECR image. The `Version` property ensures this custom resource re-runs on every stack update where `DeployVersion` is incremented.

---

### 4.7 Build Pipeline

#### `CodeBuildProject`
- **Type:** `AWS::CodeBuild::Project`
- **Name:** `{AppName}-build`
- **Environment:** `LINUX_CONTAINER` / `standard:7.0` / `BUILD_GENERAL1_SMALL`
- **Privileged Mode:** `true` (required for Docker daemon inside CodeBuild)
- **Source:** S3 — `{SourceBucket}/app.zip`
- **Build phases:**

| Phase | Commands |
|---|---|
| `pre_build` | Login to ECR via `get-login-password`; set `VERSION_TAG` to current timestamp (`date +%Y%m%d%H%M%S`) |
| `build` | `docker build` the image, tag as `:latest`; `docker tag` the same image with `:{VERSION_TAG}` |
| `post_build` | `docker push` both `:latest` and `:{VERSION_TAG}` tags to ECR |

- **Environment Variables injected:** `AWS_DEFAULT_REGION`, `AWS_ACCOUNT_ID`, `ECR_REGISTRY`, `ECR_REPO_URI`
- **Logs:** Sent to CloudWatch log group `/codebuild/{AppName}`
- **DependsOn:** `UploadFilesToS3` (source file must exist in S3 before the project is created)

---

### 4.8 Load Balancer

#### `ALB`
- **Type:** `AWS::ElasticLoadBalancingV2::LoadBalancer`
- **Scheme:** `internet-facing`
- **Subnets:** Both public subnets (spans 2 AZs for high availability)
- **DependsOn:** `IGWAttachment` + `EnsureServiceLinkedRoles` — ensures both the internet gateway is attached and the ELB service-linked role exists before the ALB is created

---

#### `ALBTargetGroup`
- **Type:** `AWS::ElasticLoadBalancingV2::TargetGroup`
- **Target Type:** `ip` (required for Fargate — tasks register by IP, not instance ID)
- **Health Check:** `GET /` every 15s; timeout 5s; 2 consecutive passes = healthy; 3 fails = unhealthy
- **Health Check Matcher:** `200-399` *(accepts redirects — not just strict 200)*
- **Port:** 80 / HTTP
- **Deregistration Delay:** 10 seconds (reduced from the AWS default of 300s to speed up task replacement during deployments)

---

#### `ALBListener`
- **Type:** `AWS::ElasticLoadBalancingV2::Listener`
- **Protocol:** HTTP / Port 80
- **Action:** Forward all traffic to `ALBTargetGroup`

---

### 4.9 ECS Compute

#### `ECSCluster`
- **Type:** `AWS::ECS::Cluster`
- **Name:** `{AppName}-cluster`
- **Capacity Provider:** `FARGATE` (serverless — no EC2 instances to manage), set as the default capacity provider strategy with weight `1`
- **DependsOn:** `EnsureServiceLinkedRoles` (the ECS service-linked role must exist before a cluster can be created in a fresh account)
- **Purpose:** Logical grouping for ECS services and tasks.

---

#### `ECSTaskDefinition`
- **Type:** `AWS::ECS::TaskDefinition`
- **Network Mode:** `awsvpc` (each task gets its own ENI and IP)
- **CPU / Memory:** `256` vCPU units / `512` MB
- **Image:** `{AccountId}.dkr.ecr.{Region}.amazonaws.com/{AppName}-repo:latest`
- **Container Port:** 80
- **Log Driver:** `awslogs` → sends logs to `/ecs/{AppName}` CloudWatch log group with stream prefix `ecs`
- **Execution Role:** `ECSTaskExecutionRole` (allows ECR pull + CloudWatch logging)

---

#### `ECSService`
- **Type:** `AWS::ECS::Service`
- **Name:** `{AppName}-service`
- **Launch Type:** `FARGATE`
- **Desired Count:** `1`
- **DependsOn:** `ALBListener` + `WaitForBuild` *(both must complete before ECS starts)*
- **Health Check Grace Period:** 60 seconds (gives the container time to start before ALB marks it unhealthy)
- **Network:** Deploys tasks in both public subnets; assigns public IPs (`AssignPublicIp: ENABLED`); uses `ECSSecurityGroup`
- **Load Balancer Registration:** Registers container port 80 into `ALBTargetGroup`

---

### 4.10 Observability

#### `ECSLogGroup`
- **Type:** `AWS::Logs::LogGroup`
- **Log Group Name:** `/ecs/{AppName}`
- **Retention:** 7 days
- **Purpose:** Captures all stdout/stderr from the NGINX container. Accessible via CloudWatch Logs console or CLI for debugging.

> CodeBuild build logs are written to a separate log group: `/codebuild/{AppName}`, configured directly on the `CodeBuildProject` resource.

---

## 5. Inter-Service Dependency Map

```
SourceBucket ◄────────────────────────────────────────────────────────────────────┐
     │                                                                             │
     ▼                                                                             │
UploadLambdaRole ──► UploadFilesLambda ──► UploadFilesToS3 (Custom Resource)      │
                                                   │ writes app.zip to ──────────► │
                                                   ▼
                                          CodeBuildProject
                                                   │
               BuildLambdaRole ──► WaitForBuildLambda
                                                   │
                                          WaitForBuild (Custom Resource)
                                         [triggers build + polls until done]
                                                   │
                          ECRRepository ◄──── CodeBuild pushes :latest + :timestamp
                                 │
                                 └──────────────────────────────────────┐
                                                                        ▼
SetupLambdaRole ──► EnsureServiceLinkedRolesLambda                ECSTaskDefinition
                           │                                            │
                  EnsureServiceLinkedRoles (Custom Resource)            ▼
                           │                                        ECSCluster (DependsOn SLR)
                           ├──────────────────────────────────────► ECSService
                           │                                            │
VPC ──► Subnets ──► RouteTable ──► IGW                                  │
          │                                                             │
          ▼                                                             │
ALBSecurityGroup ──► ALB (DependsOn IGW + SLR) ──► ALBTargetGroup      │
ECSSecurityGroup ──►─────────────────────────────► ALBListener ────────►│
                                                                        │
                                             ECSLogGroup ◄──────────────┤
                                        ECSTaskExecutionRole ◄──────────┘
                                                        │
                                                        ▼
                              ForceDeployLambdaRole ──► ForceDeployLambda
                                                        │
                                               ForceECSDeployment (Custom Resource)
                                             [DependsOn: ECSService + WaitForBuild]
                                             [Forces ECS to pull latest ECR image]
```

---

## 6. Parameters

| Parameter | Default | Required | Description |
|---|---|---|---|
| `AppName` | *(none — must be provided)* | Yes | Base name prefix applied to every resource. Change this to deploy multiple isolated instances of the stack in the same account. All resource names follow `{AppName}-{resource-type}`. |
| `CheckAcknowledgement` | `TRUE` | No | Accepts `TRUE` or `FALSE`. Acknowledges that `CAPABILITY_IAM` is required for this stack. Defaults to `TRUE`. |
| `DeployVersion` | *(none — must be provided)* | Yes | Must be incremented by 1 every time application content changes (e.g., `index.html` or `Dockerfile`). Changing this value forces the full pipeline to re-run: the Upload Lambda re-uploads `app.zip`, CodeBuild rebuilds the image, ECR receives the new image, and the Force Deploy Lambda triggers ECS to pull it. **Without incrementing this value, CloudFormation will skip all downstream custom resource steps even if source content was changed.** |

> **Naming convention:** All resources follow `{AppName}-{resource-type}[-{qualifier}]`. For example: `nginx-app-alb`, `nginx-app-cluster`, `nginx-app-source-{accountId}-{region}`.

---

## 7. Outputs

| Output Key | Description | Usage |
|---|---|---|
| `ApplicationURL` | `http://{ALB DNS Name}` | Open in browser immediately after `CREATE_COMPLETE` |
| `SourceBucketName` | S3 bucket name | For debugging — confirm `app.zip` was uploaded by the Upload Lambda |
| `ECRRepositoryURI` | Full ECR URI (`{AccountId}.dkr.ecr.{Region}.amazonaws.com/{AppName}-repo`) | Use for manual `docker pull` / `docker push` during testing or debugging |
| `DeployVersionUsed` | The `DeployVersion` parameter value used in the current deployment | Reference to confirm which version is deployed; increment this value to trigger the next full pipeline run |

---

## 8. IAM Permission Summary

| Role | Principal | Key Permissions | Scope |
|---|---|---|---|
| `SetupLambdaRole` | `lambda.amazonaws.com` | `iam:CreateServiceLinkedRole`, `iam:GetRole`, CloudWatch logs | `*` (IAM SLR creation requires broad scope) |
| `UploadLambdaRole` | `lambda.amazonaws.com` | `s3:PutObject`, `s3:DeleteObject`, CloudWatch logs | Source bucket only |
| `CodeBuildRole` | `codebuild.amazonaws.com` | ECR auth token + ECR push actions, S3 read, CloudWatch logs | ECR auth: `*` (AWS requirement); ECR push: repository ARN only; S3: bucket only |
| `ECSTaskExecutionRole` | `ecs-tasks.amazonaws.com` | ECR pull + CloudWatch logs | Managed policy (`AmazonECSTaskExecutionRolePolicy`) |
| `BuildLambdaRole` | `lambda.amazonaws.com` | `codebuild:StartBuild`, `codebuild:BatchGetBuilds`, CloudWatch logs | CodeBuild project ARN only |
| `ForceDeployLambdaRole` | `lambda.amazonaws.com` | `ecs:UpdateService`, `ecs:DescribeServices`, CloudWatch logs | ECS service ARN + ECS cluster ARN only |

> **Note on `ecr:GetAuthorizationToken` with `Resource: *`** — This is an AWS requirement. The ECR auth token API does not support resource-level restrictions. All other ECR actions (push/pull) are scoped to the specific repository ARN.

---

## 9. Failure Points & Troubleshooting

### Stack gets stuck at `EnsureServiceLinkedRoles`

This custom resource runs first in every deployment. If it fails:

1. Check CloudWatch Logs: `/aws/lambda/{AppName}-ensure-slr`
2. Common causes: Lambda execution role not yet propagated (rare IAM race condition immediately after role creation); insufficient permissions on the `SetupLambdaRole` to call `iam:CreateServiceLinkedRole`
3. If the SLR already exists from a prior deployment, the Lambda should return success immediately — check logs to confirm the `get_role` call succeeded

---

### Stack gets stuck at `WaitForBuild`

The Lambda polls CodeBuild for up to ~14 minutes. If it's taking long, check:

1. **CodeBuild Console** → Find project `{AppName}-build` → View build logs
2. Common causes: ECR Public Gallery pull throttled or unavailable, ECR auth failure, S3 source not found, Docker build error in `index.html` or `Dockerfile`
3. If Lambda timed out: Check CloudWatch log group `/aws/lambda/{AppName}-wait-for-build`

---

### ECS Service shows 0 running tasks

1. Check ECS task stopped reason in console (Events tab on the service)
2. Common causes:
   - ECR image not found (build failed silently — recheck `WaitForBuild` logs)
   - ECSSecurityGroup blocking outbound (requires port 443 for ECR pull)
   - Task running out of memory (increase `Memory` in task definition)

---

### ALB health checks failing (service loops)

1. Check container logs in CloudWatch: `/ecs/{AppName}`
2. Common causes:
   - Container returning non-2xx/3xx (currently `Matcher: 200-399` handles most cases)
   - NGINX permission issue (the `chmod 644` in Dockerfile handles this)
   - Container not started yet — grace period is 60s; ALB starts checking after that

---

### `UploadFilesToS3` custom resource fails

1. Check CloudWatch Logs: `/aws/lambda/{AppName}-upload-files`
2. Common causes:
   - IAM role not yet propagated (rare race condition — retry deployment)
   - S3 bucket name conflict (bucket already exists from a previous deployment in the same account/region)

---

### `ForceECSDeployment` custom resource fails

1. Check CloudWatch Logs: `/aws/lambda/{AppName}-force-deploy`
2. Common causes:
   - ECS service not yet stable when Lambda runs (rare timing issue — retry the stack update)
   - `ForceDeployLambdaRole` missing `ecs:UpdateService` permission (should not occur if stack was deployed cleanly)
3. Note: A failure here does not mean the container is broken — it means the forced redeploy was not issued. The ECS service may still be running the previous image. Manually trigger a new deployment from the ECS console if needed.

---

### Application content not updating after a stack update

The most common cause is **not incrementing `DeployVersion`**. CloudFormation detects changes to resources by comparing property values. If `DeployVersion` is unchanged, CloudFormation considers `UploadFilesToS3`, `WaitForBuild`, and `ForceECSDeployment` as already up-to-date and skips them — even if the Lambda source code was edited.

**Fix:** Always increment `DeployVersion` by 1 with every content change before submitting a stack update.

---

## 10. Known Design Decisions & Fixes

### Fix 1 — NGINX 403 on every request

**Root cause:** The Dockerfile used `RUN rm -rf /usr/share/nginx/html/*` which wiped inherited directory permissions. The `COPY`-ed `index.html` ended up owned by `root:root` with mode `600`. The NGINX worker process runs as uid `nginx` and could not read the file.

**Fix applied in template:**
```dockerfile
COPY index.html /usr/share/nginx/html/index.html
RUN chmod 644 /usr/share/nginx/html/index.html
```

---

### Fix 2 — ALB health checks failing on redirects

**Root cause:** Default ALB health check matcher only accepts HTTP `200`. If NGINX returned any `3xx` redirect, the task was marked unhealthy and killed in a loop.

**Fix applied in template:**
```json
"Matcher": { "HttpCode": "200-399" }
```

---

### Fix 3 — CAPABILITY_NAMED_IAM removed

**Root cause:** All IAM roles originally had explicit `RoleName` properties which triggered the `CAPABILITY_NAMED_IAM` requirement. CloudLabs deployment does not pass this capability.

**Fix applied in template:** All `RoleName:` lines removed from all six IAM roles: `SetupLambdaRole`, `UploadLambdaRole`, `CodeBuildRole`, `ECSTaskExecutionRole`, `BuildLambdaRole`, and `ForceDeployLambdaRole`. CloudFormation now auto-generates role names. All cross-references use `Fn::GetAtt` / `Ref` on logical IDs and are unaffected.

---

### Fix 4 — ECS/ELB service-linked role race condition in new accounts

**Root cause:** Brand-new AWS accounts do not have the `AWSServiceRoleForECS` or `AWSServiceRoleForElasticLoadBalancing` roles pre-created. Attempting to create an ECS cluster or ALB in such accounts fails with an IAM role not found error.

**Fix applied in template:** Added `EnsureServiceLinkedRolesLambda` and the `EnsureServiceLinkedRoles` custom resource. Both `ECSCluster` and `ALB` now have `DependsOn: EnsureServiceLinkedRoles`. The Lambda also enforces a 30-second wait after creation to allow IAM propagation across regions.

---

### Fix 5 — ECS not pulling updated image after rebuild

**Root cause:** The `ECSTaskDefinition` always references `:latest`. When a new image is pushed to ECR, the task definition does not change, so CloudFormation does not trigger a new ECS deployment.

**Fix applied in template:** Added `ForceDeployLambda` and the `ForceECSDeployment` custom resource (triggered last, after `ECSService` and `WaitForBuild`). This issues `ecs:UpdateService` with `forceNewDeployment: true` on every stack update where `DeployVersion` is incremented, ensuring ECS always pulls the freshly built image.

---

## 11. Re-Deployment Checklist

Use this checklist before deploying the stack in a new environment or after making template changes.

- [ ] `AppName` parameter is set to a unique value if deploying multiple stacks in the same account
- [ ] `DeployVersion` has been incremented by 1 if any application content (`index.html`, `Dockerfile`) has changed
- [ ] Target AWS region supports ECS Fargate, CodeBuild standard:7.0, and ECR
- [ ] No existing S3 bucket named `{AppName}-source-{AccountId}-{Region}` (S3 names are global)
- [ ] No existing ECR repository named `{AppName}-repo` in the target region
- [ ] CloudFormation is being deployed with at least `CAPABILITY_IAM`
- [ ] Lambda execution roles have no existing name conflict (auto-generated, so this is safe)
- [ ] Account has not hit ECS Fargate vCPU limits in the target region
- [ ] Account ECR image scan results reviewed after first build (scan-on-push is enabled)
- [ ] ALB DNS name from `ApplicationURL` output tested in browser after `CREATE_COMPLETE`
- [ ] CloudWatch log group `/ecs/{AppName}` is receiving logs (confirms container started correctly)
- [ ] `ForceECSDeployment` custom resource shows `CREATE_COMPLETE` in CloudFormation Events (confirms ECS is running the latest image)

---

*Last Updated: April 2026 | Maintained by: Charan kumar asam*