---
title : "Create Lambda Function"
date : 2025-07-01
weight : 2
chapter : false
pre : "<b> 2. </b>"
---

**Contents:**

- [Create a Lambda Function](#create-a-lambda-function)
- [Configure Lambda Function](#configure-lambda-function)
- [Write and Edit Code](#write-and-edit-code)
- [Test Lambda Function](#test-lambda-function)

---

## Create a Lambda Function

1. Go to [AWS Management Console](https://console.aws.amazon.com/)
2. Search for and select the **Lambda** service
3. On the Lambda dashboard, click **Create function**

![Create Lambda](/images/lab1/001.png)  
![Create Lambda](/images/lab1/002.png)

---

## Configure Lambda Function

On the **Create function** screen, do the following:

- Choose: **Author from scratch**
- Function name: `hello-aws-sls-2025`
- Runtime: choose `Python 3.13` or `Node.js 22.x` (your choice)
- Architecture: choose `arm64`

![Create Lambda](/images/lab1/003.png)

**Permissions:**

- Click **Change default execution role**
- Then choose one of the following:
  - `Create a new role with basic Lambda permissions`
  - or `Use an existing role` if you've already created one

![Create Lambda](/images/lab1/004.png)

> ✅ Review the configuration and click **Create function**

---

## Write and Edit Code

Once the function is created:

![Create Lambda](/images/lab1/005.png)

1. Scroll down to the **Function code** section  
2. You can edit the code directly in the inline editor

![Create Lambda](/images/lab1/006.png)

---

## Test Lambda Function

Click the **Test** button, or press `CTRL + SHIFT + I`, then choose **Create new test event**.

- Event Name: enter `test1`
- Event JSON: input `{"name":"Tun"}`

![Create Lambda](/images/lab1/007.png)  
![Create Lambda](/images/lab1/008.png)

Then click the **Test** button again to execute the function and see the result.

![Create Lambda](/images/lab1/009.png)
