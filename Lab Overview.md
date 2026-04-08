# Lab Overview: Deploying a Containerized App on AWS with ECS and CloudFormation

**Lab Title:** Deploying and Managing a Containerized Web Application on AWS ECS Fargate
**Difficulty:** Beginner to Intermediate
**Total Tasks:** 6
**Estimated Time:** 2 to 3 Hours
**Lab Type:** Guided Hands-On

---

## What This Lab Is About

This lab gives you real, hands-on practice with some of the most widely used tools in cloud computing. You will work directly with AWS services that companies use every day to run containerized applications — things like load balancers, container registries, build pipelines, and monitoring.

Here's the best part: you don't start from scratch. By the time you log in, the entire application is already running. A web server is live, traffic is flowing, and logs are being collected. Your job is to understand how it all fits together, explore the moving parts, make some changes, and see what happens.

Everyone gets their own private AWS account for this lab, so you can experiment freely without affecting anyone else.

---

## What's Already Set Up For You

When you log in, all of this will already be in place:

- Your own AWS account, just for this lab
- A private network (VPC) with two subnets in two different data centers
- An S3 bucket holding your application's source files
- A Docker image that was built automatically and stored in a private registry
- A live NGINX web server running inside a container
- A load balancer routing traffic to that container
- Log collection set up in CloudWatch
- All the permission roles the services need to talk to each other

You don't need to build any of this — it was all created automatically before you arrived. Your focus is on understanding what's running and why it works the way it does.

---

## Your Lab Login Details

When your lab starts, you'll get a Lab Details Page on CloudLabs with everything you need. Keep that page open the whole time.

| What It Is | What It's For |
|---|---|
| AWS Console Login URL | The web address to open the AWS dashboard |
| Username | Your login name for this lab's AWS account |
| Password | Your password |
| AWS Region | The location where all your resources are running |
| Account ID | A 12-digit number that identifies your account |

---

## The Technologies You'll Work With

This section explains every service and concept you'll encounter in the lab. Read through it before you start — it'll make everything easier to follow.

---

### AWS (Amazon Web Services)

AWS is Amazon's cloud platform. Instead of buying physical servers, you rent computing power, storage, and networking over the internet. In this lab, AWS is where your entire application lives.

---

### The AWS Management Console

This is the website where you manage everything in AWS. It's a point-and-click interface — no code required. You'll use it to view your infrastructure, check logs, and make changes throughout the lab.

---

### Regions and Availability Zones

AWS runs data centers all over the world. A **region** is a geographic area — like US East (Virginia) or Asia Pacific (Mumbai). Everything in your lab lives in one region.

Inside each region, there are multiple **Availability Zones** — separate physical data centers close enough to work together quickly, but far enough apart that a fire or power outage in one won't affect the others.

Your application runs across two Availability Zones, which means it can stay online even if one data center has a problem.

---

### CloudFormation

CloudFormation is what built your entire lab environment automatically. Instead of clicking through the AWS console to create each resource one by one, someone wrote a template file — a structured text file in JSON format — that describes everything that needs to exist. CloudFormation read that file and created everything in the right order.

Think of it like a recipe. The recipe describes what you want to end up with, and CloudFormation does the cooking.

The collection of everything created from that one template is called a **stack**. In your lab, it's called `nginx-app`.

A few things to know about how CloudFormation works:

- Everything in the stack is created, updated, and deleted together as a group.
- If something goes wrong during creation, CloudFormation automatically rolls back and removes whatever it had already created.
- You can update a running stack by changing the template and telling CloudFormation to apply it. CloudFormation figures out what changed and only touches those parts.
- The **Outputs** tab on your stack shows useful values like your application's URL.
- The **Events** tab shows a timestamped log of everything CloudFormation has done — great for troubleshooting.

#### The Three Parameters in This Template

The template has three inputs you can adjust:

