# AWS Certified Data Engineer – Associate (DEA-C01)
## Complete In-Depth Notes (Easy Language + Real-World Scenarios)

> **How to use these notes:** Read each service in this order: *What it is → Key concepts → Numbers to remember → When to use / not use → Real-world scenario → Exam traps.*
> At the end there are **decision tables, keyword maps, troubleshooting playbooks and practice questions** so you can solve *any* scenario question.
> Real-world examples use a **bank / insurance (BFSI)** flavour because that is a very common data-engineering setting.

---

# 0. EXAM OVERVIEW

| Item | Detail |
|---|---|
| Exam code | DEA-C01 |
| Questions | 65 (50 scored + 15 unscored), multiple choice + multiple response |
| Time | 130 minutes |
| Passing score | 720 / 1000 |
| Style | Scenario-based: "Which solution is MOST cost-effective / LEAST operational overhead / MOST secure / lowest latency?" |

### Domain weights

| Domain | Weight | What it really tests |
|---|---|---|
| 1. Data Ingestion & Transformation | **34%** | Kinesis, Firehose, MSK, DMS, Glue, EMR, Lambda, Step Functions, Airflow, formats, streaming vs batch |
| 2. Data Store Management | **26%** | S3, Redshift, DynamoDB, RDS/Aurora, Glue Catalog, lifecycle, data modeling, schema evolution |
| 3. Data Operations & Support | **22%** | Athena, QuickSight, CloudWatch, data quality, troubleshooting, automation, SQL |
| 4. Data Security & Governance | **18%** | IAM, KMS, Lake Formation, Macie, encryption, VPC, audit logs, privacy |

### The 6 "golden rules" for choosing answers

1. **"Least operational overhead"** → pick **serverless / fully managed** (Athena, Glue, Firehose, Lambda, Step Functions, Redshift Serverless, EMR Serverless, DynamoDB on-demand).
2. **"Cost-effective"** → S3 + Parquet + partitioning + lifecycle + Spot/Flex/serverless. Avoid always-on clusters for occasional jobs.
3. **"Real-time / sub-second"** → Kinesis Data Streams / MSK / DynamoDB / ElastiCache. **"Near real-time" (≈1 min)** → Firehose.
4. **"Replay / multiple consumers / ordering"** → Kinesis Data Streams or MSK (NOT Firehose, NOT plain SQS).
5. **"Fine-grained access (column/row/cell) on data lake"** → **Lake Formation**.
6. **"Encrypt / secret / rotate credentials"** → KMS + Secrets Manager. Never hard-code credentials.

---

# 1. BIG PICTURE – THE MODERN AWS DATA PLATFORM

```
SOURCES                INGEST                  STORE / LAKE               PROCESS                 SERVE / CONSUME
-------                ------                  ------------               -------                 ---------------
Core banking DB  --->  DMS (CDC)  ----------->                                                     
Apps / clickstream ->  Kinesis Streams ------>  S3 (raw / bronze)  ---->   Glue ETL (Spark)  --->  S3 (curated / silver / gold)
IoT / logs      --->  Firehose  ------------>                              EMR / Lambda            |
SaaS (Salesforce) -->  AppFlow  ------------>  Glue Data Catalog <-------  Step Functions /        +--> Athena (ad-hoc SQL)
Files (SFTP)    --->  Transfer Family ------>  (metadata)                  Airflow (orchestrate)   +--> Redshift (warehouse / BI)
On-prem files   --->  DataSync / Snowball -->  Lake Formation (security)                           +--> QuickSight (dashboards)
Kafka producers --->  MSK  ----------------->                                                       +--> OpenSearch / DynamoDB (apps)
                                  CloudWatch + CloudTrail + KMS + IAM wrap around everything
```

**Data lake zones (medallion architecture):**
- **Raw / Bronze** – data exactly as received (immutable, cheap, kept for replay & audit).
- **Cleaned / Silver** – validated, deduplicated, standard types, Parquet.
- **Curated / Gold** – business-ready aggregates, star schemas, feature tables.

**Batch vs Streaming:**

| | Batch | Micro-batch | Streaming |
|---|---|---|---|
| Latency | hours | seconds–minutes | milliseconds–seconds |
| AWS tools | Glue, EMR, Redshift COPY | Firehose, Spark Structured Streaming | Kinesis Streams + Lambda/Flink, MSK |
| Use | nightly reports, regulatory | near-real-time dashboards | fraud detection, alerts |

---

# 2. DOMAIN 1 — DATA INGESTION AND TRANSFORMATION (34%)

## 2.1 Ingestion fundamentals (must-know vocabulary)

| Term | Easy meaning |
|---|---|
| **Throughput** | How much data per second you can push (MB/s or records/s). |
| **Latency** | Delay between data created and data usable. |
| **Replayability** | Ability to re-read old data again (e.g., after a bug). Kinesis Streams / Kafka can; Firehose/SQS cannot. |
| **Idempotency** | Running the same operation twice gives the same result (no duplicates). Critical for retries. |
| **Backpressure** | Producer is faster than consumer; something must buffer/throttle. |
| **Fan-out** | One stream, many consumers each reading it independently. |
| **Push vs Pull** | Push: source sends to target (Firehose, SNS). Pull: consumer polls (Kinesis classic, SQS, Kafka). |
| **CDC** | Change Data Capture – capture only INSERT/UPDATE/DELETE from DB logs. |
| **Exactly-once / at-least-once** | Most AWS streaming is **at-least-once** → design for duplicates (idempotent writes, dedupe keys). |
| **Throttling** | Service rejects requests above limits → use **retry with exponential backoff + jitter**. |

---

## 2.2 Amazon Kinesis Data Streams (KDS)

**What it is (easy):** A durable "conveyor belt" for real-time data. Producers put records on it; many consumers can read them, in order, and even re-read them for a while.

**Key concepts**
- **Shard** = unit of capacity. A stream = N shards.
  - **Write per shard:** 1 MB/s **or** 1,000 records/s.
  - **Read per shard (shared/classic):** 2 MB/s total shared by all consumers; 5 `GetRecords` calls/s per shard.
  - **Enhanced Fan-Out (EFO):** each registered consumer gets its **own 2 MB/s per shard**, pushed via HTTP/2 (`SubscribeToShard`), ~70 ms latency.
- **Partition key** → hashed to decide the shard. Same key = same shard = **ordering preserved per key**.
- **Sequence number** – unique per record in a shard.
- **Record size limit:** 1 MB (data blob).
- **Retention:** 24 hours default → extendable up to **365 days**. Records can be **replayed**.
- **Capacity modes:** **Provisioned** (you choose shards; you can split/merge) vs **On-demand** (auto-scales; pay per GB; no shard planning).
- **Producers:** Kinesis Producer Library (KPL – batching, aggregation, retries), AWS SDK `PutRecord/PutRecords`, Kinesis Agent, CloudWatch Logs subscription, IoT Core rules.
- **Consumers:** Kinesis Client Library (KCL – uses **DynamoDB table** for checkpointing & lease), Lambda (event source mapping), Firehose, Managed Service for Apache Flink, Glue streaming, Spark on EMR.
- **Resharding:** Split a hot shard (more capacity); merge cold shards (save cost).
- **Security:** SSE with KMS, TLS, VPC endpoint (interface), IAM.

**Common errors**
- `ProvisionedThroughputExceededException` → hot shard or too few shards. Fix: better partition key (high cardinality, random suffix), add shards (split), use on-demand, use KPL aggregation, exponential backoff.
- Consumer lag → watch **`GetRecords.IteratorAgeMilliseconds`** (how old the data being read is). High = consumer too slow → add consumers/shards, enhanced fan-out, increase Lambda parallelization factor.

**Lambda + Kinesis settings (very exam-relevant)**
- **Batch size** and **batch window** – how many records / how long to wait before invoking.
- **Parallelization factor** (1–10) – concurrent Lambda per shard.
- **Bisect batch on function error** – split a failing batch to find the bad record.
- **Maximum retry attempts / maximum record age** and **on-failure destination (SQS/SNS/S3)** – to avoid one poison record blocking a shard forever.
- **Tumbling windows** – simple aggregations in Lambda.

**When to use:** real-time analytics, multiple consumers, ordering, replay, custom processing.
**When NOT:** just landing data in S3 with no code → **Firehose** is simpler and cheaper.

**Real-world scenario (Bank):** A bank's mobile app emits card-transaction events. KDS ingests them (partition key = `account_id` so each account's events stay ordered). One consumer is a **Lambda** doing fraud scoring (needs sub-second). Another consumer (**Firehose**) lands all events to S3 for the data lake. A third (**Flink**) computes rolling 5-minute spend per account. EFO is enabled so the three consumers don't fight over the 2 MB/s shard read limit.

**Exam traps**
- Order is guaranteed **per shard**, not across the whole stream.
- Firehose is NOT a Kinesis Streams consumer replacement when you need replay.
- KCL needs DynamoDB → throttled DynamoDB can slow your consumer.
- "Multiple consumers each need full throughput" → **Enhanced fan-out**.

---

## 2.3 Amazon Data Firehose (formerly Kinesis Data Firehose)

**What it is (easy):** A fully managed "delivery truck" – takes streaming data and delivers it to a destination. No code, no shards, auto-scales. **Near real-time**, not real-time.

**Key concepts**
- **Sources:** direct PUT (SDK/agent), Kinesis Data Streams, MSK, CloudWatch Logs/Events, IoT, WAF logs.
- **Destinations:** **S3**, **Redshift** (it stages in S3 then runs `COPY`), **OpenSearch**, Splunk, HTTP endpoint, Snowflake, Apache Iceberg tables, 3rd-party (Datadog, etc.).
- **Buffering:** delivers when **buffer size** (MBs) **or buffer interval** (seconds, up to 900) is reached first. A zero-buffer option exists for some destinations for lower latency.
- **Transformation:** attach a **Lambda** to modify/filter/enrich records (record must return `recordId`, `result` = Ok / Dropped / ProcessingFailed, `data`).
- **Format conversion:** JSON → **Parquet/ORC** using a schema from the **Glue Data Catalog**.
- **Dynamic partitioning:** writes to S3 prefixes like `year=2026/month=10/customer=XYZ/` from record content.
- **Compression:** GZIP, Snappy, ZIP, Hadoop-Snappy.
- **Error handling:** failed records go to an S3 **error prefix**; backup of source records can be enabled.
- **No replay**, no consumer ordering guarantee.

**When to use:** log/clickstream → S3/Redshift/OpenSearch with least effort.
**Real-world scenario (Insurance):** Web-portal clickstream for policy-purchase journeys → Firehose → Lambda masks PII (hash customer ID) → converts to Parquet → S3 partitioned by date → Athena for marketing analysis. Zero servers to manage.

**Exam traps**
- Firehose to Redshift = S3 intermediate + COPY (you need an S3 bucket in config).
- "Real-time" requirement → not Firehose (it is *near* real-time).
- Convert to Parquet inside Firehose requires a **Glue table** for schema.

---

## 2.4 Amazon Managed Service for Apache Flink (formerly Kinesis Data Analytics)

**What it is:** Run **Apache Flink** (Java/Scala/Python/SQL) as a managed service to do **stateful stream processing** – windowing, joins, aggregations, anomaly detection on streams.
- Sources: Kinesis Streams, MSK, others. Sinks: Kinesis, Firehose, S3, OpenSearch, DynamoDB, etc.
- **Windows:** tumbling (fixed, non-overlapping), sliding (overlapping), session (gap-based).
- **Checkpoints/snapshots** give fault-tolerance.
- **Studio notebooks** (Zeppelin) for interactive development.

**Scenario:** "Alert if a card is used in 2 countries within 10 minutes" → Flink with session/sliding window over transaction stream.

**Exam trap:** If the question says **stateful, windowed real-time analytics** → Flink. If it says "simply transform per record" → Lambda.

---

## 2.5 Amazon MSK (Managed Streaming for Apache Kafka)

**What it is (easy):** AWS runs the Kafka brokers for you. Use it when your teams *already use Kafka* or need Kafka-specific features/ecosystem.

**Key concepts**
- **Topics → partitions** (like shards). **Consumer groups** to scale consumption. **Offsets** track progress; messages are retained by time/size so they can be re-read.
- **Cluster types:** *Provisioned* (choose broker type/count, storage; optional **tiered storage**) and *Serverless* (auto-scales, no broker planning).
- **Brokers spread across multiple AZs** (typically 3) → high availability; replication factor 3 is typical.
- **Authentication:** IAM access control, SASL/SCRAM (with Secrets Manager), mutual TLS. **Encryption:** in transit (TLS) + at rest (KMS).
- **MSK Connect** – managed Kafka Connect (source/sink connectors, e.g., Debezium CDC → Kafka → S3 sink).
- **Schema Registry:** AWS Glue Schema Registry (Avro, JSON Schema, Protobuf) to enforce schema compatibility.
- Larger messages & more flexible than Kinesis (Kinesis 1 MB limit; Kafka configurable, default ~1 MB).

**Kinesis vs MSK**

| | Kinesis Data Streams | MSK |
|---|---|---|
| Management | Fully managed, simple | Managed Kafka, more knobs |
| Scaling | Shards / on-demand | Brokers & partitions (or Serverless) |
| API | AWS SDK / KCL | Open-source Kafka API (portable, no vendor lock-in) |
| Best for | AWS-native apps, simplicity | Existing Kafka workloads, migrations, Kafka ecosystem |
| Retention | up to 365 days | unlimited with tiered storage |

