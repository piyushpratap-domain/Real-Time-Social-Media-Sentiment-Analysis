# Social Media Sentiment Analysis Dashboard

This project is a full-stack, cloud-based sentiment analysis system that collects real-time tweets based on user queries, processes them using an automated AWS data pipeline, and visualizes insights via a user-friendly Streamlit dashboard.

---

## Features

-  **Live Tweet Search** by topic or hashtag
-  **Sentiment Breakdown** (Positive, Negative, Neutral)
-  **Visualizations** via interactive dashboards (Pie chart, metrics, tweet cards)
-  **Automated Data Pipeline** triggered on new data arrival
-  **NLP-based Sentiment Analysis** using VADER in AWS Glue PySpark
-  **Serverless Deployment** using AWS Lambda, EventBridge, S3, Glue
-  **Offline JSON Upload Support** for local sentiment analysis

---

##  Problem Statement

Social media platforms like Twitter are a goldmine for public sentiment and opinion. This project aims to:
- Automate the collection and analysis of tweets
- Extract sentiment in near real-time
- Provide an accessible interface for users to visualize social trends, opinions, and behaviors

---

##  Tech Stack

| Layer               | Tools Used                                                                 |
|--------------------|------------------------------------------------------------------------------|
| **Frontend**        | Streamlit, Plotly                                                           |
| **Backend**         | Python, AWS Lambda, Twitter API v2                                          |
| **Data Pipeline**   | AWS Kinesis Firehose, AWS Lambda (Consumer), S3                             |
| **ETL & Processing**| AWS Glue (PySpark), EventBridge (Trigger), VADER Sentiment Analyzer         |
| **Data Storage**    | Amazon S3 (raw/processed tweet storage)                                     |
| **Others**          | CloudWatch Logs, IAM Roles, Boto3                                           |

---

##  Pipeline Architecture

```text
  [User Input - Streamlit UI]
              ↓
    [Lambda Function (Tweet Collector)]
              ↓
       [Kinesis Data Stream]
              ↓
 [Lambda Consumer → store raw tweets in S3]
              ↓
 [EventBridge triggers AWS Glue Job]
              ↓
 [Glue (PySpark + VADER) → Save processed JSON to S3]
              ↓
  [Streamlit reads processed S3 data for dashboard display]
