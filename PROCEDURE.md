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

## 3. IAM Role Policies And Code For The Lambda Functions

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
```

### Producer Lambda Code

```python
import json
import boto3
import urllib.request
import urllib.parse
import re
import os
from datetime import datetime

# Initialize Kinesis
kinesis = boto3.client('kinesis')
KINESIS_STREAM_NAME = 'data-stream'

# Twitter API Settings
BEARER_TOKEN = os.environ['TWITTER_BEARER_TOKEN']
TWITTER_URL = 'https://api.twitter.com/2/tweets/search/recent'
MAX_RESULTS = 10  # Adjust as needed

# Get headers for API
def get_headers():
    return {
        'Authorization': f'Bearer {BEARER_TOKEN}'
    }

# Clean tweet text
def clean_text(text):
    text = re.sub(r"http\S+|@\S+|#\S+", "", text)
    return text.strip().lower()

# Convert tweet to structured format
def parse_and_clean(tweet):
    return {
        "tweet_id": tweet.get("id", ""),
        "timestamp": tweet.get("created_at", datetime.utcnow().isoformat()),
        "tweet": clean_text(tweet.get("text", "")),
        "language": tweet.get("lang", "und"),
        "source": "twitter",
        "username": tweet.get("author_id", "unknown_user")  # Add author_id as username
    }

# Fetch tweets from Twitter API
def fetch_tweets(query):
    params = {
        'query': f'{query} lang:en -is:retweet',
        'max_results': MAX_RESULTS,
        'tweet.fields': 'created_at,lang,author_id'  # Include author_id
    }

    url = f"{TWITTER_URL}?{urllib.parse.urlencode(params)}"
    req = urllib.request.Request(url, headers=get_headers())

    try:
        with urllib.request.urlopen(req) as response:
            response_data = response.read()
            return json.loads(response_data).get('data', [])
    except Exception as e:
        print(f"Error fetching tweets: {e}")
        return []

# Lambda handler
def lambda_handler(event, context):
    try:
        body = event.get('body')
        if body is None:
            return {'statusCode': 400, 'body': 'Missing request body.'}

        # Parse query
        payload = json.loads(body) if isinstance(body, str) else body
        query = payload.get('query')

        if not query:
            return {'statusCode': 400, 'body': 'Missing "query" parameter in request body.'}

        tweets = fetch_tweets(query)

        # Push tweets to Kinesis
        for tweet in tweets:
            cleaned = parse_and_clean(tweet)
            kinesis.put_record(
                StreamName=KINESIS_STREAM_NAME,
                Data=json.dumps(cleaned).encode('utf-8'),
                PartitionKey=cleaned['tweet_id']
            )

        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': f'Successfully sent {len(tweets)} tweets to Kinesis.',
                'topic': query
            })
        }

    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }

```


### Consumer Lambda Role Policy

Attach this policy to your Consumer Lambda execution role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": [
        "arn:aws:s3:::sm-twitter-data-bucket/*",
        "arn:aws:logs:<region>:<account-id>:log-group:/aws/lambda/consumer-lambda:*"
      ]
    }
  ]
}

```
### Consumer Lambda Code

```python
import json
import base64
import boto3
from datetime import datetime

s3 = boto3.client('s3')
BUCKET_NAME = 'sm-twitter-data-bucket'
FOLDER_NAME = 'twitter-data/raw'  # or any folder you prefer

def lambda_handler(event, context):
    processed = []

    for record in event['Records']:
        # Decode base64
        payload = base64.b64decode(record['kinesis']['data'])
        
        try:
            data = json.loads(payload)
            processed.append(data)
        except json.JSONDecodeError as e:
            print(f"Failed to parse record: {payload}")
            continue

    # Write to S3 if we have data
    if processed:
        filename = f"{FOLDER_NAME}/tweets_{datetime.utcnow().strftime('%Y%m%dT%H%M%SZ')}.json"
        s3.put_object(
            Bucket=BUCKET_NAME,
            Key=filename,
            Body=json.dumps(processed, indent=2),
            ContentType='application/json'
        )

    return {
        'statusCode': 200,
        'body': f'Successfully processed {len(processed)} records.'
    }

```
---

## 4. Configure Lambda Triggers

### Producer Lambda Trigger

- Trigger the **Producer Lambda** using an API Gateway HTTP POST request or any other event source.
- The POST request body should be JSON with a `"query"` key specifying the Twitter search term.

Example request body:

```json
{
  "query": "your_topic_here"
}
```
---

## 5. Create Twitter Developer Account & Get Bearer Token

- To access Twitter’s API, you must create a **Twitter Developer Account**:
  - Visit https://developer.twitter.com/ and apply for a developer account.
  - Once approved, create a project and generate your **Bearer Token**.
- Set the Bearer Token as an environment variable (`TWITTER_BEARER_TOKEN`) in your Producer Lambda function to authenticate API requests.

