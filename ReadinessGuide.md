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
| **Bootstrap** | S3 bucket created; Lambda uploads `Dockerfile` + `index.html` as `app.zip` |
| **Build** | CodeBuild pulls `app.zip` from S3, builds a Docker image, pushes it to ECR |
| **Wait** | A second Lambda polls CodeBuild every 15 seconds; CloudFormation waits for it |
| **Serve** | ECS Fargate pulls the ECR image; ALB routes HTTP traffic to the container |

> ⚠️ **ECS Service will not start until the CodeBuild build is confirmed SUCCEEDED.** This is enforced via a `DependsOn: WaitForBuild` on the ECS Service resource.

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
│         │  NGINX:1.29      │    (ALB SG → Port 80 only) │
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
UploadFilesLambda  ──►  S3 (app.zip)
WaitForBuildLambda ──►  CodeBuild (start + poll)
CloudFormation     ──►  Waits on Custom Resources before ECS
```

---

## 3. Resource Creation Flow

CloudFormation creates resources in dependency order. Below is the exact sequential flow with logical IDs.

```
PHASE 1 — Foundation (no dependencies)
─────────────────────────────────────────────────────────────
SourceBucket         → S3 bucket for source artifacts
VPC                  → Network boundary
InternetGateway      → Internet access gateway
ECRRepository        → Private Docker image registry
ECSLogGroup          → CloudWatch log group for ECS tasks
UploadLambdaRole     → IAM role for upload Lambda
CodeBuildRole        → IAM role for CodeBuild
ECSTaskExecutionRole → IAM role for ECS task
BuildLambdaRole      → IAM role for wait Lambda

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
UploadFilesLambda    → Uses UploadLambdaRole, DependsOn: SourceBucket
WaitForBuildLambda   → Uses BuildLambdaRole

         │
         ▼

PHASE 4 — Bootstrap (Custom Resource triggers Lambda)
─────────────────────────────────────────────────────────────
UploadFilesToS3      → Custom Resource → calls UploadFilesLambda
                       → Lambda creates app.zip and puts it in S3

         │
         ▼

PHASE 5 — Build Project
─────────────────────────────────────────────────────────────
CodeBuildProject     → DependsOn: UploadFilesToS3
                       → Source: S3 bucket / app.zip
                       → Pushes image to ECR

         │
         ▼

PHASE 6 — Build Execution (Custom Resource waits for build)
─────────────────────────────────────────────────────────────
WaitForBuild         → Custom Resource → calls WaitForBuildLambda
                       → Lambda starts CodeBuild build
                       → Polls every 15s until SUCCEEDED / FAILED
                       → CloudFormation blocked here until build finishes

         │
         ▼

PHASE 7 — Load Balancer
─────────────────────────────────────────────────────────────
ALB                  → DependsOn: IGWAttachment
ALBTargetGroup       → Depends on VPC
ALBListener          → Connects ALB → TargetGroup

         │
         ▼

PHASE 8 — ECS (only after build confirmed + ALB ready)
─────────────────────────────────────────────────────────────
ECSCluster           → Fargate cluster
ECSTaskDefinition    → Task with ECR image reference
ECSService           → DependsOn: ALBListener + WaitForBuild
                       → Pulls image from ECR
                       → Registers in ALBTargetGroup

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
- **Purpose:** Stores `app.zip` which contains the `Dockerfile` and `index.html`. CodeBuild pulls from this bucket to build the Docker image. The bucket is created first — all other automation depends on it.

---

### 4.3 IAM Roles

> All roles use **auto-generated names** (no `RoleName` property). This means only `CAPABILITY_IAM` is needed — not `CAPABILITY_NAMED_IAM`. This is intentional for compatibility with automated deployment platforms like CloudLabs.

#### `UploadLambdaRole`
- **Used By:** `UploadFilesLambda`
- **Permissions:**
  - `s3:PutObject`, `s3:DeleteObject` on the source bucket
  - `logs:*` on CloudWatch for Lambda logging

#### `CodeBuildRole`
- **Used By:** `CodeBuildProject`
- **Permissions:**
  - `ecr:GetAuthorizationToken` + push-related actions (resource: `*`) — token cannot be scoped to a single repo
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

