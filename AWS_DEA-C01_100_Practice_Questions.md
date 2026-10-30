# AWS Certified Data Engineer – Associate (DEA-C01)
## 100 Practice Questions with Full Explanations

**How to use this file**
- Read the question and try to answer *before* looking at the answer.
- The correct option is marked with **✅**. Every option (right and wrong) is explained so you learn *why* AWS wants a particular answer.
- Questions follow the real exam style: short scenario + "MOST cost-effective", "LEAST operational overhead", "MOST scalable", etc. **Always look for these keywords** – they decide the answer.
- Some questions say "Choose TWO" – those are marked clearly.

**Exam domains covered**
| Part | Questions | Domain |
|---|---|---|
| 1 | Q1–Q20 | Domain 1 – Data Ingestion & Transformation |
| 2 | Q21–Q40 | Domain 1 (Transformation) + Domain 2 – Data Store Management |
| 3 | Q41–Q60 | Domain 2 – Data Stores, Cataloging, Data Modeling & Lifecycle |
| 4 | Q61–Q80 | Domain 3 – Data Operations & Support |
| 5 | Q81–Q100 | Domain 4 – Data Security & Governance |

---

# PART 1 – Data Ingestion & Transformation (Q1–Q20)

---

## Q1
A company collects clickstream events from a website. The data must be delivered to Amazon S3 in **Parquet format** within about 1–2 minutes of arrival. The team wants the **LEAST operational overhead**. Which solution meets these requirements?

- A) Amazon Kinesis Data Streams with a custom consumer application on EC2 that writes Parquet files to S3
- **✅ B) Amazon Kinesis Data Firehose with record format conversion enabled (using an AWS Glue Data Catalog table) and S3 as the destination**
- C) AWS Glue batch job that runs every hour and converts JSON to Parquet
- D) Amazon EMR cluster running Spark Streaming and writing to S3

**Correct answer: B**

**Why each option:**
- **A ❌** Kinesis Data Streams works, but you must write, host and scale your own consumer on EC2. That is *high* operational overhead.
- **B ✅** Firehose is fully managed, buffers data (as low as 60 seconds), can convert JSON → Parquet/ORC using a Glue table schema, and delivers to S3. No servers, no code.
- **C ❌** An hourly batch job cannot meet the "1–2 minutes" requirement.
- **D ❌** EMR means managing a cluster. It works technically but is far from "least overhead".

💡 **Remember:** *Near real-time delivery to S3/Redshift/OpenSearch with no code = Firehose.* Format conversion (JSON→Parquet/ORC) is built into Firehose.

---

## Q2
An IoT application sends readings from thousands of devices to **Amazon Kinesis Data Streams**. Readings from the **same device must be processed in the order they were sent**. What should the data engineer do?

- A) Use a random partition key for every record
- **✅ B) Use the device ID as the partition key**
- C) Use a single shard and a random partition key
- D) Use Kinesis Data Firehose instead because it guarantees ordering

**Correct answer: B**

**Why each option:**
- **A ❌** A random key spreads records across shards in any order, so per-device ordering is lost.
- **B ✅** Records with the same partition key always go to the same shard, and Kinesis keeps order *within a shard*. So the same device's data stays in order.
- **C ❌** One shard gives global order but limits throughput to 1 MB/s in and 2 MB/s out. It won't scale for thousands of devices.
- **D ❌** Firehose has no ordering guarantee per key and has no concept of shard-level ordering for consumers.

💡 **Remember:** *Ordering in Kinesis = per shard, controlled by partition key.*

---

## Q3
A Kinesis Data Streams producer is receiving `ProvisionedThroughputExceededException` errors. CloudWatch shows **one shard is at its limit while other shards are almost idle**. What is the MOST likely cause and fix?

- A) The stream retention is too short; increase retention to 7 days
- B) The consumer is too slow; enable enhanced fan-out
- **✅ C) A "hot shard" caused by a poorly distributed partition key; choose a partition key with higher cardinality (more unique values)**
- D) The producer is using the Kinesis Producer Library; switch to the AWS SDK

**Correct answer: C**

**Why each option:**
- **A ❌** Retention only controls how long data is kept. It has nothing to do with write throttling.
- **B ❌** Enhanced fan-out helps *consumers* (dedicated 2 MB/s per consumer). The error here is on the *write* side.
- **C ✅** If most records share the same key value, they land on the same shard (a hot shard). Using a more evenly distributed key (or adding a random suffix) spreads the load. You can also split the hot shard.
- **D ❌** KPL actually helps with batching/aggregation and is not the cause.

💡 **Remember:** *One shard busy, others idle = hot shard = bad partition key.*

---

## Q4
A company wants to migrate an on-premises **Oracle** database to **Amazon Aurora PostgreSQL** with **minimal downtime**. The schema must also be converted. Which combination should be used?

- A) AWS Snowball Edge and manual SQL scripts
- B) AWS DataSync and AWS Glue
- **✅ C) AWS Schema Conversion Tool (SCT) for the schema and AWS DMS for full load plus ongoing replication (CDC)**
- D) Amazon S3 Transfer Acceleration and Athena

**Correct answer: C**

**Why each option:**
- **A ❌** Snowball moves bulk data offline and does not convert schema or keep replicating changes.
- **B ❌** DataSync copies files, not database tables. Glue is an ETL service, not a migration tool.
- **C ✅** *Heterogeneous* migration (Oracle → PostgreSQL) needs SCT to convert schema/code, and DMS to copy data + capture changes (CDC) so cutover downtime is tiny.
- **D ❌** Transfer Acceleration speeds up S3 uploads; Athena queries S3. Neither migrates databases.

💡 **Remember:** *Different engine = SCT + DMS. Same engine = DMS alone (or native tools).*

---

## Q5
An AWS Glue ETL job reads new files from an S3 prefix every day. The job re-processes **all** old files each run, wasting time and money. How can the engineer process **only new data**?

- A) Increase the number of DPUs
- B) Delete old files manually after each run
- **✅ C) Enable job bookmarks on the AWS Glue job**
- D) Convert the job to use a Python shell script

**Correct answer: C**

**Why each option:**
- **A ❌** More DPUs makes the job faster but still processes everything.
- **B ❌** Manual deletion is fragile, error-prone and you may lose data you still need.
- **C ✅** Job bookmarks remember what data was already processed, so the next run reads only new data. (Requires `transformation_ctx` in your script.)
- **D ❌** Python shell jobs are for small tasks and don't solve incremental processing.

💡 **Remember:** *Incremental processing in Glue = job bookmarks.*

---

## Q6
A data pipeline has these steps: run a Glue job → run an Athena query → send an SNS notification. The solution needs **built-in retries, error handling, and visual workflow monitoring** with **serverless** operation. Which service is BEST?

- A) Amazon MWAA with a custom DAG
- B) A chain of AWS Lambda functions calling each other
- **✅ C) AWS Step Functions**
- D) A cron job on an EC2 instance

**Correct answer: C**

**Why each option:**
- **A ❌** MWAA (managed Airflow) is powerful but is *not* serverless – you pay for the environment 24/7. Best when you already have Airflow DAGs.
- **B ❌** Lambda chaining has no visual tracking, hard error handling, and has a 15-minute limit per function.
- **C ✅** Step Functions is serverless, offers retries/catch, parallel states, visual console and direct integrations with Glue, Athena, SNS (`.sync` integration waits for the job to finish).
- **D ❌** EC2 cron means servers to patch and no built-in retry/visual tools.

💡 **Remember:** *Serverless orchestration of AWS services = Step Functions. Existing Airflow / complex DAG = MWAA.*

---

## Q7
Athena queries on a data lake are slow. The S3 prefix contains **millions of very small (a few KB) JSON files**. Which action improves performance the MOST?

- A) Increase the Athena query timeout
- **✅ B) Compact the small files into larger files (128 MB–1 GB) in a columnar format such as Parquet**
- C) Move the data to S3 Glacier Deep Archive
- D) Enable S3 versioning

**Correct answer: B**

**Why each option:**
- **A ❌** A longer timeout doesn't make queries faster.
- **B ✅** Too many small files = too many S3 requests and high overhead. Combining into larger Parquet files reduces I/O and lets Athena skip columns it doesn't need.
- **C ❌** Glacier Deep Archive is not directly queryable by Athena and would make it worse.
- **D ❌** Versioning keeps old copies; it does not improve read performance.

💡 **Remember:** *Small files problem → compact. Best formats for analytics → Parquet/ORC (columnar + compressed).*

---

## Q8
A marketing team wants to pull data from **Salesforce** into **Amazon S3** on a schedule **without writing code**. Which service should be used?

- A) AWS DataSync
- **✅ B) Amazon AppFlow**
- C) AWS Snowball
- D) Amazon Kinesis Data Streams

**Correct answer: B**

**Why each option:**
- **A ❌** DataSync moves files between storage systems (NFS, SMB, S3, EFS…), not SaaS apps.
- **B ✅** AppFlow is a no-code integration service with ready connectors for Salesforce, SAP, Slack, Zendesk, etc., and supports schedules and event triggers.
- **C ❌** Snowball is a physical device for bulk offline transfer.
- **D ❌** Kinesis is for streaming data you produce; it doesn't have SaaS connectors.

💡 **Remember:** *SaaS → AWS, no code = AppFlow.*

---

## Q9
A company needs to move **80 TB** of data from its data center to S3. The internet link is only **100 Mbps** and the transfer must finish within a few weeks. What is the MOST practical solution?

- A) Upload with the AWS CLI over the internet
- B) Use S3 Transfer Acceleration
- **✅ C) Order an AWS Snowball Edge device**
- D) Use AWS DMS

**Correct answer: C**

**Why each option:**
- **A ❌** 80 TB at 100 Mbps would take roughly 2 months or more (ideal case) – too slow.
- **B ❌** Acceleration helps with long distances, but your 100 Mbps link is still the bottleneck.
- **C ✅** Snowball Edge is shipped to you, you load data locally, and ship it back for upload to S3. Perfect for large data with weak bandwidth.
- **D ❌** DMS migrates databases, not arbitrary files.

💡 **Remember:** *Many TB + slow network = Snowball family. Online, ongoing sync = DataSync.*

---

## Q10
External partners must upload files to an S3 bucket **using SFTP** with their existing SFTP clients. The company doesn't want to manage SFTP servers. Which service fits?

- A) Amazon EC2 with an open-source SFTP server
- **✅ B) AWS Transfer Family**
- C) Amazon AppFlow
- D) AWS Direct Connect

**Correct answer: B**

**Why each option:**
- **A ❌** Works but you manage servers, patching and scaling.
- **B ✅** Transfer Family provides managed SFTP, FTPS and FTP endpoints that store files directly in S3 (or EFS).
- **C ❌** AppFlow connects SaaS apps, not SFTP clients.
- **D ❌** Direct Connect is a private network link, not a file transfer protocol server.

