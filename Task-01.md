# Task 1: Accessing Your AWS Lab Account

**Difficulty:** Beginner
**Estimated Time:** 10–15 Minutes
**Type:** Account Access (You will log in to your AWS account provided by the lab)

---

## Objective

In this task, you will **find your AWS lab credentials** from the CloudLabs Environment Details page and use them to **log in to the AWS Management Console**. By the end of this task, you will be inside your AWS account and ready to begin working with the resources.

---

## Pre-Requisites

Before starting this task, make sure you have:

- Access to the **CloudLabs portal** where your lab is running
- A web browser (Google Chrome or Microsoft Edge recommended)
- Your lab is in **Active** status on the CloudLabs portal

---

## Quick Background: What Is the AWS Management Console?

The **AWS Management Console** is the web-based interface you use to manage all AWS services. Think of it like the "control panel" for your cloud environment.

In this lab, AWS has already set up a fresh account for you. All the resources for the lab (like the application, the database, the network, etc.) are pre-deployed inside this account. You just need to log in and start working.

Your login credentials are **unique to you** and are provided on the CloudLabs Environment Details page.

---

## Part A: Find Your Credentials on CloudLabs

---

### Step 1: Open Your Lab on the CloudLabs Portal

1. Open your web browser
2. Go to the **CloudLabs portal** where your lab session is active
3. Find your currently running lab and click on it to open the lab page

---

<img src="./images/Screenshot 2026-04-07 090007.png">

---

### Step 2: Open the Environment Details Page

1. On your lab page, look for a tab or button labelled **"Environment Details"** or **"Lab Details"**
2. Click on it to open the details page
3. This page contains all the credentials and information you need for the lab

---

<img src="./Images/Screenshot 2026-04-07 090007.png">

---

### Step 3: Locate Your AWS Login Information

On the Environment Details page, find and note down the following information. You will need all three of these in the next steps:

| Item | Where to Find It | Example Format |
|---|---|---|
| **AWS Console Login URL** | Environment Details page | `https://XXXXXXXXXXXX.signin.aws.amazon.com/console` |
| **Username** | Environment Details page | `odl_user_XXXXXXX` |
| **Password** | Environment Details page | A combination of letters, numbers, and symbols |

> ⚠️ **Important:** Keep this page open in a separate browser tab. You may need to refer back to it during other tasks.

---

<img src="./Images/Screenshot 2026-04-07 090116.png">

---

## Part B: Log In to the AWS Management Console

---

### Step 4: Open the AWS Console Login URL

1. Open a **new browser tab**
2. Copy the **AWS Console Login URL** from the Environment Details page
3. Paste it into the address bar of your new browser tab
4. Press **Enter** to navigate to the page

> ℹ️ **What you will see:** A page titled **"Sign in as IAM user"** with your account number already filled in. This is different from the standard AWS login page — it is specific to your lab account.

---

<img src="./Images/Screenshot 2026-04-07 090236.png">

---

### Step 5: Enter Your IAM Username

1. On the sign-in page, find the field labelled **"IAM user name"**
2. Click on it and type your **Username** exactly as it appears on the Environment Details page
   - Example: `odl_user_2157974`
3. Make sure there are no extra spaces before or after the username

---

### Step 6: Enter Your Password

1. Click on the **"Password"** field below the username field
2. Type your **Password** exactly as shown on the Environment Details page
   - Passwords are case-sensitive — make sure uppercase and lowercase letters match exactly
3. Double-check for any special characters that might be easy to mistype



---

### Step 7: Click "Sign In"

1. After entering both your username and password, click the orange **"Sign in"** button
2. Wait a few seconds while AWS verifies your credentials and loads your account

> ⚠️ **If you see an error:** Go back to the Environment Details page and carefully re-copy your credentials. The most common mistake is an extra space or a mistyped special character in the password.
---

<img src="./Images/Screenshot 2026-04-07 090327.png">

---

### Step 8: Confirm You Are Logged In — The AWS Console Home Page

1. After a successful login, you will be taken to the **AWS Management Console home page**
2. You will see:
   - The **AWS logo** at the top left
   - A **search bar** at the top centre
   - Your **username or account alias** at the top right (confirming who you are logged in as)
   - A region selector showing something like **"US East (N. Virginia)"** or another region

> ℹ️ **What region should you see?** Your lab resources are deployed in a specific AWS region. Check the Environment Details page — it will mention which region is used. Make sure the region shown in the top right corner matches.

---

<img src="./Images/Screenshot 2026-04-07 090447.png">

---

## Part C: Verify the Correct Region is Selected

---

### Step 9: Check the Region in the Top Right Corner

1. Look at the **top right corner** of the AWS console — you will see a region name like **"N. Virginia"**, **"Oregon"**, or **"Mumbai"**
2. Click on it to see the full region name (e.g., `us-east-1`, `us-west-2`, `ap-south-1`)
3. Compare this with what is shown on your **Environment Details page**

---

<img src="./Images/Screenshot 2026-04-07 090522.png">

---

### Step 10: Change the Region If Needed

1. If the region shown does **not match** what is listed in your Environment Details page:
   - Click on the region name in the top right corner
   - A dropdown list of all AWS regions will appear
   - Scroll through the list and click on the **correct region** for your lab
2. The page will reload with the correct region selected
3. If the region already matches — you do not need to change anything
4. Make sure you select the region us-east-1.



---

### Step 11: Confirm the Console is Ready

After setting the correct region, your AWS console is fully ready to use. Confirm you can see:

- ✅ Your **username** displayed at the top right
- ✅ The **correct region** selected in the top right corner
- ✅ The AWS console home page is fully loaded with no errors

---

## Task Completion Checklist

Before marking this task as complete, confirm you have done all of the following:

```
□ Opened the CloudLabs portal and found the active lab
□ Opened the Environment Details page
□ Located the AWS Console Login URL, Username, and Password
□ Navigated to the AWS Console Login URL in a new browser tab
□ Entered your IAM username correctly
□ Entered your password correctly
□ Successfully signed in and reached the AWS Management Console home page
□ Verified that the correct AWS region is selected in the top right corner
□ Confirmed your username is visible in the top right corner of the console
```

---

> ✅ **You are now inside your AWS lab account and ready to begin the next task.**
> Move on to **Task 2: Verify Your Deployed Application**.