**Scenario (Bank):** Core banking changes captured by **Debezium on MSK Connect** (Oracle redo-log CDC) → MSK topic per table → **S3 sink connector** → data lake; Spark Structured Streaming on EMR/Glue reads the same topics for real-time aggregates.

**Exam traps**
- "Migrate existing on-prem Kafka with minimal code change" → **MSK**.
- Consumers can lag → monitor `MaxOffsetLag` / `SumOffsetLag`.
- Event source mapping from MSK to **Lambda** is supported (batching, same idea as Kinesis).

---

## 2.6 AWS Database Migration Service (DMS) + Schema Conversion Tool (SCT)

**What it is (easy):** DMS **moves data** between databases (and to S3/Kinesis/Redshift). It can keep the target **in sync continuously** using CDC. **SCT / DMS Schema Conversion** *converts the schema & code* when engines differ (e.g., Oracle → PostgreSQL / Aurora / Redshift).

**Key concepts**
- **Replication instance** (EC2 managed by DMS, runs the tasks) – or **DMS Serverless** (auto-scales capacity).
- **Endpoints** (source, target) + **Tasks**.
- **Migration types:**
  1. **Full load** – one-time copy.
  2. **CDC only** – ongoing changes.
  3. **Full load + CDC** – copy, then keep syncing. *(most common for zero/low-downtime migration)*
- **Homogeneous** (Oracle→Oracle/RDS Oracle): DMS alone (+ native tools possible). **Heterogeneous** (Oracle→Aurora PostgreSQL): **SCT first** for schema, then DMS for data.
- **Source prerequisites for CDC:** Oracle → supplemental logging (LogMiner or Binary Reader); MySQL → binlog `ROW` format; PostgreSQL → logical replication (`wal_level=logical`); SQL Server → MS-CDC/replication.
- **S3 as target:** files in CSV or **Parquet**; CDC files include an **`Op` column** (I/U/D) so you can apply changes downstream.
- **Kinesis/Kafka as target** – stream CDC events.
- **Data validation** compares source and target rows.
- **Multi-AZ replication instance** for HA. **Table mappings** (select/transform rules: rename, filter columns, add prefixes).
- **LOB handling:** limited LOB mode (fast) vs full LOB mode (slow).
- Network: needs connectivity (VPN / Direct Connect / VPC peering) to on-prem sources.

**Scenario (Bank on Oracle 19c):** Replace overnight batch exports with **DMS full load + CDC** from Oracle to S3 (Parquet). A Glue job merges CDC rows (Op = I/U/D) into an **Iceberg** table in the lake, so analysts see data fresh within minutes without hitting the production DB.

**Exam traps**
- "Continuous replication, minimal downtime" → DMS **full load + CDC**.
- Different engines → **SCT** (schema) + DMS (data). DMS alone does not convert stored procedures/schema well.
- Slow initial load → parallel-load, increase instance size, split large tables, turn off target indexes/constraints during load.

---

## 2.7 Other ingestion services

| Service | Use it when | Notes |
|---|---|---|
| **Amazon AppFlow** | Pull/push data from **SaaS** (Salesforce, SAP, ServiceNow, Zendesk, Slack…) to S3/Redshift/Snowflake etc. | No code; supports field mapping, filters, **scheduled / event-driven / on-demand** flows; private connectivity via PrivateLink; encryption with KMS. |
| **AWS DataSync** | **Bulk/scheduled transfer** of files from on-prem NFS/SMB/HDFS/other cloud to S3, EFS, FSx | Agent on-prem; integrity checks; bandwidth throttling; up to 10× faster than DIY tools. |
| **AWS Transfer Family** | Partners/customers need **SFTP / FTPS / FTP / AS2** to S3 or EFS | Managed endpoints; auth via IAM, Directory, custom Lambda. |
| **AWS Snow Family** (Snowcone, Snowball Edge) | **Huge data (TBs–PBs)**, slow/no network | Physical device shipped; also edge compute. Rule of thumb: if upload would take > ~1 week, consider Snowball. |
| **Storage Gateway** | Hybrid: on-prem apps need NFS/SMB/iSCSI backed by S3 | File / Volume / Tape gateway. |
| **S3 Transfer Acceleration** | Upload to a faraway region bucket faster | Uses CloudFront edge locations. |
| **S3 multipart upload** | Large objects (recommended >100 MB, **required >5 GB**; max object 5 TB) | Parallel parts; use lifecycle rule to abort incomplete uploads. |
| **AWS Direct Connect / Site-to-Site VPN** | Private, steady connectivity to AWS | DX = dedicated, consistent; VPN = quick, over internet (encrypted). |
| **Amazon SQS / SNS / EventBridge** | Decoupling, triggers, fan-out | SQS = queue (pull, no replay, no strict stream semantics). SNS = pub/sub push. EventBridge = event bus, schedules, SaaS events, rules. |
| **Amazon Kinesis Video Streams** | Video – rarely on DEA | Skip. |

**Scenario (Insurance):** Third-party hospital partners upload claim files daily via **SFTP** → **Transfer Family** → S3 `landing/` → S3 Event → EventBridge → Step Functions starts validation pipeline.

---

## 2.8 Ingestion design patterns & exam decision rules

**Pattern A – Streaming to the lake (simple):** Producers → Firehose → S3 (Parquet, partitioned).
**Pattern B – Real-time + lake:** Producers → KDS → (Lambda/Flink real-time) + (Firehose → S3).
**Pattern C – DB replication:** DB → DMS (CDC) → S3/Kinesis → Glue → lake/warehouse.
**Pattern D – Event-driven file ingestion:** S3 `ObjectCreated` → EventBridge/Lambda/SQS → Glue/Step Functions.
**Pattern E – SaaS:** AppFlow → S3 → Glue → Redshift.

**Making ingestion reliable**
- **Retries + exponential backoff with jitter** for throttling.
- **Dead-letter queues (DLQ)** / on-failure destinations for poison messages.
- **Idempotent writes** (use deterministic keys, MERGE/upsert, dedupe on event ID).
- **Small-file problem:** thousands of tiny files slow Athena/Spark. Fix: Firehose bigger buffers, compaction job (Glue/Athena CTAS/`OPTIMIZE` in Iceberg), Glue `groupFiles`.
- **Schema drift:** use Glue Schema Registry / crawlers / Iceberg schema evolution; keep raw data untouched.
- **Throughput planning:** shards = max( MB/s ÷ 1 , records/s ÷ 1000 ) for writes; also check read needs.

**Quick calc example:** 5,000 records/s × 2 KB = ~10 MB/s write. By MB: 10 shards. By record count: 5 shards. Need **10 shards** (take the larger).

---

## 2.9 File formats & compression (HIGH-VALUE TOPIC)

| Format | Type | Schema | Best for | Notes |
|---|---|---|---|---|
| **CSV** | Row, text | none | Simple exchange | Large, slow, no types. |
| **JSON** | Row, text | flexible | APIs, logs, semi-structured | Verbose; nested data. |
| **Avro** | **Row**, binary | embedded schema, great **schema evolution** | **Kafka/streaming**, write-heavy | Row-based = fast writes. |
| **Parquet** | **Columnar**, binary | embedded | **Analytics** (Athena, Redshift Spectrum, Spark) | Reads only needed columns, predicate pushdown, excellent compression. **Default answer for analytics.** |
| **ORC** | **Columnar**, binary | embedded | Hive/Presto/Athena | Similar to Parquet, strong in Hive ecosystem; has built-in indexes. |

**Rule:** Analytics/read-heavy → **Parquet/ORC**. Streaming/row writes & schema evolution → **Avro**.

**Compression**

| Codec | Splittable? | Notes |
|---|---|---|
| gzip | **No** (single file = one task) | Good ratio; avoid giant gzip files for Spark. |
| bzip2 | Yes | Slow. |
| Snappy | Splittable *inside* Parquet/ORC/Avro | Fast, moderate ratio — **most common with Parquet**. |
| LZO | Yes (if indexed) | Fast. |
| ZSTD | Yes (in containers) | Good ratio + speed. |

**Table formats (open lakehouse formats):** **Apache Iceberg**, Apache Hudi, Delta Lake.
- Add **ACID transactions, row-level UPDATE/DELETE/MERGE, schema evolution, time travel, partition evolution** on top of S3 Parquet files.
- AWS support: Athena (Iceberg, Hudi, Delta read), Glue, EMR, Redshift Spectrum, **S3 Tables** (managed Iceberg tables with automatic compaction).
- **Scenario:** DPDP "right to erasure" – delete one customer's rows from the lake → Iceberg `DELETE FROM` + snapshot expiry/compaction instead of rewriting whole partitions manually.

---

## 2.10 AWS Glue (the heart of the exam)

**What it is (easy):** Serverless data-integration service: **crawlers** discover data, the **Data Catalog** stores metadata, **ETL jobs** (Apache Spark) transform data, plus **DataBrew, Data Quality, workflows, schema registry**.

### 2.10.1 Glue Data Catalog
- Central **metadata repository** (databases, tables, columns, partitions, schema versions, locations). It is **Hive-metastore compatible**.
- Used by **Athena, Redshift Spectrum, EMR, Glue, Lake Formation, QuickSight**.
- Does **not store data**, only metadata (data stays in S3, etc.).
- **Partition indexes** speed up queries on tables with millions of partitions.
- **Encryption:** catalog metadata encryption with KMS; connection password encryption.
- **Cross-account sharing:** via Lake Formation / RAM / resource policy.

### 2.10.2 Glue Crawlers
- Scan data store (S3, JDBC, DynamoDB, etc.), infer schema, **create/update tables & partitions** in the catalog.
- **Classifiers** (built-in or custom) identify format (CSV/JSON/Parquet/etc.).
- **Schema change policy:** update table / add new columns only / log; **delete behavior**.
- Can be scheduled or run on **S3 event** mode (only new folders).
- **Exam traps:** Crawler creates *too many tables* → data in same prefix has different schemas/formats; fix with exclude patterns, consistent folder structure, or "create a single schema for each S3 path" option. Instead of re-crawling for new partitions every time, use **`MSCK REPAIR TABLE`**, **`ALTER TABLE ADD PARTITION`**, or **Athena partition projection**.

### 2.10.3 Glue ETL Jobs
- **Types:** Spark (batch), **Spark Streaming** (from Kinesis/Kafka), **Python Shell** (small, lightweight), **Ray** (distributed Python), Glue Studio visual jobs, **Interactive Sessions/notebooks**.
- **DynamicFrame** = Glue's flexible DataFrame that tolerates schema inconsistencies (`ResolveChoice`, `ApplyMapping`, `Relationalize` to flatten nested JSON). Convert to Spark DataFrame via `.toDF()` for SQL-style ops.
- **Worker types & capacity:** **DPU** = 4 vCPU + 16 GB. **G.1X** (1 DPU), **G.2X** (2 DPU, memory-heavy/ML/large joins), G.4X/G.8X (bigger), **Z.2X** for Ray. Python shell: 1/16 or 1 DPU.
- **Execution classes:** *Standard* (start immediately) vs **Flex** (cheaper, for non-urgent jobs, may start later).
- **Auto Scaling** (Glue 3.0+): scales workers as needed → saves cost.
- **Job bookmarks:** track what was already processed so reruns **process only new data** (works for S3 and JDBC sources; need `transformation_ctx` in code). Disable/reset for full reprocess.
- **Pushdown predicates:** `push_down_predicate="year='2026' and month='10'"` → Glue reads only the matching partitions from S3 (partition pruning *before* loading).
- **Small files:** use `groupFiles` / `groupSize` options to combine many small S3 files into one Spark task.
- **Triggers:** schedule, on-demand, conditional (job success/fail), EventBridge.
- **Workflows:** orchestrate crawler + job chain with visual graph (simple, Glue-only). For more complex → Step Functions / MWAA.
- **Connections:** JDBC to RDS/Redshift/on-prem (needs VPC subnet + security group; **S3 gateway endpoint** + NAT or VPC endpoints for access to S3/Glue).
- **Custom libraries:** `--additional-python-modules`, `--extra-py-files`, `--extra-jars`.
- **Monitoring:** CloudWatch job metrics, Spark UI, continuous logging, Job run insights.
- **Glue Schema Registry:** register Avro/JSON/Protobuf schemas for Kinesis/MSK producers & consumers, enforce compatibility modes (BACKWARD, FORWARD, FULL, NONE).
- **FindMatches ML transform:** dedupe / link records without a common unique ID (fuzzy matching).
- **Sensitive data detection:** detect & mask/hash PII inside Glue jobs.
- **Glue DataBrew:** *no-code* data preparation (250+ transforms), profiling, data quality rules, visual recipes — good for analysts.
- **Glue Data Quality:** write rules in **DQDL** (e.g., `IsComplete "policy_id"`, `ColumnValues "age" between 18 and 100`, `Uniqueness "claim_id" > 0.99`); evaluate in ETL jobs or catalog; results in CloudWatch/S3; can **fail the job or route bad rows**; ML-based anomaly detection available.
- **Glue Data Catalog Federation, zero-ETL integrations** (e.g., SaaS/Aurora/DynamoDB → Redshift/SageMaker lakehouse) exist for newer exam content.

