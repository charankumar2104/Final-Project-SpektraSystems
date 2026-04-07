# Task 6: Explore the Build Pipeline — CodeBuild and ECR

**Difficulty:** Intermediate
**Estimated Time:** 20–25 Minutes
**Type:** Exploration + Manual Trigger (You will explore the build pipeline, manually trigger a new build, and trace the Docker image from source to registry)

---

## Objective

In this task, you will **explore the AWS CodeBuild project** that automatically built your Docker image, **read the build logs** to understand every step that happened, and then **explore the ECR (Elastic Container Registry)** where the finished Docker image is stored. You will also manually trigger a new build and watch it run from start to finish.

---

## Pre-Requisites

Before starting this task, make sure you have completed:

- ✅ Task 1: Accessing Your AWS Lab Account
- ✅ Task 2: Verify Your Deployed Application

---

## Quick Background: How Was the Docker Image Built?

When the CloudFormation stack was deployed, here is what happened behind the scenes:

1. A **Lambda function** wrote a `Dockerfile` and an `index.html` file into a ZIP file and uploaded it to an **S3 bucket**
2. **AWS CodeBuild** downloaded that ZIP file, ran `docker build` to create a Docker image, and then pushed (uploaded) that image to **Amazon ECR** (your private Docker image registry)
3. **Amazon ECS** then pulled (downloaded) the image from ECR and started running it as a container

In this task, you will explore Steps 2 and 3 of this process — CodeBuild and ECR.

---

## Part A: Explore the CodeBuild Project

---

### Step 1: Navigate to CodeBuild

1. Make sure you are logged in to the **AWS Management Console**
2. Click the **search bar** at the top and type: `CodeBuild`
3. Click **CodeBuild** from the dropdown results

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the search bar with "CodeBuild" typed in and CodeBuild appearing in the dropdown*

```
[ SCREENSHOT: AWS Console search bar showing "CodeBuild" typed in and CodeBuild in the dropdown results ]
```

---

### Step 2: The CodeBuild Dashboard Opens

1. The **CodeBuild console** will open
2. You will land on the **Build projects** page
3. You should see one project listed: **`nginx-app-build`**
4. The **Status** column may show the last build result — look for `Succeeded` in green

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the CodeBuild Build projects page showing nginx-app-build in the list*

```
[ SCREENSHOT: CodeBuild Build projects page showing nginx-app-build listed with its last build status ]
```

---

### Step 3: Click on the Project Name to Open It

1. Click on the project name **`nginx-app-build`**
2. The project detail page will open
3. You will see sections including:
   - **Project configuration** — details about how the build is set up
   - **Source** — where CodeBuild gets the code from (your S3 bucket)
   - **Environment** — what kind of computer runs the build
   - **Build history** — a list of all past builds

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the CodeBuild project detail page for nginx-app-build*

```
[ SCREENSHOT: CodeBuild project detail page for nginx-app-build showing source, environment, and build history sections ]
```

---

### Step 4: Look at the "Source" Section

1. On the project detail page, find the **Source** section
2. It will show:
   - **Source provider:** Amazon S3
   - **Bucket:** The name of your S3 bucket (e.g., `nginx-app-source-XXXXXXXXXXXX-us-east-1`)
   - **S3 object key:** `app.zip`
3. This tells you that CodeBuild downloads `app.zip` from your S3 bucket every time it builds

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the Source section of the CodeBuild project showing S3 as the source and the bucket name*

```
[ SCREENSHOT: CodeBuild project detail page Source section showing Amazon S3 as the source provider and the bucket name ]
```

---

### Step 5: Look at the "Environment" Section

1. Find the **Environment** section on the same page
2. It will show:
   - **Environment image:** `aws/codebuild/standard:7.0` (a pre-built build environment from AWS)
   - **Compute:** `BUILD_GENERAL1_SMALL` (a small compute size, enough for our image)
   - **Privileged mode:** `Enabled` — this is required for running Docker commands inside CodeBuild

> ℹ️ **Why is Privileged mode needed?** Building a Docker image requires running the Docker engine. By default, containers (including the CodeBuild container) cannot run Docker inside themselves for security reasons. Privileged mode gives CodeBuild the extra permission needed to do this.

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the Environment section showing the image, compute type, and privileged mode settings*

```
[ SCREENSHOT: CodeBuild project Environment section showing the build image, compute type, and Privileged mode enabled ]
```

---

