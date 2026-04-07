# Task 4: Scale Your Application to Handle More Traffic → Then Scale Back Down

**Difficulty:** Beginner–Intermediate
**Estimated Time:** 20–25 Minutes
**Type:** Hands-On Change + Revert (You will scale your ECS service up to 2 tasks, observe the result, then scale back down to 1)

---

## Objective

In this task, you will **increase the number of running containers** (called "tasks") for your application from 1 to 2 by editing the CloudFormation template. You will then **verify** that two containers are running and that the load balancer is sending traffic to both of them. Finally, you will **scale back down** to 1 to restore the original state.

---

## Pre-Requisites

Before starting this task, make sure you have completed:

- ✅ Task 1: Accessing Your AWS Lab Account
- ✅ Task 2: Verify Your Deployed Application
- ✅ Task 3: Modify Your Application's Web Page

---

## Quick Background: What Is Scaling?

Right now, your application runs on **1 container**. A container is like a mini-computer running your NGINX web server. If that one container crashes, your application goes down until a replacement is started.

**Scaling** means increasing the number of containers (tasks) so that:
- If one crashes, the others keep serving traffic
- More containers can handle more visitors at the same time
- Traffic is shared ("load balanced") across all running containers

The AWS service that runs your containers is called **Amazon ECS (Elastic Container Service)**. The setting that controls how many containers should run at all times is called the **"DesiredCount"**.

You will change the `DesiredCount` from `1` to `2` in the CloudFormation template, update the stack, and observe two containers running simultaneously.

---

## Part A: Open the CloudFormation Template and Find the DesiredCount Setting

---

### Step 1: Navigate to CloudFormation and Open the Stack

1. Make sure you are logged in to the **AWS Management Console**
2. Click the **search bar** at the top and type: `CloudFormation`
3. Click **CloudFormation** from the dropdown
4. Click on the stack name **`nginx-app`**

---

<img src="./Images/Screenshot 2026-04-06 141710.png">

---

### Step 2: Click "Update"

1. On the stack detail page, click the **"Update"** button in the top right area

---

<img src="">

---

### Step 3: Open the using Template section 

1. On the **"Update stack"** page, select **"Edit template template section"**
2. Click **"View in Designer"**
3. The CloudFormation Designer will open with the YAML template in the bottom panel

---

<img src="./Images/Screenshot 2026-04-06 142555.png">
---

### Step 4: Use Find to Locate the DesiredCount Setting

1. Click inside the **bottom YAML editor panel** to make it active
2. Press **Ctrl + F** (Windows) or **Cmd + F** (Mac) to open the search function
3. Type the following into the search box:

```
DesiredCount: 1
```

4. Press **Enter** — the editor will jump to and highlight this line in the template

> ℹ️ **What are you looking at?** This line is inside the `ECSService` resource definition. It tells AWS how many copies (tasks) of your container should always be running. Currently it is set to `1`.

---

<img src="./Images/Screenshot 2026-04-06 142648.png">

---

## Part B: Change DesiredCount from 1 to 2

---

### Step 5: Edit the DesiredCount Value

1. Click directly on the number `1` in the highlighted line `DesiredCount: 1`
2. Delete the `1` using the **Backspace** key
3. Type `2` to replace it
4. The line should now read:

```yaml
      DesiredCount: 2
```

> ⚠️ **Important:** Only change the number `1` to `2`. Do not change any other characters, spacing, or indentation in the line. YAML is sensitive to indentation — extra or missing spaces will cause errors.

---

<img src="./Images/Screenshot 2026-04-06 142715.png">

---

## Part C: Save and Deploy the Change

---

### Step 6: Save the Template

1. Look at the top toolbar of the Designer
2. Click on the **Validate** button to validate the template.
2. Click the **Update Template** to save the template

---

<img src="./Images/Screenshot 2026-04-06 142746.png">

---

### Step 7: Click the Upload to Stack Button

1. In the next page click on **Next**. You will be redirected to other page where you can see the page similar to bottom image.
2. Update the Deploy Version by incrementing 1.
and click on **Next**.
---
<img src="./Images/Screenshot 2026-04-06 142906.png">
---

### Step 8: Click Through the Update Wizard

