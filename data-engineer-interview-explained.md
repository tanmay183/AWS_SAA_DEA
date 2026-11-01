# Data Engineer Interview — Full Transcript Explained (Section by Section)

> **What this is:** A line-by-line, topic-by-topic breakdown of a recorded Data Engineering interview (AWS-focused, banking/insurance background, ~5 years experience).
> For **every question** you get: ① what was asked, ② what the candidate answered (in plain words), ③ what was right / partial / wrong, ④ the deep technical explanation, ⑤ the **ideal answer** you should give in a real interview.
>
> **Legend:** ✅ correct · ⚠️ partially correct / incomplete · ❌ incorrect or missed · 💡 ideal answer · 🧠 concept explained

---

## 📑 Table of Contents

0. [Overview, interview flow, and transcript notes](#0-overview-interview-flow-and-transcript-notes)
1. [Round opening: self-introduction](#1-round-opening-self-introduction)
2. [Architecture deep-dive: Oracle → DMS → S3 → Redshift](#2-architecture-deep-dive-oracle--dms--s3--redshift)
3. [Data volume, CDC, and retention](#3-data-volume-cdc-and-retention)
4. [S3 versioning question ("overwrite but don't lose old data")](#4-s3-versioning-question)
5. [Reconciliation process](#5-reconciliation-process)
6. [Handling discrepancies and reprocessing](#6-handling-discrepancies-and-reprocessing)
7. [Notifying consumers (SLA / freshness)](#7-notifying-consumers-sla--freshness)
8. [S3 global namespace and securing a bucket](#8-s3-global-namespace-and-securing-a-bucket)
9. [Protecting PII data](#9-protecting-pii-data)
10. [File formats, small-files problem, CSV → Parquet](#10-file-formats-small-files-problem-csv--parquet)
11. [Schema drift / schema evolution](#11-schema-drift--schema-evolution)
12. [Glue vs EMR vs EMR Serverless; instance selection](#12-glue-vs-emr-vs-emr-serverless-instance-selection)
13. [Glue Crawler and automated schema-change detection](#13-glue-crawler-and-automated-schema-change-detection)
14. [DynamoDB + Elasticsearch: why two databases?](#14-dynamodb--elasticsearch-why-two-databases)
15. [Sharding vs partitioning](#15-sharding-vs-partitioning)
16. [DynamoDB secondary indexes (LSI vs GSI)](#16-dynamodb-secondary-indexes-lsi-vs-gsi)
17. [Scenario design: ad-hoc S3 arrivals + 30-min SLA + audit + lineage](#17-scenario-design-ad-hoc-s3-arrivals--30-min-sla--audit--lineage)
18. [CI/CD and infrastructure deployment](#18-cicd-and-infrastructure-deployment)
19. [Lambda's 15-minute limit — how to continue processing](#19-lambdas-15-minute-limit)
20. [Athena questions (partitions, loading, cross-region)](#20-athena-questions)
21. [Live SQL problem: top-selling product per category](#21-live-sql-problem-top-selling-product-per-category)
22. [Interviewer's feedback explained](#22-interviewers-feedback-explained)
23. [Scorecard and improvement plan](#23-scorecard-and-improvement-plan)
24. [Model answers cheat sheet (30-second versions)](#24-model-answers-cheat-sheet)
25. [Follow-up questions the interviewer could have asked](#25-follow-up-questions-the-interviewer-could-have-asked)
26. [Glossary and transcript term corrections](#26-glossary-and-transcript-term-corrections)

---

# 0. Overview, interview flow, and transcript notes

## 0.1 Interview at a glance

| Item | Detail |
|---|---|
| Role | Data Engineer (AWS-heavy) |
| Candidate background | ~5 years; banking & insurance data; ETL, data migration, AWS (Redshift, S3, DMS, Glue, Step Functions, SNS, Lambda), Python, SQL |
| Round type | Technical round: project walkthrough → AWS/data-engineering concepts → scenario design → live SQL |
| Interview style | Conversational "drill-down": every answer triggers a follow-up that goes one level deeper |

## 0.2 Flow of the interview (in the exact order it happened)

| # | Topic | Type |
|---|---|---|
| 1 | Self-introduction | Warm-up |
| 2 | Project architecture (Oracle → DMS → S3 → Redshift) | Project deep-dive |
| 3 | DMS data volume, CDC, retention | Project deep-dive |
| 4 | S3 versioning (disguised as a scenario) | Concept |
| 5 | Reconciliation: how, when, what to do on mismatch | Project / process |
| 6 | Telling consumers about SLA breaches | Process |
| 7 | S3 global namespace; securing S3 | Concept |
| 8 | PII protection design | Security design |
| 9 | CSV → Parquet transformation, schema drift | Project deep-dive |
| 10 | Glue Catalog vs manual schema comparison | Concept |
| 11 | Glue vs EMR vs EMR Serverless, instance sizing | Concept |
| 12 | Glue Crawler schema-change automation | Design |
| 13 | DynamoDB + Elasticsearch use case | Project deep-dive |
| 14 | Sharding vs partitioning; DynamoDB indexes | Concept |
| 15 | Scenario: ad-hoc S3 files, 30-minute SLA, audit, lineage | System design |
| 16 | Deployment / CI-CD | Process |
| 17 | Lambda 15-minute limit workaround | Design |
| 18 | Athena partitions / load / cross-region | Concept |
| 19 | Live SQL (3 tables, top product per category) | Coding |
| 20 | Feedback | Closing |

## 0.3 About the transcript text

The transcript is **auto-generated speech-to-text**, so several words are garbled. I interpret them as follows (details in [Glossary](#26-glossary-and-transcript-term-corrections)):

| In transcript | Meant |
|---|---|
| "orle" | Oracle |
| "ews" | AWS |
| "park" / "parquet" | Parquet |
| "Dynam" | DynamoDB |
| "genin" / "genkins" | Jenkins |
| "rle number" | ROW_NUMBER |
| "Cher data" | likely "sensitive/customer data" (the point is PII) |
| "write Ahead logs" | Oracle **redo logs** (the candidate's term is a PostgreSQL term) |
| "InfoWorks" | Infoworks (a data ingestion/orchestration tool) |
| "Exential", "quantify" | Employer names (garbled) |

---

# 1. Round opening: self-introduction

### What was asked
"Introduce yourself and the project you worked on."

### What the candidate said
- ~5 years in data, banking + insurance.
- Early career: ETL, data migration, implementation projects.
- Later: end-to-end data solutions on AWS — **Redshift, S3, DMS**.
- Strong in data modeling, data warehousing, Python, SQL.

### Assessment ⚠️
Good structure (years → domain → tools → languages) but **too generic**. The interviewer immediately had to ask: "mention the architecture, source, target, transformations" — meaning the intro didn't contain enough hooks.

### 🧠 How to make an intro that controls the interview
An introduction is **your chance to choose the questions you want to be asked**. Mention 3–4 things you can defend in depth, with a measurable number.

### 💡 Ideal 60-second intro (template)
> "I'm a data engineer with ~5 years of experience, mostly in insurance and banking. My most recent project was building an end-to-end incremental pipeline for an insurance platform: Oracle transaction data is captured using AWS DMS with CDC into an S3 raw zone, converted to Parquet, exposed through Redshift Spectrum, and modelled into an ODS plus dimensional facts and dimensions in Redshift, orchestrated with Step Functions, with daily reconciliation and SNS alerting. I handled roughly **X GB/day across Y tables**, with an SLA of **Z**. I'm strongest in SQL, Python, and AWS data services, and I've also worked with DynamoDB and OpenSearch for a mandate-processing use case."

Notice: it names architecture, tools, volume, SLA, and a second project — every phrase invites a follow-up you're prepared for.

---

# 2. Architecture deep-dive: Oracle → DMS → S3 → Redshift

### What was asked
"Mention the architecture: source, where data was stored, what transformations."

### What the candidate said (decoded)

```
 Oracle (OLTP, insurance transactions)
        │  (CDC from redo logs)
        ▼
  AWS DMS  ────────►  S3 RAW zone (CSV files, one batch of change files at a time)
                              │
                              │ Python: merge many small CSVs → convert to Parquet
                              ▼
                      S3 "Aggregate" zone (Parquet, compressed)
                              │
                              │ Redshift Spectrum external tables (add date partition daily)
                              ▼
                  Redshift ODS layer (exact replica of source = source of truth)
                              │  Stored procedures + SQL (MERGE, quality checks)
                              ▼
                  Redshift Dimensions + Facts (consumed by client)
        Orchestration: AWS Step Functions (ECS tasks, logs in CloudWatch)
        Notification: SNS;  Reconciliation: SQL KPI checks (policy count, premium)
```

### 🧠 Concept 1 — OLTP vs OLAP (why this pipeline exists)
- **Oracle** is an **OLTP** system: many small transactions (insert a policy, update a claim). Optimized for row-by-row writes and reads.
- **Redshift** is an **OLAP** warehouse: optimized for large scans and aggregations over columns.
- You never run analytics on the OLTP system (it slows down business operations), so you **replicate** data to a warehouse.

### 🧠 Concept 2 — What is AWS DMS?
**AWS Database Migration Service** = a managed service that moves data from a **source endpoint** to a **target endpoint** using a **replication instance** (a managed EC2-like server that runs your replication **task**).

Three task types:

| Task type | What it does |
|---|---|
| **Full load** | One-time copy of all existing rows (the "history load") |
| **CDC only** | Continuously copies only changes after a start point |
| **Full load + CDC** | Full copy first, then keeps syncing changes |

### 🧠 Concept 3 — CDC (Change Data Capture) from Oracle
- Oracle writes every change to **redo logs** (and archived redo logs). The candidate said "write-ahead logs" — that term belongs to PostgreSQL (WAL). In an interview, say **redo logs** for Oracle.
- DMS reads these logs using **Oracle LogMiner** (default) or **Binary Reader** (faster, lower impact for high-volume sources).
- Prerequisites on Oracle: database in **ARCHIVELOG mode**, **supplemental logging** enabled (so updates carry the primary key / changed columns).
- Why log-based CDC beats "SELECT where updated_at > last_run": it captures **deletes**, doesn't depend on a reliable timestamp column, and puts almost no query load on the source.

### 🧠 Concept 4 — What DMS writes to S3
When the target is S3, DMS writes **files** (CSV by default, Parquet optional):
- A **full-load** file per table (`LOAD00000001.csv`).
- **CDC files** named by timestamp, containing change rows. With the right settings each row has an **`Op` column**: `I` (insert), `U` (update), `D` (delete), and optionally a **commit timestamp** column.
- File size/batching is controlled by settings such as `CdcMaxBatchInterval` (how long to wait before writing a file) and `CdcMinFileSize`. So files are produced **per batch interval**, not strictly "one file per transaction" as the candidate said — but the practical outcome (very many small files) is the same.

### 🧠 Concept 5 — S3 zones (data lake layering)
| Zone | Candidate's name | Purpose |
|---|---|---|
| Raw / Landing / Bronze | RAW zone | Untouched copy of what DMS delivered; replayable source |
| Curated / Silver | "Aggregate" zone | Cleaned, merged, Parquet, partitioned |
| Consumption / Gold | Redshift facts/dims | Business-ready models |

### 🧠 Concept 6 — Redshift Spectrum
- A Redshift feature that lets you query **files in S3 without loading them** via **external tables**.
- Needs an **external schema** pointing to a catalog (Glue Data Catalog or Hive metastore) and an IAM role.
- Spectrum scans in a separate fleet of Spectrum nodes, so it can handle huge S3 data cheaply.
- Candidate's pattern: each day add **today's partition** (`ALTER TABLE ... ADD PARTITION (load_date='2026-11-01') LOCATION 's3://.../load_date=2026-11-01/'`) to the external table and read only that day's data → `INSERT/MERGE` into ODS.

### 🧠 Concept 7 — ODS layer
- **ODS (Operational Data Store)** = a near-real-time, **row-level replica of the source** with minimal transformation.
- Benefits: (1) a stable "source of truth" inside the warehouse; (2) you can **rebuild facts/dims** without hitting Oracle again; (3) easy reconciliation against source.

### 🧠 Concept 8 — Applying CDC rows into the ODS (what the "MERGE statements" do)
CDC files contain **many versions of the same row** (insert, then three updates, maybe delete). You must keep only the **latest change per primary key**, then apply it.

```sql
-- 1) Latest change per key from the external (Spectrum) table
CREATE TEMP TABLE latest_changes AS
SELECT *
FROM (
  SELECT c.*,
         ROW_NUMBER() OVER (PARTITION BY policy_id ORDER BY commit_ts DESC) AS rn
  FROM spectrum_schema.policy_cdc c
  WHERE load_date = '2026-11-01'
) x
WHERE rn = 1;

-- 2) Apply to ODS (Redshift supports MERGE)
MERGE INTO ods.policy
USING latest_changes s
ON ods.policy.policy_id = s.policy_id
WHEN MATCHED AND s.op = 'D' THEN DELETE
WHEN MATCHED AND s.op IN ('U','I') THEN UPDATE SET
     premium = s.premium, status = s.status, updated_ts = s.commit_ts
WHEN NOT MATCHED AND s.op <> 'D' THEN INSERT (policy_id, premium, status, updated_ts)
     VALUES (s.policy_id, s.premium, s.status, s.commit_ts);
```

Key point: this makes the load **idempotent** — re-running the same day gives the same result.

### 🧠 Concept 9 — Dimensional modelling (facts and dimensions)
- **Fact table:** measurable events (premium paid, claim amount). Narrow, huge, numeric, with foreign keys.
- **Dimension table:** descriptive context (policy, customer, product, date). Wide, smaller.
- **Star schema:** one fact surrounded by dimensions → simple, fast joins for BI.

### Assessment ✅
Architecture was **coherent and realistic** (CDC → raw → curated → ODS → dimensional model, orchestration, reconciliation, alerting). Good use of layers and the ODS rationale.

### 💡 What would have scored higher
- Mention **volume, SLA, number of tables, and run times** upfront.
- Mention **idempotency** (re-runnable loads), **error handling**, and **how deletes are handled** in CDC.
- Mention **Redshift design choices**: distribution keys, sort keys, compression (ENCODE AUTO), WLM.

---

# 3. Data volume, CDC, and retention

### What was asked
1. "How much data load, what tables?" 
2. "Do you set any retention? Will the raw data always be in S3 Standard?"

### What the candidate said
- Many insurance tables (policy, risk, coverage, claims — different granularities).
- CDC means **incremental volume is small**, but **history load** via DMS can take **> 2 hours**.
- Log retention was 2–3 days, increased to **7 days** so failures not fixed in 2–3 days don't lose data.
- After **90 days**, data is archived.

### Assessment ⚠️
The answer is reasonable but **mixes up two different retentions** and gives **no numbers**.

### 🧠 Two different "log retention" concepts
| Where | What it means | Why it matters |
|---|---|---|
| **Oracle archived redo logs** | How long Oracle keeps archived redo logs on the source server | If DMS is down longer than this retention, the logs it needs are **gone** and CDC **cannot resume** → you must do a new full load |
| **S3 raw zone retention** | How long raw files stay in S3 (and in what storage class) | Controls storage cost and replay ability |

The candidate's "2–3 days → 7 days" story is most naturally the **Oracle-side archived log retention** (protects against DMS task outage). The "90 days then archive" is the **S3 lifecycle policy**.

### 🧠 S3 storage classes and lifecycle (what "archiving after 90 days" really means)

| Class | Use | Retrieval |
|---|---|---|
| S3 Standard | Hot, frequently accessed | Instant |
| S3 Intelligent-Tiering | Unknown/changing access | Instant (auto-tiers) |
| S3 Standard-IA / One Zone-IA | Infrequent access, min 30 days | Instant, per-GB retrieval fee |
| S3 Glacier Instant Retrieval | Rare access, ms retrieval | Instant |
| S3 Glacier Flexible Retrieval | Archive | Minutes to hours |
| S3 Glacier Deep Archive | Long-term compliance | ~12–48 hours |

**Lifecycle rule example:** transition `raw/` to Standard-IA at 30 days, Glacier at 90 days, expire at 7 years (insurance/regulatory retention).

```json
{
  "Rules": [{
    "ID": "raw-zone-archive",
    "Status": "Enabled",
    "Filter": { "Prefix": "raw/" },
    "Transitions": [
      { "Days": 30, "StorageClass": "STANDARD_IA" },
      { "Days": 90, "StorageClass": "GLACIER" }
    ],
    "Expiration": { "Days": 2555 }
  }]
}
```

### 🧠 Why the history load is slow, and how to speed it up
- DMS full load reads tables sequentially by default. **Speed-ups:** parallel-load with partition/range settings (`ParallelLoadThreads`, table-mapping `parallel-load`), larger replication instance, tuning `MaxFullLoadSubTasks`, LOB mode settings (limited LOB is much faster), disable secondary indexes/constraints on target (not applicable for S3), run during off-peak, split big tables.

### 💡 Ideal answer
> "Daily incremental volume is roughly X GB across Y tables (CDC), while the initial full load was Z TB and took ~2 hours per large table. Raw CDC files stay in S3 Standard for 30 days, then move to Standard-IA, Glacier at 90 days, and expire per the 7-year regulatory policy. On the Oracle side we keep archived redo logs for 7 days so a DMS outage doesn't force a full reload."

---

# 4. S3 versioning question

### What was asked
"Say you have a location in S3 and each run **overwrites** it, but you **don't want to lose the previous data**. What S3 capability would you use?"

### What the candidate said
Described **date partitions** — each day's data goes to a new date folder; the external table points only to the latest partition.

### Assessment ❌ (missed the intended answer)
The interviewer wanted **S3 Versioning**. The candidate answered with a *design workaround* (partitioning by date), which is valid engineering but not the *feature* asked about.

### 🧠 S3 Versioning explained
When **versioning is enabled** on a bucket:
- Every PUT to the same key creates a **new version** with a unique **Version ID**; the older version is **kept**.
- A normal DELETE doesn't remove data — it adds a **delete marker**; previous versions remain recoverable.
- You can **restore** by fetching a specific `versionId` or deleting the delete marker.
- Versioning can be **suspended** but never fully disabled once enabled.
- Cost: you pay for **all stored versions** → add a lifecycle rule to expire **noncurrent versions** (e.g., after 30 days).

```
Key: s3://bank-data/policies/latest.parquet
  Version v3 (current)   ← newest overwrite
  Version v2 (noncurrent)
  Version v1 (noncurrent)
```

```bash
# Enable versioning
aws s3api put-bucket-versioning --bucket my-bucket --versioning-configuration Status=Enabled

# List all versions of a key
aws s3api list-object-versions --bucket my-bucket --prefix policies/latest.parquet

# Restore an older version (copy it back as the current one)
aws s3api copy-object --bucket my-bucket --key policies/latest.parquet \
  --copy-source "my-bucket/policies/latest.parquet?versionId=<OLD_VERSION_ID>"
```

### Related protection features (interviewers love these as follow-ups)
| Feature | Purpose |
|---|---|
| **MFA Delete** | Requires MFA to permanently delete a version or change versioning state |
| **S3 Object Lock** | **WORM** (write-once-read-many): *Governance* mode (privileged users can override) or *Compliance* mode (nobody, not even root, can delete until retention ends); also *Legal Hold* |
| **Cross-Region Replication (CRR)** | Replicate to another region for DR (requires versioning) |

### Versioning vs date-partitioning (when to use which)
| | Versioning | Date partition folders |
|---|---|---|
| Same key overwritten? | Yes, old version kept | No — new key each day |
| Query old data | Need version ID | Just point to another partition |
| Best for | Protecting against accidental overwrite/delete, audit | Analytical data (Hive-style partitions), pruning |
| Cost control | Lifecycle for noncurrent versions | Lifecycle by prefix |

### 💡 Ideal answer
> "Enable **S3 Versioning** on the bucket. Each overwrite creates a new version and the previous one remains retrievable by version ID; deletes only add a delete marker. I'd add a lifecycle rule to expire noncurrent versions after N days for cost, and if regulatory immutability is needed, add **Object Lock** and MFA Delete. For analytics datasets I'd additionally partition by load date so each run writes a new prefix."

---

# 5. Reconciliation process

### What was asked
"Are you doing any reconciliation as part of the job? How is it scheduled?"

### What the candidate said
- After production loads finish, SQL queries compare **source vs warehouse** KPIs: **policy count, premium amount**.
- Result triggers an **SNS notification** — both on success and on mismatch.
- Scheduled **daily at 9:00 AM**; loads start after midnight and finish by ~7 AM.

### Assessment ✅ / ⚠️
Good: automated, KPI-based, daily, notifies. Gaps: limited checks (only counts and sums), no mention of **tolerances**, **row-level checks**, **column-level checks**, or **what is "the source" for the comparison**.

### 🧠 Reconciliation (recon) explained
**Reconciliation** = proving that what you loaded into the target **equals** what existed at the source (or what you expected).

| Level | Check | Example |
|---|---|---|
| Row count | `COUNT(*)` source vs target | 1,204,553 vs 1,204,553 |
| Aggregate | `SUM(premium)`, `MIN/MAX(date)` | Premium total matches to 2 decimals |
| Distinct keys | `COUNT(DISTINCT policy_id)` | Detects duplicates |
| Null checks | `% NULL` in mandatory columns | `customer_id` should never be null |
| Row-level hash | `MD5(concat_ws('|', col1, col2, ...))` per row, compare sets | Finds **which** rows differ |
| Referential | Facts have a matching dimension row | No orphan keys |
| Freshness | `MAX(updated_ts)` close to now | Detects stale data |

**Types of recon:** *source → ODS*, *ODS → fact/dim*, *fact → report*.

### Example recon SQL

```sql
-- Compare ODS vs fact for a given business date
WITH ods AS (
  SELECT COUNT(DISTINCT policy_id) AS policy_cnt, SUM(premium) AS prem_sum
  FROM ods.policy WHERE business_date = '2026-10-31'
),
fct AS (
  SELECT COUNT(DISTINCT policy_key) AS policy_cnt, SUM(premium_amt) AS prem_sum
  FROM dw.fact_premium WHERE business_date = '2026-10-31'
)
SELECT o.policy_cnt AS ods_cnt, f.policy_cnt AS fact_cnt,
       o.prem_sum  AS ods_prem, f.prem_sum  AS fact_prem,
       CASE WHEN o.policy_cnt = f.policy_cnt
             AND ABS(o.prem_sum - f.prem_sum) < 0.01
            THEN 'PASS' ELSE 'FAIL' END AS status
FROM ods o CROSS JOIN fct f;
```

### 🧠 Why 9:00 AM schedule is a (small) design smell
The recon runs at a **fixed time**, not **after the pipeline completes**. If the load finishes late (8:30 AM) or fails, recon may run on incomplete data. Better: make recon the **last step of the same Step Functions workflow** (dependency-driven, not clock-driven), or trigger it from the success event.

### 💡 Ideal answer
> "Recon is the final state in our Step Functions workflow. It compares source (Oracle counts/sums captured by a DMS validation or a source-query task) with ODS, and ODS with facts — row counts, distinct keys, premium sums within a tolerance, null checks, and a freshness check. Results are written to a **recon audit table** (run_id, table, metric, source_value, target_value, status) and published via SNS: summary to everyone, failure details to the support team."

---

# 6. Handling discrepancies and reprocessing

### What was asked
"If recon finds a discrepancy (e.g., corrupt record), how do you deal with it and reprocess?"

### What the candidate said
- First check **what kind of failure**: job failure vs KPI mismatch.
- Job failure → go to **Step Functions**, open the **ECS task's CloudWatch logs**, read the error, fix the code.
- KPI mismatch → know which fact the KPI comes from, **back-track through transformations** stage by stage to the source.

### Assessment ⚠️
Correct *debugging mindset*, but the interviewer asked about **corrupt records** and **reprocessing**. The answer didn't cover: **quarantining bad records**, **idempotent re-run**, **backfill by date**, **preventing a bad load from reaching consumers**.

### 🧠 A robust error-handling & reprocessing design

```
            ┌────────────┐      good rows       ┌───────────┐
 raw file → │ Validation │ ───────────────────► │  Curated  │
            │ (schema,   │                      └───────────┘
            │  null,type)│      bad rows        ┌───────────┐
            └────────────┘ ───────────────────► │ Quarantine│ (S3 prefix + reason column)
                                                └───────────┘
```

Principles:
1. **Don't fail the entire load for a few bad rows** (unless it's a hard rule): write bad rows to a **quarantine/reject** location with an `error_reason` column and a count metric.
2. **Idempotency:** loading the same batch twice must not duplicate data → `MERGE` on primary key, or `DELETE WHERE business_date = X` then `INSERT`.
3. **Parameterised reruns:** pipeline accepts `business_date` so you can **backfill** a specific day or range.
4. **Publish-after-validate** (a.k.a. Write-Audit-Publish): load to a staging table → validate → **swap/insert** into the final table only if checks pass. Consumers never see half-loaded data.
5. **Source of truth for replay:** the immutable raw zone in S3 (this is why you keep raw data).
6. **Root-cause trail:** `run_id` on every row/table so you can trace "which run produced this number".

### How to back-track a KPI mismatch (a concrete method)
1. Re-run the KPI at **each layer**: source → raw → curated → ODS → fact → report.
2. Find the **first layer where numbers diverge**.
3. In that layer, **diff at key level**: `FULL OUTER JOIN` on primary key between the two layers; list keys missing on either side or with different values.
4. Check **join fan-out** (duplicate dimension rows doubling sums), **filters** (WHERE clauses dropping rows), **data type truncation**, **time zone/late-arriving data**.

```sql
SELECT COALESCE(a.policy_id, b.policy_id) AS policy_id, a.premium AS ods_prem, b.premium_amt AS fact_prem
FROM ods.policy a
FULL OUTER JOIN dw.fact_premium b ON a.policy_id = b.policy_id
WHERE a.policy_id IS NULL OR b.policy_id IS NULL OR a.premium <> b.premium_amt;
```

### Step Functions error handling (what to mention)
- `Retry` (with backoff) for transient errors, `Catch` → route to a failure state that publishes to SNS and records the failure.
- Execution history gives per-state input/output for debugging.

### 💡 Ideal answer
> "Bad records are quarantined with a reason code, the rest continue. For a mismatch I locate the first layer where counts/sums diverge using layer-by-layer KPIs, then key-level diff. Fix is applied in code or data; I then **re-run for that business date** — loads are idempotent via MERGE — from the raw zone, and re-run recon. Changes are logged in an audit table and communicated to consumers."

---

# 7. Notifying consumers (SLA / freshness)

### What was asked
"If your job fails and today's data isn't published, consumers will see stale data. How do you **proactively notify** them?"

### What the candidate said
- First answer drifted to **access control** (views per country, permissions per user group) — didn't answer the question.
- After the prompt: SNS notification after run completes; consumers subscribed; a **Lambda** compares data and triggers notifications; production issue → Jira story.

### Assessment ⚠️
The second answer (SNS pub/sub + Lambda in Step Functions) is the right direction. Missed: **freshness/SLA monitoring**, **different audiences**, **status page/metadata table**.

### 🧠 Concepts

**SLA / SLO / SLI (data version)**
- **SLI** (indicator): measured value, e.g. "time data became available", "rows loaded".
- **SLO** (objective): target, e.g. "facts available by 8:00 AM IST on 99% of days".
- **SLA** (agreement): the contract with consequences.

**Data freshness** = how old is the latest data (e.g., `now - MAX(load_ts)`). **Staleness** is the opposite problem.

**SNS (Simple Notification Service) pub/sub**
- **Topic** → publishers send messages → **subscribers** (email, SMS, Lambda, SQS, HTTPS) receive them.
- **Message filtering:** subscribers can attach a **filter policy** on message attributes (e.g., `country=HK`) so each consumer group only gets relevant alerts.

### 💡 Ideal design

```
Step Functions workflow
   ├─ success path → [Publish "DATA READY" to SNS: attributes {dataset, date, country}]
   └─ Catch / failure → [Publish "DELAYED/FAILED" to SNS with ETA + owner]

Independent safety net (does not depend on the pipeline itself):
 EventBridge Scheduler (08:00) → Lambda "freshness checker" 
        → query status table / MAX(load_ts)
        → if stale: SNS alert to consumers + on-call (PagerDuty/Opsgenie/Teams/Slack)
```

Key insight: **the pipeline can't reliably tell you it's dead** (if it crashed hard, no failure message is sent). A **separate watchdog** that checks "did the expected data arrive by time T?" catches silent failures.

Extra good practices: a **pipeline status table** (dataset, business_date, status, completed_ts) exposed as a view so consumers can self-check; data contracts defining SLAs; CloudWatch alarms on Step Functions `ExecutionsFailed` and on a custom metric for freshness.

---

# 8. S3 global namespace and securing a bucket

## 8.1 "Can two S3 buckets have the same name?"

### Candidate: No, names are unique, S3 is global. ✅
Follow-up: **"Why does AWS need a global namespace if you also pick a region?"** → candidate didn't know.

### 🧠 Explanation
- A bucket **lives in exactly one region**, but its **name is unique across all AWS accounts and regions** (within an AWS partition).
- Reasons:
  1. **DNS-based addressing:** buckets are reachable as `https://<bucket>.s3.<region>.amazonaws.com/key` (virtual-hosted style). The bucket name becomes part of a **DNS hostname**, so it must be unique and DNS-compatible (lowercase, 3–63 chars, no underscores).
  2. **Simple global references:** other accounts, policies, URLs, and tools can refer to `s3://bucket/key` without an account ID or region.
  3. **Historical design:** S3 launched in 2006 with one global endpoint; the naming model stayed for backward compatibility.
- Practical effect: names like `company-prod-datalake-ap-south-1` (include company, env, region) reduce collisions. A deleted bucket's name can be taken by someone else, so **never reference a deleted bucket** (risk: bucket takeover).

### 💡 Ideal answer
> "Bucket names are globally unique because the name forms the DNS hostname (virtual-hosted-style URL) and lets any account reference the bucket without an account/region qualifier; the data itself still lives in the region chosen at creation."

## 8.2 "How can you secure an S3 bucket?"

### Candidate said
Permissions via **ACL**, allow/deny using **IAM** permissions.

### Assessment ⚠️ (too shallow — and the interviewer's final feedback highlights this gap)

### 🧠 Complete S3 security checklist

| Layer | Control | Detail |
|---|---|---|
| **Access** | **Block Public Access** | Account- and bucket-level switches that override any public ACL/policy. Turn all 4 on. |
| | **IAM policies (identity-based)** | Grant users/roles only needed actions on needed prefixes (**least privilege**). Avoid `"Action": "*"`/`"Resource": "*"`. |
| | **Bucket policy (resource-based)** | JSON on the bucket; supports cross-account, conditions, deny rules. |
| | **ACLs** | Legacy. Set **Object Ownership = Bucket owner enforced** to **disable ACLs** (AWS recommended). |
| | **Access Points** | Per-application endpoints with their own policy; simplify large shared buckets. |
| | **Presigned URLs** | Time-limited access without sharing credentials. |
| **Network** | **VPC Gateway Endpoint** + bucket policy `aws:SourceVpce` | Traffic stays on AWS network; bucket accessible only from your VPC. |
| **Encryption in transit** | Enforce **TLS** | Deny when `aws:SecureTransport` is `false`. |
| **Encryption at rest** | **SSE-S3** (AWS-managed keys; default on) | Free, automatic. |
| | **SSE-KMS** | Customer-managed KMS key; key policy + CloudTrail audit of every decrypt; best for regulated data. |
| | **DSSE-KMS** | Dual-layer encryption. |
| | **Client-side encryption** | Encrypt before upload; AWS never sees plaintext. |
| **Data protection** | **Versioning**, **MFA Delete**, **Object Lock**, **Replication** | Prevents accidental/malicious deletion; DR. |
| **Monitoring / audit** | **CloudTrail data events** | API-level audit of object GET/PUT/DELETE. |
| | **S3 server access logging** | Request logs delivered to another bucket. |
| | **AWS Config / Security Hub** | Detect misconfig (public bucket, no encryption). |
| | **IAM Access Analyzer for S3** | Finds buckets shared outside the account/org. |
| | **Amazon Macie** | Scans S3 for PII/sensitive data. |
| | **GuardDuty S3 protection** | Detects suspicious access patterns. |

### Example bucket policy: TLS only + VPC endpoint only

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::my-datalake", "arn:aws:s3:::my-datalake/*"],
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    },
    {
      "Sid": "AllowOnlyFromMyVpcEndpoint",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::my-datalake", "arn:aws:s3:::my-datalake/*"],
      "Condition": { "StringNotEquals": { "aws:SourceVpce": "vpce-0abc123def4567890" } }
    }
  ]
}
```
> ⚠️ A deny-all-except-VPCE policy also blocks console access and admin tooling from outside the VPC; test carefully or add exceptions for admin roles.

### 🧠 How IAM + bucket policy + ACL combine
- Request allowed if **any** allow applies **and** no explicit **deny** exists (explicit deny always wins).
- Same account: identity policy **or** resource policy allowing is enough.
- Cross account: **both** sides must allow (the other account's identity policy + your bucket policy).

### 💡 Ideal 30-second answer
> "Block Public Access on, ACLs disabled, least-privilege IAM and bucket policies, TLS enforced, SSE-KMS encryption, access only via VPC endpoint, versioning plus Object Lock for critical data, and CloudTrail data events plus access logs for audit. Macie to discover PII and Access Analyzer to catch unintended sharing."

---

# 9. Protecting PII data

### What was asked
"You work on customer data. PII must be secured. How will you design the pipeline / secure it? Beyond masking, what else?"

### What the candidate said
1. **Masking in views:** two schemas — *restricted* (real values) and *general* (masked via `CASE` in the view).
2. **Splitting PII columns** into separate files/tables/folders with restricted access (Python reads data, writes PII file to restricted folder, rest to normal folder).

### Assessment ⚠️
Both are legitimate techniques, but **incomplete**. Missing: **classification**, **encryption**, **tokenization/hashing**, **fine-grained access (column/row-level security)**, **audit**, **retention/deletion**, and **compliance** (very relevant in India: **DPDP Act**, plus **IRDAI** norms for insurance).

### 🧠 PII protection — layered model

**Step 0 — Discover & classify**
- Know *where* PII lives (Amazon Macie for S3, data catalog tags). Label columns: `PII`, `Sensitive`, `Public`.

**Step 1 — Minimize**
- Don't ingest PII you don't need. Drop it at the earliest layer.

**Step 2 — Protect the data itself**
| Technique | What it does | Reversible? | Use when |
|---|---|---|---|
| **Encryption** (KMS) at rest & TLS in transit | Unreadable without keys | Yes (with key) | Always |
| **Masking** (`XXXX1234`) | Hides part of value | No (static) / No (dynamic masking is query-time only) | Show partial data to analysts |
| **Hashing** (SHA-256 + salt) | One-way fingerprint | No | Joining on a key without exposing it |
| **Tokenization** | Replace value with random token; real value in a secure **vault** | Yes (via vault) | Card numbers, Aadhaar-like IDs |
| **Pseudonymization** | Replace identity with surrogate ID | Yes (mapping table protected) | Analytics on behaviour |
| **Anonymization / generalization** | Age → age band, city → state | No | Sharing externally |

**Step 3 — Control access**
- **Column-level security:** Redshift supports `GRANT SELECT (col1, col2) ON table TO role`; **Lake Formation** gives column/row/cell-level permissions on S3/Glue tables.
- **Row-level security (RLS):** e.g., users only see their country's rows (the candidate's Hong Kong vs Vietnam case) — Redshift RLS policies or Lake Formation data filters.
- **Dynamic Data Masking** in Redshift: policies mask a column depending on the querying role, without needing duplicate views.
- **Role-based access (RBAC)**, least privilege, separate roles for ETL vs analysts.

**Step 4 — Audit & monitor**
- CloudTrail, Redshift audit logging, Lake Formation access logs, S3 access logs; alerts on unusual access.

**Step 5 — Lifecycle & compliance**
- Retention periods, **right to erasure/correction** (DPDP), consent tracking, **data residency** (keep within India region if required), breach response.

### The candidate's "split into two files" approach — vertical partitioning
```
Original row:   policy_id | name | phone | aadhaar | premium | status
                        └──── PII ─────┘      └── non-PII ──┘

Restricted:     policy_id | name | phone | aadhaar       → s3://.../restricted/ (tight IAM)
General:        policy_id | premium | status             → s3://.../general/
```
- Works because **policy_id (a surrogate key)** lets authorized users re-join. 
- ⚠️ Pitfall: if the key itself is PII (like phone/PAN), the "general" table is still re-identifiable. Use a **surrogate key**.

### Example: Redshift column-level + masking

```sql
-- Column-level grant: analysts cannot see aadhaar
GRANT SELECT (policy_id, premium, status) ON dw.policy TO ROLE analyst;

-- View-based masking (candidate's approach)
CREATE VIEW dw.policy_general AS
SELECT policy_id,
       CASE WHEN is_member(...)... END   -- (pattern depends on role-check approach)
```
(Modern Redshift also offers **dynamic data masking policies** attached to roles — prefer these over copy-pasted CASE statements.)

### 💡 Ideal answer
> "I classify data first (Macie + catalog tags), minimize what we ingest, encrypt with KMS at rest and TLS in transit, tokenize or hash direct identifiers like national IDs, mask for analysts, enforce column- and row-level security through Lake Formation or Redshift, store PII in a separate restricted zone with a surrogate key to rejoin, log all access via CloudTrail, and apply retention and erasure rules aligned with DPDP/IRDAI."

---

# 10. File formats, small-files problem, CSV → Parquet

### What was asked
"What was the format of the raw data? How do you transform from raw? Any AWS service?"

### What the candidate said
- DMS outputs **CSV**, many small files (could be thousands/day).
- A **Python** job **merges** the small files, converts to **Parquet** (compressed) in chunks, saves to the "aggregate zone".
- External tables on top → ODS.

### Assessment ✅ / ⚠️
Correct instinct: **merge small files, convert to Parquet**. Interviewer then probed schema handling (next section) and asked if an AWS service could have been used (Glue/EMR) — the candidate used custom Python (probably on ECS).

### 🧠 Row vs columnar formats

| | CSV | JSON | Avro | **Parquet** | ORC |
|---|---|---|---|---|---|
| Layout | Row, text | Row, text | Row, binary | **Columnar, binary** | Columnar, binary |
| Schema inside file | ❌ | Self-describing keys | ✅ (in header) | ✅ (footer) | ✅ |
| Compression | Poor | Poor | Good | **Excellent** | Excellent |
| Typical use | Exchange | APIs, logs | Streaming/Kafka, row-wise writes | **Analytics** | Hive/analytics |
| Schema evolution | ❌ position-based | ⚠️ | ✅ strong | ✅ (add columns) | ✅ |

**Why Parquet wins for analytics**
1. **Column pruning:** query `SELECT premium` reads only the premium column's bytes.
2. **Predicate pushdown:** each **row group** stores min/max stats; engines skip row groups that can't match the filter.
3. **Compression + encoding** per column (dictionary, run-length) → 5–10× smaller than CSV.
4. **Typed:** stores real data types, avoiding "everything is a string".
Spectrum/Athena cost and speed depend on bytes scanned → Parquet cuts both.

### 🧠 The small-files problem (why they merged files)
- Each file → an S3 request + task overhead. Thousands of 50 KB files make Spark/Athena/Spectrum spend most time **listing and opening files**, not reading data.
- Target: **128 MB – 1 GB per file** (Parquet), partitioned sensibly (not so much that each partition has tiny files).
- Fixes: DMS batching settings (`CdcMinFileSize`), **compaction jobs** (Glue/Spark `coalesce/repartition`, Athena `OPTIMIZE` on Iceberg), or table formats (**Iceberg/Delta/Hudi**) that compact automatically.

### 🧠 Python approach — where it breaks
```python
import pyarrow as pa, pyarrow.csv as pv, pyarrow.parquet as pq
import s3fs

fs = s3fs.S3FileSystem()
files = fs.glob("my-raw/policy/load_date=2026-11-01/*.csv")

writer = None
for f in files:                                  # stream: don't load all in memory
    with fs.open(f, "rb") as fh:
        table = pv.read_csv(fh)
        if writer is None:
            writer = pq.ParquetWriter("s3://my-curated/policy/load_date=2026-11-01/part-0.parquet",
                                      table.schema, compression="snappy", filesystem=fs)
        writer.write_table(table.cast(writer.schema))
writer.close()
```
Pitfalls: memory limits (pandas loads everything), single-machine throughput, schema mismatch across files. For large scale → **AWS Glue (Spark)** or **EMR**, or let DMS write **Parquet directly** (`DataFormat=parquet`).

### 💡 Ideal answer
> "DMS lands CSV with an Op column. A scheduled Glue Spark job (or ECS task) compacts the small files into ~256 MB Snappy-Parquet files partitioned by load date, enforces the schema, and writes to the curated zone; Spectrum reads from there. Parquet gives column pruning, predicate pushdown and compression, which cuts Spectrum scan cost."

---

# 11. Schema drift / schema evolution

### What was asked
"CSV has no schema. How do you manage **schema drift/evolution** in this pipeline?"

### What the candidate said
1. On each new file: read into a DataFrame, get the **set of columns**.
2. Query Redshift (target), get its **set of columns**.
3. **Set difference** → new columns.
4. Generate `ALTER TABLE ADD COLUMN` statements with a generic type (**VARCHAR**), later fix type manually.
5. Send notification to users for confirmation.

### Interviewer follow-up
"Can't we use **Glue Catalog** to manage schema instead of connecting to source and target and comparing?" → Candidate: "Yes, we could; we didn't because the flow was already built around DMS + SQL."

### Assessment ⚠️ (reasonable homegrown design; interviewer wanted catalog awareness)

### 🧠 Types of schema change

| Change | Example | Difficulty |
|---|---|---|
| **Column added** | New `nominee_name` | Easy |
| **Column dropped** | `fax_number` removed | Medium (downstream break) |
| **Column renamed** | `prem_amt` → `premium_amount` | Hard (looks like drop + add) |
| **Type changed** | `int` → `decimal(12,2)` | Hard (can fail casts) |
| **Order changed** | Columns swapped | Fatal for **CSV without headers** (position-based) |

The candidate's **set difference** handles only **added columns**. It would miss **type changes**, **renames**, and **dropped columns**.

### 🧠 Better approaches

**A. Use a catalog + self-describing format**
- Store curated data as **Parquet** (schema embedded) and register in **Glue Data Catalog**. A **Glue crawler** (or a job with `enableUpdateCatalog`) updates table schemas.
- Spark can read evolved Parquet using `spark.read.option("mergeSchema","true").parquet(path)`.

**B. Open table formats (modern best practice)** — Iceberg / Delta / Hudi
- Support **ALTER TABLE ADD/DROP/RENAME COLUMN** safely, with **schema evolution tracked in metadata**, without rewriting data, plus time travel.

**C. Schema registry for streaming** — **AWS Glue Schema Registry** (Avro/JSON/Protobuf) for Kafka/MSK/Kinesis with compatibility rules (BACKWARD, FORWARD, FULL).

**D. DMS itself** propagates some DDL (e.g., add column) from source to target; for S3 targets it writes new columns in subsequent files — your downstream must handle them.

**E. Data contracts + validation** — producers publish the expected schema; pipeline validates against it and routes violations to quarantine.

### Compatibility rules (good to quote)
- **Backward compatible:** new readers can read old data (e.g., adding an optional column with a default).
- **Forward compatible:** old readers can read new data.
- **Breaking:** rename, delete required column, incompatible type change.

### Example: detect drift with Glue catalog versions + notify (sketch)
```python
import boto3
glue = boto3.client("glue")

def columns(version):
    return {c["Name"]: c["Type"] for c in version["Table"]["StorageDescriptor"]["Columns"]}

versions = glue.get_table_versions(DatabaseName="insurance", TableName="policy")["TableVersions"]
latest, previous = columns(versions[0]), columns(versions[1])      # newest first

added   = {c: t for c, t in latest.items() if c not in previous}
removed = {c: t for c, t in previous.items() if c not in latest}
changed = {c: (previous[c], latest[c]) for c in latest if c in previous and previous[c] != latest[c]}
# publish {added, removed, changed} to SNS if any is non-empty
```

### 💡 Ideal answer
> "Parquet in S3 registered in the Glue Data Catalog; a crawler (or event-driven catalog update) detects changes; each run compares the current and previous catalog table versions to classify added/removed/type-changed columns. Additions are auto-accepted (nullable); removals and type changes fail fast into quarantine and alert the owners. For long-term evolution I'd move to Iceberg tables."

---

# 12. Glue vs EMR vs EMR Serverless; instance selection

### What was asked
1. "For EMR we must specify cluster config / instance types. On what basis do you choose instance types?"
2. "EMR now has a Serverless offering. When would you use EMR Serverless over Glue?"

### What the candidate said
- Used **Glue**, not EMR. Glue is serverless; it scales per load.
- For EMR, you'd analyze **data volume and types**, then decide **nodes, executors, memory, cores**.
- Glue has Data Catalog, DataBrew, crawlers — "EMR doesn't have catalog" (candidate's claim).

### Assessment ⚠️ / ❌
Honest about limited EMR experience, but the **catalog claim is incorrect**: **EMR can use the Glue Data Catalog as its Hive metastore**.

### 🧠 Glue (ETL) — essentials
- **Serverless Spark (and Python shell, Ray, streaming)**; you choose **Glue version** (4.0/5.0), **worker type**, and **number of workers**.
- **DPU** (Data Processing Unit) = 4 vCPU + 16 GB RAM. Worker types: **G.1X** (1 DPU), **G.2X** (2 DPU), larger G.4X/G.8X; **Flex** execution class for cheaper, non-urgent jobs.
- Billed **per second** (with a minimum), based on DPU-hours.
- Includes **Data Catalog**, **Crawlers**, **Glue Studio** (visual ETL), **Workflows/Triggers**, **Job bookmarks** (incremental processing), **DataBrew** (separate no-code data prep), **Schema Registry**.
- Great for: standard ETL, catalog-centric lakes, small teams who don't want to manage clusters.

### 🧠 EMR — essentials
**Elastic MapReduce**: managed big-data framework platform (Spark, Hive, Presto/Trino, HBase, Flink...). Three deployment modes:

| Mode | What it is |
|---|---|
| **EMR on EC2** | Classic cluster you size and manage; maximum control |
| **EMR on EKS** | Run Spark on your Kubernetes cluster |
| **EMR Serverless** | No clusters; submit jobs; capacity auto-scales (Spark, Hive) |

**Node types (EMR on EC2)**
| Node | Role |
|---|---|
| **Primary (master)** | Coordinates; runs YARN ResourceManager, HDFS NameNode |
| **Core** | Run tasks **and** store HDFS data |
| **Task** | Compute only (ideal for **Spot** instances; losing them doesn't lose data) |

### 🧠 Choosing instance types (what the interviewer expected)
| Workload | Instance family | Why |
|---|---|---|
| General Spark ETL | **M** (m5/m6g/m7g) | Balanced CPU/RAM |
| Memory-heavy (joins, caching, wide shuffles) | **R** (r5/r6g) | High RAM per vCPU |
| CPU-heavy (compression, parsing, ML scoring) | **C** | More vCPU per GB |
| Shuffle-heavy / HDFS-local data | **I / D** | Fast local NVMe / dense storage |
| Deep learning | **P / G** | GPUs |

**Sizing method:**
1. Estimate **input size** and **shuffle size**; Spark needs roughly 2–3× input in memory/disk for large shuffles.
2. Choose **executor size** (commonly ~4–5 cores per executor, 4–8 GB per core) and number of executors = total cores ÷ cores per executor.
3. Aim for **partition size ~128–256 MB**; number of partitions ≈ 2–4 × total cores.
4. Start small, **measure** (Spark UI: skew, spill, GC), then tune.
5. Use **Graviton (arm64)** for better price/performance, **Spot** for task nodes, **managed scaling**.

### 🧠 EMR Serverless vs Glue — when to choose which

| Criteria | **AWS Glue** | **EMR Serverless** |
|---|---|---|
| Primary pitch | Managed ETL service with integrated catalog, crawlers, visual tools | Serverless runtime for open-source frameworks (Spark/Hive) |
| Control over Spark config / versions / libs | Limited to supported Glue versions | **High** (custom images, specific Spark versions, fine-grained config) |
| Capacity model | Workers × DPU | vCPU/memory per worker; **pre-initialized capacity** for fast start |
| Billing | DPU-hours | vCPU-hours + GB-hours (+ storage) |
| Best when | Catalog-driven ETL, quick development | Migrating existing EMR/Spark workloads, custom dependencies, cost tuning at scale |
| Catalog | Glue Data Catalog | **Can also use Glue Data Catalog** |

### 💡 Ideal answer
> "Glue for catalog-centric, low-ops ETL; EMR Serverless when I need tighter control over Spark versions/config/custom libraries or I'm lifting existing EMR code, while still using the Glue Data Catalog as the metastore. For EMR on EC2 I pick M-family for general ETL, R-family for memory-heavy shuffles, C for CPU-bound, with core nodes on On-Demand and task nodes on Spot, sized from input volume and measured via the Spark UI."

---

# 13. Glue Crawler and automated schema-change detection

### What was asked
"You use a Glue crawler on S3 data; schema is current today. A few days later the schema changes. **How will Glue know?** Design a workflow so ETL pipelines are aware of schema changes (e.g., a daily scan with alerts)."

### What the candidate said
- First: "Data Catalog stores metadata; when crawling, it checks the catalog and knows about the changes." ❌ (Not how it works.)
- After prompting: **compare two partitions** (today's vs two days back), diff the columns and data types, set a flag, and invoke **Lambda/SNS** for notification.

### Assessment ⚠️
The second approach (compare recent partitions → notify) is a **valid custom design**. The first statement misunderstands crawlers.

### 🧠 How Glue crawlers really work
- A **crawler** connects to a data store (S3/JDBC/etc.), samples/reads files, uses **classifiers** to infer schema (CSV, JSON, Parquet...), and **creates/updates tables and partitions** in the **Data Catalog**.
- **A crawler doesn't run by itself** — it runs **on demand, on a schedule, or event-triggered**. The catalog doesn't "notice" changes unless a crawler (or a job with catalog updates enabled) runs.
- **Schema change policy** on the crawler:
  - *Update behavior:* update the table definition in the catalog **or** just log changes.
  - *Delete behavior:* ignore, mark deprecated, or delete table/columns that disappeared.
- **Table versions:** each schema change creates a **new version** of the catalog table (retrievable via `GetTableVersions`) → you can diff them.
- **Recrawl policy:** crawl everything, or **only new folders**, or **S3 event–based** crawling (S3 notifications → SQS → crawler) for efficiency.

### 💡 Ideal automated design

```
 S3 (new files land)
     │  S3 event / daily schedule
     ▼
 EventBridge ─► Glue Crawler (schema change policy: LOG or UPDATE)
                       │ emits "Crawler State Change" event on completion
                       ▼
                 EventBridge rule ─► Lambda "schema-diff"
                       │   get_table_versions → compare latest vs previous
                       ├── no change → exit
                       ├── additive only → publish INFO to SNS (auto-continue)
                       └── breaking (type change / dropped column) → publish ALERT to SNS,
                            set status flag in DynamoDB/SSM, Step Functions branch
                            routes the pipeline to "hold & review"
```

### Lambda diff sketch
```python
import boto3, json, os
glue, sns = boto3.client("glue"), boto3.client("sns")

def cols(v): return {c["Name"]: c["Type"] for c in v["Table"]["StorageDescriptor"]["Columns"]}

def lambda_handler(event, context):
    db, tbl = os.environ["DB"], os.environ["TABLE"]
    v = glue.get_table_versions(DatabaseName=db, TableName=tbl, MaxResults=2)["TableVersions"]
    if len(v) < 2:
        return {"status": "no_previous_version"}
    new, old = cols(v[0]), cols(v[1])
    diff = {
      "added":   [c for c in new if c not in old],
      "removed": [c for c in old if c not in new],
      "type_changed": [c for c in new if c in old and new[c] != old[c]],
    }
    if any(diff.values()):
        sns.publish(TopicArn=os.environ["TOPIC_ARN"],
                    Subject=f"Schema change detected: {db}.{tbl}",
                    Message=json.dumps(diff, indent=2))
    return diff
```

### Related: how Glue **jobs** deal with drifting data
- **DynamicFrame** allows **choice types** when a column has mixed types; resolve with `resolveChoice` (cast, project, make_cols, make_struct).
- **Job bookmarks** only track processed files — they don't detect schema change.

---

# 14. DynamoDB + Elasticsearch: why two databases?

### What was asked
"Resume says DynamoDB and Elasticsearch. What was the use case, and why two databases?"

### What the candidate said
- Use case: **mandate data** (customers authorize auto-debit of premium from bank account) arriving as **CSV/TXT** from users.
- Ingestion with **Infoworks** (not DMS) → convert to **Parquet** → S3 raw → **Glue jobs** build **JSON structures** (StructType/StructField) → write to **DynamoDB**, and also to Elasticsearch.
- Why two stores:
  - Elasticsearch: complex queries, **full-text search**, real-time analytics.
  - DynamoDB: very fast, high throughput, NoSQL.
  - Update behavior: "If the **same partition key** is written again DynamoDB **overwrites**, while Elasticsearch **indexes** it and **keeps both**, so you can query latest data."
  - Consumers differ.

### Assessment ⚠️ (use-case reasoning good, some facts need correction)

### Corrections 🔧
1. **"Elasticsearch keeps both versions"** — only if documents are written with **different `_id`s** (e.g., auto-generated IDs). If you index with the **same `_id`**, Elasticsearch/OpenSearch **overwrites** the document (and increments `_version`), just like DynamoDB's `PutItem`. To keep history in either store, you must **design for it** (e.g., add a `version`/`timestamp` in the sort key, or write new docs).
2. **"DynamoDB doesn't index"** — DynamoDB *does* index: the **primary key** (partition key + optional sort key) is the main index, plus **secondary indexes** (LSI/GSI).
3. **"Employee ID and name as partition keys"** — A DynamoDB table has **one partition key** (hash) and **optionally one sort key** (range). A **composite primary key** = partition key + sort key. You can't have two partition keys (though a GSI can use a composite attribute).
4. **AWS naming:** AWS's managed service is now **Amazon OpenSearch Service** (Elasticsearch fork); the "Elasticsearch" name remains in older domains.

### 🧠 DynamoDB essentials
- Serverless, key-value/document NoSQL, **single-digit-millisecond** latency at any scale.
- Data is spread across **partitions** by hashing the **partition key**. Each partition ≈ 10 GB storage and up to 3,000 RCU / 1,000 WCU.
- **Item size limit: 400 KB.**
- Capacity modes: **Provisioned** (RCU/WCU, with auto scaling) or **On-demand** (pay per request).
- Consistency: eventually consistent reads (default), **strongly consistent** reads (same region, costs 2×), **transactions** (`TransactWriteItems`).
- **Access-pattern-first design:** you must know your queries up front — DynamoDB is bad for ad-hoc queries/joins/aggregations.
- Features: **TTL**, **Streams** (change feed), **Global Tables** (multi-region), **PITR** backups.

### 🧠 OpenSearch/Elasticsearch essentials
- Distributed **search and analytics engine** built on Lucene. Stores **JSON documents** in **indexes**, split into **shards** (primary + replicas).
- Strengths: **full-text search** (inverted index), relevance scoring, fuzzy match, aggregations, log analytics, dashboards (Kibana/OpenSearch Dashboards).
- Weakness: not a transactional primary store; cluster sizing/maintenance; eventual consistency (near-real-time refresh ~1 s).

### 🧠 The classic pattern: DynamoDB (system of record) + OpenSearch (search/query layer)
```
Writers → DynamoDB (fast key-based reads/writes, source of truth)
              │ DynamoDB Streams (or zero-ETL integration)
              ▼
         Lambda / OpenSearch Ingestion → OpenSearch (search, aggregations, dashboards)
```
Use the right tool per access pattern: "get mandate by ID" → DynamoDB; "search mandates by name/bank/status/date range" → OpenSearch.

### 💡 Ideal answer
> "Mandates are looked up by ID at high throughput by the payments service → DynamoDB with `mandate_id` as partition key. Operations and customer-service need free-text search and filtering across attributes, plus dashboards → OpenSearch. We load both from the same Glue job (or sync via DynamoDB Streams) so they stay consistent; writes use the same ID so updates overwrite in both, and we keep history in S3."

---

# 15. Sharding vs partitioning

### What was asked
"Explain the difference between database **sharding** and **partitioning**."

### What the candidate said
- Partitioning: split a large dataset into chunks across partitions for **parallel processing** and speed.
- Sharding: knows it from **Kinesis** — shards capture streaming data; each shard has a capacity (candidate guessed "100 records").

### Assessment ⚠️ (partial; Kinesis capacity figure wrong)

### 🧠 Definitions

**Partitioning** = dividing a large table/dataset into smaller pieces called partitions, typically **within one database/system**, by a rule:
- **Horizontal partitioning:** split by **rows** (e.g., by date: `2026-10`, `2026-11`).
- **Vertical partitioning:** split by **columns** (e.g., PII columns separate).
- Benefits: **partition pruning** (read only needed partitions), easier maintenance (drop old partition), parallelism.

**Sharding** = a form of **horizontal partitioning where partitions (shards) live on different servers/nodes** (shared-nothing). Each shard is an independent database holding a **subset of rows**; a **shard key** decides where each row goes.
- Benefits: scale beyond one machine's storage/throughput.
- Challenges: choosing the **shard key** (avoid **hot shards**), **cross-shard joins/transactions**, **resharding/rebalancing**.

| | Partitioning | Sharding |
|---|---|---|
| Location | Usually same server/cluster | **Different servers/nodes** |
| Goal | Query performance, manageability | **Horizontal scale** of storage & throughput |
| Failure impact | Server down → all partitions down | One shard down → only part of data affected |
| Examples | Hive/Spark/Athena `dt=2026-11-01`, Postgres declarative partitions, Redshift sort/dist | MongoDB shards, DynamoDB partitions (automatic), Cassandra, Kafka partitions, Kinesis shards |

### 🧠 "Shard" in different systems (don't confuse them)
| System | Meaning |
|---|---|
| **Kinesis Data Streams** | **Shard** = unit of capacity: **1 MB/s or 1,000 records/s write**, **2 MB/s read** (shared across consumers; enhanced fan-out gives each consumer its own 2 MB/s). Records with the same **partition key** go to the same shard (ordering per shard). |
| **Kafka/MSK** | **Partition** = ordered log; one consumer per partition within a consumer group |
| **DynamoDB** | Internal **partitions** (managed automatically) |
| **OpenSearch** | **Primary shards** + replicas of an index |
| **Spark** | **Partitions** of an RDD/DataFrame processed by tasks |

### Shard key choice — interview gold
- Good key: **high cardinality, evenly distributed** (e.g., `customer_id`).
- Bad key: low cardinality or skewed (e.g., `country`, `status`) → **hot shard**.
- **Hash-based sharding** → even distribution, hard range scans. **Range-based** → good range scans, hot-spot risk (e.g., sequential IDs/timestamps all hit the newest shard).
- **Consistent hashing** reduces data movement when adding nodes.

### 💡 Ideal answer
> "Partitioning splits data into manageable pieces, often within one system, for pruning and parallelism. Sharding is horizontal partitioning across multiple nodes so the system scales beyond one machine; it needs a good shard key to avoid hotspots. Kinesis shards are capacity units — 1 MB/s or 1,000 records/s in, 2 MB/s out."

---

# 16. DynamoDB secondary indexes (LSI vs GSI)

### What was asked
"What are secondary indexes in DynamoDB? Use cases and cons?"

### Candidate: "Not much aware, will look into it." ❌ (honest, but it's a core DynamoDB topic)

### 🧠 Why secondary indexes exist
A DynamoDB table can be efficiently queried **only by its primary key**. If you need another access pattern (e.g., "all policies of a customer" when the table key is `policy_id`), scanning the table is slow and expensive. A **secondary index** creates an **alternate key** to Query efficiently.

### Example table
```
Policies table
  Partition key: policy_id       (get one policy)
  Attributes: customer_id, status, premium, start_date
Need: "list all policies for customer C123 sorted by start_date"  → GSI on customer_id + start_date
```

### LSI vs GSI

| Feature | **Local Secondary Index (LSI)** | **Global Secondary Index (GSI)** |
|---|---|---|
| Partition key | **Same** as base table | **Different** (any attribute) |
| Sort key | **Different** from base table | Optional, any attribute |
| When can you create it? | **Only at table creation** | **Any time** |
| Limit per table | **5** | **20** (default, adjustable) |
| Throughput | **Shares** base table's capacity | **Own** RCU/WCU (provisioned) |
| Read consistency | Eventually **or strongly** consistent | **Eventually consistent only** |
| Size constraint | Item collection (same partition key) max **10 GB** | None |
| Scope | Within a single partition-key value | Across the whole table |

### Projection (what's copied into the index)
`KEYS_ONLY`, `INCLUDE (selected attrs)`, `ALL`. More projected attributes → faster reads (no fetch back to table) but **more storage and write cost**.

### Cons / trade-offs (what the interviewer wanted)
1. **Extra cost:** each write to the table also writes to every index (**write amplification**) and indexes consume storage.
2. **GSI eventual consistency:** a just-written item may not show immediately.
3. **Throttling back-pressure:** if a GSI is under-provisioned, **writes to the base table get throttled**.
4. **LSI limits:** must be defined at creation; 10 GB item-collection cap.
5. **Hot keys:** a low-cardinality GSI key (e.g., `status = ACTIVE`) creates a hot partition (use **write sharding** — append a random suffix).
6. **Design burden:** indexes are not free-form; Query needs the exact key.

### Code example
```python
resp = table.query(
    IndexName="customer-start_date-index",
    KeyConditionExpression=Key("customer_id").eq("C123") & Key("start_date").gte("2026-01-01")
)
```

### 💡 Ideal answer
> "Secondary indexes give alternate query patterns without table scans. An LSI shares the table's partition key with a different sort key, must be created with the table, supports strong consistency, and caps item collections at 10 GB. A GSI has a different partition and sort key, can be added later, has its own throughput and is eventually consistent. The cost is extra storage, write amplification and possible throttling if the GSI is under-provisioned."

---

# 17. Scenario design: ad-hoc S3 arrivals + 30-min SLA + audit + lineage

### The problem statement (restated)
- A business unit drops files into S3 **ad hoc** (no fixed schedule).
- **SLA:** process each arrival within **30 minutes**.
- Apply **business logic**.
- Need **logging, auditing, and data lineage** (track source and transformations).

### What the candidate said
1. **S3 PUT event → Lambda** to start processing (event-driven, no schedule).
2. If data is large: Lambda **triggers a Glue job**.
3. **Logging:** CloudWatch.
4. **Lineage:** "RDD lineage in Spark" → interviewer challenged it.
5. **Auditing:** **CloudTrail** → interviewer: "that's infra-level, but what about our *pipeline* events (Lambda → Glue)?" → candidate suggested **Airflow DAGs** for dependencies/lineage.

### Assessment ⚠️ Trigger design ✅; lineage/audit ❌

### 🧠 Why "RDD lineage" ≠ "data lineage" (important!)
| | Spark RDD lineage | Data lineage |
|---|---|---|
| Meaning | The DAG of transformations needed to **recompute a lost partition** | **Business-level trace**: which source files/tables → which jobs → which outputs, who/when |
| Scope | Inside a single running Spark job (in memory) | Across systems, over time, persisted |
| Purpose | **Fault tolerance** | Audit, impact analysis, compliance, debugging |

The candidate described RDD lineage correctly (lazy evaluation, DAG, recompute on node failure) but it doesn't answer *auditable data lineage*.

### 🧠 Reference architecture (what to say)

```
Business unit uploads → S3 landing bucket
        │  (s3:ObjectCreated:*  — filtered by prefix/suffix)
        ▼
 EventBridge (or S3 → SQS queue) ── buffer + DLQ + dedupe
        ▼
 Step Functions (state machine = the audited workflow, run_id = execution ID)
   1. Validate  (Lambda: file name pattern, size, checksum, schema check → reject to quarantine)
   2. Record start in AUDIT table (DynamoDB / Redshift): run_id, source_s3_uri, etag, size, start_ts
   3. Transform (choose by size: small → Lambda;  large → Glue/EMR Serverless job via .sync)
   4. Data-quality checks (Glue DQ / Deequ) 
   5. Publish output (curated S3 prefix / Redshift COPY) + update catalog
   6. Record end in AUDIT: rows_in, rows_out, rows_rejected, output_uri, status, end_ts
   7. SNS notify (success/failure)
 Failure → Catch → quarantine + SNS + AUDIT status=FAILED
 SLA monitoring: CloudWatch alarm on (now - arrival_ts) > 25 min or execution duration > 25 min
```

### 🧠 Key design points the interviewer is probing

1. **Event-driven trigger:** S3 Event Notifications can target **Lambda, SQS, SNS, or EventBridge**. S3 events are **at-least-once** (duplicates possible) → processing must be **idempotent** (use ETag/key + `run_id` check in a state table).
2. **Multi-file arrivals:** If a "dataset" is several files, start processing on a **manifest/`_SUCCESS` marker file**, not on each file.
3. **Multipart uploads:** event fires on **CompleteMultipartUpload** — only after the full object is available (good).
4. **Don't process inside Lambda for large files** — 15-min timeout, 10 GB memory max, 10 GB `/tmp` max. Use Lambda as **orchestrator/trigger**, Glue/EMR Serverless/ECS for heavy work. Mind **Glue startup time** (tens of seconds to a few minutes) in the 30-min budget; consider **pre-initialized capacity** (EMR Serverless) or keep Glue worker count right-sized.
5. **Meeting a 30-minute SLA:** measure **end-to-end latency = job end − file arrival**; alarm at ~80% of SLA; auto-scaling (Glue auto scaling), avoid queueing behind other jobs (concurrency limits), and set timeouts + retries.
6. **Decouple with a queue:** S3 → SQS → Lambda starter gives buffering, retries, and DLQ for poison events.

### 🧠 Auditing vs logging vs lineage — three different things

| Need | What it answers | AWS tools / approach |
|---|---|---|
| **Logging** | What happened inside each component? (debugging) | CloudWatch Logs (Lambda, Glue, Step Functions), structured JSON logs with `run_id` |
| **Auditing** | Who/what ran what, when, with what result? | **Custom audit table** (run_id, file, rows, status, timestamps), **Step Functions execution history**, **CloudTrail** (API calls such as StartJobRun, S3 data events), S3 access logs |
| **Lineage** | Where did this column/table come from? Which jobs touched it? | Catalog + lineage tool: **Amazon DataZone / SageMaker Catalog lineage** (OpenLineage-compatible events), **OpenLineage/Marquez** with Spark listener, **Glue Data Catalog** + table properties (source/job/run tags), dbt docs for SQL models, Lake Formation for governance |

**Cheap, effective lineage inside your pipeline:** stamp every output row/file with metadata columns: `source_file`, `source_etag`, `run_id`, `job_name`, `job_version`, `ingest_ts`. Then lineage questions ("which input produced this row?") are answerable with a simple query.

### 🧠 CloudTrail clarification (interviewer's point)
CloudTrail records **AWS API calls** (who called `StartJobRun`, `PutObject`, etc.). It tells you *that* Lambda started Glue, not *what data* moved or *what transformation* occurred. For pipeline-level audit you need **application-level audit records** + **orchestrator history**.

### Why the candidate's "Airflow" idea is partly right
An orchestrator (Airflow/Step Functions) gives **task-level run history and dependencies**, which supports operational lineage, but it is not data/column lineage by itself.

### 💡 Ideal answer (60 seconds)
> "S3 event → EventBridge/SQS → Step Functions. The state machine validates the file, writes an audit record with run_id and source URI, runs the transformation (Glue/EMR Serverless for large data, Lambda for small), runs data-quality checks, publishes output and writes the final audit record with row counts and status. Failures are caught, quarantined and alerted via SNS. CloudWatch alarms watch the 30-minute SLA by tracking arrival-to-completion latency. Lineage comes from stamping outputs with run_id/source metadata and emitting OpenLineage events to a catalog such as DataZone; CloudTrail adds API-level audit."

---

# 18. CI/CD and infrastructure deployment

### What was asked
"How do you deploy your pipeline? Any DevOps tool?"

### What the candidate said
- Internal framework built by the team.
- **Git** repo → developers raise **pull requests**; reviewer approves and merges.
- **Jenkins** pipeline (transcript "genin") picks up the repo, deploys the stack.
- A **platform team** owns **Terraform** code that provisions resources and permissions (e.g., S3 access); the data team doesn't deploy infra directly.

### Assessment ✅ (fine answer, could mention testing & promotion)

### 🧠 The pipeline explained

```
Developer → feature branch → Pull Request → code review + automated checks
        → merge to main → Jenkins job:
             1. Checkout
             2. Lint / unit tests (pytest, SQL lint)
             3. Package (zip / Docker image → ECR)
             4. terraform init / plan (review) / apply   ← infra by platform team modules
             5. Deploy code (Glue scripts to S3, Step Functions definition, Lambda)
             6. Smoke test / integration test in DEV
             7. Promote → QA/UAT → PROD (approval gate)
```

### 🧠 Terraform essentials
- **Infrastructure as Code**: declarative `.tf` files describe resources.
- Workflow: `terraform init` → `plan` (shows diff) → `apply`.
- **State file** maps config to real resources; keep it **remote** (S3 backend with locking) so teams don't clobber each other.
- **Modules** = reusable components (e.g., a "glue-job" module with standard IAM + logging).
- Benefits: reviewable, repeatable, auditable, environment parity.
- Alternatives on AWS: CloudFormation, **AWS CDK**, SAM (serverless).

### Things worth adding in an interview
- **Environment separation** (separate AWS accounts for dev/test/prod), **secrets** in Secrets Manager (never in Git), **least-privilege deploy roles**, **automated tests** (unit tests for transformations, data-quality tests), **rollback strategy**.

---

# 19. Lambda's 15-minute limit

### What was asked
"Lambda can't run more than 15 minutes. Suppose you need **sequential Lambda triggers** — one Lambda completes (before timeout) and triggers another, and processing **continues from the same state**. How would you implement it?"

### What the candidate said
1. Lambda can invoke another Lambda.
2. Split the transformations: Lambda 1 does steps 1–2, writes an **intermediate file to S3**, and passes the location as an argument to Lambda 2.
3. Alternative: write intermediate results to **Redshift** and let the next step continue from there.

### Assessment ⚠️ Works, but is the *less preferred* design; a purpose-built service exists.

### 🧠 Lambda limits to know cold
| Limit | Value |
|---|---|
| Max timeout | **15 minutes (900 s)** |
| Memory | 128 MB – **10,240 MB** (CPU scales with memory) |
| `/tmp` ephemeral storage | 512 MB default – **10,240 MB** |
| Payload (sync invoke) | **6 MB** request/response |
| Payload (async invoke) | 256 KB (AWS has been raising this — verify current docs) |
| Package size | 50 MB zipped (direct upload), 250 MB unzipped; **container image up to 10 GB** |
| Default concurrency | 1,000 per region per account (soft limit) |

### 🧠 Options to process work longer than 15 minutes

| Option | How | Verdict |
|---|---|---|
| **A. Lambda invokes Lambda** (candidate's) | Async-invoke the next function with state in the payload | ⚠️ Works, but hard to monitor, retries are scattered, risk of **runaway chains** and cost; "Lambda calling Lambda" is generally an anti-pattern |
| **B. Step Functions orchestration** ✅ | State machine calls Lambda tasks in sequence/loops; state flows via JSON or S3 pointers | **Best practice**: visual, retries/catch, 1-year max duration (Standard), audit trail |
| **C. Chunk + fan-out** | Split input (e.g., by byte range / S3 prefix) → SQS or **Step Functions Distributed Map** → many parallel Lambdas | Great for parallelizable work |
| **D. Checkpoint & resume** | Lambda checks `context.get_remaining_time_in_millis()`; before timing out it **saves a cursor/checkpoint** (S3/DynamoDB) and re-enqueues itself via SQS/Step Functions | Good for loops over big data |
| **E. Use a different compute** | **Glue / EMR Serverless / ECS Fargate / AWS Batch** for long-running or heavy jobs | Often the right answer if each unit of work is long |

### Checkpoint pattern (code)
```python
import json, time, boto3
sqs = boto3.client("sqs")
SAFETY_MS = 60_000   # stop 60s before timeout

def lambda_handler(event, context):
    cursor = event.get("cursor", 0)                 # state passed from previous run
    rows = load_rows_from(cursor)                   # e.g., S3 range read / DB keyset pagination

    for row in rows:
        if context.get_remaining_time_in_millis() < SAFETY_MS:
            # out of time → persist progress and continue in a new invocation
            sqs.send_message(QueueUrl=QUEUE_URL, MessageBody=json.dumps({"cursor": cursor}))
            return {"status": "CONTINUE", "cursor": cursor}
        process(row)
        cursor += 1

    return {"status": "DONE", "cursor": cursor}
```

### Step Functions loop (conceptual)
```
Start → [Lambda: process chunk (returns status + cursor)] → Choice
          ├─ status == CONTINUE → back to the Lambda task with {"cursor": ...}
          └─ status == DONE     → [SNS notify] → End
```
State between invocations is carried in the **state JSON** (limit **256 KB**) — for bigger state, store in S3 and pass only the S3 URI.

### Why intermediate data in S3 is better than Redshift (as the candidate's second idea)
Redshift is a warehouse, not a general scratch store. S3 is cheaper, decoupled and fits "pass a pointer" patterns. Use Redshift only if the next step actually needs SQL on that data.

### 💡 Ideal answer
> "I'd use **Step Functions** to orchestrate multiple Lambda tasks, passing state or an S3 pointer between steps, with retries and catch handlers. For loops over big input I'd make the Lambda checkpoint its cursor using `get_remaining_time_in_millis()` and let the state machine re-invoke it, or fan out chunks with Distributed Map. If a single unit of work is inherently long, I'd move it to Glue or Fargate rather than fight the 15-minute limit."

---

# 20. Athena questions

### Q1. "Table created without partitions; next day data is partitioned. Will Athena detect partitions automatically?"
**Candidate:** Thinks you must supply partitions; unsure whether creating a table without partitions errors. ⚠️

### 🧠 Athena in 30 seconds
- **Serverless interactive SQL** on data in **S3** (Trino/Presto-based engine). No loading; pay **per data scanned** (~$5 per TB; verify regional pricing).
- Table definitions live in the **Glue Data Catalog** (or Hive metastore); **metadata only** — data stays in S3.

### Answers
1. **Creating a table without partitions does NOT error.** Athena simply reads **all files under the table's LOCATION** (recursively). 
2. **Partitions are not auto-discovered.** If a table is defined with `PARTITIONED BY (dt string)`, Athena only knows partitions registered in the catalog. Ways to add them:

| Method | When |
|---|---|
| `MSCK REPAIR TABLE tbl;` | Folders follow Hive style `dt=2026-11-01/`; scans S3 and adds missing partitions (slow on huge layouts) |
| `ALTER TABLE tbl ADD PARTITION (dt='2026-11-01') LOCATION 's3://bucket/path/dt=2026-11-01/';` | Explicit, fast, scriptable (Lambda after each load) |
| **Glue Crawler** | Auto-adds partitions on schedule/event |
| **Partition Projection** ✅ | Define partition pattern/ranges in table properties; Athena **computes partitions at query time** — no catalog updates, great for date partitions |

```sql
CREATE EXTERNAL TABLE policy_events (
  policy_id string, premium double
)
PARTITIONED BY (dt string)
STORED AS PARQUET
LOCATION 's3://my-curated/policy/'
TBLPROPERTIES (
  'projection.enabled'='true',
  'projection.dt.type'='date',
  'projection.dt.format'='yyyy-MM-dd',
  'projection.dt.range'='2024-01-01,NOW',
  'storage.location.template'='s3://my-curated/policy/dt=${dt}/'
);
```

**Important:** if the folder layout is *not* `key=value`, MSCK won't work — use `ADD PARTITION` or projection.

### Q2. "Does Athena physically load data? It has a catalog/metastore."
**Candidate:** No — it refers to data in S3. ✅ Correct.
Add: Athena stores only **metadata** (schema, partitions, location). Query results are written to an **S3 results bucket**. Performance/cost tips: Parquet/ORC, partitioning, compression, avoid `SELECT *`, `CTAS` to rewrite data into optimized form.

### Q3. "Can Athena in one region query an S3 bucket in another region?"
**Candidate:** Not aware.

### 🧠 Answer
- **Yes, it is technically possible** — Athena can read S3 data from a bucket in a different region (the table's `LOCATION` can point to it), but:
  - You pay **cross-region data transfer** charges and get **higher latency**.
  - The Glue catalog and Athena workgroup are **regional**; the table is defined in Athena's region.
  - Some features (e.g., certain encryption/KMS setups) need extra cross-region permissions.
- **Best practice:** keep Athena, Glue Catalog, and S3 data in the **same region**; if data must be elsewhere, **replicate** it (S3 Cross-Region Replication) rather than query remotely. (Also relevant for data-residency rules.)

---

# 21. Live SQL problem: top-selling product per category

## 21.1 Problem statement (as given)

Three tables for an e-commerce dataset:

| Table | Granularity | Columns (as described) |
|---|---|---|
| `orders` | One row per order (high level) | `order_id`, `order_date`, `total_amount` (may be inaccurate → **ignore**) |
| `order_details` | One row per product line in an order (granular) | `order_id`, `product_id`, `quantity`, `unit_price` |
| `products` | One row per product | `product_id`, `product_name`, `category` |

**Task:** For the **last quarter**, find, **within each category**, the **top-selling product** where *top-selling = highest revenue*, with **revenue = Σ(quantity × unit_price)**.
**Output columns:** `category`, `product_name`, `revenue`.

**Follow-up requirement:** If two products in a category tie on revenue, return **only one row per category** — the one that is **alphabetically first** by product name.

## 21.2 Sample data (so you can run and verify)

Assume today is **2026-11-01**, so "last quarter" = **Q3: 2026-07-01 to 2026-09-30**.

```sql
CREATE TABLE products (product_id INT, product_name VARCHAR(50), category VARCHAR(50));
INSERT INTO products VALUES
 (1,'Pen Basic','Stationery'), (2,'Notebook','Stationery'),
 (3,'Laptop Pro','Electronics'), (4,'Headphones','Electronics'), (5,'Mouse','Electronics');

CREATE TABLE orders (order_id INT, order_date DATE, total_amount DECIMAL(12,2));
INSERT INTO orders VALUES
 (101,'2026-07-10',0),(102,'2026-08-05',0),(103,'2026-09-20',0),
 (104,'2026-10-02',0),   -- current quarter (exclude)
 (105,'2026-06-28',0);   -- previous quarter (exclude)

CREATE TABLE order_details (order_id INT, product_id INT, quantity INT, unit_price DECIMAL(10,2));
INSERT INTO order_details VALUES
 (101,1,100,5),   -- Pen Basic   500
 (101,3,2,800),   -- Laptop Pro  1600
 (102,2,50,10),   -- Notebook    500
 (102,4,5,200),   -- Headphones  1000
 (103,5,10,160),  -- Mouse       1600
 (104,4,100,200), -- Oct (excluded)
 (105,2,1000,10); -- June (excluded)
```

Revenue in Q3: Pen 500 · Notebook 500 (tie in Stationery) · Laptop 1600 · Mouse 1600 (tie in Electronics) · Headphones 1000.

**Expected result:**

| category | product_name | revenue |
|---|---|---|
| Electronics | Laptop Pro | 1600 |
| Stationery | Notebook | 500 |

(Laptop Pro < Mouse alphabetically; Notebook < Pen Basic.)

## 21.3 Step-by-step solution logic (how to think aloud)

1. **Clarify** (see 21.6): what is "last quarter"? calendar quarter or last 3 months? Are product names unique? Include cancelled orders? Currency/discounts?
2. **Filter** orders to the last quarter by `order_date`.
3. **Join** `orders → order_details → products` (`INNER JOIN` is correct: only orders that have detail lines contribute revenue).
4. **Aggregate**: `SUM(quantity * unit_price)` per **product** (group by `product_id`, plus `category`, `product_name`).
5. **Rank within each category** with a window function: `PARTITION BY category ORDER BY revenue DESC, product_name ASC`.
6. **Filter** to `rank = 1` (use `ROW_NUMBER` to guarantee a single row).

## 21.4 Final query (PostgreSQL / Redshift)

```sql
WITH quarter_sales AS (
    SELECT  p.category,
            p.product_id,
            p.product_name,
            SUM(od.quantity * od.unit_price) AS revenue
    FROM    orders o
    JOIN    order_details od ON od.order_id = o.order_id
    JOIN    products p       ON p.product_id = od.product_id
    WHERE   o.order_date >= DATE_TRUNC('quarter', CURRENT_DATE) - INTERVAL '3 months'
      AND   o.order_date <  DATE_TRUNC('quarter', CURRENT_DATE)
    GROUP BY p.category, p.product_id, p.product_name
),
ranked AS (
    SELECT  *,
            ROW_NUMBER() OVER (
                PARTITION BY category
                ORDER BY revenue DESC, product_name ASC   -- tie-breaker folded into one window
            ) AS rn
    FROM quarter_sales
)
SELECT category, product_name, revenue
FROM   ranked
WHERE  rn = 1;
```

### Variants for other databases
```sql
-- Snowflake: previous calendar quarter
WHERE o.order_date >= DATEADD(quarter, -1, DATE_TRUNC('quarter', CURRENT_DATE))
  AND o.order_date <  DATE_TRUNC('quarter', CURRENT_DATE)

-- MySQL: previous calendar quarter
WHERE o.order_date >= DATE_SUB(MAKEDATE(YEAR(CURDATE()),1) + INTERVAL (QUARTER(CURDATE())-1) QUARTER, INTERVAL 1 QUARTER)
  AND o.order_date <  MAKEDATE(YEAR(CURDATE()),1) + INTERVAL (QUARTER(CURDATE())-1) QUARTER

-- With QUALIFY (Snowflake, BigQuery, Redshift, Teradata) — no outer query needed
SELECT category, product_name, revenue
FROM quarter_sales
QUALIFY ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC, product_name) = 1;
```

## 21.5 RANK vs DENSE_RANK vs ROW_NUMBER (the heart of the follow-up)

For Electronics revenue values 1600, 1600, 1000 ordered DESC:

| product | revenue | ROW_NUMBER | RANK | DENSE_RANK |
|---|---|---|---|---|
| Laptop Pro | 1600 | 1 | 1 | 1 |
| Mouse | 1600 | 2 | 1 | 1 |
| Headphones | 1000 | 3 | 3 | 2 |

- `RANK` / `DENSE_RANK`: ties share the same number → **two rows** have `rank = 1` (the candidate correctly predicted this and also said RANK skips the next number ✅).
- `ROW_NUMBER`: always unique → with a **deterministic tie-breaker** in `ORDER BY` (product_name) it returns exactly one row per category.
- The candidate first proposed a **second window function** with `ROW_NUMBER`, then realized simply adding **`product_name ASC` as a second ORDER BY key** solves it in one window ✅ — the cleaner solution.

## 21.6 Where the candidate went wrong / what the interviewer noticed

| Moment | What happened | Lesson |
|---|---|---|
| **"Assuming product ID and name unique, group by category"** | The candidate assumed each product had one category and, at one point, considered grouping by `category` only (which would **sum all products in a category into one number**). | The interviewer pointed to the products table: a category (e.g., "Business Card") has **multiple products (A, D)**. **Never assume — ask.** |
| **LEFT JOIN vs INNER JOIN** | Candidate chose **LEFT JOIN** "to avoid data loss". | For revenue, orders without detail rows contribute nothing — **INNER JOIN** is correct and clearer. A LEFT JOIN plus a filter on the right table (`WHERE od.x ...`) silently turns into an inner join anyway. |
| **"Last 3 months"** | Candidate used last 3 months from today. | The report is **quarterly** → clarify "calendar quarter?" and use exact bounds (`>= start AND < next_start`). Avoid `BETWEEN` with timestamps (can miss the last day's time portion). |
| **`LIMIT 1`** | First idea: order by revenue and limit 1. | That returns **one row overall**, not per category. Per-group top-N = **window function**. |
| **Second window function** | Initially added `ROW_NUMBER` after `RANK`. | Fine, but can be done in **one window with two sort keys**. |

### 💡 Clarifying questions you should ask in a real interview
1. "Is *last quarter* the previous **calendar quarter**, or the **last 3 months** rolling?"
2. "Are `product_name` values unique, or can two products share a name (even across categories)? Should I group by `product_id`?"
3. "Should **cancelled/returned orders** be excluded? Is there an order `status` column?"
4. "Any **discounts/taxes** to apply, or just `quantity × unit_price`?"
5. "What about **ties** — return all tied products or exactly one?" *(this exact question became the follow-up)*
6. "Which SQL engine (Redshift/Snowflake/MySQL/Postgres)? Is `QUALIFY` available?"

### Performance notes (extra credit)
- Filter `orders` on `order_date` **before** joining to reduce rows.
- Aggregate at `product_id` level first, then join to `products` if the table is huge.
- In Redshift: **sort key** on `order_date`, **dist key** on `order_id` for orders/order_details to co-locate joins.

---

# 22. Interviewer's feedback explained

Paraphrased from the closing remarks:

| Feedback | Meaning | How to act on it |
|---|---|---|
| **"You were very good at SQL; logical thinking is strong."** | Positive: breaks problems down, explains reasoning aloud | Keep narrating your thought process |
| **"Don't assume anything — clarify assumptions with the interviewer."** | The candidate assumed unique product names/categories; the data showed otherwise | State assumptions explicitly: "I'm assuming X — is that right?" |
| **"Focus on data security."** | Weak answers on S3 security and PII (only ACL/IAM and masking) | Learn the layered security model (Sections 8 and 9) |
| **Specific security items mentioned:** encryption, server-side logging, IAM roles, **least-privilege** (avoid `*` permissions) | Standard AWS security vocabulary | Be able to say each term and give an example |
| **"Good at designing, but security is important for a data engineer."** | Design answers were decent | Add security/governance to every design |
| **"Little more on different services/infrastructure details."** | Gaps: DynamoDB indexes, Athena partitions, EMR, cross-region | Revise service fundamentals (Sections 12–20) |
| **"You're clear on the use case for DynamoDB + Elasticsearch."** | Positive on justification of two stores | Still verify technical claims (Section 14 corrections) |

---

# 23. Scorecard and improvement plan

## 23.1 My assessment by topic

| Topic | Rating | Comment |
|---|---|---|
| Project architecture | 🟢 Strong | Clear, layered, realistic |
| CDC/DMS knowledge | 🟡 Good | Needs correct Oracle terms, DMS tuning details |
| S3 versioning | 🔴 Missed | Answered with partitioning instead |
| Reconciliation | 🟡 Good | Add tolerance, audit table, dependency-based scheduling |
| Error handling / reprocessing | 🟡 Okay | Debugging OK; missing quarantine & idempotent re-run |
| Consumer notifications | 🟡 Okay | Needs freshness watchdog |
| S3 security | 🔴 Weak | Only ACL/IAM |
| PII | 🟡 Okay | Masking & splitting good; missing encryption, RLS/CLS, tokenization, audit |
| File formats / Parquet | 🟢 Good | Solid reasoning |
| Schema evolution | 🟡 Okay | Custom solution works for adds only |
| Glue / EMR | 🟡 Honest but thin | Wrong claim about EMR catalog |
| Glue crawler behavior | 🔴 Weak initially | Recovered with custom approach |
| DynamoDB / OpenSearch | 🟡 Okay | Use-case good, details need fixing |
| Sharding vs partitioning | 🟡 Partial | Kinesis facts wrong |
| DynamoDB indexes | 🔴 Gap | Not known |
| Scenario design | 🟡 Okay | Trigger good; lineage/audit weak |
| CI/CD | 🟢 Good | Clear flow |
| Lambda limits | 🟡 Okay | Missed Step Functions |
| Athena | 🔴 Gap | Partition behavior, cross-region |
| SQL | 🟢 Strong | Logic good, clarify assumptions |

## 23.2 7-day improvement plan

| Day | Focus | Output |
|---|---|---|
| 1 | S3 deep-dive: versioning, lifecycle, security, Object Lock | Build a bucket with SSE-KMS, TLS-only policy, versioning, lifecycle; write notes |
| 2 | IAM least privilege, KMS, Macie, Lake Formation, Redshift CLS/RLS/DDM | Mini lab: column-level grants + masking policy |
| 3 | DynamoDB: keys, LSI/GSI, capacity modes, Streams; OpenSearch basics | Create a table with a GSI, query it, stream to Lambda |
| 4 | Glue (jobs, crawlers, catalog versions, DQ), EMR vs EMR Serverless | Crawler + schema-diff Lambda from Section 13 |
| 5 | Athena (partition projection, CTAS), Redshift Spectrum | Convert CSV → Parquet with CTAS; compare scanned bytes |
| 6 | Orchestration: Step Functions patterns, Lambda limits, event-driven ingestion with audit table | Implement the Section 17 architecture end to end (small scale) |
| 7 | SQL drills (top-N per group, ties, running totals, SCD2) + mock interview | Re-answer this entire interview aloud, timing yourself |

---

# 24. Model answers cheat sheet

(Memorize the *shape*; adapt the specifics to your own projects.)

| Question | 30-second answer |
|---|---|
| **Describe your architecture** | "Oracle → DMS (full load + CDC from redo logs) → S3 raw (CSV) → compaction to Parquet in curated zone → Redshift Spectrum external tables → ODS (MERGE for upserts/deletes) → dimensional facts/dims; Step Functions orchestration; SNS alerts; recon table." |
| **Keep old data when overwriting in S3** | "S3 Versioning (+ lifecycle for noncurrent versions; Object Lock/MFA delete for immutability)." |
| **How do you reconcile?** | "Final workflow step: counts, distinct keys, sums within tolerance, null/freshness checks between source↔ODS↔fact; results to an audit table; SNS alert on failure." |
| **Bad records?** | "Quarantine with reason; continue good rows; idempotent MERGE; rerun by business date from the raw zone." |
| **Notify consumers of delays?** | "SNS topic per dataset with filtered subscriptions; failure path publishes; plus an independent freshness watchdog at the SLA time." |
| **Why are S3 bucket names global?** | "Name becomes DNS host `bucket.s3.region.amazonaws.com` and allows universal references; data lives in one region." |
| **Secure S3** | "Block public access, no ACLs, least-privilege IAM/bucket policy, TLS-only, SSE-KMS, VPC endpoint restriction, versioning+Object Lock, CloudTrail data events + access logs, Macie, Access Analyzer." |
| **Protect PII** | "Classify → minimize → encrypt → tokenize/hash → mask → column/row-level security → separate restricted zone → audit → retention/erasure (DPDP)." |
| **Why Parquet?** | "Columnar, compressed, typed; column pruning + predicate pushdown → lower scan cost; compact small files to ~256 MB." |
| **Schema drift** | "Parquet + Glue Catalog; diff table versions; auto-accept additive; alert/quarantine on breaking changes; Iceberg for long-term evolution." |
| **Glue vs EMR Serverless** | "Glue for catalog-centric managed ETL; EMR Serverless for tighter Spark control/custom libs; both can use the Glue Catalog." |
| **Crawler detecting changes** | "Crawlers run on schedule/event; catalog stores table versions; Lambda diffs versions on crawler completion and alerts via SNS." |
| **DynamoDB + OpenSearch** | "DynamoDB = key-based low-latency source of truth; OpenSearch = search/aggregation layer fed via Streams." |
| **Sharding vs partitioning** | "Partitioning splits data (often one system); sharding spreads partitions across nodes for scale and needs a good shard key." |
| **LSI vs GSI** | "LSI: same PK, new SK, create-time only, shared throughput, strong reads, 10 GB limit. GSI: new PK/SK, anytime, own throughput, eventual, extra write cost." |
| **30-min SLA design** | "S3 event → EventBridge/SQS → Step Functions → Glue/EMR Serverless → DQ → publish; audit table + CloudWatch SLA alarm; lineage via run_id stamping + OpenLineage." |
| **Lambda 15-min limit** | "Step Functions orchestration with checkpoints/chunking, or move long work to Glue/Fargate." |
| **Athena partitions** | "Not automatic: MSCK REPAIR, ADD PARTITION, crawler, or partition projection." |
| **Top product per category** | "Aggregate revenue per product; `ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC, product_name)`; filter rn=1." |

---

# 25. Follow-up questions the interviewer could have asked

Practice these — each is a natural next step from the answers in this interview.

**DMS / CDC**
1. How does DMS handle **DDL changes** and **LOB columns**? What are limited vs full LOB modes?
2. How do you handle **DMS task failure** and resume CDC without data loss? (checkpoint/log sequence number, retention of archived logs)
3. How do you handle **deletes** from CDC in the warehouse (hard vs soft delete)?
4. How do you validate DMS data? (DMS data validation feature, recon queries)

**Redshift**
5. Explain **distribution styles** (KEY, ALL, EVEN, AUTO) and **sort keys**. When would you pick each?
6. What are **Spectrum** limitations/cost model? When to load vs query external?
7. How do you tune a slow query? (`EXPLAIN`, `SVL_QUERY_REPORT`, `VACUUM`, `ANALYZE`, WLM)
8. Difference between **Redshift Serverless** and provisioned.

**S3 / security**
9. Difference between **SSE-S3, SSE-KMS, SSE-C**? What is **envelope encryption**?
10. How do bucket policies and IAM policies interact in cross-account access?
11. What are **S3 Access Points** and **Object Ownership**?

**Glue / Spark**
12. What are **job bookmarks**? How to process incrementally?
13. How do you handle **data skew** in Spark? (salting, AQE, broadcast joins)
14. **Coalesce vs repartition**? When to use which?
15. **Narrow vs wide transformations**; what triggers a shuffle?

**Streaming / NoSQL**
16. Kinesis vs Kafka/MSK vs SQS? Ordering, retention, replay, throughput.
17. How do you avoid a **hot partition** in DynamoDB?
18. What are DynamoDB **Streams** and typical consumers?

**Orchestration / design**
19. Step Functions **Standard vs Express**; when to use Airflow (MWAA) instead?
20. How do you make a pipeline **idempotent**? Exactly-once vs at-least-once.
21. How do you implement **SCD Type 2** in Redshift?
22. How would you implement **data quality** (Glue Data Quality, Deequ, Great Expectations)?

**SQL**
23. Find the **second-highest** salary / Nth highest per group.
24. Compute **running total** and **month-over-month growth**.
25. Remove **duplicates** keeping the latest record.
26. Explain **CTE vs subquery vs temp table**.

---

# 26. Glossary and transcript term corrections

| Term | Meaning |
|---|---|
| **ACL** | Access Control List — legacy per-bucket/object permission list in S3 (disable via Object Ownership) |
| **Archived redo logs** | Oracle's saved copies of redo logs; DMS CDC reads them |
| **CDC** | Change Data Capture — capturing inserts/updates/deletes from a source |
| **CTAS** | `CREATE TABLE AS SELECT` (in Athena it can write optimized Parquet to S3) |
| **DAG** | Directed Acyclic Graph — dependency graph of tasks/transformations |
| **DMS** | AWS Database Migration Service |
| **DPU** | Glue Data Processing Unit (4 vCPU, 16 GB) |
| **DPDP Act** | India's Digital Personal Data Protection Act, 2023 |
| **EventBridge** | Serverless event bus/scheduler routing events between AWS services |
| **GSI / LSI** | Global / Local Secondary Index (DynamoDB) |
| **IRDAI** | Insurance Regulatory and Development Authority of India |
| **Idempotent** | Running an operation multiple times gives the same result as running it once |
| **KMS** | AWS Key Management Service |
| **Least privilege** | Grant only the minimum permissions needed |
| **ODS** | Operational Data Store |
| **OLTP / OLAP** | Transactional / analytical processing |
| **PII** | Personally Identifiable Information |
| **Quarantine zone** | Location for rejected records with reasons for later review |
| **RDD** | Resilient Distributed Dataset (Spark's low-level data abstraction) |
| **Recon** | Reconciliation between source and target |
| **SLA / SLO / SLI** | Agreement / Objective / Indicator of service level |
| **SNS** | Simple Notification Service (pub/sub) |
| **Spectrum** | Redshift feature to query S3 data via external tables |
| **WORM** | Write Once, Read Many (S3 Object Lock) |

---

### ✅ How to use this document
1. Read each section's **ideal answer** aloud until it's natural.
2. For every ❌/⚠️ item, spend 20–30 minutes on a **hands-on** mini-lab (create it in your AWS account).
3. Re-record yourself answering the whole interview in 45 minutes, using the cheat sheet in Section 24.

*End of document.*