💡 **Remember:** *SFTP/FTPS/FTP into S3 = Transfer Family.*

---

## Q11
A company has an on-premises NFS file server. They want to **automatically and regularly copy new/changed files to S3**, with scheduling, bandwidth control and data integrity validation. Which service is MOST suitable?

- A) AWS Storage Gateway volume gateway
- **✅ B) AWS DataSync**
- C) AWS Glue
- D) Amazon Kinesis Data Firehose

**Correct answer: B**

**Why each option:**
- **A ❌** Storage Gateway gives hybrid access to cloud storage as a local drive; it is not primarily built for scheduled bulk sync tasks.
- **B ✅** DataSync uses an agent, runs on a schedule, copies only changes, validates integrity and can throttle bandwidth.
- **C ❌** Glue is for ETL, not for file replication from NFS.
- **D ❌** Firehose is for streaming records.

💡 **Remember:** *On-prem files ↔ S3/EFS/FSx on a schedule = DataSync.*

---

## Q12
A data engineer needs to automatically discover the schema of new files in S3 and make them queryable in Athena. Which feature should be used?

- A) AWS Glue DataBrew
- **✅ B) AWS Glue crawler with the AWS Glue Data Catalog**
- C) Amazon Macie
- D) AWS Lake Formation blueprints only

**Correct answer: B**

**Why each option:**
- **A ❌** DataBrew is a visual data preparation tool, not a schema-discovery catalog.
- **B ✅** A crawler scans the data, infers the schema/partitions, and creates or updates table definitions in the Glue Data Catalog, which Athena, Redshift Spectrum and EMR can all use.
- **C ❌** Macie finds sensitive data (PII), not schema.
- **D ❌** Lake Formation blueprints ingest data, and use Glue underneath; "only" makes this wrong for plain schema discovery.

💡 **Remember:** *Schema discovery + metadata = Glue crawler + Data Catalog.*

---

## Q13
A processing workflow must start **automatically whenever a new object is uploaded** to an S3 bucket, with the LEAST operational overhead. Which approach works?

- A) Poll the bucket every minute with a cron job on EC2
- **✅ B) Configure S3 Event Notifications (or EventBridge) to trigger a Lambda function or Step Functions workflow**
- C) Use AWS Batch with a scheduled job
- D) Use CloudTrail logs and manually check them

**Correct answer: B**

**Why each option:**
- **A ❌** Polling is wasteful and needs a server.
- **B ✅** S3 can emit an event on object creation to Lambda, SQS, SNS or EventBridge. This is event-driven and serverless.
- **C ❌** A scheduled job isn't event-driven and may delay processing.
- **D ❌** Manual checking is not automation.

💡 **Remember:** *Event-driven pipelines: S3 → EventBridge / Lambda / SQS / SNS.*

---

## Q14
A company already runs **Apache Kafka** applications on-premises and wants to move to AWS **without changing the application code**, while AWS manages the brokers. Which service?

- A) Amazon Kinesis Data Streams
- B) Amazon SQS
- **✅ C) Amazon MSK (Managed Streaming for Apache Kafka)**
- D) Amazon Kinesis Data Firehose

**Correct answer: C**

**Why each option:**
- **A ❌** Kinesis has a different API; applications must be rewritten.
- **B ❌** SQS is a queue, not Kafka-compatible.
- **C ✅** MSK runs real Apache Kafka, so existing producers/consumers/tools work as-is. AWS manages brokers and ZooKeeper/KRaft.
- **D ❌** Firehose is a delivery service, not Kafka.

💡 **Remember:** *Kafka-compatible = MSK. AWS-native streaming = Kinesis.*

---

## Q15
A company needs to calculate **per-minute rolling averages** from a real-time stream and write the results to another stream. They want a managed service using **SQL or Apache Flink**. Which service?

- A) AWS Glue ETL batch job
- B) Amazon Athena
- **✅ C) Amazon Managed Service for Apache Flink**
- D) AWS Lambda triggered every hour

**Correct answer: C**

**Why each option:**
- **A ❌** Glue batch jobs aren't designed for continuous low-latency windowed analytics (Glue streaming exists but Flink is the purpose-built answer here).
- **B ❌** Athena queries data at rest in S3, not live streams.
- **C ✅** Managed Flink (formerly Kinesis Data Analytics) supports windowing (tumbling, sliding), joins and aggregations on streaming data.
- **D ❌** An hourly Lambda is batch, not real-time.

💡 **Remember:** *Real-time windowed analytics = Managed Service for Apache Flink.*

---

## Q16
A streaming pipeline has **multiple producers** that occasionally change the JSON/Avro schema, breaking consumers. How can the company **enforce and evolve schemas safely**?

- A) Store schemas in a shared Word document
- **✅ B) Use AWS Glue Schema Registry**
- C) Use Amazon Macie
- D) Increase Kinesis shard count

**Correct answer: B**

**Why each option:**
- **A ❌** Manual documentation does not enforce anything.
- **B ✅** Glue Schema Registry stores versioned schemas, validates producer data and supports compatibility rules (e.g., backward compatible). It integrates with Kinesis, MSK, and Flink.
- **C ❌** Macie finds PII in S3.
- **D ❌** Shards affect throughput, not schema.

💡 **Remember:** *Streaming schema governance = Glue Schema Registry.*

---

## Q17
A data engineer loads a **large dataset from S3 into Amazon Redshift** using COPY. The load is slow because there is one single **100 GB** gzip file. What should be done?

- A) Add more sort keys
- **✅ B) Split the data into multiple files of roughly equal size (a multiple of the number of slices) and COPY from the prefix**
- C) Use INSERT statements in a loop
- D) Increase the table's VARCHAR length

**Correct answer: B**

**Why each option:**
- **A ❌** Sort keys optimize queries, not loading from one file.
- **B ✅** Redshift loads in parallel – each slice processes a file. One huge file = only one slice works. Many equal files (1 MB–1 GB compressed) = full parallelism.
- **C ❌** Row-by-row INSERT is the slowest approach in Redshift.
- **D ❌** Column length doesn't affect load parallelism.

💡 **Remember:** *COPY = parallel. Give it many files. Never loop INSERT.*

---

## Q18
An AWS Glue job is **not time-sensitive** (it can finish anytime within several hours) and the company wants the **LOWEST cost**. Which option should be chosen?

- A) G.2X workers with standard execution class
- **✅ B) Glue Flex execution class**
- C) Provision an EMR cluster with On-Demand instances
- D) Increase the number of DPUs

**Correct answer: B**

**Why each option:**
- **A ❌** Standard execution is for time-sensitive jobs and costs more.
- **B ✅** Flex uses spare capacity at a lower price (roughly 34% cheaper), ideal for non-urgent jobs like nightly or test jobs.
- **C ❌** EMR On-Demand is typically more expensive and requires more management.
- **D ❌** More DPUs = higher cost.

💡 **Remember:** *Non-urgent Glue job + save money = Flex.*

---

## Q19
A company runs transient Spark jobs on **Amazon EMR**. They want to reduce cost while keeping the cluster reliable. What is the BEST node strategy?

- A) All nodes as Spot Instances including the primary node
- **✅ B) Primary and core nodes On-Demand; task nodes as Spot Instances**
- C) All nodes as Reserved Instances
- D) Only a single large primary node

**Correct answer: B**

**Why each option:**
- **A ❌** Losing the primary node (Spot interruption) kills the cluster.
- **B ✅** Primary and core nodes store HDFS data/cluster state, so they need stability. Task nodes hold no data, so losing them is safe → cheap Spot.
- **C ❌** Reserved instances only make sense for steady 24/7 clusters, not transient jobs.
- **D ❌** No parallelism and a single point of failure.

💡 **Remember:** *EMR: primary + core = On-Demand, task = Spot.*

---

## Q20
A Lambda function that processes large files fails after **15 minutes**. Files take about 40 minutes to process with a Python/Spark transformation. What is the BEST alternative?

- A) Increase the Lambda timeout to 40 minutes
- **✅ B) Move the processing to an AWS Glue job (or ECS/Fargate task) triggered by the event**
- C) Split S3 into more buckets
- D) Increase Lambda memory to 10 GB

**Correct answer: B**

**Why each option:**
- **A ❌** The Lambda maximum timeout is 15 minutes – it can't be raised higher.
- **B ✅** Glue (serverless Spark) or Fargate can run for hours. Lambda/EventBridge/Step Functions can trigger them.
- **C ❌** Number of buckets has no effect on run time.
- **D ❌** More memory speeds things up a bit but does not remove the 15-minute cap.

💡 **Remember:** *Lambda max = 15 min. Longer = Glue / EMR / Fargate / Batch.*

---

# PART 2 – Transformation & Data Store Management (Q21–Q40)

---

## Q21
A data engineer queries 5 years of log data in S3 using Athena. Most queries filter by **date**. How can query **cost and time** be reduced the MOST?

- A) Store the data in a single large CSV file
- **✅ B) Partition the data by date (e.g., `year=/month=/day=`) and store it as Parquet**
- C) Enable S3 Transfer Acceleration
- D) Use SELECT * in all queries

**Correct answer: B**

**Why each option:**
- **A ❌** One big CSV forces Athena to scan everything. CSV is row-based and not compressed efficiently.
- **B ✅** Athena charges by data scanned. With partitions, a `WHERE date = ...` filter reads only matching folders (partition pruning). Parquet adds column pruning and compression.
- **C ❌** Acceleration is for uploads, not queries.
- **D ❌** `SELECT *` scans all columns – the opposite of cost saving.

💡 **Remember:** *Athena cost = bytes scanned. Reduce with partitions + columnar format + compression + selecting only needed columns.*

---

## Q22
An Athena table has **thousands of partitions** and queries spend a long time listing partitions. The partition values are predictable (e.g., dates and a fixed set of regions). What feature helps?

- A) MSCK REPAIR TABLE after every query
- **✅ B) Athena partition projection**
- C) Convert the table to a view
- D) Disable partition pruning

**Correct answer: B**

**Why each option:**
- **A ❌** MSCK REPAIR scans S3 to add partitions and gets slower with more partitions.
- **B ✅** Partition projection calculates partition locations from table properties instead of reading them from the catalog, so queries plan faster and you don't need crawlers/ALTER TABLE ADD PARTITION.
- **C ❌** A view doesn't change partition lookup behavior.
- **D ❌** Disabling pruning would scan all data.

💡 **Remember:** *Many predictable partitions → partition projection.*

---

## Q23
A Redshift fact table is joined often with a large dimension table on `customer_id`. Joins are slow because of **data shuffling between nodes**. What distribution style helps?

- A) EVEN on both tables
- B) ALL on the fact table
- **✅ C) KEY distribution on `customer_id` for both tables**
- D) AUTO is never useful

**Correct answer: C**

