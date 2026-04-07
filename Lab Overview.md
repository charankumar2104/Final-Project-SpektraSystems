# Lab Overview: Deploying Containerized Applications on AWS with ECS and CloudFormation

**Lab Title:** Deploying and Managing a Containerized Web Application on AWS ECS Fargate — End-to-End Hands-On Lab
**Difficulty:** Beginner to Intermediate
**Total Tasks:** 6
**Total Estimated Duration:** 2 to 3 Hours
**Lab Type:** Guided Hands-On

---

## About This Lab

This lab is designed to give you practical, real-world experience working with core Amazon Web Services (AWS) — specifically around containerized application deployment, automated infrastructure provisioning, and cloud monitoring.

You will not just read about these technologies — you will use them directly through a browser-based AWS environment that has been pre-configured specifically for you. Before you even log in, the entire application infrastructure — including networking, a Docker container, a load balancer, a build pipeline, and a container registry — has already been deployed and is running live.

Each participant in this lab receives their own isolated AWS account. This means everything you do stays within your own assigned space and does not affect any other participant. You are free to explore, make changes, and learn without the risk of affecting anyone else's work.

By the end of this lab, you will have worked hands-on with the core AWS services that power modern cloud-native applications — understanding not just what each service does, but how they connect and depend on each other to deliver a running web application.

---

## How the Lab Environment Works

When you receive your Lab Details Page on CloudLabs, your environment has already been provisioned and is ready for you. The following things were automatically set up before you arrived:

- A dedicated AWS account provisioned exclusively for you
- A full networking foundation: a VPC, two public subnets across different Availability Zones, an Internet Gateway, and route tables
- An S3 bucket pre-populated with your application source files
- A Docker image automatically built using CodeBuild and stored in a private ECR repository
- A running NGINX web application inside an ECS Fargate container
- An Application Load Balancer distributing traffic to your running container
- CloudWatch logging configured to capture all application output
- All IAM roles required for the services to communicate with each other

You do not need to set up any of the above. Your job is to explore, understand, use, and build on top of what has been provisioned.

---

## Lab Environment Details

At the start of the lab, your **Lab Details Page on CloudLabs** will contain the following information. Keep this page open throughout the entire lab, as you will refer to it many times.

| Item | Description |
|---|---|
| AWS Console Login URL | The web address to access the AWS Management Console for your account |
| Username | Your personal IAM user login for the AWS Console |
| Password | Your password for the AWS Console |
| AWS Region | The AWS region where all your lab resources are deployed |
| Account ID | The unique 12-digit identifier of your AWS account |

---

## Key Technologies and Resources You Will Work With

Below is a plain-language definition of every technology and resource you will encounter during this lab. Read through these definitions before starting the tasks so that you have a baseline understanding of what each thing is.

---

### Amazon Web Services (AWS)

Amazon Web Services is a cloud computing platform provided by Amazon. It allows individuals and organizations to rent computing resources — such as servers, storage, databases, and networking — over the internet rather than buying and managing physical hardware themselves.

In this lab, AWS is the platform where your entire application infrastructure lives. You will interact with it through the AWS Management Console, which is a web-based graphical interface.

---

### AWS Management Console

The AWS Management Console is the web-based interface you use to manage all of your AWS resources. You access it by going to the login URL provided on your Lab Details Page. From the console, you can view, create, modify, and monitor AWS resources using a graphical point-and-click interface without needing to write any code.

In this lab, you will use the AWS Management Console to explore your infrastructure, verify your running application, make changes, and monitor logs.

---

### AWS Region and Availability Zones

An AWS **Region** is a specific geographic location where AWS operates data centers — for example, US East (N. Virginia), EU West (Ireland), or Asia Pacific (Mumbai). Every resource you create in AWS must be placed in a specific region.

Within each region, there are multiple **Availability Zones (AZs)** — physically separate data centers that are close enough to communicate quickly but far enough apart that if one experiences a power outage or hardware failure, the others are not affected.

In this lab, your application is deployed across **two Availability Zones** within a single region. This means your load balancer and containers can continue running even if one data center experiences a problem.

---

### AWS CloudFormation

AWS CloudFormation is the service that deployed everything in this lab for you automatically — before you even logged in.

CloudFormation is an **Infrastructure as Code (IaC)** service. Instead of manually clicking through the AWS Console to create resources one by one, a CloudFormation **template** — a structured text file written in YAML or JSON format — describes every resource you need, and CloudFormation reads that template and creates everything in the correct order automatically.

