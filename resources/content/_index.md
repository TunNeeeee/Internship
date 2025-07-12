---
title: "Introduction to AWS Lambda"
date: "`r Sys.Date()`"
weight: 1
chapter: false
---

# Introduction to AWS Lambda

#### Overview

In this lab, you will learn about **AWS Lambda**, one of the most powerful and widely-used serverless services in the AWS ecosystem. You will understand what Lambda is, when to use it, how it gets triggered, and how its pricing model works. This foundational knowledge will prepare you to build and deploy serverless applications effectively.

---

#### What is AWS Lambda?

**AWS Lambda** is a **serverless compute service** provided by Amazon Web Services. With Lambda, you can write your code and deploy it without worrying about provisioning or managing servers. AWS automatically handles:

- Compute resource provisioning
- Operating system and security patches
- Auto-scaling based on usage
- Monitoring and logging

Lambda is ideal for executing small, stateless units of code in response to events.

![AWS Lambda Service Overview](/images/lambda/lambda-overview.png?featherlight=false&width=90pc)

{{% notice info %}}
Lambda is **event-driven**, which means your code only runs when triggered by a specific event.
{{% /notice %}}

---

#### When Should You Use Lambda?

Use AWS Lambda in the following scenarios:

| Use Case                          | Why Lambda is a Good Fit                       |
|----------------------------------|------------------------------------------------|
| Process images after S3 upload   | Trigger automatically from S3 events          |
| Send notifications/emails        | Combine with SNS, SES, or EventBridge         |
| Backend for REST APIs            | Integrates seamlessly with API Gateway        |
| Scheduled tasks / cron jobs      | Trigger using EventBridge or CloudWatch Events|
| Event-based messaging            | Triggered by SQS, SNS, DynamoDB Streams, etc. |

---

#### How Does Lambda Get Triggered?

A Lambda function only runs when a **trigger event** occurs. These triggers come from other AWS services or external sources.

##### 🔄 Common trigger sources include:
- **Amazon S3** – on object upload/delete events
- **API Gateway** – on HTTP/REST API requests
- **DynamoDB Streams** – on table insert/update/delete
- **CloudWatch Events/EventBridge** – on scheduled rules
- **SNS/SQS** – on message received

{{% notice tip %}}
Triggers allow you to build **automated and reactive systems** without maintaining background workers or polling logic.
{{% /notice %}}

---

#### How is Lambda Priced?

Lambda pricing is based on **two primary factors**:

1. **Number of invocations** (how many times the function is called)
2. **Compute usage**, which includes memory size and execution duration

##### 🎯 Pricing formula:
```text
Cost = (execution time in milliseconds) × (allocated memory) × (number of requests)