**Why each option:**
- **A ❌** EVEN spreads rows round-robin, so matching rows are on different nodes → shuffling.
- **B ❌** ALL copies the full table to every node. For a big fact table this wastes space and slows loads. (ALL is good for *small* dimension tables.)
- **C ✅** KEY puts rows with the same `customer_id` on the same slice, so the join is *co-located* – no network redistribution.
- **D ❌** AUTO is a good default (Redshift adjusts) but for a known heavy join, KEY on the join column is the targeted fix.

💡 **Remember:** *Big-to-big join = KEY on join column. Small dimension = ALL. Unsure = AUTO.*

---

## Q24
Dashboard queries in Redshift repeatedly aggregate the same huge tables and are slow. Data changes slowly. How can the engineer **speed up queries with minimal effort**?

- A) Create a regular view
- **✅ B) Create a materialized view (with auto refresh)**
- C) Delete old data
- D) Increase the VARCHAR size

**Correct answer: B**

**Why each option:**
- **A ❌** A normal view stores only the query, so it re-computes every time.
- **B ✅** A materialized view stores the precomputed result and can refresh automatically/incrementally. Redshift can even rewrite queries to use it.
- **C ❌** Might help slightly but loses data; not the real fix.
- **D ❌** Irrelevant to performance.

💡 **Remember:** *Repeated heavy aggregations = materialized view.*

---

## Q25
A company has **cold historical data in S3** (years of logs) and **hot recent data in Redshift**. Analysts want to query both with one SQL query **without loading the old data into Redshift**. What should be used?

- A) AWS DMS
- **✅ B) Amazon Redshift Spectrum with external tables**
- C) Amazon QuickSight SPICE
- D) AWS Snowball

**Correct answer: B**

**Why each option:**
- **A ❌** DMS migrates/replicates; the requirement is *not* to load the data.
- **B ✅** Spectrum lets Redshift query S3 data directly through external tables (defined in the Glue Data Catalog) and join it with local Redshift tables.
- **C ❌** SPICE is QuickSight's in-memory cache, not a query layer for Redshift.
- **D ❌** Snowball is physical transfer.

💡 **Remember:** *Query S3 from Redshift without loading = Spectrum.*

---

## Q26
A company stores log data in S3. Access patterns are **unknown and change over time**. They want automatic cost savings **without retrieval fees or manual lifecycle rules**. Which S3 storage class?

- A) S3 Standard-IA
- B) S3 One Zone-IA
- **✅ C) S3 Intelligent-Tiering**
- D) S3 Glacier Deep Archive

**Correct answer: C**

**Why each option:**
- **A ❌** Standard-IA charges per-GB retrieval fees and has no automatic movement.
- **B ❌** One Zone-IA stores in one AZ (less resilient) and also charges retrieval.
- **C ✅** Intelligent-Tiering monitors access and moves objects between tiers automatically with no retrieval charges (small monitoring fee per object).
- **D ❌** Deep Archive has hours-long retrieval; unsuitable for unknown access.

💡 **Remember:** *Unknown/changing access pattern = Intelligent-Tiering.*

---

## Q27
Logs must be kept in S3 for **7 years** for compliance. They are read often for the first 30 days, rarely after that, and almost never after 1 year. Retrieval within **hours** is acceptable for old data. Which is the MOST cost-effective approach?

- A) Keep everything in S3 Standard
- **✅ B) S3 Lifecycle rule: Standard → Standard-IA after 30 days → Glacier Flexible Retrieval (or Deep Archive) after 1 year → expire after 7 years**
- C) Delete the logs after 30 days
- D) Move logs to EBS volumes

**Correct answer: B**

**Why each option:**
- **A ❌** Works but most expensive.
- **B ✅** Lifecycle transitions match the access pattern. Archive tiers are very cheap and "hours" retrieval is acceptable.
- **C ❌** Violates the 7-year retention.
- **D ❌** EBS is block storage, more expensive and not designed for archival.

💡 **Remember:** *Predictable aging data = Lifecycle rules. Unpredictable = Intelligent-Tiering.*

---

## Q28
An application needs **single-digit millisecond** reads/writes at any scale for a key-value workload with unpredictable traffic spikes. Operations should be minimal. Which choice?

- A) Amazon RDS MySQL
- B) Amazon Redshift
- **✅ C) Amazon DynamoDB with on-demand capacity mode**
- D) Amazon Athena

**Correct answer: C**

**Why each option:**
- **A ❌** RDS can't easily scale to massive spikes with consistently low latency.
- **B ❌** Redshift is an analytics warehouse (OLAP), not for OLTP key-value access.
- **C ✅** DynamoDB is serverless, delivers single-digit ms latency, and on-demand mode scales automatically for unpredictable traffic.
- **D ❌** Athena is for ad-hoc analytics, with query times in seconds.

💡 **Remember:** *Key-value + ms latency + spiky = DynamoDB on-demand. Steady/predictable = provisioned (cheaper).*

---

## Q29
A DynamoDB table stores user sessions. Items should be **automatically deleted after 24 hours** at no extra write cost. What should be used?

- A) A Lambda function running every minute to delete items
- **✅ B) DynamoDB Time to Live (TTL) on an expiry-timestamp attribute**
- C) DynamoDB Streams
- D) Global tables

**Correct answer: B**

**Why each option:**
- **A ❌** Consumes write capacity and needs code.
- **B ✅** TTL deletes expired items in the background without consuming write capacity. Store an epoch-time attribute and enable TTL on it.
- **C ❌** Streams capture changes; they don't delete items.
- **D ❌** Global tables replicate across Regions.

💡 **Remember:** *Auto-expire items = DynamoDB TTL (free deletes).*

---

## Q30
A DynamoDB table has partition key `OrderId`. A new feature needs to query orders by `CustomerId` and sort by `OrderDate`. What is the BEST solution?

- A) Scan the entire table and filter
- **✅ B) Create a Global Secondary Index (GSI) with partition key `CustomerId` and sort key `OrderDate`**
- C) Create a Local Secondary Index with a new partition key
- D) Export the table to S3 each time

**Correct answer: B**

**Why each option:**
- **A ❌** Scan reads every item – slow and expensive.
- **B ✅** A GSI lets you query on a different partition/sort key. You can add it anytime.
- **C ❌** An LSI must use the **same partition key** as the table and must be created at table creation time.
- **D ❌** Exporting for every query is slow and unnecessary.

💡 **Remember:** *GSI = different PK, can add anytime. LSI = same PK, only at table creation.*

---

## Q31
A read-heavy application using DynamoDB needs **microsecond** read latency for repeated reads with minimal code changes. What should be added?

- A) ElastiCache for Memcached with custom caching logic
- **✅ B) Amazon DynamoDB Accelerator (DAX)**
- C) A larger partition key
- D) Read replicas

**Correct answer: B**

**Why each option:**
- **A ❌** Possible but requires you to write cache logic. DAX is API-compatible with DynamoDB.
- **B ✅** DAX is an in-memory cache designed for DynamoDB, giving microsecond reads for cached items with almost no code change.
- **C ❌** Key size doesn't affect latency.
- **D ❌** DynamoDB doesn't have read replicas in that sense.

💡 **Remember:** *DynamoDB + microsecond reads = DAX.*

---

## Q32
A company wants an **open table format on S3** supporting **ACID transactions, row-level updates/deletes, and time travel**, queryable from Athena. Which should be chosen?

- A) Plain CSV files
- B) Plain Parquet files
- **✅ C) Apache Iceberg tables**
- D) Compressed JSON

**Correct answer: C**

**Why each option:**
- **A ❌** CSV has no transactions or updates.
- **B ❌** Parquet is just a file format; to update rows you must rewrite files manually.
- **C ✅** Iceberg (also Hudi/Delta) adds a metadata layer for ACID, `MERGE/UPDATE/DELETE`, schema evolution and time travel. Athena, Glue and EMR support it.
- **D ❌** JSON has none of these features.

💡 **Remember:** *Updates/deletes on S3 data lake + time travel = Iceberg/Hudi/Delta.*

---

## Q33
An analyst needs to run a single SQL query in Athena that **joins S3 data with data in an RDS PostgreSQL database**, without moving data first. What is the right feature?

- A) Athena CTAS
- **✅ B) Athena Federated Query using a data source connector (Lambda-based)**
- C) AWS Snowball
- D) Glue crawler on RDS only

**Correct answer: B**

**Why each option:**
- **A ❌** CTAS creates new tables from query results; it doesn't connect to RDS.
- **B ✅** Federated queries use connectors (Lambda functions) to query RDS, DynamoDB, Redshift, etc., and join with S3 data in the same SQL.
- **C ❌** Physical transfer isn't relevant.
- **D ❌** A crawler only creates catalog metadata; it doesn't give live joins by itself.

💡 **Remember:** *Query across sources in place with Athena = Federated Query.*

---

## Q34
Two AWS accounts need to share **live Redshift data** (no copies, no ETL), and consumers need up-to-date data at all times. Which feature?

- A) UNLOAD to S3 and COPY into the second cluster nightly
- **✅ B) Amazon Redshift data sharing**
- C) Redshift snapshot restore every hour
- D) AWS DMS from Redshift to Redshift

**Correct answer: B**

**Why each option:**
- **A ❌** Creates copies and delays.
- **B ✅** Data sharing gives live, read access to data in another cluster/workgroup/account without copying or moving it.
- **C ❌** Snapshots are point-in-time and slow to restore.
- **D ❌** Adds replication overhead and copies.

💡 **Remember:** *Share Redshift data live without copies = data sharing.*

---

## Q35
Orders are stored in Amazon Aurora. The team wants to analyze them in Redshift **in near real time** with the LEAST engineering effort (no custom ETL pipeline). Which feature?

- A) AWS Glue job every minute
- B) Custom Lambda function with DMS
- **✅ C) Aurora zero-ETL integration with Amazon Redshift**
- D) Manual CSV export

**Correct answer: C**

**Why each option:**
- **A ❌** Needs a job to build and manage.
- **B ❌** Requires custom components to maintain.
- **C ✅** Zero-ETL integration automatically replicates transactional data into Redshift within seconds, with no pipeline to maintain.
- **D ❌** Manual and slow.

💡 **Remember:** *"No ETL pipeline" + Aurora/DynamoDB/RDS → Redshift = zero-ETL.*

---

## Q36
Streaming data in Kinesis Data Streams needs to be available in Redshift **with low latency and without staging in S3**. What is the simplest option?

- A) Firehose → S3 → COPY
- **✅ B) Redshift streaming ingestion directly from Kinesis Data Streams (or MSK) into a materialized view**
- C) Glue batch each night
- D) DMS from Kinesis

**Correct answer: B**

**Why each option:**
- **A ❌** Works but adds S3 staging and more latency.
- **B ✅** Redshift can ingest directly from KDS/MSK using an external schema and a materialized view, enabling seconds-level latency.
- **C ❌** Nightly is not low-latency.
- **D ❌** DMS doesn't use Kinesis as a source.