A CloudFormation **Stack** is the collection of AWS resources that were created together from a single template. Think of the stack as a group — all the resources in your lab (the network, the containers, the load balancer, the storage) belong to one stack called `nginx-app`.

Key characteristics of CloudFormation:
- All resources in a stack are created, updated, and deleted together
- If any resource fails to create, CloudFormation rolls back everything automatically
- You can update a running stack by editing the template and submitting the change — CloudFormation figures out exactly what needs to change and applies only those differences
- The **Outputs** tab of a stack shows you important values like the URL of your deployed application
- The **Events** tab shows you a timestamped log of every action CloudFormation has taken

In this lab, you will update the CloudFormation stack to make changes to your application, and then revert those changes — all without manually clicking through individual service consoles.

#### CloudFormation Parameters

The template for this lab uses three parameters that control how it behaves:

| Parameter | Purpose |
|---|---|
| `AppName` | The base name applied to every resource in the stack (e.g., `nginx-app`). Do not change this value. |
| `CheckAcknowledgement` | Acknowledges that the stack requires IAM capabilities. Defaults to `TRUE`. |
| `DeployVersion` | A version number you increment by 1 every time you change application content. Changing this value is what forces the full pipeline to re-run — it causes the Upload Lambda to re-upload `app.zip` to S3, CodeBuild to rebuild the Docker image, ECR to receive the new image, and ECS to force-deploy the updated container. **Without incrementing this value, CloudFormation will skip all downstream steps even if you edited other parts of the template.** |

---

### Amazon VPC (Virtual Private Cloud)

A **VPC** is your own private, isolated section of the AWS network. Think of it as a fenced-off piece of the internet that belongs entirely to you — other AWS customers cannot see into it or reach the resources inside it unless you explicitly allow them to.

Inside your VPC, your resources can communicate with each other privately. Resources that need to be accessible from the internet (like a load balancer) are placed in **public subnets**, while resources that should stay private (like databases) are placed in **private subnets**.

In this lab, your VPC uses the address range `10.0.0.0/16`. It contains two public subnets — one in each Availability Zone — where your load balancer and containers run.

---

### Subnets

A **subnet** is a smaller subdivision of your VPC. It is tied to a specific Availability Zone and has its own range of IP addresses. Resources launched into a subnet get an IP address from that subnet's range.

In this lab, your two public subnets use the address ranges `10.0.1.0/24` (AZ 1) and `10.0.2.0/24` (AZ 2). They are marked as **public subnets** because they have a route to the Internet Gateway, which means resources in them can communicate with the internet.

---

### Internet Gateway

An **Internet Gateway** is the component that connects your VPC to the internet. Without it, nothing inside your VPC can send or receive traffic from the public internet — not even resources with a public IP address.

Think of it like the front door of your building. The building (VPC) is fully secured, but the Internet Gateway is the door that allows authorized traffic to come in and go out.

In this lab, the Internet Gateway is attached to your VPC and routes all outbound internet traffic to and from your public subnets.

---

### Security Groups

A **Security Group** is a virtual firewall that controls what network traffic is allowed to reach (inbound) or leave (outbound) an AWS resource.

Security Groups use **rules** that specify:
- The **protocol** (TCP, UDP, or all)
- The **port** (e.g., port 80 for web traffic, port 443 for HTTPS)
- The **source or destination** (an IP address range, or another Security Group)

By default, a Security Group denies all inbound traffic and allows all outbound traffic. You must explicitly add rules to allow inbound connections.

In this lab, there are two Security Groups:
- **ALB Security Group** — allows inbound web traffic (port 80) from anyone on the internet, so the load balancer can receive visitor requests
- **ECS Security Group** — allows inbound traffic on port 80 **only from the ALB Security Group** — meaning your containers can only be reached through the load balancer, never directly from the internet

---

### Amazon S3 (Simple Storage Service)

**Amazon S3** is an object storage service — a place to store any type of file, from a single text file to terabytes of data. Files stored in S3 are called **objects**, and they are organized into **buckets** (named containers, similar to folders).

S3 is highly durable — your files are automatically replicated across multiple physical locations to protect against hardware failure.

In this lab, an S3 bucket stores a ZIP file called `app.zip`. This ZIP file contains your `Dockerfile` (the recipe for building your Docker image) and your `index.html` (the web page your NGINX server will serve). AWS CodeBuild downloads this file from S3 at the start of every build.