1. On the **"Specify template"** step — click **Next**
2. On the **"Specify stack details"** step — do not change anything — click **Next**
3. On the **"Configure stack options"** step — do not change anything — click **Next**
4. On the **"Review and update"** step:
   - Look at the **"Change set preview"** — it should show that the `ECSService` resource will be updated
   - Scroll down and check the **IAM acknowledgement checkbox**
   - Click **Submit**

---

<img src="./Images/Screenshot 2026-04-06 143220.png">

---
5. Check the change set then you got to know what are the resources that are being modified or added or deleted.
---
<img src="./Images/Screenshot 2026-04-06 143250.png">
---



### Step 9: Wait for the Update to Complete

1. After clicking Submit, you will be taken to the stack detail page
2. The status will show **`UPDATE_IN_PROGRESS`**
3. Click the **Events** tab and refresh every 30 seconds to watch progress
4. This update is much faster than Task 3 — ECS simply starts a second task without rebuilding anything. It should complete in about **2–4 minutes**
5. Wait until the stack shows **`UPDATE_COMPLETE`**

---

<img src="./Images/Screenshot 2026-04-06 143432.png">

---

## Part D: Verify That Two Containers Are Now Running

---

### Step 10: Navigate to the ECS Service

1. In the AWS Console, click the **search bar** at the top and type: `ECS`
2. Click **Elastic Container Service** from the dropdown
3. You will land on the ECS clusters page

---

<img src="./Images/Screenshot 2026-04-06 143541.png">

---

### Step 11: Open the Cluster

1. Click on the cluster named **`nginx-app-cluster`**
2. The cluster detail page will open

---

<img src="./Images/Screenshot 2026-04-06 143614.png">

---

### Step 12: Open the ECS Service

1. On the cluster page, click the **Services** tab
2. You will see the service named **`nginx-app-service`**
3. Click on it to open the service detail page

---

<img src="./Images/Screenshot 2026-04-06 143920.png">

---

### Step 13: Confirm Two Tasks Are Running

1. On the service detail page, look at the **Service overview** section at the top
2. Check the task counts:
   - **Desired tasks:** 2
   - **Running tasks:** 2
   - **Pending tasks:** 0
3. All three values should confirm that 2 tasks are now running

---

<img src="./Images/Screenshot 2026-04-06 144029.png">

---

## Part E: Check That the Load Balancer Sees Both Tasks

---

### Step 14: Navigate to the Load Balancer Target Group

1. Click the **search bar** at the top and type: `EC2`
2. Click **EC2** from the results
3. In the EC2 console left sidebar, scroll down and click **"Target Groups"** (under the **Load Balancing** section)

<img src="./Images/Screenshot 2026-04-06 144459.png">

---

### Step 15: Open the nginx-app Target Group

1. In the Target Groups list, find the target group named **`nginx-app-tg`**
2. Click on it to open the target group detail page

---

<img src="./Images/Screenshot 2026-04-06 144526.png">

---

### Step 16: Click the "Targets" Tab and Verify Two Healthy Targets

1. On the target group detail page, click the **"Targets"** tab
2. You will see a table listing all registered targets
3. There should now be **2 targets** listed — one for each running ECS task
4. Each target shows:
   - An **IP address** (the private IP of the ECS task)
   - A **Port** (80)
   - **Health status: healthy** (shown in green)

> ℹ️ **What does this mean?** The Application Load Balancer (ALB) now knows about both containers. It is sending incoming visitors' requests to both containers in turn — one request goes to Task 1, the next goes to Task 2, and so on. This is called **load balancing**.

---

<img src="./Images/Screenshot 2026-04-06 144708.png">

---

## Task Completion Checklist

Before marking this task as complete, confirm you have done all of the following:

```
□ Opened the CloudFormation stack and clicked Update
□ Opened the template in the Designer
□ Found the "DesiredCount: 1" line using Ctrl+F
□ Changed DesiredCount to 2
□ Saved and submitted the stack update
□ Waited for UPDATE_COMPLETE
□ Verified in ECS that Running tasks = 2 and Desired tasks = 2
□ Verified in the Tasks tab that 2 tasks are listed with RUNNING status
□ Verified in the Target Group Targets tab that 2 targets are healthy
```

---

> ✅ **You have successfully scaled your application up and then back down using CloudFormation.**
> Move on to **Task 5: Monitor Your Application Using CloudWatch Logs**.