# Lab Overview: NGINX on AWS ECS Fargate — End-to-End Hands-On Lab

**Lab Title:** Deploying and Managing a Containerized Application on AWS ECS Fargate
**Difficulty:** Beginner to Intermediate
**Total Tasks:** 6
**Total Estimated Duration:** 2 to 3 Hours
**Lab Type:** Guided Hands-On

---

## About This Lab

This lab is designed to give you practical, real-world experience working with core AWS services — specifically around containerized application deployment and management. You will not just read about these technologies — you will use them directly through a browser-based AWS environment that has been pre-configured specifically for you.

Each participant in this lab receives their own isolated AWS account. Everything you do stays within your own assigned space and does not affect any other participant. You are free to explore, make changes, and learn without the risk of breaking anything shared.

By the end of this lab, you will have logged in to a real AWS account, verified a live running web application, made changes to it and watched those changes automatically propagate through a full deployment pipeline, scaled the application up and down, monitored it using CloudWatch Logs, and explored the build pipeline that packaged and published the Docker image.

---

## How the Lab Environment Works

When you receive your Lab Details Page, your environment has already been fully provisioned and is ready for you. The following things were automatically set up before you arrived:

- A dedicated AWS account scoped to your lab session
- A complete networking stack — VPC, subnets, route tables, internet gateway, and security groups
- An Application Load Balancer (ALB) routing HTTP traffic to your application
- An Amazon ECR private repository holding the Docker image for your application
- An ECS Fargate cluster running an NGINX container serving a live web page
- A CodeBuild project that built and pushed the Docker image to ECR
- Lambda functions that automated the entire build and deployment pipeline
- CloudWatch Log Groups capturing all application logs in real time

You do not need to set up any of the above. Your job is to explore, understand, use, and modify what has been provisioned.

---

## Lab Environment Details

At the start of the lab, your Lab Details Page will contain the following information. Keep this page open throughout the entire lab as you will refer to it in every task.

| Item | Description |
|---|---|
| AWS Console Login URL | The web address to log in to your AWS account |
| Username | Your personal IAM username for this lab |
| Password | Your password for the AWS Console |
| AWS Region | The region where all your lab resources are deployed |

---

## Key Technologies You Will Work With

Below is a plain-language definition of every technology and AWS service you will encounter during this lab. Read through these before starting the tasks so you have a baseline understanding of what each service does.

---

### AWS Management Console

The AWS Management Console is the web-based graphical interface you use to manage all AWS services. You access it by going to a login URL provided on your Lab Details Page. From the console, you can view, create, modify, and interact with AWS resources using a point-and-click interface — no coding required for this lab.

---

### AWS CloudFormation

AWS CloudFormation is the service that deployed everything in this lab automatically. You describe your entire infrastructure in a template file, and CloudFormation reads it and creates all the resources in the correct order. Think of it as a recipe — the recipe lists every ingredient (AWS service) and the exact steps to assemble them.

In this lab, a CloudFormation **stack** — a collection of all the resources deployed together — was created for you before you logged in. You will explore this stack, view its resources and outputs, and make controlled changes to it.

---

### Amazon S3 (Simple Storage Service)

Amazon S3 is AWS's object storage service — a place to store files of any kind. In this lab, an S3 bucket holds the source files (`Dockerfile` and `index.html`) used to build your application's Docker image. Every time the application content is updated, a Lambda function uploads a fresh copy of these files to S3, which then triggers a new build.

---

### AWS Lambda

AWS Lambda is a service that runs small pieces of code automatically in response to events — without you needing to manage any server. In this lab, Lambda functions are used behind the scenes to automate the entire pipeline: uploading files to S3, starting CodeBuild, waiting for the build to finish, and telling ECS to switch to the new container image.

---

### AWS CodeBuild

AWS CodeBuild is a fully managed build service. It takes source code (or in this case, a `Dockerfile`) and compiles, packages, or builds it into a deployable artifact. In this lab, CodeBuild reads the `Dockerfile` from S3, runs `docker build` to create a Docker image, and then pushes that image to ECR. You will explore past build logs and even manually trigger a new build.

---

### Amazon ECR (Elastic Container Registry)

Amazon ECR is a private Docker image registry hosted on AWS. Think of it like a private version of Docker Hub. After CodeBuild builds the Docker image, it pushes it to ECR. When ECS needs to start a new container, it pulls the image from ECR. In this lab, you will explore the ECR repository and see the image tags that were created by the build pipeline.

---

### Amazon ECS (Elastic Container Service)