💡 **Remember:** *Streaming → Redshift without S3 = streaming ingestion.*

---

## Q37
Which file format is BEST for analytical queries that read only a few columns from wide tables in S3?

- A) CSV
- B) JSON
- C) Avro
- **✅ D) Parquet**

**Correct answer: D**

**Why each option:**
- **A ❌** Row-based and uncompressed by default – must read whole rows.
- **B ❌** Row-based text; bulky and slow to parse.
- **C ❌** Avro is row-based; great for streaming/schema evolution and write-heavy workloads, not column analytics.
- **D ✅** Parquet is columnar, compressed, and carries statistics, so engines read only needed columns and skip blocks.

💡 **Remember:** *Analytics (read few columns) = Parquet/ORC. Streaming/serialization with schema = Avro.*

---

## Q38
A company wants to build a **search and log analytics** solution with full-text search and dashboards on application logs. Which service?

- A) Amazon Redshift
- B) Amazon DynamoDB
- **✅ C) Amazon OpenSearch Service**
- D) Amazon Neptune

**Correct answer: C**

**Why each option:**
- **A ❌** Warehouse for SQL analytics, not full-text search.
- **B ❌** Key-value store without full-text search.
- **C ✅** OpenSearch provides full-text search, log analytics and OpenSearch Dashboards. Firehose can deliver logs to it directly.
- **D ❌** Neptune is a graph database.

💡 **Remember:** *Full-text search/log analytics = OpenSearch. Graph = Neptune. Time series = Timestream.*

---

## Q39
An application stores **IoT sensor readings** with timestamps and needs fast time-range queries, built-in time-series functions, and automatic data tiering from memory to magnetic storage. Which database?

- A) Amazon RDS for MySQL
- **✅ B) Amazon Timestream**
- C) Amazon DocumentDB
- D) Amazon Neptune

**Correct answer: B**

**Why each option:**
- **A ❌** Not optimized for massive time-series ingestion and tiering.
- **B ✅** Timestream is purpose-built for time series with a memory store and magnetic store, plus time-series SQL functions.
- **C ❌** DocumentDB is MongoDB-compatible document DB.
- **D ❌** Graph database.

💡 **Remember:** *Time series = Timestream.*

---

## Q40
A company needs a **relational** database for complex joins and transactions with **automatic scaling, MySQL compatibility, and up to 15 read replicas with fast failover**. Which service?

- A) Amazon DynamoDB
- B) Amazon Redshift
- **✅ C) Amazon Aurora**
- D) Amazon ElastiCache

**Correct answer: C**

**Why each option:**
- **A ❌** NoSQL; no complex joins/relational model.
- **B ❌** OLAP warehouse, not for OLTP transactions.
- **C ✅** Aurora is MySQL/PostgreSQL-compatible with a shared distributed storage layer, up to 15 low-lag replicas, and fast failover.
- **D ❌** In-memory cache, not a primary relational DB.

💡 **Remember:** *OLTP relational at scale = Aurora. OLAP = Redshift.*

---

# PART 3 – Data Stores, Cataloging, Data Modeling & Lifecycle (Q41–Q60)

---

## Q41
A data engineer must **clean and normalize a dataset visually** (remove nulls, change formats, standardize values) with **no code**, and then save the result to S3. Which service?

- A) AWS Glue Studio only with PySpark scripts
- **✅ B) AWS Glue DataBrew**
- C) Amazon Macie
- D) AWS Lambda

**Correct answer: B**

**Why each option:**
- **A ❌** Glue Studio has a visual editor too, but the "PySpark scripts" choice means code.
- **B ✅** DataBrew is a visual, no-code data preparation tool with 250+ built-in transformations, data profiling and recipes.
- **C ❌** Macie discovers sensitive data in S3.
- **D ❌** Lambda requires code.

💡 **Remember:** *No-code data cleaning for analysts = DataBrew.*

---

## Q42
A pipeline must **validate data quality** (e.g., column not null, values between 0 and 100, row count > 0) and fail or alert when rules are violated. Which AWS feature?

- A) Amazon Inspector
- **✅ B) AWS Glue Data Quality (rules written in DQDL)**
- C) AWS Config
- D) Amazon GuardDuty

**Correct answer: B**

**Why each option:**
- **A ❌** Inspector scans for software vulnerabilities.
- **B ✅** Glue Data Quality lets you define rules in Data Quality Definition Language (DQDL), evaluate datasets in Glue jobs or catalog tables, and emit results to CloudWatch/EventBridge.
- **C ❌** Config tracks AWS resource configuration compliance.
- **D ❌** GuardDuty detects threats.

💡 **Remember:** *Data rules and quality checks = Glue Data Quality (DQDL).*

---

## Q43
Redshift queries are slowing down because many tables have had **lots of deletes and updates**. Deleted rows still take space, and statistics are stale. What should be run?

- A) DROP TABLE and recreate manually every time
- **✅ B) VACUUM to reclaim space and re-sort rows, and ANALYZE to update statistics**
- C) Increase the S3 bucket size
- D) Disable automatic compression

**Correct answer: B**

**Why each option:**
- **A ❌** Disruptive and unnecessary.
- **B ✅** Redshift marks deleted rows but doesn't physically remove them immediately. VACUUM reclaims space and sorts; ANALYZE updates the planner's statistics. (Redshift also does auto-vacuum/auto-analyze in the background.)
- **C ❌** S3 is unrelated to Redshift internal storage.
- **D ❌** Compression is not the issue.

💡 **Remember:** *Deletes/updates leave "ghost rows" → VACUUM. Stale stats → ANALYZE.*

---

## Q44
In Redshift, a table is almost always filtered by `order_date` range (e.g., last 7 days). Which design choice speeds up these queries?

- **✅ A) Define `order_date` as the SORT KEY**
- B) Define `order_date` as DISTSTYLE ALL
- C) Remove compression
- D) Use a random column as a sort key

**Correct answer: A**

**Why each option:**
- **A ✅** Sort keys store data physically ordered. With zone maps, Redshift skips blocks outside the date range, reading much less data.
- **B ❌** DISTSTYLE ALL is a distribution setting for small tables and doesn't relate to range filtering.
- **C ❌** Compression reduces I/O; removing it hurts performance.
- **D ❌** A random sort key gives no benefit.

💡 **Remember:** *Filter/range column = SORT KEY. Join column = DIST KEY.*

---

## Q45
A Redshift cluster is fine most of the day but suffers **queuing during short peaks of many concurrent dashboard users**. The company wants to handle spikes **without permanently resizing**. What feature?

- A) Reduce the number of users
- **✅ B) Enable Concurrency Scaling**
- C) Switch to a smaller node type
- D) Turn off WLM

**Correct answer: B**

**Why each option:**
- **A ❌** Not a technical solution.
- **B ✅** Concurrency Scaling automatically adds temporary capacity to process queued read queries during spikes, then removes it.
- **C ❌** Less capacity will make queuing worse.
- **D ❌** WLM manages queues and priorities; disabling doesn't help.

💡 **Remember:** *Spiky concurrency in Redshift = Concurrency Scaling. Unpredictable workloads overall = Redshift Serverless.*

---

## Q46
A startup has **unpredictable and intermittent** analytics workloads and doesn't want to manage clusters or pay for idle time. Which is BEST?

- A) Provisioned Redshift RA3 cluster running 24/7
- **✅ B) Amazon Redshift Serverless**
- C) A self-managed data warehouse on EC2
- D) DynamoDB provisioned capacity

**Correct answer: B**

**Why each option:**
- **A ❌** Pay for idle hours.
- **B ✅** Serverless auto-provisions and scales capacity (RPUs), charging only for usage when queries run.
- **C ❌** Highest ops burden.
- **D ❌** Not a data warehouse.

💡 **Remember:** *Unpredictable / sporadic warehouse use = Redshift Serverless. Steady 24/7 = provisioned (often cheaper, reserved nodes).*

---

## Q47
A DynamoDB table has **steady and predictable** traffic. The company wants to **minimize cost** while avoiding throttling when load grows slowly during the day. Which choice?

- A) On-demand mode always
- **✅ B) Provisioned capacity with auto scaling**
- C) Reserved Lambda concurrency
- D) Increase item size

**Correct answer: B**

**Why each option:**
- **A ❌** On-demand is more expensive per request when traffic is steady and predictable.
- **B ✅** Provisioned mode is cheaper for predictable load; auto scaling adjusts RCU/WCU as traffic changes gradually.
- **C ❌** Lambda setting, not DynamoDB.
- **D ❌** Larger items consume more capacity.

💡 **Remember:** *Unpredictable → On-demand. Predictable → Provisioned + auto scaling (+ reserved capacity for more savings).*

---

## Q48
Regulations require that certain S3 objects **cannot be deleted or overwritten for 5 years**, even by the root user. Which feature?

- A) S3 Versioning only
- B) A bucket policy denying delete
- **✅ C) S3 Object Lock in Compliance mode**
- D) S3 Cross-Region Replication

**Correct answer: C**

**Why each option:**
- **A ❌** Versioning keeps old versions but permitted users can still delete versions.
- **B ❌** A policy can be changed by an admin or root.
- **C ✅** Object Lock (WORM) in *Compliance* mode: no user, including root, can delete or overwrite until retention ends. (Governance mode allows privileged users to override.)
- **D ❌** Replication copies data; it doesn't prevent deletion.

💡 **Remember:** *WORM compliance = Object Lock Compliance mode. Glacier Vault Lock is the same idea for Glacier vaults.*

---

## Q49
A data lake bucket holds critical data. The company wants protection against **accidental overwrites and deletes**, with ability to restore previous versions. What should be enabled?

- A) S3 Transfer Acceleration
- **✅ B) S3 Versioning (optionally with MFA Delete)**
- C) S3 Requester Pays
- D) S3 Inventory

**Correct answer: B**

**Why each option:**
- **A ❌** Speeds uploads only.
- **B ✅** Versioning keeps every version; a delete just adds a delete marker, so you can restore. MFA Delete adds extra protection for permanent deletions.
- **C ❌** Billing feature.
- **D ❌** Inventory generates reports of objects.

💡 **Remember:** *Accidental delete protection = Versioning.*

---

## Q50
A team wants to **cap Athena costs** for a group of analysts so one user cannot scan unlimited data, and track costs per team. What should be used?

- A) IAM password policy
- **✅ B) Athena workgroups with per-query and per-workgroup data usage limits (plus cost allocation tags)**
- C) S3 Select
- D) Glue crawlers

**Correct answer: B**

**Why each option:**
- **A ❌** Passwords have nothing to do with query cost.
- **B ✅** Workgroups isolate teams, enforce data-scanned limits, set the query result location and settings, and publish metrics to CloudWatch.
- **C ❌** S3 Select retrieves a subset from a single object, no cost governance.
- **D ❌** Crawlers create metadata.