| Parameter | What It Does |
|---|---|
| `AppName` | The name used as a prefix for every resource. All your resources are named things like `nginx-app-alb`, `nginx-app-cluster`, and so on. Don't change this — it'll break the naming of everything. |
| `CheckAcknowledgement` | Just a confirmation flag that the stack needs IAM permissions to run. Leave it as `TRUE`. |
| `DeployVersion` | This is how you tell CloudFormation to re-run the build pipeline. Every time you change something in your application, you need to increase this number by 1. Without doing that, CloudFormation won't re-upload your files, won't rebuild the Docker image, and won't redeploy your container — even if your code changed. It's the trigger that makes updates actually happen. |

---

### VPC (Virtual Private Cloud)

A VPC is your own private section of the AWS network. Nothing outside can reach your resources unless you explicitly open the door. Think of it as a fenced compound — you decide what goes in and what comes out.

Your VPC uses the address range `10.0.0.0/16`, which gives you a large pool of internal IP addresses to assign to your resources.

---

### Subnets

A subnet is a smaller section carved out of your VPC. Each subnet lives in one Availability Zone and has its own range of IP addresses.

Your lab has two subnets:
- `10.0.1.0/24` in the first Availability Zone
- `10.0.2.0/24` in the second Availability Zone

Both are public subnets, meaning resources inside them can communicate with the internet.

---

### Internet Gateway

The Internet Gateway is the connection between your VPC and the public internet. Without it, nothing inside your VPC can send or receive internet traffic — even if it has a public IP address.

Think of it as the front door of your building. The VPC is the building, and the Internet Gateway is the only door.

---

### Security Groups

A Security Group is a firewall for individual AWS resources. It controls which traffic is allowed in (inbound) and which traffic is allowed out (outbound).

Your lab has two Security Groups:

- **ALB Security Group** — lets anyone on the internet send HTTP traffic (port 80) to the load balancer. This is how visitors reach your website.
- **ECS Security Group** — only lets traffic in from the ALB Security Group. This means your containers can never be reached directly from the internet — all traffic has to go through the load balancer first.

---

### S3 (Simple Storage Service)

S3 is a file storage service. You store files called **objects** inside containers called **buckets**.

In this lab, an S3 bucket holds a file called `app.zip`. Inside that ZIP are two files: a `Dockerfile` (instructions for building your container image) and an `index.html` (the web page your server will show visitors). The build system downloads this ZIP from S3 whenever it needs to build a new image.

The bucket name includes your account ID and region to make it globally unique — for example, `nginx-app-source-123456789012-us-east-1`. Versioning is turned on, so every time the ZIP is uploaded, the old version is preserved as a backup.

---

### Lambda

Lambda is a way to run code without managing any servers. You write a function, and AWS runs it when something triggers it. When the function finishes, AWS shuts everything down. You only pay for the time your code actually ran.

Your lab uses **four Lambda functions**, each with a specific job in the deployment process:

**1. Ensure Service Linked Roles Lambda** (`nginx-app-ensure-slr`)

Before ECS and the load balancer can be created, AWS needs certain background permission roles to already exist. In a brand new AWS account, these roles don't exist yet. This Lambda checks whether they're there — and if not, creates them. It then waits 30 seconds for those changes to take effect before giving CloudFormation the green light to continue. Without this step, the ECS cluster and load balancer creation would fail in fresh accounts.

**2. Upload Files Lambda** (`nginx-app-upload-files`)

This Lambda generates the `Dockerfile` and `index.html` files from scratch in memory, packages them into `app.zip`, and uploads that ZIP to the S3 bucket. This is how your source files end up in S3 — you don't have to do it manually. On stack deletion, it also cleans up by removing `app.zip` from S3.

**3. Wait For Build Lambda** (`nginx-app-wait-for-build`)

Once the build system starts building your Docker image, this Lambda waits for it to finish. It checks the build status every 15 seconds. When the build succeeds, it tells CloudFormation to continue. If the build fails, it tells CloudFormation to roll back. This is the gatekeeper — the ECS service won't start until this Lambda gives the all-clear. It has a 15-minute timeout and includes a safety check: if it's about to run out of time, it fails gracefully instead of leaving CloudFormation hanging indefinitely.