**Typical Glue ETL skeleton (PySpark)**
```python
import sys
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glue = GlueContext(sc)
job = Job(glue); job.init(args['JOB_NAME'], args)

src = glue.create_dynamic_frame.from_catalog(
        database="bank_raw", table_name="transactions",
        push_down_predicate="ingest_date='2026-10-30'",
        transformation_ctx="src")           # needed for bookmarks

df = src.toDF().dropDuplicates(["txn_id"])  # idempotency / dedupe
df.write.mode("append").partitionBy("txn_date") \
  .parquet("s3://bank-curated/transactions/")
job.commit()                                 # saves bookmark state
```

**Scenario (Insurance):** Daily claim CSVs land in S3. A Glue crawler registers them → Glue job (bookmarks ON, pushdown predicate on date) cleans, validates with **Glue Data Quality**, masks PII, writes **Parquet partitioned by `claim_date`** → second crawler/partition add → **Athena** and **Redshift Spectrum** query it. Bad rows go to a `quarantine/` prefix and an SNS alert is sent via EventBridge on DQ failure.

**Glue troubleshooting cheat-sheet**

| Symptom | Likely cause | Fix |
|---|---|---|
| Job reprocesses same data | Bookmarks disabled / no `transformation_ctx` | Enable bookmarks, add ctx |
| `OutOfMemoryError` driver | `collect()`, huge broadcast, many small files | Use bigger worker (G.2X), `groupFiles`, avoid collect, tune partitions |
| Executor OOM / slow stage | **Data skew** | Salting keys, repartition, broadcast small table, enable AQE |
| Slow, many S3 listings | Small files, too many partitions | Compaction, partition indexes, bounded execution |
| Job can't reach RDS/S3 | VPC networking | Connection with subnet+SG, **self-referencing security group**, S3 VPC endpoint/NAT |
| Cost too high | Over-provisioned, always Standard | Auto scaling, Flex, right-size workers, process only new data |
| Crawler too slow | Re-scanning everything | Incremental crawls / S3 event mode / sample size |
| Access denied | Missing IAM role perms/Lake Formation grants | Fix role (S3, Glue, KMS) and LF permissions |

---

## 2.11 Amazon EMR

**What it is (easy):** Managed big-data cluster for **Spark, Hadoop, Hive, Presto/Trino, HBase, Flink**, etc. Gives more control than Glue (custom configs, large/special workloads).

**Deployment options**
1. **EMR on EC2** – classic cluster: **Primary (master)**, **Core** (HDFS + compute), **Task** (compute only, ideal for **Spot**).
2. **EMR on EKS** – run Spark on existing Kubernetes.
3. **EMR Serverless** – no cluster management; pay per vCPU/GB used; auto-scale. *(least overhead)*

**Key concepts**
- **Transient cluster:** start → run steps → terminate (cost-efficient for batch). **Long-running:** interactive/streaming/HBase.
- **EMRFS:** access S3 as a file system (decouples storage from compute → shut down cluster without losing data). **HDFS** is ephemeral (lost on termination).
- **Instance fleets / instance groups**, **Managed scaling**, **Spot** for task nodes (never put only Spot in core nodes holding HDFS).
- **Steps** (jobs), **bootstrap actions** (install software on node start), **S3DistCp** (efficient copy S3↔HDFS).
- **Security:** IAM roles (service role, EC2 instance profile), **security configurations** (encryption at rest/in transit), Kerberos, Lake Formation integration, VPC private subnets.
- **Spark tuning basics:** right partitions (`spark.sql.shuffle.partitions`), file sizes 128 MB–1 GB, Kryo serializer, broadcast joins for small tables, cache wisely, avoid skew with salting, use columnar formats.

**Glue vs EMR**

| | Glue | EMR |
|---|---|---|
| Ops | Serverless, no cluster | You choose cluster (or EMR Serverless) |
| Control | Limited config | Full framework control (Hive, Presto, HBase, custom libs) |
| Best for | Standard ETL, catalog-centric | Large/complex, specialized frameworks, existing Hadoop/Spark code, tighter cost tuning at scale |
| Pricing | per DPU-second | per EC2 (+EMR fee) or per vCPU/GB (Serverless) |

**Scenario (Bank — Hadoop modernization):** On-prem Hadoop Hive jobs → **EMR on EC2 transient clusters (Spot task nodes)** reading/writing **S3 via EMRFS**, Hive metastore replaced by **Glue Data Catalog**, orchestration with **MWAA/Step Functions**. Result: storage and compute scale independently, cost drops because clusters exist only during job windows.

**Exam traps**
- "Existing Spark/Hive/HBase code, want minimal change" → EMR.
- "Reduce EMR cost" → transient clusters, Spot task nodes, managed scaling, EMR Serverless, Graviton.
- Data persistence → write results to **S3**, not HDFS.

---

## 2.12 AWS Lambda for data engineering

- **Event-driven serverless functions**: S3 events, Kinesis/MSK/SQS/DynamoDB Streams mappings, EventBridge schedules, API Gateway.
- **Limits:** timeout **15 minutes**, memory up to **10,240 MB** (CPU scales with memory), `/tmp` up to 10,240 MB, deployment package 50 MB zipped / 250 MB unzipped (container image up to 10 GB), **sync payload 6 MB**, default **1,000 concurrent executions per region** (soft limit).
- **Concurrency:** *reserved* (guarantee/limit), *provisioned* (pre-warmed, avoids cold start).
- **Layers** for shared libraries (e.g., pandas via AWS SDK for pandas layer).
- **Invocation:** synchronous, asynchronous (retries twice + DLQ/destinations), **poll-based event source mappings** (Kinesis, SQS, DynamoDB Streams, MSK).
- **Partial batch response** (`ReportBatchItemFailures`) for SQS/Kinesis: only failed items are retried.
- **VPC Lambda** needs NAT/VPC endpoints for internet/AWS API access.
- **Use for:** light transforms, glue-code, triggers, validations, small files. **Not for:** long (>15 min) or very heavy jobs → Glue/EMR/Fargate/Batch.

**Scenario:** A new claim PDF lands in S3 → Lambda calls **Textract**, writes JSON to S3 → triggers Glue workflow. SQS between S3 and Lambda smooths bursts and gives retries/DLQ.

---

## 2.13 Orchestration & event-driven automation

### AWS Step Functions
- Serverless **state machine** (JSON/ASL) to coordinate services visually.
- **Standard workflows:** up to **1 year**, exactly-once, full history, priced per state transition → long ETL.
- **Express workflows:** up to **5 minutes**, high volume (100k+/s), at-least-once, priced by duration → streaming/IoT/short.
- **States:** Task, Choice, Parallel, **Map** (iterate over items; **Distributed Map** for millions of S3 objects), Wait, Pass, Fail, Succeed.
- **Error handling:** `Retry` (with `IntervalSeconds`, `BackoffRate`, `MaxAttempts`) and `Catch` (fallback state).
- **Service integrations:** Request-Response, **`.sync`** (wait until job completes – e.g., Glue `startJobRun.sync`, EMR, Athena, Batch), **`.waitForTaskToken`** (pause until external system/human callback).
- Direct integrations with 200+ services (Glue, Athena, EMR, Redshift Data API, DynamoDB, SNS, SQS, Lambda…) → fewer Lambdas.

### Amazon MWAA (Managed Workflows for Apache Airflow)
- Managed **Apache Airflow**; DAGs in Python stored in an **S3 bucket** (`dags/`, `plugins.zip`, `requirements.txt`).
- Use when team **already uses Airflow**, needs complex dependencies, backfills, many integrations/operators, portability.
- Environment classes (small/medium/large), workers autoscale; runs in VPC; logs to CloudWatch; secrets via Secrets Manager backend.
- Concepts: **DAG, Task, Operator, Sensor, XCom, Hook, Connection, Variable, Executor, schedule_interval, catchup/backfill, retries, SLAs**.

```python
from airflow import DAG
from airflow.providers.amazon.aws.operators.glue import GlueJobOperator
from datetime import datetime
with DAG("claims_daily", start_date=datetime(2026,1,1), schedule="@daily", catchup=False) as dag:
    clean = GlueJobOperator(task_id="clean", job_name="claims_clean", wait_for_completion=True)
```

### Amazon EventBridge
- **Event bus**: rules match events (patterns) → targets (Lambda, Step Functions, SQS, SNS, Glue workflow, etc.).
- **Scheduler / scheduled rules** (cron/rate) — replaces CloudWatch Events. **Pipes** for point-to-point source→filter→enrich→target.
- Receives **AWS service events** (e.g., Glue job state change, S3 object created *via EventBridge*, Redshift) → alerting & orchestration.

### Others
- **SNS** (notify, fan-out), **SQS** (buffer; **FIFO** for ordering/exactly-once, **DLQ**, visibility timeout), **AWS Batch** (containerized batch jobs, Spot), **ECS/EKS/Fargate** (containers), **Glue workflows**.

**Which orchestrator?**

| Need | Pick |
|---|---|
| AWS-native, serverless, simple-to-medium flows, retries, branching | **Step Functions** |
| Team uses Airflow, complex DAGs, custom operators, backfills | **MWAA** |
| Only Glue crawlers + jobs | **Glue workflows** |
| Time-based or event-based trigger | **EventBridge** |
| Streaming/high volume short flows | Step Functions **Express** |

**Scenario (Bank nightly batch):** EventBridge Scheduler (01:00 IST) → Step Functions: (1) DMS/AppFlow extract check → (2) Glue job `.sync` → (3) Glue Data Quality → **Choice**: pass → Redshift `COPY` via Redshift Data API → SNS success; fail → SNS alert + quarantine. Retries with backoff on transient errors.

---

## 2.14 Transformation & processing concepts exam loves

**ETL vs ELT:** ETL = transform before loading (Glue/EMR). **ELT** = load raw into warehouse/lake then transform with SQL (Redshift, Athena CTAS, dbt).

**Common transformations:** filtering, dedupe, joins, aggregation, windowing, pivot/unpivot, flatten nested JSON, type casting, normalization, standardizing dates/time zones, enrichment (lookups), masking, **SCD handling**.

**Spark concepts:** RDD / DataFrame, lazy evaluation, transformations vs actions, **narrow vs wide (shuffle)** operations, partitions, broadcast join, caching, skew, **AQE** (adaptive query execution).

**Data skew fixes:** salting (add random suffix to hot key), broadcast small tables, repartition by better key, filter out null-heavy keys.

**Joins at scale:** broadcast hash join (small × big), sort-merge join (big × big).

**Batch ingestion extras:** "late-arriving data" → partition by event date and reprocess last N days; "exactly-once" → dedupe by key, upsert (MERGE) into Iceberg/Redshift.

**Streaming concepts:** event time vs processing time, **watermarks** (how late data may be), windows, checkpointing, state, exactly-once sinks.

**Infrastructure as code & CI/CD (appears in Domains 1 & 3):** **CloudFormation / CDK / SAM / Terraform**, **CodeCommit/GitHub → CodeBuild → CodePipeline → CodeDeploy**, version control for ETL scripts, parameterize with environment variables, deploy pipelines across dev/test/prod, **SAM** for serverless (Lambda, Step Functions).

**Containers:** package custom code in Docker → ECR → run on ECS/EKS/Fargate/Lambda/Batch. **Fargate** = serverless containers (no EC2 to manage).

**Throttling & rate limits:** DynamoDB (WCU/RCU), Kinesis (per shard), Lambda concurrency, Athena (DML concurrency quotas), API Gateway. Answer pattern: **backoff + jitter, batch requests, increase capacity/quotas, buffer with SQS/Kinesis**.

---

# 3. DOMAIN 2 — DATA STORE MANAGEMENT (26%)

## 3.1 Choosing the right data store (master table)

| Need | Best service | Why |
|---|---|---|
| Cheap, unlimited, durable storage; data lake | **S3** | 11 nines durability, any format, decoupled from compute |
| SQL analytics on petabytes, BI, joins, aggregates | **Redshift** | Columnar MPP warehouse |
| Ad-hoc SQL on S3, pay per query | **Athena** | Serverless Trino/Presto |
| Single-digit-ms key-value / NoSQL at any scale | **DynamoDB** | Serverless, auto scale |
| Relational OLTP (transactions, joins) | **RDS / Aurora** | Managed SQL engines |
| Microsecond caching / sessions | **ElastiCache (Redis/Valkey, Memcached)**, **DAX** (DynamoDB only), **MemoryDB** (durable Redis) |
| Full-text search, log analytics, dashboards | **OpenSearch** | Inverted index, Kibana-like dashboards |
| Time-series (IoT, metrics) | **Timestream** | Memory + magnetic tiers, time functions |
| Graph relationships (fraud rings) | **Neptune** | Gremlin / SPARQL / openCypher |
| MongoDB-compatible documents | **DocumentDB** | |
| Cassandra-compatible wide column | **Keyspaces** | |
| Ledger / immutable history | **QLDB** (being retired — know the concept; alternatives: Aurora/ledger tables) | |
| Shared file storage | **EFS** (Linux NFS), **FSx** (Windows/Lustre/NetApp/OpenZFS) | |
| Block storage for EC2 | **EBS** | |

**OLTP vs OLAP:** OLTP = many small reads/writes, row-oriented (RDS/Aurora/DynamoDB). OLAP = big scans & aggregates, column-oriented (Redshift/Athena/Parquet).

---

## 3.2 Amazon S3 (the foundation)

**What it is:** Object storage. Buckets → objects (key, value, metadata, version ID). Flat namespace (folders are just key prefixes). **Strongly consistent** for all operations (read-after-write, list).