Amazon ECS is the AWS service that runs and manages Docker containers. It handles starting containers, keeping them running, replacing them if they fail, and connecting them to load balancers. In this lab, ECS runs your NGINX container on AWS Fargate (see below).

Key ECS concepts you will work with:

| Term | What It Means |
|---|---|
| **Cluster** | A logical grouping that holds all your ECS services and tasks |
| **Task Definition** | The blueprint for your container — what image to use, how much CPU and memory, what ports to open |
| **Task** | A single running instance of a container based on the task definition |
| **Service** | The ECS component that keeps a specific number of tasks running at all times and connects them to the load balancer |
| **Desired Count** | How many copies (tasks) of your container ECS should keep running simultaneously |

---

### AWS Fargate

AWS Fargate is the compute engine that runs your ECS containers without you needing to manage any virtual machines. With Fargate, you say "run my container with 256 CPU units and 512 MB of memory" and AWS handles all the underlying infrastructure. You only pay for the exact CPU and memory used while the container is running.

---

### Application Load Balancer (ALB)

An Application Load Balancer is the entry point for all incoming traffic to your application. It accepts HTTP requests from the internet and forwards them to your ECS containers. If you run multiple containers (tasks), the ALB automatically distributes traffic across all of them. The ALB also performs **health checks** — it periodically sends a test request to each container to confirm it is alive and healthy before sending real user traffic to it.

---

### Amazon CloudWatch

Amazon CloudWatch is the monitoring and observability service in AWS. In this lab, you will use two features of CloudWatch:

- **CloudWatch Logs** — Every HTTP request processed by your NGINX container is written as a log entry and sent to a CloudWatch Log Group in real time. You can read these logs directly in the console.
- **CloudWatch Logs Insights** — A built-in query engine that lets you run structured queries against your log data to filter, count, and analyze requests.

---

### Docker and Containers

A **Docker container** is a lightweight, self-contained package that includes everything needed to run an application — the code, the runtime, the libraries, and the configuration. Containers are consistent: the same container image runs identically on a developer's laptop, a test server, and in a cloud data center.

A **Docker image** is the template from which containers are created. It is built from a `Dockerfile` — a text file that describes exactly how to assemble the image step by step.

In this lab, your NGINX web server runs inside a container. The container image was built by CodeBuild from a `Dockerfile` and stored in ECR. ECS pulls that image and runs it as a container.

---

### NGINX

NGINX (pronounced "engine-x") is a popular, lightweight, and high-performance web server. In this lab, NGINX is used to serve a single HTML web page. It runs inside your Docker container and listens on port 80 for incoming HTTP requests.

---

## Lab Architecture — How Everything Connects

The diagram below shows how all the components in this lab connect to each other. Understanding this diagram will help you see the bigger picture as you work through each task.

```
 ┌─────────────────────────────────────────────────────────────────┐
 │                    AWS Account (Your Lab)                        │
 │                                                                  │
 │  ┌─────────────────────────────────────────────────────────┐    │
 │  │                   CloudFormation Stack                   │    │
 │  │                                                          │    │
 │  │   Lambda ──► S3 (app.zip) ──► CodeBuild ──► ECR         │    │
 │  │                                   │          │           │    │
 │  │                          (Docker  │   Image  │           │    │
 │  │                            Build) │   Push   │           │    │
 │  │                                   └──────────┘           │    │
 │  │                                                │          │    │
 │  │                                         Image Pull       │    │
 │  │                                                │          │    │
 │  │   Internet ──► ALB ──► Target Group ──► ECS Fargate      │    │
 │  │   (Port 80)            (Health Check)   (NGINX Container) │    │
 │  │                                                │          │    │
 │  │                                         CloudWatch Logs  │    │
 │  └─────────────────────────────────────────────────────────┘    │
 └─────────────────────────────────────────────────────────────────┘
```

**Deployment flow** (happens automatically when the stack is deployed or updated):

```
1. Lambda generates Dockerfile + index.html → uploads as app.zip to S3
2. CodeBuild downloads app.zip from S3 → builds Docker image → pushes to ECR
3. ECS Fargate pulls the Docker image from ECR → starts NGINX container
4. ALB routes incoming HTTP traffic to the running container
5. CloudWatch Logs captures every request the NGINX container processes
```

---

## Important Rules for This Lab

- Do not delete the CloudFormation stack or any of its core resources (VPC, ECS cluster, ALB, ECR repository)
- Every task that asks you to make a change also tells you how to revert it — always revert before moving to the next task unless instructed otherwise
- Do not attempt to access other participants' accounts or resources
- If you encounter an unexpected error at any point, re-read the task step carefully before retrying