**4. Force Deploy Lambda** (`nginx-app-force-deploy`)

After the ECS service is up and running, this Lambda tells ECS to force a fresh deployment — even if nothing in the task definition itself changed. This guarantees that every time you increment `DeployVersion` and CloudFormation runs the pipeline, ECS will actually pull the latest image from ECR and replace the running container. Without this, ECS might keep running an old container.

All four functions are triggered by CloudFormation during the stack deployment — CloudFormation calls each one and waits for it to report back before moving on.

---

### ECR (Elastic Container Registry)

ECR is a private Docker image registry — like a secure, private version of Docker Hub. This is where your Docker images are stored after they're built.

Your repository is named `nginx-app-repo`. Every time a build runs, two versions of the image are pushed:
- `:latest` — always points to the most recently built image
- `:YYYYMMDDHHMMSS` — a timestamp snapshot of that exact build, so you can always go back to a specific version

Image scanning on push is enabled, meaning every new image is automatically checked for known security vulnerabilities.

---

### CodeBuild

CodeBuild is AWS's build service. It takes your source files, runs a set of build commands, and produces an output — in this case, a Docker image pushed to ECR.

Each build runs in a temporary, clean environment that gets thrown away when the build finishes. This means every build starts fresh with no leftover state from previous builds.

The build follows three phases:

| Phase | What Happens |
|---|---|
| Pre-build | Logs into ECR using AWS credentials; sets a version tag using the current date and time |
| Build | Runs `docker build` to create the image; tags it as `:latest` and with the timestamp tag |
| Post-build | Runs `docker push` to send both tagged images to ECR |

The CodeBuild project is named `nginx-app-build`. Build logs go to CloudWatch under `/codebuild/nginx-app`.

---

### Docker and Containers

A container is a self-contained package that includes an application and everything it needs to run — the web server, configuration files, your HTML page. It runs the same way regardless of what machine it's on.

Docker is the tool used to build and run containers. A **Dockerfile** is a text file with step-by-step instructions for building a Docker image.

Your Dockerfile does this:
1. Starts from the official NGINX image pulled from the **Amazon ECR Public Gallery** (`public.ecr.aws/nginx/nginx:1.29`). The ECR Public Gallery is used instead of Docker Hub because Docker Hub has rate limits that can cause builds to fail on shared infrastructure.
2. Clears out the default HTML files that come with NGINX
3. Copies your `index.html` into the right folder inside the container
4. Sets the file permissions so NGINX can read it
5. Tells NGINX to start automatically when the container runs

---

### ECS (Elastic Container Service)

ECS is the AWS service that runs and manages your containers. It handles starting them, restarting them if they crash, and scaling them up or down.

Here are the four ECS concepts you'll encounter:

**Cluster** — The logical container for all your running tasks. Yours is named `nginx-app-cluster`. It uses Fargate as its compute engine.

**Task Definition** — The blueprint for your container. It specifies which image to run (`:latest` from ECR), how much CPU (256 units) and memory (512 MB) to give it, which port to open (port 80), where to send logs, and which IAM role gives it permission to pull images and write logs.

**Task** — A single running instance of your container. With `DesiredCount` set to 1, ECS always tries to keep one task running. If it crashes, ECS automatically starts a replacement.

**Service** — The manager that keeps the right number of tasks running and connects them to the load balancer. Yours is named `nginx-app-service`.

New containers get a 60-second grace period when they start up before the load balancer checks whether they're healthy. This prevents ECS from killing a container that just needs a few seconds to initialize.

---

### Fargate

Fargate is the engine underneath ECS. It runs your containers without requiring you to manage any servers. You don't provision virtual machines, patch operating systems, or worry about capacity. You just define your container, and Fargate handles the rest. You only pay for the CPU and memory your containers actually use.

---

### Application Load Balancer (ALB)

The ALB is the front door for your application. It accepts HTTP requests from the internet on port 80 and distributes them to your running containers. Visitors never talk to your containers directly — they talk to the ALB, and the ALB passes requests along.