---

### 4.4 Container Registry

#### `ECRRepository`
- **Type:** `AWS::ECR::Repository`
- **Name:** `{AppName}-repo`
- **Scan on Push:** Enabled (automatic vulnerability scanning)
- **Purpose:** Private Docker image registry. CodeBuild pushes the built image here; ECS pulls from here when launching tasks. The image is tagged as both `:latest` and a timestamp tag (e.g., `:20240315120000`).

---

### 4.5 Automation — Lambda Functions

#### `UploadFilesLambda`
- **Runtime:** Python 3.12
- **Timeout:** 60 seconds
- **Trigger:** CloudFormation Custom Resource (`UploadFilesToS3`)
- **What it does:**
  - On `Create`/`Update`: Generates a ZIP file in memory containing a hardcoded `Dockerfile` and `index.html`, then uploads it to the S3 bucket as `app.zip`.
  - On `Delete`: Removes `app.zip` from S3 (cleanup).
- **Why it exists:** CloudFormation cannot natively write files to S3. This Lambda bridges that gap — making the stack truly self-contained with zero pre-requisites.

---

#### `WaitForBuildLambda`
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

### 4.6 Automation — Custom Resources

#### `UploadFilesToS3`
- **Type:** `Custom::UploadFilesToS3`
- **Backed by:** `UploadFilesLambda`
- **Properties passed:** `BucketName`, `ZipKey` (`app.zip`)
- **Role in stack:** Acts as the trigger for file upload. CloudFormation treats this like any other resource — it waits for the Lambda to respond before proceeding to create `CodeBuildProject`.

---

#### `WaitForBuild`
- **Type:** `Custom::WaitForBuild`
- **Backed by:** `WaitForBuildLambda`
- **Properties passed:** `ProjectName` (CodeBuild project name)
- **DependsOn:** `CodeBuildProject`, `UploadFilesToS3`
- **Role in stack:** This is the synchronization point of the entire stack. CloudFormation is blocked here until the CodeBuild build finishes. `ECSService` has `DependsOn: WaitForBuild`, guaranteeing the Docker image is in ECR before ECS tries to pull it.

---

### 4.7 Build Pipeline

#### `CodeBuildProject`
- **Type:** `AWS::CodeBuild::Project`
- **Environment:** `LINUX_CONTAINER` / `standard:7.0` / `BUILD_GENERAL1_SMALL`
- **Privileged Mode:** `true` (required for Docker daemon inside CodeBuild)
- **Source:** S3 — `{SourceBucket}/app.zip`
- **Build phases:**

| Phase | Commands |
|---|---|
| `pre_build` | Login to ECR via `get-login-password`; set `VERSION_TAG` to current timestamp |
| `build` | `docker build` the image; tag as `:latest` and `:{VERSION_TAG}` |
| `post_build` | `docker push` both tags to ECR |

- **Environment Variables injected:** `AWS_DEFAULT_REGION`, `AWS_ACCOUNT_ID`, `ECR_REPO_URI`
- **Logs:** Sent to CloudWatch log group `/codebuild/{AppName}`

---

### 4.8 Load Balancer

#### `ALB`
- **Type:** `AWS::ElasticLoadBalancingV2::LoadBalancer`
- **Scheme:** `internet-facing`
- **Subnets:** Both public subnets (spans 2 AZs for high availability)
- **DependsOn:** `IGWAttachment` — ensures the internet gateway is attached before ALB is created

---

#### `ALBTargetGroup`
- **Type:** `AWS::ElasticLoadBalancingV2::TargetGroup`
- **Target Type:** `ip` (required for Fargate — tasks register by IP, not instance ID)
- **Health Check:** `GET /` every 15s; 2 consecutive passes = healthy; 3 fails = unhealthy
- **Health Check Matcher:** `200-399` *(accepts redirects — not just strict 200)*
- **Port:** 80 / HTTP

---

#### `ALBListener`
- **Type:** `AWS::ElasticLoadBalancingV2::Listener`
- **Protocol:** HTTP / Port 80
- **Action:** Forward all traffic to `ALBTargetGroup`

