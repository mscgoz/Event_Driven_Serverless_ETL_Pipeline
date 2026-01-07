# Event-Driven ETL Pipeline on AWS

## 1. Introduction

The goal of this project is to showcase a **serverless, scalable, and decoupled ETL architecture** using AWS managed services.

When data is uploaded to Amazon S3, an **Amazon EventBridge notification** is generated.  
EventBridge then triggers **Lambda processor functions** to transform and aggregate the data.  
The processed output is stored in a **Consumption S3 bucket** for analytics and downstream use cases.



---

## 2. AWS Architecture Diagram

The diagram below illustrates the high-level architecture of the event-driven ETL pipeline, including data ingestion, event handling, processing, and consumption layers.


```
Lambda (Backend App)
        |
        v
S3 Raw Bucket
        |
        v
Amazon EventBridge
        |
        v
Lambda (Processor)
        |
        v
S3 Consumption Bucket
        |
        v
Lambda (Promotion)
```


---

## 3. Data Dictionary

| Column Name        | Description |
|--------------------|-------------|
| `cart_id`          | Unique identifier of the shopping cart |
| `customer_id`      | Unique identifier of the customer |
| `product_id`       | Unique identifier of the abandoned product |
| `product_amount`   | Quantity of the abondene product |
| `product_price`    | Price of the product |

---

## 4. ETL Architecture Flow

The ETL pipeline follows an **event-driven workflow** as described below:

### 4.1. Data Generation & Ingestion to the Raw Bucket

This Lambda function generates **synthetic e-commerce cart data** and uploads it to an Amazon S3 bucket.  
It is used as the **data producer** in the event-driven ETL pipeline.

#### Responsibilities
- Generate fake transactional data using the Faker library
- Create a CSV file inside the Lambda `/tmp` directory
- Upload the generated file to an S3 Raw bucket

#### Environment Variables
- `input_bucket`: Target S3 bucket where generated CSV files are uploaded

#### Lambda Code
```python
from faker import Faker
import random
from faker.providers import currency

import logging
import boto3
from botocore.exceptions import ClientError
import os

import pandas as pd
from collections import defaultdict

inputBucket = os.environ['input_bucket']

def handler(event, context):
    generate_data()
    upload_file(
        file_name='/tmp/cart_abandonment_data.csv',
        bucket=inputBucket
    )

def generate_data():
    fake = Faker()
    fake.add_provider(currency)

    fake_data = defaultdict(list)

    for _ in range(1000):
        fake_data["cart_id"].append(random.randint(0, 10))
        fake_data["customer_id"].append(random.randint(0, 10))
        fake_data["product_id"].append(random.randint(0, 10))
        fake_data["product_amount"].append(random.randint(1, 20))
        fake_data["product_price"].append(fake.pricetag())

    df_fake_data = pd.DataFrame(fake_data)
    df_fake_data.to_csv("/tmp/cart_abandonment_data.csv", index=False)

def upload_file(file_name, bucket, object_name=None):
    if object_name is None:
        object_name = os.path.basename(file_name)

    s3_client = boto3.client('s3')
    try:
        s3_client.upload_file(file_name, bucket, object_name)
    except ClientError as e:
        logging.error(e)
        return False
    return True
```


### 4.2. Event Notification  
The Raw S3 bucket sends **object creation notifications** to **Amazon EventBridge**.

### 4.3. Event Processing Trigger  
Amazon EventBridge evaluates incoming events and **triggers a Lambda processor function** based on predefined event patterns.

![EventBridge Pattern](eventbridgepattern.png)

### 4.4. Data Processing and Aggregation Then Upload to the Consumption Bucket

This Lambda function is responsible for **processing raw cart data** stored in Amazon S3.  
It performs aggregation logic on transactional data and writes the transformed output to a **Consumption S3 bucket**.

This function represents the **Transform (T)** stage of the ETL pipeline.

---

#### Responsibilities
- Download raw CSV data from the S3 Raw bucket
- Aggregate cart data at the product level
- Generate a summarized CSV file for analytics
- Upload aggregated results to the Consumption bucket