The ALB continuously runs health checks — it sends a test request to each container every 15 seconds and checks the response. A container needs to pass 2 health checks in a row to be considered healthy, and fail 3 in a row to be considered unhealthy. The health check accepts any response between 200 and 399 (not just 200) so that NGINX redirects don't get misread as failures.

When a container is being shut down, the ALB gives it 10 seconds to finish handling any active requests before fully removing it. This short deregistration delay keeps deployments fast.

The ALB spans both public subnets across two Availability Zones, so it stays available even if one data center has an issue.

**Target Group** — This is the list of containers the ALB knows about. When ECS starts a new container, it automatically adds that container's private IP to the target group. When the container stops, it's automatically removed.

**Application URL** — The ALB's public DNS name is your application's address. You'll find it in the CloudFormation Outputs tab as `ApplicationURL`.

---

### CloudWatch

CloudWatch is AWS's monitoring service. It collects logs and metrics from your running services and lets you search and analyze them.

In this lab, CloudWatch does two things:

**Container Logs** — Everything your NGINX container writes to the console gets sent to a CloudWatch Log Group named `/ecs/nginx-app`. Logs are kept for 7 days. Each container writes to its own log stream, so if you're running multiple containers, you'll see multiple streams.

CodeBuild also sends its build logs to CloudWatch, under `/codebuild/nginx-app`.

**Logs Insights** — A built-in query tool that lets you search your logs with simple queries. Instead of scrolling through thousands of lines, you can filter and count entries in seconds. In this lab, you'll use it to count page requests and find 404 errors.

---

### IAM (Identity and Access Management)

IAM controls who or what is allowed to do things in AWS. Every action — whether from a user, a Lambda function, or an ECS task — has to be authorized.

IAM uses **Roles** — a set of permissions that gets attached to a service or user. All six IAM roles in this lab are created automatically by CloudFormation with auto-generated names (no hardcoded names in the template). You don't need to create them manually.

Here's what each role does:

| Role | Used By | What It's Allowed To Do |
|---|---|---|
| Setup Lambda Role | Ensure SLR Lambda | Create ECS and ELB service-linked roles in IAM; read existing IAM roles; write logs to CloudWatch |
| Upload Lambda Role | Upload Files Lambda | Put and delete objects in the S3 source bucket; write logs to CloudWatch |
| CodeBuild Role | CodeBuild project | Log in to ECR; push images to the ECR repository; read from the S3 bucket; write build logs to CloudWatch |
| ECS Task Execution Role | ECS tasks | Pull images from ECR; send container output to CloudWatch logs |
| Build Lambda Role | Wait For Build Lambda | Start a CodeBuild build; check build status; write logs to CloudWatch |
| Force Deploy Lambda Role | Force Deploy Lambda | Update the ECS service; describe the ECS service and cluster; write logs to CloudWatch |

Each role is given only the minimum permissions it needs and nothing more. For example, the Upload Lambda Role can only write to the one specific S3 bucket created by this stack — not any other bucket in the account.

Because all role names are auto-generated, this stack only needs the basic `CAPABILITY_IAM` permission to deploy. If the roles had hardcoded names, it would need the more restricted `CAPABILITY_NAMED_IAM`, which some lab platforms don't allow.

---

## How Everything Connects

Here's a simplified view of how data flows through the system — from the moment the stack is deployed to a visitor loading the web page:

```
CloudFormation deploys the stack
          │
          ▼
Ensure SLR Lambda
  Checks/creates IAM service-linked roles for ECS and ELB
  Waits 30 seconds for IAM to propagate
          │
          ▼
Upload Files Lambda
  Builds Dockerfile + index.html in memory
  Packages them into app.zip
  Uploads app.zip to S3
          │
          ▼
CodeBuild Project
  Downloads app.zip from S3
  Builds the Docker image (from ECR Public Gallery base)
  Pushes :latest and :timestamp image to ECR
          │
          ▼
Wait For Build Lambda
  Polls CodeBuild every 15 seconds
  Signals CloudFormation when build succeeds
          │
          ▼
ALB + ECS Cluster + ECS Service created
  ECS pulls :latest image from ECR
  Container registers with the ALB target group
  ALB health checks pass
          │
          ▼
Force Deploy Lambda
  Triggers a fresh ECS deployment
  Ensures the latest image is actually running
          │
          ▼
Application is live at the ALB URL

Visitor → ALB (port 80) → ECS Container (port 80) → NGINX → index.html
Container logs → CloudWatch /ecs/nginx-app
```