---

### 4.9 ECS Compute

#### `ECSCluster`
- **Type:** `AWS::ECS::Cluster`
- **Capacity Provider:** `FARGATE` (serverless — no EC2 instances to manage)
- **Purpose:** Logical grouping for ECS services and tasks.

---

#### `ECSTaskDefinition`
- **Type:** `AWS::ECS::TaskDefinition`
- **Network Mode:** `awsvpc` (each task gets its own ENI and IP)
- **CPU / Memory:** `256` vCPU units / `512` MB
- **Image:** `{AccountId}.dkr.ecr.{Region}.amazonaws.com/{AppName}-repo:latest`
- **Container Port:** 80
- **Log Driver:** `awslogs` → sends logs to `/ecs/{AppName}` CloudWatch log group
- **Execution Role:** `ECSTaskExecutionRole` (allows ECR pull + CloudWatch logging)

---

#### `ECSService`
- **Type:** `AWS::ECS::Service`
- **Launch Type:** `FARGATE`
- **Desired Count:** `1`
- **DependsOn:** `ALBListener` + `WaitForBuild` *(both must complete before ECS starts)*
- **Health Check Grace Period:** 60 seconds (gives the container time to start before ALB marks it unhealthy)
- **Network:** Deploys tasks in both public subnets; assigns public IPs; uses `ECSSecurityGroup`
- **Load Balancer Registration:** Registers container port 80 into `ALBTargetGroup`

---

### 4.10 Observability

#### `ECSLogGroup`
- **Type:** `AWS::Logs::LogGroup`
- **Log Group Name:** `/ecs/{AppName}`
- **Retention:** 7 days
- **Purpose:** Captures all stdout/stderr from the NGINX container. Accessible via CloudWatch Logs console or CLI for debugging.

---

## 5. Inter-Service Dependency Map

```
SourceBucket ◄────────────────────────────────────────────────────────────────┐
     │                                                                        │
     ▼                                                                        │
UploadLambdaRole ──► UploadFilesLambda ──► UploadFilesToS3 (Custom Resource)  │
                                                   │ writes app.zip to ─────► │
                                                   ▼
                                          CodeBuildProject
                                                   │
               BuildLambdaRole ──► WaitForBuildLambda
                                                   │
                                          WaitForBuild (Custom Resource)
                                         [triggers build + polls until done]
                                                   │
                          ECRRepository ◄──── CodeBuild pushes image
                                 │
                                 └────────────────────────────────┐
                                                                  ▼
VPC ──► Subnets ──► RouteTable ──► IGW                   ECSTaskDefinition
          │                                                       │
          ▼                                                       ▼
ALBSecurityGroup ──► ALB ──► ALBTargetGroup ──► ALBListener    ECSCluster
ECSSecurityGroup ──►─────────────────────────────────────────► ECSService
                                                                  │
                                              ECSLogGroup ◄───────┤
                                         ECSTaskExecutionRole ◄───┘
```

---

## 6. Parameters

| Parameter | Default | Description |
|---|---|---|
| `AppName` | `nginx-app` | Base name prefix applied to every resource. Change this to deploy multiple isolated instances of the stack in the same account. |

> **Naming convention:** All resources follow `{AppName}-{resource-type}[-{qualifier}]`. For example, `nginx-app-alb`, `nginx-app-ecs-cluster`, `nginx-app-source-{accountId}-{region}`.

---

## 7. Outputs

| Output Key | Description | Usage |
|---|---|---|
| `ApplicationURL` | `http://{ALB DNS Name}` | Open in browser immediately after `CREATE_COMPLETE` |
| `SourceBucketName` | S3 bucket name | For debugging — check if `app.zip` was uploaded |
| `ECRRepositoryURI` | Full ECR URI | Use for manual docker pull / push during testing |
| `CodeBuildProjectName` | Project name | Navigate to CodeBuild console to view build logs |
| `ECSClusterName` | Cluster name | Use with ECS console or CLI to inspect running tasks |
| `ECSServiceName` | Service name | Use to check service health, desired/running task count |

---

## 8. IAM Permission Summary

