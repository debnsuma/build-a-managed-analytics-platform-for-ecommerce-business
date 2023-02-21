# Build a managed analytics platform for an ecommerce business on AWS 

With the increase in popularity of online shopping, building an analytics platform for ecommerce is important for any organization, as it provides insights about the business, trends, and customer behavior. But, more importantly, it can uncover hidden insights that can trigger revenue-generating business decisions and actions. In this blog, we will learn how to build a complete analytics platform in batch and real-time mode. The real-time analytics pipeline also shows how to detect distributed denial of service (DDoS) and bot attacks, which is a common requirement for such use cases.

## Introduction 

Ecommerce analytics is the process of collecting data from all of the sources that affect a certain online business. Data Analysts or Business Analysts can then utilize this information to deduce changes in customer behavior and online shopping patterns. Ecommerce analytics spans the whole customer journey, starting from discovery through acquisition, conversion, and eventually retention and support.

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