The bucket name follows the format `{AppName}-source-{AccountId}-{Region}` — for example, `nginx-app-source-123456789012-us-east-1`. Versioning is enabled on the bucket so that each upload of `app.zip` is preserved, making it possible to track changes over time.

---

### AWS Lambda

**AWS Lambda** is a service that lets you run small pieces of code without provisioning or managing any servers. You upload your code, define what triggers it to run, and Lambda handles everything else — starting a temporary compute environment, running your code, and shutting it down when done.

Lambda functions are billed per millisecond of execution time and automatically scale to handle any number of simultaneous runs.

In this lab, **four Lambda functions** play roles in the automated deployment:

1. **Ensure Service Linked Roles Lambda** (`{AppName}-ensure-slr`) — Runs first during deployment and checks whether the AWS-managed service-linked IAM roles for ECS and Elastic Load Balancing already exist in the account. If they do not, it creates them automatically. This is critical in brand-new AWS accounts where these roles have never been created before. It then waits 30 seconds for IAM changes to propagate before signaling CloudFormation to continue. Without this step, creating the ECS cluster and load balancer could fail in fresh accounts.

2. **Upload Files Lambda** (`{AppName}-upload-files`) — Generates the `Dockerfile` and `index.html` files in memory, compresses them into a ZIP file, and uploads the ZIP to your S3 bucket. This means you do not need to create or upload any files manually — the Lambda does it automatically as part of the CloudFormation deployment.

3. **Wait For Build Lambda** (`{AppName}-wait-for-build`) — After CodeBuild starts building the Docker image, this Lambda waits and polls CodeBuild every 15 seconds until the build either succeeds or fails. It then signals back to CloudFormation whether to continue or roll back. This is what prevents ECS from trying to start containers before the Docker image even exists in ECR.

4. **Force Deploy Lambda** (`{AppName}-force-deploy`) — Runs after the ECS service is created or updated and issues a `ForceNewDeployment` command to ECS. This ensures that whenever `DeployVersion` is incremented, ECS pulls the freshly built image from ECR and replaces the running container — even if the task definition itself has not changed.

Lambda functions in this lab are triggered by **CloudFormation Custom Resources** — a mechanism that allows CloudFormation to call a Lambda function during a stack deployment and wait for it to report success or failure before continuing.

---

### Amazon ECR (Elastic Container Registry)

**Amazon ECR** is a fully managed private Docker container registry. Think of it like a secure, private version of Docker Hub — a place where you store Docker images that only your own AWS resources can pull from.

A **Docker image** is a packaged, portable snapshot of an application and everything it needs to run — the operating system libraries, the web server software, your custom HTML files, and the configuration. When you want to run a container (a live, running instance of an image), you pull the image from a registry and start it.

In this lab, your ECR repository is named `{AppName}-repo` (e.g., `nginx-app-repo`). Image scanning on push is enabled, meaning every new image is automatically scanned for known security vulnerabilities. After every build, CodeBuild pushes two versions of the image into this repository:
- **`:latest`** — a tag that always points to the most recently built image
- **`:YYYYMMDDHHMMSS`** — a timestamp tag that permanently records the exact image from that specific build, making it possible to roll back to an older version if needed

---

### AWS CodeBuild

**AWS CodeBuild** is a fully managed build service. It takes your source code (in this case, a `Dockerfile` and an `index.html`), runs a sequence of build commands you define, and produces an output — in this lab, a Docker image pushed to ECR.

CodeBuild runs each build inside a temporary, isolated container environment. When the build finishes, that environment is discarded. This ensures that every build starts from a clean state.

A CodeBuild **buildspec** is the set of instructions CodeBuild follows. It is divided into phases:

| Phase | What Happens |
|---|---|
| `pre_build` | Log in to ECR and prepare environment variables (like a version tag based on the current timestamp) |
| `build` | Run `docker build` to create the Docker image from the Dockerfile and tag it with both `:latest` and the timestamp tag |
| `post_build` | Run `docker push` to upload both tagged images to ECR |

In this lab, the CodeBuild project is named `{AppName}-build` (e.g., `nginx-app-build`). It is configured to read your source ZIP from S3 and push the resulting Docker image to ECR. Build logs for every run are stored in a CloudWatch Logs group named `/codebuild/{AppName}`. You can view the full build logs for every run in the CodeBuild console.

---