---

## Task Overview

Below is a summary of all 6 tasks in this lab. Read through the full list before starting so you understand the journey ahead. Each task builds on the knowledge from the previous one.

---

### Task 1 — Accessing Your AWS Lab Account

**Estimated Time:** 10–15 Minutes
**Difficulty:** Beginner
**Type:** Account Access

**What you will do:**
You will begin by finding your unique AWS login credentials on the CloudLabs Environment Details page. Using those credentials, you will log in to the AWS Management Console — the web-based control panel for your entire lab environment. You will then verify that you are in the correct AWS region where your lab resources are deployed.

**Why this task matters:**
Every task in this lab takes place inside the AWS console. Before you can do anything else, you need to successfully log in and confirm you are looking at the right account and region. This task also introduces you to the IAM (Identity and Access Management) login flow, which is how most organizations grant access to AWS accounts without sharing root credentials.

**What you will learn:**
- How to find lab credentials on the CloudLabs portal
- The difference between a root AWS login and an IAM user login
- How AWS regions work and why selecting the correct one matters
- How to confirm your identity is correct inside the AWS console

---

### Task 2 — Verify Your Deployed Application

**Estimated Time:** 15–20 Minutes
**Difficulty:** Beginner
**Type:** Exploration and Verification

**What you will do:**
You will navigate to the AWS CloudFormation service and explore the pre-deployed stack that contains all your lab resources. You will find the live application URL in the stack's Outputs tab, open it in a browser, and confirm the NGINX web application is running and healthy. You will then explore the full list of resources that CloudFormation created, and click through to the ECS Service to verify that your container is actively running.

**Why this task matters:**
Before making any changes, it is essential to first confirm that the baseline environment is healthy. In real-world cloud operations, the first thing any engineer does when inheriting a system is verify its current state. This task also introduces CloudFormation — the service that automated the entire setup of your lab — and helps you understand the relationship between a stack, its resources, and its outputs.

**What you will learn:**
- How to navigate to CloudFormation and read a deployed stack
- What CloudFormation Outputs are and how to use them
- How to find a live application URL and verify it is serving traffic
- How to use CloudFormation resource links to jump directly to any deployed service
- How to read the ECS Service overview to confirm container health

---

### Task 3 — Modify Your Application's Web Page and Redeploy

**Estimated Time:** 25–35 Minutes
**Difficulty:** Beginner–Intermediate
**Type:** Hands-On Change and Revert

**What you will do:**
You will open the CloudFormation template in the built-in Designer, locate the HTML content that generates your web page, and change the heading text from "Running on AWS ECS" to something of your own choosing. You will then submit a stack update and watch the Events tab as the change automatically propagates through the entire pipeline — Lambda re-uploads the files to S3, CodeBuild rebuilds the Docker image, ECR stores the new image, and ECS replaces the running container. Once you confirm the new text appears on the live page.

**Why this task matters:**
This task demonstrates the full power of Infrastructure as Code. A single edit in a template triggers an entire automated deployment pipeline without any manual intervention. This is exactly how modern cloud teams deploy application updates — by changing a file, not by clicking through consoles. Understanding this end-to-end flow is fundamental to working with containerized applications on AWS.

**What you will learn:**
- How to open and edit a CloudFormation template in the Designer
- How to submit a stack update and monitor its progress via the Events tab
- How a single template change triggers the Lambda → S3 → CodeBuild → ECR → ECS pipeline
- How ECS replaces containers with zero downtime during a deployment
- Why the Custom Resource `Version` property is required to force re-deployment

---

### Task 4 — Scale Your Application to Handle More Traffic

**Estimated Time:** 20–25 Minutes
**Difficulty:** Beginner–Intermediate
**Type:** Hands-On Change and Revert

**What you will do:**
You will edit the CloudFormation template to increase the ECS Service's `DesiredCount` from 1 to 2, submit a stack update, and then verify in three places that two containers are now running: the ECS Service overview, the ECS Tasks tab, and the ALB Target Group's Targets tab. You will observe how the Application Load Balancer automatically registers and health-checks both running containers.

**Why this task matters:**
Running a single container is a single point of failure. In production environments, applications always run multiple copies (replicas) spread across different Availability Zones so that if one fails, traffic automatically shifts to the others. This task teaches you the most fundamental scaling operation in ECS and shows you exactly how the load balancer and ECS service work together to maintain availability.

