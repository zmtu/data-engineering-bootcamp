# Data Processing

Data processing involves taking source data which has been ingested into your data platform and cleansing it, combining it, and modeling it for downstream use. Historically the most popular way to transform data has been with the SQL language and data engineers have built data transformation pipelines using SQL often with the help of ETL/ELT tools. But recently many folks have also begun adopting the DataFrame API in languages like Python/Spark for this task. For the most part a data engineer can accomplish the same data transformations with either approach, and deciding between the two is mostly a matter of preference and particular use cases. That being said, there are use cases where a particular data transform can't be expressed in SQL and a different approach is needed. The most popular approach for these use cases is Python/Spark along with a DataFrame API.

![](https://user-images.githubusercontent.com/62965911/214255202-0563b49c-1708-4dbb-ac2f-cca3843055d6.gif)

### Transformation in the Warehouse using SQL

It’s time to transform the raw data into the end-user data model. Transformations can affect

- Data processing time.
- Data warehouse cost. Modern data warehouses usually charge based on the amount of data scanned.
- Data pipeline development speed and issues.

#### Transformation types

The ultimate goal for optimizing transformations is to reduce the movement of data within your data warehouse. Data warehouses are distributed systems with the data stored as chunks across the cluster. Reducing the movement of data across the machines within the distributed system significantly speeds up the processing of data.

There are two major types of transformations as explained below.

1. Narrow transformations: These are transformations that do not involve the movement of data across machines within the warehouse. The transformations are applied to the rows without having to move these rows to other machines within the warehouse.

E.g. Lower(), Concat(), etc are functions that are applied directly to the data in memory

2. Wide transformations: These are transformations that involve the movement of data across machines within the warehouse.

E.g. When you join 2 tables, the warehouse engine will move the smaller table’s data to the same machine(s) as the larger table’s data. This is so that these 2 tables can be joined. Moving data around is a high-cost operation in a distributed system, and as such, the warehouse engine will optimize to keep the data movement to a minimum.

When self-joining, it’s beneficial to join on the partitioned column(s) as this will keep data movement within the system to a minimum.

Some common transformations to know are

- Joins, anti joins
- String, numeric, and date functions
- Group by, aggregates, order by, union, having
- CTEs
- Window functions
- Parsing JSON
- Stored procedures, sub queries and functions

Some points you need answered/explored are

1. How does transformation time increase with an increase in the data size? Is it linear or worse? Hint: A cross join will not scale linearly
2. Read the data warehouse documentation to know what features exist. This allows you to go back to the docs in case you need to use a feature. Most transformations can be done within your data warehouse.
3. When evaluating performance be aware of cached reads on subsequent queries.
4. When possible, filter the data before or during the transformation query.
5. Most SQL queries are a mix of wide and narrow transformations.

#### Query planner

The query planner lets you see what steps the warehouse engine will take to run your query. You can use the EXPLAIN command to see the query plan.

Most data warehouse documentation has steps you can take to optimize your queries. E.G. Snowflake’s common query issues, Redshift’s query plan and execution

In short

1. Use explain to see the query plan.
2. Optimize the steps that have the highest costs. Use available warehouse documentation for optimization help.

### Most Common Data Transformations

#### File format optimizations

CSV, XML, JSON, and other types of plaintext files are commonly used to store structured and semi-structured data. These file formats are useful when manually exploring data, but there are much better, binary-based file formats to use for computer-based analytics. A common binary format that is optimized for read-heavy analytics is the Apache Parquet format. A common transformation is to convert plaintext files into an optimized format, such as Apache Parquet.

Within modern data lake environments, there are a number of file formats that can be used that are optimized for data analytics. From an analytics perspective, the most popular file format currently is **Apache Parquet**.

**Parquet** files are columnar-based, meaning that the contents of the file are physically stored to have data grouped by columns, rather than grouped by rows as with most file formats. (CSV files, for example, are physically stored to be grouped by rows.) As a result, queries that select a set of specific columns (rather than the entire row) do not need to read through all the data in the Parquet file to return a result, leading to performance improvements.

Parquet files also contain metadata about the data they store. This includes schema information (the data type for each column), as well as statistics such as the minimum and maximum value for a column contained in the file, the number of rows in the file, and so on.

A further benefit of Parquet files is that they are optimized for compression. A 1 TB dataset in CSV format could potentially be stored as 130 GB in Parquet format once compressed. Parquet supports multiple compression algorithms, although Snappy is the most widely used compression algorithm.

These optimizations result in significant savings, both in terms of storage space used and for running queries.

For example, the cost of an Amazon Athena query is based on the amount of compressed data scanned (at the time of writing, this cost was $5 per TB of scanned data). If only certain columns are queried of a Parquet file, then between the compression and only needing to read the data chunks for the specific columns, significantly less data needs to be scanned to resolve the query.

In a scenario where your data table is stored across perhaps hundreds of Parquet files in a data lake, the analytics engine is able to get further performance advantages by reading the metadata of the files. For example, if your query is just to count all the rows in a table, this information is stored in the Parquet file metadata, so the query doesn't need to actually scan any of the data. For this type of query, you will see that Athena indicates that 0 KB of data was scanned, therefore there is no cost for the query.

Or, if your query is for where the sales amount is above a specific value, the analytics engine can read the metadata for a column to determine the minimum and maximum values stored in the specific data chunk. If the value you are searching for is higher than the maximum value recorded in the metadata, then the analytics engine knows that it does not need to scan that specific column data chunk. This results in both cost savings and increased performance for queries.

Because of these performance improvements and cost savings, a very common transformation is to convert incoming files from their original format (such as CSV, JSON, XML, and so on) into the analytics-optimized Parquet format.

#### Data standardization

When building out a pipeline, we often load data from multiple different data sources, and each of those data sources may have different naming conventions for referring to the same item. For example, a field containing someone's birth date may be called *DOB*, *dateOfBirth*, *birth_date*, and so on. The format of the birth date may also be stored as *mm/dd/yy*, *dd/mm/yyyy*, or in a multitude of other formats.

One of the tasks we may want to do when optimizing data for analytics is to standardize column names, types, and formats. By having a corporate-wide analytic program, standard definitions can be created and adopted across all analytic projects in the organization.

#### Data quality checks

Another aspect of data transformation may be the process of verifying data quality and highlighting any ingested data that does not meet the expected quality standards.

#### Data partitioning

Partitioning and bucketing are used to maximize benefits while minimizing adverse effects. It can reduce the overhead of shuffling, the need for serialization, and network traffic. In the end, it improves performance, cluster utilization, and cost-efficiency.

Partition helps in localizing data and reducing data shuffling across the network nodes, reducing network latency, which is a major component of the transformation operation, thereby reducing the time of completion. A good partitioning strategy knows about data and its structure, and cluster configuration. Bad partitioning can lead to bad performance, mostly in 3 fields:

- Too many partitions regarding your cluster size and you won’t use efficiently your cluster. For example, it will produce intense task scheduling.
- Not enough partitions regarding your cluster size, and you will have to deal with memory and CPU issues: memory because your executor nodes will have to put high volume of data in memory (possibly causing OOM Exception), and CPU because compute across the cluster will be unequal.
- Skewed data in your partitions can occur. When a Spark task is executed in these partitioned, they will be distributed across executor slots and CPUs. If your partitions are unbalanced in terms of data volume, some tasks will run longer compared to others and will slow down the global execution time of the tasks (and a node will probably burn more CPU that others).

**How to decide the partition key(s)?**

- Choose low cardinality columns as partition columns (since a HDFS directory will be created for each partition value combination). Generally speaking, the total number of partition combinations should be less than 50K. (For example, don’t use partition keys such as roll_no, employee_ID etc. Instead use the state code, country code, geo_code, etc.)
- Choose the columns used frequently in filtering conditions.
- Use at most 2 partition columns as each partition column creates a new layer of directory.

##### Different methods that exist in PySpark

**1. Repartitioning**

The first way to manage partitions is the repartition operation. Repartitioning is the operation to reduce or increase the number of partitions in which the data in the cluster will be split. This process involves a full shuffle. Consequently, it is clear that repartitioning is an expensive process. In a typical scenario, most of the data should be serialized, moved, and deserialized.

```py
repartitioned = df.repartition(8)
```

In addition to specifying the number of partitions directly, you can pass in the name of the column by which you want to partition the data.

```py
repartitioned = df.repartition('country')
```

**2. Coalesce**

The second way to manage partitions is coalesce. This operation reduces the number of partitions and avoids a full shuffle. The executor can safely leave data on a minimum number of partitions, moving data only from redundant nodes. Therefore, it is better to use coalesce than repartition if you need to reduce the number of partitions.

```py
coalesced = df.coalesce(2)
```

**3. PartitionBy**

partitionBy(cols) is used to define the folder structure of data. However, there is no specific control over how many partitions are going to be created. Different from the coalesce andrepartition functions, partitionBy effects the folder structure and does not have a direct effect on the number of partition files that are going to be created nor the partition sizes.

```py
green_df \ 
    .write \ 
.partitionBy("pickup_year", "pickup_month") \ 
    .mode("overwrite") \ 
    .csv("data/partitions/partitionBy.csv", header=True)
```

##### Benefits of partitioning

Partitioning has several benefits apart from just query performance. Let's take a look at a few important ones.

**1. Improving performance**

Partitioning helps improve the parallelization of queries by splitting massive monolithic data into smaller, easily consumable chunks.

Apart from parallelization, partitioning also improves performance via **data pruning**. Using data pruning queries can ignore non-relevant partitions, thereby reducing the **input/output** (**I/O**) required for queries.

Partitions also help with the archiving or deletion of older data. For example, let's assume we need to delete all data older than 12 months. If we partition the data into units of monthly data, we can delete a full month's data with just a single **DELETE** command, instead of deleting all the files one by one or all the entries of a table row by row.

Let's next look at how partitioning helps with scalability.

**2. Improving scalability**

In the world of big data processing, there are two types of scaling: vertical and horizontal.

**Vertical scaling** refers to the technique of increasing the capacity of individual machines by adding more memory, CPU, storage, or network to improve performance. This usually helps in the short term, but eventually hits a limit beyond which we cannot scale.

The second type of scaling is called **horizontal scaling**. This refers to the technique of increasing processing and storage capacity by adding more and more machines to a cluster, with regular hardware specifications that are easily available in the market (commodity hardware). As and when the data grows, we just need to add more machines and redirect the new data to the new machines. This method theoretically has no upper bounds and can grow forever. Data lakes are based on the concept of horizontal scaling.

Data partitioning helps naturally with horizontal scaling. For example, let's assume that we store data at a per-day interval in partitions, so we will have about 30 partitions per month. Now, if we need to generate a monthly report, we can configure the cluster to have 30 nodes so that each node can process one day's worth of data. If the requirement increases to process quarterly reports (that is, reports every 3 months), we can just add more nodes---say, 60 more nodes to our original cluster size of 30 to process 90 days of data in parallel. Hence, if we can design our data partition strategy in such a way that we can split the data easily across new machines, this will help us scale faster.

Let's next look at how partitioning helps with data management.

**3. Improving manageability**

In many analytical systems, we will have to deal with data from a wide variety of sources, and each of these sources might have different data governance policies assigned to them. For example, some data might be confidential, so we need to restrict access to that; some might be transient data that can be regenerated at will; some might be logging data that can be deleted after a few months; some might be transaction data that needs to be archived for years; and so on. Similarly, there might be data that needs faster access, so we might choose to persist it on premium **solid-state drive** (**SSD**) stores and other data on a **hard disk drive** (**HDD**) to save on cost.

If we store such different sets of data in their own storage partitions, then applying separate rules---such as access restrictions, or configuring different data life cycle management activities such as deleting or archiving data, and so on---for the individual partitions becomes easy. Hence, partitioning reduces the management overhead, especially when dealing with multiple different types of data such as in a data lake.

Let's next look at how data partitioning helps with security.

**4. Improving security**

As we saw in the previous section about improving manageability, confidential datasets can have different access and privacy levels. Customer data will usually have the highest security and privacy levels. On the other hand, product catalogs might not need very high levels of security and privacy.

So, by partitioning the data based on security requirements, we can isolate the secure data and apply independent access-control and audit rules to those partitions, thereby allowing only privileged users to access such data.

Let's next look at how we can improve data availability using partitioning.

**5. Improving availability**

If our data is split into multiple partitions that are stored in different machines, applications can continue to serve at least partial data even if a few partitions are down. Only a subset of customers whose partitions went down might get impacted, while the rest of the customers will not see any impact. This is better than the entire application going down. Hence, physically partitioning the data helps improve the availability of services.

In general, if we plan our partition strategy correctly, the returns could be significant. I hope you have now understood the benefits of partitioning data. Let's next look at some partition strategies from a storage/files perspective.

##### Key Points

1. Do not partition by columns with high cardinality.
2. Partition by specific columns that are mostly used during filter and groupBy operations.
3. Even though there is no best number, it is recommended to keep each partition file size between 256MB to 1GB.
4. If you are increasing the number of partitions, use repartition()(performing full shuffle).
5. If you are decreasing the number of partitions, use coalesce() (minimizes shuffles).
6. Default no of partitions is equal to the number of CPU cores in the machine.
7. GroupByKey, ReduceByKey — by default this operation uses Hash Partitioning with default parameters.

A common optimization strategy for analytics is to partition the data, grouping the data at the physical storage layer by a field that is often used in queries. For example, if data is often queried by a date range, then data can be partitioned by a date field. If storing sales data, for example, all the sales transactions for a specific month would be stored in the same Amazon S3 prefix (which is much like a directory). When a query is run that selects all the data for a specific day, the analytic engine only needs to read the data in the directory that's storing data for the relevant month.

Another common approach for optimizing datasets for analytics is to **partition** the data, which relates to how the data files are organized in the storage system for a data lake.

**Hive partitioning** splits the data from a table to be grouped together in different folders, based on one or more of the columns in the dataset. While you can partition the data in any column, a common partitioning strategy that works for many datasets is to partition based on date.

For example, suppose you had sales data for the past four years from around the country, and you had columns in the dataset for **Day**, **Month** and **Year**. In this scenario, you could select to partition the data based on the **Year** column. When the data was written to storage, all the data for each of the past few years would be grouped together with the following structure:

datalake_bucket/year=2021/file1.parquet

datalake_bucket/year=2020/file1.parquet

datalake_bucket/year=2019/file1.parquet

datalake_bucket/year=2018/file1.parquet

If you then run a SQL query and include a **WHERE Year = 2018** clause, for example, the analytics engine only needs to open up the single file in the **datalake_bucket/year=2018** folder. Because less data needs to be scanned by the query, it costs less and completes quicker.

Deciding on which column to partition by requires that you have a good understanding of how the dataset will be used. If you partition your dataset by year but a majority of your queries are by the **business unit** (**BU**) column across all years, then the partitioning strategy would not be effective.

Queries you run that do not use the partitioned columns may also end up causing those queries to run slower if you have a large number of partitions. The reason for this is that the analytics engine needs to read data in all partitions, and there is some overhead in working between all the different folders. If there is no clear common query pattern, it may be better to not even partition your data. But if a majority of your queries use a common pattern, then partitioning can provide significant performance and cost benefits.

You can also partition across multiple columns. For example, if you regularly process data at the day level, then you could implement the following partition strategy:

datalake_bucket/year=2021/month=6/day=1/file1.parquet

This significantly reduces the amount of data to be scanned when queries are run at the daily level and also works for queries at the month or year level. However, another warning regarding partitioning is that you want to ensure that you don't end up with a large number of small files. The optimal size of Parquet files in a data lake is 128 MB–1 GB. The Parquet file format can be split, which means that multiple nodes in a cluster can process data from a file in parallel. However, having lots of small files requires a lot of overhead for opening, reading metadata, scanning data, and closing each file, and can significantly impact performance.

Partitioning is an important data optimization strategy and is based on how the data is expected to be used, either for the next transformation stage or for the final analytics stage. Determining the best partitioning strategy requires that you understand how the data will be used next.

#### Data denormalization

In traditional relational database systems, the data is normalized, meaning that each table contains information on a specific focused topic, and associated, or related, information is contained in a separate table. The tables can then be linked through the use of foreign keys.

For data lakes, combining the data from multiple tables into a single table can often improve query performance. Data denormalization takes two (or more) tables and creates a new table with data from both tables.

#### Data cataloging

Another important component that we should include in the transformation section of our pipeline architecture is the process of cataloging the dataset. During this process, we ensure all the datasets in the data lake are referenced in the data catalog and can add additional business metadata.

## Processing Streaming data

In the big data era, people like to correlate big data with real-time data. Some people say that if the data is not real-time, then it's not big data. This statement is partially true. In practice, the majority of data pipelines in the world use the batch approach, and that's why it's still very important for data engineers to understand the batch data pipeline.

However, real-time capabilities in the big data era are something that many data engineers need to start to rethink in terms of data architecture. To understand more about architecture, we first need to have a clear definition of what real-time data is.

From the end-user perspective, real-time data can mean anything---anything from faster access to data, more frequent data refreshes, and detecting events as soon as they happen. From a data engineer perspective, what we need to know is *how* to make it happen. For example, if you search any keyword in Google Search, you will get an immediate result in real time. Or, in another example, if you open any social media page, you can find your account statistics and the number of friends, visitors, and other information in real time. 

But if you think about it, you may notice that there is nothing new in both of the examples. As an end user, you can get fast access to the data because of the backend database, and this is a common practice dating from the 1990s. In big data, what's new is the real time in terms of incoming data and processing. This specific real-time aspect is called **streaming data**. For example, you need to detect fraud as soon as an event happens. In most cases, detecting fraud needs many parameters from multiple data sources, which are impossible to be handled in the application database. If fraud has no relevance to your company, you can think of other use cases such as real-time marketing campaigns and real-time dashboards for sales reporting. Both use cases may require a large volume of data from multiple data sources. In these kinds of cases, being able to handle data in streams may come in handy.

### Streaming data for data engineers

Streaming data in data engineering means a data pipeline that flows data from the upstream to the downstream as soon as the data is created. In nature, all data is created in real time, so processing data in batches is a way to simplify the process.

What does this mean?

If you think about how every piece of data is created, it's always in real time---for example, a user registered to a mobile application. The moment the user registered, data was created in the database. Another example is of data being input into a spreadsheet. The moment the data entry is written in the spreadsheet, data is created in the sheet. Looking at the examples, you can see that data always has a value and the time when it's created. Processing streaming data means we process the data as soon as the data is inputted into a system. 

Take a look at the following diagram. Here, there are two databases: the source database and the target database. For streaming data, every time a new record is inserted into the source database (illustrated as boxes with numbers), the record is immediately processed as an event and inserted into the target database. These events happen continuously, which makes the data in the target database real-time compared to the source database:

![Figure_6 1](https://user-images.githubusercontent.com/62965911/219853662-f177ccc8-463d-417f-8c18-ddc5eef722ba.jpg)

In data engineering, the antonym for *stream* is *batch*. Batching is a very common approach for data engineers to process data. Even though we know data is real-time in nature, it's a lot easier to process data in a set time. For example, if your **chief executive officer** (**CEO**) wants to know about the company's revenue, it's a lot easier to populate the purchase history in a month and calculate the revenue, rather than build a system then continuously calculate the purchase transactions. Or, in a more granular level, a batch is commonly done on a weekly, daily, or hourly basis. The batch approach is a widely used practice in the data engineering world; I can say 90% of the data pipeline is a batch. 

Take a look at the following diagram. Here, the two databases are the same as the above figure. The nature of how data is inserted into the source database is also the same---it's always real-time from the source database perspective. The difference is in the processing step, where the records are grouped into batches. Every batch is scheduled individually to be loaded to the target database:

![Figure_6 2](https://user-images.githubusercontent.com/62965911/219853667-6aa1ca55-587b-4133-bdb9-6ce486724a3a.jpg)

There are two major reasons why processing data in batches is more popular compared to streams. The first is because of the data user's needs. The data user, who is usually called a decision maker, naturally asks for information over a period of time. As we've discussed in the previous example, common questions are *How much X in a day?* or *How many X in a month?*. It's very rare to have a question such as *How much X occurred this second compared to the last second?*.

The second reason is the complexity of technology. Batching is easier than streaming because you can control a lot of things in a batch---for example, you can control how you want to schedule data pipeline jobs. This doesn't apply to stream processing. A streaming process is one job; once it runs, it will process all the incoming data and it never ends. To handle all the complexities, we need specific sets of technologies.

## Real-Time Analytics Applications

Real-time analytics applications combine characteristics from both traditional reporting analytics and transactional applications.

![img](https://user-images.githubusercontent.com/62965911/216754384-be942702-f617-4ce9-a231-061b4a291f19.png "Real-Time Analytics Applications")

Streaming data is data that is generated continuously by thousands of data sources, which typically send in the data records simultaneously, and in small sizes (order of Kilobytes). Streaming data includes a wide variety of data such as log files generated by customers using your mobile or web applications, ecommerce purchases, in-game player activity, information from social networks, financial trading floors, or geospatial services, and telemetry from connected devices or instrumentation in data centers.

Analyzing and measuring data as soon as it enters the database is referred to as real-time analytics. Thus, users gain insights or may conclude as soon as data enters their system. Businesses can react quickly using real-time analytics. They can grasp opportunities and avert issues before they occur. On the other hand, Batch-style analytics might take hours or even days to provide findings. As a result, batch analytical systems frequently produce only static insights based on lagging indications. Real-time analytics insights may help organizations stay ahead of the competition. These pipelines for streaming data generally follow a 3 step process, i.e., Ingest, Analyze and Deliver.

Data from various sources, including applications, devices, sensors, clickstreams, and social media feeds, can be analyzed to find patterns and linkages. These patterns can be used to start workflows and trigger events like alert creation, information feeding into reporting tools, or storing altered data for future use.

This data needs to be processed sequentially and incrementally on a record-by-record basis or over sliding time windows, and used for a wide variety of analytics including correlations, aggregations, filtering, and sampling. Information derived from such analysis gives companies visibility into many aspects of their business and customer activity such as –service usage (for metering/billing), server activity, website clicks, and geo-location of devices, people, and physical goods –and enables them to respond promptly to emerging situations. For example, businesses can track changes in public sentiment on their brands and products by continuously analyzing social media streams, and respond in a timely fashion as the necessity arises.

There are four key requirements for supporting real-time analytics. They are latency, freshness, throughput, and concurrency, as seen below:

![](https://user-images.githubusercontent.com/62965911/214567242-a0f0fb33-53f1-4d05-99b5-cc574e05713e.png)

These four ideas work together to control business’s ability to make decisions on the fly. They are required to ensure that data transforms into actionable analytics as close to real-time as possible (in fractions of a second). They are also necessary to quickly tie current transactions with historical data for comparisons and trends without having to preprocess the data.

AWS has a very robust stack of technologies such as Kinesis Data Streams, Kinesis Data Firehose, Kinesis Data Analytics, and Managed Streaming for Kafka when it comes to working with streaming data.

### Examples of Real-Time Analytics Applications

#### Netflix

To ensure a consistently great experience to more than 100 million members in more than 190 countries enjoying 125 million hours **of TV shows** and movies each day, Netflix built a real-time analytics application for user experience monitoring. By turning log streams into real-time metrics, Netflix can see how over 300 million devices are performing at all times in the field. Ingesting over 2 million events per second and querying over 1.5 trillion rows, Netflix engineers can pinpoint anomalies within their infrastructure, endpoint activity, and content flow.

An ongoing challenge for Netflix is consistently delivering a great streaming entertainment experience while continuously pushing innovative technology updates. As Netflix’s adoption has skyrocketed, this challenge has grown more complex. With over 300 million devices spanning four major UIs including iOS, Android, smart TVs, and their own website, Netflix has a constant need to identify and isolate issues that may affect only a certain group, such as a version of the app, certain types of devices, or particular countries. Ben Sykes, software engineer at Netflix, says that “with this data arriving at over 2 million events per second, getting it into a database that can be queried quickly is formidable. We need sufficient dimensionality for the data to be useful in isolating issues and as such we generate over 115 billion rows per day.”

To quantify how seamlessly users’ devices are handling browsing and playback, Netflix derives measurements using real-time logs from playback devices as a source of events.

![The log to metric data pipeline at NetflixSource for the figure comes from Ben Sykes   How Netflix Uses Druid for Real Time Insights.](https://user-images.githubusercontent.com/62965911/216756143-7c88c769-22f4-4bfb-bb29-19cfd6226dba.png)

Netflix collects these measures and feeds them into the real-time analytics database. Every measure is tagged with anonymized details about the kind of device being used—for example, whether the device is a smart TV, an iPad, or an Android phone. This enables Netflix to classify devices and view the data according to various aspects. This aggregated data is available immediately for querying, either via dashboards or ad hoc queries. “We’re currently ingesting at over 2 million events per second,” Sykes says, “and querying over 1.5 trillion rows to get detailed insights into how our users are **experiencing** the service. All this helps us maintain a high-quality Netflix experience, while enabling constant innovation.”

The ultimate benefit is speed, which is essential for a service that needs to react to a massive number of users in near real time.

#### Walmart

To compete with Amazon and other retailers, Walmart needs to track the pricing of their competitors in real time and allow analysts to explore the gathered data interactively.

Many data sets at Walmart are generated by the digital business, modeled as streams of events ranging from server logs to application metrics to product purchases. The goal of the team at Walmart Labs is to make it easy for the right people across Walmart to access this data, analyze it, and make decisions in the least amount of time **possible.**

Walmart’s first attempt to provide low-latency analytics was to leverage the Hadoop ecosystem. It tried using Apache Hive first and then Presto. According to Amaresh Nayak, distinguished SW engineer at Walmart Labs, “The problem we faced with both of these SQL-on-Hadoop solutions was that queries would sometimes take hours to complete, which significantly impacted our ability to make rapid decisions. Although our data was arriving in real time, our queries quickly became a bottleneck in our decision-making cycle as our data volumes grew. We quickly realized that the workflow we were aiming to optimize was one where we could look at our event streams (both real-time and historical events) and slice and dice the data to look at specific subsections, determine trends, find root causes, and take actions accordingly.”

The team at Walmart knew it needed to make a change. The types of queries the team runs require column aggregation, which involves scanning a lot of rows across multiple shards. Walmart found that relational databases were poorly suited for this because they cannot efficiently enable data exploration in real time. The team also ruled out a NoSQL key-value database because of the need to query multiple partitions across a number of nodes. Using this database would have caused inefficient aggregation calculations and exponentially increased storage requirements by storing aggregates for all possible column combinations.

Using a real-time analytics application, Walmart can now pre-aggregate records as they are being ingested. Instead of getting a single price for an item for a specific moment in time, Walmart can now understand the changing price of an item over any span of time. This combined level of depth and speed to insights is essential to enabling Walmart to make critical pricing decisions in the least amount of time possible. “After we switched [to real-time analytics],” Nayak says, “our query latencies also dropped to near sub-second and in general, the project fulfilled most of our requirements. Today, our cluster ingests nearly 1B+ events per day (2TB of raw data).”

## Streaming in Google Cloud

Watch this video: https://www.youtube.com/watch?v=3Y7PhzFbz_s

## Streaming data challenges

Watch this video: https://www.youtube.com/watch?v=0vbdoFNCUOA

## More Resources

1. https://www.fivetran.com/blog/what-is-data-transformation