---

#### Environment Variables
- `input_bucket`: S3 bucket containing raw cart data
- `output_bucket`: S3 bucket for aggregated results

---

#### Lambda Code

```python
import os
import logging
import boto3
from botocore.exceptions import ClientError

import pandas as pd
from collections import defaultdict

inputBucket = os.environ['input_bucket']
outputBucket = os.environ['output_bucket']

def handler(event, context):
    download_file(file_name='cart_abandonment_data.csv', bucket=inputBucket)
    process_file(file_name='cart_abandonment_data.csv')
    upload_file(
        file_name='/tmp/cart_aggregated_data.csv',
        bucket=outputBucket
    )

def download_file(file_name, bucket, object_name=None):
    s3 = boto3.client('s3')
    s3.download_file(bucket, file_name, '/tmp/' + file_name)

def process_file(file_name):
    raw_data = pd.read_csv('/tmp/' + file_name, index_col=0)

    aggregate_data = (
        raw_data
        .groupby('product_id')['product_amount']
        .sum()
        .nlargest(50)
        .reset_index()
    )

    aggregate_data.columns = ['product_id', 'abandoned_amount']
    aggregate_data.to_csv('/tmp/cart_aggregated_data.csv', index=False)

def upload_file(file_name, bucket, object_name=None):
    if object_name is None:
        object_name = os.path.basename(file_name)

    s3_client = boto3.client('s3')
    try:
        s3_client.upload_file(file_name, bucket, object_name)
    except ClientError as e:
        logging.error(e)
        return False
    return True
```


### 4.5. Low Stream Consumnption  

This Lambda function applies **business logic** on cart abandonment data to generate
**promotion-ready datasets** for marketing use cases.

It identifies **top abandoned products per customer** and prepares data that can be used
to design targeted marketing campaigns.

This function represents the **business logic layer** of the event-driven ETL pipeline.

---

#### Responsibilities
- Download processed cart data from the Consumption S3 bucket
- Apply aggregation logic at the customer and product level
- Identify top products per customer based on abandonment volume
- Generate promotion-focused datasets
- Upload promotion data to the target S3 bucket

---

#### Environment Variables
- `input_bucket`: S3 bucket containing processed cart data
- `output_bucket`: S3 bucket for promotion-ready datasets

---

#### Lambda Code

```python
import os
import logging
import boto3
from botocore.exceptions import ClientError

import pandas as pd
from collections import defaultdict

inputBucket = os.environ['input_bucket']
outputBucket = os.environ['output_bucket']

def handler(event, context):
    download_file(file_name='cart_abandonment_data.csv', bucket=inputBucket)
    process_file(file_name='cart_abandonment_data.csv')
    upload_file(
        file_name='/tmp/promotion_data.csv',
        bucket=outputBucket
    )

def download_file(file_name, bucket, object_name=None):
    s3 = boto3.client('s3')
    s3.download_file(bucket, file_name, '/tmp/' + file_name)

def process_file(file_name):
    raw_data = pd.read_csv('/tmp/' + file_name)

    aggregate_data = (
        raw_data
        .groupby(['customer_id', 'product_id'])
        .agg({'product_amount': sum})
    )

    group_data = (
        aggregate_data['product_amount']
        .groupby('customer_id', group_keys=False)
        .nlargest(10)
    )

    group_data.to_csv('/tmp/promotion_data.csv')

def upload_file(file_name, bucket, object_name=None):
    if object_name is None:
        object_name = os.path.basename(file_name)

    s3_client = boto3.client('s3')
    try:
        s3_client.upload_file(file_name, bucket, object_name)
    except ClientError as e:
        logging.error(e)
        return False
    return True
```


---

## 4.6. Source

This project was completed as part of hands-on practice inspired by AWS Skill Builder training content:  
https://skillbuilder.aws/learn/T7ZQ2ZQ435/data-engineering-on-aws--a-data-lake-solution-includes-labs/W4NU348ADM?parentId=7UPVWWCC45