💡 **Remember:** *Control/segregate Athena users and costs = Workgroups.*

---

## Q51
A data engineer wants to **save the result of an Athena query as a new Parquet table, partitioned**, so later queries are cheaper. Which statement?

- A) CREATE VIEW
- **✅ B) CREATE TABLE AS SELECT (CTAS) with `format = 'PARQUET'` and `partitioned_by`**
- C) MSCK REPAIR TABLE
- D) DROP TABLE

**Correct answer: B**

**Why each option:**
- **A ❌** A view stores only the query text; data stays in the original format.
- **B ✅** CTAS writes query results to S3 in a chosen format (Parquet), with partitioning and bucketing, and registers the new table in the Data Catalog.
- **C ❌** MSCK REPAIR loads partitions for existing tables.
- **D ❌** Removes a table.

💡 **Remember:** *Convert/optimize data using Athena = CTAS (and INSERT INTO for appends).*

---

## Q52
A QuickSight dashboard on a large Athena dataset is slow and costs a lot. The data updates only once a day. What improves performance and cost?

- A) Use direct query on every refresh
- **✅ B) Import the dataset into SPICE and schedule a daily refresh**
- C) Use CSV in S3 with SELECT *
- D) Disable caching

**Correct answer: B**

**Why each option:**
- **A ❌** Direct query hits Athena on every interaction → slower, more costly.
- **B ✅** SPICE is QuickSight's in-memory engine; dashboards respond quickly and the source is queried only on refresh.
- **C ❌** Worse for performance.
- **D ❌** Defeats the purpose.

💡 **Remember:** *Fast, cheap dashboards on slowly changing data = SPICE.*

---

## Q53
An analytics team needs a **star schema** for sales reporting. Which design is correct?

- A) One giant table with all columns duplicated everywhere
- **✅ B) A central fact table (measures, foreign keys) surrounded by denormalized dimension tables**
- C) Third normal form with dozens of tables joined for each query
- D) Only key-value pairs

**Correct answer: B**

**Why each option:**
- **A ❌** That's a flat table; hard to maintain and wasteful.
- **B ✅** Star schema = fact table + dimension tables (customer, product, date). It is simple and fast for analytics in Redshift.
- **C ❌** 3NF is for OLTP; too many joins hurt analytics speed. (Snowflake schema normalizes dimensions further.)
- **D ❌** Key-value isn't a warehouse model.

💡 **Remember:** *OLAP model = star schema. OLTP = normalized.*

---

## Q54
A customer's address changes, but reports must show **the address at the time of each historical order**. Which data modeling technique?

- A) Overwrite the old address (SCD Type 1)
- **✅ B) Slowly Changing Dimension Type 2 – add a new row with effective dates/current flag**
- C) Delete the customer
- D) Use a random key

**Correct answer: B**

**Why each option:**
- **A ❌** Type 1 overwrites and loses history.
- **B ✅** Type 2 keeps full history by adding a new row for each change with start/end dates and a current flag.
- **C ❌** Destroys history.
- **D ❌** Not a dimensional strategy.

💡 **Remember:** *Need history = SCD Type 2. Don't need history = Type 1.*

---

## Q55
A company wants to **replicate S3 objects to another Region** for disaster recovery and **meet an RTO of minutes**, including replicating existing objects. Which feature?

- A) S3 Lifecycle
- **✅ B) S3 Cross-Region Replication (CRR), with S3 Batch Replication for existing objects (optionally Replication Time Control)**
- C) Glacier Select
- D) S3 Inventory

**Correct answer: B**

**Why each option:**
- **A ❌** Lifecycle moves/expires objects within storage classes.
- **B ✅** CRR replicates new objects automatically; Batch Replication handles existing ones; RTC gives an SLA (99.99% of objects in 15 minutes). Versioning must be enabled.
- **C ❌** Glacier Select queries archived data.
- **D ❌** Inventory only lists objects.

💡 **Remember:** *Replication requires versioning on both buckets. Existing objects → Batch Replication.*

---

## Q56
Data in **S3 Glacier Flexible Retrieval** is needed urgently within **minutes** for an investigation. Which retrieval option?

- A) Bulk retrieval
- B) Standard retrieval
- **✅ C) Expedited retrieval**
- D) It is impossible to retrieve

**Correct answer: C**

**Why each option:**
- **A ❌** Bulk takes about 5–12 hours (cheapest).
- **B ❌** Standard takes about 3–5 hours.
- **C ✅** Expedited typically returns data in 1–5 minutes (costs more).
- **D ❌** Archive data is retrievable.

💡 **Remember:** *Glacier Flexible: Expedited (1–5 min), Standard (3–5 h), Bulk (5–12 h). Deep Archive: Standard (12 h), Bulk (48 h). Glacier Instant Retrieval = milliseconds.*

---

## Q57
A Lake Formation–governed data lake must let analysts query only **specific columns and rows** (e.g., only region = "EU") from a table. How?

- A) Create separate buckets per analyst manually
- **✅ B) Use Lake Formation data filters (column-level, row-level, cell-level security)**
- C) Use S3 ACLs
- D) Share the root credentials

**Correct answer: B**

**Why each option:**
- **A ❌** Not scalable.
- **B ✅** Lake Formation grants permissions on catalog tables down to columns, rows and cells, and enforces them for Athena, Redshift Spectrum, EMR and Glue.
- **C ❌** ACLs work only at object level.
- **D ❌** Never share root credentials.

💡 **Remember:** *Fine-grained (row/column/cell) access for data lake = Lake Formation.*

---

## Q58
A data engineer must move **tables from an on-premises PostgreSQL** into S3 as **Parquet**, including **ongoing changes**, with minimal effort. Which?

- **✅ A) DMS with a PostgreSQL source, S3 target (Parquet) and full load + CDC**
- B) Manually export CSV daily
- C) Snowball only
- D) Kinesis Data Firehose

**Correct answer: A**

**Why each option:**
- **✅ A** DMS can write to S3 in Parquet and supports CDC (changes are tagged with Op = I/U/D), keeping the lake in sync.
- **B ❌** Manual and doesn't capture changes efficiently.
- **C ❌** One-time bulk, not ongoing.
- **D ❌** Firehose receives records you push; it doesn't read from DB logs.

💡 **Remember:** *DB → S3 data lake with changes = DMS (full load + CDC), Parquet target.*

---

## Q59
A Glue Data Catalog table has **hundreds of thousands of partitions**, and queries that filter partitions are slow at planning. Which feature speeds up partition lookups?

- A) Add more Glue DPUs
- **✅ B) AWS Glue partition indexes**
- C) Use gzip for all files
- D) Disable the Data Catalog

**Correct answer: B**

**Why each option:**
- **A ❌** DPUs affect job execution, not catalog lookups.
- **B ✅** Partition indexes let the catalog filter partitions faster (used by Athena, EMR, Redshift Spectrum, Glue).
- **C ❌** Compression doesn't help partition listing.
- **D ❌** Breaks everything.

💡 **Remember:** *Huge partition counts → partition indexes (or partition projection for Athena).*

---

## Q60
An application needs **shared, POSIX-compliant file storage** that is accessed concurrently by **hundreds of Linux EC2 instances** across AZs and scales automatically. Which?

- A) Amazon EBS
- B) Amazon S3
- **✅ C) Amazon EFS**
- D) EC2 instance store

**Correct answer: C**

**Why each option:**
- **A ❌** EBS attaches to one instance (Multi-Attach has limits) and lives in one AZ.
- **B ❌** Object storage, not POSIX file system.
- **C ✅** EFS is a managed NFS file system, multi-AZ, grows/shrinks automatically, shared across many instances.
- **D ❌** Temporary and local to the instance.

💡 **Remember:** *Shared Linux file system = EFS. Windows shared = FSx for Windows. HPC/Lustre = FSx for Lustre.*

---

# PART 4 – Data Operations & Support (Q61–Q80)

---

## Q61
A Lambda function consumes a Kinesis Data Stream. The CloudWatch metric **`GetRecords.IteratorAgeMilliseconds` keeps increasing**. What does it mean and what is a good fix?

- A) Producers are sending too little data; reduce shards
- **✅ B) The consumer is falling behind; increase parallelization (e.g., parallelization factor, more shards, faster function)**
- C) The stream retention is too long; reduce it
- D) The stream is encrypted; disable encryption

**Correct answer: B**

**Why each option:**
- **A ❌** Rising iterator age means *too much* unprocessed data, not too little.
- **B ✅** Iterator age = how old the oldest unprocessed record is. If it climbs, consumers can't keep up. Fix: raise the Lambda **parallelization factor** (up to 10 concurrent batches per shard), add shards, optimize code, or use enhanced fan-out.
- **C ❌** Retention doesn't cause lag. If lag exceeds retention you actually lose data.
- **D ❌** Encryption overhead is negligible.

💡 **Remember:** *IteratorAge high = consumer lag. Watch it; if it passes retention, data is lost.*

---

## Q62
Several consumers read from the same Kinesis Data Stream and each consumer needs its **own dedicated 2 MB/s throughput per shard** with low latency (~70 ms). What should be used?

- A) Shared throughput with polling GetRecords
- **✅ B) Enhanced fan-out**
- C) Firehose buffering
- D) Increase retention to 365 days

**Correct answer: B**

**Why each option:**
- **A ❌** Shared throughput = 2 MB/s per shard split across *all* consumers.
- **B ✅** Enhanced fan-out gives each registered consumer its own 2 MB/s per shard using HTTP/2 push (SubscribeToShard).
- **C ❌** Firehose is a delivery target, not a way to raise consumer throughput.
- **D ❌** Retention doesn't affect throughput.

💡 **Remember:** *Many consumers on one stream + low latency = enhanced fan-out.*

---

## Q63
A Lambda function processes a Kinesis batch. **One bad record keeps failing**, blocking the shard and causing repeated retries. How can the pipeline continue while preserving the bad record for later?

- A) Delete the stream
- **✅ B) Configure the event source mapping with bisect-on-error, limited retry attempts/max record age, and an on-failure destination (SQS or SNS)**
- C) Increase Lambda timeout to 15 minutes
- D) Ignore all failures silently

**Correct answer: B**

**Why each option:**
- **A ❌** Destroys all data.
- **B ✅** Stream-based event source mappings retry the whole batch until success or expiry. Bisect-on-error isolates the poison record, retry/age limits stop endless loops, and the on-failure destination stores failure details for later analysis.
- **C ❌** Doesn't fix a bad record.
- **D ❌** Silently losing data is not acceptable.

💡 **Remember:** *Poison-pill record = bisect batch + max retries + on-failure destination (DLQ).*

---

## Q64
A Glue Spark job fails with **`java.lang.OutOfMemoryError`** while processing a large dataset. What should the engineer do FIRST?