| Role | Principal | Key Permissions | Scope |
|---|---|---|---|
| `UploadLambdaRole` | `lambda.amazonaws.com` | `s3:PutObject`, `s3:DeleteObject` | Source bucket only |
| `CodeBuildRole` | `codebuild.amazonaws.com` | ECR push + S3 read + CloudWatch logs | ECR: `*` (token restriction); S3: bucket only |
| `ECSTaskExecutionRole` | `ecs-tasks.amazonaws.com` | ECR pull + CloudWatch logs | Managed policy |
| `BuildLambdaRole` | `lambda.amazonaws.com` | `codebuild:StartBuild`, `BatchGetBuilds` | CodeBuild project ARN only |

> **Note on `ecr:GetAuthorizationToken` with `Resource: *`** — This is an AWS requirement. The ECR auth token API does not support resource-level restrictions. All other ECR actions (push/pull) are scoped to the specific repository.

---

## 9. Failure Points & Troubleshooting

### Stack gets stuck at `WaitForBuild`

The Lambda polls CodeBuild for up to ~14 minutes. If it's taking long, check:

1. **CodeBuild Console** → Find project `{AppName}-build` → View build logs
2. Common causes: Docker Hub rate limit during base image pull, ECR auth failure, S3 source not found
3. If Lambda timed out: Check CloudWatch log group `/aws/lambda/{AppName}-wait-for-build`

---

### ECS Service shows 0 running tasks

1. Check ECS task stopped reason in console (Events tab on the service)
2. Common causes:
   - ECR image not found (build failed silently — recheck `WaitForBuild`)
   - ECSSecurityGroup blocking outbound (requires port 443 for ECR pull via NAT or VPC endpoint)
   - Task running out of memory (increase `Memory` in task definition)

---

### ALB health checks failing (service loops)

1. Check container logs in CloudWatch: `/ecs/{AppName}`
2. Common causes:
   - Container returning non-2xx/3xx (currently `Matcher: 200-399` handles this)
   - NGINX permission issue (the `chmod 644` in Dockerfile handles this)
   - Container not started yet — grace period is 60s; ALB starts checking after that

---

### `UploadFilesToS3` custom resource fails

1. Check CloudWatch Logs: `/aws/lambda/{AppName}-upload-files`
2. Common causes:
   - IAM role not yet propagated (rare race condition — retry deployment)
   - S3 bucket name conflict (bucket already exists from a previous deployment)

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
```yaml
Matcher:
  HttpCode: 200-399
```

---

### Fix 3 — CAPABILITY_NAMED_IAM removed

**Root cause:** All four IAM roles had explicit `RoleName` properties which triggered the `CAPABILITY_NAMED_IAM` requirement. CloudLabs deployment does not pass this capability.

**Fix applied in template:** All `RoleName:` lines removed from `UploadLambdaRole`, `CodeBuildRole`, `ECSTaskExecutionRole`, and `BuildLambdaRole`. CloudFormation now auto-generates role names. All cross-references use `!GetAtt` / `!Ref` logical IDs and are unaffected.

---

## 11. Re-Deployment Checklist

Use this checklist before deploying the stack in a new environment or after making template changes.

- [ ] `AppName` parameter is set to a unique value if deploying multiple stacks in the same account
- [ ] Target AWS region supports ECS Fargate, CodeBuild standard:7.0, and ECR
- [ ] No existing S3 bucket named `{AppName}-source-{AccountId}-{Region}` (S3 names are global)
- [ ] No existing ECR repository named `{AppName}-repo` in the target region
- [ ] CloudFormation is being deployed with at least `CAPABILITY_IAM`
- [ ] Lambda execution role has no existing name conflict (auto-generated, so this is safe)
- [ ] Account has not hit ECS Fargate vCPU limits in the target region
- [ ] Account ECR image scan results reviewed after first build (scan-on-push is enabled)
- [ ] ALB DNS name from `ApplicationURL` output tested in browser after `CREATE_COMPLETE`
- [ ] CloudWatch log group `/ecs/{AppName}` is receiving logs (confirms container started correctly)

---

*Last Updated: April 2026 | Maintained by: Charan kumar asam*