## Part B: Read the Most Recent Build Logs

---

### Step 6: Click "Build history" to See Past Builds

1. Scroll down to the **Build history** section on the project page
2. You will see a list of past build runs, each with a **Build ID**, **Status**, **Duration**, and **When** it ran
3. Click on the most recent build ID (the top row in the list)

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the Build history section showing the list of past builds*

```
[ SCREENSHOT: CodeBuild Build history section showing a list of past builds with their Build IDs and statuses ]
```

---

### Step 7: The Build Detail Page Opens

1. The build detail page opens
2. You will see three main sections at the top:
   - **Build status** — Succeeded or Failed
   - **Build duration** — how long the build took in total
   - **Start time / End time**

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the CodeBuild build detail page showing the build status and duration at the top*

```
[ SCREENSHOT: CodeBuild build detail page showing Build status = Succeeded and the start time and duration ]
```

---

### Step 8: Expand the Build Logs Section

1. Scroll down on the build detail page
2. Find the **"Build logs"** section
3. The logs may be collapsed — click on **"Tail logs"** or expand the section to see the full log output
4. You will see all the output that CodeBuild printed during the build

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the Build logs section showing the log output from the build*

```
[ SCREENSHOT: CodeBuild build detail page Build logs section showing the console output from the build ]
```

---

### Step 9: Read Through the Build Log Phases

1. Scroll through the build log and find the three phases: `PRE_BUILD`, `BUILD`, and `POST_BUILD`
2. Identify the following lines in the log:

   **In PRE_BUILD:**
   - A line starting with `Logging in to Amazon ECR...`
   - A line showing `Login Succeeded`
   - A line showing `Version tag is YYYYMMDDHHMMSS` (the timestamp tag for the image)

   **In BUILD:**
   - `Building Docker image` — this is where the Dockerfile is executed
   - Lines showing each Docker build step (FROM nginx, RUN chmod, COPY, etc.)
   - A line showing `Successfully built...` — the image is ready

   **In POST_BUILD:**
   - `Pushing image to ECR` — the image is being uploaded
   - Lines showing `docker push` commands being executed
   - `Build complete` at the very end

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the build log showing the PRE_BUILD phase with the ECR login and version tag lines*

```
[ SCREENSHOT: CodeBuild build log showing the PRE_BUILD phase with "Logging in to Amazon ECR" and "Version tag" lines visible ]
```

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the build log showing the POST_BUILD phase with the docker push commands and "Build complete" line*

```
[ SCREENSHOT: CodeBuild build log showing the POST_BUILD phase with docker push commands and "Build complete" message ]
```

---

## Part C: Explore the ECR Repository

---

### Step 10: Navigate to ECR

1. Click the **search bar** at the top of the AWS Console
2. Type: `ECR`
3. Click **Elastic Container Registry** from the dropdown results

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the search bar showing "ECR" typed in with Elastic Container Registry in the dropdown*

```
[ SCREENSHOT: AWS Console search bar with "ECR" typed in and Elastic Container Registry in the dropdown ]
```

---

### Step 11: Open the ECR Repository

1. The ECR console will open showing **Private repositories**
2. You should see one repository: **`nginx-app-repo`**
3. Click on **`nginx-app-repo`** to open it

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the ECR Private repositories page showing nginx-app-repo*

```
[ SCREENSHOT: ECR Private repositories page showing nginx-app-repo listed ]
```

---

### Step 12: View the Docker Images in the Repository

1. Inside the repository, you will see the **Images** tab showing all Docker images that have been pushed here
2. You should see **two image tags** listed:
   - **`latest`** — the most recently pushed image, always updated with each build
   - **A timestamp tag** like `20240315120000` — a specific version of the image from the exact time it was built
3. Both tags point to the **same image** (same Image digest/SHA256 hash)

> ℹ️ **Why two tags?** The `latest` tag is always updated — it always points to the newest image. The timestamp tag is a permanent record — it never moves or gets updated. In production, it is best practice to always keep a versioned tag so you can roll back to a previous version if needed.

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the ECR repository Images tab showing both the "latest" tag and the timestamp tag*

```
[ SCREENSHOT: ECR nginx-app-repo Images tab showing two tags: "latest" and a timestamp tag (e.g., 20240315120000) ]
```

---

### Step 13: Note the Image Digest

1. Look at the **Image digest** column in the images table
2. Both the `latest` tag and the timestamp tag show the **same SHA256 digest** (a long string starting with `sha256:`)
3. This hash is the unique fingerprint of the image — it is computed from the image's exact content