**What you will learn:**
- What `DesiredCount` controls in an ECS Service
- How to scale an ECS service up and down through a CloudFormation update
- How ECS places tasks across multiple Availability Zones automatically
- How the ALB Target Group registers new tasks and performs health checks on them
- The difference between Desired tasks, Running tasks, and Pending tasks in ECS

---

### Task 5 — Monitor Your Application Using CloudWatch Logs

**Estimated Time:** 20–25 Minutes
**Difficulty:** Intermediate
**Type:** Hands-On Monitoring

**What you will do:**
You will first generate traffic to your application by visiting the live URL multiple times and also deliberately accessing paths that do not exist (such as `/test`, `/about`, and `/admin`) to create error log entries. You will then navigate to CloudWatch Logs, find the log group for your ECS container, and read the raw access log entries that NGINX wrote for each of your requests.

**Why this task matters:**
In real production environments, logs are the first place engineers look when something goes wrong. Being able to find the right log group, read raw log output, and run queries to filter and count log entries is a core operational skill for anyone working with cloud applications. This task also introduces the difference between raw log streams and the query-based analysis that Logs Insights enables.

**What you will learn:**
- How to navigate CloudWatch to find a specific log group and log stream
- How to read NGINX access log format — what each field means
- How to identify ALB health check requests versus real user requests in the logs

---

<!-- ### Task 6 — Explore the Build Pipeline — CodeBuild and ECR

**Estimated Time:** 20–25 Minutes
**Difficulty:** Intermediate
**Type:** Exploration and Manual Trigger

**What you will do:**
You will navigate to AWS CodeBuild and open the build project that was used to create your Docker image. You will read the project configuration to understand where it gets its source files from (S3) and what compute environment it runs in. You will then open the most recent build and read through the full build log — tracing each phase from ECR authentication through Docker image construction to the final image push. You will then navigate to Amazon ECR to see the resulting Docker image and its version tags. To complete the task, you will manually trigger a brand new CodeBuild build and watch it run live from start to finish.

**Why this task matters:**
Understanding how a Docker image gets built and published is fundamental to working with any containerized workload. In real organizations, this pipeline runs automatically every time code changes — but being able to read build logs, understand what each phase does, and manually trigger a build when needed are critical diagnostic and operational skills. This task closes the loop on the entire deployment pipeline by showing you every step from source file to running container.

**What you will learn:**
- How to read a CodeBuild project configuration — source, environment, and build phases
- Why Privileged mode is required for Docker builds inside CodeBuild
- How to read and interpret a CodeBuild build log phase by phase
- What ECR image tags are and why both `:latest` and timestamp tags exist
- What an image digest (SHA256) is and why it matters for rollbacks
- How to manually trigger a CodeBuild build and monitor it in real time

--- -->

## Task Execution Order

The tasks must be completed in the order listed. Each task builds on knowledge and actions from the previous one.

<!-- ```
Task 1 ──► Task 2 ──► Task 3 ──► Task 4 ──► Task 5 ──► Task 6
  │           │           │           │           │           │
Login to   Verify     Edit HTML   Scale ECS   Monitor    Explore
AWS        the app    and watch   up to 2     with       CodeBuild
console    is live    pipeline    tasks       CloudWatch  and ECR
                      re-run      then back   Logs        pipeline
``` -->

---

## Quick Reference — AWS Services Used in This Lab

| AWS Service | What It Does in This Lab |
|---|---|
| **CloudFormation** | Deployed all resources automatically; you use it to update the stack |
| **S3** | Stores the source ZIP file (`Dockerfile` + `index.html`) used by CodeBuild |
| **Lambda** | Automates file uploads, build triggers, and ECS force-deployment |
| **CodeBuild** | Builds the Docker image and pushes it to ECR |
| **ECR** | Stores the Docker image that ECS pulls to run your container |
| **ECS (Fargate)** | Runs the NGINX container — you scale and monitor it |
| **ALB** | Routes HTTP traffic to your containers and health-checks them |
| **CloudWatch Logs** | Captures all NGINX request logs from your container in real time |

---

## What Happens to the Environment After the Lab

After your lab session ends, the entire environment — the AWS account, all resources inside it, and the CloudFormation stack — will be automatically cleaned up by the CloudLabs provisioning system. You do not need to manually delete anything at the end of the lab unless explicitly instructed to do so during a specific task.

---

## Proceed to Task 1

Once you have read and understood this overview, proceed to **Task 1: Accessing Your AWS Lab Account** to begin.