---

## The Full Deployment Sequence

CloudFormation creates everything in dependency order. Here's exactly what happened — step by step — when your lab was provisioned:

| Step | What Happened | Service Involved |
|---|---|---|
| 1 | VPC, subnets, Internet Gateway, and route tables created | CloudFormation + EC2 |
| 2 | Security groups created for the ALB and ECS tasks | CloudFormation + EC2 |
| 3 | S3 bucket created (with versioning enabled) to store source files | CloudFormation + S3 |
| 4 | ECR repository created (with scan on push enabled) to store Docker images | CloudFormation + ECR |
| 5 | All six IAM roles created | CloudFormation + IAM |
| 6 | All four Lambda functions created and ready | CloudFormation + Lambda |
| 7 | Ensure SLR Lambda ran: checked for and created ECS and ELB service-linked roles in IAM, then waited 30 seconds for those changes to propagate | Lambda + IAM |
| 8 | Upload Files Lambda ran: generated Dockerfile and index.html, zipped them as app.zip, uploaded to S3 | Lambda + S3 |
| 9 | CodeBuild project created (pointing at the app.zip in S3) | CloudFormation + CodeBuild |
| 10 | Wait For Build Lambda ran: started the CodeBuild build and began polling every 15 seconds | Lambda + CodeBuild |
| 11 | CodeBuild built the Docker image using the ECR Public Gallery NGINX base image, then pushed both `:latest` and a timestamp-tagged copy to ECR | CodeBuild + ECR |
| 12 | Wait For Build Lambda confirmed build success and told CloudFormation it was safe to continue | Lambda + CloudFormation |
| 13 | ECS cluster (Fargate), task definition, ALB, target group, and listener created | CloudFormation + ECS + EC2 |
| 14 | ECS service started and pulled `:latest` from ECR; container came online | ECS + ECR |
| 15 | Container's private IP registered in the ALB target group | ECS + ALB |
| 16 | ALB health checks passed — application is live | ALB |
| 17 | Force Deploy Lambda ran: issued a forced redeployment to guarantee the latest image is running | Lambda + ECS |

> **Why does step order matter?** If ECS had started before step 11 finished, every container launch would fail because there was nothing to pull from ECR. The Wait For Build Lambda is the gatekeeper that prevents that. And step 7 matters in brand-new accounts — without the ECS and ELB service-linked roles, steps 13 and beyond would fail immediately.

---

## Rules to Follow During the Lab

- Don't delete the CloudFormation stack or any resources inside it while the lab is running.
- Don't change the `AppName` parameter — all resource names are built from it, and changing it will break things.
- Every time you change your application content, increment `DeployVersion` by 1. Without that, your changes won't actually go anywhere.
- Every task that asks you to make a change will also ask you to revert it. Always finish the revert before moving to the next task.
- Don't manually create or delete IAM roles — CloudFormation owns them.
- If a stack update ends with `UPDATE_ROLLBACK_COMPLETE` instead of `UPDATE_COMPLETE`, there was an error in your template. Read the Events tab for the error message, fix it, and try again.
- Don't try to access resources in other participants' accounts.
- If something unexpected goes wrong that isn't described in the task instructions, contact your instructor.

---

## After the Lab Ends

When the lab session is over, the CloudLabs platform automatically shuts down and deletes the entire environment — the account, all resources, and all data. You don't need to clean up anything yourself unless a specific task tells you to.

---

## Ready to Start?

Head to **Task Overview** to know what are the tasks that you are going to perform hands-on.