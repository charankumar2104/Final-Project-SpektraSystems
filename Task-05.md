# Task 5: Monitor Your Application Using CloudWatch Logs

**Difficulty:** Intermediate
**Estimated Time:** 20–25 Minutes
**Type:** Hands-On Monitoring (You will generate traffic to your application and then read the live logs it produces)

---

## Objective

In this task, you will **generate traffic** to your NGINX application by visiting it in the browser, and then **read the real-time logs** that your container produces in AWS CloudWatch. You will also run a **log query** to filter and count specific types of requests.

---

## Pre-Requisites

Before starting this task, make sure you have completed:

- ✅ Task 1: Accessing Your AWS Lab Account
- ✅ Task 2: Verify Your Deployed Application

Also have ready:
- The **ApplicationURL** from Task 2 (the `http://` link that opens your NGINX app)

---

## Quick Background: What Is CloudWatch?

**Amazon CloudWatch** is the monitoring and logging service in AWS. Every time a visitor loads your web page, your NGINX container writes a log entry describing that request — who visited, what page they asked for, what response was sent back, and more.

These log entries are automatically sent to CloudWatch Logs, where you can read them, search through them, and run queries to analyze them.

Think of CloudWatch Logs like a diary that your application writes in — every request gets recorded.

---

## Part A: Generate Some Traffic to Your Application

---

### Step 1: Open Your Application URL in the Browser

1. Open a browser tab and navigate to your **ApplicationURL** (the `http://` link from the CloudFormation Outputs tab in Task 2)
2. The NGINX web page should load as before

---

<img src="./Images/Screenshot 2026-04-06 111337.png">

---

### Step 2: Refresh the Page Several Times

1. Press **Ctrl + R** (Windows) or **Cmd + R** (Mac) to refresh the page
2. Refresh it **5 to 10 times** — each refresh is a new request that will be logged
3. This generates enough log entries to make the next steps more interesting

---

### Step 3: Visit a Page That Does Not Exist

1. In the browser address bar, add `/test` to the end of your ApplicationURL and press Enter
   - Example: `http://nginx-app-alb-XXXX.us-east-1.elb.amazonaws.com/test`
2. You will see a **404 Not Found** error page from NGINX — that is expected
3. This creates a log entry with a **404 status code** that you will find in CloudWatch

---

<img src="./Images/Screenshot 2026-04-06 145110.png">

---

### Step 4: Also Visit `/about` and `/admin`

1. Change the URL in the browser to add `/about` at the end → press Enter (this gives another 404)
2. Change the URL to add `/admin` at the end → press Enter (this also gives a 404)
3. These different paths will appear as separate entries in the logs

---

## Part B: Navigate to CloudWatch Logs

---

### Step 5: Open CloudWatch from the AWS Console

1. Click the **search bar** at the top of the AWS Console
2. Type: `CloudWatch`
3. Click **CloudWatch** from the dropdown results

---

<img src="./Images/Screenshot 2026-04-06 145153.png">

---

### Step 6: Click on "Log Groups" in the Left Sidebar

1. The CloudWatch console will open
2. In the **left navigation menu**, look for the section called **"Logs"**
3. Click on **"Log groups"** under the Logs section

---

<img src="./Images/Screenshot 2026-04-06 145236.png">

---

### Step 7: Find the ECS Log Group

1. On the Log Groups page, you will see a list of log groups
2. Look for the log group named **`/ecs/nginx-app`**
3. If you cannot find it, use the **search box** at the top of the page and type `/ecs/nginx-app`
4. Click on **`/ecs/nginx-app`** to open it

> **What is this log group?** This is the CloudWatch Log Group that was created by the CloudFormation stack specifically to collect all log output from your NGINX container. Every time the container prints something to the screen, it appears here.

---

<img src="./Images/Screenshot 2026-04-06 145343.png">

---

## Part C: Read the Log Streams

---

### Step 8: Open the Log Group and Find the Log Stream

1. After clicking on `/ecs/nginx-app`, you will see the **log streams** inside this log group
2. A log stream is like a separate file — one stream per running container task
3. You will see a log stream with a name that looks like: `ecs/nginx-app-container/XXXXXXXXXX`
4. Click on that log stream to open it

---

<img src="./Images/Screenshot 2026-04-06 145827.png">

---

### Step 9: Read the Log Entries

1. After clicking the log stream, you will see a list of **log events** (individual log lines)
2. Each line represents one HTTP request that was made to your application
3. The log format looks like this:

```
10.0.1.45 - - [15/Mar/2024:10:30:22 +0000] "GET / HTTP/1.1" 200 1245 "-" "Mozilla/5.0..."
```

4. Read through the columns in each log line:
   - **IP address** at the start — where the request came from
   - **Date and time** — when the request happened
   - **"GET /"** — what page was requested (the method and path)
   - **200** or **404** — the response code (200 = success, 404 = page not found)
   - **Number** after the code — how many bytes were sent back

---

### Step 10: Find Your 404 Log Entries

1. Scroll through the log entries and look for lines that contain **`404`** in them
2. These are the requests you made to `/test`, `/about`, and `/admin` in Steps 3 and 4
3. You should be able to see the exact path that was requested in each 404 entry

>  **Interesting observation:** You will also see many log entries from **the AWS Load Balancer** doing its health checks. These requests come from internal AWS IP addresses and always request the path `/`. Their user agent contains the text `ELB-HealthChecker/2.0`.

---

<img src="./Images/Screenshot 2026-04-06 150007.png">

---

## Task Completion Checklist

Before marking this task as complete, confirm you have done all of the following:

```
□ Opened the ApplicationURL in the browser and refreshed it 5–10 times
□ Visited /test, /about, and /admin paths (generating 404 errors)
□ Navigated to CloudWatch using the AWS Console search bar
□ Opened Log Groups in the left sidebar
□ Found and opened the /ecs/nginx-app log group
□ Opened the log stream inside the log group
□ Read through the raw log entries and identified the 404 entries
```

---

> ✅ **You have successfully monitored your application's traffic using CloudWatch Logs and Logs Insights.**
> Move on to **Task 6: Explore the Build Pipeline — CodeBuild and ECR**.