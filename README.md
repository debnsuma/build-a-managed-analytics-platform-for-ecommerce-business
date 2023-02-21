# Build a managed analytics platform for an ecommerce business on AWS 

With the increase in popularity of online shopping, building an analytics platform for ecommerce is important for any organization, as it provides insights about the business, trends, and customer behavior. But, more importantly, it can uncover hidden insights that can trigger revenue-generating business decisions and actions. In this blog, we will learn how to build a complete analytics platform in batch and real-time mode. The real-time analytics pipeline also shows how to detect distributed denial of service (DDoS) and bot attacks, which is a common requirement for such use cases.

## Introduction 

E-commerce analytics is the process of collecting data from all of the sources that affect a certain online business. Data Analysts or Business Analysts can then utilize this information to deduce changes in customer behavior and online shopping patterns. E-commerce analytics spans the whole customer journey, starting from discovery through acquisition, conversion, and eventually retention and support.

In this blog, we will use an eCommerce dataset from Kaggle to simulate the logs of user purchases, product views, cart history, and the user’s journey on the online platform to create two analytical pipelines:

**Batch Processing**  

The `Batch processing` will involve data ingestion, Lake House architecture, processing, visualization using Amazon Kinesis, Glue, S3, and QuickSight to draw insights regarding the following:

- Unique visitors per day

- During a certain time, the users add products to their carts but don’t buy them

- Top categories per hour or weekday (i.e. to promote discounts based on trends)

- To know which brands need more marketing

**Real-time Processing** 

The `Real-time processing` involves detecting Distributed denial of service (DDoS) and Bot attacks using AWS Lambda, DynamoDB, CloudWatch, and AWS SNS.

![Img1](/img/img1.png)


## Dataset 

For this blog, we are going to use the [eCommerce behavior data from multi category store](https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store)

This file contains the behavior data for 7 months (from October 2019 to April 2020) from a large multi-category online store.

Each row in the file represents an event. All events are related to products and users. Each event is like many-to-many relation between products and users.

## Architecture 

**Batch Processing**  

We are going to build an end to end data engineering pipeline where we will start with this [eCommerce behavior data from multi category store](https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store) dataset as an input, which we will use to simulate or mimic real time e-commerce workload. 

This input stream of data will be coming to an Amazon Kinesis Data Stream, which will send the data to Amazon Kinesis Data Analytics (`stream1`), where we use an Flink application to detect any DDoS attack, and the filtered data will be send to another Amazon Kinesis Data Stream (`stream2`). 

We are going to use SQL to build the `Apache Flink` application using Amazon Kinesis Data Analytics, and hence we would need a metadata store, for which we are going to use AWS Glue. 

And then this `stream2` will trigger an AWS Lambda function which will send an Amazon SNS notification to the stakeholders and shall store the fraudulent transaction details in a DynamoDB table. 

So, the architecture would look like this. 

![Img1](/img/img2.png)

**Real-time Processing** 

If we look into the architecture diagram above, we will see that we are not storing the `raw` incoming data anywhere, whatever data is coming from the `stream1` we are passing it to Amazon Kinesis Data Analytics to analyze. And it might happen that later on we discover some bug in our `Apache Flink` application, and at that point, we can fix the bug and resume processing the data, but we can not process the old data (which was processed by our buggy `Apache Flink` application, since we have not stored the raw data anywhere which we can revisit)

And thats why its always recommended to alway have a copy of the `raw` data stored in some storage (e.g. on Amazon S3) so that we can revisit the data if needed for reprocessing and/or for batch processing. 

And this is exactly what we are going to do, we will use the same incoming data stream from Amazon Kinesis Data Stream (`stream1`) and pass it on to Amazon Kinesis Firehose which can write the data on Amazon S3. 

Now we can have the `raw` incoming data on Amazon S3, and we can use AWS Glue to catalog that data and using AWS Glue ETL to process or clean that data which we can further use by Amazon Athena to run any analytical queries. 

At last we would leverage Amazon QuickSight to build a dashboard for visualization.  

![Img1](/img/img3.png)

## Step by step walk through

Lets build this application step by step. We are could to use an [AWS Cloud9 instance](https://aws.amazon.com/cloud9/), but it is not mandatory to use for this project. But if you wish to spin up an AWS Cloud9 instance, you may like to follow steps mentions [here](https://docs.aws.amazon.com/cloud9/latest/user-guide/create-environment-main.html) and proceed further. 


### Download the dataset and clone the GirHub Repo 

Clone the project and change it to the right directory:

```bash

# Project repository 
git clone https://github.com/debnsuma/build-a-managed-analytics-platform-for-ecommerce-business.git

cd build-a-managed-analytics-platform-for-ecommerce-business/

# Create a folder to store the dataset 
mkdir dataset 

```

Download the dataset from [here]((https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store)) and move the downloaded `2019-Nov.csv.zip` under the `dataset` folder  

![img](/img/img4.png)

Now, lets unzip the file and create a sample version of the dataset by just taking the first `1000` records from the file. 

```bash

cd dataset 

unzip 2019-Nov.csv.zip

cat 2019-Nov.csv | head -n 1000 > 202019-Nov-sample.csv

```

### Create an Amazon S3 bucket 

Now we can create an Amazon S3 bucket and upload this dataset 

- Name of the Bucket : `ecommerce-raw-us-east-1-dev` (replace this with your own `BUCKET_NAME`)

```bash

# Copy all the files in the S3 bucket 
aws s3 cp 2019-Nov.csv.zip s3://<BUCKET_NAME>/ecomm_user_activity/p_year=2019/p_month=11/
aws s3 cp 202019-Nov-sample.csv s3://<BUCKET_NAME>/ecomm_user_activity_sample/202019-Nov-sample.csv
aws s3 cp 2019-Nov.csv s3://<BUCKET_NAME>/ecomm_user_activity_unconcompressed/p_year=2019/p_month=11/

```

### Create the Kinesis Data Stream 

Now, lets create the first Kinesis data stream which we will be using as the incoming stream. Open the AWS Console and then:

- Go to **Amazon Kinesis** 
- Click on **Create data stream** 

![](/img/img5.png)

- Put `ecommerce-raw-user-activity-stream-1` as the Data stream name
- Click on **Create data stream** 

![](/img/img6.png)


Lets create another Kinesis data stream which we are going to use later on. This time use the Data stream name as `ecommerce-raw-user-activity-stream-2` 

![](/img/img7.png)

### Start the e-commerce traffic 

Now that we have our **Kinesis Data Stream** is ready we can start the e-commerce traffic using a stimulator. This stimulator (a `python script`) reads the `202019-Nov-sample.csv` (the dataset which we downloaded) line by line and send it to the Kinesis data stream. 

But before you run the stimulator, just edit the `stream-data-app-simulation.py` with your *<BUCKET_NAME>*

```python

# S3 buckect details (UPDATE THIS)
BUCKET_NAME = "ecommerce-raw-us-east-1-dev"       
```

Once its updated, we can run the stimulator. 

```bash 
# Go back to the project root directory 
cd .. 

# Run stimulator 
python code/ecomm-simulation-app/stream-data-app-simulation.py 

```

