# Task 2: Verify Your Deployed Application

**Difficulty:** Beginner
**Estimated Time:** 15–20 Minutes
**Type:** Exploration + Verification (You will find the live application URL and confirm it is running)

---

## Objective

In this task, you will **locate your live application URL** from the CloudFormation stack outputs and **open it in a browser** to confirm the NGINX web application is running. You will also explore the CloudFormation stack to understand the resources that were deployed automatically for you.

---

## Pre-Requisites

Before starting this task, make sure you have completed:

- Task 1: Accessing Your AWS Lab Account

Also have ready from your **Environment Details Page**:

| Item | Where to Find It |
|---|---|
| AWS Console Login URL | Environment Details Page |
| Username | Environment Details Page |
| Password | Environment Details Page |

---

## Quick Background: What Is CloudFormation?

**AWS CloudFormation** is a service that lets you describe your entire infrastructure in a file (called a template) and deploy it automatically. Think of it like a recipe — the recipe lists all the ingredients (AWS services) and the exact steps to put them together.

In this lab, a CloudFormation **stack** was already deployed for you before you logged in. A stack is a collection of AWS resources (like servers, networks, databases, etc.) that were all created together from a single template.

You will now look at this stack to find the URL of the application that was deployed.

---

## Part A: Navigate to the CloudFormation Service

---

### Step 1: Open the AWS Console Search Bar

1. Make sure you are logged in to the **AWS Management Console** (from Task 1)
2. Look at the very top of the page — there is a **search bar** in the centre with the placeholder text **"Search for services, features, blogs, docs, and more"**
3. Click on it

---

<img src="./Images/Screenshot 2026-04-07 090652.png">

---

### Step 2: Search for CloudFormation

1. With the search bar active, type: `CloudFormation`
2. As you type, a dropdown list of matching services will appear
3. You will see **"CloudFormation"** appear in the results under the **"Services"** section
4. Click on **CloudFormation**

---

<img src="./Images/Screenshot 2026-04-07 090724.png">

---

### Step 3: The CloudFormation Dashboard Opens

1. The **CloudFormation console** will open
2. You will land on the **Stacks** page, which shows all the stacks in your account
3. You should see **one stack** listed — this is the stack that was pre-deployed for you
4. The stack name will look something like **`nginx-app`**
5. Look at the **Status** column — it should show **`CREATE_COMPLETE`** in green

> ℹ️ **What does CREATE_COMPLETE mean?** It means the stack finished deploying successfully. All resources (networking, containers, load balancer, etc.) were created without errors.

---

<img src="./Images/Screenshot 2026-04-07 090809.png">

---

## Part B: Find the Application URL in Stack Outputs

---

### Step 4: Click on the Stack Name to Open It

1. In the list of stacks, click on the **stack name** (e.g., `stack-xxxxxxx`)
2. A detail page for this stack will open
3. At the top, you will see several tabs: **Stack info**, **Events**, **Resources**, **Outputs**, **Parameters**, and **Template**

---

<img src="./Images/Screenshot 2026-04-07 090919.png">

---

### Step 5: Click on the "Outputs" Tab

1. Click on the **"Outputs"** tab
2. This tab shows all the important values that the stack generated after deploying
3. You will see a table with columns: **Key**, **Value**, and **Description**

---

<img src="./Images/Screenshot 2026-04-07 091003.png">

---

### Step 6: Find the "ApplicationURL" Output

1. In the Outputs table, look for the row where the **Key** column says **`ApplicationURL`**
2. The **Value** column next to it will contain a URL starting with `http://` — this is the live URL of your deployed NGINX application
3. Copy this URL by clicking on it or manually selecting and copying the text

> ⚠️ **Do not close this tab.** You may need to come back to it. Open the URL in a new tab instead.

---

<img src="./Images/Screenshot 2026-04-07 091036.png">

---

## Part C: Open and Verify the Live Application

---

### Step 7: Open the Application URL in a New Browser Tab

1. Open a **new browser tab**
2. Paste the **ApplicationURL** you copied into the address bar
3. Press **Enter**
4. Wait a few seconds for the page to load

---

### Step 8: Confirm the Application Is Running

1. The page should load and display the **NGINX application** — a dark-themed web page with green text
2. The page will show a heading like **"Running on AWS ECS"**
3. Below the heading, you will see a list of technology stack details:
   - Container: NGINX
   - Registry: Amazon ECR Private
   - Build: AWS CodeBuild
   - Compute: AWS Fargate Serverless
   - Network: ALB to ECS Port 80
   - IaC: AWS CloudFormation
4. At the bottom of the page, you will see a pulsing green dot with the text **"Service is healthy and running"**

>  **What is this page?** This HTML page was automatically built and packaged into a Docker container by a Lambda function and CodeBuild during the stack deployment. It is now being served by an NGINX web server running inside a container on AWS ECS Fargate.

---

<img src="./Images/Screenshot 2026-04-06 111337.png">

---

## Part D: View the Resources Inside the Stack

---

### Step 9: Go Back to the CloudFormation Stack and Click "Resources"

1. Switch back to the browser tab with the **CloudFormation console**
2. Make sure you are still on the stack detail page
3. Click the **"Resources"** tab

---

### Step 10: Review the Resources List

1. A table will appear listing every AWS resource that was created by this stack
2. Each row shows:
   - **Logical ID** — the name used inside the CloudFormation template
   - **Physical ID** — the actual name of the resource as it exists in AWS (click this to go directly to that resource in the console)
   - **Type** — the type of AWS resource (e.g., `AWS::ECS::Service`, `AWS::S3::Bucket`)
   - **Status** — should show `CREATE_COMPLETE` for all rows
3. Scroll through the full list and count how many resources were created

> ℹ️ **How many resources should you see?** The stack creates around 23 resources — including networking (VPC, subnets), compute (ECS cluster, Fargate tasks), a load balancer, ECR registry, Lambda functions, CodeBuild project, and more.

---

<img src="./Images/Screenshot 2026-04-07 091212.png">

---

### Step 11: Click on the Physical ID of the ECS Service

1. In the Resources list, find the row where the **Logical ID** is `ECSService`
2. Click the **Physical ID link** in that row (it will look like a blue hyperlink)
3. This will open the **ECS Service** page directly in the AWS ECS console

---

### Step 12: Confirm the ECS Service Is Healthy

1. On the ECS Service page, look at the **Service overview** section at the top
2. Confirm you can see:
   - **Running tasks:** 1 (or more)
   - **Desired tasks:** 1 (or more)
   - **Pending tasks:** 0
3. Both **Running** and **Desired** should show the same number — this means all tasks are healthy and running

---

<img src="./Images/Screenshot 2026-04-06 111954.png">

---

## Task Completion Checklist

Before marking this task as complete, confirm you have done all of the following:

```
□ Navigated to CloudFormation from the AWS Console search bar
□ Found the nginx-app stack with status CREATE_COMPLETE
□ Opened the stack's Outputs tab
□ Located and copied the ApplicationURL value
□ Opened the ApplicationURL in a new browser tab
□ Confirmed the NGINX application page loaded successfully
□ Verified the page shows "Service is healthy and running"
□ Returned to CloudFormation and clicked the Resources tab
□ Scrolled through the full list of created resources
□ Clicked the ECSService Physical ID and confirmed Running tasks = Desired tasks
```

---

> ✅ **Your application is live and confirmed healthy.**
> Move on to **Task 3: Modify Your Application's Web Page**.