- A) Switch to a Python shell job
- **✅ B) Use a larger worker type (e.g., G.2X) or more workers, and check CloudWatch job metrics for memory usage and data skew**
- C) Delete the Data Catalog
- D) Disable job bookmarks and re-run on a smaller Region

**Correct answer: B**

**Why each option:**
- **A ❌** Python shell has *less* capacity (single node) – worse for big data.
- **B ✅** G.2X workers have more memory per worker. The Glue job metrics (and Spark UI) reveal whether memory or skew is the issue; also consider grouping small files and repartitioning.
- **C ❌** The catalog is metadata and isn't the cause.
- **D ❌** Region is irrelevant to OOM.

💡 **Remember:** *Glue OOM → bigger worker type/more workers, fix skew, enable file grouping, check metrics/Spark UI.*

---

## Q65
The team wants to **receive an email whenever an AWS Glue job fails**, with the LEAST effort. What should they do?

- A) Check the console every morning
- **✅ B) Create an Amazon EventBridge rule for Glue Job State Change events (state = FAILED) that targets an Amazon SNS topic with email subscriptions**
- C) Write a cron script on EC2 to poll Glue
- D) Enable S3 versioning

**Correct answer: B**

**Why each option:**
- **A ❌** Manual and unreliable.
- **B ✅** Glue emits state-change events to EventBridge. A rule filters FAILED/TIMEOUT and sends to SNS → email/SMS/Lambda/Slack via Chatbot.
- **C ❌** Needs infrastructure and code.
- **D ❌** Irrelevant.

💡 **Remember:** *Alert on service state change = EventBridge rule → SNS.*

---

## Q66
A team needs to **search and analyze application logs in CloudWatch Logs with queries** to find the top 10 error messages in the last hour. Which feature?

- A) AWS Config
- **✅ B) CloudWatch Logs Insights**
- C) AWS X-Ray traces only
- D) Amazon Macie

**Correct answer: B**

**Why each option:**
- **A ❌** Config tracks resource configuration changes.
- **B ✅** Logs Insights has a purpose-built query language (`fields`, `filter`, `stats`, `sort`, `limit`) for interactive log analysis.
- **C ❌** X-Ray traces request paths and latency, not log text aggregation.
- **D ❌** Macie discovers sensitive data.

💡 **Remember:** *Interactive log queries in CloudWatch = Logs Insights. Logs in S3 → Athena.*

---

## Q67
An auditor asks, "**Who deleted the S3 bucket and when?**" Which service has this information?

- A) Amazon CloudWatch metrics
- **✅ B) AWS CloudTrail**
- C) AWS Glue Data Catalog
- D) Amazon Inspector

**Correct answer: B**

**Why each option:**
- **A ❌** Metrics show performance numbers, not who made API calls.
- **B ✅** CloudTrail records API calls (who, what, when, from where). For long-term queryable history use CloudTrail Lake or deliver trails to S3 and use Athena.
- **C ❌** Catalog holds metadata about data, not account activity.
- **D ❌** Inspector scans vulnerabilities.

💡 **Remember:** *Who did what in my account = CloudTrail. Performance = CloudWatch. Config history = AWS Config.*

---

## Q68
A scheduled Step Functions workflow occasionally fails because a downstream API is **temporarily unavailable**. How can the engineer make the workflow resilient?

- A) Remove the failing step
- **✅ B) Add `Retry` blocks with exponential backoff (IntervalSeconds, BackoffRate, MaxAttempts) and a `Catch` block that routes to a failure-handling state**
- C) Increase the number of DPUs
- D) Manually restart the workflow every time

**Correct answer: B**

**Why each option:**
- **A ❌** Removes required functionality.
- **B ✅** Retry handles transient errors automatically; Catch handles permanent failures by running fallback/notification steps.
- **C ❌** Unrelated to API availability.
- **D ❌** Manual effort.

💡 **Remember:** *Transient failures = Retry with backoff. Unrecoverable = Catch → handle/notify.*

---

## Q69
A pipeline may run **more than once for the same input** (retries, duplicate events). The final table must not contain duplicate rows. Which design principle?

- A) At-most-once without any tracking
- **✅ B) Make the processing idempotent (e.g., use deterministic keys with MERGE/UPSERT, or overwrite partitions)**
- C) Increase memory
- D) Disable retries

**Correct answer: B**

**Why each option:**
- **A ❌** Risks data loss.
- **B ✅** Idempotent writes produce the same result no matter how many times they run. Use unique IDs and upserts (Redshift MERGE, Iceberg MERGE INTO, DynamoDB conditional writes) or overwrite the partition.
- **C ❌** Doesn't prevent duplicates.
- **D ❌** Removing retries reduces reliability.

💡 **Remember:** *Distributed systems deliver "at least once" → design idempotent consumers.*

---

## Q70
A company runs **hundreds of existing Apache Airflow DAGs** on-premises and wants a managed service with minimal code changes. Which service?

- A) AWS Step Functions
- **✅ B) Amazon Managed Workflows for Apache Airflow (MWAA)**
- C) AWS Glue workflows
- D) Amazon EventBridge Scheduler

**Correct answer: B**

**Why each option:**
- **A ❌** Would need DAGs to be rewritten in Amazon States Language.
- **B ✅** MWAA runs open-source Airflow as a managed service, so DAGs (Python) work with little or no change.
- **C ❌** Glue workflows orchestrate Glue crawlers/jobs only.
- **D ❌** Only schedules invocations; not a DAG engine.

💡 **Remember:** *Existing Airflow = MWAA.*

---

## Q71
A Step Functions state machine needs to **process 10,000 files from S3 in parallel**, each with a Lambda/Glue task, with controlled concurrency. Which state?

- A) Wait state
- B) Pass state
- **✅ C) Map state (Distributed Map for large scale)**
- D) Choice state

**Correct answer: C**

**Why each option:**
- **A ❌** Wait just pauses.
- **B ❌** Pass passes input to output (testing/shaping data).
- **C ✅** Map runs the same steps for each item. **Distributed Map** can read items directly from S3 (list, CSV, JSON) and run up to 10,000 parallel executions with MaxConcurrency control.
- **D ❌** Choice does branching based on conditions.

💡 **Remember:** *Step Functions: Parallel = different branches at once. Map = same branch per item.*

---

## Q72
An Athena query fails with an error saying **the result is larger than expected / query exhausted resources**, because of a huge `ORDER BY` and a large join. Which tuning helps?

- A) Sort the entire dataset in one go and add DISTINCT
- **✅ B) Reduce data scanned with partition filters, select only needed columns, put the larger table on the left of joins, and avoid unnecessary ORDER BY (or add LIMIT)**
- C) Use CSV instead of Parquet
- D) Increase the S3 bucket size

**Correct answer: B**

**Why each option:**
- **A ❌** Makes it worse.
- **B ✅** Standard Athena (Trino) tuning: scan less, avoid global sorts, apply LIMIT, keep the large table on the left so the smaller one is broadcast to workers.
- **C ❌** CSV is slower and more expensive.
- **D ❌** Bucket size doesn't change compute.

💡 **Remember:** *Athena tuning = less data scanned, columnar, partitions, compress, avoid big sorts, large table left in joins.*

---

## Q73
S3 returns **HTTP 503 Slow Down** errors when thousands of concurrent requests hit the same prefix. What is the BEST approach?

- A) Move to a different Region
- **✅ B) Spread objects across multiple prefixes and use exponential backoff with retries**
- C) Switch to S3 Glacier
- D) Disable server-side encryption

**Correct answer: B**

**Why each option:**
- **A ❌** Doesn't solve per-prefix request limits.
- **B ✅** S3 supports at least 3,500 PUT/POST/DELETE and 5,500 GET/HEAD requests per second *per prefix*. Parallelize across prefixes and let SDK retry with backoff.
- **C ❌** Glacier is for archival with slow retrieval.
- **D ❌** Encryption isn't the cause.

💡 **Remember:** *S3 scales per prefix → more prefixes = more throughput.*

---

## Q74
A team needs a **repeatable, version-controlled** way to deploy Glue jobs, S3 buckets, and IAM roles across dev/test/prod. Which approach?

- A) Click through the console each time
- **✅ B) Infrastructure as Code with AWS CloudFormation, AWS CDK, or AWS SAM, deployed through a CI/CD pipeline (CodePipeline/CodeBuild)**
- C) Copy-paste JSON into emails
- D) Create resources only in production

**Correct answer: B**

**Why each option:**
- **A ❌** Not repeatable, error-prone.
- **B ✅** IaC ensures consistent environments, supports review via Git and automated testing/deployment.
- **C ❌** Not automation.
- **D ❌** Dangerous: no testing environments.

💡 **Remember:** *CloudFormation = JSON/YAML templates. CDK = code (Python/TS) that generates CloudFormation. SAM = shorthand for serverless apps.*

---

## Q75
Kinesis Data Firehose cannot deliver some records to S3 because the **Lambda transformation fails**. Where do the failed records go, and what should the engineer configure?

- A) They are silently lost
- **✅ B) Firehose writes them to an S3 error output prefix (e.g., `processing-failed/`); configure the error prefix and CloudWatch error logging to reprocess**
- C) They are sent back to the producer
- D) They are stored in DynamoDB automatically

**Correct answer: B**

**Why each option:**
- **A ❌** Firehose retains failed records in S3 so nothing is lost.
- **B ✅** Failed transformation/delivery records go to the configured error output location. Review the CloudWatch logs and re-process them.
- **C ❌** Firehose can't send data back to producers.
- **D ❌** DynamoDB isn't used for this.

💡 **Remember:** *Firehose failures → S3 error prefix + CloudWatch Logs. Delivery retry duration is configurable.*

---

## Q76
An engineer needs to understand **why a specific Redshift query is slow** and which queries are waiting in queues. Where to look?

- A) AWS CloudTrail only
- **✅ B) Redshift system tables/views (e.g., `SYS_QUERY_HISTORY`, `STL_WLM_QUERY`, `SVL_QUERY_SUMMARY`) and `EXPLAIN` plans, plus the Redshift console query monitoring**
- C) Amazon Macie
- D) S3 server access logs

**Correct answer: B**

**Why each option:**
- **A ❌** CloudTrail logs API calls, not query performance details.
- **B ✅** System views show execution time, queue time, steps and spills to disk. `EXPLAIN` shows the plan, e.g., a costly broadcast/redistribution.
- **C ❌** Not related.
- **D ❌** S3 access logs aren't Redshift query metrics.

💡 **Remember:** *Slow Redshift query → EXPLAIN + system views + check dist/sort keys, stale stats, WLM queue time.*

---

## Q77
A data engineer wants to **trace which source tables feed which reports** (data lineage) and document business definitions for the data lake. Which service helps?

- A) Amazon Inspector
- **✅ B) Amazon DataZone (data catalog, business glossary, lineage and governed data sharing)**
- C) Amazon SNS
- D) AWS Shield