---

## 6. AWS Glue Job for Sentiment Analysis

- Create an **AWS Glue ETL job** to process raw tweets stored in S3 and perform sentiment classification.
- Use **PySpark** with NLP libraries such as `TextBlob` or `NLTK` to analyze tweet text and assign sentiment labels (positive, negative, neutral).
- Configure the Glue job to trigger via **AWS EventBridge** when new tweet data files arrive in the S3 bucket.
- Store the enriched, sentiment-labeled data back into S3 for downstream consumption.

### Code I Used For Analysis

```python
from awsglue.context import GlueContext
from awsglue.utils import getResolvedOptions
from awsglue.job import Job
from pyspark.context import SparkContext
from pyspark.sql.functions import udf, col
from pyspark.sql.types import StringType
import sys
import boto3
import os
import json
from datetime import datetime
# ------------------------
# Glue job setup
# ------------------------
args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# ------------------------
# Download VADER lexicon manually from S3 to /tmp
# ------------------------
s3 = boto3.client('s3')
bucket = "sm-twitter-data-bucket"
key = "vader/vader_lexicon.txt"
lexicon_path = "/tmp/vader_lexicon.txt"

response = s3.get_object(Bucket=bucket, Key=key)
with open(lexicon_path, "w", encoding='utf-8') as f:
    f.write(response['Body'].read().decode('utf-8'))

# ------------------------
# Load VADER Lexicon
# ------------------------
def load_vader_lexicon(path):
    lex_dict = {}
    with open(path, 'r', encoding='utf-8') as f:
        for line in f:
            if not line.strip(): continue
            tokens = line.strip().split('\t')
            if len(tokens) >= 2:
                word, measure = tokens[0], float(tokens[1])
                lex_dict[word] = measure
    return lex_dict

vader_lexicon = load_vader_lexicon(lexicon_path)

def compute_sentiment(text):
    if not text:
        return "Neutral"
    words = text.lower().split()
    score = sum(vader_lexicon.get(word, 0.0) for word in words)
    if score >= 1.0:
        return "Positive"
    elif score <= -1.0:
        return "Negative"
    else:
        return "Neutral"

sentiment_udf = udf(compute_sentiment, StringType())

# ------------------------
# Load tweets from S3
# ------------------------
df = spark.read.json("s3://sm-twitter-data-bucket/twitter-data/raw/", multiLine=True)

# ------------------------
# Add sentiment column
# ------------------------
df_result = df.withColumn("sentiment", sentiment_udf(col("tweet")))

# ------------------------
# Convert to Pandas and build final JSON
# ------------------------
df_pd = df_result.select("username", "tweet", "timestamp", "sentiment").toPandas()
df_pd.columns = ["username", "text", "timestamp", "sentiment"]

total = len(df_pd)
positive = len(df_pd[df_pd['sentiment'] == 'Positive'])
negative = len(df_pd[df_pd['sentiment'] == 'Negative'])
neutral = len(df_pd[df_pd['sentiment'] == 'Neutral'])

# Overall sentiment
sentiment_counts = {
    "Positive": positive,
    "Negative": negative,
    "Neutral": neutral
}
overall_sentiment = max(sentiment_counts, key=sentiment_counts.get)

# All words for word cloud
words = []
df_pd["text"].str.split().apply(lambda x: words.extend([w for w in x if len(w) > 2]))

# Final output structure
result = {
    "total_tweets": total,
    "positive_pct": round(positive * 100 / total, 2),
    "negative_pct": round(negative * 100 / total, 2),
    "neutral_pct": round(neutral * 100 / total, 2),
    "overall_sentiment": overall_sentiment,
    "tweets": df_pd.to_dict(orient="records"),
    "words": words
}

# ------------------------
# Write JSON to S3
# ------------------------


output_key = f"twitter-data/processed/result_{datetime.utcnow().strftime('%Y%m%dT%H%M%SZ')}.json"


s3.put_object(
    Bucket=bucket,
    Key=output_key,
    Body=json.dumps(result, indent=2),
    ContentType="application/json"
)

job.commit()
```
---

## 7. Streamlit Frontend for Visualization

- Develop a **Streamlit application** that enables users to:
  - Enter a topic and initiate tweet fetching by calling the Producer Lambda API.
  - View real-time sentiment analysis results by loading processed data from S3.
  - Display interactive charts showing counts of positive, negative, and neutral tweets.
- This frontend provides an intuitive interface for exploring social media sentiment trends dynamically.