### Docker and Containers

A **container** is a lightweight, portable, self-contained unit of software. It packages an application together with all its dependencies — libraries, configuration files, and the runtime — into a single unit that runs consistently regardless of what environment it is deployed on.

**Docker** is the most widely used platform for building and running containers. A **Dockerfile** is a text file that describes the step-by-step instructions for building a Docker image.

In this lab, the Dockerfile for your application does the following:
1. Starts from the official NGINX image pulled from the **Amazon ECR Public Gallery** (`public.ecr.aws/nginx/nginx:1.29`). The ECR Public Gallery is used instead of Docker Hub to avoid anonymous pull rate limits that can occur on shared build infrastructure.
2. Removes any default HTML files that come with the base NGINX image
3. Copies your `index.html` web page into the correct folder inside the container
4. Sets the correct file permissions so NGINX can read the file
5. Configures NGINX to start automatically when the container runs

---

### Amazon ECS (Elastic Container Service)

**Amazon ECS** is the AWS service that runs and manages your Docker containers at scale. It handles starting containers, stopping them, restarting them if they crash, and scaling the number of running containers up or down based on demand.

ECS uses several key concepts you will encounter in this lab:

**Cluster**
A cluster is the logical grouping that contains your running containers. Think of it like a pool of computing capacity. In this lab, your cluster is named `{AppName}-cluster` (e.g., `nginx-app-cluster`). The cluster is configured to use Fargate as its capacity provider.

**Task Definition**
A Task Definition is like a recipe for your container. It describes:
- Which Docker image to run (pulled from ECR using the `:latest` tag)
- How much CPU and memory to allocate
- Which ports to open
- Where to send log output
- Which IAM role to use for permissions

In this lab, the task definition pulls the `{AppName}-repo:latest` image from ECR and runs it with 256 CPU units and 512 MB of memory.

**Task**
A Task is a single running instance of a container, created from a Task Definition. When you set `DesiredCount: 1`, ECS ensures exactly one task is always running. If that task crashes, ECS automatically starts a replacement.

**Service**
A Service is the configuration that tells ECS how many tasks to keep running at all times and how to connect them to a load balancer. In this lab, your service is named `{AppName}-service` (e.g., `nginx-app-service`) and is configured to maintain the desired number of running tasks at all times.

**Health Check Grace Period**
When a new task starts, it needs a moment to initialize before it can handle traffic. The Health Check Grace Period (set to 60 seconds in this lab) tells ECS to wait before checking whether a new task is healthy — preventing premature task restarts during startup.

---

### AWS Fargate

**AWS Fargate** is the compute engine that actually runs your ECS containers without requiring you to manage any servers. When you use Fargate, you do not need to create, patch, or scale EC2 virtual machine instances — you simply define your container and AWS handles all the underlying infrastructure.

Fargate is described as **serverless compute for containers** — you only pay for the CPU and memory your containers use while they are running, with no charge for idle capacity.

In this lab, all ECS tasks run on Fargate. This means you never need to log in to a server, patch an operating system, or worry about capacity planning.

---

### Application Load Balancer (ALB)

An **Application Load Balancer** is a traffic distribution service that sits in front of your containers and distributes incoming requests across all healthy running containers. It is the public-facing entry point for your application — visitors access the ALB's DNS name, and the ALB decides which container should handle each request.

The ALB continuously performs **health checks** — it periodically sends a test request (`GET /`) to each container and checks the response code. If a container is healthy, it receives traffic. If a container fails health checks (for example, because it crashed), the ALB stops sending traffic to it while ECS starts a replacement.

In this lab:
- The ALB listens on **port 80** (HTTP)
- It forwards traffic to the **Target Group**, which contains your running ECS tasks
- The health check accepts response codes **200–399** (not just 200) so that redirects from NGINX do not cause false unhealthy alerts
- Health checks run every 15 seconds, with a 5-second timeout, and a container must pass 2 consecutive checks to be considered healthy or fail 3 consecutive checks to be considered unhealthy
- When a task is deregistered (stopped), the ALB waits only 10 seconds before fully removing it — this is a reduced deregistration delay to make deployments faster
- The ALB spans **both public subnets** across two Availability Zones, so it remains available even if one data center has issues

**Target Group**
A Target Group is the collection of destinations the ALB can forward traffic to. For ECS Fargate, targets are registered by their private IP address. When ECS starts a new task, it automatically registers the task's IP in the target group. When a task stops, ECS automatically deregisters it.