> ℹ️ **Why does the digest matter?** If someone accidentally pushes a broken image with the `:latest` tag, the old timestamp tag still exists and has a different digest. You can always roll back by telling ECS to use the old digest instead of `:latest`.

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the ECR Images table showing that both tags share the same Image digest value*

```
[ SCREENSHOT: ECR Images table showing both "latest" and timestamp tags with the same Image digest SHA256 hash ]
```

---

## Part D: Manually Start a New Build and Watch It Run

---

### Step 14: Go Back to CodeBuild

1. Click the **search bar** and type `CodeBuild` → click it
2. Click on the project **`nginx-app-build`**

---

### Step 15: Click "Start build"

1. On the project detail page, look for the **"Start build"** button in the top right area
2. Click **"Start build"**
3. A dropdown may appear with options — select **"Start build"** (not "Start build with overrides")

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the CodeBuild project page with the "Start build" button visible*

```
[ SCREENSHOT: CodeBuild project nginx-app-build detail page with the Start build button highlighted ]
```

---

### Step 16: Watch the Build Run in Real Time

1. After clicking Start build, you will be taken to the **build detail page** for this new build
2. The status will show **"In progress"**
3. Scroll down to the **"Build logs"** section
4. Click **"Tail logs"** — the logs will update in real time as the build runs
5. Watch the phases appear one by one: `PROVISIONING` → `DOWNLOAD_SOURCE` → `PRE_BUILD` → `BUILD` → `POST_BUILD` → `FINALIZING`

> ℹ️ **How long will this take?** Approximately 3–5 minutes. Most of the time is spent in the `BUILD` phase where Docker downloads the nginx base image and builds your custom image on top of it.

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the CodeBuild build detail page showing the build "In progress" with the live logs updating*

```
[ SCREENSHOT: CodeBuild build detail page showing build status "In progress" with live log output appearing in the Build logs section ]
```

---

### Step 17: Wait for "Succeeded" Status

1. Keep watching the logs
2. When the build finishes, the status at the top will change to **"Succeeded"** in green
3. The last line in the logs should say **"Build complete"**

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the CodeBuild build detail page showing the final status "Succeeded" with "Build complete" in the logs*

```
[ SCREENSHOT: CodeBuild build detail page showing Build status = Succeeded and "Build complete" visible in the build logs ]
```

---

### Step 18: Go Back to ECR and Verify a New Image Was Pushed

1. Navigate back to **ECR** → **`nginx-app-repo`** → **Images** tab
2. You should now see a **new timestamp tag** was added — this is the image that was just built
3. The **`latest`** tag now points to this new image (it has the same digest as the newest timestamp tag)
4. The old timestamp tag from the previous build is also still there

> ℹ️ **What you are seeing:** ECR keeps all previous image versions unless you manually delete them. In production, companies set up **lifecycle policies** to automatically delete images older than a certain number of days to manage storage costs.

---

> 📸 **Screenshot Placeholder**
> *Take a screenshot of the ECR Images tab showing the new timestamp tag alongside the "latest" tag after the manual build*

```
[ SCREENSHOT: ECR nginx-app-repo Images tab showing a new timestamp tag added after the manual build, alongside the updated "latest" tag ]
```

---

## Task Completion Checklist

Before marking this task as complete, confirm you have done all of the following:

```
□ Navigated to CodeBuild and opened the nginx-app-build project
□ Read the Source section — confirmed S3 as the source and identified the bucket name
□ Read the Environment section — noted privileged mode is enabled
□ Opened the most recent build from Build history
□ Read through the build logs and identified the PRE_BUILD, BUILD, and POST_BUILD phases
□ Found the "Login Succeeded" line in PRE_BUILD
□ Found the "Successfully built" line in BUILD
□ Found the "Build complete" line in POST_BUILD
□ Navigated to ECR and opened nginx-app-repo
□ Verified two image tags exist: latest and a timestamp tag
□ Confirmed both tags share the same image digest
□ Went back to CodeBuild and clicked Start build to manually trigger a new build
□ Watched the live build logs and waited for Succeeded status
□ Returned to ECR and confirmed a new timestamp tag was added after the manual build
```

---

> ✅ **You have successfully explored the full build pipeline from CodeBuild to ECR.**
> You have now completed all 6 tasks in this lab. Well done!