### Code I used For Visualization
```python
import streamlit as st
import plotly.graph_objects as go
from wordcloud import WordCloud
import pandas as pd
import matplotlib.pyplot as plt
import requests
from datetime import datetime

# ------------------ Streamlit Config ------------------ #
st.set_page_config(
    page_title="Twitter Sentiment Analysis",
    page_icon="🐦",
    layout="wide",
    initial_sidebar_state="collapsed"
)

st.markdown("""
<style>
    .main-header { font-size: 2.5rem; font-weight: 600; color: #1f2937; margin-bottom: 2rem; }
    .tweet-card { background: white; padding: 1rem; border-radius: 8px; border: 1px solid #e5e7eb; margin-bottom: 0.5rem; }
    .sentiment-tag { padding: 0.25rem 0.75rem; border-radius: 20px; font-size: 0.875rem; font-weight: 500; display: inline-block; margin-top: 0.5rem; }
    .sentiment-positive { background-color: #16a34a; color: white; }
    .sentiment-negative { background-color: #dc2626; color: white; }
    .sentiment-neutral { background-color: #6b7280; color: white; }
    .search-container { margin-bottom: 2rem; }
</style>
""", unsafe_allow_html=True)

# ------------------ Session Init ------------------ #
if 'search_performed' not in st.session_state:
    st.session_state.search_performed = False
if 'current_query' not in st.session_state:
    st.session_state.current_query = ""

# ------------------ Header ------------------ #
st.markdown('<h1 class="main-header">Twitter Sentiment Analysis</h1>', unsafe_allow_html=True)

# ------------------ Search UI ------------------ #
st.markdown('<div class="search-container">', unsafe_allow_html=True)
col1, col2 = st.columns([4, 1])
with col1:
    search_query = st.text_input("", placeholder="Search a topic or hashtag...", key="search_input", label_visibility="collapsed")
with col2:
    search_button = st.button("Search", type="primary", use_container_width=True)
st.markdown('</div>', unsafe_allow_html=True)

# ------------------ API Integration ------------------ #
def fetch_sentiment_data(query):
    API_URL = "https://fl9f3tz9si.execute-api.eu-north-1.amazonaws.com/prod/collect-tweets"
    try:
        response = requests.post(API_URL, json={"query": query}, timeout=15)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        st.error(f"API call failed: {e}")
        return None

# ------------------ Search Trigger ------------------ #
if search_button and search_query:
    st.session_state.search_performed = True
    st.session_state.current_query = search_query
    data = fetch_sentiment_data(search_query)
    if data and isinstance(data, dict) and data.get("tweets"):
        st.session_state.data = data
    else:
        st.session_state.search_performed = False
        st.warning("No data found. Try a different topic.")

# ------------------ Show Results ------------------ #
if st.session_state.search_performed:
    data = st.session_state.data
    st.subheader(f"Results for: `{st.session_state.current_query}`")

    # ---------- Summary Metrics ---------- #
    total_tweets = data.get('total_tweets', len(data.get('tweets', [])))
    positive_pct = data.get('positive_pct', 0)
    negative_pct = data.get('negative_pct', 0)
    neutral_pct = data.get('neutral_pct', 0)
    overall_sentiment = data.get('overall_sentiment', 'Neutral')
    tweets = data.get("tweets", [])

    col1, col2, col3, col4 = st.columns(4)
    with col1:
        st.metric("Total Tweets", f"{total_tweets:,}")
    with col2:
        st.metric("Positive", f"{positive_pct}%")
    with col3:
        st.metric("Negative", f"{negative_pct}%")
    with col4:
        emoji = {"Positive": "🟢", "Negative": "🔴", "Neutral": "🟡"}.get(overall_sentiment, "")
        st.metric("Overall Sentiment", f"{emoji} {overall_sentiment}")

    st.markdown("---")

    # ---------- Pie Chart ---------- #
    col1, col2 = st.columns(2)
    with col1:
        st.subheader("Sentiment Distribution")
        fig_pie = go.Figure(data=[go.Pie(
            labels=['Positive', 'Negative', 'Neutral'],
            values=[positive_pct, negative_pct, neutral_pct],
            hole=0.4,
            marker_colors=['#16a34a', '#dc2626', '#6b7280']
        )])
        st.plotly_chart(fig_pie, use_container_width=True)

    # ---------- Tweet Cards ---------- #
    st.subheader("Recent Tweets")
    for tweet in tweets:
        with st.container():
            st.write(f"**{tweet.get('username', 'Unknown User')}**")
            st.write(tweet.get("text", ""))
            tag_class = {
                "Positive": "sentiment-positive",
                "Negative": "sentiment-negative",
                "Neutral": "sentiment-neutral"
            }.get(tweet.get("sentiment", "Neutral"), "sentiment-neutral")
            st.markdown(f"<span class='sentiment-tag {tag_class}'>{tweet.get('sentiment', 'Neutral')}</span>", unsafe_allow_html=True)
            st.caption(f"{tweet.get('timestamp', '')}")
        st.markdown("---")
else:
    st.info("👆 Enter a topic or hashtag and click 'Search' to start analysis.")

# ------------------ Footer ------------------ #
st.markdown("---")
st.caption("Built with ❤️ using Streamlit | Connects to AWS Lambda + Twitter API (via snscrape)")

```
---