**Numbers to remember**
- Durability **99.999999999%** (11 nines). Max object size **5 TB**. Single PUT up to **5 GB**; multipart for larger.
- Request rates: **3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD per second per prefix** → spread keys across multiple prefixes (and partitions) to scale reads.

**Storage classes**

| Class | Use | Retrieval | Min duration |
|---|---|---|---|
| **Standard** | Frequent access | ms | none |
| **Intelligent-Tiering** | Unknown / changing access | ms (auto-moves between tiers; small monitoring fee, no retrieval fee) | none |
| **Standard-IA** | Infrequent, needs ms access, ≥3 AZs | ms (retrieval fee) | 30 days |
| **One Zone-IA** | Re-creatable infrequent data, 1 AZ | ms | 30 days |
| **Glacier Instant Retrieval** | Archive but rarely accessed *with ms access* | ms | 90 days |
| **Glacier Flexible Retrieval** | Archive | Expedited 1–5 min, Standard 3–5 h, Bulk 5–12 h | 90 days |
| **Glacier Deep Archive** | Cheapest, long-term compliance (7–10 yrs) | Standard 12 h, Bulk 48 h | 180 days |
| **Express One Zone** | Single-digit-ms, high-performance (ML, analytics hot data), one AZ | ms | n/a |

**Lifecycle rules:** Transition (Standard → IA → Glacier → Deep Archive), **Expire** current/non-current versions, **abort incomplete multipart uploads**, delete expired delete-markers. Objects < 128 KB are not transitioned to IA classes by default (not cost-effective).

**Features you must know**
- **Versioning** (protect from overwrite/delete; needed for replication & Object Lock; MFA Delete adds protection).
- **Replication:** **CRR** (cross-region: DR, latency, compliance) and **SRR** (same-region: log aggregation, prod/test copy). Needs versioning on both; only **new objects** replicate by default (use **S3 Batch Replication** for existing); delete markers optional; **Replication Time Control** = 15-min SLA.
- **Object Lock:** WORM. **Governance mode** (privileged users can override) vs **Compliance mode** (nobody, even root, until retention ends) + **Legal hold**. → regulatory retention (SEC 17a-4, IRDAI/RBI record-keeping).
- **Event notifications** → Lambda / SQS / SNS / **EventBridge** (EventBridge gives filtering on many attributes, multiple targets).
- **S3 Select / Glacier Select** – (legacy) SQL on a single object; prefer Athena for new work.
- **Inventory** (daily/weekly object listing, CSV/Parquet/ORC) – audit & Batch Operations input. **Storage Lens** (org-wide usage/cost metrics). **Storage class analysis**.
- **S3 Batch Operations:** run actions (copy, tag, invoke Lambda, restore, Object Lock) on billions of objects.
- **Access points** (per-app policies/VPC-restricted), **Multi-Region Access Points**, **Object Lambda** (transform on read, e.g., redact PII).
- **Requester Pays**, **Pre-signed URLs** (temporary access), **Transfer Acceleration**, **S3 Tables** (managed Iceberg).
- **Performance:** parallelize with multipart & **byte-range fetches**, use larger files (128 MB–1 GB) for analytics, avoid hot prefixes, use CloudFront for repeated reads.

**Security (see Domain 4 for more):** Block Public Access (account/bucket), bucket policies, IAM, ACLs (disable via *Bucket owner enforced*), SSE-S3 (default), **SSE-KMS**, SSE-C, **DSSE-KMS**, client-side encryption, **S3 Bucket Keys** (cut KMS request cost by up to ~99%), VPC gateway endpoint, Macie.

**Partitioning in S3 (Hive style):** `s3://lake/claims/year=2026/month=10/day=30/part-0001.parquet`.
- Partition by columns frequently used in `WHERE` (date, region).
- **Don't over-partition** (millions of tiny partitions/files). Aim for ≥128 MB per file, sensible cardinality.

**Real-world scenario (Bank compliance):** Audit logs must be kept 8 years, tamper-proof. S3 bucket with **Object Lock (Compliance mode, 8-year retention)** + **lifecycle**: Standard 90 days → Glacier Instant Retrieval 1 year → **Deep Archive** after. **CRR** to a second region for DR. **Athena** queries recent logs; older logs restored on demand.

**Exam traps**
- "Cannot be deleted by anyone including root" → **Compliance mode**.
- "Rarely accessed but needs millisecond access" → Standard-IA or **Glacier Instant Retrieval**.
- "Unpredictable access patterns" → **Intelligent-Tiering**.
- Replication isn't retroactive (existing objects) → **Batch Replication**.
- Delete via lifecycle needs both current & **noncurrent version** expiration when versioning is on.

---

## 3.3 Amazon Redshift

**What it is (easy):** AWS's **columnar, massively parallel (MPP) data warehouse**. A **leader node** plans queries; **compute nodes** (split into **slices**) scan data in parallel.

**Deployment**
- **Provisioned clusters:** **RA3** nodes (managed storage on S3 – compute and storage scale independently; recommended), DC2 (local SSD, small data), legacy DS2.
- **Redshift Serverless:** auto-scales **RPUs** (Redshift Processing Units), pay per use, no cluster management – **least ops** answer.
- **Multi-AZ** deployments, **Elastic resize** (minutes, change node count/type within limits) vs **classic resize** (longer, more flexible), **concurrency scaling** (extra clusters for bursts of queued queries; free credits per day), **AQUA** (older caching layer; now built into RA3).

**Table design (HEAVILY TESTED)**

*Distribution style* – how rows are spread across slices:

| Style | Meaning | Use for |
|---|---|---|
| **AUTO** (default) | Redshift picks (starts ALL for small, switches to EVEN/KEY as table grows) | Default, when unsure |
| **KEY** | Rows with same key value on same slice | Large fact + large dimension joined on same key (co-located join, avoids shuffling) |
| **ALL** | Full copy on every node | Small, slowly-changing dimension tables |
| **EVEN** | Round-robin | No joins / no obvious key |

*Sort keys* – physical order on disk (enables **zone maps** to skip blocks):
- **Compound** (default; columns in order – good when queries filter on leading column(s)).
- **Interleaved** (equal weight to each column; heavier VACUUM; rare).
- **AUTO** (Redshift learns). Put **timestamp / frequently range-filtered** columns first.

*Compression encodings*: `ENCODE AUTO` (recommended) – saves I/O and storage. **Constraints (PK/FK/unique) are informational only** – not enforced (but help optimizer).

**Loading data**
- **`COPY`** from S3 (best), DynamoDB, EMR, SSH. Parallel load: **split data into multiple files — number of files ideally a multiple of the number of slices**, compress (gzip, bzip2, zstd), use **manifest** for exact file list, use IAM role (not keys), `COMPUPDATE`/`STATUPDATE` options, `MAXERROR`.
- **Never load with many single-row `INSERT`s** — slow. Use COPY or multi-row insert.
- **UPSERT** pattern: load into staging table → `MERGE` (now supported) or delete-then-insert in a transaction.
- **`UNLOAD`** → export query results to S3 (Parquet supported, parallel).
- **Auto-copy from S3** (continuous ingestion rules), **streaming ingestion** (direct from Kinesis Data Streams / MSK via **materialized views**, low latency), **zero-ETL integrations** (Aurora MySQL/PostgreSQL, RDS MySQL, DynamoDB → Redshift automatically).
- **Redshift Data API:** run SQL over HTTPS without managing connections/drivers (great for Lambda/Step Functions; async; IAM/Secrets Manager auth).

**Querying outside the cluster**
- **Redshift Spectrum:** query S3 data via **external schema/tables** (Glue Catalog) without loading; best with **Parquet + partitions**; scales with its own compute layer; join S3 data with local tables.
- **Federated queries:** query live **RDS/Aurora (PostgreSQL/MySQL)** from Redshift.
- **Data sharing:** share live data between clusters/workgroups/accounts/regions **without copying** (producer/consumer; RA3 or Serverless).
- **Redshift ML:** `CREATE MODEL` using SageMaker from SQL.

**Maintenance & performance**
- **VACUUM** (reclaim space & re-sort after deletes/updates; auto vacuum exists), **ANALYZE** (update stats; auto analyze exists).
- **Materialized views:** precompute joins/aggregates; **auto refresh**; **automatic query rewriting**; incremental refresh when possible.
- **WLM (Workload Management):** **Auto WLM** (ML-based concurrency & memory) with **query priorities**, or manual queues; **QMR – Query Monitoring Rules** (log/abort/hop queries exceeding thresholds); **SQA – Short Query Acceleration**; **concurrency scaling**.
- **Result caching** (identical queries return instantly), **automatic table optimization (ATO)**.
- **System tables/views:** `SYS_QUERY_HISTORY`, `STL_*`, `SVL_*`, `STV_*`, `SVV_TABLE_INFO` (find skew, unsorted %, stats off), `STL_ALERT_EVENT_LOG` (flags nested loop joins, missing stats), `EXPLAIN` (look for `DS_BCAST_INNER`, `DS_DIST_BOTH` = expensive data movement).
- **Backup:** automatic snapshots (to S3; 1 day default retention, up to 35), manual snapshots, **cross-region snapshot copy** (DR; encrypted snapshots need KMS grant in target region), **Multi-AZ**, **Pause/Resume** (provisioned) to save cost, **Reserved Instances** for steady workloads.

**Security:** KMS/HSM encryption at rest, SSL in transit (`require_ssl`), VPC, enhanced VPC routing (force COPY/UNLOAD traffic through VPC), IAM roles, **column-level GRANT**, **row-level security (RLS) policies**, **dynamic data masking**, **Lake Formation** integration for Spectrum, audit logging (connection log, user log, **user activity log** – requires `enable_user_activity_logging` parameter) to S3/CloudWatch.

**Redshift vs Athena vs EMR (analytics):**

| | Redshift | Athena | EMR |
|---|---|---|---|
| Data location | Loaded in warehouse (+ Spectrum for S3) | S3 | S3/HDFS |
| Best for | Repeated BI/dashboard queries, complex joins, high concurrency | Ad-hoc / occasional queries, log analysis | Heavy/custom big-data processing |
| Cost | Per node-hour/RPU | Per TB scanned | Per instance-hour |
| Ops | Low–medium | **None** | Medium |

**Scenario (Insurance BI):** Claims fact table (billions of rows) `DISTKEY(policy_id)`, `SORTKEY(claim_date)`; policy dimension also `DISTKEY(policy_id)` → co-located join; small `product` dimension `DISTSTYLE ALL`. Nightly **COPY** from S3 Parquet via Step Functions; **materialized view** for monthly loss-ratio; QuickSight on top; old years remain in S3 queried through **Spectrum**; analysts' queries isolated using **WLM queues/workgroups**.

**Exam traps**
- "Joins slow due to data shuffling" → align **DISTKEY** on join column, or **ALL** for small dims.
- "Queries filter by date range" → **SORTKEY on date**.
- "COPY slow" → single large file → split into many (multiple of slices), compress.
- "Share data with another team without copying" → **Data sharing**.
- "Spiky BI usage / unpredictable" → **Redshift Serverless** or concurrency scaling.
- "Query S3 data from Redshift without loading" → **Spectrum**.
- Encrypting an existing unencrypted cluster → modify cluster to enable encryption (migration happens automatically).

---

## 3.4 Amazon DynamoDB

**What it is (easy):** Serverless NoSQL key-value/document DB with single-digit-millisecond latency at any scale.

