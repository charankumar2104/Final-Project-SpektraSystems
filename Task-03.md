# Task 3: Modify Your Application's Web Page and Redeploy

**Difficulty:** Beginner–Intermediate
**Estimated Time:** 25–35 Minutes
**Type:** Hands-On Change + Revert (You will edit the CloudFormation template, redeploy, verify the change, then restore the original)

---

## Objective

In this task, you will **edit the CloudFormation template** to change the text displayed on your NGINX web page. After saving the change, you will **update the stack** and watch AWS automatically rebuild and redeploy the application for you. At the end, you will **revert the change** to restore the original page content.

---

## Pre-Requisites

Before starting this task, make sure you have completed:

- ✅ Task 1: Accessing Your AWS Lab Account
- ✅ Task 2: Verify Your Deployed Application

Also have ready from your **Environment Details Page**:

| Item | Where to Find It |
|---|---|
| AWS Console Login URL | Environment Details Page |
| Username | Environment Details Page |

---

## Quick Background: What Will Happen When You Update the Stack?

When you make a change to the CloudFormation template and update the stack, here is what happens automatically — you do not need to do any of these manually:

1. **CloudFormation detects the change** in the Lambda function code that generates the web page
2. **A Lambda function re-runs** and uploads a new version of the application files to S3
3. **CodeBuild automatically rebuilds** the Docker image with your new HTML page inside it
4. **The new image is pushed** to the ECR (container registry)
5. **ECS stops the old container** and starts a new one with the updated image
6. **Your web page now shows** the new content

All of this happens automatically. You only need to make the edit in the template and click Update.

---

## Part A: Open the CloudFormation Template Editor

---

### Step 1: Navigate to CloudFormation

1. Make sure you are logged in to the **AWS Management Console**
2. Click the **search bar** at the top and type: `CloudFormation`
3. Click **CloudFormation** from the results

---

<img src="./Images/Screenshot 2026-04-06 112444.png">

---

### Step 2: Open the Stack

1. On the CloudFormation Stacks page, click the stack name **`nginx-app`**
2. The stack detail page will open

---

<img src="./Images/Screenshot 2026-04-06 112352.png">

---

### Step 3: Click "Update stack"

1. On the stack detail page, look at the top right area — you will see several buttons
2. Click the **"Update stack"** button
3. Click on the make direct Update button

---

<img src="./Images/Screenshot 2026-04-06 112633.png">

---

### Step 4: Choose "Edit in Infrastructure Composer"

1. A page titled **"Update stack"** will open with a few options under **"Prerequisite — Prepare template"**
2. Select the option **"Edit in Infrastructure composer"**
3. Click the **"View in Designer"** button

> ℹ️ **What is the Template?** The CloudFormation Designer is a visual and text editor built into the AWS console. It lets you view and edit the CloudFormation template directly in your browser without downloading any files.

---

<img src="./Images/Screenshot 2026-04-06 112933.png">

---

### Step 5: The Template Designer Opens

1. After clicking **View in Designer**, the **CloudFormation Designer** will open in a new view
2. You will see two sections:
   - **Top half:** A visual diagram showing all the resources and their connections
   - **Bottom half:** The raw template text (YAML format)
3. The template text at the bottom is what you need to edit

---

<img src="./Images/Screenshot 2026-04-06 113821.png">

---

## Part B: Find the Text You Will Change

---

### Step 6: Look at the Bottom Panel — the YAML Editor

1. Click anywhere in the **bottom panel** where the template text is shown
2. This is the YAML template — it is a long file with many sections
3. You need to find a specific line to change

---

### Step 7: Use the Find Function to Locate the Heading Text

1. Click inside the bottom YAML editor panel to make sure it is active
2. Press **Ctrl + F** (Windows) or **Cmd + F** (Mac) on your keyboard to open the search/find function
3. A small search box will appear inside the editor
4. Type the following text exactly as shown into the search box:

```
Deployed Successfully
```

5. Press **Enter** — the editor will jump to and highlight this line in the template

>  **What are you looking at?** This is a line inside the Python code of a Lambda function that is embedded inside the CloudFormation template. This Lambda function generates the HTML for your web page. By changing this line, you are changing the heading that appears on the live web page.

---

<img src="./Images/Screenshot 2026-04-06 114503.png">

---

## Part C: Make the Change

---

### Step 8: Edit the Heading Text

1. In the YAML editor, find the highlighted line that looks like this:

```python
"    <h1>Running on <span>AWS ECS</span></h1>"
```

2. Click at the end of `AWS ECS` — place your cursor right before `</span>`
3. Delete the text `AWS ECS` using the **Backspace** key
4. Type your own text to replace it — for example, type your **name** or your **team name**

For example, change it to:

```python
"    <h1>Running on <span>My First ECS Deployment</span></h1>"
```

> ⚠️ **Important:** Be very careful when editing. Only change the words `AWS ECS` inside the `<span>` tags. Do not delete the quotes, the `\n`, or any other punctuation around it. The template is sensitive to spacing and special characters.

---

<img src="./Images/Screenshot 2026-04-06 115052.png">

---

### Step 9: Also Change the Badge Text (Optional but Recommended)