**ALB DNS Name**
Instead of a fixed IP address, the ALB is assigned a DNS name (a web address) that you use to access your application. This DNS name is visible in the CloudFormation **Outputs** tab as the `ApplicationURL`.

---

### Amazon CloudWatch

**Amazon CloudWatch** is the monitoring and observability service in AWS. It collects metrics, logs, and events from virtually every AWS service and lets you search, visualize, and alert on them.

In this lab, CloudWatch is used for two things:

**CloudWatch Logs**
Every time your NGINX container handles a web request, it writes a log line describing that request. These log lines are automatically sent to a CloudWatch **Log Group** named `/ecs/{AppName}` (e.g., `/ecs/nginx-app`). Log entries are retained for 7 days.

Inside the log group, each running container writes to its own **Log Stream** — a separate file with a name that identifies which task generated the logs. This means if you are running two tasks, you will see two log streams.

CodeBuild build logs are also sent to CloudWatch, under a separate log group named `/codebuild/{AppName}`.

**CloudWatch Logs Insights**
Logs Insights is a query engine built into CloudWatch that lets you run structured queries across your log data using a simple query language. Instead of manually scrolling through thousands of log lines, you can write a query that filters, counts, or groups log entries and get results in seconds.

In this lab, you will use Logs Insights to count how many times each page was requested and identify which requests resulted in 404 (Not Found) errors.

---

### IAM (Identity and Access Management)

**AWS IAM** is the service that controls who or what is allowed to perform actions on AWS resources. Every request in AWS — whether made by a user, an application, or an AWS service — must be authenticated (proving who you are) and authorized (proving you are allowed to do what you are asking).

IAM works through **Roles** — collections of permissions that can be attached to users, AWS services, or applications. In this lab, all IAM roles are automatically created by CloudFormation. You do not need to create them manually.

The six roles in this lab:

| Role | Who Uses It | What It Allows |
|---|---|---|
| Setup Lambda Role | The Ensure Service Linked Roles Lambda | Create ECS and ELB service-linked roles in IAM, check if they already exist, and write logs to CloudWatch |
| Upload Lambda Role | The Upload Files Lambda | Write files to the S3 source bucket and write logs to CloudWatch |
| CodeBuild Role | The CodeBuild project | Read from S3, push images to ECR, and write logs to CloudWatch |
| ECS Task Execution Role | ECS, on behalf of your container | Pull images from ECR and send container logs to CloudWatch |
| Build Lambda Role | The Wait For Build Lambda | Start CodeBuild builds, check their status, and write logs to CloudWatch |
| Force Deploy Lambda Role | The Force Deploy Lambda | Update and describe the ECS service to trigger a new deployment, and write logs to CloudWatch |

An important security principle followed in this lab is **Least Privilege** — each role is given only the exact permissions it needs and nothing more. For example, the Upload Lambda Role can only write to the specific S3 bucket created by this stack, not to any other bucket in the account. Similarly, the Force Deploy Lambda Role can only update the specific ECS service created by this stack.

> ⚠️ **Note on IAM and this lab:** All IAM roles in this lab are created with **auto-generated names** (no custom name is specified). This is intentional — it means the lab deployment only requires the basic `CAPABILITY_IAM` permission, which automated lab platforms like CloudLabs already provide. It avoids the more restrictive `CAPABILITY_NAMED_IAM` requirement that would otherwise block automated deployment.

---

### YAML / JSON

AWS CloudFormation templates can be written in either **YAML** or **JSON** format. The template used in this lab is written in **JSON** — a structured text format that uses curly braces `{}`, square brackets `[]`, and quoted strings to represent configuration data.

**YAML** stands for "Yet Another Markup Language." It is an alternative to JSON that uses indentation (spaces at the beginning of lines) to define hierarchy, making it more readable for humans. You may encounter YAML in other CloudFormation examples and documentation.

This is important to understand because in Task 3 of this lab, you will edit a line inside the CloudFormation template. When editing, you must be careful to change only the specific text you intend to change and not to accidentally break the surrounding JSON structure — for example, by removing a quotation mark, a comma, or a closing brace. Doing so can cause the template to become invalid.

---

## Lab Architecture Summary

The diagram below shows how all the components in this lab connect to each other and the order in which data flows through the system — from initial deployment all the way through to a visitor loading the web page in a browser.