**Correct answer: B**

**Why each option:**
- **A ❌** Vulnerability scanner.
- **B ✅** DataZone (together with the Glue Data Catalog) lets teams publish, discover, subscribe to and govern data assets with business context and lineage. (SageMaker Catalog/Unified Studio builds on the same ideas.)
- **C ❌** Messaging.
- **D ❌** DDoS protection.

💡 **Remember:** *Business catalog + data sharing/governance portal = DataZone. Technical metadata = Glue Data Catalog.*

---

## Q78
A job pulls data via **JDBC from an RDS database in a private subnet** using Glue. The job fails with network timeouts. What must be configured?

- A) Make the RDS database public
- **✅ B) Create a Glue connection with the VPC, subnet and security group; make sure the security group allows traffic, and the subnet has a route to S3 (VPC endpoint/NAT) and other needed services**
- C) Increase DPUs
- D) Use S3 Transfer Acceleration

**Correct answer: B**

**Why each option:**
- **A ❌** Exposes the database – bad security practice.
- **B ✅** Glue runs ENIs in your VPC through the connection. Needs a self-referencing security group rule, access to the DB port, and either an S3 gateway endpoint or NAT to reach S3 and Glue endpoints.
- **C ❌** Compute capacity doesn't fix networking.
- **D ❌** Unrelated.

💡 **Remember:** *Glue → private resources: Glue connection + VPC settings + S3 VPC endpoint/NAT.*

---

## Q79
Athena is queried repeatedly with **identical queries** from a dashboard on data that rarely changes. How can cost and latency be reduced without changing the queries?

- A) Delete the results bucket
- **✅ B) Enable Athena query result reuse**
- C) Turn off partitions
- D) Increase file count

**Correct answer: B**

**Why each option:**
- **A ❌** Would remove cached results.
- **B ✅** Query result reuse returns a recent identical result instead of rescanning data (you choose the max age), saving cost and time.
- **C ❌** Hurts performance.
- **D ❌** More files = slower.

💡 **Remember:** *Repeated identical Athena queries = result reuse.*

---

## Q80
A nightly Glue job occasionally runs **much longer than usual**. The team wants an **automatic alert** if a job runs over 2 hours. What is the simplest option?

- A) Look at logs every morning
- **✅ B) Set the Glue job's timeout and a CloudWatch alarm / EventBridge rule on job state (TIMEOUT/FAILED) or job-run duration metrics, notifying via SNS**
- C) Remove the job
- D) Use S3 Replication

**Correct answer: B**

**Why each option:**
- **A ❌** Manual.
- **B ✅** The job timeout stops runaway jobs (state = TIMEOUT) and EventBridge/CloudWatch can alert you through SNS. Glue also supports job run duration notification thresholds.
- **C ❌** Doesn't solve anything.
- **D ❌** Irrelevant.

💡 **Remember:** *Monitoring = CloudWatch metrics/alarms + EventBridge events + SNS notifications.*

---

# PART 5 – Data Security & Governance (Q81–Q100)

---

## Q81
A Glue job needs to read from one S3 bucket and write to another. What is the BEST way to give it permissions?

- A) Embed IAM user access keys in the script
- **✅ B) Attach an IAM role to the Glue job with least-privilege permissions to only those buckets/prefixes**
- C) Make both buckets public
- D) Use the root account's credentials

**Correct answer: B**

**Why each option:**
- **A ❌** Hard-coded keys can leak and must be rotated manually.
- **B ✅** AWS services assume IAM roles to get temporary credentials. Grant only what is needed (least privilege).
- **C ❌** Public buckets expose data to everybody.
- **D ❌** Never use root for workloads.

💡 **Remember:** *AWS service → AWS resource = IAM role (temporary credentials), never long-term keys in code.*

---

## Q82
A data pipeline connects to a **database whose password must be rotated automatically every 30 days**, without code changes. Where should credentials be stored?

- A) In plain text in the Glue script
- B) In an S3 object
- **✅ C) AWS Secrets Manager with automatic rotation enabled**
- D) In environment variables in a public GitHub repo

**Correct answer: C**

**Why each option:**
- **A ❌** Plain text secrets are a security risk.
- **B ❌** Not designed for secrets and no rotation.
- **C ✅** Secrets Manager stores secrets encrypted, supports built-in rotation for RDS/Redshift/DocumentDB (via Lambda), and applications fetch the current value at runtime.
- **D ❌** Publicly exposing secrets is the worst option.

💡 **Remember:** *Auto-rotating DB credentials = Secrets Manager. Simple config/non-rotating = SSM Parameter Store (cheaper).*

---

## Q83
Company policy says S3 data must be encrypted with keys the company **controls, can audit, and can disable**. Which encryption option fits?

- A) SSE-S3 (Amazon S3 managed keys)
- **✅ B) SSE-KMS with a customer managed KMS key**
- C) No encryption but private bucket
- D) Client-side encoding with Base64

**Correct answer: B**

**Why each option:**
- **A ❌** Keys are fully managed by S3; you can't control or audit key usage.
- **B ✅** With customer managed KMS keys you control key policies, can see every use in CloudTrail, rotate keys, and disable/delete a key to block access.
- **C ❌** Private ≠ encrypted.
- **D ❌** Base64 is encoding, not encryption.

💡 **Remember:** *Control + audit + disable keys = SSE-KMS with CMK. Simplest default = SSE-S3.*

---

## Q84
Traffic from Glue jobs in a private subnet to S3 **must not traverse the public internet**, and the company wants the **lowest cost**. What should be created?

- A) NAT gateway
- B) Internet gateway
- **✅ C) VPC gateway endpoint for Amazon S3**
- D) AWS Direct Connect

**Correct answer: C**

**Why each option:**
- **A ❌** NAT sends traffic out to the internet path and costs per hour and per GB.
- **B ❌** Direct internet access violates the requirement.
- **C ✅** S3 gateway endpoints keep traffic on the AWS network and are **free** (you pay only normal S3 charges). They are added to route tables.
- **D ❌** Dedicated connection from on-premises; unnecessary and costly.

💡 **Remember:** *S3 and DynamoDB = gateway endpoints (free). Most other services = interface endpoints (PrivateLink, paid).*

---

## Q85
The security team wants to **automatically find PII** (names, credit-card numbers) stored in S3 buckets across the organization. Which service?

- A) Amazon GuardDuty
- B) AWS Shield
- **✅ C) Amazon Macie**
- D) AWS WAF

**Correct answer: C**

**Why each option:**
- **A ❌** GuardDuty detects threats and suspicious activity.
- **B ❌** DDoS protection.
- **C ✅** Macie uses ML and pattern matching to discover and classify sensitive data in S3 and reports findings (can integrate with Security Hub/EventBridge).
- **D ❌** Web application firewall.

💡 **Remember:** *Sensitive data discovery in S3 = Macie. Threat detection = GuardDuty. Vulnerabilities = Inspector.*

---

## Q86
In a Redshift table, support staff must be able to query `credit_card` but see only the **last 4 digits**, while auditors see the full value. Which feature?

- A) Create two copies of the table manually
- **✅ B) Redshift dynamic data masking policies attached to roles**
- C) Delete the column
- D) Encrypt the whole cluster only

**Correct answer: B**

**Why each option:**
- **A ❌** Duplicates sensitive data and is hard to maintain.
- **B ✅** Dynamic data masking changes the *displayed* value at query time based on role/user, with no data duplication.
- **C ❌** Auditors need access.
- **D ❌** Cluster encryption protects data at rest; it doesn't hide values from authorized users.

💡 **Remember:** *Show different views of the same column by role = dynamic data masking. Row filtering = row-level security.*

---

## Q87
A company wants to **share a Glue Data Catalog database with another AWS account** and control access centrally down to the table/column level. What is the recommended method?

- A) Email CSV exports
- **✅ B) AWS Lake Formation cross-account grants (using AWS RAM resource links)**
- C) Share IAM user passwords
- D) Make the S3 buckets public

**Correct answer: B**

**Why each option:**
- **A ❌** Copies data and loses governance.
- **B ✅** Lake Formation grants permissions to another account (or organization/OU). The recipient creates a resource link in their catalog and queries via Athena/Redshift Spectrum, with fine-grained control.
- **C ❌** Sharing passwords is a security violation.
- **D ❌** Public = no control.

💡 **Remember:** *Cross-account lake sharing = Lake Formation + RAM.*

---

## Q88
Hundreds of tables are tagged `classification=confidential` or `public`. Instead of granting access table by table, the company wants **scalable permissions based on tags**. Which Lake Formation feature?

- A) Named resource grants only
- **✅ B) Lake Formation tag-based access control (LF-TBAC) with LF-tags**
- C) S3 ACLs
- D) Security groups

**Correct answer: B**

**Why each option:**
- **A ❌** Named grants work but don't scale for hundreds of tables.
- **B ✅** LF-Tags are attached to databases, tables and columns. Permissions are granted to principals on tag expressions, so new tables with the same tag are covered automatically.
- **C ❌** Object-level only, not catalog aware.
- **D ❌** Network firewalls, not data permissions.

💡 **Remember:** *Scale permissions across many tables = LF-tags (tag-based access control).*

---

## Q89
A bucket policy must **reject any request that does not use HTTPS**. Which condition is used?

- A) `"aws:SourceIp"`
- **✅ B) `"Bool": {"aws:SecureTransport": "false"}` with an explicit Deny**
- C) `"aws:MultiFactorAuthPresent"`
- D) `"s3:x-amz-acl"`

**Correct answer: B**

**Why each option:**
- **A ❌** Restricts by IP, not by protocol.
- **B ✅** `aws:SecureTransport` is false when the request is over HTTP. An explicit Deny on that condition forces TLS (encryption in transit).
- **C ❌** Checks MFA usage.
- **D ❌** Controls ACL headers.

💡 **Remember:** *Enforce encryption in transit on S3 = Deny when aws:SecureTransport = false.*

---

## Q90
**Choose TWO.** Which actions help ensure data in an Amazon Redshift cluster is protected **at rest and in transit**?

- **✅ A) Enable cluster encryption using AWS KMS**
- B) Make the cluster publicly accessible with open security group
- **✅ C) Require SSL connections (`require_ssl` parameter)**
- D) Disable audit logging
- E) Share the admin password with all analysts

**Correct answers: A and C**

**Why each option:**
- **A ✅** Encryption at rest with KMS (or HSM) protects data blocks, backups and snapshots.
- **B ❌** Public access with open rules increases attack surface.
- **C ✅** Forcing SSL encrypts data in transit between clients and cluster.
- **D ❌** Audit logging should be enabled (to S3/CloudWatch) for compliance.
- **E ❌** Violates least privilege.

💡 **Remember:** *At rest = KMS. In transit = TLS/SSL. Also: private subnets, IAM auth, audit logs.*