1. Use **Ctrl + F** again to search for:

```
Deployed Successfully
```

2. Find the line that looks like:

```python
'"    <div class=\'badge\'>Deployed Successfully</div>\n"'
```

3. Change `Deployed Successfully` to something like `My Lab Deployment` or your own text

---
<img src="./Images/Screenshot 2026-04-06 115226.png">
---

## Part D: Save and Deploy the Change

---

### Step 10: Save the Template in the Designer

1. Look at the **top toolbar** of the CloudFormation Designer
2. Find the **Update Template** — it looks like a floppy disk  or there may be a button labelled **"Update template"**
3. Click it to save the template changes you just made

---

<img src="./Images/Screenshot 2026-04-06 115303.png">

---

### Step 11: Click the "Create Stack" / Upload Button in the Designer Toolbar

1. In the Designer toolbar, look for a button that looks like a **cloud with an up arrow** or is labelled **"confirm and continue to cloudformation"**
2. Click it — this will take you out of the Designer and back to the CloudFormation update wizard with your new template loaded

---

<img src="./Images/Screenshot 2026-04-06 115440.png">

---

### Step 12: Click Through Step 2 (Specify Stack Details)

1. You are on the **"Specify stack details"** step
2. You will see the **Parameters** section — there is one parameter called `AppName` with a value of `nginx-app`
3. Do **not** change this value
4. Change the Value of **Deployment Version** From **"1"** to **"2"**
4. Click **"Next"** at the bottom right

---

<img src="./Images/Screenshot 2026-04-06 125540.png">

---

### Step 13: Click Through Step 3 (Configure Stack Options)

1. You are on the **"Configure stack options"** step
2. You do not need to change anything on this page
3. Scroll down to the bottom and click **"Next"**

---

### Step 14: Review and Submit the Update

1. You are now on the **"Review and update"** step — this is a summary of all changes
2. Scroll down to the section labelled **"Change set preview"** — this shows exactly what CloudFormation will change
3. You should see entries showing that the **Lambda function** (`UploadFilesLambda`) will be updated
4. Scroll further down — look for a **checkbox** near the bottom of the page that says:

```
I acknowledge that AWS CloudFormation might create IAM resources.
```

5. Check this checkbox by clicking it

---

### Step 16: Click "Submit"

1. After checking the IAM acknowledgement checkbox, scroll to the very bottom of the page
2. Click the orange **"Submit"** button
3. You will be taken back to the stack detail page

---

## Part E: Watch the Update Progress

---

### Step 17: Open the Events Tab to Watch the Progress

1. On the stack detail page, click the **"Events"** tab
2. The Events tab shows a live log of everything CloudFormation is doing right now
3. Click the **refresh button** (circular arrow icon) every 30 seconds to see new events appear
4. You will see entries like:
   - `UploadFilesToS3 — UPDATE_IN_PROGRESS` (Lambda is uploading the new files to S3)
   - `WaitForBuild — UPDATE_IN_PROGRESS` (CodeBuild is rebuilding the Docker image — this can take 3–5 minutes)
   - `ECSService — UPDATE_IN_PROGRESS` (ECS is replacing the old container with the new one)
   - `nginx-app — UPDATE_COMPLETE` (the entire update is done)

> ⚠️ **The update will take approximately 5–8 minutes.** Most of this time is CodeBuild rebuilding and pushing the Docker image. Do not navigate away or click Submit again.

---

<img src="./Images/Screenshot 2026-04-06 115827.png">

---

### Step 18: Wait for UPDATE_COMPLETE

1. Keep refreshing the Events tab every 30–60 seconds
2. When you see the very top event shows the stack name (e.g., `nginx-app`) with status **`UPDATE_COMPLETE`**, the update is finished
3. The stack status badge at the top of the page will also change from `UPDATE_IN_PROGRESS` to **`UPDATE_COMPLETE`** in green

---
<img src="./Images/Screenshot 2026-04-06 115827.png">

---

## Part F: Verify Your Change Is Live

---

### Step 19: Refresh the Application URL in Your Browser

1. Go back to the browser tab where you had the NGINX application open
2. Press **Ctrl + R** (Windows) or **Cmd + R** (Mac) to hard-refresh the page
3. The page will reload

---

### Step 20: Confirm the New Text Appears

1. Look at the heading on the page — it should now show **your new text** instead of "AWS ECS"
2. If you also changed the badge text, check that the badge shows your new text at the top of the card

---

<img src="./Images/Screenshot 2026-04-06 125742.png">

---

## Task Completion Checklist

Before marking this task as complete, confirm you have done all of the following:

```
□ Opened the CloudFormation stack and clicked Update
□ Selected "Edit template in designer" and opened the Designer
□ Used Ctrl+F to find the heading text in the YAML editor
□ Changed "AWS ECS" to your own text
□ Optionally also changed the badge text
□ Saved the template in the Designer
□ Clicked through the Update wizard and submitted the update
□ Watched the Events tab and waited for UPDATE_COMPLETE
□ Refreshed the application URL and confirmed the new text appeared
```

---

> ✅ **You have successfully changed your application's web page through CloudFormation.**
> Move on to **Task 4: Scale Your Application to Handle More Traffic**.