```
CloudLabs Automated Deployment
          |
          | deploys
          v
+---------------------------------------------+
|        AWS CloudFormation Stack             |
|              (nginx-app)                    |
+---------------------------------------------+
          |
          | creates and orchestrates
          v
+------------------------------------------------------------+
|                  AWS Lambda Functions                      |
|                                                            |
|  [1] Ensure Service Linked Roles Lambda                    |
|      Checks if ECS and ELB service-linked roles exist      |
|      Creates them in IAM if missing                        |
|      Waits 30 seconds for IAM propagation                  |
|      Signals CloudFormation to continue                    |
|                                                            |
|  [2] Upload Files Lambda                                   |
|      Generates Dockerfile + index.html in memory          |
|      Packages them into app.zip                            |
|      Uploads app.zip to S3                                 |
|                                                            |
|  [3] Wait For Build Lambda                                 |
|      Starts the CodeBuild build                            |
|      Polls every 15 seconds until build succeeds           |
|      Signals CloudFormation to continue                    |
|                                                            |
|  [4] Force Deploy Lambda                                   |
|      Issues a ForceNewDeployment to ECS                    |
|      Ensures ECS pulls the new image from ECR              |
+------------------------------------------------------------+
          |
          | uploads app.zip to
          v
+---------------------------+
|       Amazon S3 Bucket    |
|  (nginx-app-source-...)   |
|   stores: app.zip         |
|   versioning: enabled     |
+---------------------------+
          |
          | CodeBuild downloads app.zip from
          v
+-----------------------------------------------+
|            AWS CodeBuild Project              |
|            (nginx-app-build)                  |
|                                               |
|  PRE_BUILD:   Login to ECR, set version tag   |
|  BUILD:       docker build (creates image)    |
|  POST_BUILD:  docker push :latest + :tag      |
|  LOGS:        /codebuild/nginx-app            |
+-----------------------------------------------+
          |
          | pushes Docker image to
          v
+-----------------------------------------------+
|       Amazon ECR Repository                   |
|          (nginx-app-repo)                     |
|   scan on push: enabled                       |
|   stores: :latest image                       |
|           :YYYYMMDDHHMMSS image               |
+-----------------------------------------------+
          |
          | ECS pulls image from ECR
          v
+---------------------------------------------------+
|              Amazon VPC (10.0.0.0/16)             |
|                                                   |
|   Internet Gateway  --  Public Route Table        |
|                                                   |
|  +------------------+  +------------------+       |
|  | Public Subnet AZ1|  | Public Subnet AZ2|       |
|  |  (10.0.1.0/24)   |  |  (10.0.2.0/24)   |       |
|  +--------+---------+  +---------+--------+       |
|           |                      |                |
|           +----------+-----------+                |
|                      |                            |
|         +------------+-------------+              |
|         | Application Load Balancer|              |
|         |  (nginx-app-alb)         |              |
|         |  Port 80, internet-facing|              |
|         |  Health checks: 200-399  |              |
|         |  Deregister delay: 10s   |              |
|         +------------+-------------+              |
|                      |                            |
|                      | forwards traffic to        |
|                      v                            |
|         +----------------------------+            |
|         |  ALB Target Group          |            |
|         |  (nginx-app-tg)            |            |
|         |  registers ECS task IPs    |            |
|         +----------------------------+            |
|                      |                            |
|                      | routes to                  |
|                      v                            |
|         +----------------------------+            |
|         |  ECS Fargate Cluster       |            |
|         |  (nginx-app-cluster)       |            |
|         |                            |            |
|         |  Service: nginx-app-service|            |
|         |  Task: NGINX container     |            |
|         |  Image: ECR :latest        |            |
|         |  Port 80                   |            |
|         +----------------------------+            |
|                      |                            |
+---------------------------------------------------+
                        |
                        | container logs sent to
                        v
              +------------------------+
              |   Amazon CloudWatch    |
              |   Log Group:           |
              |   /ecs/nginx-app       |
              |   Retention: 7 days    |
              |                        |
              |   Logs Insights:       |
              |   query and analyze    |
              +------------------------+

Visitor in Browser
          |
          | opens ApplicationURL (ALB DNS Name)
          v
  NGINX web page loads:
  "Running on AWS ECS — Service is healthy and running"
```

---

