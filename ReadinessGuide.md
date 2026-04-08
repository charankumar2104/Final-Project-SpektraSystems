# CloudFormation Stack — Internal Team Readiness Guide

> **Document Type:** Internal Reference
> **Stack Name:** `nginx-app` (set by the `AppName` parameter)
> **Purpose:** A fully self-contained ECS Fargate deployment of an NGINX web app — no manual steps needed after stack creation.
> **IAM Capability Required:** `CAPABILITY_IAM` only — role names are auto-generated so `CAPABILITY_NAMED_IAM` is not needed.
> **Last Updated:** April 2026 | Maintained by: Charan Kumar Asam

---

## Table of Contents

1. [What This Stack Does](#1-what-this-stack-does)
2. [Architecture at a Glance](#2-architecture-at-a-glance)
3. [How Resources Are Created (Step by Step)](#3-how-resources-are-created-step-by-step)
4. [Every Resource Explained](#4-every-resource-explained)
   - [Networking](#41-networking)
   - [Storage](#42-storage)
   - [IAM Roles](#43-iam-roles)
   - [Container Registry](#44-container-registry-ecr)
   - [Lambda Functions](#45-lambda-functions)
   - [CloudFormation Custom Resources](#46-cloudformation-custom-resources)
   - [Build Pipeline](#47-build-pipeline-codebuild)
   - [Load Balancer](#48-load-balancer-alb)
   - [ECS Compute](#49-ecs-compute)
   - [Logging](#410-logging-cloudwatch)
5. [How Everything Depends on Each Other](#5-how-everything-depends-on-each-other)
6. [Parameters](#6-parameters)
7. [Outputs](#7-outputs)
8. [IAM Permissions Summary](#8-iam-permissions-summary)
9. [What Can Go Wrong and How to Fix It](#9-what-can-go-wrong-and-how-to-fix-it)
10. [Design Decisions Worth Knowing](#10-design-decisions-worth-knowing)
11. [Checklist Before Deploying](#11-checklist-before-deploying)

---

## 1. What This Stack Does

When you deploy this stack, CloudFormation builds a complete containerized web application from scratch — automatically and in the right order. By the time the stack reaches `CREATE_COMPLETE`, the NGINX web page is already live and reachable at a public URL.

Here's the short version of what happens under the hood:

| Stage | What Actually Happens |
|---|---|
| **Prep** | Lambda generates a `Dockerfile` and `index.html`, zips them, and uploads the ZIP to S3 |
| **Build** | CodeBuild downloads the ZIP, builds a Docker image, and pushes it to ECR |
| **Wait** | A second Lambda polls CodeBuild every 15 seconds; CloudFormation doesn't move forward until the build succeeds |
| **Deploy** | ECS Fargate pulls the image from ECR; the ALB starts routing traffic to the container |
| **Confirm** | A third Lambda forces a fresh ECS deployment to make sure the latest image is actually running |

One important thing to understand: **the ECS service will never start before the Docker image exists in ECR.** This is enforced by having the ECS service declare a dependency on the `WaitForBuild` custom resource. If the build fails, CloudFormation rolls everything back.

---

## 2. Architecture at a Glance

```
Visitor's browser
       │
       ▼  HTTP port 80
┌──────────────────────────────────────────────────────────┐
│                    VPC (10.0.0.0/16)                     │
│                                                          │
│  Internet Gateway ──► Public Route Table                 │
│                                                          │
│  ┌──────────────┐       ┌──────────────┐                 │
│  │ Subnet AZ1   │       │ Subnet AZ2   │                 │
│  │ 10.0.1.0/24  │       │ 10.0.2.0/24  │                 │
│  └──────┬───────┘       └──────┬───────┘                 │
│         └──────────┬───────────┘                         │
│                    ▼                                     │
│         ┌─────────────────────┐                          │
│         │  ALB  (port 80)     │ ◄── ALB Security Group   │
│         │  internet-facing    │     (0.0.0.0/0 → 80)     │
│         └──────────┬──────────┘                          │
│                    │                                     │
│                    ▼                                     │
│         ┌─────────────────────┐                          │
│         │  Target Group       │                          │
│         │  health: GET /      │                          │
│         │  matcher: 200-399   │                          │
│         └──────────┬──────────┘                          │
│                    │                                     │
│                    ▼                                     │
│         ┌─────────────────────┐                          │
│         │  ECS Fargate Task   │ ◄── ECS Security Group   │
│         │  NGINX 1.29 (ECR)   │     (ALB SG → port 80)   │
│         └──────────┬──────────┘                          │
│                    │ logs                                │
│                    ▼                                     │
│         CloudWatch /ecs/nginx-app                        │
│                    ▲                                     │
│                    │ pulls image                         │
│         ┌──────────┴──────────┐                          │
│         │   ECR Repository    │                          │
│         │   nginx-app-repo    │                          │
│         └──────────┬──────────┘                          │
│                    ▲                                     │
│                    │ docker push                         │
│         ┌──────────┴──────────┐                          │
│         │  CodeBuild          │ ◄── S3 (app.zip)         │
│         │  nginx-app-build    │                          │
│         └─────────────────────┘                          │
└──────────────────────────────────────────────────────────┘

CloudFormation automation layer:
  EnsureServiceLinkedRolesLambda  ──► IAM (creates ECS + ELB SLRs)
  UploadFilesLambda               ──► S3 (generates + uploads app.zip)
  WaitForBuildLambda              ──► CodeBuild (starts build + polls)
  ForceDeployLambda               ──► ECS (forces fresh deployment)
```

---

## 3. How Resources Are Created (Step by Step)

CloudFormation doesn't create everything at once — it follows dependencies. Here's the exact order, organized by phase:

```
PHASE 1 — Foundation (created first, no dependencies)
──────────────────────────────────────────────────────────────
SourceBucket          S3 bucket to hold app.zip
VPC                   Private network
InternetGateway       Gateway connecting VPC to the internet
ECRRepository         Private Docker image registry
ECSLogGroup           CloudWatch log group for container logs
SetupLambdaRole       IAM role for the Ensure SLR Lambda
UploadLambdaRole      IAM role for the Upload Files Lambda
CodeBuildRole         IAM role for CodeBuild
ECSTaskExecutionRole  IAM role for ECS tasks
BuildLambdaRole       IAM role for the Wait For Build Lambda
ForceDeployLambdaRole IAM role for the Force Deploy Lambda

          │
          ▼

PHASE 2 — Network Wiring
──────────────────────────────────────────────────────────────
IGWAttachment              Attaches the Internet Gateway to the VPC
PublicSubnet1              Subnet in first Availability Zone
PublicSubnet2              Subnet in second Availability Zone
ALBSecurityGroup           Firewall for the ALB
ECSSecurityGroup           Firewall for ECS tasks (only ALB can reach in)
PublicRouteTable           Route table for internet traffic
PublicRoute                Default route: all traffic → Internet Gateway
Subnet1RouteTableAssociation
Subnet2RouteTableAssociation

          │
          ▼

PHASE 3 — Lambda Functions Created
──────────────────────────────────────────────────────────────
EnsureServiceLinkedRolesLambda   Uses SetupLambdaRole
UploadFilesLambda                Uses UploadLambdaRole, DependsOn: SourceBucket
WaitForBuildLambda               Uses BuildLambdaRole
ForceDeployLambda                Uses ForceDeployLambdaRole

          │
          ▼

PHASE 4 — Service Linked Roles Confirmed
──────────────────────────────────────────────────────────────
EnsureServiceLinkedRoles (Custom Resource)
  → Calls EnsureServiceLinkedRolesLambda
  → Lambda checks if ECS and ELB service-linked IAM roles exist
  → Creates them if missing
  → Waits 30 seconds for IAM propagation
  → CloudFormation waits here before creating the ALB and ECS cluster

          │
          ▼

PHASE 5 — Source Files Uploaded
──────────────────────────────────────────────────────────────
UploadFilesToS3 (Custom Resource)
  → Calls UploadFilesLambda
  → Lambda generates Dockerfile + index.html in memory
  → Packages them as app.zip
  → Uploads app.zip to S3

          │
          ▼

PHASE 6 — Build Project Created
──────────────────────────────────────────────────────────────
CodeBuildProject
  → DependsOn: UploadFilesToS3
  → Configured to read from S3 (app.zip) and push to ECR

          │
          ▼

PHASE 7 — Image Built and Confirmed in ECR
──────────────────────────────────────────────────────────────
WaitForBuild (Custom Resource)
  → DependsOn: CodeBuildProject, UploadFilesToS3
  → Calls WaitForBuildLambda
  → Lambda starts the CodeBuild build
  → Polls every 15 seconds until SUCCEEDED or FAILED
  → CloudFormation is completely blocked here until the Lambda reports back
  → If build fails → CloudFormation rolls back the stack

          │
          ▼

PHASE 8 — Serving Layer Created
──────────────────────────────────────────────────────────────
ALB                   DependsOn: IGWAttachment, EnsureServiceLinkedRoles
ALBTargetGroup        Targets: ECS task IPs; health check: GET / → 200-399
ALBListener           Forwards port 80 traffic to ALBTargetGroup
ECSCluster            DependsOn: EnsureServiceLinkedRoles; Fargate capacity provider
ECSTaskDefinition     Image: ECR :latest; CPU 256; Memory 512 MB

          │
          ▼

PHASE 9 — ECS Service Started
──────────────────────────────────────────────────────────────
ECSService
  → DependsOn: ALBListener, WaitForBuild
  → Pulls :latest image from ECR
  → Registers container IP in ALBTargetGroup
  → ALB health checks begin (60-second grace period)
  → Application goes live

          │
          ▼

PHASE 10 — Fresh Deployment Forced
──────────────────────────────────────────────────────────────
ForceECSDeployment (Custom Resource)
  → DependsOn: ECSService, WaitForBuild
  → Calls ForceDeployLambda
  → Lambda calls ecs:UpdateService with forceNewDeployment=True
  → Guarantees ECS is running the latest image, not a cached one

          │
          ▼

STACK STATUS: CREATE_COMPLETE
Application is live at the ALB URL (see Outputs → ApplicationURL)
```

---

## 4. Every Resource Explained

### 4.1 Networking

**VPC** (`AWS::EC2::VPC`)
CIDR: `10.0.0.0/16`. The isolated network boundary for the entire stack. DNS support and DNS hostnames are both enabled — ECS tasks need DNS resolution to find ECR endpoints when pulling images.

**Internet Gateway + IGW Attachment** (`AWS::EC2::InternetGateway` / `AWS::EC2::VPCGatewayAttachment`)
The gateway that connects the VPC to the public internet. Nothing can reach the internet — inbound or outbound — without it. The attachment resource links the gateway to the VPC.

**Public Subnets** (`AWS::EC2::Subnet`)
Two subnets, one in each Availability Zone: `10.0.1.0/24` (AZ 0) and `10.0.2.0/24` (AZ 1). Both have `MapPublicIpOnLaunch: true`, so ECS tasks get a public IP automatically — they need one to reach ECR for image pulls without a NAT gateway. The ALB requires at least two subnets in different AZs, so two subnets is the minimum.

**Route Table + Route** (`AWS::EC2::RouteTable` / `AWS::EC2::Route`)
Sends all outbound traffic (`0.0.0.0/0`) through the Internet Gateway. Both subnets are associated with this route table.

**ALB Security Group** (`AWS::EC2::SecurityGroup`)
Inbound: TCP port 80 from anywhere (`0.0.0.0/0`). This is what lets the public internet reach the load balancer.

**ECS Security Group** (`AWS::EC2::SecurityGroup`)
Inbound: TCP port 80 only from the ALB Security Group — not from the internet directly. Outbound: all traffic allowed (ECS tasks need outbound access to pull images from ECR and send logs to CloudWatch). This setup means containers are never directly reachable from the internet.

---

### 4.2 Storage

**Source Bucket** (`AWS::S3::Bucket`)
Name format: `{AppName}-source-{AccountId}-{Region}` — globally unique by design. Versioning is enabled so every upload of `app.zip` is preserved. This bucket is the starting point for the whole build pipeline — everything downstream depends on it existing first.

---

### 4.3 IAM Roles

All roles use auto-generated names (no `RoleName` property in the template). This is intentional — it means the stack only needs `CAPABILITY_IAM` to deploy. If names were hardcoded, `CAPABILITY_NAMED_IAM` would be required, which some deployment platforms restrict.

**SetupLambdaRole**
Used by: `EnsureServiceLinkedRolesLambda`
Permissions: Create ECS and ELB service-linked roles (`iam:CreateServiceLinkedRole`); read existing IAM roles (`iam:GetRole`); write Lambda logs to CloudWatch.

**UploadLambdaRole**
Used by: `UploadFilesLambda`
Permissions: Put and delete objects in the source S3 bucket only (`s3:PutObject`, `s3:DeleteObject`); write Lambda logs to CloudWatch.

**CodeBuildRole**
Used by: `CodeBuildProject`
Permissions: Get an ECR auth token (`ecr:GetAuthorizationToken` — this has to be `Resource: *` because AWS doesn't support resource-level scoping for this API call); push image layers to the ECR repository; read `app.zip` from the S3 bucket; write build logs to CloudWatch.

**ECSTaskExecutionRole**
Used by: `ECSTaskDefinition`
Permissions: AWS managed policy `AmazonECSTaskExecutionRolePolicy` — lets ECS pull images from ECR and send container output to CloudWatch on behalf of the task.

**BuildLambdaRole**
Used by: `WaitForBuildLambda`
Permissions: Start a CodeBuild build and check its status (`codebuild:StartBuild`, `codebuild:BatchGetBuilds`) — scoped to the specific CodeBuild project ARN, not all projects; write Lambda logs to CloudWatch.

**ForceDeployLambdaRole**
Used by: `ForceDeployLambda`
Permissions: Update the ECS service (`ecs:UpdateService`) and describe the service and cluster (`ecs:DescribeServices`) — scoped to the specific ECS service and cluster ARNs; write Lambda logs to CloudWatch.

---

### 4.4 Container Registry (ECR)

**ECR Repository** (`AWS::ECR::Repository`)
Name: `{AppName}-repo`. Scan on push is enabled, so every image is automatically scanned for known vulnerabilities when it's pushed. CodeBuild pushes two tags on every build:
- `:latest` — always points to the most recent build
- `:YYYYMMDDHHMMSS` — a permanent timestamp snapshot for rollback purposes

ECS always pulls `:latest` when starting a new task.

---

### 4.5 Lambda Functions

**EnsureServiceLinkedRolesLambda** (`{AppName}-ensure-slr`)
Runtime: Python 3.9 | Timeout: 60 seconds

Checks whether the `AWSServiceRoleForECS` and `AWSServiceRoleForElasticLoadBalancing` IAM service-linked roles exist in the account. If either is missing, it creates them. If both already exist, it skips creation. After any creation, it waits 30 seconds for IAM to propagate changes. This solves a real problem: brand-new AWS accounts don't have these roles yet, and trying to create an ECS cluster or ALB without them fails immediately.

**UploadFilesLambda** (`{AppName}-upload-files`)
Runtime: Python 3.12 | Timeout: 60 seconds

On `Create` or `Update`: Builds the `Dockerfile` and `index.html` from hardcoded strings in the function code, packages them into a ZIP file in memory (never touches disk), and uploads the ZIP to S3 as `app.zip`. On `Delete`: Removes `app.zip` from S3. This function is why the stack is fully self-contained — there are no pre-existing files required anywhere.

Note: The Dockerfile pulls the NGINX base image from the ECR Public Gallery (`public.ecr.aws/nginx/nginx:1.29`) instead of Docker Hub. This avoids Docker Hub's anonymous pull rate limits, which can cause builds to fail on shared CodeBuild infrastructure.

**WaitForBuildLambda** (`{AppName}-wait-for-build`)
Runtime: Python 3.12 | Timeout: 900 seconds (15 minutes — the maximum Lambda allows)

On `Create` or `Update`: Calls `codebuild:StartBuild` to kick off a build, then enters a polling loop — every 15 seconds it checks the build status. When the build finishes, it reports `SUCCESS` or `FAILED` back to CloudFormation.

Two safety details worth knowing:
1. If the Lambda has less than 60 seconds of runtime left, it proactively sends `FAILED` to CloudFormation rather than letting Lambda time out uncleanly (which would leave CloudFormation waiting forever).
2. On `Delete`, it immediately returns `SUCCESS` — there's nothing to clean up.

**ForceDeployLambda** (`{AppName}-force-deploy`)
Runtime: Python 3.12 | Timeout: 60 seconds

Calls `ecs:UpdateService` with `forceNewDeployment=True` on the `{AppName}-service` in the `{AppName}-cluster`. This forces ECS to pull the latest image from ECR and restart the container, even if the task definition itself hasn't changed. Without this, ECS might continue running a previously cached container after a new build completes. On `Delete`, returns `SUCCESS` immediately.

---

### 4.6 CloudFormation Custom Resources

Custom Resources are how CloudFormation calls Lambda functions during a deployment and waits for a result before continuing.

**EnsureServiceLinkedRoles** (`Custom::EnsureServiceLinkedRoles`)
Backed by `EnsureServiceLinkedRolesLambda`. This runs early in the deployment — before the ALB and ECS cluster are created — to make sure the required IAM roles exist.

**UploadFilesToS3** (`Custom::UploadFilesToS3`)
Backed by `UploadFilesLambda`. Properties passed: `BucketName`, `ZipKey` (`app.zip`), and `Version` (the `DeployVersion` parameter value). Including `Version` is what makes CloudFormation re-trigger this resource on updates — if `Version` didn't change, CloudFormation would assume nothing changed and skip the Lambda call entirely.

**WaitForBuild** (`Custom::WaitForBuild`)
Backed by `WaitForBuildLambda`. DependsOn: `CodeBuildProject` and `UploadFilesToS3`. Properties passed: `ProjectName` and `Version`. This is the synchronization point of the entire stack — everything downstream (ECS service, Force Deploy) waits for this to succeed.

**ForceECSDeployment** (`Custom::ForceECSDeployment`)
Backed by `ForceDeployLambda`. DependsOn: `ECSService` and `WaitForBuild`. Properties passed: `ClusterName`, `ServiceName`, and `Version`. The `Version` property here is also what triggers this resource to re-run when `DeployVersion` is incremented.

---

### 4.7 Build Pipeline (CodeBuild)

**CodeBuildProject** (`AWS::CodeBuild::Project`)
Name: `{AppName}-build`
Environment: `LINUX_CONTAINER` / `standard:7.0` / `BUILD_GENERAL1_SMALL` / Privileged Mode ON (required for running Docker inside CodeBuild)
Source: S3 — reads from `{SourceBucket}/app.zip`

Build phases:

| Phase | What Runs |
|---|---|
| `pre_build` | Logs into ECR with `aws ecr get-login-password`; sets `VERSION_TAG` to the current date/time |
| `build` | Runs `docker build` on the Dockerfile; tags the image as both `:latest` and `:{VERSION_TAG}` |
| `post_build` | Runs `docker push` for both tags to ECR |

Environment variables injected: `AWS_DEFAULT_REGION`, `AWS_ACCOUNT_ID`, `ECR_REGISTRY`, `ECR_REPO_URI`

Build logs: CloudWatch log group `/codebuild/{AppName}`

---

### 4.8 Load Balancer (ALB)

**ALB** (`AWS::ElasticLoadBalancingV2::LoadBalancer`)
Name: `{AppName}-alb`. Internet-facing, type `application`. Spans both public subnets for high availability. DependsOn `IGWAttachment` and `EnsureServiceLinkedRoles`.

**ALB Target Group** (`AWS::ElasticLoadBalancingV2::TargetGroup`)
Name: `{AppName}-tg`. Target type: `ip` — required for Fargate because tasks register by IP address, not by EC2 instance ID. Health check settings: `GET /` every 15 seconds, 5-second timeout, 2 consecutive passes to be healthy, 3 consecutive failures to be unhealthy. Matcher: `200-399` (accepts redirects). Deregistration delay: 10 seconds (reduced from the default 300 seconds to make deployments faster).

**ALB Listener** (`AWS::ElasticLoadBalancingV2::Listener`)
Listens on port 80 / HTTP. Default action: forward all requests to the target group.

---

### 4.9 ECS Compute

**ECS Cluster** (`AWS::ECS::Cluster`)
Name: `{AppName}-cluster`. DependsOn `EnsureServiceLinkedRoles`. Capacity provider: `FARGATE` — serverless, no EC2 instances to manage.

**ECS Task Definition** (`AWS::ECS::TaskDefinition`)
Family: `{AppName}-task`. Network mode: `awsvpc` (each task gets its own network interface and private IP). CPU: `256` units. Memory: `512` MB. Image: `{AccountId}.dkr.ecr.{Region}.amazonaws.com/{AppName}-repo:latest`. Container port: `80`. Log driver: `awslogs` → sends output to `/ecs/{AppName}` in CloudWatch. Execution role: `ECSTaskExecutionRole`.

**ECS Service** (`AWS::ECS::Service`)
Name: `{AppName}-service`. DependsOn: `ALBListener` and `WaitForBuild` (both must be complete before ECS can start). Launch type: `FARGATE`. Desired count: `1`. Health check grace period: 60 seconds — gives the container time to fully start before ALB begins checking it. Network: deploys in both public subnets with public IPs assigned; uses `ECSSecurityGroup`. Load balancer: registers container port 80 into the target group.

---

### 4.10 Logging (CloudWatch)

**ECS Log Group** (`AWS::Logs::LogGroup`)
Name: `/ecs/{AppName}`. Retention: 7 days. All stdout/stderr from the NGINX container is sent here automatically by the `awslogs` log driver configured in the task definition. Each container instance writes to its own log stream, prefixed with `ecs/`.

CodeBuild logs go to a separate group: `/codebuild/{AppName}` (created automatically by CodeBuild, not explicitly in the template).

---

## 5. How Everything Depends on Each Other

Reading this map tells you why things have to happen in the order they do:

```
SourceBucket
    │
    ├──► UploadFilesLambda (DependsOn SourceBucket)
    │         │
    │         ▼
    │    UploadFilesToS3 (Custom Resource → uploads app.zip)
    │         │
    │         ▼
    │    CodeBuildProject (DependsOn UploadFilesToS3)
    │         │
    │         ▼
    │    WaitForBuild (Custom Resource → starts build, polls until done)
    │         │
    │         ├──► ECSService (DependsOn ALBListener + WaitForBuild)
    │         │         │
    │         │         ▼
    │         │    ForceECSDeployment (Custom Resource → forces redeploy)
    │         │
    │         └──► ECRRepository ◄─── CodeBuild pushes image here

VPC ──► Subnets
           │
           ├──► ALBSecurityGroup
           ├──► ECSSecurityGroup
           └──► IGWAttachment ──► PublicRoute

EnsureServiceLinkedRoles (Custom Resource)
    │   (runs first — creates ECS + ELB IAM service-linked roles)
    │
    ├──► ALB (DependsOn IGWAttachment + EnsureServiceLinkedRoles)
    │         │
    │         ▼
    │    ALBTargetGroup ──► ALBListener ──► ECSService
    │
    └──► ECSCluster (DependsOn EnsureServiceLinkedRoles)
              │
              └──► ECSTaskDefinition ──► ECSService
```

---

## 6. Parameters

| Parameter | Default | What It Does |
|---|---|---|
| `AppName` | *(required)* | The name prefix applied to every resource the stack creates. For example, if `AppName` is `nginx-app`, your cluster will be `nginx-app-cluster`, your S3 bucket will be `nginx-app-source-{accountId}-{region}`, and so on. Don't change this on a running stack. |
| `CheckAcknowledgement` | `TRUE` | Confirmation that the stack needs IAM permissions. Leave it as `TRUE`. |
| `DeployVersion` | *(required)* | A version number you control. **Every time you change application content — the `index.html`, the Dockerfile, or anything else — increment this by 1.** Changing this value is what tells CloudFormation to re-run the Upload Lambda, rebuild the Docker image, push it to ECR, and force a fresh ECS deployment. Without incrementing it, CloudFormation sees no change and skips all those steps — your update will never reach the running container. |

---

## 7. Outputs

| Output Key | What It Contains | How to Use It |
|---|---|---|
| `ApplicationURL` | `http://{ALB DNS Name}` | Open this in a browser once the stack shows `CREATE_COMPLETE`. This is your live application. |
| `SourceBucketName` | The full S3 bucket name | Useful for debugging — you can check here to confirm `app.zip` was uploaded successfully. |
| `ECRRepositoryURI` | Full ECR repository URI | Use this for manual `docker pull` or `docker push` commands during testing or debugging. |
| `DeployVersionUsed` | The `DeployVersion` value from this deployment | Helpful reference to know which version is currently running, especially when rolling back. |

---

## 8. IAM Permissions Summary

| Role | Used By | Key Permissions | Scope |
|---|---|---|---|
| `SetupLambdaRole` | EnsureServiceLinkedRolesLambda | `iam:CreateServiceLinkedRole`, `iam:GetRole` | All resources (`*`) — required because service-linked roles don't support resource-level ARN scoping |
| `UploadLambdaRole` | UploadFilesLambda | `s3:PutObject`, `s3:DeleteObject` | Source S3 bucket only |
| `CodeBuildRole` | CodeBuildProject | ECR auth token + push actions; `s3:GetObject` | ECR auth: `*` (AWS limitation); ECR push: repository ARN only; S3: bucket only |
| `ECSTaskExecutionRole` | ECS tasks | Pull from ECR; write logs to CloudWatch | AWS managed policy |
| `BuildLambdaRole` | WaitForBuildLambda | `codebuild:StartBuild`, `codebuild:BatchGetBuilds` | Specific CodeBuild project ARN only |
| `ForceDeployLambdaRole` | ForceDeployLambda | `ecs:UpdateService`, `ecs:DescribeServices` | Specific ECS service and cluster ARNs only |

**Why `ecr:GetAuthorizationToken` uses `Resource: *`:** This is an AWS constraint. The ECR token API doesn't support resource-level restrictions — you can't scope it to a single repository. All actual image push actions are scoped to the specific repository ARN.

---

## 9. What Can Go Wrong and How to Fix It

### Stack gets stuck at `WaitForBuild` for a long time

This Lambda polls for up to about 14 minutes. If it's taking unusually long:

1. Go to **CodeBuild → Projects → `{AppName}-build`** and open the build logs.
2. Common causes: Docker Hub rate limit (shouldn't happen since we use ECR Public Gallery, but check), ECR authentication failure, or `app.zip` not found in S3.
3. If the Lambda itself timed out, check its logs at **CloudWatch → Log Groups → `/aws/lambda/{AppName}-wait-for-build`**.

---

### ECS service shows 0 running tasks

1. Go to **ECS → Clusters → `{AppName}-cluster` → Services → `{AppName}-service` → Tasks tab**.
2. Click a stopped task to see the stopped reason.
3. Common causes:
   - ECR image not found — the build may have failed but `WaitForBuild` missed it somehow. Check CodeBuild logs.
   - ECS task can't reach ECR — make sure the ECS security group allows outbound traffic (it does by default) and the subnets have a route to the internet gateway.
   - Out of memory — 512 MB is the minimum needed; increasing it won't hurt.

---

### ALB health checks failing (tasks keep getting replaced)

1. Check container logs: **CloudWatch → Log Groups → `/ecs/{AppName}`**.
2. Common causes:
   - Container is returning a non-200/3xx response. The matcher is set to `200-399`, which should handle most cases.
   - NGINX file permission issue. The Dockerfile applies `chmod 644` to `index.html` — if this step was skipped or failed, NGINX will return 403.
   - Container hasn't fully started yet. The 60-second grace period should cover this, but if startup is slower than expected, you may need to increase it in the ECS service definition.

---

### `UploadFilesToS3` custom resource fails

1. Check **CloudWatch → Log Groups → `/aws/lambda/{AppName}-upload-files`**.
2. Common causes:
   - S3 bucket already exists from a previous deployment with the same `AppName` and account/region combination (S3 bucket names are global and must be unique).
   - IAM role not propagated yet — very rare, but try redeploying if this happens on a fresh account right after the role is created.

---

### `EnsureServiceLinkedRoles` custom resource fails

1. Check **CloudWatch → Log Groups → `/aws/lambda/{AppName}-ensure-slr`**.
2. Common cause: The Lambda's IAM role (`SetupLambdaRole`) doesn't have permission to call `iam:CreateServiceLinkedRole`. Verify the role was created correctly in IAM.

---

### `ForceECSDeployment` custom resource fails

1. Check **CloudWatch → Log Groups → `/aws/lambda/{AppName}-force-deploy`**.
2. Common cause: ECS service doesn't exist yet (unlikely given the `DependsOn`, but possible if the service creation failed). Also check that the `ForceDeployLambdaRole` has `ecs:UpdateService` permission on the correct service ARN.

---

## 10. Design Decisions Worth Knowing

### Why ECR Public Gallery instead of Docker Hub

The Dockerfile pulls the NGINX base image from `public.ecr.aws/nginx/nginx:1.29` rather than Docker Hub. Docker Hub limits anonymous pulls to 100 per 6 hours per IP address. CodeBuild runs on shared AWS infrastructure with shared NAT IPs, so it's easy to hit those limits and have builds fail mid-pipeline. The ECR Public Gallery has no such restrictions for AWS services.

---

### Why NGINX needs `chmod 644`

The Dockerfile removes all files from `/usr/share/nginx/html/` first (`RUN rm -rf /usr/share/nginx/html/*`), then copies in the custom `index.html`. When files are copied via `COPY` in Docker, they're owned by `root:root` with mode `644` by default — but the `rm -rf` step can strip inherited directory permissions. Without the explicit `chmod 644`, the NGINX worker process (which runs as the `nginx` user, not root) can't read the file and returns HTTP 403 on every request. The fix is the explicit permission step.

---

### Why the ALB health check matcher is `200-399`

The default ALB health check only accepts HTTP `200`. If NGINX returns any redirect (`301`, `302`, `307`), the ALB marks the task as unhealthy and kills it — triggering an endless replacement loop. Setting the matcher to `200-399` accepts all successful and redirect responses as healthy.

---

### Why IAM role names are auto-generated

All six IAM roles in this template have no `RoleName` property. CloudFormation auto-generates names for them. This is intentional: hardcoded role names require `CAPABILITY_NAMED_IAM` when deploying, which some automated lab and CI platforms don't pass. Auto-generated names only require `CAPABILITY_IAM`, which is much more widely supported. All internal cross-references between resources use `!GetAtt` and `!Ref` on logical resource IDs, so they work fine regardless of what name CloudFormation assigns.

---

### Why `DeployVersion` exists

CloudFormation only re-triggers a Custom Resource Lambda when at least one of its declared properties changes. If you update the Lambda code (for example, changing the `index.html` content inside `UploadFilesLambda`) but don't change any property of the `UploadFilesToS3` Custom Resource, CloudFormation won't call the Lambda again — it assumes nothing changed. The `Version` property, which maps to the `DeployVersion` parameter, is the deliberately-included "force trigger." Incrementing it by 1 on every content update makes CloudFormation see a property change, which triggers the Lambda call, which re-uploads the ZIP, which causes CodeBuild to rebuild, which causes ECS to redeploy.

---

## 11. Checklist Before Deploying

Use this before deploying into any new account or region.

- [ ] `AppName` is set — and if running multiple stacks in the same account, each one has a unique value
- [ ] `DeployVersion` is set to a number (start at `1` for a new deployment)
- [ ] `CheckAcknowledgement` is `TRUE`
- [ ] The target AWS region supports ECS Fargate, CodeBuild `standard:7.0`, and ECR
- [ ] No existing S3 bucket named `{AppName}-source-{AccountId}-{Region}` — S3 names must be globally unique
- [ ] No existing ECR repository named `{AppName}-repo` in the target region
- [ ] The deployment is being run with at least `CAPABILITY_IAM`
- [ ] The account hasn't hit ECS Fargate CPU limits in the target region
- [ ] After `CREATE_COMPLETE`, the `ApplicationURL` output opens and shows the NGINX page in a browser
- [ ] CloudWatch log group `/ecs/{AppName}` is receiving logs, confirming the container started correctly
- [ ] ECR scan results reviewed after the first build (scan-on-push is enabled and results appear in the ECR console)