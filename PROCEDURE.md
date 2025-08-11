# AWS Lambda Setup for Social Media Sentiment Analysis

This guide covers the creation and configuration of two AWS Lambda functions used to fetch, stream, and store Twitter data for sentiment analysis.

---

## Prerequisites

- AWS account with permissions to create Lambda functions, Kinesis streams, S3 buckets, and IAM roles.
- Twitter API Bearer Token.

---

## 1. Create Kinesis Data Stream

- Name: `data-stream`  
- Shard count: 1 (adjust based on expected load)

---

## 2. Create S3 Bucket

- Bucket name: `sm-twitter-data-bucket`  
- Folder: `twitter-data/raw` (used for storing raw tweets)

---

## 3. IAM Role Policies

### Producer Lambda Role Policy

Attach this policy to your Producer Lambda execution role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kinesis:PutRecord",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": [
        "arn:aws:kinesis:<region>:<account-id>:stream/data-stream",
        "arn:aws:logs:<region>:<account-id>:log-group:/aws/lambda/producer-lambda:*"
      ]
    }
  ]
}