> 📸 **Image Placeholder**
>
> *Insert a visual architecture diagram here showing all the above components as labeled boxes with color-coded groupings and directional arrows indicating the flow of data and dependencies between each service.*
>
> ```
> [ DIAGRAM: Full lab architecture diagram showing VPC, subnets, ALB, ECS cluster, ECR, CodeBuild,
>   S3 bucket, Lambda functions (x4), CloudWatch, and CloudFormation stack — all connected with labeled arrows ]
> ```

---

## The Automated Deployment Sequence

One of the most important things to understand about this lab is **the order in which everything is created and how the services depend on each other**. CloudFormation does not just create all resources at the same time — it waits for dependencies to complete before moving to the next step.

Here is the sequence that ran automatically when your lab was provisioned:

| Step | What Happened | Service |
|---|---|---|
| 1 | Networking created: VPC, subnets, internet gateway, route tables | CloudFormation + EC2 |
| 2 | Security groups created for ALB and ECS | CloudFormation + EC2 |
| 3 | S3 bucket created with versioning enabled to store application source files | CloudFormation + S3 |
| 4 | ECR repository created with image scanning enabled to store Docker images | CloudFormation + ECR |
| 5 | IAM roles created for all six Lambda, CodeBuild, and ECS roles | CloudFormation + IAM |
| 6 | All four Lambda functions created and ready | CloudFormation + Lambda |
| 7 | Ensure Service Linked Roles Lambda triggered: checked for and created ECS and ELB service-linked roles in IAM, then waited 30 seconds for propagation | Lambda + IAM |
| 8 | Upload Files Lambda triggered: generated Dockerfile and index.html, packaged them as app.zip, and uploaded to S3 | Lambda + S3 |
| 9 | CodeBuild project created | CloudFormation + CodeBuild |
| 10 | Wait For Build Lambda triggered: started the CodeBuild build, waited 3–5 minutes polling every 15 seconds | Lambda + CodeBuild |
| 11 | CodeBuild built the Docker image from the ECR Public Gallery base image and pushed both `:latest` and a timestamp-tagged image to ECR | CodeBuild + ECR |
| 12 | Wait For Build Lambda confirmed the build succeeded and notified CloudFormation | Lambda + CloudFormation |
| 13 | ECS cluster (with Fargate capacity provider), task definition, and ALB created | CloudFormation + ECS + EC2 |
| 14 | ECS service started the NGINX container (pulled `:latest` image from ECR) | ECS + ECR |
| 15 | Container registered with the ALB target group | ECS + ALB |
| 16 | ALB health checks passed — application is live and receiving traffic | ALB |
| 17 | Force Deploy Lambda triggered: issued a ForceNewDeployment to the ECS service to ensure the latest image is running | Lambda + ECS |

> ℹ️ **Why does this order matter?** Steps 11 and 12 are critical — if the ECS service started (Step 13) before the Docker image existed in ECR (Step 11), every container launch would fail because there is nothing to pull. The Wait For Build Lambda (Step 10) acts as a gate — CloudFormation cannot move past it until the build is confirmed successful. Similarly, Step 7 is critical in brand-new AWS accounts: the ECS cluster and ALB cannot be created until their service-linked IAM roles exist.

---

## Important Rules for This Lab

- Do not delete the CloudFormation stack or any of the resources created by it during the lab
- Do not change the `AppName` parameter value in the CloudFormation template — all resources are named based on this value and changing it will cause errors
- **Increment the `DeployVersion` parameter by 1 every time you change application content** — without this, CloudFormation will not re-run the build pipeline or redeploy your containers
- Every task in this lab that asks you to make a change also asks you to revert it — always complete the revert step before moving to the next task
- Do not manually create or delete IAM roles — the stack manages all IAM permissions automatically
- If a CloudFormation stack update shows `UPDATE_ROLLBACK_COMPLETE` instead of `UPDATE_COMPLETE`, it means the template had an error — read the Events tab error message, fix the template, and try again
- Do not attempt to access resources belonging to other lab participants
- If you see an error that you do not expect and that is not described as a normal part of the task you are working on, contact your instructor

---

## What Happens to the Environment After the Lab

After the lab session ends, the entire environment — the AWS account, all resources inside it, and all data — will be automatically deprovisioned by the CloudLabs lab platform. You do not need to manually delete anything at the end of the lab unless explicitly instructed to do so during a specific task.

---

## Proceed to Task 1

Once you have read and understood this overview, proceed to **Task 1: Accessing Your AWS Lab Account**, which will guide you step-by-step through logging in to your AWS Console using the credentials on your Lab Details Page.