**Core design**
- **Primary key:** *Partition key (PK)* alone, or *PK + Sort key (SK)* (composite). Items with the same PK are stored together, sorted by SK.
- **Item max size 400 KB.** Schemaless attributes (except keys). Data types: scalar, set, list, map.
- **Choose a high-cardinality, evenly-accessed partition key** → avoid **hot partitions** (e.g., don't use `date` or a status flag alone). Fix with **write sharding** (suffix) or composite keys.
- **Query** (by PK [+ SK condition]) = efficient. **Scan** = reads whole table = slow/expensive (use only when necessary; use parallel scan, filters don't reduce read cost).

**Capacity modes**
- **On-demand:** pay per request; instantly accommodates traffic; unpredictable workloads.
- **Provisioned:** specify **RCU/WCU** (+ optional **auto scaling**); cheaper for predictable load; **Reserved capacity** for more savings.
- **Read capacity units:** 1 RCU = **1 strongly consistent read/s up to 4 KB**, or **2 eventually consistent reads/s**; transactional reads cost **2 RCU**.
- **Write capacity units:** 1 WCU = **1 write/s up to 1 KB**; transactional writes **2 WCU**.
  - *Example:* 100 strongly-consistent reads/s of 8 KB items → 100 × (8/4 = 2) = **200 RCU**; eventually consistent → **100 RCU**. 50 writes/s of 3 KB → 50 × 3 = **150 WCU**.
- Burst & **adaptive capacity** help but don't replace good key design.
- Table classes: **Standard** vs **Standard-IA** (cheaper storage, higher access cost).

**Secondary indexes**
- **GSI (Global Secondary Index):** different PK/SK, can be added **any time**, **eventually consistent**, own provisioned capacity (throttled GSI can throttle base table writes!), spans all partitions.
- **LSI (Local Secondary Index):** same PK, different SK; **must be created at table creation**; supports strongly consistent reads; **10 GB limit per partition-key value**.

**Other features**
- **DynamoDB Streams:** ordered change log (24-hour retention) → trigger Lambda, replicate, audit; view types: KEYS_ONLY, NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES. Alternative: **Kinesis Data Streams for DynamoDB** (longer retention, multiple consumers).
- **TTL:** auto-delete expired items (epoch seconds attribute) at no WCU cost; deletes appear in Streams.
- **DAX:** in-memory write-through cache for DynamoDB → **microsecond reads** with no app logic change (only for read-heavy, eventually consistent reads).
- **Global Tables:** multi-Region, multi-active replication (last-writer-wins) for low-latency global apps/DR.
- **Backup:** **PITR** (continuous, restore to any second in the last **35 days**), on-demand backups, **AWS Backup**; **Export to S3** (no RCU consumed; for analytics in Athena), **Import from S3**.
- **Transactions** (`TransactGetItems/TransactWriteItems`, up to 100 items; ACID), **conditional writes** (optimistic locking), **PartiQL** SQL-like.
- **Batch operations:** `BatchGetItem` (100 items / 16 MB), `BatchWriteItem` (25 items / 16 MB) – handle `UnprocessedItems` with backoff.
- **Security:** encryption at rest always on (AWS-owned, AWS-managed, or customer-managed KMS key), IAM **fine-grained access** with `dynamodb:LeadingKeys` (restrict by PK) and attribute conditions, VPC **gateway endpoint**.

**Scenario (Bank):** Real-time account-balance/“last 10 transactions” API. Table: PK=`account_id`, SK=`txn_timestamp`. Query last 10 with `ScanIndexForward=false, Limit=10`. **GSI** on `merchant_id` for merchant lookups. **Streams → Lambda** updates a fraud-feature store. **TTL** deletes session tokens. **Export to S3** nightly → Athena for analytics (no impact on production RCU).

**Exam traps**
- ProvisionedThroughputExceeded → hot key, under-provisioned, GSI throttling → better key, on-demand, auto scaling, DAX, backoff.
- Analytics on DynamoDB → don't `Scan`; export to S3 / zero-ETL to Redshift / Streams → Kinesis → Firehose.
- Need strong consistency across Regions? Global tables are eventually consistent across Regions.

---

## 3.5 Amazon RDS & Aurora

**RDS** (managed MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2): automated backups (up to 35 days) + PITR, **Multi-AZ** (sync standby = **HA/failover**, NOT for read scaling), **Read replicas** (async, **read scaling**, can be cross-Region/promoted), snapshots, patching, storage autoscaling, parameter groups, **RDS Proxy** (connection pooling for Lambda/serverless; IAM auth).
- **Encryption** must be enabled at creation (to encrypt existing: snapshot → copy with encryption → restore).

**Aurora** (MySQL/PostgreSQL-compatible): storage auto-scales to 128 TiB, **6 copies across 3 AZs**, up to **15 read replicas**, **reader/writer endpoints**, **Aurora Serverless v2** (fine-grained auto scaling), **Global Database** (cross-Region, <1 s lag), **Backtrack** (MySQL; rewind without restore), **Parallel Query**, **zero-ETL to Redshift**, **cloning** (fast copy-on-write), export snapshot to S3 as Parquet.

**Scenario:** Oracle 19c core system → modernize to **Aurora PostgreSQL** using **SCT + DMS**; reporting load moved to **Aurora Readers** and **zero-ETL to Redshift** so BI never touches the OLTP writer.

**Exam traps:** Multi-AZ ≠ read scaling. Read replica ≠ HA (manual promotion). Too many Lambda connections → **RDS Proxy**.

---

## 3.6 Other data stores (know the “when”)

| Service | Key facts for exam |
|---|---|
| **Amazon OpenSearch Service / Serverless** | Search + log analytics; domain with data nodes, dedicated master nodes; **UltraWarm / cold storage** for older log data; **Index State Management (ISM)** for rollover/retention; ingest via **Firehose, OpenSearch Ingestion (Data Prepper), Logstash, Lambda**; fine-grained access control, KMS, VPC. Use for text search, observability, security analytics. |
| **Amazon Timestream** | Time-series; **memory store** (recent) + **magnetic store** (history) with retention policies; SQL with time functions; IoT/DevOps metrics. |
| **Amazon Neptune** | Graph DB (property graph: Gremlin/openCypher; RDF: SPARQL) — fraud rings, recommendations, knowledge graphs. |
| **Amazon DocumentDB** | MongoDB-API compatible; document JSON data. |
| **Amazon Keyspaces** | Managed Apache Cassandra-compatible (CQL). |
| **Amazon MemoryDB** | Durable Redis-compatible in-memory primary database (multi-AZ transaction log). |
| **ElastiCache** | Cache layer (Redis/Valkey: persistence, replication, data structures, pub/sub; Memcached: simple, multi-threaded). Caching strategies: **lazy loading** (cache-aside), **write-through**, **TTL**. |
| **Amazon QLDB** | Immutable, cryptographically verifiable ledger (service is being retired; focus on the *concept*). |
| **AWS Lake Formation / S3 + Glue** | Governed data lake (see Domain 4). |
| **Amazon SageMaker Feature Store** | Online/offline store of ML features. |

---

## 3.7 Data cataloging, discovery & lineage

- **Glue Data Catalog** – technical metadata (tables, schemas, partitions). Populate with crawlers, `CREATE TABLE` DDL (Athena), `batch_create_partition`, Glue jobs (`enableUpdateCatalog`).
- **Amazon DataZone / SageMaker Catalog** – business data catalog & data-product marketplace (publish, discover, subscribe/approve access; business glossary, governance workflows). Think “data mesh / self-service data portal”.
- **AWS Glue Schema Registry** – streaming schema governance.
- **Lineage:** DataZone lineage (OpenLineage compatible), Glue job logs + CloudTrail, EMR/Spark lineage tools; show data origin → transformations → consumers (important for audits).
- **Tags & metadata:** S3 object tags, Glue table properties, **LF-Tags**.
- **AWS Clean Rooms:** collaborate with partners on combined datasets without sharing raw data.
- **AWS Data Exchange:** subscribe to third-party datasets.

---

## 3.8 Data modeling & schema design

**Dimensional modeling**
- **Star schema:** central **fact table** (measures: premium_amount, claim_amount; foreign keys) + denormalized **dimension tables** (customer, policy, date, product). Fast for analytics.
- **Snowflake schema:** dimensions normalized further (less storage, more joins).
- **Data Vault:** hubs, links, satellites (auditable, agile for enterprise).
- **Normalization (3NF)** for OLTP; **denormalization** for OLAP/NoSQL.

**Slowly Changing Dimensions (SCD)**

| Type | Behaviour | Example |
|---|---|---|
| **Type 1** | Overwrite; no history | Fix a spelling mistake in customer name |
| **Type 2** | New row per change with `valid_from`, `valid_to`, `is_current` | Customer address changes — keep history for audit |
| **Type 3** | Extra column for previous value | `previous_city` |

*Type-2 MERGE pattern:* match on business key + current flag; if attributes changed → close old row (`valid_to = now, is_current=false`) and insert new row.

**DynamoDB modeling:** design around **access patterns first**; single-table design; overloaded keys; GSIs for alternative lookups.

**Schema evolution:** add columns (nullable) safely; Avro/Parquet/Iceberg handle evolution; Athena handles added columns at end for CSV; avoid renaming/reordering in plain Parquet without a table format; use **Glue Schema Registry** compatibility; **Iceberg** supports add/drop/rename/reorder & partition evolution safely.

**Indexing / partitioning / clustering strategy:** partition by date/region (S3), sort+dist keys (Redshift), GSIs (DynamoDB), bucketing in Athena/Glue (for high-cardinality join keys).

**Data lifecycle & retention:** S3 lifecycle policies, DynamoDB TTL, Redshift unload old data → S3 → Spectrum, OpenSearch ISM, CloudWatch Logs retention, RDS snapshot retention, **legal hold** vs deletion for **GDPR/DPDP “right to erasure”**; keep **raw immutable** + **delete flags** + propagate deletions downstream.

**Data migration approaches (AWS):** DMS (DB), DataSync (files), Snowball (bulk offline), AppFlow (SaaS), Transfer Family (SFTP), Glue/Spark (transform during move), S3 Batch (object ops), **Redshift COPY/UNLOAD**, **Lake Formation blueprints** (legacy).

**Scenario (Insurance):** Policy dimension requires audit history for IRDAI → **SCD Type 2** in Redshift using `MERGE`. Customer address typos → Type 1. Claim facts partitioned by `claim_date`; Date dimension prebuilt.

---

# 4. DOMAIN 3 — DATA OPERATIONS AND SUPPORT (22%)

## 4.1 Amazon Athena

**What it is (easy):** Serverless SQL on data in S3 (Trino/Presto engine). **No servers, no loading.** You pay **per TB of data scanned** (DDL and failed queries are free). So the whole game is: **scan less data**.

**How to scan less (and pay less) — memorize**
1. **Columnar formats** (Parquet/ORC) → read only needed columns.
2. **Compression** (Snappy/ZSTD/gzip).
3. **Partitioning** → `WHERE` on partition column prunes folders.
4. **Bucketing** (on join/filter high-cardinality key).
5. **Right-sized files** (~128 MB+; avoid thousands of tiny files; compact them).
6. Select only needed columns (**never `SELECT *`**); use `LIMIT` carefully (still scans).
7. **Partition projection:** Athena computes partitions from table properties instead of reading the catalog → faster for millions of partitions, no `MSCK REPAIR` needed (good for date-partitioned logs).
8. **Query result reuse** (cache) for repeated queries.

**Key features**
- **CTAS** (`CREATE TABLE AS SELECT`) – convert CSV→Parquet, repartition, compact, create bucketed tables. **`INSERT INTO`** to append. **`UNLOAD`** to export results in Parquet/ORC/etc.
- **Views**, **prepared statements**, **named queries**.
- **Workgroups:** separate users/teams; **per-query & per-workgroup data-scanned limits** (cost control), separate result locations, encryption settings, CloudWatch metrics, **engine version** control.
- **Federated queries:** Lambda-based **data source connectors** query RDS, DynamoDB, Redshift, CloudWatch Logs, on-prem DBs, etc. and **join with S3** in one query.
- **Athena for Apache Spark** (notebooks, interactive Spark; serverless).
- **Table formats:** **Iceberg** (full ACID: INSERT/UPDATE/DELETE/MERGE, time travel `FOR TIMESTAMP AS OF`, `OPTIMIZE`, `VACUUM`), Hudi, Delta (read).
- **Security:** IAM + **Lake Formation** permissions, S3 bucket policies, encrypted results (SSE-S3/KMS/CSE-KMS), VPC endpoints (interface) for private access.
- **Common errors:** `HIVE_PARTITION_SCHEMA_MISMATCH`, `HIVE_BAD_DATA`, `HIVE_CURSOR_ERROR`, S3 `SlowDown` (503 – too many requests to a prefix: spread keys/reduce small files), `Query exhausted resources` (too heavy → partition, reduce joins ordering: **put the larger table on the left, smaller on right** in joins; add `LIMIT`; use approximate functions e.g. `approx_distinct()`), `TABLE_NOT_FOUND`, new partitions invisible (run `MSCK REPAIR TABLE` / `ALTER TABLE ADD PARTITION` / projection).
- **Query tips:** `approx_distinct`, `ORDER BY` with `LIMIT` (avoid global sorts), use `LIKE` carefully, avoid unnecessary window functions on huge sets.

**Example – cost-optimization CTAS (CSV → partitioned Parquet)**
```sql
CREATE TABLE claims_parquet
WITH (
  format = 'PARQUET',
  parquet_compression = 'SNAPPY',
  partitioned_by = ARRAY['claim_year','claim_month'],
  external_location = 's3://insurance-curated/claims_parquet/'
) AS
SELECT claim_id, policy_id, claim_amount, status,
       year(claim_date) AS claim_year, month(claim_date) AS claim_month
FROM claims_raw_csv;
```
(Partition columns must be the **last** columns in the SELECT.)

**Scenario (Insurance audit):** Auditors occasionally query 3 years of claims logs. Keep as **Parquet partitioned by date in S3**, query with **Athena** using a dedicated **workgroup with a 1 TB scan limit** and results encrypted with KMS. Cost drops >90% vs. CSV scans. No cluster idle cost.

**Athena vs Redshift Spectrum:** Athena = serverless ad-hoc. Spectrum = extend an existing Redshift warehouse to S3. **Athena vs Redshift:** occasional/ad-hoc → Athena; frequent, concurrent, complex BI with tuning → Redshift.

---

## 4.2 Amazon QuickSight

**What it is:** Serverless **BI / dashboards** with ML insights, embedded analytics.
- **SPICE** – in-memory engine; import data for fast dashboards (scheduled/incremental refresh; limits per account). vs **Direct Query** (live; hits source each time).
- **Data sources:** S3 (manifest), Athena, Redshift, RDS/Aurora, Snowflake, OpenSearch, Salesforce, etc.
- **Security:** **Row-level security (RLS)** (dataset rules via permissions dataset or tags), **Column-level security (CLS)**, IAM/AD/SSO, **VPC connections** to private data, **Lake Formation** integration, encryption.
- **Features:** Q (natural language), anomaly detection, forecasting, **embedding** (for apps/customers), email reports, alerts, **Paginated reports** (Enterprise).
- **Editions:** Standard vs Enterprise (Enterprise: RLS/CLS, encryption at rest, embedding, AD).
- **Exam traps:** Slow dashboards on huge data → **SPICE** or aggregate in source/materialized view. Show different rows to different users → **RLS**. Private Redshift/RDS source → QuickSight **VPC connection** + security group allowing QuickSight IP range/ENI.

**Other visualization/analytics tools to recognise:** **Athena + Grafana (Amazon Managed Grafana)**, **OpenSearch Dashboards**, **SageMaker Data Wrangler / Canvas**, **Glue DataBrew** (profile/visualize), **Amazon Q in QuickSight**.

---

## 4.3 Monitoring, logging & alerting

### Amazon CloudWatch
- **Metrics** (standard 1-min; high-resolution 1-s; custom via `PutMetricData` or **Embedded Metric Format (EMF)** in logs), **Alarms** (static, anomaly detection, composite) → SNS / Auto Scaling / Lambda / EventBridge, **Dashboards**, **Logs** (log groups/streams, **retention policy**, **metric filters**, **subscription filters** → Kinesis/Firehose/Lambda/OpenSearch, **Logs Insights** query language, **Live Tail**), **Contributor Insights**, **Synthetics**, **Cross-account observability**.
- **Export logs to S3** (batch) or **stream** with subscription filters.

### AWS CloudTrail
- Records **API calls/management events** (who, what, when, from where). **Data events** (S3 object-level, Lambda invoke, DynamoDB item-level) are optional/extra cost. **CloudTrail Lake** = managed SQL-queryable event store. **Organization trails**, **log file integrity validation**, deliver to S3 (+ CloudWatch Logs).
- *Use for:* audit, security investigation, compliance ("who deleted this table/object?").

### Other
- **AWS Config:** resource configuration history & **compliance rules** (e.g., "S3 buckets must be encrypted"), auto-remediation via SSM.
- **AWS X-Ray:** tracing distributed apps (Lambda, API GW).
- **AWS Health / Trusted Advisor / Service Quotas**: limits, recommendations.
- **Amazon Managed Service for Prometheus + Managed Grafana:** container/metrics dashboards.
- **EventBridge rules on service events** for automated alerts (e.g., Glue Job State Change = FAILED → SNS/Lambda/PagerDuty).

### Metrics to know per service
| Service | Metric / signal | Meaning |
|---|---|---|
| Kinesis Streams | `GetRecords.IteratorAgeMilliseconds` | Consumer lag |
| Kinesis Streams | `WriteProvisionedThroughputExceeded`, `ReadProvisionedThroughputExceeded` | Throttled—need more shards/EFO |
| Firehose | `DeliveryToS3.DataFreshness`, `...Success` | Delivery delay/failures |
| Lambda | `Errors`, `Throttles`, `Duration`, `IteratorAge`, `ConcurrentExecutions` | Health & scaling |
| DynamoDB | `ThrottledRequests`, `ConsumedRead/WriteCapacityUnits`, `SystemErrors` | Capacity problems |
| Redshift | `CPUUtilization`, `PercentageDiskSpaceUsed`, `WLMQueueLength`, `DatabaseConnections` | Load & capacity |
| Glue | `glue.driver.aggregate.numFailedTasks`, `bytesRead`, executor memory/CPU | Job health |
| MSK | `CpuUser`, `KafkaDataLogsDiskUsed`, `MaxOffsetLag` | Broker & consumer health |
| EMR | `YARNMemoryAvailablePercentage`, `HDFSUtilization`, `IsIdle` | Cluster capacity |
| S3 | Request metrics (4xx/5xx, latency) | Prefix hot-spots |
| Athena | `DataScannedInBytes`, `EngineExecutionTime` | Cost & performance |

---

## 4.4 Data quality, validation & profiling

**Dimensions of quality:** completeness, uniqueness, validity (format/range), consistency (across systems), accuracy, timeliness/freshness.

**Where & how on AWS**
- **Glue Data Quality (DQDL rules)** – in ETL pipelines or on catalog tables; rule recommendations; anomaly detection; actions: fail job, publish metrics to CloudWatch, route failed rows.
- **Glue DataBrew** – profile jobs & data quality rulesets (no code).
- **Deequ / PyDeequ** (on EMR/Glue) – unit tests for data in Spark.
- **Lambda / Step Functions** validations (schema, null checks, row counts).
- **Redshift** – constraints are informational; enforce quality in ETL SQL (`CASE`, `WHERE`, checks) and verify with count/sum reconciliation queries.
- **DMS validation** – source vs target row comparisons.
- **Athena** queries to profile (nulls, distinct, min/max, duplicates).

**Common data-cleaning tasks:** handle nulls (drop/impute), dedupe (`ROW_NUMBER() OVER (PARTITION BY key ORDER BY updated_at DESC) = 1`), standardize formats/time zones, outlier detection, referential integrity checks, reconcile totals (control totals/hash totals).

**Pattern – quarantine & alert:** valid rows → curated; invalid rows → `s3://…/quarantine/` + error reason column; EventBridge/SNS alert; dashboard on bad-row %; reprocess after fix.

**Data sampling & profiling:** DataBrew profile, Athena `TABLESAMPLE`, Spark `sample()`.

---

## 4.5 SQL you must be comfortable with (Athena / Redshift)

```sql
-- 1) Deduplicate: keep latest record per policy
SELECT * FROM (
  SELECT p.*, ROW_NUMBER() OVER (PARTITION BY policy_id ORDER BY updated_at DESC) rn
  FROM policies p) t
WHERE rn = 1;

-- 2) Running total & ranking (window functions)
SELECT customer_id, txn_date, amount,
       SUM(amount) OVER (PARTITION BY customer_id ORDER BY txn_date
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
       RANK() OVER (PARTITION BY txn_date ORDER BY amount DESC) AS day_rank,
       LAG(amount) OVER (PARTITION BY customer_id ORDER BY txn_date) AS prev_amount
FROM transactions;

-- 3) CTE + aggregation + HAVING
WITH monthly AS (
  SELECT policy_id, date_trunc('month', claim_date) m, SUM(claim_amount) amt
  FROM claims GROUP BY 1,2)
SELECT policy_id, AVG(amt) avg_monthly FROM monthly GROUP BY policy_id HAVING AVG(amt) > 100000;

-- 4) Redshift upsert with MERGE
MERGE INTO dim_customer USING stg_customer s ON dim_customer.cust_id = s.cust_id
WHEN MATCHED THEN UPDATE SET name = s.name, city = s.city
WHEN NOT MATCHED THEN INSERT VALUES (s.cust_id, s.name, s.city);

-- 5) Redshift COPY
COPY sales FROM 's3://bucket/sales/' IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftCopyRole'
FORMAT AS PARQUET;

-- 6) Semi-structured (Redshift SUPER / Athena JSON)
SELECT json_extract_scalar(payload, '$.customer.id') FROM events;   -- Athena
```
**Know:** INNER/LEFT/RIGHT/FULL/CROSS joins, `UNION` vs `UNION ALL`, `GROUP BY` rules, `EXISTS` vs `IN`, `COALESCE`, `CASE`, date functions, `PIVOT`-style conditional aggregation, `EXPLAIN`.

---

## 4.6 Performance tuning playbook (by service)

| Service | Make it faster/cheaper |
|---|---|
| **S3/Athena** | Parquet + partitions + compaction + right file size, partition projection, `SELECT` columns, bucketing |
| **Glue** | Pushdown predicates, bookmarks, groupFiles, G.2X for memory-heavy, auto scaling, Flex, avoid `collect()`, fix skew |
| **EMR** | Spot task nodes, managed scaling, right-size executors, compact files, transient clusters, Graviton |
| **Redshift** | Correct DIST/SORT keys, `COPY` multi-file, ANALYZE/VACUUM, materialized views, WLM/SQA, concurrency scaling, RA3, avoid `SELECT *` & cross joins |
| **DynamoDB** | Good partition key, GSIs for access patterns, DAX, on-demand/auto-scaling, batch ops, avoid Scan |
| **Kinesis** | More shards / on-demand, EFO, KPL aggregation, better partition key, Lambda parallelization |
| **Lambda** | More memory (more CPU), provisioned concurrency, batch size tuning, reuse connections, RDS Proxy |
| **RDS/Aurora** | Read replicas, indexes, RDS Proxy, Aurora Serverless v2, parameter tuning |
| **OpenSearch** | Right shard size (10–50 GB), ISM, UltraWarm for old data, bulk indexing |

**Cost-optimization quick list:** serverless where spiky, Spot (EMR task, Batch), Reserved/Savings Plans for steady (Redshift RA3 RI, OpenSearch RI), S3 lifecycle & Intelligent-Tiering, compress+columnar, delete unused snapshots/EBS, stop/pause idle (Redshift pause), tag resources + **Cost Explorer / Budgets / Cost Allocation Tags / CUR (Cost & Usage Report)**.

---

## 4.7 Troubleshooting playbook — "Symptom → Cause → Fix"

| Symptom | Most likely cause | Best fix |
|---|---|---|
| Kinesis `ProvisionedThroughputExceeded` on producer | Hot shard / too few shards | Better partition key, add/split shards, on-demand, KPL, backoff |
| Kinesis consumer falling behind | Slow processing / shared 2 MB/s | EFO, more shards, Lambda parallelization factor, scale consumers |
| Lambda on Kinesis stuck repeating | Poison record | Bisect on error, max retries/record age, on-failure destination (DLQ) |
| Firehose → S3 tiny files | Small buffer | Increase buffer size/interval; compact later |
| Firehose → Redshift fails | COPY error/IAM/network | Check error prefix in S3 & `STL_LOAD_ERRORS`, security group/IP allow |
| DMS CDC lag high | Small replication instance, large transactions, LOBs | Bigger instance, parallel apply, limited LOB mode, split tasks |
| DMS can't capture changes | Source logging not on | Enable supplemental logging / binlog ROW / logical replication |
| Glue job slow or OOM | Skew, small files, wrong worker | See Glue table above |
| Glue job re-reads all data | Bookmark off | Enable bookmark + `transformation_ctx` |
| Athena slow & expensive | CSV/JSON, no partitions | CTAS to Parquet+partitions; projection |
| Athena new partition missing | Catalog not updated | `MSCK REPAIR`, add partition, projection, crawler |
| Redshift query slow, high network | Redistribution in joins | Align DISTKEY, ALL for small dims, check `EXPLAIN` |
| Redshift disk full | Skew, unvacuumed deletes, big temp spill | VACUUM, fix distribution, resize/RA3, archive to S3/Spectrum |
| Redshift queries queued | Concurrency limits | Auto WLM, concurrency scaling, SQA, Serverless |
| COPY slow | One huge file | Split to multiple compressed files (multiple of slices) |
| DynamoDB throttling | Hot partition / GSI under-provisioned / low WCU | Re-key, on-demand, auto scaling, DAX for reads |
| Lambda timeouts (15 min) | Job too big | Move to Glue/Fargate/EMR; Step Functions Map to chunk |
| Step Function task fails intermittently | Transient errors | Add `Retry` with backoff; `Catch` to fallback |
| S3 503 SlowDown | Too many requests on one prefix | Spread across prefixes, reduce request rate, exponential backoff |
| Access denied reading S3 with KMS | Missing `kms:Decrypt` | Add to role and key policy |
| Cross-account access fails | Needs both sides allowed | Bucket/key policy in owner + IAM in caller (or LF/RAM grant) |
| MWAA DAG not appearing | Syntax error / file not in S3 `dags/` | Fix DAG, check scheduler logs in CloudWatch |
| QuickSight can't connect to private DB | Network | VPC connection, SG inbound from QuickSight ENI |

**Exam mindset for troubleshooting:** (1) Read the symptom → (2) identify service limit or design cause → (3) choose the *smallest* change that solves it (managed feature over custom code).

---

## 4.8 Reliability, HA & DR for data platforms

- **RPO** (how much data loss tolerated) vs **RTO** (how fast to recover). DR strategies: Backup & restore → Pilot light → Warm standby → Multi-site active/active.
- **S3:** versioning + CRR; **DynamoDB:** PITR + Global Tables; **Redshift:** snapshots + cross-region copy + Multi-AZ; **RDS:** Multi-AZ + cross-region replicas; **Aurora:** Global Database; **Kinesis/MSK:** multi-AZ built-in, cross-region via replication (MSK Replicator); **Glue/Step Functions:** infrastructure as code → redeploy in another region; **AWS Backup** centralizes policies (cross-account/region copies, **Vault Lock** WORM).
- **Idempotent pipelines + checkpointing + replayable raw zone** make recovery easy.
- **AWS Fault Injection Service** for chaos testing (awareness only).

---

# 5. DOMAIN 4 — DATA SECURITY AND GOVERNANCE (18%)

## 5.1 Security mental model

**Shared responsibility:** AWS secures the cloud (hardware, managed-service software). **You** secure what's *in* the cloud (IAM, data, encryption choices, network config, classification).

**Four layers to always think of:** **Identity** (who) → **Network** (from where) → **Data** (encryption, masking) → **Audit** (logging, monitoring).

---

## 5.2 IAM (Identity and Access Management)

- **Principals:** users, groups, **roles** (assumed temporarily via **STS**; preferred for services/apps/cross-account), federated identities (SAML/OIDC/IAM Identity Center).
- **Policies:** identity-based (attached to user/role), **resource-based** (S3 bucket policy, KMS key policy, SQS, Lambda – can allow **cross-account** directly), **permissions boundaries**, **SCPs** (Organizations – guardrails, don't grant), **session policies**, VPC endpoint policies.
- **Evaluation logic:** default **deny** → explicit **Allow** needed → **explicit Deny always wins**. Cross-account: **both** the caller's identity policy *and* the resource's policy must allow (except when a role is assumed – then the role's permissions apply).
- **Least privilege:** scope actions, resources, add **conditions** (`aws:SourceIp`, `aws:SecureTransport`, `aws:PrincipalOrgID`, `aws:RequestedRegion`, `s3:prefix`, `kms:ViaService`).
- **Service roles:** Glue role needs S3 + Glue + KMS + CloudWatch; Redshift COPY role needs S3 read (+ KMS decrypt); EMR has **service role + EC2 instance profile**; Lambda **execution role**; DMS needs VPC role.
- **Tools:** IAM Access Analyzer (finds external sharing, generates least-privilege policies from CloudTrail), credential report, MFA, password policy, **IAM Identity Center** (SSO).
- **Best practice:** never use root; roles over long-term keys; rotate; **no hard-coded credentials**.

**PassRole:** To let a user start a Glue job/EMR with a role, they need `iam:PassRole` on that role (common exam trap).

---

## 5.3 AWS KMS & encryption

**Encryption at rest + in transit** are baseline requirements.

- **KMS keys (CMKs/"KMS keys"):** *AWS owned* (invisible), *AWS managed* (`aws/s3`, `aws/redshift`; free, auto-rotated yearly, can't edit policy), **Customer managed** (you control policy, rotation, grants, deletion; $1/mo). **Symmetric** (AES-256, default) vs **asymmetric** (RSA/ECC for sign/verify).
- **Envelope encryption:** KMS encrypts a small **data key**; the data key encrypts the large data (KMS API only handles ≤4 KB directly). `GenerateDataKey`, `Decrypt`.
- **Key policy** is the primary access control (cross-account: allow in key policy **and** IAM policy of the caller). **Grants** give temporary delegated permissions (used by AWS services).
- **Rotation:** customer-managed symmetric keys can auto-rotate (default yearly; configurable); **imported key material** = manual rotation. **Multi-Region keys** for cross-region encrypted copies/DR.
- **Key deletion:** 7–30 days waiting period (scheduled deletion; can cancel). Disable key to stop use.
- **Throttling:** high-volume services calling KMS (e.g., S3 SSE-KMS) can throttle → **S3 Bucket Keys**, data key caching.
- **CloudHSM:** dedicated single-tenant HSM when compliance demands you control the hardware / FIPS 140-2 Level 3; KMS custom key store can use it.

**Encryption per service (quick table)**

| Service | At rest | In transit | Notes |
|---|---|---|---|
| S3 | SSE-S3 (default), SSE-KMS, DSSE-KMS, SSE-C, client-side | HTTPS (enforce via `aws:SecureTransport` deny policy) | Bucket default encryption |
| Redshift | KMS or HSM (cluster-level; can enable later) | SSL (`require_ssl`) | Encrypted snapshots; cross-region copy needs KMS grant |
| RDS/Aurora | KMS at creation | SSL/TLS | Replicas & snapshots inherit |
| DynamoDB | Always encrypted | TLS | AWS-owned / AWS-managed / customer-managed key |
| Kinesis Streams | SSE-KMS (enable) | TLS | |
| Firehose | Uses source/target encryption + KMS option | TLS | |
| MSK | KMS at rest | TLS in-cluster & client–broker | Auth: IAM, SASL/SCRAM, mTLS |
| EMR | Security configuration: S3 (SSE/CSE), EBS/local disk (LUKS/EBS encryption), HDFS TDE | TLS (in-transit) | |
| Glue | Catalog encryption, job bookmarks/logs/S3 output via Security Configuration | TLS | |
| EBS/EFS | KMS | TLS (EFS mount helper) | |
| Lambda env vars | KMS (custom key optional) | n/a | Use Secrets Manager for secrets |
| CloudWatch Logs/CloudTrail | KMS | TLS | |

**Client-side vs server-side:** *Server-side* = AWS encrypts after receipt. *Client-side* = you encrypt before upload (AWS never sees plaintext) – used for strongest control.

---

## 5.4 Secrets, credentials & parameter storage

| | **Secrets Manager** | **Systems Manager Parameter Store** |
|---|---|---|
| Purpose | Credentials/API keys with **automatic rotation** | Config + secrets (SecureString) |
| Rotation | **Built-in (Lambda-based; RDS/Redshift/DocumentDB native)** | Manual/custom |
| Cost | Per secret per month + API calls | Standard tier free; advanced paid |
| Cross-account/replication | Yes (multi-Region replication) | Limited |
| Use when | DB passwords needing rotation | App configuration, simple secrets |

- Glue connections, Redshift, MSK SCRAM, MWAA, DMS endpoints can pull creds from **Secrets Manager**.
- **IAM database authentication** (RDS/Aurora) and **IAM auth for Redshift (`GetClusterCredentials`)** avoid static passwords.

---

## 5.5 Network security for data

- **VPC, subnets (private for data), security groups (stateful, allow-only), NACLs (stateless)**.
- **VPC endpoints:**
  - **Gateway endpoints** (free): **S3 and DynamoDB only** — route-table based.
  - **Interface endpoints (PrivateLink)**: Kinesis, Glue, Athena, STS, KMS, Secrets Manager, SQS, SNS, etc. — ENI in your subnet, costs per hour + GB.
- **Endpoint policies** restrict which buckets/actions are reachable.
- **Glue jobs in VPC:** need subnet with route to S3 (gateway endpoint) and **self-referencing security group**; to reach internet/other services → NAT or interface endpoints.
- **Redshift:** private subnet, **enhanced VPC routing** (COPY/UNLOAD/Spectrum traffic stays in your VPC and obeys endpoint/NACL/flow logs), **publicly accessible = false**.
- **Lambda in VPC** → NAT gateway or endpoints for AWS APIs.
- **Hybrid:** Site-to-Site VPN, **Direct Connect** (+ MACsec/VPN for encryption).
- **VPC Flow Logs** (network audit), **AWS WAF/Shield** (web edge), **PrivateLink** to expose services privately, **Network Firewall**.
- **S3 Block Public Access** + bucket policy `aws:SourceVpce` to restrict to a VPC endpoint.

---

## 5.6 AWS Lake Formation (governance for the lake) — high exam yield

**What it is (easy):** A central place to **grant/revoke access to data lake databases, tables, columns, rows, cells** (instead of fiddling with S3 bucket policies + IAM for each user). Works on top of **Glue Data Catalog + S3**.

**Key concepts**
- **Register S3 locations** with Lake Formation (using a service-linked or custom role) → LF vends temporary credentials to engines when authorized.
- **Permissions model:** `GRANT SELECT ON TABLE ... (columns) TO principal`. Also DESCRIBE, ALTER, INSERT, DELETE, DROP, CREATE_TABLE, **DATA_LOCATION_ACCESS**, grant option.
- **Fine-grained controls:** **column-level**, **row-level**, **cell-level** via **Data Filters**.
- **LF-Tags (tag-based access control, TBAC / LF-TBAC):** tag databases/tables/columns (e.g., `sensitivity=PII`, `domain=claims`), then grant by tag expression → scales better than named-resource grants (preferred for large/many tables).
- **Cross-account sharing:** grant to another account (via **AWS RAM**); recipient creates **resource links**; works with **LF-tags** too.
- **Integrates with:** Athena, Redshift Spectrum, EMR, Glue ETL, QuickSight, SageMaker.
- **Hybrid access mode:** allows gradual migration from IAM/S3-based permissions to Lake Formation.
- **Two permission layers:** IAM must allow the *API* (e.g., `glue:GetTable`, `lakeformation:GetDataAccess`) **and** Lake Formation must grant data permission. If the user "has S3 access but gets access denied in Athena" → check LF grants.
- **Audit:** CloudTrail logs all LF permission calls & data access.

**Scenario (Insurance):** Analysts should see claims data but **not** columns `aadhaar_no` / `phone`, and regional managers should only see **rows for their state**. Use LF **LF-tag `pii=true`** on sensitive columns (deny to analyst group) and **row filter `state = 'MH'`** for the Maharashtra manager group. All queries via Athena/Redshift Spectrum automatically honor this — no duplicate datasets needed.

**Exam traps**
- "Column/row level access in data lake with least effort" → **Lake Formation**.
- Don't try to implement row-level security with S3 prefix policies.
- Cross-account lake sharing → LF + RAM (not copying data).

---

## 5.7 Privacy, PII & data protection

- **Amazon Macie:** ML-based **sensitive data discovery in S3** (PII, financial data, credentials); findings → Security Hub/EventBridge; automated sensitive-data discovery; custom identifiers.
- **Glue sensitive data detection** / **DataBrew** to detect & redact/hash/mask inside pipelines.
- **Redshift dynamic data masking**, **column-level grants**, **RLS**.
- **Techniques:** masking, **hashing** (SHA-256 with salt), **tokenization**, encryption, **pseudonymization**, anonymization/k-anonymity, generalization (age bands), differential privacy (concept), **format-preserving** tokens.
- **Data classification** (public/internal/confidential/restricted) driven by tags → policy.
- **AWS Clean Rooms** for privacy-preserving collaboration.
- **Regulations:** GDPR, HIPAA, PCI-DSS, SOC, ISO — and for BFSI in India **RBI data localisation, IRDAI guidelines, DPDP Act** (consent, purpose limitation, erasure, breach notification, data residency). Use **AWS Artifact** for compliance reports.
- **Data sovereignty / residency:** restrict Regions with **SCP** (`aws:RequestedRegion`), avoid cross-region replication, regional endpoints, keep KMS keys in-region.
- **Right to erasure:** design raw/curated with identifiers + Iceberg deletes, vacuum/expire snapshots; track erasure requests in a ledger table.
- **Data retention vs deletion conflict:** use classification + legal hold + retention policies (Object Lock, lifecycle).

---

## 5.8 Authentication & authorization patterns for data services

| Service | Auth options |
|---|---|
| Redshift | IAM (`GetClusterCredentials`), Secrets Manager, federated SSO (SAML/IdP via IAM Identity Center), database users/groups/roles; **IAM roles for COPY/Spectrum** |
| RDS/Aurora | Password, **IAM DB auth**, Kerberos (Directory Service) |
| MSK | IAM, SASL/SCRAM (Secrets Manager), mutual TLS, ACLs |
| Athena/Glue | IAM + Lake Formation |
| OpenSearch | **Fine-grained access control** (internal users, IAM, SAML, Cognito), VPC, resource policies |
| QuickSight | IAM/AD/SSO, RLS/CLS |
| DynamoDB | IAM with conditions (`LeadingKeys`), federated via Cognito |
| API/app users | **Amazon Cognito** (user pools = auth; identity pools = AWS creds) |
| Cross-account | Assume role (STS), resource policies, RAM, LF grants |

---

## 5.9 Auditing & compliance logging

- **CloudTrail** (management + data events; CloudTrail Lake; org trail; validation) – *who did what*.
- **CloudWatch Logs** (app/engine logs; retention; metric filters; KMS) – *what happened inside*.
- **S3 server access logs** (request-level, best effort) / CloudTrail S3 data events (reliable object-level).
- **Redshift audit logs:** connection log, user log, user activity log → S3 or CloudWatch.
- **VPC Flow Logs, ALB/CloudFront logs, WAF logs, Route 53 resolver logs.**
- **AWS Config** (configuration compliance), **Security Hub** (aggregated findings), **GuardDuty** (threat detection incl. S3/Runtime), **Inspector** (vulnerabilities), **Detective** (investigation), **Audit Manager** (evidence collection), **Macie** (data).
- **Query logs with Athena:** create tables over CloudTrail/ALB/VPC Flow/S3 logs in S3 (partition projection by date) → ad-hoc audit & investigation.
- **Centralize** logs in a dedicated **log-archive account** with Object Lock; use **Organizations + SCPs + Control Tower** for guardrails.
- **Data lineage & provenance:** who created/changed datasets (Glue run history, DataZone lineage, CloudTrail).

**Scenario (Bank audit):** Regulator asks "who accessed customer table X last quarter?" → CloudTrail Lake query on Lake Formation `GetDataAccess` + Athena `StartQueryExecution` events, Redshift user activity logs in S3 queried by Athena; evidence stored in a WORM bucket.

---

## 5.10 Governance frameworks on AWS (summary)

- **Data catalog + ownership + classification + quality + lineage + access policy + retention + audit** = governance.
- **Preventive** (IAM, SCP, LF, encryption), **detective** (CloudTrail, Config, Macie, GuardDuty), **corrective** (auto-remediation via EventBridge+Lambda/SSM, Config rules).
- **Multi-account strategy:** Org units (prod/non-prod/security/log-archive/data-lake/data-producer accounts); **data mesh**: domain-owned data products published via DataZone/LF cross-account sharing, central governance.

---

# 6. END-TO-END REAL-WORLD ARCHITECTURES (study these as templates)

## 6.1 Bank: Real-time fraud + analytics lakehouse

```
Card swipes/UPI events → API GW/app → Kinesis Data Streams (PK=account_id, on-demand)
   ├─> Lambda (fraud scoring using DynamoDB features, SageMaker endpoint) → SNS alert / block
   ├─> Managed Flink (5-min sliding windows) → OpenSearch (live fraud dashboard)
   └─> Firehose (Parquet, dynamic partition by date) → S3 raw → Glue ETL → S3 curated (Iceberg)
Core banking Oracle 19c → DMS (full load + CDC) → S3 / MSK → Glue/EMR merge → Iceberg
S3 curated → Athena (analysts), Redshift Spectrum/COPY (BI marts) → QuickSight
Security: KMS CMKs, Lake Formation (PII column tags), Macie on raw bucket, CloudTrail Lake, VPC endpoints, SCP region lock
Ops: Step Functions + EventBridge, CloudWatch alarms (IteratorAge, DQ failures), SNS to on-call
```
*Why this answers exam questions:* real-time + lake dual path, replayable stream, CDC without hitting prod, fine-grained access, audit trail.

## 6.2 Insurance: Claims processing data platform

```
Hospitals/partners SFTP → Transfer Family → S3 landing → S3 event → EventBridge → Step Functions
   → Lambda (Textract/OCR, validation) → Glue ETL (cleanse, mask PII, DQDL checks) → S3 curated (Parquet)
   → Glue Catalog → Athena (ad-hoc), Redshift (claims mart: SCD-2 policy dim, DISTKEY(policy_id), SORTKEY(claim_date))
   → QuickSight (RLS by region) ; Fraud models via SageMaker; alerts via SNS
Retention: Object Lock (compliance) for regulatory records; lifecycle to Glacier; DPDP erasure via Iceberg deletes
```

## 6.3 Large employee/enterprise data platform (600k+ users)

```
HR/ERP/Timesheet systems (RDS/Oracle/SaaS) → DMS CDC + AppFlow → MSK/S3 raw
Glue/EMR Spark → conformed dimensions (employee, org, project) → Redshift RA3 (data sharing to teams) + S3 Iceberg
DataZone for data products; LF-tags for HR-sensitive attributes; QuickSight embedded dashboards (RLS by manager hierarchy)
DynamoDB for low-latency profile lookups (DAX) ; OpenSearch for people-search
```

## 6.4 Hadoop on-prem → AWS modernization

| On-prem | AWS |
|---|---|
| HDFS | **S3** (+ Iceberg) |
| Hive metastore | **Glue Data Catalog** |
| Hive/Spark jobs | **EMR (transient/Serverless)** or **Glue** |
| Oozie / Airflow | **MWAA / Step Functions** |
| Kafka | **MSK** |
| Sqoop | **DMS / Glue JDBC** |
| Impala/Presto | **Athena / Redshift** |
| Ranger | **Lake Formation / IAM** |
| HBase | **DynamoDB / EMR HBase** |

---

# 7. QUICK-DECISION CHEAT SHEETS

## 7.1 "Which service?" keyword map (EXAM GOLD)

| Keyword in question | Think |
|---|---|
| Real-time, ordered, replay, multiple consumers | **Kinesis Data Streams / MSK** |
| Near-real-time delivery to S3/Redshift/OpenSearch, no code | **Firehose** |
| Stateful windowing / real-time analytics | **Managed Flink** |
| Existing Kafka | **MSK** |
| Replicate database continuously / migrate DB | **DMS (+ SCT if heterogeneous)** |
| SaaS data into AWS | **AppFlow** |
| SFTP | **Transfer Family** |
| On-prem file bulk/scheduled | **DataSync** |
| Petabytes, no bandwidth | **Snowball Edge** |
| Serverless ETL / catalog / crawler | **Glue** |
| No-code data prep | **DataBrew** |
| Hadoop/Spark/Hive control, Spot savings | **EMR** |
| Serverless ad-hoc SQL on S3 | **Athena** |
| Data warehouse, BI, joins, high concurrency | **Redshift** |
| Query S3 from Redshift | **Spectrum** |
| Share warehouse data live without copy | **Redshift data sharing** |
| Single-digit-ms NoSQL | **DynamoDB** |
| Cache for DynamoDB (µs) | **DAX** |
| Full-text search, log analytics | **OpenSearch** |
| Time-series | **Timestream** |
| Graph | **Neptune** |
| Orchestrate AWS services, retries, branching | **Step Functions** |
| Airflow DAGs | **MWAA** |
| Schedule / react to events | **EventBridge** |
| Dashboards / BI | **QuickSight** |
| Column/row-level lake security | **Lake Formation** |
| Find PII in S3 | **Macie** |
| Who called API / audit | **CloudTrail** |
| Config compliance / drift | **AWS Config** |
| Rotate DB passwords | **Secrets Manager** |
| Encrypt, key control | **KMS** (hardware control → CloudHSM) |
| WORM / immutability | **S3 Object Lock** (compliance mode) / Backup Vault Lock |
| Data quality rules | **Glue Data Quality (DQDL)** |
| Business data catalog / data products | **DataZone** |
| Streaming schema enforcement | **Glue Schema Registry** |

## 7.2 "Least operational overhead" ladder
Fully serverless managed feature > managed service with config > self-managed on EC2.
(Firehose > KDS+Lambda > KDS+KCL on EC2; Athena > Redshift > EMR on EC2; EMR Serverless > EMR EC2; Redshift Serverless > provisioned; Step Functions > custom orchestrator.)

## 7.3 Numbers & limits flashcards

| Fact | Value |
|---|---|
| Kinesis shard write | 1 MB/s or 1,000 rec/s |
| Kinesis shard read | 2 MB/s shared; EFO 2 MB/s per consumer |
| Kinesis retention | 24 h (default) → 365 days |
| Kinesis record max | 1 MB |
| Firehose buffer interval | up to 900 s |
| Lambda max timeout / memory | 15 min / 10,240 MB |
| Step Functions Standard / Express max | 1 year / 5 min |
| SQS message max | 256 KB (use S3 for larger — extended client) |
| S3 max object / single PUT | 5 TB / 5 GB |
| S3 request rate per prefix | 3,500 write / 5,500 read per second |
| S3 min storage duration | IA/One Zone-IA 30 d; GIR/GFR 90 d; Deep Archive 180 d |
| DynamoDB item max | 400 KB |
| DynamoDB RCU / WCU | 4 KB strongly consistent read / 1 KB write |
| DynamoDB PITR | 35 days |
| DynamoDB Streams retention | 24 h |
| DynamoDB LSI partition limit | 10 GB per PK value |
| Redshift snapshot retention | up to 35 days (automated) |
| RDS automated backup retention | up to 35 days |
| Aurora read replicas | up to 15 |
| Glue DPU | 4 vCPU + 16 GB |
| KMS direct encrypt size | 4 KB |
| KMS key deletion wait | 7–30 days |
| CloudWatch Logs default retention | Never expire (set it!) |
| Athena price model | per TB scanned |

---

# 8. PRACTICE QUESTIONS WITH REASONING (learn the *thinking*)

**Q1.** A bank needs to ingest 50,000 card events/s, process fraud rules in under a second, and also store everything in S3 for later analysis with the least operational overhead. What architecture?
**A:** Kinesis Data Streams (on-demand) → Lambda for fraud rules; second consumer Firehose → S3 (Parquet). *Why:* real-time + replay + dual consumers; Firehose handles lake landing without code.

**Q2.** Athena queries on 5 TB of CSV logs are slow and expensive. Queries filter by date and select 3 columns. Best fix?
**A:** Convert to Parquet (Snappy) partitioned by date using Glue/CTAS. *Why:* columnar + partition pruning cut scanned bytes dramatically.

**Q3.** A Glue job runs daily and reprocesses all historical files, wasting cost. Fix?
**A:** Enable job bookmarks (with `transformation_ctx`) and use pushdown predicates on partitions.

**Q4.** Redshift joins between `fact_claims` (2B rows) and `dim_policy` (500M rows) are slow with heavy network traffic. Fix?
**A:** Set both tables `DISTSTYLE KEY` on `policy_id` (co-located join). Small dims → `DISTSTYLE ALL`.

**Q5.** A consumer of a Kinesis stream shows rising `IteratorAgeMilliseconds`; three apps read the same stream. Fix with minimal change?
**A:** Enable **enhanced fan-out** for consumers (and/or add shards). *Why:* shared 2 MB/s per shard is exhausted.

**Q6.** Analysts must see claims data, but some columns hidden and regional managers should see only their region rows, across Athena and Redshift Spectrum. Solution?
**A:** Lake Formation with column permissions/LF-tags + row filter data filters.

**Q7.** Compliance requires records to be unmodifiable and undeletable for 7 years, even by the root user. Solution?
**A:** S3 Object Lock in **Compliance mode** with 7-year retention (versioning on).

**Q8.** Migrate on-prem Oracle to Aurora PostgreSQL with minimal downtime.
**A:** SCT (schema/code conversion) + DMS full load + CDC; cutover when lag ~0.

**Q9.** Lambda processing a Kinesis shard keeps failing on one bad record and the shard is blocked.
**A:** Configure bisect batch on error, max retry attempts / max record age, and an on-failure destination (SQS/SNS/S3 DLQ).

**Q10.** DynamoDB table throttles during a flash sale even though total capacity is enough.
**A:** Hot partition key. Redesign key/write-shard; use on-demand; DAX for read-heavy items.

**Q11.** Need to load 200 GB of data into Redshift daily as fast as possible.
**A:** Split into many compressed files (multiple of slices) in S3 and run a single `COPY` using an IAM role (manifest if needed).

**Q12.** A Step Functions workflow must wait for a Glue job to finish before continuing.
**A:** Use the `.sync` service integration (`arn:aws:states:::glue:startJobRun.sync`).

**Q13.** Encrypt data in S3 with keys the company controls and audit each key use, while avoiding high KMS cost for millions of objects.
**A:** SSE-KMS with customer-managed key + **S3 Bucket Keys**; CloudTrail logs KMS usage.

**Q14.** Team needs to run existing Airflow DAGs on AWS with minimal changes.
**A:** Amazon MWAA.

**Q15.** Query Redshift data and live PostgreSQL data together without ETL.
**A:** Redshift **federated query** (and Spectrum for S3).

**Q16.** Processing nightly Spark job needing Hive/Presto and 70% cost reduction.
**A:** EMR transient cluster with **Spot task nodes**, data in S3 via EMRFS, Glue Catalog as metastore (or EMR Serverless).

**Q17.** Unknown/changing S3 access patterns and want to reduce storage costs automatically.
**A:** S3 Intelligent-Tiering.

**Q18.** Detect PII in thousands of S3 buckets and alert.
**A:** Amazon Macie → EventBridge/Security Hub → SNS.

**Q19.** Need to ensure Glue jobs reach private RDS but not the internet.
**A:** Glue connection in private subnet + security groups (self-referencing) + S3 gateway endpoint; no NAT needed if only S3/VPC resources used.

**Q20.** Real-time dashboards from clickstream with 60-second freshness and minimal management.
**A:** Firehose → OpenSearch (or S3 + Athena/QuickSight with SPICE refresh).

---

# 9. 6-WEEK STUDY PLAN (adapt to your time)

| Week | Focus | Hands-on |
|---|---|---|
| 1 | S3, IAM, KMS, Glue Catalog, Athena, file formats | Build a data lake: CSV → Parquet via CTAS, partition, Lifecycle |
| 2 | Glue ETL/crawlers/bookmarks/DQ, Lambda, Step Functions, EventBridge | Event-driven pipeline S3 → Step Functions → Glue → Athena |
| 3 | Kinesis, Firehose, MSK, Flink, DMS | Stream → Firehose → S3 Parquet; DMS RDS → S3 CDC |
| 4 | Redshift (design/COPY/Spectrum/WLM), DynamoDB, RDS/Aurora, OpenSearch | Load TPC-style data, tune dist/sort keys, DynamoDB GSIs/Streams |
| 5 | EMR, MWAA, Lake Formation, Macie, Secrets Manager, CloudWatch/CloudTrail, QuickSight | LF column/row filters, QuickSight RLS, alarms |
| 6 | Full practice exams, weak-area review, flashcards (section 7.3) | Review troubleshooting table (4.7) daily |

**Exam-day tactics**
1. Read the *last sentence* first (what is asked: cheapest / fastest / least ops / most secure).
2. Eliminate options that are self-managed, over-engineered, or violate a stated constraint.
3. Watch **qualifiers**: "real-time", "near-real-time", "minimal code change", "without moving data", "cross-account", "audit", "immutable".
4. If two answers work, prefer the **more managed/serverless** one unless a requirement forces control.
5. Flag & skip long questions; return later. Use all 130 minutes.

**Final revision checklist**
- [ ] Kinesis limits, EFO, Lambda mapping settings
- [ ] Firehose vs KDS vs MSK vs Flink
- [ ] DMS types, SCT, CDC prerequisites
- [ ] Glue: bookmarks, pushdown, DQDL, workers, crawlers, catalog
- [ ] Parquet/ORC/Avro, compression, Iceberg
- [ ] S3 classes, lifecycle, Object Lock, replication, partitioning
- [ ] Redshift DIST/SORT, COPY, Spectrum, WLM, data sharing, Serverless
- [ ] DynamoDB keys, RCU/WCU math, GSI/LSI, Streams, TTL, DAX
- [ ] Athena cost levers, CTAS, partition projection, workgroups
- [ ] Step Functions vs MWAA vs EventBridge
- [ ] Lake Formation (LF-tags, filters, cross-account)
- [ ] KMS (key policy, envelope, rotation), Secrets Manager, VPC endpoints
- [ ] CloudWatch/CloudTrail/Config/Macie roles
- [ ] SCD types, star schema, schema evolution

> **Tip:** Verify exact current limits/features on the official exam guide and service docs just before the exam — AWS services evolve quickly (e.g., newer Redshift/Glue/zero-ETL features), but the *design reasoning* in these notes stays the same.

**Good luck — you've got this! 🚀**