---

## Q91
The company must **record every read/write of objects in a sensitive S3 bucket** for audit (object-level activity). What should be enabled?

- A) CloudTrail management events only
- **✅ B) CloudTrail data events for S3 on that bucket**
- C) S3 Transfer Acceleration
- D) VPC Flow Logs only

**Correct answer: B**

**Why each option:**
- **A ❌** Management events log control-plane actions (CreateBucket, PutBucketPolicy) but not object-level GetObject/PutObject.
- **B ✅** Data events capture object-level API activity (GetObject, PutObject, DeleteObject). They are off by default and cost extra. (S3 server access logs are another option.)
- **C ❌** Speeds uploads.
- **D ❌** Flow logs show network traffic, not S3 object operations.

💡 **Remember:** *Object-level S3 audit = CloudTrail data events (or S3 server access logs).*

---

## Q92
Under privacy regulation, a company must **delete a specific user's records** from a large S3 data lake in Parquet. What is the MOST efficient approach?

- A) Rewrite the entire data lake manually using scripts every time
- **✅ B) Store the data in Apache Iceberg (or Hudi/Delta) tables and use `DELETE FROM ... WHERE user_id = ...` (followed by compaction/expire snapshots)**
- C) Ignore the request
- D) Delete the whole bucket

**Correct answer: B**

**Why each option:**
- **A ❌** Expensive and error-prone.
- **B ✅** Table formats like Iceberg support row-level deletes. Remember to expire old snapshots/remove old data files so deleted data is physically gone.
- **C ❌** Non-compliant.
- **D ❌** Destroys everything.

💡 **Remember:** *Right to be forgotten at scale = row-level deletes with Iceberg/Hudi/Delta.*

---

## Q93
A Lambda function in **Account A** must read from an S3 bucket encrypted with a **customer managed KMS key** in **Account B**. What is needed?

- A) Only an S3 bucket policy in Account B
- **✅ B) Permission on both: the bucket policy (or role trust) for S3 access AND the KMS key policy in Account B allowing Account A's role, plus IAM permissions on the Lambda role for S3 and KMS (`kms:Decrypt`)**
- C) Make the KMS key public
- D) Nothing; cross-account works automatically

**Correct answer: B**

**Why each option:**
- **A ❌** S3 access alone isn't enough; the caller must also be allowed to use the KMS key to decrypt.
- **B ✅** Cross-account needs permissions at *both ends*: the resource side (bucket policy + KMS key policy) and the identity side (IAM policy on the role).
- **C ❌** There's no such thing as a public KMS key.
- **D ❌** Cross-account access is always explicit.

💡 **Remember:** *Cross-account + KMS = key policy + IAM policy + resource policy. Missing KMS permission = AccessDenied on encrypted objects.*

---

## Q94
A company wants **centralized, long-term, queryable audit logs** of API activity across all accounts in AWS Organizations using SQL, without building a pipeline. Which feature?

- A) CloudWatch dashboards
- **✅ B) AWS CloudTrail Lake (or an organization trail to S3 queried with Athena)**
- C) Amazon SNS
- D) AWS Direct Connect

**Correct answer: B**

**Why each option:**
- **A ❌** Dashboards visualize metrics.
- **B ✅** CloudTrail Lake stores events in managed event data stores and lets you run SQL queries; an organization trail delivers all accounts' events to one S3 bucket where Athena can query them.
- **C ❌** Messaging service.
- **D ❌** Network connectivity.

💡 **Remember:** *Audit logs + SQL = CloudTrail Lake or CloudTrail → S3 → Athena.*

---

## Q95
Different teams should access **only their own prefix** in a shared S3 bucket, each with distinct network and policy requirements, without one huge bucket policy. Which feature simplifies this?

- A) S3 Object Lock
- **✅ B) S3 Access Points (each with its own policy and optional VPC restriction)**
- C) S3 Select
- D) S3 Transfer Acceleration

**Correct answer: B**

**Why each option:**
- **A ❌** WORM retention.
- **B ✅** Access points give each application/team a unique endpoint and a dedicated policy, making permissions manageable for shared datasets.
- **C ❌** Query feature.
- **D ❌** Upload speed.

💡 **Remember:** *Many teams, one big shared bucket = S3 Access Points.*

---

## Q96
An engineer must give a **contractor temporary** access to run Athena queries for 8 hours, without creating a permanent IAM user. What is BEST?

- A) Create an IAM user with an unlimited password
- **✅ B) Provide an IAM role the contractor can assume via AWS STS (or IAM Identity Center permission set) with a limited session duration**
- C) Share an admin user
- D) Use root account

**Correct answer: B**

**Why each option:**
- **A ❌** Permanent credentials can be forgotten and leaked.
- **B ✅** STS issues temporary credentials that expire automatically; permissions are limited by the role policy.
- **C ❌** Violates least privilege.
- **D ❌** Never share root.

💡 **Remember:** *Temporary/external/cross-account access = IAM roles + STS.*

---

## Q97
An organization wants to **prevent any account** from creating resources outside approved Regions (data residency). What is the best control?

- A) IAM user policy in each account
- **✅ B) Service control policy (SCP) in AWS Organizations that denies actions outside approved Regions (using `aws:RequestedRegion`)**
- C) A CloudWatch alarm
- D) S3 Lifecycle

**Correct answer: B**

**Why each option:**
- **A ❌** Account admins could change/ignore them; it doesn't scale.
- **B ✅** SCPs set maximum permissions for accounts in the organization. Even account admins can't exceed them.
- **C ❌** Alarm only notifies after the fact.
- **D ❌** Unrelated.

💡 **Remember:** *Organization-wide guardrails = SCPs (they restrict; they don't grant).*

---

## Q98
**Choose TWO.** Which steps help ensure **Kinesis Data Streams** data is secure?

- **✅ A) Enable server-side encryption with a KMS key on the stream**
- B) Allow `kinesis:*` to all principals (`"Principal": "*"`)
- **✅ C) Use a VPC interface endpoint (PrivateLink) so producers/consumers in a VPC reach Kinesis privately and apply least-privilege IAM policies**
- D) Turn off CloudTrail
- E) Share access keys in the app code

**Correct answers: A and C**

**Why each option:**
- **A ✅** Server-side encryption protects data at rest in the stream.
- **B ❌** Wide-open permissions are dangerous.
- **C ✅** Interface endpoints keep traffic private; IAM least privilege restricts who can put/get records.
- **D ❌** CloudTrail records API usage and is needed for auditing.
- **E ❌** Hard-coded keys are insecure; use roles.

💡 **Remember:** *Secure streaming = SSE-KMS + TLS + VPC endpoint + least-privilege IAM.*

---

## Q99
A data analyst must access Redshift using **corporate single sign-on (SSO)** credentials and have **permissions mapped from identity provider groups**. Which approach is recommended?

- A) Create local database users with shared passwords
- **✅ B) Integrate Redshift with IAM Identity Center (or an external IdP using SAML/federation) so users sign in with corporate identities**
- C) Use the cluster admin user
- D) Open the cluster to the public internet

**Correct answer: B**

**Why each option:**
- **A ❌** Shared passwords are hard to audit and rotate.
- **B ✅** Federated identity centralizes authentication and group-based access, and lets you audit individual users. Redshift supports IAM Identity Center and IdP federation.
- **C ❌** Overly privileged.
- **D ❌** Security risk.

💡 **Remember:** *Corporate SSO + AWS analytics = IAM Identity Center / SAML federation. Avoid shared DB users.*

---

## Q100
A company must prove **data was not tampered with and track configuration changes of AWS resources** (e.g., someone made a bucket public). Which service continuously **evaluates resource configuration against rules**?

- **✅ A) AWS Config**
- B) Amazon Athena
- C) AWS Glue
- D) Amazon QuickSight

**Correct answer: A**

**Why each option:**
- **✅ A** AWS Config records configuration history of resources and evaluates them against rules (e.g., `s3-bucket-public-read-prohibited`), flagging non-compliance and optionally auto-remediating with Systems Manager.
- **B ❌** Athena queries data; it doesn't track resource configuration.
- **C ❌** Glue is an ETL service.
- **D ❌** QuickSight visualizes data.

💡 **Remember:** *Config = what changed/is it compliant? CloudTrail = who did the API call? CloudWatch = how is it performing?*

---

# 🎯 Quick Revision Cheat-Sheet (Keyword → Answer)

| If the question says... | Think... |
|---|---|
| Near real-time delivery to S3/Redshift/OpenSearch, no code | **Kinesis Data Firehose** |
| Custom real-time processing, ordering per key, replay | **Kinesis Data Streams** |
| Existing Kafka apps | **Amazon MSK** |
| Real-time windowed analytics | **Managed Service for Apache Flink** |
| Serverless orchestration | **Step Functions** |
| Existing Airflow DAGs | **MWAA** |
| SaaS → S3, no code | **AppFlow** |
| On-prem files → S3 on a schedule | **DataSync** |
| SFTP into S3 | **Transfer Family** |
| Petabytes/TBs, poor network | **Snowball Edge** |
| DB migration + CDC | **DMS (+ SCT if engine changes)** |
| Schema discovery | **Glue Crawler + Data Catalog** |
| Process only new data in Glue | **Job bookmarks** |
| Non-urgent cheap Glue | **Glue Flex** |
| No-code data cleaning | **Glue DataBrew** |
| Data quality rules | **Glue Data Quality (DQDL)** |
| Streaming schema governance | **Glue Schema Registry** |
| Serverless SQL on S3 | **Athena** (partition, Parquet, compress, workgroups) |
| Query S3 from Redshift | **Redshift Spectrum** |
| Spiky Redshift concurrency | **Concurrency Scaling** |
| Sporadic warehouse workload | **Redshift Serverless** |
| Join column / filter column in Redshift | **DIST KEY / SORT KEY** |
| Aurora/DynamoDB → Redshift without ETL | **Zero-ETL** |
| Key-value, ms latency | **DynamoDB** (+ **DAX** for µs reads, **TTL** for expiry) |
| Time series | **Timestream** |
| Full-text search / logs | **OpenSearch** |
| ACID + updates on S3 | **Iceberg / Hudi / Delta** |
| Unknown access pattern | **S3 Intelligent-Tiering** |
| WORM / legal retention | **S3 Object Lock (Compliance mode)** |
| Row/column/cell security in lake | **Lake Formation** (LF-tags for scale) |
| Find PII in S3 | **Macie** |
| Rotate DB credentials | **Secrets Manager** |
| Control and audit keys | **KMS customer managed key** |
| Private S3 access, cheapest | **S3 gateway VPC endpoint** |
| Who did what (API calls) | **CloudTrail** |
| Is resource compliant / config history | **AWS Config** |
| Org-wide guardrails | **SCPs** |

**Good luck with your AWS Certified Data Engineer – Associate exam! 🚀**
